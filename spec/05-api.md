# Спецификация — 05. Программный API

> Заземление: З2 (технологии — gRPC, OpenAPI/grpc-gateway), З1 (поиск в обе
> стороны, UI+API), ADR-3/4/5 (модель, уникальность, поиск), ADR-8 (TS UI), spec/01
> (пути), spec/04 (stale-индикация).
> **Rev v0.4 (раунд 2 loop):** единый список ошибок, добавлены Search*ByPhone с
> match-mode в таблицу, контракт токен→owner_id, чтение с primary (strong), семантика
> ResolvePhone (Exact|Prefix), типы полей/ошибок→gRPC-статусы. Ревью — 
> `reviews/findings_loop_r2.md`.

## Транспорт и контракты
- **gRPC** — основной внутренний контракт (proto-пакет `contacts.v1`).
- **grpc-gateway** — публикует HTTP REST + **OpenAPI (Swagger)** поверх тех же
  методов для веб-браузера (З2).
- Клиент веб-UI — TypeScript (ADR-8); типы для клиента можно генерировать из proto.
- Авторизация: вызывающая сторона аутентифицируется (токен сотрудника); все
  операции привязаны к текущему пользователю (owner scoping).

### Формальный proto-контракт (impl-1)
Контракт фиксируется proto/OpenAPI-файлами (пакет `contacts.v1`), а не прозой.
**Полная схема — `spec/contacts.v1.proto`** (поля, сообщения, пагинация,
staleness, маппинг ошибок); ей управляются текст ниже и TypeScript-клиент
(генерация из proto). Ниже — сводная модель методов и ошибок.

**Службы и методы (gRPC → REST-путь через grpc-gateway):**

| gRPC-метод | REST | Роль | Семантика |
| --- | --- | --- | --- |
| `Employees.Create` | `POST /v1/employees` | admin | создать сотрудника (I2) |
| `Employees.Get` | `GET /v1/employees/{id}` | all | прочитать сотрудника |
| `Employees.Update` | `PATCH /v1/employees/{id}` | admin | обновить (version) |
| `Employees.Delete` | `DELETE /v1/employees/{id}` | admin | tombstone |
| `Employees.SearchByName` | `GET /v1/employees?name={prefix}` | all | префикс по имени |
| `Employees.SearchByPhone` | `GET /v1/employees/by-phone?phone={num}&mode={exact\|prefix}` | all | точный/префикс номер в каталоге (I2) |
| `Employees.ResolvePhone` | `GET /v1/resolve-phone?phone={num}&mode={exact\|prefix}` | all | составной номер→каталог+моя книга |
| `Contacts.Add` | `POST /v1/me/contacts` | owner | добавить контакт (I3) |
| `Contacts.Get` | `GET /v1/me/contacts/{id}` | owner | прочитать |
| `Contacts.Update` | `PATCH /v1/me/contacts/{id}` | owner | обновить (version) |
| `Contacts.Delete` | `DELETE /v1/me/contacts/{id}` | owner | tombstone |
| `Contacts.SearchByName` | `GET /v1/me/contacts?name={prefix}` | owner | префикс в своей книге |
| `Contacts.SearchByPhone` | `GET /v1/me/contacts/by-phone?phone={num}&mode={exact\|prefix}` | owner | номер→имя в своей книге |
| `Profile.Whoami` | `GET /v1/profile` | all | principal/owner_id |

**Сообщения (сводка; полные тела — `spec/contacts.v1.proto`):**
`Employee{employee_id, full_name, work_phone, department?, title?,
external_subject?, is_admin?, version}`, `Contact{contact_id, owner_id, name,
phone, version}`. Имена полей соответствуют proto (R5-M5). Поле `deleted` не
отдаётся read-контрактам (tombstone только в репликации, I6; `external_subject`
— только админу, R5-M1). Обогащаются метаданными отсталости (см. ниже).

