# B2B Sales CRM Architecture Case

Синтетический архитектурный кейс компании, продающей сложные B2B-решения с длинным циклом сделки, несколькими участниками, коммерческими предложениями, согласованиями, повторными продажами и аккаунт-менеджментом.

> **Статус:** кейс является референсной архитектурой. Наличие лидов, Smart Processes, роботов, бизнес-процессов, документов, BI, телефонии, email, мессенджеров и приложений зависит от Cloud/On-Premise, тарифа или редакции, версии, прав, региона и провайдера. Конкретная реализация требует Discovery портала.

> **Безопасность:** все компании, люди, суммы, домены, номера и идентификаторы синтетические. Реальные клиентские данные, токены, webhook URL и выгрузки в Git запрещены.

> **Архитектурный принцип:** компания — единый клиентский аккаунт, контакт — физическое лицо в buying committee, сделка — отдельная коммерческая opportunity. Разные продукты и подразделения не создают дубли компаний без бизнес-обоснования.

## 1. Business Context

| Аспект | Референсное описание |
|---|---|
| Тип бизнеса | Продажа комплексных решений юридическим лицам |
| Цикл | От первичной возможности до договора и развития аккаунта |
| Длительность | Длинная, зависит от бюджета, тендера и согласований |
| Участники клиента | ЛПР, инициатор, пользователь, закупки, финансы, юрист, blocker |
| Участники продавца | Менеджер, solution expert, руководитель, юрист, сопровождение |

### Особенности B2B-продаж

- много контактов с разной ролью и влиянием;
- решение принимается buying committee, а не одним человеком;
- сумма, вероятность и прогноз меняются по мере квалификации;
- предложение может иметь несколько версий;
- скидка, договор и нестандартные условия требуют согласования;
- выигранная сделка открывает account-management lifecycle.

### Цели CRM

- не терять opportunities и следующий шаг;
- хранить единый 360°-контекст компании и участников;
- стандартизировать стадии и критерии выхода;
- повысить достоверность прогноза;
- контролировать активности и риски сделки;
- развивать повторные продажи без дублей клиента.

## 2. AS IS Problems

```text
Leads/emails/calls -> personal spreadsheets -> manager memory
 -> proposal files -> manual approvals -> opaque forecast
```

| Проблема | Причина | Влияние |
|---|---|---|
| Сделки теряются | Нет owner/next step/SLA | Потеря pipeline |
| Нет контроля этапов | Стадии не имеют exit criteria | Ложный прогресс |
| Нет истории коммуникаций | Каналы и файлы разрознены | Зависимость от менеджера |
| Нет прозрачности прогноза | Вероятность субъективна | Неверный план |
| Менеджеры работают по-разному | Нет общего playbook | Непредсказуемая конверсия |
| Нет контроля активности | Дела не связаны со сделками | «Зависшие» opportunities |
| Компании дублируются | Название используется как key | Разрыв account history |
| Контакты не классифицированы | Нет роли/influence map | Неизвестен ЛПР |

## 3. TO BE Architecture

```text
Inbound/Prospecting
       |
       v
Optional Lead -> Company + Contacts -> Deal opportunity
                    |                   |       |
             Buying committee      Products   Proposal/Agreement
                    |                   |       |
                    +--------------> Won sale -> Account development

ERP/Finance master <-> Middleware/REST <-> Bitrix24 -> BI
```

| Слой | Целевая модель |
|---|---|
| CRM | Companies, contacts, deals; leads only for unqualified flow |
| Product | CRM products or external ERP catalog by ownership |
| Smart Processes | Contract/project/service only with independent lifecycle |
| Automation | Robots/tasks for local actions; BP for approvals/exceptions |
| Integration | Website, telephony, email, messengers, ERP and BI |
| Operations | Testing, deployment, monitoring and rollback |

Cloud использует опубликованные настройки, приложения и REST. On-Premise может использовать поддерживаемые модули/расширения после проверки версии; ядро изменять нельзя.

## 4. CRM Data Model

