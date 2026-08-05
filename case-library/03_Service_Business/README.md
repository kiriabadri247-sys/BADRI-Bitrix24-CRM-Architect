# Service Business CRM Architecture Case

Синтетический архитектурный кейс компании, которая продаёт и выполняет услуги, управляет договорами, проектами, задачами специалистов, сроками, клиентской поддержкой и повторными обращениями.

> **Статус:** кейс является референсной архитектурой. Доступность лидов, Smart Processes, проектов, задач, роботов, бизнес-процессов, документов, BI, телефонии и приложений зависит от Cloud/On-Premise, тарифа или редакции, версии, прав, региона и провайдера. Конкретная реализация требует Discovery портала.

> **Безопасность:** все компании, сотрудники, суммы, домены и идентификаторы синтетические. Реальные клиентские данные, файлы, токены и webhook URL в Git запрещены.

> **Архитектурный принцип:** сделка отвечает за продажу, delivery-проект — за выполнение согласованного объёма работ, сервисное обращение — за отдельный запрос поддержки, задача — за конкретную работу исполнителя. Эти lifecycle не объединяются в одну гигантскую воронку.

## 1. Business Context

| Аспект | Референсное описание |
|---|---|
| Бизнес | Оказание проектных и регулярных услуг B2B/B2C |
| Вход | Заявки с сайта, звонки, email, мессенджеры, повторные обращения |
| Продажа | Оценка задачи, КП, согласование и договор |
| Delivery | Проект, этапы, задачи, сроки, результат и акт |
| Service | Поддержка по SLA/приоритету после продажи |

### Цели CRM

- не терять новые и повторные заявки;
- разделить продажу и выполнение работ;
- контролировать договорной scope, сроки и ответственность;
- видеть загрузку специалистов и риски delivery;
- управлять клиентскими обращениями и SLA;
- формировать аналитику продаж, прибыльности и качества услуг.

### Stakeholders

| Роль | Интерес | Ответственность |
|---|---|---|
| Собственник/директор | Revenue, margin, quality | Sponsor and acceptance |
| Руководитель продаж | Pipeline and conversion | Sales process owner |
| Руководитель delivery | Scope, capacity, deadlines | Project delivery owner |
| Специалист | Tasks and result | Execution |
| Бухгалтер | Contract, invoices, acts, payment facts | Financial/document owner |
| Support lead | SLA and service quality | Service process owner |

## 2. AS IS Problems

```text
Requests -> chats/spreadsheets -> sale -> manual handover
 -> tasks without project context -> missed deadlines -> reactive support
```

| Проблема | Причина | Последствие |
|---|---|---|
| Заявки теряются | Нет единого intake | Потеря продаж |
| Продажа смешана с delivery | Одна воронка на всё | Непрозрачные сроки и выручка |
| Договор не связан с работами | Файлы отдельно | Scope disputes |
| Проекты ведутся в чатах | Нет owner/plan | Срыв сроков |
| Задачи не связаны с результатом | Task list без проекта | Нет progress control |
| Обращения поддержки теряются | Нет service lifecycle | Нарушение SLA |
| Повторные заявки создают дубль клиента | Нет identity rules | Разрыв истории |
| Нет аналитики загрузки/маржи | Нет данных и формул | Неверное планирование |

## 3. TO BE Architecture

```text
Channels -> Lead/Deal sale -> Contract/Scope -> Delivery Project
                 |               |                 +-> Tasks/Results
                 |               |                 +-> Act/Acceptance
                 |               +-> Service entitlement
                 +-> Company/Contact
                              \-> Service Request N -> SLA -> Resolution
                              \-> Repeat Deal N

ERP/Accounting <-> Integration layer <-> Bitrix24 -> BI
```

| Слой | Целевая модель |
|---|---|
| CRM | Leads when qualification is needed; deals for sale; contacts/companies for clients |
| Services | CRM products/reference or ERP catalog by master ownership |
| Contract | Deal-linked document/register; Smart Process only for independent lifecycle |
| Delivery | Project/tasks or confirmed Smart Process + tasks |
| Support | Service Request Smart Process/external service desk by complexity |
| Integration | Website, communications, ERP/accounting and BI |