**Канонический список кодов ошибок** (единый, используется UI и клиентом):
`OK`, `NOT_FOUND`, `DUPLICATE_PHONE`, **`DUPLICATE_SUBJECT`** (дубль
`external_subject`), `VALIDATION_ERROR`, `VERSION_CONFLICT`, `FORBIDDEN`,
`SERVICE_UNAVAILABLE`. Каждый — код gRPC (маппинг: `NOT_FOUND`→`NotFound`,
`DUPLICATE_PHONE`→`AlreadyExists`, `DUPLICATE_SUBJECT`→`AlreadyExists`,
`VALIDATION_ERROR`→`InvalidArgument`, `VERSION_CONFLICT`→`Aborted`,
`FORBIDDEN`→`PermissionDenied`, `SERVICE_UNAVAILABLE`→`Unavailable`) +
человекочитаемое сообщение; UI отображает их (spec/07).

**Типизированный detail (обязательный, R5-B1):** поскольку несколько логических
кодов мапятся в один gRPC-статус (`DUPLICATE_PHONE` и `DUPLICATE_SUBJECT` → оба
`AlreadyExists`), каждый ответ об ошибке **обязан** нести структурированный
типизированный признак канонического кода (напр. `google.rpc.ErrorInfo.reason`,
где `reason = "DUPLICATE_SUBJECT"` и т.п.), а не только человекочитаемое сообщение.
UI и клиент различают сценарии **по этому detail, а не парсингом текста** (spec/07).
Маппинг «канонический код ⇄ reason ⇄ gRPC-статус» — контракт, не «на усмотрение».

**Типы полей (impl-14, v0.5):**
- `version` — `int64`, монотонный счётчик записи, **начинается с 1** (0 зарезервирован).
- `expected_version` — **`optional int64`** (impl-19): в proto3 `optional` отделяет
  «не указано» (0/отсутствует) от «ожидаю 0». Семантика: поле не задано → применяется
  последняя версия (semantics server-trivial); задано и не совпадает с текущей →
  `VERSION_CONFLICT`. Начальная версия — 1.
- `staleness` — `Staleness.Known` (proto) с **`oneof { bool current; int64 seconds; }`**
  (R5-M6): `current` и `seconds` взаимоисключающие (см. «Поле отсталости»);
  заполняется только при распределённой конфигурации (annex, zoom-3).
- Пагинация: запрос `{limit: int32, page_token: string?}`; ответ `{items, 
  next_page_token: string?, has_more: bool}` (курсор spec/06). `limit` вне [20,200] —
  clamp к границам (impl-27).
- `client_request_id` — `string?`; область дедупликации: сервер хранит его
  на оконченный срок (≥ 5 мин, retention/GC конфигурируемо), переживает рестарт без
  потери идемпотентности. **Replay-семантика (R5-idem):** повторный вызов с тем же
  `client_request_id` и тем же payload возвращает результат исходной операции
  (созданную запись/её версию) без повторной записи; **тот же id с другим payload** →
  `VALIDATION_ERROR`. Ключ дедупликации — `(client_request_id, owner)`.
- Match-режим для поиска по номеру — `enum Exact|Prefix` (см. ResolvePhone).
  **Имена (R5-M7/N4):** в gRPC/proto поле зовётся `mode`, значения —
  `MATCH_EXACT|MATCH_PREFIX`; в REST-запросе — `mode={exact|prefix}`; в прозе/UI —
  `match_mode`. Это один и тот же параметр; мост спека↔proto (`mode`) и
  proto↔HTTP-кодировка (`MATCH_*` → `exact|prefix`) применяется семантически
  (bridge-заметка, см. также ПУ-2 spec/01).
- Tombstone: чтения не возвращают `deleted=true`-записи (I6); `deleted` поле есть в
  модели для репликации, но не отдаётся read-контрактам.

## Общая семантика
- Область поиска: корпоративный каталог (для всех) и приватная книга текущего
  пользователя (только свои, spec/02 I5). Чужие книги — недоступны (ADR-5).
- Чтение возвращает признак отсталости реплики (см. ниже и spec/04); это аннекс-поле
  (при однодцовой конфигурации всегда `current`, zoom-3).
- **Staleness на изменяющих ответах (R5-M11):** записи идут только на primary
  (ADR-6), поэтому в мутирующих ответах `staleness` всегда `current` (либо absent) —
  никогда `stale`. Поле там присутствует для симметрии контракта и не несёт
  дополнительной информации; UI не обрабатывает его на записи.
