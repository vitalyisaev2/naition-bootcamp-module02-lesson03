# egm-review-loop — FindingsList раунда 3 (v0.4 → v0.5)

Конфигурация та же. Раунд 3 подтвердил закрытие раундов 1–2 и выявил блокеры в
ядре нормализации + контрактный недокомплект proto + рассинхрон ADR-11. Все
блокеры/мажоры переведены в propose-fix и применены в **v0.5**.

## FIXED в v0.5

| ID | Severity | Находка | Правка |
|---|---|---|---|
| design-1 🔴 | 🔴 | Нормализация некорректно обрабатывает международный ввод без `+` для 375/380/49 (ложный canon → раздубль I2/I3, ложнопустой поиск) | Раздел «Нормализация» в spec/02: ядро = **один регион `+7`**; иной `+`-код → VALIDATION_ERROR; № без `+` разбирается детерминированно; тихий canon не строится (контент ранее в `spec/02-normalization.md`, влит в spec/02) |
| design-4 / zoom-2 | 🟡 | Лидирующие коды `8`/`0` неоднозначны (BY/UA/DE) — multi-таблица раздута (YAGNI) | Тот же — multi-коды вынесены из ядра (ADR-15); один лидирующий `8/7→+7` устраняет неоднозначность |
| struct-1 / design-2 🔴 | 🔴 | ADR-11 «единоличный», spec/04 «Raft-кворум 3+»; «ADR-11 обновлён» ложна | ADR-11 «Решение» синхронизирован (кворум, lease T_lease=30с, сверка при продлении, fail-closed, 2×skew); добавлен ADR-15 (нормализация = один регион) |
| impl-18 🔴 | 🔴 | 13/19 сообщений proto без тел (Get/Update/Delete/Search/Whoami) | `spec/contacts.v1.proto` v0.5: полные тела всех сообщений, Update field-mask, expected_version, read_consistency/staleness на read |
| impl-19 🔴 | 🔴 | «expected_version не указан» vs 0 неразличимы; старт счетчика не задан | `expected_version` = `optional int64`; version стартует с 1, 0 зарезервирован (spec/05, proto) |
| impl-20 🔴 | 🔴 | Нет пути создания привязки principal→employee_id (scoping I5) | spec/05 + proto: поле `external_subject` в Employee/CreateEmployee; индекс external_subject→employee_id; DUPLICATE_SUBJECT; роль admin |
| impl-24 / zoom-3 | 🟡 | read_consistency не на всех read; distributed-поля вшиты в ядро как обязательные | proto: `read_consistency` на Get/Search*; `staleness`/`ReadConsistency` — annex/optional (single-DC без них) |
| impl-23 | 🟡 | Курсор ResolvePhone по catalog+mine недоопределён | spec/06: сквозной единый курсор по объединённому ключу нормализованной формы |
| struct-3 | 🟡 | spec `match_mode`/`read_consistency` vs proto `mode`/`consistency` | proto — `mode`/`consistency`, пометка-мост «spec↔proto имена» (struct-3) |
| struct-4 | 🟢 | REST каталога SearchByPhone без mode | таблица: `&mode={exact|prefix}` добавлен |
| struct-2 | 🟡 | contacts.v1.proto осиротел (нет ссылок) | spec/05 «Формальный proto-контракт» + индекс spec/00 (normalization+proto) |
| impl-21 | 🟡 | Мин. длина номера не задана | normalization п.4: нац. часть ≥ 7 |
| impl-22 / design-5 | 🟡 | clock-skew/период продления lease не квантифицированы; RTO без сметы | spec/04 + spec/08: skew ≤ 5с (NTP), продление T_lease/3≈10с, смета RTO = детект+fence+кворум+догон (≤5 мин) |
| impl-25 | 🟡 | write/read-RPS «напр.» — не цели | spec/08: **write-RPS ≥ 200, read-RPS ≥ 5000 закреплены** как первичные таргеты |
| design-3 | 🟡 | Нет критерия выбора промоут-реплики по лагу ≤ RPO | spec/04: промоут реплики с мин. лагом и лгом ≤ RPO; иначе блокировка/оператор |
| struct-5/6/7, impl-26/27, zoom-1 | 🟢 | Маркеры ревью в теле, дубль списка ошибок, «write-стал», deleted, clamp limit, аннекс объёмнее ядра | Вычищены висячие маркеры M8/B4; единый список ошибок; write-stall; deleted — вне read (proto); limit→clamp; spec/04 scope-пометка «annex-резюме» |
| design-6 | 🟢 | Нормализация без Alternatives (E.164/libphonenumber) | ADR-15: отклонены (избыточны для ядра, ADR-2); добавлены как «если понадобится» |

## Scope: ядро vs аннекс (zoom-1/zoom-3)
- Аннекс (spec/04) остаётся подробнее ядра по объёму, но это **уровневый документ**;
  в шапке прокинута пометка «annex — резюме; читается по ADR-11; при single-DC не
  контракт». Distributed-поля в API/proto помечены **optional/annex** — ядро
  собирается без них. Нормализация сокращена до одного региона (снято ~50 строк
  спекулятивной таблицы).

## Итог раунда 3
- Раунды 1–2 подтверждены.
- Раунд 3 блокеры (нормализация, ADR-11, proto-недокомплект, expected_version,
  principal→employee_id) — устранены в v0.5 (ADR-13…15 → ADR-15).
- Для «2 чистых раунда»: раунд 4 должен пройти без новых блокеров (ожидание —
  только ниты/уточнения), дав первый чистый раунд; раунд 5 — подтверждение second
  clean раундом и converged.

## Следующий шаг
Раунд 4 (свежие рыбки против v0.5) → цель: чистый раунд без блокеров.