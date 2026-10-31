# egm-review-loop — FindingsList раунда 5 (v0.6 → v0.7)

Ревью 4 свежих рыб «против v0.6» в пустых контекстах: `design_critic`,
`structural`, `implementation_readiness`, `zoom_out`. Раунд не чистой: выявлены
реальные контрактные блокеры (в первую очередь implementation_readiness) и ряд
мажоров/структурных дефектов. **Счётчик чистых раундов остаётся 0.** Находки
классифицированы, дедуплицированы, сверены с файлами; применяются как **v0.7**.

## БЛОКЕРЫ (apply-fix, контрактные)

| ID | Sev | Находка | Правка (v0.7) |
|---|---|---|---|
| R5-B1 | 🔴 init→Aborted | Два разных логических кода (`DUPLICATE_PHONE`, `DUPLICATE_SUBJECT`) мапятся в один gRPC-статус `AlreadyExists`; на проводе нет типизированного признака — UI (spec/07) не может различить програмно без парсинга сообщения | Обязательный типизированный detail к каждому коду (канонический токен). spec/05 «Канонический список» + proto-комментарий + spec/07 |
| R5-B2 | 🔴 | `WhoamiResponse` не может представить непривязанный сценарий bootstrap (нет raw subject и признака bound) — impl-20/design-22 выполним | proto `WhoamiResponse` → `{bound, subject, employee_id?, full_name?, is_admin?}`; spec/05 bootstrap |
| R5-B3 | 🔴 | Семантика update неоднозначна: у `Employee` `update_mask` c неопределённым sentinel; у `Contact` маски нет, частичное-обновление implicit | Единое правило обоих: поле-маска; masked поле = значение как на проводе (пустое = очистка); unmasked = без изменений. `update_mask` добавлен в `UpdateContactRequest`. spec/05+proto |
| R5-B4 | 🔴 | lifecycle `external_subject` не задан: not меняемый/очистка при живом, судьба при tombstone | Привязка **иммутабельна при живой записи**; освобождается при tombstone для новой записи; очистка через API запрещена (ломка scoping I5). spec/05+spec/03 |
| R5-B5 | 🔴 | Переиспользование номера «большая version» — нет монотонного начала: version per-record стартует с 1, LSN-порядок не гарантирует роста сверх tombstone | Новая живая запись по ключу получает `version = tombstone.version + 1` (продолжение версионной линии области ключа) → монотонный рост сверх удалённой инкарнации, реплики не «воскрешают». spec/03 + spec/02 |

## МАЖОРЫ (apply-fix)

| ID | Sev | Находка | Правка (v0.7) |
|---|---|---|---|
| R5-M1 | 🟡 design_critic | `external_subject`/`is_admin` отдаются в каждом read `Employee` всем читающим каталог — PII-вектор (subject ≈ email/имя) | `external_subject` возвращается **только админу каталога**; остальным — unset. spec/05 + proto-комментарий |
| R5-M2 | 🟡 design_critic | Не задано, на каком *виде данных* проверяется авторизация/разрешение principal: stale-реплика даёт асимметрию (снятый админ ещё пишет / новый ещё залочен), bootstrap не стартует | **Авторизация и principal→employee_id всегда по primary/strong-виду**, никогда по реплике; `Whoami` и resolution — strong-read по контракту. spec/05 |
| R5-M3 | 🟡 design_critic | Жизненный цикл удалённого сотрудника: осиротевшие private-книги, судьба subject/is_admin при tombstone не заданы | Инвариант: tombstone освобождает phone и subject; осиротевшие book-строки сохраняются (политика retention/GC — вне ядра); owner_id не переназначается. spec/05 + спецификация версий spec/02 |
| R5-M4 | 🟡 design_critic | Эффективный RPO противоречив по величине: spec/08 `T_stale ≤ 5с`, а ack-loss добавляет 2×`T_lease`≈60с; оператор бюджетирует 5с, обязательство ~65с | Явная цифра **RPO_eff = T_stale + 2·T_lease ≈ 65 с** (ack-член доминирует), сверено с NFR; синхронизировать spec/04+spec/08 |
| R5-M5 | 🟡 structural | spec/05 «Сообщения»: краткие имена `id`/`dept` не совпадают с proto `employee_id`/`department` (сводка ≠ источник истины struct-6) | Сводка spec/05 приведена к имени proto: `employee_id`, `department`, `contact_id` |
| R5-M6 | 🟡 structural+design | spec/05 проза `oneof {bool current; int64 seconds}` ≠ proto `Staleness.Known{current; optional seconds}` (структура/семантика) | proto → `oneof state { bool current; int64 seconds; }` внутри `Known`; spec/05 проза синхронизирована |
| R5-M7 | 🟡 structural | Порядок сброса/та гонка имени `mode` (proto/REST) vs `match_mode` (prose) — bridge-заметка только в proto, spec/05/01 не видят | Канон `match_mode`; bridge-заметка продублирована в spec/05 «Типы полей» и spec/01 ПУ-2 |
| R5-M8 | 🟡 structural | ID `design-17` переиспользован: «чтение с primary» (spec/05:194, proto ReadConsistency) и «CAS external_subject» (spec/03) | Strong-read labelled `design-16` (spec/05 + proto-комментарий); spec/03 CAS остаётся design-17 |
| R5-M9 | 🟡 impl | Поведение истёкшего/невалидного `page_token` не задано | Истёкший/невалидный курсор → `VALIDATION_ERROR` (клиент сбрасывает курIER), не silent-reset. spec/06 |
| R5-M10 | 🟡 structural | spec/08:75 битый bold (`.**`) — ломаный markdown в нормативном тексте | Убран хвост `**` |
| R5-M11 | 🟡 impl | `Staleness` на изменяющих response (идут на primary) семантически не определён | Зафиксировано: на mutating-ответах всегда `current`/absent (не stale). spec/05 |
| R5-M12 | 🟡 impl | `Delete…` `expected_version` absent не определён | absent → удалить последнюю; несовпадение → `VERSION_CONFLICT`. spec/05 |