- Изменяющие операции идут только на primary; проверяют уникальность телефона в
  области (I2/I3) и атомарны (ADR-9).
- Ошибки — типизированные, **единый канонический список** в «Формальном
  proto-контракте» (см. выше); здесь не дублируется (struct-6).

## Методы — корпоративный каталог
Операции над каталогом (`Employee`). **Изменяющие операции — только для роли
«администратор каталога»** (spec/05 «Роли и права»); чтение/поиск — для всех
аутентифицированных:
- `CreateEmployee(full_name, work_phone, dept?, title?) → Employee`
  — проверка глобальной уникальности work_phone (I2); дубль → `DUPLICATE_PHONE`.
  Идемпотентен: клиент может передать `client_request_id`; при повторе того же
  `client_request_id` повторно дубликат не создаётся (особая семантика retry).
- **Единая семантика update (R5-B3):** для `UpdateEmployee` и `UpdateContact`
  частичное обновление управляется `update_mask` (proto). Masked поле
  записывается как значение на проводе (пустое значение `""`/`false` = очистка
  опционального поля); unmasked поле — без изменений. Пустая маска = полная
  замена. Присутствие-семантика proto3-`optional` не override-ит маску: маска —
  единственный канон выбора полей. Это же правило снимает неопределённый
  «sentinel очистки» для `department`/`title`/`name`/`is_admin`.
- `GetEmployee(employee_id) → Employee`
- `UpdateEmployee(employee_id, {..}, expected_version?) → Employee`
  — атомарно (ADR-9); если `expected_version` задан и не совпадает →
  `VERSION_CONFLICT`; при отсутствии — обновление последней версии (spec/02).
- `DeleteEmployee(employee_id, expected_version?) → ok` — tombstone (ADR-9);
  `expected_version` absent → удалить последнюю версию; задан и не совпадает →
  `VERSION_CONFLICT` (R5-M12).
- `SearchEmployeesByName(prefix, pagination) → [Employee]` — префикс (spec/06).
- `SearchEmployeeByPhone(phone, match_mode=Exact) → [Employee]`
  — по умолчанию **точное совпадение** по каноническому номеру (полный номер
  сотрудника, обычно 1 по I2); с `match_mode=Prefix` — префикс по индексированным
  формы (spec/06, spec/02 «Нормализация»). Метод назван по умолчанию-точности, режим
  управляется явно (struct-2). Для составного ответа «каталог + моя книга»
  используется `ResolvePhone`.

## Методы — приватная книга (для текущего пользователя)
Набор операций над своими контактами (`Contact`):
- `AddContact(name, phone, client_request_id?) → Contact` — локальная уникальность
  телефона в книге (I3); дубль в своей книге → `DUPLICATE_PHONE`; идемпотентен по
  `client_request_id`.
- `GetContact(contact_id) → Contact`
- `UpdateContact(contact_id, {..}, expected_version?) → Contact` — атомарно;
  `expected_version` → `VERSION_CONFLICT` при расхождении (spec/02); частичное
  обновление — по `update_mask` (R5-B3, см. выше).
- `DeleteContact(contact_id, expected_version?) → ok` — tombstone (ADR-9);
  `expected_version` absent → удалить последнюю; несовпадение →
  `VERSION_CONFLICT` (R5-M12).
- `SearchContactsByName(prefix, pagination) → [Contact]` — в своей книге.
- `SearchContactsByPhone(phone, match_mode=Exact|Prefix) → [Contact]` — точное/префикс
  по номеру → имена в своей книге; `match_mode` задаётся как в `ResolvePhone`
  (struct-2), по умолчанию `Exact`.

## Аутентификация и «текущий пользователь» (🟡 M8 resolved)
- Каждый вызов предъявляет **токен сотрудника**; сервер из токена получает
  **principal** и отображает его на *запись сотрудника каталога*
  (employee_id = owner_id для приватной книги). Если principal не найден в
  каталоге — доступ к приватной книге и к personal-данным запрещается (`FORBIDDEN`),
  каталог для чтения по-прежнему доступен (гостевой-чтение), если разрешено политикой.
