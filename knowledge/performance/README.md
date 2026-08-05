# Bitrix24 Performance Engineering Knowledge Base

Инженерная база знаний для диагностики, масштабирования и безопасной оптимизации решений Bitrix24. Производительность — свойство всей цепочки, а не одного компонента.

> **Статус:** факты основаны только на `knowledge/SOURCES.md`. Точные лимиты, SLA, мощности тарифов и конфигурации серверов не приводятся. Любые значения определяются официальной документацией, измерением конкретного портала и согласованным SLO.

> **Безопасность:** нагрузочные и массовые проверки выполняются только на согласованном тестовом контуре с синтетическими данными. Неконтролируемое тестирование production запрещено.

## 1. Модель производительности Bitrix24

| Слой | Сигнал | Типичный риск |
|---|---|---|
| UI/CRM | Load/interaction latency | Поля, фильтры, объём, сеть |
| Robots/BP/Smart Process | Queue/execution time | Каскады, циклы, ожидания |
| REST/webhooks/apps | Latency/errors/lag | Limits, burst, retries |
| Telephony/BI/files | Route/report/storage delay | Provider, query, volume |
| Box infrastructure | CPU/RAM/disk/DB/web/PHP/cache | Capacity/config/custom code |

```text
Пользователь → Bitrix24 → Автоматизация → Интеграция → Внешняя система
```

End-to-end latency измеряется по границам; ускорение одного звена не гарантирует результат.

## 2. Cloud и On-Premise

| Зона | Cloud | On-Premise | Контроль клиента | Комментарий |
|---|---|---|---|---|
| Platform | Managed | Server/OS/DB/web/PHP owned | Только Box | Не придумывать внутренности Cloud |
| CRM/automation/apps | Config/tariff dependent | Config/version dependent | Оба | Измерять сценарий |
| REST/webhooks/BI | Published boundary | Module/version boundary | Integration side | Limits verify |
| Cache/network/cron/queue | Ограниченный контроль | Owner control | Box | Monitoring required |

```text
Cloud: optimize configuration + integration
Box: optimize configuration + integration + infrastructure + custom code
```

## 3. Методика диагностики

1. Зафиксировать симптом, пользователя, время и бизнес-эффект.
2. Локализовать UI/CRM/automation/REST/external/network/infrastructure.
3. Воспроизвести безопасно.
4. Измерить baseline по границам.
5. Найти bottleneck.
6. Сформировать одну проверяемую гипотезу.
7. Проверить на малой выборке.
8. Внести минимальное изменение.
9. Повторно измерить тем же сценарием.
10. Задокументировать факт, результат и rollback.

```text
Symptom → scope → evidence → hypothesis → experiment → compare → decision
```

## 4. Производительность CRM

| Фактор | Влияние | Действие |
|---|---|---|
| Лиды/сделки/контакты/компании | Lists/search/analytics volume | Archive policy, exact filters |
| Поля/множественные/обязательные | Card complexity and validation | Field inventory and reduction |
| History/comments/files/activities/tasks | Timeline/storage/loading | Retention and separation |
| Open deals/kanban | Active working set | Close/archive stale items |
| Duplicates | Search and analytics noise | Governed deduplication |
| Automation | Extra writes/tasks/events | Registry and minimization |

## 5. Пользовательские поля

| Риск | Рекомендация |
|---|---|
| Слишком много полей | Owner/consumer for each; remove only after dependency audit |
| Multiple/list/bindings/files | Use only for real semantics |
| Search/filter on free text | Structured type and normalized dictionary |
| Mass fill | Dry-run, batches, side-effect review |
| Bad inherited schema | Migration plan, not more duplicate fields |

```text
Business fact → correct type → minimal field → governed use
```

## 6. Воронки и стадии