```text
Company 1
  +-- Contact N [role, influence, decision status]
  +-- Deal N [opportunity, amount, forecast, competitor]
  |     +-- Product rows
  |     +-- Activities
  |     +-- Proposal/Contract relation
  +-- Account plan / repeat opportunities
```

### Компания

| Поле | Назначение | Правило |
|---|---|---|
| Industry | Сегментация | Controlled dictionary |
| Size | Потенциал/account tier | Формальные диапазоны |
| Segment | Strategic/enterprise/mid-market etc. | Business-owned classifier |
| Client status | Prospect/client/inactive | Не дублирует deal stage |
| External ID | Связь с ERP/master | Устойчивый key |

### Контакт

| Поле | Назначение | Ограничение |
|---|---|---|
| Role | ЛПР, initiator, user, finance, legal, procurement | Может меняться по сделке |
| Position | Должность | Не равно роли в покупке |
| Influence | High/medium/low по согласованной модели | Не выдавать за объективный факт |
| Decision stance | Supporter/neutral/blocker | Sensitive; права и purpose review |

Если один контакт играет разные роли в разных сделках, роль buying committee должна храниться в контексте связи/сделки доступным подтверждённым способом, а не только глобально в контакте.

### Сделка

| Поле | Назначение | Контроль |
|---|---|---|
| Amount/currency | Commercial potential | Источник и валюта определены |
| Probability | Forecast model | Не редактировать без governance |
| Stage | Business state | Entry/exit criteria |
| Forecast category | Pipeline/commit/upside etc. | Formal definition |
| Competitor | Конкурентный контекст | Controlled reference, access |
| Next step/date | Управление движением | Обязателен для активной opportunity |

## 5. Sales Pipeline Architecture

```text
Новая возможность -> Квалификация -> Выявление потребности
 -> Презентация решения -> Коммерческое предложение -> Переговоры
 -> Согласование договора -> Успешная продажа -> Развитие клиента
                              \-> Проиграно с причиной
```

| Стадия | Цель | Обязательные данные | Действия | Автоматизация |
|---|---|---|---|---|
| Новая возможность | Назначить owner и подтвердить context | Company/contact/source/product interest | Первичный контакт | Task/SLA reminder |
| Квалификация | Подтвердить fit и реальную opportunity | Need, potential, timing, participants | Qualification call | Required fields/escalation |
| Выявление потребности | Формализовать business/technical need | Use case, criteria, stakeholders, budget status | Discovery meetings | Next-step task |
| Презентация решения | Показать value и fit | Solution scope, participants, feedback | Demo/presentation | Follow-up reminder |
| Коммерческое предложение | Зафиксировать предложение | Version, products, amount, validity | Prepare/review/send | Approval/document task |
| Переговоры | Согласовать условия | Objections, competitor, forecast, next step | Negotiation plan | Risk notification |
| Согласование договора | Завершить legal/procurement | Contract version, approvals, planned close | Legal/finance workflow | BP approval/escalation |
| Успешная продажа | Подтвердить коммерческий результат | Signed/confirmed outcome per policy | Handover | Project/service/account task |
| Развитие клиента | Перевести в account management | Owner, success plan, renewal/upsell date | Account review | Repeat-sale trigger |

«Развитие клиента» может быть отдельной воронкой/Smart Process/account plan, если lifecycle и роли существенно отличаются. Не держать выигранную продажу бесконечно активной ради сопровождения.

## 6. Roles and Permissions

| Роль | Доступ CRM | Сделки | Клиенты | Аналитика | Настройки |
|---|---|---|---|---|---|
| Менеджер | Свой scope | Свои/участвующие | Свои accounts | Personal/team limited | Нет |
| Руководитель продаж | Отдел | Команда, approval/risk | Accounts отдела | Funnel/forecast/team | Controlled business settings |
| Директор | По утверждённому scope | Strategic/high-level | Strategic accounts | Executive dashboards | Нет operational admin по умолчанию |
| Маркетинг | Минимальный CRM context | Source/aggregate | Ограниченный доступ | Marketing attribution | Нет |
| Сопровождение | Linked won clients | Read/handover | Assigned accounts | Service/account metrics | Нет CRM admin |

