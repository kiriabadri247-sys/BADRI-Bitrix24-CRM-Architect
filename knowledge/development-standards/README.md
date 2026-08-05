# Bitrix24 Development Standards Knowledge Base

Инженерные стандарты разработки, интеграции и сопровождения решений Bitrix24 для Senior Bitrix24 Developer, Solution Architect, Integration Engineer и Technical Lead.

> **Статус сведений:** продуктовые факты основаны только на официальных источниках из `knowledge/SOURCES.md`. Точные методы, события, поля, OAuth-сценарии, лимиты, функции приложений и On-Premise-механизмы проверяются по официальной документации и на целевом портале. Общие инженерные практики внешнего сервиса не считаются встроенными функциями Bitrix24.

> **Нормативность:** правила с идентификаторами `STD-*` обязательны для проектного baseline. Исключение требует ADR, владельца риска, срока и компенсирующих мер.

> **Безопасность:** клиентские данные, действующие токены, webhook URL, API keys, реальные домены и необезличенные логи запрещены в Git, тестах и документации.

## 1. Принципы разработки Bitrix24

```text
Бизнес-задача -> Стандартная возможность -> Конфигурация
 -> Кастомизация -> Тестирование -> Эксплуатация и поддержка
```

| ID | Стандарт | Критерий |
|---|---|---|
| STD-001 | No-code/standard-first | Проверены CRM settings, fields, robots, BP, Smart Processes, connectors |
| STD-002 | Минимизация кода | Каждый custom component имеет обоснование |
| STD-003 | Архитектурная фиксация | Сложное решение имеет ADR и диаграмму |
| STD-004 | Secure by design | Threats, secrets, rights и ПДн рассмотрены до разработки |
| STD-005 | Testable by design | Positive, negative, repeat и recovery tests определены |
| STD-006 | Maintainability | Owner, README, changelog и support runbook существуют |
| STD-007 | Scalability evidence | Объём, рост и workload измерены; гарантии без теста запрещены |
| STD-008 | Reversibility | Deployment и массовые операции имеют rollback/compensation |

Код не нужен, если задача надёжно решается штатной настройкой с приемлемыми правами, аналитикой и сопровождением.

## 2. Cloud vs On-Premise разработка

| Область | Cloud | On-Premise | Риск |
|---|---|---|---|
| Extension boundary | REST, webhooks, apps, external services | То же + поддерживаемые модули/события | Неверная применимость |
| Server code | Внутри платформы недоступен | В поддерживаемом расширении | Изменение ядра |
| Database | Нет прямого доступа | Владелец инфраструктуры | Прямая запись повреждает модель |
| Agents/cron | Не управляются клиентом как серверные механизмы | Зависят от версии/конфигурации | Дубли/зависания |
| Infrastructure | Managed vendor boundary | Server/web/PHP/DB/files | Нет monitoring/backup |

| ID | Стандарт |
|---|---|
| STD-009 | Cloud-код выполняется во внешнем сервисе или официальном приложении через опубликованные контракты. |
| STD-010 | On-Premise-расширение не изменяет ядро и проходит тест обновления. |
| STD-011 | Прямая запись в БД не заменяет поддерживаемый прикладной контракт. |
| STD-012 | Версия продукта, PHP, модулей и инфраструктуры фиксируется в environment passport. |

## 3. Архитектура кастомных решений

Код оправдан при подтверждённом gap: сложная интеграция, алгоритм, нагрузка, внешняя транзакция, тиражируемое приложение или требование, недоступное штатно.

```text
Bitrix24 published boundary <-> App/Middleware <-> Queue/Workers
                                      <-> External systems
```

| Компонент | Когда | Ответственность |
|---|---|---|
| Внешний сервис | Cloud logic, integrations | Hosting, security, monitoring, rollback |
| Middleware | Несколько систем/contracts | Mapping, queue, identity, observability |
| Приложение | OAuth/multi-portal/UI scenario | Lifecycle, scopes, vendor support |
| On-Premise module | Поддерживаемое Box extension | Compatibility and update tests |
| Event handler | Подтверждённое событие | Idempotency, recursion guard, errors |

| ID | Стандарт |
|---|---|
| STD-013 | Customization начинается с ADR и списка отвергнутых штатных вариантов. |
| STD-014 | Компоненты разделяют business logic, Bitrix24 adapter, storage и transport. |
| STD-015 | Каждый внешний dependency имеет timeout, failure policy и owner. |