| Фактор | Контроль |
|---|---|
| Directions/stages | Separate only independent lifecycles |
| Long/duplicate stages | Merge by business meaning |
| Robots per stage | Count side effects and cycles |
| Mass move | Pilot, dry-run, automation impact |
| Archive deals | Preserve history/analytics |

## 7. Роботы

Автоматизация должна быть минимальной, наблюдаемой и идемпотентной.

| Риск | Контроль |
|---|---|
| Repeat/cascade | Operation key and trigger map |
| Delay/condition | Explicit SLA and fallback |
| Task/notification storm | Dedup and useful recipient |
| Field write loop | Single owner per field |
| Webhook/BP launch | Timeout, queue, monitoring |

```text
Stage event → condition → one owned action → result/audit
```

## 8. Бизнес-процессы

| Признак перегрузки | Решение |
|---|---|
| Очень длинный монолит | Modules/subprocess contracts |
| Много branches/loops | Simplify and cap iterations |
| Паузы без timeout | SLA/escalation |
| Массовый старт | External toolkit/queue |
| External sync call | Timeout, retry, idempotency |
| Зависшие экземпляры | Journal, owner, recovery runbook |

## 9. Smart Processes

| Фактор | Контроль |
|---|---|
| Items/categories/stages | Working-set and lifecycle design |
| Fields/relations | Minimal normalized schema |
| Robots/BP | Side-effect registry |
| REST/filter/bulk | Select/pagination/dry-run |
| Analytics/archive | Preserve keys and stage history |

```text
Entity type → active set → lifecycle → archive while preserving relations
```

## 10. REST API

| Область | Практика |
|---|---|
| Frequency/parallelism | Per-tenant controlled workers |
| Pagination | Follow actual cursor/`next` |
| Filter/order/select | Exact, stable, minimal |
| Timeout/retry | Bounded, classified, exponential backoff+jitter |
| Cache | Metadata with TTL/invalidation |
| Incremental | Modified date + ID tie-breaker + checkpoint |
| Idempotency | External/operation ID and reconciliation |

