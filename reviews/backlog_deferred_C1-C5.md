# Отложенный backlog решений (CONTEST C1–C5, раунд 5)

Статус: **отложены** (решение заказчика, раунд 5). Это re-архитектурные
предложения zoom_out/design_critic, **не применённые в v0.7**, т.к. затрагивают
принятые ADR. Ядро от них не зависит; активация = новый ADR + целевой раунд.

| ID | Предложение | Затрагивает | Условие активации (новый ADR) |
|---|---|---|---|
| C1 | Физически карантинить распределённый annex в один пакет; снять `read_consistency=strong` и `Staleness` с per-RPC-поверхности, оставить single strong-read-path | API-контракт (ReadConsistency/Staleness принять в раундах 1–4), proto (`Staleness` в 15+ сообщениях) | Когда annex переводится из добровольного в обязательный/поддерживаемый профиль |
| C2 | Заменить epoch-регулятор (Raft-кворум+lease, ADR-11) на leader-lease уже выбранной raft-библиотеки или операторский fence | ADR-11 (fencing-кворум+lease, fail-closed, 2×skew), spec/04 | Если выбранная raft-библиотека даёт доказуемые leader-lease и упрощается безопасность fencing |
| C3 | Демотировать числовые микро-бюджеты (FPR≤1%, ≤8 ГБ, amp≤15×) и write-RPS≥200 из контракта в «калибровочные заметки» до benchmark | spec/08 бюджеты, spec/03 group-commit | На этапе benchmark, когда появятся реальные измерения |
| C4 | Отказаться от персистентной идемпотентности `client_request_id`; полагаться на естественные invariants (I2/I3, expected_version) | spec/05 idempotency, proto (create/update/delete request) | Появление конкретного at-least-once требования / real retry case |
| C5 | Свести scoping к `owner_id = principal цифру` напрямую; отложить `external_subject`/`DUPLICATE_SUBJECT`/bootstrap | spec/05 auth, impl-20/design-22, I5-scoping, ADR-3/5 | Выбор реального identity-бэкенда за пределами курсового контракта |

Каждая запись — кандидат на собственный ADR (ADR-16+) при принятии; отмечает
исходную мотивацию (zoom_out: YAGNI/минимизация; design_critic: risk) и ожидаемое
снижение сложности. Пока не активированы, спецификация v0.7 остаётся валидной.