Cloud использует опубликованные функции и внешний код через REST. On-Premise может использовать поддерживаемые модули после проверки версии; ядро не изменяется.

## 4. CRM Model

### Лиды

Используются для необработанных обращений, если нужна квалификация до opportunity. Для действующего клиента с подтверждённым запросом может сразу создаваться новая сделка или service request по формальному правилу.

### Сделки

Одна сделка — одна коммерческая возможность. Она хранит service scope, amount, source, responsible, expected close, proposal and contract context. Выигранная сделка закрывается после подтверждённого коммерческого результата и запускает handover.

### Контакты и компании

Контакт — физическое лицо, компания — юридическое лицо/аккаунт. Повторная услуга связывается с существующим клиентом. Название компании не используется как единственный ключ.

### Услуги

| Вариант | Когда | Ограничение |
|---|---|---|
| Товары CRM | Услуги продаются как позиции сделки | Каталог/цены зависят от версии и master system |
| Controlled field/reference | Малый стабильный список | Не подходит сложной калькуляции |
| ERP catalog | ERP владеет номенклатурой/ценой | Нужны external IDs и reconciliation |

### Проекты

Проект представляет согласованный delivery scope, имеет owner, сроки, этапы, risks и acceptance. Реализация через проекты/задачи, Smart Process или внешнюю PM-систему выбирается по объёму, зависимостям, отчётности и доступным функциям.

```text
Company -> Contact N
        -> Deal N -> Service/Product rows
                  -> Contract/Proposal
                  -> Delivery Project -> Tasks N
        -> Service Request N
        -> Repeat Deal N
```

## 5. Service Pipeline

```text
Новая заявка -> Квалификация -> Оценка задачи -> КП
 -> Согласование -> Договор -> В работе -> Завершение
 -> Повторное обслуживание
```

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новая заявка | Зафиксировать и назначить owner | Client, source, request, contact | Assignment/SLA task |
| Квалификация | Подтвердить fit, urgency and potential | Need, service, budget/timing | Required fields/reminder |
| Оценка задачи | Определить scope, effort and risks | Deliverables, estimate, specialist input | Estimation task/escalation |
| КП | Подготовить предложение | Service rows, amount, validity, version | Document/approval task |
| Согласование | Согласовать условия | Discount, scope, objections, next step | Approval/reminder |
| Договор | Подтвердить legal and commercial terms | Contract version/status, planned dates | Legal/finance workflow |
| В работе | Handover to delivery | Project owner, scope, dates, team | Create/link project/tasks once |
| Завершение | Подтвердить результат | Deliverable, acceptance, act, lessons | Acceptance/closure checklist |
| Повторное обслуживание | Создать новую opportunity/request | Client, entitlement, new need | Repeat-deal/service trigger |

Для длинных проектов «В работе» не должна заменять полноценный delivery lifecycle. При выигрыше продажа закрывается, а delivery продолжается в связанном объекте.

## 6. Project Delivery Architecture

```text
Won Deal -> Handover checklist -> Project
 -> Milestones/Tasks -> Review -> Client acceptance -> Close
```

| Компонент | Стандарт |
|---|---|
| Project owner | Один ответственный за delivery outcome |
| Scope | Версия объёма работ и change procedure |
| Tasks | Исполнитель, срок, результат, relation to project/client |
| Milestones | Контрольные точки и acceptance criteria |
| Dependencies | Явные predecessor/blocked conditions при применимости |
| Progress | Не только task count, но deliverables/milestones |
| Reporting | Status, forecast, risks, effort/cost where data exists |

### Контроль выполнения

- overdue and blocked tasks;
- milestone slippage;
- change requests and scope drift;
- unassigned tasks;
- missing acceptance evidence;
- workload/capacity only with reliable planning data.

