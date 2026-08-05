# Bitrix24 Architecture Patterns Knowledge Base

Практическая библиотека архитектурных паттернов и антипаттернов для Senior Bitrix24 Solution Architect, Enterprise CRM Architect, Integration Architect и Business Process Architect.

> **Статус сведений:** продуктовые факты основаны только на официальных источниках из `knowledge/SOURCES.md`. Паттерн — инженерный способ организации решения, а не гарантия штатной функции Bitrix24. Доступность сущностей, настроек, автоматизаций, REST, BI, приложений и инфраструктурных механизмов зависит от Cloud/On-Premise, тарифа или редакции, версии, модулей, прав и провайдера.

> **Обозначения:** «Факт» — обобщение официальных источников; «рекомендация архитектора» — проектное решение; «требует проверки» — возможность или ограничение необходимо подтвердить для целевого портала.

> **Безопасность:** примеры обезличены. Пароли, токены, webhook URL, реальные домены, CRM ID, номера телефонов и клиентские данные запрещены.

## 1. Что такое архитектурный паттерн

Паттерн описывает повторяемую проблему, контекст, силы выбора, структуру решения и последствия. Он не является готовой конфигурацией и не доказывает наличие функции Bitrix24.

```text
Бизнес-задача -> Ограничения -> Паттерн -> Реализация -> Контроль результата
```

| Аспект | Паттерн | Готовое решение | Функция Bitrix24 |
|---|---|---|---|
| Назначение | Направляет архитектурный выбор | Решает конкретный scope | Возможность продукта |
| Контекст | Условия применимости | Конкретный портал | Зависит от поставки/тарифа |
| Ограничения | Явная часть паттерна | Проектные | Официально подтверждаются |
| Последствия | Trade-offs и риски | Фактические результаты | Не гарантируют архитектуру |

Критерии выбора: бизнес-ценность, сущности, ownership данных, объём, consistency, права, интеграции, automation, analytics, безопасность, эксплуатация, стоимость изменений и обратимость.

## 2. Методика выбора архитектуры

1. Определить измеримую бизнес-цель.
2. Описать AS IS и текущие ограничения.
3. Спроектировать TO BE и критерии готовности.
4. Определить сущности и их жизненные циклы.
5. Назначить владельцев данных и мастер-системы.
6. Оценить объёмы, рост и пиковую нагрузку.
7. Спроектировать роли, права и исключения.
8. Зафиксировать интеграции и контракты.
9. Выбрать automation mechanisms по приоритету штатных средств.
10. Определить метрики, аналитику и качество данных.
11. Разделить Cloud и On-Premise ограничения.
12. Выбрать минимальный набор паттернов.
13. Оценить риски и failure modes.
14. Подготовить positive, negative, repeat и recovery tests.
15. Зафиксировать решение в ADR.

```text
Goal -> AS IS -> TO BE -> Options -> Trade-offs -> ADR
    -> Pilot -> Test -> Deploy -> Monitor -> Review
```

## 3. CRM Entity Pattern

**Pattern card 01.** Один бизнес-объект представляется сущностью, соответствующей его идентичности и жизненному циклу.

| Бизнес-объект | Рекомендуемая сущность | Когда применять | Когда не применять | Риск |
|---|---|---|---|---|
| Неквалифицированное обращение | Лид | Нужна отдельная квалификация | Процесс сразу создаёт opportunity | Параллельная база клиентов |
| Коммерческая возможность | Сделка | Есть сумма/воронка/результат | Для постоянного клиента как справочника | Дубли истории |
| Физическое лицо | Контакт | Персона и коммуникации | Для юрлица | Смешение типов |
| Юридическое лицо | Компания | B2B-контекст и реквизиты | Для разных направлений одного клиента | Дубли компаний |
| Отдельный управляемый объект | Smart Process | Собственный lifecycle/права/связи | Если достаточно стандартной сущности | Избыточная модель |
| Исполнимая работа | Задача | Есть исполнитель, срок и результат | Как реестр бизнес-объектов | Потеря lifecycle |
| Простой справочник/реестр | Список | Если официально применим | Для сложной CRM-логики | Ограничения модели |
| Внешний master object | Внешняя система | Bitrix24 не владелец | Если работа должна жить в CRM | Рассинхронизация |

## 4. Lead-Based Pattern

**Pattern card 02.** Лид изолирует необработанный входящий поток до квалификации.

- использовать при большом неоднородном потоке и формальных критериях квалификации;
- фиксировать источник, owner, SLA, причину дисквалификации и правила дублей;
- конвертация должна создавать/связывать целевые сущности по подтверждённым правилам;
- аналитика отделяет вход, квалификацию и продажу.

Варианты: классические лиды; простая CRM без лидов; лид только как входящий необработанный запрос. Не применять лид как вечный клиентский реестр или параллельную сделку.

## 5. Deal-First Pattern

**Pattern card 03.** При подтверждённой коммерческой возможности создаётся сделка сразу, без лида.

```text
Source -> Deal -> Link Contact/Company -> Activities -> Won/Lost
                        \-> next Deal for repeat opportunity
```

Источник сохраняется в сделке; контакт/компания дедуплицируются; повторное обращение обычно создаёт новую opportunity. Последствие: исчезает lead qualification funnel, поэтому критерий входа сделки и аналитика источников должны быть строже.

## 6. Multiple Pipelines Pattern

**Pattern card 04.** Отдельные направления применяются для действительно разных lifecycle, ролей, обязательности, automation или аналитики.