Точная модель уровней и inheritance проверяется на целевом портале. Positive и negative tests выполняются отдельными пользователями через UI, search, direct URL, REST, export и BI.

## 7. Sales Automation

### Роботы, задачи и уведомления

| Trigger | Action | Guard | Failure control |
|---|---|---|---|
| Deal created | Назначить owner/первичную задачу | Territory/account rule | Fallback owner |
| Stage entered | Создать next-step task | Operation+stage key | No duplicate on re-entry |
| Next step overdue | Reminder/escalation | Deal still active | ROP report |
| High-value/risk | Notify manager/ROP | Threshold/access | Noise budget |
| No activity | Create review task | No future activity | Aging monitoring |

### Бизнес-процессы

- согласование скидки: threshold, margin context, approver, substitute, reject/rework;
- согласование крупной сделки: strategic risk, solution, finance and executive approval;
- договор: version, legal review, comments, approval result and escalation.

Automation не должна автоматически повышать вероятность или переводить стадию без подтверждённого business event.

## 8. Commercial Proposal Process

```text
Deal data + Product scope -> Proposal draft vN -> Internal review
 -> Approval -> Send -> Client response -> revise/accept/reject
```

| Этап | Контроль |
|---|---|
| Подготовка | Product rows, price source, scope, validity, template version |
| Согласование | Discount/terms threshold, approver, comments, decision |
| Отправка | Authorized channel, final version, timestamp |
| Результат | Viewed/response only if officially available; otherwise manager activity |
| Изменение | New version; previous proposal not overwritten silently |

Генерация, подпись, tracking и хранение документов зависят от доступного официального механизма или внешнего приложения. Нельзя обещать контроль просмотра без подтверждения.

## 9. Account Management

После продажи company остаётся клиентским аккаунтом, а каждая новая коммерческая opportunity создаётся отдельной сделкой.

| Область | Практика | Метрика |
|---|---|---|
| Развитие клиента | Account plan, stakeholder map, goals | Coverage/engagement |
| Повторные продажи | New deal linked to same company | Repeat revenue/conversion |
| Допродажи | Product gap and timing | Upsell pipeline |
| Задачи account manager | Business review, renewal, risk | On-time activities |
| Handover | Sales→delivery/support/account | Acceptance checklist |

Не создавать отдельную компанию для каждого продукта и не переоткрывать старую выигранную сделку.

## 10. Integration Architecture

| Интеграция | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| Сайт | Inquiry, company/contact, source/UTM | Backend→REST/official connector | Validation, dedup, reconciliation |
| Телефония | Call ID, direction, user, CRM link | Official telephony/connector/app | Synthetic call, provider boundary |
| Email | Communication/activity context | Official mail integration where available | Ownership, duplicate communication |
| Мессенджеры | Participant/channel/message context | Supported connector/application | Identity, rights, retention |
| ERP | Products, prices, orders, payments, external IDs | Middleware/official contract | Queue, idempotency, balance |
| BI | CRM/datasets/financial facts | BI Constructor or external BI as applicable | Freshness, formula, access |

```text
Channels -> Integration boundary -> Bitrix24 CRM
ERP master <-> Middleware/queue <-> Deals/products/status
BI <---------- governed datasets/metric contracts
```