Задача не заменяет проект: она описывает отдельную исполнимую работу, а проект агрегирует scope и результат.

## 7. Service Management

Service request — отдельное обращение, связанное с клиентом, договором/entitlement, продуктом или проектом.

| Элемент | Решение |
|---|---|
| Channel | Website, email, telephony, messenger via confirmed connectors |
| Priority | Formal impact/urgency matrix |
| SLA | Response/resolution targets agreed by service class |
| Status | New→Assigned→In progress→Waiting→Resolved→Closed/Reopened |
| Owner | Service queue/team and assignee |
| Quality | Resolution, client confirmation, reopen reason |

```text
Request -> Validate entitlement -> Prioritize -> Assign
 -> Work / Wait -> Resolve -> Confirm -> Close
                      \-> Escalate / Reopen
```

Smart Process подходит при подтверждённых стадиях, правах и automation. При сложных ITSM, omnichannel, CMDB и SLA может потребоваться специализированная система.

## 8. Automation

| Trigger | Action | Guard | Monitoring |
|---|---|---|---|
| New request | Assign owner/create first task | Queue/service/region | Missing assignment |
| Stage entry | Create next-step task | Operation+stage key | Duplicate task |
| Deadline approaching | Reminder | Object still active | Notification count |
| Overdue | Escalate to lead | Grace/priority rule | Overdue age |
| Won sale | Create delivery handover | Not already created | Orphan/duplicate project |
| Service breach risk | Notify support lead | SLA clock valid | Breach dashboard |

Robоты применяются для локальных действий; БП — для согласований, multi-step exceptions и документов при доступности. Повторный вход не должен создавать двойной проект, задачу или уведомление.

## 9. Documents

| Документ | Источник данных | Lifecycle | Контроль |
|---|---|---|---|
| КП | Deal, service rows, pricing | Draft→approved→sent→superseded | Template/version/validity |
| Договор | Approved terms/client details | Draft→review→approved/signed→active | Version/rights/relation |
| Акт | Delivered scope/milestones | Draft→review→accepted/rejected | Delivery evidence |
| Внутренний документ | Project/request data | Role-specific | Minimal access/retention |

```text
Approved data snapshot + Template version -> Generate -> Review
 -> Sign/approve by supported process -> Store/link -> Archive/retain
```

Подпись, ЭДО и автоматическая генерация зависят от доступного официального механизма или внешнего приложения. Старые версии не перезаписываются без следа.

## 10. Permissions

| Роль | CRM | Сделки | Проекты | Задачи | Аналитика |
|---|---|---|---|---|---|
| Менеджер | Свои clients/leads | Свои opportunities | Read linked/handover | Sales tasks | Personal/team limited |
| Руководитель | Отдел | Команда/approvals | Portfolio read | Team/escalations | Sales/delivery summary |
| Специалист | Minimum linked client | Read scope only | Assigned projects | Assigned tasks | Personal workload |
| Бухгалтер | Required client fields | Financial/contract read | Minimal | Document/payment tasks | Financial reports |
| Директор | Approved broad scope | Strategic view | Portfolio | High-level | Executive dashboards |

Права на документы, service requests, export, BI and settings проверяются отдельно. Точная модель уровней и inheritance зависит от портала.

## 11. Integrations

| Интеграция | Данные | Метод | Контроль ошибок |
|---|---|---|---|
| Сайт | Request, contact, service, source | Backend→REST/official connector | Validation, dedup, synthetic submit |
| Телефония | Call ID, direction, user, CRM link | Official telephony/connector | Provider boundary, test calls |
| Почта | Communication/activity context | Official mail integration | Ownership, duplicate/thread context |
| Мессенджеры | Channel identity and messages | Supported connector/app | Rights, retention, availability |
| ERP/accounting | Services, prices, orders, payments, acts | Middleware/official contract | External IDs, queue, reconciliation |
| BI | CRM/project/service/finance facts | BI datasets/external BI as applicable | Freshness, formula, permissions |