Актуальные ограничения: [официальный REST API](https://apidocs.bitrix24.ru/).

## 11. Batch

| Использовать | Не использовать |
|---|---|
| Независимые короткие команды | Нужна общая транзакция |
| Управляемые зависимости | Тяжёлые files/payload или сложные branches |

Каждая команда имеет статус; batch size определяется официальным контрактом и тестом. Partial result → targeted retry/compensation, не полный слепой повтор.

```text
Plan → batch → per-command result → checkpoint → retry/compensate
```

## 12. Пагинация

`start`, `next`, `total` применяются только по контракту выбранной версии; `total` не заменяет cursor.

| Риск | Контроль |
|---|---|
| Data changes during scan | Stable order, overlap window |
| Missing/duplicates | ID tie-breaker and dedup |
| Restart | Durable checkpoint |
| Date filter | Timezone and inclusive boundary policy |

## 13. Webhooks и события

| Фактор | Контроль |
|---|---|
| Burst | Fast receiver + durable queue |
| Re-delivery | Dedup/idempotency |
| Debounce | Only if business semantics allow |
| Ack | After durable enqueue |
| Failure | Retry queue, DLQ, manual retry |
| Lag | Queue age and last-event monitoring |

```text
Event → receiver → queue → dedup → worker → REST/reconcile
```

## 14. Очереди

| Компонент | Назначение |
|---|---|
| Producer | Validate/publish |
| Consumer | Idempotent processing |
| Retry/DLQ | Temporary/permanent separation |
| Priority/order | Only where required |
| Concurrency/backpressure | Protect dependencies |
| Recovery | Replay from checkpoint |

Технология брокера не фиксируется без требований.

## 15. Интеграции

| Категория | Объём/частота/SLA | Degradation policy |
|---|---|---|
| 1С | Catalog/orders/payments; config-specific | Queue, incremental, reconcile |
| Site | Leads/orders; burst | Accept safely, throttle downstream |
| Telephony/messengers | Event bursts/provider SLA | Queue/fallback |
| Payment | Low tolerance for duplicates | Idempotency and status check |
| BI | Large reads/scheduled | Incremental marts |
| External API/middleware | Contract-specific | Circuit breaker and monitoring |

Master system, retries and SLO are defined per flow, never universal.

## 16. Интеграция с 1С

| Объект | Performance rule |
|---|---|
| Large catalogs/products/prices/stocks | Incremental by stable ID/version |
| Orders/payments/docs | Idempotent status exchange |
| Full exchange | Exceptional, planned and reconciled |
| Locks/errors | Short transactions and safe journal |
| Windows | Business-approved; night is not automatically safe |
| Test/prod | Separate bases and credentials |

Конфигурации 1С различаются; механизм **требует проверки**.

## 17. Телефония

| Сценарий | Performance risk/control |
|---|---|
| Incoming/outgoing | Provider/route latency and capacity |
| CRM lookup/lead create | Normalized phone and dedup |
| Recording | Storage/bandwidth/retention |
| External PBX | Provider outage/fallback |
| Mass calls | Only confirmed mechanism and capacity plan |
| Operator queue | Wait/overflow monitoring |

Источник подключения: [телефония Bitrix24](https://helpdesk.bitrix24.ru/open/6450911/).

## 18. BI и аналитика

| Фактор | Оптимизация |
|---|---|
| Dataset volume | Minimal grain/fields and retention |
| Heavy SQL/filter/join | Query review and aggregate mart |
| Refresh | Business-appropriate schedule |
| External BI/Power BI | Separate licensed architecture |
| Cache/precompute | Governed TTL and materialization |
| Quality | Prevent expensive wrong reports |

Источник: [BI datasets](https://helpdesk.bitrix24.ru/open/24355590/).

## 19. Файлы и диск

| Риск | Контроль |
|---|---|
| Large/many files and versions | Storage inventory and retention |
| Shared folders/CRM docs | Rights and working-set design |
| Call recordings | Separate retention policy |
| Export/backups/download | Controlled off-platform storage |
| Delete/archive | Preserve references and recovery |

## 20. Поиск и фильтрация

| Поле/операция | Рекомендация |
|---|---|
| Standard/indexed behavior | Use exact documented filters |
| Multiple/text/bindings | Avoid as universal search key |
| Dates | Bounded ranges and timezone |
| Sort | Stable key |
| Bulk/duplicates | Narrow candidate set, staged processing |

Индексация Cloud не управляется клиентом; Box-изменения требуют официального анализа и теста обновлений.

## 21. Архивирование

| Объект | Правило |
|---|---|
| Closed deals/old leads/tasks/process items | Lifecycle-based archive, preserve analytics |
| Files/logs/recordings | Retention, rights and deletion verification |
| Client exports | Local protected storage and expiry |
| Dictionaries | Version, do not break historical labels |

```text
Active working set → archive policy → historical analytics/recovery
```

## 22. Массовые операции

```text
Scope → backup/export → dry-run → small sample → approve → batches
      → per-item report → reconcile → rollback/compensate
```

Fields, stages, import/export, BP starts, deletes, tasks and relation updates require error classification and side-effect review. Delete is prohibited by default.

## 23. Идемпотентность

| Mechanism | Purpose |
|---|---|
| External ID | Object identity |
| Operation/idempotency key | Attempt identity |
| Upsert/dedup | Repeat-safe mutation |
| Partial-state journal | Resume/compensate |
| Reconciliation | Resolve unknown outcome |

## 24. Кэширование

| Cache candidate | Rule |
|---|---|
| Dictionaries/metadata/fields/stages | TTL + schema/version invalidation |
| Users/permissions | Short TTL; never assume permanent rights |
| Frequent results | Only if business staleness accepted |

Устаревший cache может нарушить данные и права; master остаётся источником истины.

## 25. Инкрементальная синхронизация

```text
checkpoint → overlap window → modified filter + stable ID → dedup
           → apply → save checkpoint → reconcile → periodic full resync
```

Удаления требуют отдельной стратегии: событие, tombstone, archive check или периодическая сверка — только по доступному контракту.

## 26. Наблюдаемость

| Signal | Examples |
|---|---|
| Logs | Structured, correlation/operation ID, safe metadata |
| Metrics | latency, throughput, error/retry, queue length, lag |
| Dashboard/alerts | SLO breach, stale checkpoint, DLQ growth |
| Journal | Object, direction, result, attempt |

Конкретный monitoring tool не считается штатно поддерживаемым без источника.

## 27. Метрики производительности

| Метрика | Что показывает | Где измерять | Порог определяет | Действие |
|---|---|---|---|---|
| Response/card load | User latency | Browser+boundary | Business SLO/baseline | Localize layer |
| BP duration | Automation latency | Process journal | Process owner | Find wait/activity |
| REST latency/errors | API health | Client metrics | Integration SLO/docs | Backoff/optimize |
| Queue length/lag/retries | Backlog | Broker/worker | Capacity SLO | Scale/throttle/fix |
| Records throughput | Processing rate | Pipeline | Volume plan | Tune batches |
| Report build | BI latency | BI | Audience SLO | Model/query optimize |
| Import/export time | Bulk duration | Toolkit | Change window | Batch/checkpoint |

## 28. Нагрузочное тестирование

| Этап | Контроль |
|---|---|
| Goal/scenarios/data | Business journey, synthetic volume |
| Environment | Isolated and representative |
| Ramp/steady/spike | Planned profile, no universal numbers |
| Stop criteria | Errors, safety, dependency impact |
| Recovery | Backlog drain and correctness |

```text
Baseline → ramp-up → steady → spike → recovery → compare/reconcile
```

## 29. Тестирование отказов

| Fault | Expected behavior |
|---|---|
| Bitrix24/external unavailable | Queue and bounded retry |
| Timeout/partial | Unknown state + reconcile |
| Auth error | Stop/re-auth/manual |
| Limit response | Backoff/reduce concurrency |
| Queue full/corrupt message | Backpressure/DLQ |
| Connection loss/recovery | Resume checkpoint without duplicates |

## 30. Оптимизация коробочной версии

| Layer | Architecture review |
|---|---|
| CPU/RAM/disk/network | Measured saturation and capacity |
| DB | Query/profile/maintenance by qualified owner |
| Web/PHP/cache | Supported configuration and profiling |
| Agents/cron/queues | Schedule, overlap, failures |
| Custom code | Hot paths, external calls, memory |
| Updates/monitoring | Stage, backup, regression, dashboards |

Никаких универсальных server specs. Источник: [On-Premise docs](https://dev.1c-bitrix.ru/docs/portal.php).

## 31. Кастомный код

| Риск | Контроль |
|---|---|
| REST N+1 | Batch/prefetch/cache where valid |
| Event handler/agent/cron | Short work, queue heavy tasks |
| SQL/DB | Supported parameterized access; no unsafe direct writes |
| Blocking external call | Timeout/async/circuit breaker |
| Memory/library | Profiling, dependency review |
| Logging | Sampling/safe structured logs |

## 32. Управление изменениями

```text
Baseline → hypothesis → test → canary if applicable → rollout
         → same measurement → accept/rollback → document
```

Canary применим только если архитектура позволяет безопасно изолировать группу.

## 33. Capacity Planning

| Driver | Plan |
|---|---|
| Users/CRM/files/calls | Current, growth, retention |
| Integrations/reports | Frequency, peaks, concurrency |
| Seasonality | Peak calendar and fallback |
| Headroom | Risk-based, measured, budgeted |
| Ownership | Platform, integration, business owners |

## 34. Антипаттерны

| Антипаттерн | Замена |
|---|---|
| Sync everything with everything | Master/field ownership |
| Full sync each run | Incremental+reconcile |
| Select all fields/poll endlessly | Minimal select/event+poll hybrid |
| No queue/cache/checkpoint/retry/timeout/monitoring | Reliability baseline |
| Giant BP/robot for every action | Minimal modular automation |
| Bulk start without dry-run | Staged safe operation |
| All data in one Smart Process | Proper domain model |
| All reports real-time | Business refresh SLA |
| Files instead of structured data | Typed fields/relations |

## 35. Типовые ошибки

| № | Ошибка | Последствие |
|---:|---|---|
| 1 | Нет измерений | Мнение вместо факта |
| 2 | Оптимизация без baseline | Нельзя доказать эффект |
| 3 | Нет журналирования | Нет диагностики |
| 4 | Нет pagination | Пропуски |
| 5 | Неверный batch | Partial loss |
| 6 | Excess parallelism | Throttling |
| 7 | Нет backoff | Retry storm |
| 8 | Full import всегда | Нагрузка |
| 9 | Нет external ID | Дубли |
| 10 | Нет idempotency | Повторные записи |
| 11 | Нет queue | Потеря burst |
| 12 | Нет DLQ | Потеря ошибок |
| 13 | Нет dry-run | Массовая порча |
| 14 | Нет rollback | Долгий сбой |
| 15 | Тяжёлые фильтры | Медленные списки |
| 16 | Слишком много полей | Медленная карточка |
| 17 | Слишком много роботов | Каскад |
| 18 | Длинный BP | Зависания |
| 19 | Реальные данные в load test | Утечка |
| 20 | Load test production | Инцидент |
| 21 | Нет контроля external system | End-to-end деградация |
| 22 | Симптом не локализован | Неверный fix |
| 23 | Сеть не измерена | Ошибка диагноза |
| 24 | Browser cache принят за решение | Скрытый bottleneck |
| 25 | Дубли не очищены | Search/analytics noise |
| 26 | Open deals не управляются | Большой working set |
| 27 | Stages дублируются | Сложность |
| 28 | Robot loop | Бесконечные записи |
| 29 | Task storm | Перегрузка |
| 30 | Notification storm | Шум |
| 31 | External call sync in BP | Blocking |
| 32 | Pause без SLA | Hanging instance |
| 33 | EntityType mapping hardcoded | Cross-portal errors |
| 34 | `SELECT *` | Load/PII |
| 35 | Unstable sort | Missing/duplicates |
| 36 | `total` controls pagination | Incomplete scan |
| 37 | Нет checkpoint | Restart from zero |
| 38 | Нет overlap window | Missed updates |
| 39 | Timeout = failure | Duplicate write |
| 40 | Batch = transaction | Wrong rollback |
| 41 | Full retry partial batch | Duplicates |
| 42 | Ack before enqueue | Event loss |
| 43 | Нет dedup events | Repeat action |
| 44 | Unlimited queue consumers | Dependency overload |
| 45 | Нет backpressure | Collapse |
| 46 | Cache без TTL | Stale data |
| 47 | Permissions cached forever | Access risk |
| 48 | Night load assumed safe | User/automation conflict |
| 49 | BI on operational grain always | Slow reports |
| 50 | Cartesian join | Data explosion |
| 51 | Files never archived | Storage growth |
| 52 | Call records unlimited | Storage/privacy risk |
| 53 | Delete used as optimization | Data loss |
| 54 | Direct DB tuning without review | Upgrade/integrity risk |
| 55 | Core modified | Unsupported system |
| 56 | Cron overlaps | Duplicate work |
| 57 | No memory profiling | Worker crash |
| 58 | N+1 API/SQL | Latency explosion |
| 59 | No capacity plan | Surprise saturation |
| 60 | No SLO owner | No action |
| 61 | Metric threshold copied universally | Wrong alerts |
| 62 | Change not documented | Regression |
| 63 | No post-change measurement | Unknown result |
| 64 | No recovery test | False assurance |
| 65 | Cloud internals assumed | Invalid architecture |

## 36. Чек-лист Performance Architect

- [ ] Symptom/scope/baseline/metrics fixed.
- [ ] CRM fields/stages/robots/BP/Smart Processes inventoried.
- [ ] REST/webhooks/queues/apps/integrations measured.
- [ ] 1С/telephony/BI/files/cache/sync dependencies mapped.
- [ ] Monitoring, load/failure tests, rollback and docs ready.

## 37. Чек-лист аудита производительности

| Проверка | Evidence |
|---|---|
| Complaints/cards/filters | Reproducible timings |
| Automation/BP/apps | Registry, counts, errors |
| REST/integrations/queues | Latency, throughput, lag |
| BI/files/duplicates/archive | Volume and retention |
| Box infrastructure | CPU/RAM/disk/DB/web/PHP/cron metrics |
| Monitoring | Dashboards, alerts, ownership |

## 38. Практические сценарии

| № | Сценарий | Симптом/гипотеза | Диагностика/безопасное действие | Критерий |
|---:|---|---|---|---|
| 1 | Медленная сделка | Fields/timeline/apps/network | Layer timing, disable only in test | Same scenario faster, correct |
| 2 | Массовый импорт | Volume/automation | Dry-run, small batch, checkpoint | Counts/relations reconcile |
| 3 | Отстающая 1С | Full sync/errors | Queue lag, incremental mapping | Lag returns to SLO |
| 4 | Webhook queue | Burst/slow consumer | Depth/age, throttle/scale | Backlog drains no duplicates |
| 5 | Медленный BI | Join/grain/refresh | Query/model review, mart | Same result, lower latency |
| 6 | Зависший BP | Wait/external call | Instance journal, timeout | Completes/recoverable |
| 7 | Всплеск лидов | Form burst/robots | Queue, dedup, rate control | No loss/storm |
| 8 | Массовая телефония | Provider/capacity | Approved synthetic pilot | Route stable, CRM correct |
| 9 | Большой Smart Process | Items/fields/automation | Working-set/filter audit | Stable list/card/REST |
| 10 | Рост файлов | Versions/records/exports | Inventory, retention, archive | Growth controlled, links valid |

## 39. Cloud vs On-Premise

| Контроль | Cloud | On-Premise | Можно оптимизировать | Риск |
|---|---|---|---|---|
| CRM/BP/Smart/files/BI/apps | Config/tariff | Config/version | Model, working set, automation | Unsupported assumptions |
| REST | Published limits | Module/version | Client patterns | Throttling |
| Server/DB/cache/network/cron | No direct control | Owner control | Infrastructure/code | Misconfiguration |
| Queue/monitoring | External | External/owned | Full stack | Blind spots |
| Updates/custom code | Vendor/no core | Owner/modules | Stage/profile | Regression |

## 40. Официальные источники

| Источник | Использование | Проверено |
|---|---|---|
| [REST API](https://apidocs.bitrix24.ru/) | Limits, batch, pagination, errors | 2026-08-05 |
| [API reference](https://apidocs.bitrix24.ru/api-reference/) | Contracts and fields | 2026-08-05 |
| [BI datasets](https://helpdesk.bitrix24.ru/open/24355590/) | BI data context | 2026-08-05 |
| [Telephony](https://helpdesk.bitrix24.ru/open/6450911/) | Connection context | 2026-08-05 |
| [On-Premise docs](https://dev.1c-bitrix.ru/docs/portal.php) | Box architecture | 2026-08-05 |
| [Support24](https://helpdesk.bitrix24.ru/) | Current product behavior | 2026-08-05 |

```text
Official source → portal/version measurement → hypothesis → safe test → evidence
```

Сторонние лимиты, benchmarks и server recipes не использованы.