## 4. Структура проекта разработки

Рекомендуемая логическая структура; конкретные каталоги создаются только при появлении материалов:

```text
project/
  README.md
  docs/              architecture, ADR, operations
  src/               application code
  config/            safe templates, no secrets
  tests/             unit, contract, integration fixtures
  migrations/        versioned reversible changes
  scripts/           dry-run tools
  environments/      metadata, no credentials
  logs/               local/runtime only, ignored
  CHANGELOG.md
```

| ID | Стандарт |
|---|---|
| STD-016 | README содержит purpose, architecture, setup, test, deployment, monitoring и support. |
| STD-017 | Runtime logs, exports и client configuration исключаются из Git. |
| STD-018 | Configuration templates содержат placeholders, schema и validation. |
| STD-019 | Миграции версионируются, тестируются и имеют обратный план. |

## 5. Работа с REST API

Точные контракты проверяются в [официальном REST API reference](https://apidocs.bitrix24.ru/api-reference/).

| ID | Стандарт |
|---|---|
| STD-020 | Авторизация выбирается из официально поддерживаемого сценария и минимального scope. |
| STD-021 | Tokens/webhook URL хранятся в secret manager или runtime environment, не в коде. |
| STD-022 | Клиент различает transport, HTTP и API errors. |
| STD-023 | Устанавливаются connect/read/overall timeout по контексту. |
| STD-024 | Retry ограничен, применяется только к классифицированным временным ошибкам. |
| STD-025 | Pagination проходит весь набор и запрашивает минимальный `select`. |
| STD-026 | Batch проверяет результат каждой команды; атомарность не предполагается. |
| STD-027 | Write-операции имеют external/operation ID и идемпотентную стратегию. |
| STD-028 | Rate/usage контролируются без вымышленных универсальных лимитов. |
| STD-029 | Карточка метода фиксирует official URL, params, response, rights и tests. |

```text
Validate -> Authorize -> Request -> HTTP/API validation -> State verification
                         \-> classify -> retry/reconcile/manual review
```

## 6. Webhook Development

Входящий webhook — secret-bearing REST URL; исходящее событие — недоверенный input до проверки. Конкретная модель подтверждается [официальной документацией локальных интеграций](https://apidocs.bitrix24.ru/local-integrations/index.html).

| ID | Стандарт |
|---|---|
| STD-030 | Webhook имеет named owner, purpose, scope, environment и revoke procedure. |
| STD-031 | Полный URL никогда не логируется и не помещается в тикеты. |
| STD-032 | Receiver быстро validate/enqueue и не выполняет долгую логику до ответа. |
| STD-033 | Duplicate/out-of-order delivery обрабатываются идемпотентно. |
| STD-034 | Queue, retry, dead-letter и reconciliation имеют monitoring и owner. |
| STD-035 | При увольнении владельца выполняются review, rotation/revoke и ownership transfer. |

Не предполагать подпись, гарантированный порядок или retry policy без официального контракта.

## 7. OAuth Applications

| Область | Требование |
|---|---|
| Client | Зарегистрирован по официальному сценарию; owner известен |
| Redirect | Allowlist и HTTPS согласно подтверждённому контракту |
| Tokens | Encrypted at rest; masked in logs; lifecycle handled |
| Refresh | Только официальный flow, errors и revoke обработаны |
| Permissions | Least privilege и periodic review |
| Environments | Отдельные registrations/credentials, если применимо |
| Production | Monitoring, support, rotation и incident runbook |
| Testing | Test portal/data; negative auth cases |

| ID | Стандарт |
|---|---|
| STD-036 | OAuth flow, redirect rules и token lifecycle не придумываются и сверяются официально. |
| STD-037 | Development, test и production не используют общий secret. |
| STD-038 | Изменение scopes требует security review и regression. |

## 8. Код-стайл

| ID | Стандарт |
|---|---|
| STD-039 | Имена отражают domain intent; сокращения документируются. |
| STD-040 | Функция выполняет одну ответственность и имеет ограниченную сложность. |
| STD-041 | Business logic не смешивается с REST transport и persistence. |
| STD-042 | Ошибки обрабатываются явно; пустые catch/except запрещены. |
| STD-043 | Комментарий объясняет причину и trade-off, не очевидное действие. |
| STD-044 | Дублирование устраняется после подтверждения общего контракта. |
| STD-045 | Input/output types и validation применяются там, где поддерживаются стеком. |

Code formatting автоматизируется выбранным formatter/linter проекта; конкретный инструмент определяется стеком, а не Bitrix24.

## 9. Работа с данными

```text
Input -> Schema validation -> Normalize -> Identity/deduplicate
 -> Map -> Apply -> Reconcile -> Quality report
```

| ID | Стандарт |
|---|---|
| STD-046 | Входные данные проверяются до побочных эффектов. |
| STD-047 | Phone, email, identifiers, dates, currencies и dictionaries нормализуются по формальным правилам. |
| STD-048 | Mapping versioned и содержит source, target, transform, default и owner. |
| STD-049 | External ID сохраняется для сопоставления и идемпотентности. |
| STD-050 | Dedup keys и conflict disposition согласованы с data owner. |
| STD-051 | Каждая batch имеет balance: processed + skipped + failed + unknown. |

## 10. Безопасность разработки

| Контроль | Правило |
|---|---|
| Secrets | Secret manager/runtime environment; `.env` только локально и ignored |
| API keys/tokens | Least scope, rotation, revoke, no output |
| Rights | Dedicated technical identity; no admin by default |
| Logs | Safe metadata; no PII/credentials/payload by default |
| Test data | Synthetic only |
| Dependencies | Minimal, reviewed, pinned per stack policy |

| ID | Стандарт |
|---|---|
| STD-052 | Утечка credential считается security incident и требует отзыва. |
| STD-053 | `.env.example` содержит только фиктивные значения. |
| STD-054 | Персональные данные собираются и логируются только при подтверждённой необходимости. |
| STD-055 | Security checks входят в code review, tests, deployment и monitoring. |

## 11. Git Standards

```text
Issue/ADR -> codex-or-feature branch -> small commits -> Pull Request
 -> review + tests -> merge -> release tag where policy requires -> monitor/rollback
```

- ветка описывает одну связанную задачу;
- commit атомарен, понятен и не содержит секретов/генерируемых данных;
- Pull Request содержит цель, scope, риск, tests, deployment и rollback;
- code review выполняет другой компетентный участник для production changes;
- release/tag policy определяется проектом и не подменяет changelog;
- history rewriting, force push и destructive Git запрещены без согласования.

| ID | Стандарт |
|---|---|
| STD-056 | Перед работой и перед передачей проверяются `git status` и diff. |
| STD-057 | Секрет, попавший в Git, отзывается; простого удаления строки недостаточно. |
| STD-058 | Commit/PR связывается с requirement, ADR или change ID. |
| STD-059 | Rollback релиза основан на проверенном artifact/config/data plan. |

## 12. Environment Management

| Среда | Назначение | Данные | Доступ |
|---|---|---|---|
| Development | Локальная разработка | Synthetic/mocks | Developers |
| Testing | Functional/contract/integration tests | Synthetic | QA/team |
| Staging | Production-like rehearsal | Synthetic/anonymized only by approved policy | Restricted |
| Production | Реальный процесс | Authorized data | Least privilege |

| ID | Стандарт |
|---|---|
| STD-060 | Каждая среда имеет отдельные secrets, webhooks, tokens и endpoints. |
| STD-061 | Production data не копируется в non-production без правового основания и необратимого обезличивания. |
| STD-062 | Environment configuration валидируется и имеет owner/version. |

## 13. Testing Standards

Связанный материал: `knowledge/testing/README.md`.

| Вид | Минимальное назначение |
|---|---|
| Unit | Pure business logic, mapping, validation |
| Integration | REST adapter, queue, external contracts |
| Regression | Impacted critical paths |
| Smoke | Post-deployment viability and one negative guard |
| UAT | Business acceptance by authorized owner |

| ID | Стандарт |
|---|---|
| STD-063 | Каждый defect fix имеет regression test на соответствующем уровне. |
| STD-064 | Tests покрывают success, validation error, timeout, retry, duplicate и partial result. |
| STD-065 | Critical tests не выполняются на реальных клиентах без согласования. |

## 14. Logging Standards

```text
timestamp + level + service + environment + correlation_id
 + operation_id + event/method alias + status + duration + retry_count
```

Логировать: безопасные IDs, class ошибки, timing, transitions и outcome. Не логировать: token, password, webhook URL, cookie, full payload, phone/email, документы и запись разговора.

| Level | Назначение |
|---|---|
| DEBUG | Локальная диагностика без sensitive data |
| INFO | Значимое состояние операции |
| WARN | Recoverable anomaly |
| ERROR | Failed operation requiring action |
| CRITICAL | Service/data/security impact |

| ID | Стандарт |
|---|---|
| STD-066 | Correlation ID opaque и не содержит ПДн. |
| STD-067 | Retention, access и deletion policy определены. |
| STD-068 | Ошибка логируется один раз на ответственном слое с causal context. |

## 15. Error Handling

| Класс | Пример | Реакция |
|---|---|---|
| User/validation | Неверный input | Понятный ответ без internals |
| Business | Недопустимый transition | Controlled rejection |
| Technical permanent | Unknown method/field | Fix contract; no retry |
| External transient | Timeout/temporary unavailable | Bounded retry/backoff |
| Unknown outcome | Write timeout | Reconciliation before retry |
| Security | Auth/revoke/scope | Stop, alert, incident as needed |

| ID | Стандарт |
|---|---|
| STD-069 | Error taxonomy едина между code, logs, metrics и runbooks. |
| STD-070 | Пользователь не получает stack trace, secret или internal detail. |
| STD-071 | Fallback не скрывает потерю данных и имеет monitoring. |
| STD-072 | Permanent error не ретраится автоматически. |

## 16. Integration Development Standards

| Интеграция | Архитектура | Данные/ошибки | Monitoring/support |
|---|---|---|---|
| 1С | Contract зависит от configuration/app | Master, mapping, external IDs | Reconciliation and joint runbook |
| Телефония | Provider boundary + CRM integration | Caller normalization, call IDs | Synthetic call, provider escalation |
| Сайты | Backend adapter, not secret in browser | Validation, spam, consent, source | Submission journey |
| Платежи | Provider as financial truth if designed | Transaction ID, unknown states | Reconciliation/refund handling |
| BI | Versioned datasets/metric contracts | Grain, freshness, permissions | Control totals |
| External API | Adapter/queue based on SLA | Timeout, retry, idempotency | SLI, alerts, ownership |

| ID | Стандарт |
|---|---|
| STD-073 | Для каждого типа данных назначена master system. |
| STD-074 | Contract version, mapping, external IDs и consistency model документированы. |
| STD-075 | Интеграция имеет reconciliation, incident и offboarding procedures. |

## 17. Database Standards (On-Premise)

Применяется только к On-Premise и сверяется с [официальной документацией](https://dev.1c-bitrix.ru/docs/portal.php).

- ядро продукта и системная схема не изменяются неподдерживаемым способом;
- прикладные migrations versioned, reviewed, backed up и rehearsed;
- индексы и запросы меняются только после evidence, compatibility и update analysis;
- параметры, permissions и secrets не hardcoded;
- backup согласует БД/файлы и имеет restore test;
- прямые внутренние детали не предполагаются без официального источника.

| ID | Стандарт |
|---|---|
| STD-076 | DB change имеет forward/rollback или documented compensation. |
| STD-077 | Production query проходит ограниченную test sample и execution-risk review. |
| STD-078 | Backup existence без restore evidence не считается готовностью. |

## 18. Performance Standards

| Практика | Стандарт |
|---|---|
| REST queries | Минимальный `select`, фильтры и pagination |
| Batch | Ограниченные пакеты и per-item result |
| Queues | Async для долгих/массовых операций |
| Cache | Только при ясной invalidation/consistency модели |
| Blocking | Нет долгих внешних вызовов в user callback |
| Capacity | Baseline, growth, percentiles, backlog |

| ID | Стандарт |
|---|---|
| STD-079 | Performance claim подтверждается измерением на согласованном профиле. |
| STD-080 | Нагрузочные тесты production запрещены без отдельного согласования. |
| STD-081 | Optimization не ухудшает consistency, security и recovery. |

## 19. Documentation Standards

Каждый production-проект содержит:

- architecture context/dataflow;
- ADR и Cloud/On-Premise applicability;
- integration contracts и method registry;
- configuration, fields, stages, roles and rights;
- test strategy/cases/results;
- deployment, migration and rollback;
- monitoring, alerts and incident runbook;
- support guide, owners and changelog.

| ID | Стандарт |
|---|---|
| STD-082 | Документация обновляется в том же change scope. |
| STD-083 | Диаграммы и примеры обезличены. |
| STD-084 | Known limitations и unverified assumptions выделены явно. |

## 20. Code Review Checklist

- [ ] architecture/ADR и scope понятны;
- [ ] Cloud/On-Premise применимость подтверждена;
- [ ] secrets и ПДн отсутствуют;
- [ ] rights/scopes минимальны;
- [ ] code читаем, типизирован и не дублирует logic без причины;
- [ ] input validation и error taxonomy применены;
- [ ] timeout/retry/idempotency/partial errors обработаны;
- [ ] logs безопасны и имеют correlation ID;
- [ ] tests покрывают positive/negative/repeat/recovery;
- [ ] performance и query volume оценены;
- [ ] docs/changelog/runbook обновлены;
- [ ] deployment/monitoring/rollback готовы.

| ID | Стандарт |
|---|---|
| STD-085 | Reviewer фиксирует actionable findings и блокирует critical security/data risks. |
| STD-086 | Self-review не заменяет independent review production change. |

## 21. Release Standards

```text
Freeze scope -> Review/tests -> Backup/checkpoint -> Deploy pilot
 -> Smoke -> Reconcile -> Observe -> Close or rollback
```

| ID | Стандарт |
|---|---|
| STD-087 | Release имеет owner, window, change set, approvals и stop conditions. |
| STD-088 | Backup/rollback проверяются до первого production change. |
| STD-089 | Post-deployment smoke включает business path и negative permission guard. |
| STD-090 | Success подтверждается monitoring window и business owner. |

## 22. Support Handover

```text
Developer -> Solution Architect -> Support -> Client owner
 code/tests    architecture/risks   runbook/SLA   acceptance
```

Передаются: service map, owners, environments, safe configuration references, contracts, dashboards, alerts, error taxonomy, known issues, deployment/rollback, troubleshooting and escalation.

| ID | Стандарт |
|---|---|
| STD-091 | Support подтверждает доступ и выполнение ключевого runbook. |
| STD-092 | Временные developer permissions отзываются после handover. |
| STD-093 | Client получает user-facing ограничения без внутренних секретов. |

## 23. Типовые ошибки разработки

| № | Ошибка | Причина | Последствие | Как правильно |
|---:|---|---|---|---|
| 1 | Код до анализа штатных средств | Developer bias | Лишняя стоимость | Standard-first review |
| 2 | Нет бизнес-цели | Technical task only | Непроверяемый результат | KPI/acceptance |
| 3 | Нет AS IS | Discovery skipped | Ломаются зависимости | Inventory |
| 4 | Нет ADR | Решение устное | Повтор споров | Document decision |
| 5 | Cloud/Box смешаны | Context unknown | Невыполнимая архитектура | Environment passport |
| 6 | Тариф/версия не проверены | Assumption | Missing function | Verify portal/docs |
| 7 | Изменение ядра | Shortcut | Broken updates | Supported module |
| 8 | Прямая запись в системную БД | Speed | Data corruption | Supported API/model |
| 9 | Giant service | No boundaries | Hard maintenance | Layered components |
| 10 | Business logic в controller | Quick implementation | Untestable code | Domain service |
| 11 | REST calls scattered | No adapter | Contract drift | Central adapter |
| 12 | Дублирование mapping | Copy/paste | Inconsistent data | Versioned mapping |
| 13 | Hardcoded portal domain | One environment | Wrong target | Environment config |
| 14 | Hardcoded user/field ID | Quick fix | Breaks migration | Metadata/config mapping |
| 15 | Hardcoded secret | Convenience | Compromise | Secret store |
| 16 | Token в Git | No scanning/review | Persistent leak | Revoke and secure config |
| 17 | `.env` tracked | Misconfiguration | Secret exposure | Ignore/local only |
| 18 | Реальный webhook в docs | Copy example | Credential leak | Placeholder |
| 19 | Один secret для всех сред | Simplicity | Cross-environment impact | Separate secrets |
| 20 | Admin credential by default | Ease | Excess privilege | Dedicated least scope |
| 21 | Нет revoke procedure | Lifecycle ignored | Orphan access | Credential registry |
| 22 | OAuth flow придуман | Incomplete docs read | Auth failure | Official contract |
| 23 | Redirect слишком широк | Convenience | Token risk | Exact allowlist |
| 24 | Scope не reviewed | Feature creep | Excess access | Periodic review |
| 25 | Секрет в браузере | Wrong architecture | Public exposure | Backend exchange |
| 26 | Нет input validation | Trust external data | Corruption/injection | Schema validation |
| 27 | Пустые значения не обработаны | Happy path | Exceptions | Boundary tests |
| 28 | Unknown fields ignored silently | Loose parser | Hidden drift | Explicit policy |
| 29 | Phone/email не normalized | No data rules | Duplicates | Formal normalization |
| 30 | Название компании как key | No external ID | False match | Stable ID |
| 31 | External ID отсутствует | UI-centric design | Duplicates | Persist crosswalk |
| 32 | Mapping не versioned | Manual changes | Irreproducible result | Mapping artifact |
| 33 | Нет master system | Bidirectional edits | Conflicts | Ownership |
| 34 | Conflict silently overwritten | Last-write-wins accident | Data loss | Disposition workflow |
| 35 | Нет dedup policy | Create-only design | Duplicates | Identity rules |
| 36 | HTTP 200 равен success | Shallow validation | Hidden API error | Validate payload/state |
| 37 | HTTP/API errors смешаны | No taxonomy | Wrong retry | Layered errors |
| 38 | Нет timeout | Default client | Hung workers | Explicit timeouts |
| 39 | Retry отсутствует | No resilience | Lost transient operations | Bounded retry |
| 40 | Retry всех ошибок | Simplicity | Storm | Classification |
| 41 | Retry write after timeout | Unknown result | Duplicate effect | Reconcile first |
| 42 | Нет backoff/jitter | Immediate retry | Thundering herd | Bounded strategy |
| 43 | Infinite retry | No terminal state | Stuck queue | DLQ/manual review |
| 44 | Нет idempotency | Happy path | Duplicates | Operation ID |
| 45 | Поиск перед create не race-safe | Naive dedup | Concurrent duplicates | External guard/reconcile |
| 46 | Pagination забыта | Small test data | Missing records | Full traversal |
| 47 | Запрашиваются все поля | Convenience | Latency/load | Minimal select |
| 48 | Неподтверждённый limit hardcoded | Guess | Production failure | Official source/config |
| 49 | Batch считается atomic | Misunderstanding | Partial loss | Per-command result |
| 50 | Огромный batch | Speed | Timeout/recovery cost | Controlled chunks |
| 51 | Нет dry-run | Deadline pressure | Mass damage | Preview/sample |
| 52 | Нет checkpoint | One-shot script | Restart from zero | Durable state |
| 53 | Нет batch report | No audit | Unknown outcome | Full disposition |
| 54 | Failed rows discarded | Simplistic loop | Data loss | Error store/review |
| 55 | Callback делает долгую работу | No queue | Timeout/duplicates | Validate/enqueue |
| 56 | Нет queue monitoring | Backend focus | Hidden backlog | Depth/age alerts |
| 57 | DLQ отсутствует | Retry-only | Lost poison messages | Controlled DLQ |
| 58 | Replay без dry-run | Incident pressure | Duplicate effects | Review/idempotency |
| 59 | Event order assumed | Sequential thinking | Stale overwrite | Version/reconcile |
| 60 | Webhook signature assumed | Wishful security | False trust | Verify contract |
| 61 | Один webhook на все systems | Reuse | No isolation | Per-consumer credential |
| 62 | Webhook без owner | Personal setup | Orphan access | Named owner |
| 63 | Payload полностью логируется | Debug convenience | PII leak | Safe metadata |
| 64 | Нет correlation ID | Local logs only | No traceability | Opaque correlation |
| 65 | Correlation ID содержит email | Convenience | PII exposure | Random opaque ID |
| 66 | Ошибка проглатывается | Empty catch | False success | Explicit handling |
| 67 | Stack trace пользователю | Debug mode | Info leak | Safe error message |
| 68 | Один generic exception | Simplicity | Wrong reaction | Error taxonomy |
| 69 | Fallback скрывает сбой | Availability at any cost | Data drift | Alert/reconcile |
| 70 | Нет логов | Minimal code | No diagnostics | Structured safe logs |
| 71 | Слишком много DEBUG в prod | No policy | Cost/leak/noise | Levels/retention |
| 72 | Логи без retention | No owner | Cost/compliance | Lifecycle policy |
| 73 | Логи доступны всем | Convenience | Sensitive access | Least privilege |
| 74 | Нет metrics/alerts | Project ends at deploy | Late incidents | Monitoring |
| 75 | Alert без runbook | Dashboard-driven | No action | Owner/runbook |
| 76 | Нет unit tests | Deadline | Logic regressions | Test pure logic |
| 77 | Нет integration tests | Mocks only | Contract failure | Sandbox/contract tests |
| 78 | Нет negative tests | Happy path | Security/data errors | Failure cases |
| 79 | Нет repeat tests | One run | Duplicates | Idempotency test |
| 80 | Нет regression after fix | Narrow retest | Side effects | Targeted regression |
| 81 | Production data в tests | Convenience | Privacy breach | Synthetic data |
| 82 | Tests depend on order | Shared state | Flaky suite | Isolated fixtures |
| 83 | Cleanup широким filter | Easy teardown | Deletes other data | Exact run IDs |
| 84 | Performance not measured | Assumptions | Missed SLO | Baseline/profile |
| 85 | Load test production | No environment | Outage | Approved test scope |
| 86 | Average only | Simple dashboard | Tail hidden | Percentiles |
| 87 | Caching без invalidation | Premature optimization | Stale data | Consistency design |
| 88 | Sync call in webhook | Simplicity | Callback timeout | Async worker |
| 89 | No capacity planning | Current-only view | Future outage | Growth forecast |
| 90 | Нет documentation | Code speaks itself | No support | Required docs |
| 91 | README устарел | Separate process | Wrong operations | Same-change update |
| 92 | Нет changelog | No release discipline | Unknown baseline | Versioned changes |
| 93 | Нет deployment plan | Manual steps | Production error | Runbook |
| 94 | Нет backup | Optimism | No recovery | Backup gate |
| 95 | Backup without restore test | Checkbox | Failed recovery | Restore rehearsal |
| 96 | Нет rollback | Success-only plan | Long outage | Tested rollback |
| 97 | Нет smoke after release | Rush | Hidden failure | Critical journey |
| 98 | Нет observation window | Immediate closure | Recurrence missed | Monitor period |
| 99 | Нет support handover | Developer dependency | Slow incidents | Runbook/training |
| 100 | Temporary access not revoked | Closure missed | Security debt | Handover cleanup |
| 101 | 1С treated as universal contract | Product label only | Wrong mapping | Configuration passport |
| 102 | Telephony provider ignored | CRM-only view | Misdiagnosis | Boundary runbook |
| 103 | Payment callback trusted blindly | Event optimism | False payment | Provider verification |
| 104 | BI grain undocumented | Dashboard focus | Wrong totals | Metric contract |
| 105 | On-Premise update untested | Direct production | Incompatibility | Staging/regression |

## 24. Практические сценарии

| № | Задача | Архитектура | Реализация | Проверка | Риски |
|---:|---|---|---|---|---|
| 1 | 1С exchange | Middleware + master data | Mapping, IDs, queue | Reconciliation/repeat | Config differences |
| 2 | REST portal app | OAuth backend adapter | Official flow/scopes | Auth/contract tests | Token lifecycle |
| 3 | Incoming webhook client | Secure REST adapter | Secret store/timeouts | Negative auth/revoke | URL leak |
| 4 | Event webhook handler | Receiver→queue→worker | Dedup/correlation | Duplicate/out-of-order | Delivery assumptions |
| 5 | CRM field sync | Idempotent incremental | External ID/checkpoint | Repeat/boundary | Overwrite conflict |
| 6 | CRM automation | Standard-first robot/BP | Spec + guarded enable | Re-entry/error | Cycles |
| 7 | Smart Process service | Config + REST adapter | Metadata/category/stage | Rights/lifecycle | Tariff/features |
| 8 | BI integration | Dataset/metric contract | Grain/freshness | Control totals | Wrong formula |
| 9 | Telephony connector | Provider boundary | Call IDs/CRM mapping | Synthetic calls | Provider variance |
| 10 | Payment integration | Async verification | Transaction ID/status | Unknown/replay | False paid |
| 11 | Site form | Backend adapter | Validate/normalize/source | Synthetic submit | Spam/PII |
| 12 | Bulk import | Batch async tool | Dry-run/checkpoint/report | Balance/repeat | Mass damage |
| 13 | Data migration | ETL + reconciliation | Versioned mapping | Sample/full totals | Data loss |
| 14 | Duplicate cleanup | HITL data workflow | Candidate queue | Negative merge test | Wrong merge |
| 15 | Notification service | Event→dedup→channel | Template/owner | Noise/delivery | Alert fatigue |
| 16 | Document generation | Versioned template | Data snapshot/review | Field/version tests | Wrong document |
| 17 | On-Premise module | Supported extension | Isolated module/migrations | Upgrade/rollback | Core coupling |
| 18 | API polling | Incremental sync | Overlap/checkpoint | Boundary/full reconcile | Missed changes |
| 19 | Multi-system orchestration | External saga | Durable state/compensation | Partial failure | Irreversible step |
| 20 | Support monitoring | Logs/metrics/alerts | Correlation/runbooks | Incident drill | Blind spots |
| 21 | OAuth scope change | Security-controlled release | Review/new config | Regression/revoke | Excess access |
| 22 | Webhook owner offboarding | Credential lifecycle | Rotate/reassign/revoke | Old access denied | Orphan integration |

## 25. ADR Template

```markdown
# ADR-<ID>: <Decision>

- Date:
- Status:
- Owner:
- Approvers:

## Context
## Problem and constraints
## Decision
## Alternatives considered
## Consequences
## Risks and mitigations
## Security, data and rights
## Testing and acceptance
## Deployment, monitoring and rollback
## Review triggers
## Official sources
```

## 26. Developer Checklist

### Требования и архитектура

- [ ] business goal, scope, AS IS/TO BE и acceptance согласованы;
- [ ] standard-first assessment и ADR выполнены;
- [ ] Cloud/On-Premise, тариф/редакция, версия и права подтверждены;
- [ ] entities, master data, volumes и integration contracts описаны.

### Безопасность и код

- [ ] secrets/scopes/ПДн/threats reviewed;
- [ ] code разделён на domain/adapters/transport;
- [ ] input validation, error taxonomy и safe logging реализованы;
- [ ] timeout, retry, idempotency и partial results обработаны;
- [ ] dependencies минимальны и controlled.

### Тесты и выпуск

- [ ] unit/integration/regression/smoke/UAT scope готов;
- [ ] synthetic data и safe cleanup используются;
- [ ] deployment, backup, rollback и stop conditions проверены;
- [ ] monitoring, alerts, dashboards и runbooks готовы;
- [ ] documentation/changelog/support handover завершены;
- [ ] temporary access отозван после выпуска.

## 27. Official Sources

Дата последней проверки источника хранится в `knowledge/SOURCES.md`; перед реализацией актуальность проверяется повторно.

| Источник | Тип / приоритет | Использование |
|---|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | Официальная пользовательская документация / высокий | Функции, настройки и продуктовые ограничения |
| [Документация REST API](https://apidocs.bitrix24.ru/) | Официальная документация разработчика / критический | REST, OAuth, webhooks и приложения |
| [REST API reference](https://apidocs.bitrix24.ru/api-reference/) | Официальный справочник / критический | Методы, параметры, поля и ошибки |
| [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/index.html) | Официальная документация / критический | Webhook development и локальные сценарии |
| [Официальный Developer Hub](https://github.com/bitrix24/bitrix24-dev-hub) | Официальный GitHub / высокий | Навигация и официальные примеры без копирования репозитория |
| [Официальный PHP SDK](https://github.com/bitrix24/b24phpsdk) | Официальный GitHub / высокий | SDK reference после проверки версии |
| [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) | Официальная статья / высокий | Smart Process development context |
| [Варианты подключения телефонии](https://helpdesk.bitrix24.ru/open/6450911/) | Официальная статья / высокий | Telephony integration boundary |
| [Наборы данных BI](https://helpdesk.bitrix24.ru/open/24355590/) | Официальная статья / высокий | BI development and datasets |
| [Документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php) | Официальная документация 1С-Битрикс / критический | On-Premise modules and infrastructure context |

Сторонние блоги и форумы не используются. Неподтверждённые method, event, limit, OAuth flow, database detail, module capability или performance guarantee помечаются **«требует проверки»** и не включаются в production contract.