Точные REST-методы проверяются в [API reference](https://apidocs.bitrix24.ru/api-reference/); телефония — по [официальной документации](https://helpdesk.bitrix24.ru/open/6450911/).

## 11. Analytics

| Отчёт | Основные показатели | Контроль качества |
|---|---|---|
| Воронка продаж | Count/amount/conversion by stage | Stage semantics and dates |
| Прогноз | Pipeline/weighted/commit by period | Probability governance |
| Менеджеры | Activities, response, conversion, revenue | Ownership and team periods |
| Длительность сделки | Time overall/by stage | Timezone and reopen rules |
| Причины проигрыша | Reason/competitor/stage | Required controlled reasons |
| Повторные продажи | Repeat deals/revenue/account coverage | Stable company identity |

BI и datasets зависят от тарифа и прав; источник — [официальная документация наборов данных](https://helpdesk.bitrix24.ru/open/24355590/). Forecast не считается достоверным без формальной probability/category model.

## 12. Testing Plan

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Создание клиента | Company/contact validation and dedup | Один account, корректные связи |
| 2 | Новая сделка | Company, contacts, source, owner | Opportunity создана один раз |
| 3 | Переход стадии | Entry/exit fields and permissions | Допустимый переход, один side effect |
| 4 | Запрещённый переход | Target role/empty required data | Переход отклонён без side effect |
| 5 | Согласование скидки | Threshold, approver, reject/rework | Решение и history сохранены |
| 6 | Крупная сделка | Additional approval route | Correct approvers/escalation |
| 7 | Договор | Version, legal review, result | Correct version linked to deal |
| 8 | Успешная продажа | Closure/handover conditions | Deal closed, handover created once |
| 9 | Повторная продажа | Existing company/new opportunity | Новая сделка без дубля компании |
| 10 | Ошибка ERP | Timeout/queue/retry/reconcile | Нет дубля, alert/manual review |
| 11 | Ошибка webhook | Duplicate/out-of-order | Idempotent business effect |
| 12 | Права менеджера | UI/direct URL/REST/export | Только разрешённый scope |
| 13 | Прогноз | Formula, probability, close date | Совпадает с контрольной выборкой |
| 14 | Телефонный звонок | Synthetic inbound/outbound and CRM link | Correct activity/account or conflict queue |

Тестовые данные синтетические; UAT выполняют Sales, Management, Marketing и Support owners.

## 13. Deployment Plan

```text
Audit -> Architecture -> Pilot configuration -> Data rehearsal
 -> Training/UAT -> Cutover -> Smoke -> Hypercare -> Optimize
```

| Этап | Результат | Gate |
|---|---|---|
| Аудит | AS IS, data quality, fields, automation, integrations | Scope sign-off |
| Настройка | Target entities, stages, rights, disabled/tested automation | Peer review |
| Перенос данных | Mapping, dry-run, sample, balance | Data owner acceptance |
| Обучение | Role-based playbook and exercises | Readiness confirmation |
| Запуск | Window, backup/checkpoint, cutover, smoke | Go/no-go |
| Hypercare | Monitoring, incident routing, fixes | Exit criteria |

Работающая воронка изменяется через параллельную настройку или пилот, если это возможно. On-Premise дополнительно требует server/PHP/DB/modules compatibility и restore test.

## 14. Monitoring

| Контроль | Сигнал | Действие |
|---|---|---|
| Просроченные задачи | Count/age/owner | Manager/ROP escalation |
| Зависшие сделки | Stage age, no next step/activity | Opportunity review |
| Активность менеджеров | Completed/planned meaningful activities | Coaching, not vanity count |
| Ошибки интеграций | Error rate, lag, backlog, unknown | Runbook/reconciliation |
| Дубли | New duplicate candidates | Data steward review |
| Forecast drift | Plan/forecast vs outcome | Model calibration |
| Automation | Missing/duplicate/error/duration | Automation owner |

Пороги устанавливаются по baseline и бизнес-риску; универсальные SLA не задаются.

## 15. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Потеря opportunity | Нет owner/next step | Потеря revenue | SLA/task/aging monitoring |
| 2 | Дубли компаний | Name-only matching | Разрыв account history | Stable IDs/dedup review |
| 3 | Дубли контактов | Email/phone normalization gap | Несогласованная коммуникация | Identity rules |
| 4 | Дубли сделок | Retry/event repeat | Искажённый pipeline | Idempotency key |
| 5 | Buying committee неполон | Один контакт в сделке | Неизвестен decision process | Stakeholder map |
| 6 | Роль контакта устарела | Глобальное поле | Ошибка influence model | Deal-context review |
| 7 | ЛПР определён субъективно | Нет evidence/governance | Неверная стратегия | Qualification criteria |
| 8 | Giant pipeline | Продажа и delivery смешаны | Сложная аналитика | Separate lifecycle |
| 9 | Слишком много воронок | Деление по менеджерам | Fragmentation | Roles/one lifecycle |
| 10 | Стадия без exit criteria | Название вместо состояния | Ложный forecast | State machine rules |
| 11 | Probability субъективна | Manual optimism | Неверный план | Governed model/calibration |
| 12 | Close date не обновляется | Нет review cadence | Forecast drift | Aging/forecast review |
| 13 | Next step отсутствует | Низкая дисциплина | Сделка зависает | Required next action |
| 14 | Activities vanity metric | Считается количество | Имитация работы | Outcome-linked activities |
| 15 | КП без версии | Файлы перезаписываются | Отправлено неверное предложение | Version control |
| 16 | Скидка без согласования | Нет threshold | Потеря маржи | Approval workflow |
| 17 | Approval зависает | Нет substitute/escalation | Срыв срока | SLA/delegation |
| 18 | Договор не связан со сделкой | Manual files | Нет traceability | Relation/document register |
| 19 | Старый договор используется | Нет version status | Legal risk | Approved version marker |
| 20 | Account manager не назначен | Handover gap | Нет развития клиента | Formal handover |
| 21 | Повторная продажа в старой сделке | Удобство | Потеря истории | New opportunity |
| 22 | Компания дублируется по продукту | Org design error | Fragmented account | Single company master |
| 23 | ERP и CRM оба master | Free bidirectional edits | Conflicts | Ownership by data type |
| 24 | Webhook утёк | URL in code/log | Unauthorized access | Secret store/revoke |
| 25 | Пропуск события | Event-only design | Data drift | Hybrid reconciliation |
| 26 | Retry storm | No classification/backoff | API degradation | Bounded retry |
| 27 | Телефония связала не тот account | Caller ambiguity | Data disclosure | Conflict/manual review |
| 28 | Мессенджер недоступен | App/provider dependency | Lost history | Availability/fallback check |
| 29 | BI показывает неверный forecast | Formula/data quality | Wrong decisions | Metric contract/control totals |
| 30 | Избыточные права менеджера | Convenience | Data leak/export | Least privilege tests |
| 31 | Маркетинг видит коммерческие условия | Access model gap | Confidentiality risk | Aggregated/limited access |
| 32 | Нет rollback | Success-only plan | Long outage | Tested rollback |
| 33 | Миграция без dry-run | Deadline | Data corruption | Sample/rehearsal |
| 34 | Cloud function unavailable | Tariff not verified | Scope failure | Early portal check |
| 35 | Box core modified | Shortcut | Broken updates | Supported extension |

## 16. ADR

### ADR-B2B-001: Использовать лиды или Deal-First

| Поле | Решение |
|---|---|
| Context | Есть inbound raw flow и outbound named accounts |
| Decision | Leads only for unqualified inbound; Deal-First for qualified named opportunity |
| Alternative | Все обращения как leads или deals |
| Consequences | Два entry paths с единым qualification contract |
| Review trigger | Изменение lead mix/process |

### ADR-B2B-002: Структура сделки

| Поле | Решение |
|---|---|
| Context | Одна company имеет несколько independent opportunities |
| Decision | Одна deal = одна коммерческая opportunity |
| Alternative | Deal per company forever |
| Consequences | Чистая история и repeat sales, больше entities |
| Review trigger | Изменяется unit of sale |

### ADR-B2B-003: Хранение клиентов

| Поле | Решение |
|---|---|
| Context | Company-centric model with many contacts |
| Decision | Company is account; contacts linked; no product-specific company duplicates |
| Alternative | Separate company per division/product |
| Consequences | Нужны stable identity and access rules |
| Review trigger | Юридически разные entities/ownership |

### ADR-B2B-004: Модель повторных продаж

| Поле | Решение |
|---|---|
| Context | Нужно сохранять историю выигранных продаж |
| Decision | New repeat/upsell deal linked to same company |
| Alternative | Reopen old deal |
| Consequences | Correct LTV/pipeline; duplicate guard required |
| Review trigger | Subscription/renewal model changes |

### ADR-B2B-005: Автоматизация согласований

| Поле | Решение |
|---|---|
| Context | Discounts, large deals and contracts require controlled decision |
| Decision | BP/approval flow where confirmed; robots create tasks/notifications |
| Alternative | Manager messages/manual email |
| Consequences | Auditability and SLA; needs substitute/escalation |
| Review trigger | Approval matrix/product capability changes |

### ADR-B2B-006: Договор как отдельный объект

| Поле | Решение |
|---|---|
| Context | Contract may have versions, terms and lifecycle independent from deal |
| Decision | Use document register/Smart Process only if lifecycle complexity justifies it |
| Alternative | File/link fields in deal |
| Consequences | More structure and permissions; feature verification required |
| Review trigger | Contract volume/renewals increase |

## 17. Типовые ошибки B2B внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Сделка используется как компания | Нет entity model | Дубли клиента | Company account + deals |
| 2 | Компания создаётся на каждый продукт | Удобство менеджера | Fragmented history | Single account |
| 3 | Название компании — unique key | Нет external ID | False duplicates | Stable identifiers |
| 4 | Контакт хранится текстом в сделке | Быстрая настройка | Нет communication history | Contact entity/relation |
| 5 | Должность равна buying role | Упрощение | Неверная stakeholder map | Separate role/influence |
| 6 | Один контакт считается ЛПР всегда | Глобальное поле | Wrong opportunity strategy | Deal-context role |
| 7 | Нет buying committee | Focus on one person | Deal risk hidden | Stakeholder checklist |
| 8 | Все обращения — лиды | Habit | Long qualification backlog | Entry criteria |
| 9 | Все обращения — сделки | Simplification | Polluted pipeline | Optional lead path |
| 10 | Одна сделка на всю историю клиента | Account/deal mixed | No repeat-sales analytics | One opportunity per deal |
| 11 | Выигранная сделка переоткрывается | Convenience | Broken close metrics | New repeat deal |
| 12 | Воронка для менеджера | Rights not designed | Configuration duplication | Roles/ownership |
| 13 | Воронка для отрасли | Analytics shortcut | Fragmentation | Industry field |
| 14 | Одна giant pipeline для sales/delivery | Want one view | Unmanageable stages | Separate lifecycles |
| 15 | Стадии названы действиями | No state design | Ambiguous progress | Exit criteria |
| 16 | Нет обязательных данных по стадиям | Low governance | False forecast | Stage requirements |
| 17 | Все поля обязательны сразу | Overcontrol | Garbage input | Progressive requirements |
| 18 | Probability задаётся на глаз | Optimism | Wrong forecast | Governed categories |
| 19 | Close date не reviewed | No cadence | Stale forecast | Forecast review |
| 20 | Нет next step | Weak discipline | Stalled deals | Mandatory action/date |
| 21 | Activity count = effectiveness | Easy metric | Gaming | Outcome metrics |
| 22 | Нет причины проигрыша | Avoidance | No learning | Controlled required reasons |
| 23 | Конкурент — свободный текст | Quick field | Bad analytics | Controlled reference |
| 24 | КП перезаписывается | No versioning | Wrong proposal | Versioned document |
| 25 | Цена в КП не связана с source | Manual edit | Margin errors | Product/master rules |
| 26 | Скидка согласуется в чате | Fast communication | No audit/SLA | Approval workflow |
| 27 | Нет substitute approver | Vacation | Blocked deal | Delegation/escalation |
| 28 | Timeout автоматически одобряет | Convenience | Uncontrolled risk | Escalate/manual decision |
| 29 | Договор хранится без связи | File share habit | No traceability | Deal relation/register |
| 30 | Старые версии договора не маркируются | No lifecycle | Legal mistake | Version/status |
| 31 | Handover не формализован | Sales-only focus | Client drop after win | Acceptance checklist |
| 32 | Account manager не owner | Org ambiguity | No development | Named role |
| 33 | Repeat sale создаёт duplicate company | Campaign import | Fragmented LTV | Identity/dedup |
| 34 | ERP и CRM freely edit prices | No master | Conflicts | Ownership by field |
| 35 | External IDs отсутствуют | UI-first | Duplicate sync | Cross-system IDs |
| 36 | REST method guessed | Analogy | Production errors | Official reference |
| 37 | HTTP 200 means success | Shallow validation | Hidden error | Payload/state validation |
| 38 | Timeout without reconciliation | Blind retry | Duplicates | Read-back before retry |
| 39 | No idempotency | Happy path | Duplicate entities | Operation key |
| 40 | Webhook in code/log | Copy/paste | Credential leak | Secret manager/revoke |
| 41 | Event delivery assumed perfect | Event-only | Silent drift | Reconciliation |
| 42 | Queue has no monitoring | Backend hidden | Backlog | Depth/age alerts |
| 43 | Telephony provider ignored | CRM-centric | Wrong diagnosis | Boundary runbook |
| 44 | Email/messenger capabilities assumed | App optimism | Missing function | Verify connector/app |
| 45 | BI built before data quality | Dashboard pressure | False KPI | DQ gate |
| 46 | Forecast formula undocumented | Spreadsheet legacy | Disputes | Metric contract |
| 47 | Manager tested as admin | Convenience | Hidden permissions bug | Target-role tests |
| 48 | Marketing gets full CRM export | No access model | Confidentiality breach | Aggregate rights |
| 49 | Production data used in tests | Convenience | Privacy breach | Synthetic fixtures |
| 50 | Migration without mapping | Deadline | Field/relationship loss | Mapping/rehearsal |
| 51 | Automation enabled during import | No cutover plan | Mass tasks/messages | Disabled load |
| 52 | No rollback | Optimism | Long outage | Tested reverse plan |
| 53 | No monitoring after launch | Project ends | Late incidents | Operational SLI/alerts |
| 54 | No support handover | Developer dependency | Slow recovery | Runbook/RACI |
| 55 | Cloud and Box treated identically | Generic design | Impossible solution | Explicit applicability |

## 18. Implementation Roadmap

| Этап | Результат | Минимальный scope | Критерий готовности |
|---|---|---|---|
| 1. Discovery | AS IS, data, stakeholders, integrations | Interviews/inventory/baseline | Scope approved |
| 2. Архитектура | TO BE, entity model, pipeline, ADR | Diagrams/matrices/risks | Sign-off |
| 3. Настройка CRM | Companies, contacts, deals, fields, rights | Pilot pipeline | Configuration tests |
| 4. Автоматизация | Tasks, reminders, approvals | Critical stages only | Repeat/error tests |
| 5. Интеграции | Site, ERP, communications, BI | Highest-value contracts | E2E/reconciliation |
| 6. Тестирование | Functional, regression, rights, UAT | Synthetic/pilot data | Acceptance signed |
| 7. Запуск | Migration, training, cutover, smoke | One team/pilot then rollout | Stable hypercare |
| 8. Оптимизация | Forecast calibration, DQ, monitoring | Evidence-based backlog | KPI improvement cycle |

### Официальные источники

| Источник | Использование |
|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | CRM, права, automation и Cloud-функции |
| [Документация REST API](https://apidocs.bitrix24.ru/) | REST, webhooks, OAuth и интеграции |
| [REST API reference](https://apidocs.bitrix24.ru/api-reference/) | Контракты методов/полей |
| [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) | Договоры/проекты при отдельном lifecycle |
| [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) | Варианты подключения |
| [Наборы данных BI](https://helpdesk.bitrix24.ru/open/24355590/) | Analytics datasets |
| [Документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php) | On-Premise applicability |

Если функция, приложение, событие, метод, перенос или ограничение не подтверждены официальным источником, положение отмечается **«требует проверки»** и не включается в обязательства перед заказчиком.