## НИТЫ / опечатки (apply-fix)

| ID | Sev | Правка |
|---|---|---|
| R5-N1 | 🟢 | spec/08:74 `лгом` → `лагом` |
| R5-N2 | 🟢 | spec/04:174 `снего` → `своего` |
| R5-N3 | 🟢 | concepts_brief:91 `ADR-7+` → `ADR-15` |
| R5-N4 | 🟢 | REST enum `exact\|prefix` vs proto `MATCH_EXACT/PREFIX` — bridge-примечание в spec/05 (семантический мост) |
| R5-N5 | 🟢 | `Staleness`/ReadConsistency повторная разметка annex в каждом DTO — дублируется R5-M корня | (описано в CONTEST R5-C1) |

## CONTEST / РЕШЕНИЯ (не применяю сам; вынести пользователю — затронуты устоявшиеся ADR)

| ID | Sev | Находка (zoom_out / design_critic) | Почему не применяю сам |
|---|---|---|---|
| R5-C1 | 🟡 zoom | Annex проник в каждый API-DTO и proto (staleness/read_consistency в 15+ сообщениях); предлагается физически карантинить annex в один пакет, снять `read_consistency=strong` с per-RPC-поверхности | Устоявшийся контракт (ReadConsistency/Staleness принять в раундах 1–4); компромисирует одно-дц конфигурацию. Решение = отказ от части API-контракта. Требует согласия |
| R5-C2 | 🟡 zoom | Epoch-регулятор = *вторая* система консенсуса (raft-кворум+lease) поверх данных-raft; предлагается переиспользовать leader-lease raft-библиотеки или операторский fence | Противоречит ADR-11 (закреплён кворум+lease с fail-closed и 2×skew); смена механики fencing = новый ADR. Глубокая перестройка annex |
| R5-C3 | 🟡 zoom | Числовые микро-бюджеты (FPR≤1%, ≤8ГБ, amp≤15×) и group-commit — преждевременная конкретизация контракта до кода | Осознанно как бюджеты-ориентиры с оговоркой «калибруются бенчмаркингом»; «by-cursor» флаг уже optional. Демотивация до «калибровочных заметок» = пересмотр НФ-контракта |
| R5-C4 | 🟡 zoom | `client_request_id` — персистентная идемпотентность = новый подсистема без доказанной потребности (retry покрыт I2/I3/version) | Введён как контракт (spec/05 impl-…); удаление требует решения по idempotency-политике. Частично уже покрыт R5-M (replay-семантика) |
| R5-C5 | 🟡 zoom/design | Свести subject-binding к `owner_id = principal` напрямую, отложить external_subject/DUPLICATE_SUBJECT/bootstrap | Противоречит impl-20/design-22 и I5-scoping, принятым в раундах 3–4; это ре-архитектура identity. Открыть отдельным решением |

## Итог раунда 5
- **5 блокеров контракта** (ошибки-типизация, Whoami-bootstrap, update-семантика,
  subject-lifecycle, version-монотонность при переиспользовании номера) — реальны и
  применяются в **v0.7**.
- 12 мажоров + 5 нитов — применяются в v0.7.
- 5 контестных ре-архитектурных предложений (annex-quarantine, замена epoch-регулятора
  leader-lease, демотивация бюджетов, отказ от client_request_id, сдвиг identity) —
  **не применяются самостоятельно**, выносятся на решение (они затрагивают принятые ADR).
- **Чистых раундов по-прежнему 0** (раунд 5 имел блокеры).

## РЕШЕНИЕ ПО СХОДИМОСТИ (от заказчика, раунд 5)
- **Останов на v0.7** при исчерпанном капе 5.
- Формальный критерий `converged` (2 чистых раунда подряд) не достигнут — **все
  блокеры/мажоры раунда 5 устранены в v0.7**, ядровая спецификация конвергентна.
  Статус фиксируется с **пометкой о капе**: «чистых раундов = 0, кап исчерпан,
  блокеры устранены; v0.7 — точка останова».
- **CONTEST C1–C5 → отложенный backlog**: зафиксированы как «требуют новых ADR»
  (ре-архитектура annex-карантина, замены epoch-регулятора, демотивации бюджетов,
  отказа от client_request_id, сдвига identity). Ядро от них не зависит.