```text
Channels -> Integration layer -> CRM sale/service
ERP <---- versioned mapping ----> Contracts/financial status
BI <----- governed metrics ------ CRM/Delivery/Support
```

Точные REST methods проверяются в [официальном API reference](https://apidocs.bitrix24.ru/api-reference/), телефония — по [официальной документации](https://helpdesk.bitrix24.ru/open/6450911/).

## 12. Analytics

| Отчёт | Метрики | Предусловие |
|---|---|---|
| Продажи | Leads, deals, conversion, amount | Stable stages/sources/amount semantics |
| Загрузка специалистов | Planned/assigned/completed effort | Reliable estimates/capacity data |
| Прибыльность | Revenue minus agreed cost model | Financial master and cost ownership |
| Сроки | Project/milestone/task cycle time | Consistent timestamps/timezone |
| SLA | Response/resolution/breach by priority | Formal SLA clock and exceptions |
| Повторные продажи | Repeat deals/revenue/client cohort | Stable client identity |

BI Конструктор и datasets зависят от тарифа/прав; источник — [официальная документация](https://helpdesk.bitrix24.ru/open/24355590/). Прибыльность не рассчитывается достоверно без подтверждённых costs and revenue.

## 13. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Новая заявка | Validation, dedup, assignment | Один объект, owner and first task |
| 2 | Продажа услуги | Deal fields/stages/products | Correct lifecycle and amount |
| 3 | КП | Version, approval, send | Approved version linked |
| 4 | Договор | Review, rights, relation | Correct active version |
| 5 | Создание проекта | Won-deal handover | Один linked project |
| 6 | Постановка задачи | Assignee/deadline/project relation | One actionable task |
| 7 | Просрочка | Reminder/escalation | Correct owner notified once |
| 8 | Завершение проекта | Deliverables/acceptance/act | Project closed with evidence |
| 9 | Service request | Priority/SLA/assignment | Correct queue and clock |
| 10 | SLA breach | Valid clock/escalation | Alert and incident trail |
| 11 | Повторное обращение | Existing client/new request/deal | No duplicate client |
| 12 | ERP timeout | Retry/reconcile | No duplicate financial effect |
| 13 | Права специалиста | UI/direct URL/REST/export | Only assigned scope |
| 14 | Повтор automation | Re-entry/duplicate event | No duplicate project/task |

Тестовые данные синтетические. UAT подписывают Sales, Delivery, Service и Finance owners.

## 14. Deployment

```text
Audit -> Architecture -> Pilot CRM -> Delivery/Service setup
 -> Integration rehearsal -> Training/UAT -> Cutover -> Hypercare
```

| Этап | Результат | Gate |
|---|---|---|
| Аудит | AS IS, data, processes, dependencies | Scope approved |
| Настройка | Entities, stages, rights, forms | Configuration tests |
| Automation | Disabled/tested robots and BP | Repeat/error tests |
| Migration | Mapping, dry-run, sample, balance | Data owner acceptance |
| Training | Role-based scenarios | Readiness |
| Launch | Backup/checkpoint, cutover, smoke | Go/no-go |
| Hypercare | Monitoring, incidents, support | Exit criteria |

Рабочие процессы изменяются через pilot/parallel configuration, если возможно. On-Premise требует совместимости server/PHP/DB/modules и restore test.

## 15. Monitoring

| Контроль | Сигнал | Действие |
|---|---|---|
| Intake | Expected vs created requests | Channel/integration investigation |
| Продажи | Stage aging/no next activity | Manager escalation |
| Проекты | Milestone/task overdue, blocked | Delivery review |
| Загрузка | Capacity overload/underload | Replan by owner |
| SLA | At-risk/breached requests | Support escalation |
| Documents | Missing/expired/unsigned status | Owner task |
| Integrations | Errors, lag, backlog, unknown | Runbook/reconciliation |
| Automation | Missing/duplicate/error | Automation owner |

Пороги определяются бизнесом и baseline; универсальные SLA и capacity values не задаются.

## 16. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Потеря заявки | Channel failure | Потеря продажи | Synthetic journey/reconciliation |
| 2 | Дубли клиентов | Weak identity | Fragmented history | Stable keys/review |
| 3 | Дубли сделок | Retry/repeat event | False pipeline | Idempotency |
| 4 | Продажа и delivery смешаны | One pipeline | No clear ownership | Separate objects |
| 5 | Project not created | Handover failure | Work not started | Handover monitoring |
| 6 | Duplicate project | Re-entry/retry | Double work | Operation key |
| 7 | Scope lost | Contract/project not linked | Dispute/rework | Versioned scope relation |
| 8 | Scope creep | Change process absent | Margin/deadline loss | Change request control |
| 9 | Wrong specialist | Assignment rule error | Quality/delay | Skill/owner review |
| 10 | Unassigned task | Automation/data gap | Work omitted | Assignment alert |
| 11 | Overdue task hidden | No monitoring | Project delay | Aging/escalation |
| 12 | Task count gives false progress | Vanity metric | Wrong forecast | Milestones/deliverables |
| 13 | No client acceptance | Closure by task completion | Payment/dispute risk | Acceptance criteria |
| 14 | Wrong document version | Overwrite/manual files | Legal error | Version/status |
| 15 | Act not linked | Document gap | Payment delay | Relation/checklist |
| 16 | Service request lost | Channels fragmented | SLA breach | Unified intake |
| 17 | Wrong priority | Subjective assignment | SLA misrouting | Impact/urgency matrix |
| 18 | SLA clock incorrect | Waiting/exceptions not defined | False breach | Formal clock rules |
| 19 | Request closed too early | No confirmation | Reopen/dissatisfaction | Resolution/confirmation |
| 20 | Repeat request duplicates client | No identity | Lost history | Existing-client lookup |
| 21 | Repeat sale reopens old deal | Convenience | Broken history | New deal |
| 22 | ERP/CRM conflict | Both masters | Wrong price/payment | Ownership by data type |
| 23 | Webhook leak | URL in code/log | Security incident | Secret store/revoke |
| 24 | Event missed | Event-only flow | Data drift | Reconciliation |
| 25 | Retry storm | No taxonomy/backoff | Degradation | Bounded retry |
| 26 | Telephony linked wrong client | Caller ambiguity | Data disclosure | Conflict queue |
| 27 | BI profitability false | Missing costs | Wrong decisions | Financial data gate |
| 28 | Excess specialist rights | Convenience | Client data leak | Least privilege |
| 29 | Real data in tests | No synthetic policy | Privacy breach | Synthetic fixtures |
| 30 | Migration without dry-run | Deadline | Data corruption | Rehearsal/sample |
| 31 | No rollback | Optimism | Long outage | Tested reverse plan |
| 32 | Cloud function unavailable | Tariff not checked | Scope failure | Early portal verification |
| 33 | Box core modified | Shortcut | Update failures | Supported extension |
| 34 | No support owner | Handover missing | Slow incidents | RACI/runbooks |
| 35 | No monitoring | Project ends at launch | Late failures | Operational SLI/alerts |

## 17. ADR

### ADR-SVC-001: Модель проекта

| Поле | Решение |
|---|---|
| Context | Delivery has scope, team, milestones and acceptance |
| Decision | Separate linked project/delivery object after won deal |
| Alternative | Keep deal in «В работе» until all tasks finish |
| Consequences | Clear ownership and analytics; requires handover |
| Review trigger | Project complexity/tooling changes |

### ADR-SVC-002: Хранение услуг

| Поле | Решение |
|---|---|
| Context | Services need pricing and deal product rows |
| Decision | CRM products or ERP catalog by master ownership |
| Alternative | Free-text service fields |
| Consequences | Better analytics; catalog governance required |
| Review trigger | Pricing/catalog model changes |

### ADR-SVC-003: Модель SLA

| Поле | Решение |
|---|---|
| Context | Service classes and priorities differ |
| Decision | Formal SLA per class/priority with clock exceptions |
| Alternative | One universal response time |
| Consequences | Accurate control; requires reliable timestamps |
| Review trigger | Contracts/service tiers change |

### ADR-SVC-004: Роль задач

| Поле | Решение |
|---|---|
| Context | Tasks execute work but do not represent full project/request |
| Decision | Tasks linked to project or service request |
| Alternative | Task-only registry |
| Consequences | Traceability; relation discipline needed |
| Review trigger | PM/service tooling changes |

### ADR-SVC-005: Повторные обращения

| Поле | Решение |
|---|---|
| Context | Existing clients request support and new paid work |
| Decision | New service request for support; new deal for commercial opportunity |
| Alternative | Reopen old deal/project for every request |
| Consequences | Correct history and analytics; classification required |
| Review trigger | Entitlement/support model changes |

### ADR-SVC-006: Service Desk implementation

| Поле | Решение |
|---|---|
| Context | Basic support lifecycle may fit Smart Process; complex ITSM may not |
| Decision | Use confirmed Smart Process only if SLA/rights/reporting needs are met |
| Alternative | External service desk integrated with CRM |
| Consequences | Standard-first, but capability verification mandatory |
| Review trigger | Volume, omnichannel, CMDB or SLA complexity grows |

## 18. Типовые ошибки

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Одна воронка для sale/project/support | Want one screen | Giant lifecycle | Separate objects |
| 2 | Сделка остаётся открытой до конца delivery | Sale and execution mixed | Wrong sales cycle | Close and handover |
| 3 | Проект хранится как text field | Quick setup | No control | Project object |
| 4 | Задачи заменяют проект | Task-centric design | No scope/milestones | Project + tasks |
| 5 | Service request заменён задачей | Easy assignment | No SLA/status/client history | Request object |
| 6 | Один клиент создаётся повторно | No identity policy | Fragmented history | Stable keys |
| 7 | Название компании как key | Convenience | False matches | External/verified ID |
| 8 | Повторная услуга в старой сделке | Reuse | Broken revenue history | New deal |
| 9 | Support request превращается в deal всегда | Commercial bias | Polluted pipeline | Classify support vs sale |
| 10 | Платная работа ведётся как бесплатный support | No entitlement rules | Margin loss | Commercial qualification |
| 11 | Услуга хранится свободным текстом | Fast data entry | Poor analytics | Catalog/reference |
| 12 | ERP и CRM оба edit prices | No master | Conflicts | Ownership |
| 13 | Scope не versioned | Informal agreement | Disputes | Approved scope version |
| 14 | Change request не фиксируется | Customer pressure | Scope creep | Change workflow |
| 15 | Нет handover checklist | Sales throws over wall | Work starts incorrectly | Formal acceptance |
| 16 | Project owner не назначен | Shared responsibility | No outcome owner | Single accountable owner |
| 17 | Tasks без project relation | Quick creation | No progress view | Mandatory link |
| 18 | Task without result criteria | Action-only title | False completion | Definition of done |
| 19 | Все задачи одинакового приоритета | No planning | Critical work delayed | Priority rules |
| 20 | Dependencies ignored | Flat task list | Rework/delay | Milestones/dependencies |
| 21 | Progress equals completed task count | Easy metric | False status | Deliverable/milestone |
| 22 | Overdue has no escalation | Notification-only | Chronic delay | Owner/escalation |
| 23 | Specialist sees all clients | Convenience | Data exposure | Linked scope |
| 24 | Manager closes project alone | No delivery governance | Unaccepted work | Client/internal acceptance |
| 25 | Act generated from unverified data | Automation first | Wrong document | Approved snapshot |
| 26 | Documents overwritten | No version control | Legal ambiguity | Version/status |
| 27 | Signature capability assumed | App optimism | Missing function | Verify official mechanism |
| 28 | SLA one value for all requests | Copy template | Unrealistic control | Service class/priority |
| 29 | Priority chosen freely | No matrix | Queue manipulation | Impact/urgency rules |
| 30 | Waiting client counts as team delay | Clock undefined | False breach | SLA exception rules |
| 31 | Request closed without resolution | Speed KPI | Reopens/dissatisfaction | Resolution evidence |
| 32 | Reopen creates unrelated request | No policy | Lost history | Reopen/link rule |
| 33 | No service entitlement | All requests accepted | Cost leakage | Contract/service check |
| 34 | Robot creates duplicate project | Re-entry | Double work | Idempotency key |
| 35 | Robot creates task on every edit | Broad trigger | Task spam | Exact event/guard |
| 36 | Notifications replace tasks | Convenience | No accountability | Task-Based pattern |
| 37 | Approval timeout means approve | Easy flow | Uncontrolled risk | Escalate/manual decision |
| 38 | Business process giant | Everything in one | Hard support | Modular workflows |
| 39 | No error branch | Happy path | Silent failure | Error/escalation |
| 40 | Website secret in browser | Wrong architecture | Credential leak | Backend adapter |
| 41 | Webhook in logs | Debug convenience | Compromise | Redact/revoke |
| 42 | No timeout/retry | Minimal client | Lost/hung integration | Explicit policy |
| 43 | Retry write blindly | Unknown outcome | Duplicates | Reconcile first |
| 44 | No idempotency | One-delivery assumption | Duplicate entities | Operation ID |
| 45 | No reconciliation | Event-only | Silent drift | Periodic compare |
| 46 | Telephony provider ignored | CRM-centric | Wrong diagnostics | Boundary runbook |
| 47 | BI margin without costs | Dashboard demand | False profitability | Financial data gate |
| 48 | Workload without estimates | Task count only | False capacity | Reliable effort model |
| 49 | Tests run as admin | Convenience | Hidden rights error | Target roles |
| 50 | Real clients in tests | No synthetic policy | Privacy breach | Synthetic fixtures |
| 51 | Migration without mapping | Deadline | Lost fields/relations | Dry-run/rehearsal |
| 52 | Automation active during import | No cutover plan | Mass side effects | Disabled load |
| 53 | No rollback | Optimism | Long outage | Tested plan |
| 54 | No monitoring after launch | Project closure | Late incidents | Operational controls |
| 55 | Cloud and Box treated equally | Generic design | Impossible solution | Explicit applicability |

## 19. Implementation Roadmap

| Этап | Результат | Минимальный scope | Критерий готовности |
|---|---|---|---|
| 1. Discovery | AS IS sales/delivery/service/data | Interviews, inventory, baseline | Scope approved |
| 2. Архитектура | TO BE, entities, stages, rights, ADR | Diagrams/matrices/risks | Sign-off |
| 3. CRM | Clients, leads/deals, services, permissions | Pilot sales flow | Configuration tests |
| 4. Delivery | Project/tasks/handover/documents | One service type | Delivery UAT |
| 5. Service | Request/SLA/priority/escalation | One support queue | SLA tests |
| 6. Integrations | Channels, ERP/accounting, BI | Critical contracts | E2E/reconciliation |
| 7. Launch | Migration, training, cutover, monitoring | Pilot team→rollout | Stable hypercare |
| 8. Optimization | Capacity, margin, SLA, repeat sales | Evidence-based backlog | Improvement cycle |

### Официальные источники

| Источник | Использование |
|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | CRM, задачи, права, automation и Cloud-функции |
| [Документация REST API](https://apidocs.bitrix24.ru/) | REST, webhooks, OAuth и integrations |
| [REST API reference](https://apidocs.bitrix24.ru/api-reference/) | Контракты методов/полей |
| [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) | Project/service objects при самостоятельном lifecycle |
| [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) | Варианты подключения |
| [Наборы данных BI](https://helpdesk.bitrix24.ru/open/24355590/) | Analytics datasets |
| [Документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php) | On-Premise applicability |

Если функция, приложение, SLA-механизм, событие, метод или ограничение не подтверждены официальным источником, положение отмечается **«требует проверки»** и не включается в обязательства перед заказчиком.