- Формат/проверка токена и привязка principal→owner_id — **закреплены как контракт
  здесь** (impl-13), а не отложены:
  - **Формат токена:** Bearer-токен, несёт **principal** — стабильный
    идентификатор сотрудника (subject claim). Сервер валидирует подпись/актуальность
    (TLS + подпись токена; детали крипто — при реализации, но наличие principal в
    токене — контракт).
  - **Маппинг principal→employee_id:** сервер хранит соответствие
    `principal → employee_id` (тот же движок, catalog-запись). Если principal
    соответствует существующей записи каталога — `owner_id = employee_id`.
  - **Путь создания привязки (impl-20):** запись сотрудника каталога несёт
    **опциональное поле `external_subject`** (постоянный subject-клейм токена
    этого сотрудника). При создании сотрудника администратор (spec/05 роли)
    заполняет `external_subject`; сервер индексирует `external_subject →
    employee_id` как единичное-соответствие. Так scoping I5 получает вход:
    токен → principal → external_subject → employee_id(=owner_id). При дубле
    `external_subject` на двух записях сервер возвращает ошибку `DUPLICATE_SUBJECT`
    (маппится на `AlreadyExists`, spec/05 «Канонический список»).
  - **Lifecycle `external_subject` (R5-B4):** привязка **иммутабельна на время
    жизни живой записи** — очистка/смена уже созданной привязки через API
    запрещена (иначе ломается scoping I5 и возможна подмена владельца/привилегии;
    после того как стало известно про смену subject — создаётся `VERSION_CONFLICT`/
    отказ изменения этой привязки). При **tombstone** запись умирает, и её
    `external_subject` освобождается для новой записи (как и `work_phone`, см.
    spec/03 «Переиспользование номера»); осиротевшие book-строки по старому
    `owner_id` сохраняются и владельцу не переназначаются (retention/GC — вне
    ядра, R5-M3). Флаг `is_admin` мутабелен отдельно (только админ).
  - **Конфиденциальность subject (R5-M1):** `external_subject` — персонально-
    идентифицирующий клейм; в read-контрактах он возвращается **только админу
    каталога**, обычным читателям каталога — unset (см. proto Employee).
  - **Оценка авторизации по strong-виду (R5-M2):** решение об авторизации
    (`is_admin`) и резолвинг principal→employee_id выполняются **всегда по
    primary/strong-виду каталога**, никогда по отставшей реплике (иначе снятый
    админ ещё пишет / новый ещё залочен, а bootstrap падает на stale). `Whoami`
    — strong-read по контракту.
  - **Роль admin — единый источник (impl-31):** администрирующие права определяет
    **только флаг `is_admin` в записи сотрудника каталога** (поле `is_admin` в
    proto). Claim `role=admin` из токена **не используется** как источник прав:
    токен служит лишь для аутентификации principal→external_subject→employee_id,
    а роль берётся из этой записи. Это устраняет неоднозначность двух источников.
    Флаг задаётся администратором при create/update и не самоназначается через
    API (только админ меняет; первый админ — через bootstrap).
  - **Bootstrap первого админа / получения subject (design-22):** до первой
    привязки каталог пуст и никто не mapped. Для первичной посадки: первый
    аутентифицированный вызов `Profile().whoami` без привязки возвращает **raw
    subject-клейм** токена (без прав каталога), и развёртывающая сторона seed-ит
    запись первого админа с этим subject через ручную операцию/стартовый скрипт.
    После этого привязка доступна, и админ заводит остальных.
  - **Если principal нет в каталоге:** приватизация запрещается (`FORBIDDEN`),
    доступ к каталогу на чтение остаётся (гостевой) при разрешении политикой.
- `Profile(): whoami` — **всегда** возвращает `{bound, subject, employee_id?,
  full_name?, is_admin?}` (proto): итого демонстрирует и привязанного, и
  непривязанного пользователя (R5-B2). При `bound=true` наполняются
  `employee_id`/роль/имя из записи каталога; при `bound=false` — только raw
  `subject` (для bootstrap, см. выше). Резолвинг principal→запись всегда идёт
  по strong-виду каталога (R5-M2).