Допустимые контексты: разные продукты, отделы, регионы, B2B/B2C, продажи/сопровождение, новые/повторные продажи, сервис и конфиденциальные процессы — только при реальном различии.

Не создавать направление ради менеджера, источника, одного поля, отчёта или косметического различия. Риски: дубли стадий, разъезд automation, сложные права и фрагментированная аналитика.

## 7. Single Pipeline Pattern

**Pattern card 05.** Одна воронка используется при общем lifecycle и контролируется полями/условиями.

Преимущества: единая аналитика, меньше дублирования, проще поддержка. Ограничения: чрезмерное ветвление перегружает стадии и условия. Если роли, финальные результаты или процесс существенно различаются, пересмотреть Multiple Pipelines.

## 8. Smart Process Pattern

**Pattern card 06.** Smart Process представляет отдельный объект с собственным жизненным циклом. Доступность функций проверяется по [официальной статье](https://helpdesk.bitrix24.ru/open/18913884/).

| Объект | Стадии | Связи | Права/automation/analytics | Ограничение |
|---|---|---|---|---|
| Договор | Draft→Approval→Active→Closed | Client/deal | Legal roles, approval, expiry | ЭДО требует отдельной системы/проверки |
| Заявка | New→Work→Resolved→Closed | Client/object | Queue, SLA, escalation | Сложный ITSM может требовать внешнюю систему |
| Объект | Planned→Active→Archived | Deals/requests | Owner, inspections | Гео/asset features проверить |
| Оборудование | Registered→In use→Service→Retired | Company/service | Custodian, maintenance | CMDB-scale требования отдельно |
| Сервис | New→Diagnosis→Fix→Accepted | Client/equipment | SLA and support roles | Omnichannel зависит от инструментов |
| Закупка | Request→Approval→Order→Received | Supplier/project | Spend approvals | Учёт остаётся в ERP при её master role |
| HR | Candidate/employee sub-lifecycle | User/tasks/docs | Restricted roles | ПДн и HR-system boundary |
| Лицензия | Ordered→Active→Renewal→Expired | Company/product | Renewal automation | Vendor truth external |
| Проект | Initiated→Delivery→Accepted→Closed | Deals/tasks | Portfolio links | Задачи не заменяются item-реестром |

## 9. Parent–Child Pattern

**Pattern card 07.** Родитель хранит общий контекст, дочерние элементы — повторяемые независимые части.

```text
Parent 1 -> Child N -> Activity/transaction
   \-> aggregate status/amount from verified children
```

Примеры: клиент→договоры, проект→этапы, объект→заявки, сделка→платежи, оборудование→обслуживание. Определить ownership, cardinality, cascade policy, агрегаты, права и REST mapping. Не удалять parent каскадно без подтверждённого правила.

## 10. Master Data Pattern

**Pattern card 08.** Для каждого типа данных назначается одна мастер-система.

| Данные | Возможный master | Consumer | Conflict rule |
|---|---|---|---|
| Клиент/коммуникации | Bitrix24 или внешняя система | Другие системы | Проектное решение |
| Реквизиты/номенклатура/цены | Часто 1С/ERP | Bitrix24 | ERP wins при согласовании |
| Web lead | Сайт до приёма, затем CRM | CRM | Transition ownership |
| Статус внешней операции | Внешний сервис | Bitrix24 | Provider-authoritative |

Журнал синхронизации хранит safe operation ID, version, result и conflict disposition. Свободная двусторонняя запись запрещена.

## 11. Event-Driven Pattern

**Pattern card 09.** Producer сообщает о подтверждённом событии, consumer обрабатывает его асинхронно.

```text
Producer -> Webhook/event -> Receiver -> Queue -> Idempotent consumer
                                      \-> DLQ -> Manual review
```

Внешняя реализация обеспечивает быстрый приём, validation, retry classification, deduplication, idempotency, dead-letter и monitoring. Наличие конкретного события, подписи, порядка и retry policy проверяется официально.

## 12. Polling Pattern

**Pattern card 10.** Consumer периодически запрашивает изменения через опубликованный REST-контракт.

Checkpoint хранит последнюю безопасную позицию; окно перекрытия компенсирует граничные изменения; dedup устраняет повторы. Риски: пропуски из-за фильтра/часового пояса, нагрузка и задержка. Webhook предпочтительнее для низкой задержки, если событие официально доступно и надёжность дополнена reconciliation.

## 13. Hybrid Integration Pattern

**Pattern card 11.** Webhook сигнализирует, REST получает актуальное состояние, polling/reconciliation закрывает пропуски.

```text
Webhook -> Queue -> REST read -> Apply
Periodic checkpoint scan --------^ -> Reconcile -> Full resync on approval
```

Это рекомендуемый подход для критичных обменов при доступных официальных событиях. Full resync — аварийный/периодический контроль, а не постоянная замена инкремента.

## 14. Middleware Pattern

**Pattern card 12.** Внешний middleware отделяет Bitrix24 от множества контрактов.

Функции: validation, mapping, routing, queue, idempotency, secrets, logging, monitoring и scaling. Он имеет owner, SLA/SLO, deployment, rollback и support runbook. В Cloud middleware всегда внешнее; On-Premise может иметь поддерживаемое расширение, но ядро не изменяется.

## 15. Point-to-Point Pattern

**Pattern card 13.** Одна система напрямую интегрируется с Bitrix24.

Подходит для одного простого стабильного контракта и ограниченного scope. Преимущества — низкая начальная стоимость; риски — рост связей `N×M`, дублирование mapping/secrets/retry и отсутствие единого контроля. При третьем и последующих потребителях пересмотреть middleware/hub.

## 16. Hub-and-Spoke Pattern

**Pattern card 14.** Центральный интеграционный слой соединяет внешние системы через единые правила.

```text
ERP --\
Site ---> Integration Hub ---> Bitrix24
BI  --/       | mapping, queue, audit, security
```

Плюсы: единые contracts, identity, monitoring. Риски: central failure, bottleneck и сильная зависимость. Требуются HA по бизнес-требованию, capacity, ownership, bypass/contingency и versioned interfaces.

## 17. Synchronous Pattern

**Pattern card 15.** Пользователь или caller ждёт немедленный результат.

Подходит для короткой операции, где результат нужен до продолжения. Обязательны timeout, понятная ошибка, запрет слепого retry write-операции и reconciliation неизвестного результата. Не использовать для длительных массовых цепочек.

## 18. Asynchronous Pattern

**Pattern card 16.** Запрос принимается, ставится в очередь и завершается позже.

```text
Submit -> Accepted/operation ID -> Queue -> Worker -> Final status
                                      \-> retry/DLQ/notify
```

Подходит для batch, внешних провайдеров и пиков. Требует статуса, eventual consistency contract, дедупликации, monitoring и пользовательского сценария ожидания.

## 19. Saga / Compensation Pattern

**Pattern card 17.** Внешний orchestration разбивает транзакцию на шаги и компенсации.

| Шаг | Действие | Компенсация |
|---|---|---|
| 1 | Создать/обновить CRM-object | Точная отмена/маркировка |
| 2 | Создать operation в ERP | Cancel command при поддержке |
| 3 | Связать IDs | Удалить/пометить relation |

Журнал хранит state и operation IDs; retry идемпотентен; необратимые шаги ведут в manual review. Штатная поддержка Saga в Bitrix24 не утверждается.

## 20. Idempotent Operation Pattern

**Pattern card 18.** Повтор одной логической операции не создаёт новый бизнес-эффект.

Использовать external ID, operation ID/idempotency key во внешнем сервисе, поиск по устойчивому ключу, controlled create/update и result registry. Универсальный `upsert` REST-метод не предполагается: точные методы проверяются в [API reference](https://apidocs.bitrix24.ru/api-reference/).

## 21. Batch Processing Pattern

**Pattern card 19.** Массовая обработка делится на контролируемые пакеты.

```text
Dry-run -> Sample -> Batch N -> Per-item results -> Checkpoint
                 -> retry eligible only -> reconcile -> next batch
```

Batch не считается атомарным. Нужны timeout, partial failure, безопасный retry, журнал, pause/resume, rollback/compensation и итоговый баланс.

## 22. Incremental Synchronization Pattern

**Pattern card 20.** Переносятся изменения после checkpoint с окном перекрытия.

Дата изменения используется только при подтверждённой семантике; overlapping window и dedup защищают границы. Удаление требует tombstone/status/reconciliation; периодический full comparison обнаруживает drift.

## 23. Full Synchronization Pattern

**Pattern card 21.** Полный набор перечитывается и сверяется при небольшом объёме, первоначальной загрузке или восстановлении.

Риски: стоимость, блокировки, rate pressure, длительное окно и перезапись новых данных. Нельзя использовать постоянно без необходимости; предпочтительны incremental sync и контрольные full comparisons.

## 24. Human-in-the-Loop Pattern

**Pattern card 22.** Автоматизация подготавливает решение, человек подтверждает исключение или рискованное действие.

Фиксируются initiator, assignee, input, deadline, escalation, decision, reason и audit evidence. Timeout не означает автоматическое одобрение. Применяется для конфликтов дублей, скидок, юридических исключений и необратимых операций.

## 25. Approval Pattern

**Pattern card 23.** Формальный маршрут согласования скидки, договора, оплаты, расхода, документа или нестандартного условия.

```text
Initiator -> Validate -> Approver -> approve/reject/rework
                       \-> substitute -> escalation
```

Определить threshold, initiator, approver, substitute, срок, отказ, повторное согласование после изменения, эскалацию и историю. Доступный механизм БП/robot зависит от портала.

## 26. State Machine Pattern

**Pattern card 24.** Объект имеет конечный набор состояний и разрешённых переходов.

| Элемент | Решение |
|---|---|
| States | Business meaning, не список действий |
| Transitions | From/to, actor, guard, side effect |
| Forbidden | Явно проверяются |
| Final | Success/failure/cancel semantics |
| Recovery | Reopen/compensation rule |

Применим к стадиям сделок, Smart Processes и статусным БП при доступности. Не создавать циклы роботами.

## 27. Orchestration Pattern

**Pattern card 25.** Центральный process/coordinator управляет последовательностью, состоянием, ошибками и компенсациями.

Плюсы: видимый flow и единый контроль. Минусы: центральная сложность и coupling. Для внешних сервисов нужны durable state, idempotency и monitoring; встроенные БП применяются только в подтверждённых пределах.

## 28. Choreography Pattern

**Pattern card 26.** Независимые участники реагируют на события без центрального оркестратора.

Плюсы: автономность; риски: трудно восстановить causal chain, циклы и eventual consistency. Обязательны correlation ID, contract registry, ownership, deduplication и reconciliation.

## 29. Notification Pattern

**Pattern card 27.** Событие порождает адресное уведомление по выбранному каналу.

Фиксируются event, recipient rule, channel, template, dedup window, escalation, delivery evidence и noise budget. Уведомление не заменяет задачу, если требуется исполнение и результат.

## 30. Task-Based Pattern

**Pattern card 28.** Задача — единица исполнимой работы с постановщиком, исполнителем, сроком и проверяемым результатом.

Связь с CRM обеспечивает контекст; task status не должен подменять lifecycle договора, заявки или оборудования. Для бизнес-объекта с полями, связями и аналитикой выбрать CRM/Smart Process.

## 31. Document Generation Pattern

**Pattern card 29.** Документ формируется из versioned template и проверенных CRM-данных.

```text
Template version + Approved data snapshot -> Generate -> Review/approve
 -> Sign (supported/external) -> Store -> Retain/archive
```

Определить источник каждого поля, version, права, согласование, хранение, подпись и retention. Подпись/ЭДО зависит от официального приложения или внешней системы.

## 32. Payment Tracking Pattern

**Pattern card 30.** Bitrix24 отображает состояние счёта/оплаты, а финансовая мастер-система подтверждает денежный факт, если так спроектировано.

Поддержать partial payment, debt, refund, status, external transaction ID, duplicate callback и reconciliation. Не считать callback доказательством оплаты без подтверждённого business status.

## 33. Repeated Sales Pattern

**Pattern card 31.** Повторная opportunity создаётся новой сделкой и связывается с тем же клиентом.

Trigger основан на lifecycle/сегменте, назначает owner и не создаёт дубль существующей opportunity. Аналитика разделяет new/repeat revenue; история остаётся у клиента и отдельных сделок.

## 34. Service Desk Pattern

**Pattern card 32.** Обращение — отдельная сущность с каналом, приоритетом, SLA, исполнителем, эскалацией, решением и closure/reopen policy.

Smart Process применим при подтверждённых возможностях; сложные ITSM/CMDB/omnichannel требования могут потребовать специализированную систему. SLA не задаётся универсальным числом.

## 35. Booking Pattern

**Pattern card 33.** Бронирование резервирует ресурс на временной интервал с контролем конфликтов.

```text
Request -> Check availability -> Hold -> Confirm/payment -> Book
                              \-> expire/cancel -> Release
```

Нужны resource ID, timezone, overlap rule, hold expiry, confirmation, cancellation, payment state и reminders. Штатная функция бронирования не предполагается без проверки тарифа/приложения; atomic conflict control может требовать внешнего сервиса.

## 36. Employee Lifecycle Pattern

**Pattern card 34.** Candidate→offer→hire→onboarding→role change→offboarding управляется связанными объектами и задачами.

Доступы выдаются по роли и сроку; HR data изолируются; offboarding отзывает sessions, apps, webhook ownership и передаёт объекты. HR/identity master и product capabilities требуют проверки.

## 37. Data Quality Pattern

**Pattern card 35.** Качество данных управляется как процесс с владельцем и измеримыми правилами.

| Контроль | Пример |
|---|---|
| Completeness | Required by stage |
| Validity | Type/range/dictionary |
| Normalization | Phone/email/name rules |
| Uniqueness | Approved duplicate keys |
| Ownership | Source and steward |
| Correction | Queue, reason, audit, retest |

## 38. Audit Trail Pattern

**Pattern card 36.** Для критичного изменения фиксируются who, what, when, old/new, source и operation ID в доступных журналах и внешней безопасной трассировке.

Единый штатный журнал для всех сущностей не предполагается. Доступность, полнота и retention истории проверяются на портале. Доступ к audit evidence ограничен.

## 39. Observability Pattern

**Pattern card 37.** Logs, metrics и correlation context позволяют обнаружить, объяснить и восстановить бизнес-операцию.

```text
Operation ID -> Safe logs + metrics + correlation chain
 -> alert -> dashboard -> runbook -> incident -> RCA
```

Traces и ручной replay реализуются внешними компонентами при необходимости; replay требует dry-run/idempotency. Не логировать ПДн и credentials.

## 40. Cloud Architecture Patterns

**Pattern card 38.** Cloud-first архитектура выбирает no-code/standard-first: CRM settings→fields/cards→robots/triggers→BP→Smart Processes→connectors/apps→REST/webhooks→external middleware.

| Слой | Подход | Ограничение |
|---|---|---|
| Configuration | Штатные настройки | Тариф/права |
| Automation | Robots/BP when available | Нет server PHP |
| Extension | REST/webhooks/apps | Published contracts only |
| Complex logic | External middleware | Own operations/security |

Внутри Cloud нет доступа к ядру, серверному коду и БД.

## 41. On-Premise Architecture Patterns

**Pattern card 39.** On-Premise сочетает штатные модули и REST с поддерживаемыми расширениями и управляемой инфраструктурой.

Учитывать events, agents, cron, queues, custom code, DB, files, web/PHP, monitoring, backup и updates только в пределах официальной версии и редакции. Кастомный код изолируется в поддерживаемом модуле; ядро не изменяется. Источник: [официальная документация](https://dev.1c-bitrix.ru/docs/portal.php).

## 42. Pattern Selection Matrix

| Задача | Рекомендуемый паттерн | Альтернатива | Ограничение | Риск |
|---|---|---|---|---|
| Неквалифицированный поток | Lead-Based | Deal-First | Нужны критерии | Двойная база |
| Сразу opportunity | Deal-First | Lead-Based | Чёткий entry rule | Мусорные сделки |
| Разные lifecycle | Multiple Pipelines | Single Pipeline | Реальная разница | Fragmentation |
| Общий lifecycle | Single Pipeline | Multiple Pipelines | Не перегружать branches | Giant pipeline |
| Договоры | Smart Process | External system | Возможности проверить | Overengineering |
| Проект и этапы | Parent–Child | Tasks | Relations/aggregation | Lost links |
| Номенклатура из ERP | Master Data | Manual sync | Ownership | Conflicts |
| Быстрое событие | Event-Driven | Polling | Event contract | Lost/duplicate event |
| Нет события | Polling | Hybrid | Checkpoint/filter | Lag/load |
| Критичная интеграция | Hybrid | Event-only | Reconciliation cost | Drift |
| Много систем | Middleware | Point-to-Point | Operations cost | Central dependency |
| Одна простая связь | Point-to-Point | Middleware | Growth trigger | Spaghetti |
| Общий integration policy | Hub-and-Spoke | Mesh/P2P | Hub capacity | SPOF |
| Мгновенный ответ | Synchronous | Async | Short operation | Timeout |
| Массовая операция | Asynchronous | Sync | Status/queue | Eventual consistency |
| Multi-system transaction | Saga/Compensation | Manual | External implementation | Partial state |
| Повтор доставки | Idempotent Operation | Manual dedup | Stable key | Duplicates |
| Массовое обновление | Batch Processing | One-by-one | Partial errors | Data damage |
| Частые изменения | Incremental Sync | Full Sync | Reliable checkpoint | Missed changes |
| Initial load | Full Sync | Incremental | Volume/window | Load spike |
| Рискованное исключение | Human-in-the-Loop | Full automation | SLA/escalation | Bottleneck |
| Скидка | Approval | Manager task | Formal thresholds | Informal decision |
| Lifecycle | State Machine | Free status field | Transition rules | Invalid state |
| Центральный workflow | Orchestration | Choreography | Coordinator reliability | Coupling |
| Autonomous consumers | Choreography | Orchestration | Observability | Hard diagnosis |
| Информирование | Notification | Task | Noise controls | Alert fatigue |
| Исполнимая работа | Task-Based | Smart Process | Not registry | Entity loss |
| Документ | Document Generation | Manual template | App/signature check | Wrong version |
| Оплата | Payment Tracking | Deal field only | Financial master | False paid status |
| Повторная продажа | Repeated Sales | Reopen deal | Duplicate guard | Broken history |
| Поддержка | Service Desk | Tasks | ITSM limits | No SLA model |
| Резерв ресурса | Booking | External booking | Atomicity/app check | Double booking |
| Жизненный цикл сотрудника | Employee Lifecycle | Task list | HR/security | PII leak |
| Грязные данные | Data Quality | Manual cleanup | Owner and rules | Recurrence |
| Критичный аудит | Audit Trail | Timeline only | Availability/retention | Missing evidence |
| Эксплуатация | Observability | Logs only | External tooling | Blind spots |

## 43. Architecture Decision Record

```markdown
# ADR-<ID>: <Decision title>

- Date:
- Status: proposed / accepted / superseded
- Author:
- Approvers:

## Context and business goal
## Problem
## Constraints
- Cloud/On-Premise, tariff/edition/version:
- Rights, volume, security, integrations:

## Options considered
### Option A
### Option B

## Decision
## Consequences
- Positive:
- Negative:
## Risks and mitigations
## Test and acceptance criteria
## Deployment, monitoring and rollback
## Review triggers and date
## Official sources
```

## 44. Антипаттерны

| № | Антипаттерн | Почему возникает | Последствие | Правильная альтернатива |
|---:|---|---|---|---|
| 1 | Одна гигантская воронка | Экономия настройки | Нечитаемые стадии | Single pipeline с ясным lifecycle или разделение |
| 2 | Воронка для каждого менеджера | Права через структуру | Дублирование | Roles/ownership |
| 3 | Воронка для источника | Аналитика через stages | Fragmentation | Source field |
| 4 | Smart Process вместо всех сущностей | Универсальность | Потеря CRM-семантики | Entity selection |
| 5 | Задача вместо бизнес-объекта | Быстро создать | Нет реестра/lifecycle | CRM/Smart Process + tasks |
| 6 | Сделка как клиент | Нет модели | Дубли identity | Contact/company master |
| 7 | Компания на каждое направление | Упрощение прав | Дубли клиента | Одна company + deals |
| 8 | Лид навсегда | Нет qualification exit | Parallel CRM | Lead-Based с conversion |
| 9 | Переоткрывать одну сделку | Экономия | Сломана история продаж | Repeated Sales |
| 10 | Стадии как список действий | Не выделены состояния | Giant state machine | Tasks/automation |
| 11 | Поле на каждый отчёт | BI-driven config | Card overload | Metric/model review |
| 12 | Дубли стандартных полей | Не изучена модель | Conflicting values | Standard-first |
| 13 | Текст вместо справочника | Быстро | Некачественная аналитика | Controlled enum |
| 14 | Обязательны все поля сразу | Максимальный контроль | Менеджеры вводят мусор | Stage-based requirement |
| 15 | Нет data owner | Общая ответственность | Drift | Master Data |
| 16 | Свободная двусторонняя синхронизация | «Всегда актуально» | Conflicts | Master + conflict policy |
| 17 | Название компании как ключ | Нет external ID | False match | Stable identifier |
| 18 | Телефон как абсолютно уникальный | Удобство | Wrong merge | Composite/verified identity |
| 19 | Full sync всегда | Простота | Load/overwrite | Incremental + reconciliation |
| 20 | Event-only без сверки | Доверие доставке | Silent loss | Hybrid Integration |
| 21 | Polling без checkpoint | Stateless script | Gaps/duplicates | Incremental Sync |
| 22 | Retry любого сбоя | Нет classification | Storm | Bounded retry |
| 23 | Retry write после timeout | Unknown state | Duplicate effect | Reconciliation/idempotency |
| 24 | Нет идемпотентности | Happy-path design | Duplicates | Idempotent Operation |
| 25 | Batch считается транзакцией | API abstraction | Partial loss | Per-command handling |
| 26 | Один огромный batch | Speed | Timeout/recovery cost | Controlled batches |
| 27 | Нет dry-run | Спешка | Mass damage | Preview/sample |
| 28 | Нет checkpoint | One-shot tool | Restart from zero | Durable checkpoint |
| 29 | Point-to-point mesh | Organic growth | Spaghetti | Middleware/Hub |
| 30 | Middleware без owner | Проектная времянка | Critical orphan | Service ownership |
| 31 | Central hub без resilience | Централизация | SPOF | Capacity/contingency |
| 32 | Sync для долгой операции | UI convenience | Timeout | Async status |
| 33 | Async без статуса | Backend-only thinking | User uncertainty | Operation state |
| 34 | Saga без compensation | Happy path | Partial state | Compensation/manual review |
| 35 | Choreography без correlation ID | Loose coupling | Нет RCA | Observability |
| 36 | Giant business process | Всё в одном | Неподдерживаемость | Modular orchestration |
| 37 | Робот на каждое действие | No design | Cycles/noise | Automation specification |
| 38 | Automation плохого процесса | «Оцифровать как есть» | Faster chaos | TO BE redesign |
| 39 | Автоодобрение по timeout | Удобство | Uncontrolled risk | Escalation/manual decision |
| 40 | Уведомление вместо задачи | Легко | Нет ownership/result | Task-Based |
| 41 | Задача и уведомление на всё | Fear of missing | Noise | Event/action matrix |
| 42 | Документ без version | Manual template | Wrong terms | Document Generation |
| 43 | Callback равен оплате | Integration shortcut | False paid | Payment Tracking/reconcile |
| 44 | Booking без atomic guard | CRM-only design | Double booking | External/controlled reservation |
| 45 | Audit trail в обычном text field | Нет журнала | Tampering/no context | Confirmed history/external audit |
| 46 | Логи с ПДн | Debug convenience | Data leak | Safe metadata |
| 47 | Токен в коде | Быстро | Compromise | Secret manager/env |
| 48 | Webhook без владельца | Personal setup | Orphan access | Credential register |
| 49 | Один webhook на все системы | Reuse | No attribution/revoke | Separate credentials |
| 50 | Нет мониторинга | Project ends at launch | Late incidents | Observability |
| 51 | Dashboard без action | Vanity metrics | No response | SLI/owner/runbook |
| 52 | Нет тестирования повторов | Happy path | Duplicates | Repeat tests |
| 53 | Нет negative rights tests | Admin-only QA | Data exposure | Permission matrix |
| 54 | Нет rollback | Optimism | Long outage | Deployment rollback |
| 55 | Нет ADR | Устные решения | Repeated debate | Architecture Decision Record |
| 56 | Cloud как собственный сервер | Box thinking | Impossible design | Cloud patterns |
| 57 | Box как повод менять ядро | Short-cut | Broken updates | Supported module/REST |
| 58 | Кастомизация раньше штатных средств | Developer bias | Cost/debt | Standard-first priority |
| 59 | App без scope/vendor review | Marketplace optimism | Security/lock-in | App assessment |
| 60 | Performance guarantee без теста | Sales promise | Unmet SLO | Baseline/capacity test |

## 45. Типовые ошибки выбора архитектуры

| № | Ошибка | Риск / корректирующее действие |
|---:|---|---|
| 1 | Цель не измерима | Сформулировать business outcome |
| 2 | AS IS пропущен | Инвентаризировать dependencies |
| 3 | TO BE копирует плохой процесс | Перепроектировать |
| 4 | Решение выбрано до требований | Выполнить discovery |
| 5 | Пожелание принято за архитектуру | Проверить альтернативы |
| 6 | Не определён owner решения | Назначить accountable |
| 7 | Нет ADR | Зафиксировать context/trade-offs |
| 8 | Паттерн принят за функцию | Проверить официальную возможность |
| 9 | Cloud/Box не разделены | Выбрать deployment context |
| 10 | Тариф/редакция неизвестны | Проверить портал |
| 11 | Права администратора предполагаются | Access discovery |
| 12 | Неверно выбрана CRM-сущность | Entity matrix |
| 13 | Нет самостоятельного lifecycle | Не создавать Smart Process |
| 14 | Стандартная сущность отвергнута без причины | Standard-first review |
| 15 | Задача заменяет реестр | Разделить object/work |
| 16 | Нет правил identity | Определить stable keys |
| 17 | Нет master system | Ownership matrix |
| 18 | Разные данные имеют одного master по привычке | Per-data ownership |
| 19 | Conflict resolution отсутствует | Formal rules |
| 20 | Объём не измерен | Baseline/growth |
| 21 | Peak load не оценён | Capacity profile |
| 22 | История считается бесконечной | Retention/archive |
| 23 | Связи не кардинализированы | Parent–Child design |
| 24 | Cascade delete не определён | Safe lifecycle |
| 25 | Агрегат доверяет неполным children | Reconciliation |
| 26 | Pipelines делятся по людям | Roles instead |
| 27 | Разные lifecycle смешаны | Multiple Pipelines |
| 28 | Единый lifecycle раздроблен | Single Pipeline |
| 29 | Stage semantics не определена | State Machine |
| 30 | Forbidden transitions не тестируются | Negative tests |
| 31 | Automation actor не указан | Execution context |
| 32 | Repeat run не спроектирован | Idempotency |
| 33 | Error branch отсутствует | Failure handling |
| 34 | Escalation не имеет owner | RACI/SLA |
| 35 | Human decision автоматизирован без права | HITL/Approval |
| 36 | Согласование без substitute | Continuity gap |
| 37 | Notification recipient hardcoded | Role-based resolution |
| 38 | Noise budget отсутствует | Alert fatigue |
| 39 | Sync выбран для batch | Async pattern |
| 40 | Async выбран без user status | Operation tracking |
| 41 | Event contract не подтверждён | Official verification |
| 42 | Polling filter semantics не проверена | Boundary tests |
| 43 | Checkpoint хранится ненадёжно | Durable state |
| 44 | Overlap window отсутствует | Missed updates |
| 45 | Full sync не ограничен | Load/risk controls |
| 46 | Webhook считается гарантированной доставкой | Reconciliation |
| 47 | Порядок событий предполагается | Version/order handling |
| 48 | Signature предполагается | Verify contract |
| 49 | Queue без DLQ/review | Lost failures |
| 50 | DLQ без privacy controls | Data leak |
| 51 | Correlation ID содержит ПДн | Use opaque ID |
| 52 | Retry без jitter/backoff | Storm |
| 53 | Circuit behavior не согласован | Failure mode design |
| 54 | Point-to-point growth trigger не задан | Architecture drift |
| 55 | Middleware становится shadow master | Clarify ownership |
| 56 | Hub schema универсализирует всё | Domain contracts |
| 57 | Saga compensation необратима | Manual review |
| 58 | Payment status редактируется свободно | Financial master |
| 59 | Booking timezone не указан | Double/incorrect booking |
| 60 | Hold expiry отсутствует | Resource leak |
| 61 | Service SLA не определён | No priority model |
| 62 | Reopen policy отсутствует | Duplicate tickets |
| 63 | Repeat sale открывает старую сделку | New opportunity |
| 64 | HR права общие | Data exposure |
| 65 | Offboarding не связан с integrations | Orphan access |
| 66 | Data quality owner отсутствует | Recurrence |
| 67 | Validation только в UI | API bypass |
| 68 | Audit completeness предполагается | Portal verification |
| 69 | Logs равны audit trail | Separate evidence model |
| 70 | Monitoring после запуска | Design before deployment |
| 71 | Метрика без action | Owner/runbook |
| 72 | SLO задан без SLI | Formal definition |
| 73 | Security рассматривается в конце | Threat model early |
| 74 | Secrets копируются в docs | Secure configuration |
| 75 | ПДн используются в тестах | Synthetic data |
| 76 | Performance обещана без baseline | Test and measure |
| 77 | Migration не входит в архитектуру | Mapping/reconciliation |
| 78 | Deployment не имеет feature isolation | Pilot/controlled rollout |
| 79 | Rollback не учитывает side effects | Compensation |
| 80 | Решение не имеет review trigger | ADR lifecycle |
| 81 | Приложение выбрано без exit strategy | Vendor/lock-in review |
| 82 | 1С названа единым универсальным контрактом | Configuration passport |
| 83 | Телефония смешана с CRM core | Provider boundary |
| 84 | BI строится до data quality | DQ gate |
| 85 | Metric formula не версионируется | Metric contract |

## 46. Практические сценарии

| № | Задача | Ограничения | Выбранный паттерн | Реализация | Риски | Проверка |
|---:|---|---|---|---|---|---|
| 1 | EdTech: заявки | Много неквалифицированных входов | Lead-Based→Deal | Qualification + course deal | Дубли | Conversion sample |
| 2 | EdTech: обучение | Отдельный lifecycle | Smart Process + Parent–Child | Enrollment→modules/tasks | Overmodeling | UAT by roles |
| 3 | B2B продажи | Company-centric | Deal-First + Master Data | Company→multiple deals | Duplicate companies | Identity audit |
| 4 | B2C продажи | Person-centric | Deal-First | Contact→opportunities | Phone ambiguity | Dedup tests |
| 5 | Производство | ERP master | Hub + Async + Saga | CRM order→ERP states | Partial success | Reconciliation |
| 6 | Сервис оборудования | Asset lifecycle | Smart Process + Parent–Child | Equipment→requests | CMDB limits | Relation/SLA tests |
| 7 | Недвижимость | Object and booking | Smart Process + Booking | Object→viewing hold | Double booking | Concurrency test |
| 8 | Медицина | Sensitive data | External master + HITL | Minimal CRM routing | Compliance risk | Access review |
| 9 | Логистика | External statuses | Event-Driven + Hybrid | Events→queue→REST→reconcile | Ordering | Version tests |
| 10 | HR onboarding | Restricted lifecycle | Employee Lifecycle | Approval→tasks→access | PII/access | Negative rights |
| 11 | Документооборот | Version/approval | Document Generation + Approval | Template→review→store | Wrong version | Snapshot test |
| 12 | 1С | Master varies by data | Middleware + Master Data | Versioned mapping/IDs | Conflicts | E2E/balance |
| 13 | Бронирование | Concurrency | Booking + external service | Hold/confirm/cancel | Atomicity | Parallel requests |
| 14 | Повторные продажи | Preserve history | Repeated Sales | Segment trigger→new deal | Duplicate opportunity | Guard test |
| 15 | Call-центр | High event flow | Event-Driven + Lead/Deal | Call→queue→CRM link | Caller ambiguity | Synthetic calls |
| 16 | Поддержка | SLA/escalation | Service Desk + State Machine | Request lifecycle/tasks | Tool limits | SLA paths |
| 17 | Оплаты | Financial truth external | Payment Tracking + Saga | Callback→verify→CRM status | False payment | Reconciliation |
| 18 | Массовая миграция | Volume and partial errors | Batch + Idempotency | Dry-run/checkpoint/report | Data loss | Repeat/rollback |
| 19 | Multi-system customer | Several consumers | Hub-and-Spoke | Canonical IDs/contracts | Hub SPOF | Failure tests |
| 20 | BI quality | Mixed sources | Data Quality + Observability | DQ gates/metric contracts | False KPI | Control totals |
| 21 | Согласование скидки | Role/threshold | Approval + HITL | Guard→approver→escalate | Delay | Timeout/reject tests |
| 22 | Cloud complex integration | No server code | External Middleware + Hybrid | Webhook/REST/queue | Operations cost | Contract/recovery |

## 47. Чек-лист Solution Architect

- [ ] бизнес-цель, KPI и criteria of success определены;
- [ ] AS IS, TO BE и scope согласованы;
- [ ] сущности и lifecycle выбраны обоснованно;
- [ ] данные, identity, retention и quality rules описаны;
- [ ] master system назначена для каждого типа данных;
- [ ] роли, права и negative cases определены;
- [ ] объёмы, рост, peak и capacity оценены;
- [ ] automation имеет trigger, actor, retry/error/escalation;
- [ ] integrations имеют contracts, IDs, mapping и consistency model;
- [ ] analytics имеет grain, formulas, sources и owners;
- [ ] security, secrets, ПДн и threat model учтены;
- [ ] performance SLI/SLO и test profile определены;
- [ ] testing покрывает positive/negative/repeat/recovery;
- [ ] deployment, pilot, monitoring и support готовы;
- [ ] rollback/compensation проверены;
- [ ] документация, diagrams, runbooks и changelog готовы;
- [ ] ADR фиксирует решение, trade-offs и review triggers;
- [ ] Cloud/On-Premise и тарифные зависимости подтверждены.

## 48. Чек-лист архитектурного ревью

- [ ] цель и бизнес-результат понятны;
- [ ] выбран правильный CRM/Smart Process/task/external object;
- [ ] нет дублирования сущностей и полей;
- [ ] определены master data и conflict resolution;
- [ ] права минимальны и проверяемы;
- [ ] ошибки, timeout и partial success обработаны;
- [ ] операции идемпотентны;
- [ ] monitoring, alerts и RCA path существуют;
- [ ] testing и synthetic data определены;
- [ ] deployment и migration воспроизводимы;
- [ ] rollback/compensation practically executable;
- [ ] ограничения Cloud/On-Premise явны;
- [ ] официальные product facts процитированы;
- [ ] решение документировано ADR;
- [ ] owner и review date назначены.

## 49. Cloud vs On-Premise

| Архитектурная область | Cloud | On-Premise | Дополнительный контроль | Риск |
|---|---|---|---|---|
| Configuration | Published settings | То же по версии | Tariff/edition | Capability mismatch |
| Server code | Недоступен внутри Cloud | Supported custom module | Review/update tests | Core modification |
| Database | Нет прямого доступа | Owner-controlled | Backup/schema discipline | Data damage |
| REST/webhooks | Published contracts | Зависят от версии/modules | Auth/limits/tests | Contract drift |
| Automation | Available robots/BP | Version-specific + extensions | Actor/error/retry | Cycles |
| Middleware | External | External or supported module | Operations/security | SPOF |
| Agents/cron/queues | Internal platform not controlled | Owner-controlled | Monitoring/idempotency | Duplicate jobs |
| Files/cache | Published functions only | Infrastructure control | Capacity/permissions | Loss/degradation |
| Monitoring | UI/API/external signals | + server/PHP/DB | SLO/runbooks | Blind spots |
| Updates | Vendor-managed | Owner-managed | Stage/backup/regression | Incompatibility |

## 50. Официальные источники

Дата последней проверки источника хранится в `knowledge/SOURCES.md`; перед ADR применимость проверяется повторно.

| Источник | Тип / приоритет | Использование |
|---|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | Официальная пользовательская документация / высокий | CRM, automation, права, Cloud-функции |
| [Документация REST API](https://apidocs.bitrix24.ru/) | Официальная документация разработчика / критический | REST, webhooks, OAuth, события |
| [REST API reference](https://apidocs.bitrix24.ru/api-reference/) | Официальный справочник / критический | Методы, поля, параметры и ошибки |
| [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/index.html) | Официальная документация / критический | Webhook и локальные сценарии |
| [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) | Официальная статья / высокий | Smart Process Pattern |
| [Варианты подключения телефонии](https://helpdesk.bitrix24.ru/open/6450911/) | Официальная статья / высокий | Телефония и provider boundary |
| [Наборы данных BI](https://helpdesk.bitrix24.ru/open/24355590/) | Официальная статья / высокий | Analytics/Data Quality/Observability |
| [Документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php) | Официальная документация 1С-Битрикс / критический | On-Premise и инфраструктура |

Сторонние блоги и форумы не используются. Если источник не подтверждает функцию, метод, событие, ограничение, журнал, импорт, подпись, производительность или серверный механизм, положение помечается **«требует проверки»** и не включается в обязательства перед клиентом.