## Роли и права (rev v0.2, M1 resolved)
- **Обычный сотрудник** — читает корпоративный каталог и ведёт **только свою**
  приватную книгу (create/update/delete/search своих контактов).
- **Администратор каталога** — дополнительно управляет записями корпоративного
  каталога (`CreateEmployee`/`UpdateEmployee`/`DeleteEmployee`), ведёт список
  сотрудников. Право админа определяется **флагом `is_admin` в записи сотрудника**
  (единый источник, impl-31; см. «Аутентификация»); чужие приватные книги админ не
  видит (принцип владельца I5 сохраняется).
- Обычный сотрудник без роли админа при вызове изменяющих методов каталога получает
  `FORBIDDEN`. Каталог — для чтения всем аутентифицированным (и настраиваемо —
  гостям), запись — только админу.

## Решение номера: составной метод ResolvePhone (impl-3; семантика v0.4)
- **`ResolvePhone(phone, match_mode=Exact|Prefix) → {catalog, mine}`** — **единый
  обязательный** метод для закрытия ПУ-2/ПУ-5. Выполняет поиск номера по
  корпоративному каталогу и по приватной книге текущего пользователя, разделяя
  ответ по источникам.
- **Match-режим закреплён (struct-2/design-20):** поле `match_mode` в запросе
  определяет поведение обеих веток единообразно:
  - `Exact` — полный номер: точные совпадения по канонической форме (по I2 в
    каталоге обычно 0..1; в своей книге — уникальные для неё I3).
  - `Prefix` — незавершённый ввод: префиксные совпадения по индексированным
    формам (spec/06, spec/02 «Нормализация» п. 6), первая страница + курсор.
  - Правило для UI «незнакомый номер»: полный номер (Exact), частичный — Prefix.
- **Семантика каталог-ветки согласована:** `ResolvePhone` в режиме `Exact`
  ведёт себя как `SearchEmployeeByPhone` (точный); в режиме `Prefix` — как
  префиксный поиск spec/06. Точечные `SearchEmployeeByPhone`/`SearchContactsByPhone`
  — те же одиночные источники с тем же `match_mode`. Единое правило: полный номер →
  точный, частичный → префикс. Список ошибок и типы — общие (см. выше).
- Дедупликация между источниками не делается: один и тот же номер физически может
  быть и в каталоге, и в книге.

## Чтение с primary (strong consistency, design-16)
- Для «гарантированно-актуального» результата (например, «свободен ли номер» при
  уникальности I2, или анализ перед решающим действием) клиент может запросить
  **`read_consistency=strong`** (по умолчанию `default`/replica). В режиме `strong`
  операция идёт **на primary** (не отставшую реплику) и **не возвращает
  `staleness=stale`** (гарантированно актуально). Доступно для read-методов и
  `ResolvePhone`. Стоимость — та же, что ожидание на primary (p99, spec/08).
- По умолчанию (`default`) чтение идёт на ближайшую реплику (spec/08 geo-affinity)
  с обязательной stale-пометкой; UI отображает баннер (spec/07; ADR-12 «не сообщать
  свежее при возможной устарелости»).

## Поле отсталости (stale-индикация, spec/04; impl-2 resolved)
- **Транспорт закреплён, а не «или»:** в gRPC — **отдельное поле `staleness`** в
  каждом read-сообщении ответа (`Staleness.Known` = `oneof { current | seconds }`,
  R5-M6); в HTTP (grpc-gateway)
  эти же данные дублируются **заголовком `X-Lag-Seconds`** и полем в JSON-ответе.
  Клиент web-UI обязан читать это поле/заголовок и показывать баннер (spec/07).
- **Требование (ADR-6/НФ):** API не должен молча возвращать устаревшие данные как
  «свежие»; `current` = отставание < `T_stale`, иначе `stale` (spec/04, ADR-12).

## HTTPS / TLS, наблюдаемость
- Шифрование передачи (TLS) — обязательная граница для API (НФ, spec/08).
- Каждой RPC — трассируемый идентификатор для отладки (НФ, spec/08).