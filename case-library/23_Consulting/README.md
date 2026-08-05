# Consulting and Professional Services CRM Architecture Case

Архитектурный кейс для синтетической консалтинговой компании, которая продаёт профессиональные услуги, формирует проектные команды, управляет экспертами, документами, договорами и результатами проектов. Документ задаёт референсную архитектуру Bitrix24, а не готовую конфигурацию, методологию конкретной практики, юридическое заключение или замену ERP, ЭДО/DMS и специализированной PSA/resource-management системы.

> **Граница данных:** реальные клиенты, эксперты, сотрудники, проекты, методики, договоры, бюджеты, счета, результаты обследований, телефоны, email, CRM ID, webhook URL и токены не используются. Все примеры обезличены и синтетические.

> **Архитектурная граница:** Bitrix24 отвечает за CRM, коммерческие возможности, координацию delivery, задачи, согласования и клиентские коммуникации. 1С/ERP остаётся master счетов, оплат, затрат и бухгалтерских фактов; ЭДО/DMS — authoritative documents; HR/PSA/resource system — компетенций, capacity и timesheet, если такая система утверждена. CRM хранит необходимые проекции и ссылки.

Обозначения:

- **факт Bitrix24** — подтверждается официальным источником из knowledge/SOURCES.md;
- **решение архитектора** — рекомендация, утверждаемая на Discovery;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа или редакции, версии, прав и приложения;
- **внешний факт** — данные, подтверждаемые профильной master-системой.

## 1. Business Context

### Модель консалтингового бизнеса

Консалтинговая компания продаёт экспертизу и измеримый результат. До договора выполняется диагностика, формируется scope, оцениваются трудоёмкость, команда, зависимости и риски. После договора коммерческая opportunity передаётся в delivery-контур, где управляются этапы, задачи, изменения, качество и acceptance.

| Тип услуги | Модель исполнения | Коммерческая единица | Основной риск |
|---|---|---|---|
| Диагностика и аудит | Fixed scope | Deal + Project | Неполные входные данные |
| Стратегический консалтинг | Milestones/deliverables | Deal + Project | Неизмеримый результат |
| Внедрение | Phased delivery | Deal + Project/Workstreams | Scope creep |
| Сопровождение | Retainer/service period | Contract + recurring Project/Case | Смешение support и project |
| Обучение | Program/session | Deal + Project/Event | Неопределённые критерии освоения |
| Экспертная консультация | Time/material or package | Deal + Assignment | Нет capacity/evidence |

Цели CRM:

- единая история клиента, контактов и возможностей;
- управляемая диагностика и квалификация;
- версия предложения, scope и допущений;
- безопасная передача продажи в delivery;
- планирование экспертов на основании компетенций и capacity;
- контроль сроков, качества, изменений и acceptance;
- повторные проекты без перезаписи истории;
- достоверная прибыльность и загрузка после сверки с master-системами.

**Решение архитектора:** до настройки подтверждаются Cloud/On-Premise, тариф, роли, практики, billing model, действующие сущности, проекты, документы, integrations, volumes, допустимый простой and test environment.

## 2. AS IS Problems

| Проблема | Симптом | Причина | Целевой контроль |
|---|---|---|---|
| Нет единой базы клиентов | Дубли компаний и контактов | Нет account model | Stable IDs and dedup |
| Теряется история | Переписка и решения в личных каналах | Нет activity/document policy | Governed interaction history |
| Проекты непрозрачны | Статус известен только PM | Нет milestones/owner | Project lifecycle |
| Загрузка экспертов неизвестна | Назначения конфликтуют | Нет capacity source | Resource plan and conflict check |
| Scope меняется незаметно | Комментарий заменяет baseline | Нет change control | Versioned scope/change request |
| Документы расходятся | Несколько финальных версий | CRM используется как DMS | Authoritative reference |
| Финансы недостоверны | Deal amount считается прибылью | Нет finance reconciliation | ERP/1С facts |
| Аналитика слабая | Нельзя сравнить проекты | Нет grain/metric contracts | Governed dataset |

AS IS фиксирует sales channels, client hierarchy, service catalog, qualification, estimation, contracts, project delivery, expert capacity, time/cost, documents, integrations, reports and exception paths.

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Lead | Неквалифицированный запрос, если lead model утверждена | Bitrix24 |
| CRM | Contact | Представитель клиента или партнёра | Bitrix24/MDM |
| CRM | Company | Клиент, партнёр или подрядчик | Bitrix24/ERP/MDM |
| CRM | Deal | Отдельная коммерческая возможность | Bitrix24 |
| CRM/Smart Process | Request | Клиентский вопрос вне отдельной продажи | Bitrix24 |
| Smart Process | Consulting Project | Delivery lifecycle, scope and milestones | Bitrix24/PSA |
| Smart Process | Service | Governed service offering/version projection | Product/ERP/Bitrix24 |
| Smart Process | Expert Profile | Skills and availability projection | HR/PSA |
| Smart Process | Contract | Approval and authoritative reference | DMS/ЭДО |
| Smart Process | Proposal | Versioned scope, assumptions and validity | Bitrix24/DMS |
| Smart Process | Client Task | Клиентское действие или dependency | Bitrix24/project system |
| Tasks | Task | Конкретная исполнимая работа | Bitrix24 |

~~~text
Website / Referral / Call -> Lead or Request -> Contact + Company -> Deal
                                                        |
                                             Proposal -> Contract
                                                        |
                                                        v
                                             Consulting Project
                                              /       |       \
                                      Workstreams   Experts   Client Tasks
                                           |          |          |
                                      Deliverables  Capacity  Dependencies
                                           \          |          /
                                             Acceptance -> Support / New Deal
~~~

### Правила целостности

1. Deal описывает sale, Consulting Project — delivery, Task — работу, Deliverable — результат.
2. Expert Profile не заменяет User account и HR record; хранит разрешённую проекцию навыков и availability.
3. Proposal, Contract and Deliverable имеют version, owner, status and authoritative reference.
4. Revenue, invoice, payment, cost and margin — разные финансовые факты.
5. Service Catalog version не переписывает условия уже проданного проекта.
6. Integration projection хранит source system, external ID, source timestamp, sync status and correlation ID.
7. Повторное событие не создаёт дубли проекта, задач, документов или уведомлений.

**Cloud:** нет доступа к ядру, серверному PHP и базе данных; сложные integrations and advanced resource calculations размещаются во внешнем контуре. **On-Premise:** дополнительно проектируются server, PHP, DB, files, cron, backup, updates, performance and security; ядро не изменяется.

## 4. Customer Journey

~~~text
Первичный контакт -> Диагностика -> Предложение -> Договор
 -> Проект -> Результат -> Поддержка -> Повторный проект

Exception paths:
No-fit / no capacity / proposal rejected / scope change / pause / termination
 -> structured decision -> preserved evidence -> account follow-up
~~~

| Этап | Цель | Обязательный результат | Gate |
|---|---|---|---|
| Первичный контакт | Зафиксировать account and need | Owner, source, next step | Dedup |
| Диагностика | Уточнить problem, outcome and constraints | Diagnostic brief | Stakeholder/access readiness |
| Предложение | Зафиксировать scope, effort and assumptions | Approved proposal version | Capacity and review |
| Договор | Подтвердить commitments | Contract reference/status | Authority and readiness |
| Проект | Исполнить agreed scope | Milestones, tasks, risks | Handover accepted |
| Результат | Передать deliverables | Acceptance/evidence | Quality and source version |
| Поддержка | Обработать agreed follow-up | Case/task and service boundary | Entitlement |
| Повторный проект | Создать новую opportunity | New Deal linked to account | No overwrite/duplicate |

Journey не заканчивается успешной стадией Deal. Project delivery, acceptance, invoicing, payment and satisfaction имеют отдельные источники, владельцев и временные точки.

## 5. Consulting Data Model

| Сущность | Основные данные | Связи | Master |
|---|---|---|---|
| Клиент | Industry, segment, account owner, needs, project history | Contacts, deals, projects | CRM/MDM |
| Проект | Goal, scope version, dates, team, budget projection, status | Deal, contract, tasks, deliverables | Bitrix24/PSA |
| Эксперт | Skills, level, location/timezone, capacity projection | Assignments, projects | HR/PSA |
| Услуга | Practice, package, unit, price/version, constraints | Proposals, deals, projects | ERP/product owner |
| Предложение | Version, scope, assumptions, effort, validity | Deal, services, approvals | Bitrix24/DMS |
| Договор | External ID, version, status, effective dates | Company, Deal, Project | DMS/ЭДО |
| Deliverable | Type, version, owner, acceptance | Project, tasks, document ref | DMS/project repository |
| Client Task | Requested input, owner, due date, status | Project/milestone | Bitrix24 |

~~~text
Company 1 ---- N Contact
   |
   +---- N Deal ---- N Proposal Version ---- 1 Contract Reference
                         |
                         v
                  Consulting Project 1 ---- N Deliverable
                         |     \
                         |      +---- N Client Task
                         |
                         +---- M Expert Assignment M ---- 1 Expert Profile
~~~

### Идентичность и финансовые grains

- one client can have several business units without automatic duplicate Company;
- one Deal produces one or several Projects only by approved decomposition;
- one Expert Assignment links person, project, role, period and planned capacity;
- planned effort, approved budget, booked revenue, invoiced amount, paid amount and actual cost are separate;
- project history is immutable; repeat work creates a new Deal/Project relation;
- fields have type, owner, source, sensitivity, required stage and quality rule.

## 6. Sales Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новый контакт | Зарегистрировать demand | Source, account/contact, owner | Ack/task/dedup |
| Квалификация | Проверить fit and authority | Need, stakeholders, timing, next step | SLA/reminder |
| Диагностика | Определить problem and desired outcome | Brief, constraints, inputs | Expert task/checklist |
| Подготовка предложения | Сформировать scope and estimate | Service version, effort, assumptions | Review tasks |
| Согласование | Утвердить version, price and terms | Decision makers, changes, approvals | Approval route |
| Договор | Зафиксировать commitments | DMS ref/status, dates, readiness | Completeness gate |
| Проектирование | Детализировать delivery plan | Project, milestones, RACI | Handover task set |
| Реализация | Контролировать commercial delivery | Progress, risks, changes | Milestone control |
| Завершение | Подтвердить acceptance/outcome | Deliverables, acceptance, finance refs | Closure/account task |

~~~text
Новый контакт -> Квалификация -> Диагностика -> Предложение
 -> Согласование -> Договор -> Проектирование -> Реализация -> Завершение

Closed-lost -> Structured reason -> Account follow-up
Scope change -> Impact assessment -> Approved change / new Deal
~~~

Проектирование и реализация в Deal являются коммерческими контрольными стадиями. Детальный delivery lifecycle ведётся в Consulting Project, иначе pipeline превращается в план задач и искажает sales analytics.

## 7. Project Management Architecture

| Проектный объект | Назначение | Обязательные данные | Gate |
|---|---|---|---|
| Project | Delivery container | Goal, scope version, owner, dates | Handover accepted |
| Phase/Workstream | Управляемая часть scope | Owner, milestones, dependencies | Plan approved |
| Milestone | Контроль результата | Date, deliverable, acceptance | Evidence |
| Task | Конкретная работа | Assignee, deadline, result | Completion criteria |
| Deliverable | Передаваемый результат | Version, owner, status | Quality/acceptance |
| Risk/Issue | Неопределённость или проблема | Owner, impact, action | Review/escalation |
| Change Request | Изменение baseline | Reason, impact, version, approval | No work before decision |

~~~text
Contract / Approved Proposal -> Handover -> Project Baseline
 -> Workstreams -> Tasks -> Deliverables -> Quality Review -> Acceptance
                         \-> Risk / Issue / Change Control -> Replan
~~~

Проект закрывается после agreed acceptance and open-item decision, а не только после закрытия задач. Baseline, scope, dates and budget projections не перезаписываются без change record.

## 8. Expert Management

| Область | Данные | Master/контроль |
|---|---|---|
| Эксперт | User relation, role, practice, active status | HR |
| Компетенции | Skill, level, evidence date, owner | HR/PSA |
| Capacity | Calendar, allocation, absence projection | HR/PSA/resource system |
| Назначение | Project, role, period, planned effort | Project/resource owner |
| Факт времени | Approved timesheet/effort if used | PSA/time system |
| Конфликт | Overlap, missing skill, unavailable period | Resource control |

~~~text
Project Demand -> Required Skills / Period / Effort
 -> Candidate Capacity Check -> Approval -> Expert Assignment
 -> Task/Deliverable Work -> Actuals Source -> Variance Review
~~~

Smart Process Expert Profile применяется как управляемая CRM-проекция, но не дублирует кадровое досье. Рейтинг эксперта не строится только по скорости или billable utilization: учитываются complexity, role, quality, non-billable duties and capacity context.

## 9. Service Catalog

| Элемент | Содержание | Контроль |
|---|---|---|
| Practice | Направление экспертизы | Owner and lifecycle |
| Service | Outcome, scope, exclusions, prerequisites | Effective version |
| Package | Included deliverables, limits and options | Version/validity |
| Pricing model | Fixed, time/material, retainer or approved hybrid | Finance approval |
| Unit | Hour, day, milestone, package where applicable | Consistent UOM |
| Rate/price | Currency, amount, effective dates | ERP/approved source |
| Template | Proposal/project checklist | Version and applicability |

~~~text
Practice -> Service Version -> Package / Pricing Model
 -> Proposal Line -> Deal -> Project Scope Snapshot
~~~

Проданная услуга сохраняет snapshot/version. Обновление каталога не меняет прошлые предложения и действующие проекты. Стоимость, скидка, налог и валюта подтверждаются финансовым владельцем.

## 10. Document Architecture

| Документ | CRM projection | Master | Gate |
|---|---|---|---|
| КП | Version, scope summary, validity, protected link | DMS/Bitrix24 by ADR | Commercial approval |
| Договор | ID, version, status, dates, link | ЭДО/DMS | Authorized signature/status |
| Отчёт | Deliverable metadata, version, acceptance | DMS/project repository | Quality/client acceptance |
| Акт | ID, status, period, link | ЭДО/1С/DMS | Finance/legal confirmation |
| Счёт | Number, amount, currency, status projection | 1С/ERP | Authoritative status |
| Внутренний документ | Class, owner, version, access | DMS | Confidentiality |

~~~text
Proposal Version -> Contract Reference -> Project Deliverable -> Acceptance
                                              |
Invoice / Act / Payment references <- ERP / 1C / EDO / DMS
~~~

Bitrix24 не объявляется системой юридически значимой подписи или бухгалтерским master. Документы, versions, approvals, retention and access are governed by authorized systems and policies.

## 11. Rights Model

| Роль | Клиенты | Проекты | Документы | Финансы | Аналитика |
|---|---|---|---|---|---|
| Консультант | Assigned minimum | Assigned workstreams | Working refs by role | Нет/минимум | Own delivery |
| Руководитель проекта | Project accounts | Full assigned project | Project scope/docs | Approved project projection | Project |
| Эксперт | Need-to-know | Assigned tasks/deliverables | Expert inputs only | Нет | Own assignments |
| Менеджер продаж | Own/team accounts | Commercial summary | Proposal/contract refs | Deal projection | Sales |
| Бухгалтер | Counterparty minimum | Billing context | Financial docs | Invoices/payments/costs | Finance |
| Директор | Governed portfolio | Portfolio | Approved scope | Executive/approved | Executive |
| Администратор | Нет automatic business entitlement | Configuration | Нет automatic content right | Нет automatic finance right | Dataset config as needed |

Принципы: deny-by-default, least privilege, client/project/team scopes, segregation of sales/delivery/finance duties, named accounts, temporary access expiry, recertification, export controls and negative testing.

Внешний доступ клиента не предполагается как автоматически доступная функция: портал, приложение, extranet or protected delivery channel выбираются после проверки официальной документации, тарифа и security model.

## 12. Automation Architecture

| Автоматизация | Trigger | Действие | Защита |
|---|---|---|---|
| Lead/Deal SLA | First valid intake | Owner/task/ack | Source key/idempotency |
| Diagnostic task | Qualified opportunity | Assign approved expert | Capacity/role check |
| Proposal review | Specific version ready | Approval route/tasks | Version lock |
| Handover | Contract/readiness gates passed | Create/link Project and task set | Unique Deal-project key |
| Milestone control | Threshold reached | Reminder/escalation | One event/threshold |
| Client dependency | Due date approaching | Owner/client follow-up task | Current status/version |
| Quality review | Deliverable ready | Reviewer task | No self-approval by rule |
| Integration failure | Error/timeout | Queue/alert/bounded retry | Correlation/DLQ |

~~~text
Event -> Validate source / version / rights / capacity -> Idempotency
 -> Bounded action -> Audit result -> Exception queue / escalation
~~~

Робот не подтверждает компетенцию, contract signature, deliverable acceptance, invoice, payment, actual cost or timesheet без authoritative/human gate. Массовые операции используют dry-run, synthetic sample, batch report and rollback.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| Bitrix24 CRM | Accounts, opportunities, projects and tasks | Native configuration/REST as verified | Rights, IDs, audit |
| 1С/ERP | Counterparties, services, invoices, acts, payments, costs | Supported integration/middleware | Stable IDs, amounts, currency, reconcile |
| ЭДО/DMS | Proposal/contract/report/act IDs and statuses | Vendor API/middleware | Version, signature/status, access |
| Телефония | Call events and CRM relation | Bitrix24 telephony/SIP/REST app | Rights, missed calls, correlation |
| BI/DWH | Governed datasets and aggregates | BI Constructor or approved ETL | Grain, access, DQ, freshness |
| Сайт | Leads, requests, forms | CRM form/approved app/webhook/REST | Validation, consent, dedup |
| HR/PSA | Expert skills, capacity, assignments, actuals | Documented API/middleware | Person mapping, period, reconcile |

~~~text
Source -> AuthN/AuthZ -> Minimize -> Validate schema/version
 -> Map stable IDs -> Idempotency -> Bitrix24 projection
 -> Result log -> Bounded retry / quarantine -> Reconciliation
~~~

Готовый коннектор к конкретной PSA, DMS, ЭДО or HR configuration не предполагается. REST methods, scopes, limits and applications проверяются по official docs and target portal. Secrets are outside Git and payload logs.

## 14. Analytics

| Отчёт | Grain | Показатели | Контроль качества |
|---|---|---|---|
| Продажи услуг | Deal/service/period | Pipeline, conversion, cycle, booked value | Stage/service version |
| Прибыльность проектов | Project/period | Revenue, direct cost, margin | ERP/1С reconcile |
| Загрузка экспертов | Assignment/person/period | Planned, available, actual where approved | Capacity calendar/source |
| Эффективность менеджеров | Deal/activity cohort | Conversion, cycle, next-step discipline | Segment/territory context |
| Удовлетворённость клиентов | Accepted project/survey sample | Score, response, themes | Method/sample/no forced inference |
| Delivery quality | Project/deliverable | On-time, rework, acceptance | Scope/version context |
| Повторные проекты | Company + accepted Projects | Repeat rate, interval, cohort | Stable identity |

~~~text
CRM + Project/PSA + ERP/1C + DMS -> Conformed Client / Project / Expert IDs
 -> DQ / Reconciliation -> Metric Layer -> Role-based Dashboards
~~~

Каждая метрика имеет owner, formula, grain, period, timezone, currency, utilization definition, exclusions, source, freshness, access and control total. Deal, Project, Task, Assignment, Time Entry, Invoice and Payment are separate grains.

## 15. Security Architecture

| Контроль | Реализация | Проверка |
|---|---|---|
| Client confidentiality | Project/client scopes | Cross-client negative test |
| Need-to-know | Role-based fields/docs | Unauthorized role test |
| External access | Separate approved channel and expiry | Revocation test |
| Documents | Protected DMS links and versions | Expired/revoked access |
| Commercial data | Sales/finance segregation | Cross-role matrix |
| Integrations | Least-privilege credentials and rotation | Scope/revoke drill |
| Logs | Masked payload, correlation ID | Secret/PII scan |
| Export | Approval/minimum/protected target | Export test |
| Audit | CRM history plus integration/security logs | End-to-end trace |

**Cloud:** complex integration, secret management and extended logs remain external; no server/PHP/DB access. **On-Premise:** additionally secure OS, web server, PHP, DB, modules, files, cron, backup and logs; core is not modified.

Материалы разных клиентов не смешиваются в общих задачах, комментариях, shared links or BI datasets. Confidentiality labels, retention, deletion and legal hold are approved outside CRM.

## 16. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Новый lead с сайта | Valid synthetic payload | One Lead/Deal, source ID and owner |
| 2 | Duplicate intake | Same external ID | No duplicate |
| 3 | B2B account | Company with two contacts | Correct relations |
| 4 | Qualification no-fit | Approved reason | Closed-lost structured |
| 5 | Diagnostic assignment | Required skill and capacity | Eligible expert assigned |
| 6 | Capacity conflict | Expert already allocated | Conflict detected/no silent overbooking |
| 7 | Proposal versions | v1 rejected, v2 approved | Only v2 used |
| 8 | Contract handover | Readiness complete | One Project/task set created |
| 9 | Handover incomplete | Missing scope input | Project creation blocked/routed |
| 10 | Scope change | Approved change request | Baseline/version preserved |
| 11 | Deliverable rejection | Quality issue | Rework path/history preserved |
| 12 | Project acceptance | Current deliverable accepted | Correct closure/finance follow-up |
| 13 | Invoice mismatch | CRM vs 1С synthetic totals | Reconciliation incident |
| 14 | Cross-client rights | Consultant from another project | Access denied |
| 15 | Expired external access | Access period ended | Link/access revoked |
| 16 | Integration timeout | Accepted then timeout | Idempotent bounded retry |
| 17 | Out-of-order actuals | Older source version | New state preserved |
| 18 | Repeat project | Accepted prior project | New Deal/Project, old history intact |
| 19 | Migration dry-run | Synthetic sample | Report without production write |
| 20 | Rollback | Failed release | Previous state restored |

Tests use synthetic clients, experts, proposals, projects, documents, invoices and payments. Positive, negative, repeated, rights, capacity, versioning, failure, recovery and reconciliation paths are required.

## 17. Deployment

| Этап | Результат | Gate |
|---|---|---|
| Discovery | AS IS, services, systems, roles, volumes | Owners/security approval |
| Архитектура | TO BE, masters, grains, rights, ADR, NFR | Architecture review |
| Настройка | Isolated CRM/Smart Processes/automation | Configuration review |
| Миграция | Inventory, mapping, clean, dry-run | Reconciliation/sign-off |
| Интеграции | Contracts, secrets, retry, monitoring | Failure/security/replay tests |
| Обучение | Role-based synthetic scenarios | UAT |
| Запуск | Cutover, communication, fallback, rollback | Go/no-go |
| Stabilization | Hypercare, baseline and incidents | Exit criteria |

~~~text
Inventory -> Dependency / Rights Review -> Test Configuration
 -> Synthetic UAT -> Migration Dry-run -> Go/No-Go
 -> Release -> Smoke / Reconciliation -> Hypercare or Rollback
~~~

Fields, stages, roles, automation and integrations are not removed or renamed before dependency analysis, backup/export, pilot and approved rollback. Existing delivery processes use parallel pilot where possible.

## 18. Monitoring

| Сигнал | Правило | Действие | Владелец |
|---|---|---|---|
| Проект без owner | Empty/inactive responsible | Assign/escalate | PMO |
| Просроченный milestone | Approved date breached | Review/replan/escalate | Project manager |
| Expert over-allocation | Planned capacity threshold | Resource review | Practice lead |
| Client task overdue | Dependency deadline | Follow-up/risk | Project manager |
| Deliverable pending review | Review SLA | Assign/escalate | Quality owner |
| Integration error | Error/retry/DLQ | Investigate/replay | Integration support |
| Finance mismatch | Control totals differ | Incident/reconcile | Finance owner |
| Access anomaly | Unexpected view/export | Security incident | Security owner |
| BI stale | Freshness threshold | Mark stale/investigate | BI owner |

Alert includes system, object reference, correlation ID, time, severity, owner and runbook; without confidential deliverable content, contract body, secret, payment details or unnecessary PII.

## 19. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Потеря заявки | Нет ack | Lost sale | Intake reconciliation |
| 2 | Duplicate Lead/Deal | Multiple channels/retry | Double work | Source key/dedup |
| 3 | False company merge | Weak identifiers | Client disclosure | Stable IDs/review |
| 4 | Business units duplicated | Flat model | Fragmented history | Account hierarchy |
| 5 | Lead used for support | Universal pattern | Mixed lifecycle | Classification |
| 6 | One Deal for all projects | Convenience | No repeat analytics | Deal per opportunity |
| 7 | Deal equals Project | One pipeline | Sales/delivery conflict | Separate lifecycle |
| 8 | Task equals deliverable | Completion shortcut | No quality evidence | Deliverable gate |
| 9 | Proposal overwritten | One document | Scope dispute | Immutable versions |
| 10 | Assumptions omitted | Sales pressure | Unbounded scope | Proposal checklist |
| 11 | Capacity not checked | No resource source | Missed deadlines | Capacity gate |
| 12 | Skill self-declared/stale | No owner/evidence | Wrong assignment | HR/PSA governance |
| 13 | Expert profile duplicates HR | Overmodeling | Drift/privacy | Minimal projection |
| 14 | Over-allocation hidden | Parallel sheets | Burnout/delay | Resource reconciliation |
| 15 | Timesheet guessed in CRM | No actuals source | False margin | PSA/time master |
| 16 | Scope change informal | Chat decision | Margin loss | Change control |
| 17 | Baseline overwritten | No versioning | No audit | Versioned plan |
| 18 | Client dependency untracked | Internal-only plan | Project delay | Client Task |
| 19 | Deliverable closes without acceptance | Task trusted | Dispute | Acceptance gate |
| 20 | Contract status manual | No DMS sync | Unauthorized work | DMS authority |
| 21 | Invoice equals Deal amount | CRM-only finance | False receivable | ERP/1С source |
| 22 | Payment manual | Convenience | Financial error | Bank/ERP source |
| 23 | Cost absent | Revenue focus | False profitability | Finance costs |
| 24 | Service catalog overwritten | No effective version | Historical mismatch | Version snapshot |
| 25 | Price free-text | No catalog owner | Inconsistent offer | Governed pricing |
| 26 | Client files copied broadly | Convenience | Confidentiality breach | Protected repository |
| 27 | Cross-client task links | Shared workspace | Data leak | Project isolation |
| 28 | External access permanent | No expiry | Residual access | Time-bound access |
| 29 | Admin is content super-reader | Role confusion | Uncontrolled access | Separate privileges |
| 30 | Export unrestricted | Defaults | Mass data leak | Approval/minimum |
| 31 | Secret in Git | Real example | Compromise | Secret store/rotation |
| 32 | Broad integration token | One account | Large blast radius | Least privilege |
| 33 | Raw payload logged | Debugging | Client data leak | Masking |
| 34 | Infinite retry | No DLQ | Load/outage | Bounded backoff |
| 35 | Retry duplicates Project | No idempotency | Double delivery | Business key |
| 36 | Out-of-order event | Async ignored | Old status wins | Version/order guard |
| 37 | Schema drift | Source change | Corrupt data | Contract validation |
| 38 | No correlation ID | Fragmented logs | Slow RCA | End-to-end trace |
| 39 | Free bidirectional edit | No master | Ping-pong | Field ownership |
| 40 | Connector assumed | Unverified feature | Blocked design | Official verification |
| 41 | Migration without inventory | Unknown scope | Missing history | Source report |
| 42 | Import without dry-run | Time pressure | Mass corruption | Sample/report/rollback |
| 43 | Real client data in UAT | Production copy | Confidentiality breach | Synthetic data |
| 44 | No reconciliation | HTTP success trusted | Silent loss | Counts/control totals |
| 45 | BI mixes grains | Dashboard-first | False KPI | Metric contract |
| 46 | Utilization formula undefined | Management shortcut | Wrong decisions | Governed formula |
| 47 | Satisfaction sample biased | Only happy clients | False quality | Methodology/sample |
| 48 | Expert ranking ignores complexity | Simplistic KPI | Harmful management | Context/capacity |
| 49 | Peak staffing not tested | Average plan | Delivery collapse | Scenario/load planning |
| 50 | Tariff not checked | Assumption | Feature unavailable | Portal assessment |
| 51 | Cloud assumes DB/custom runtime | On-Premise habit | Impossible design | External boundary |
| 52 | On-Premise core modified | Quick fix | Upgrade/security failure | Extensions only |
| 53 | Release lacks rights tests | Happy path | Hidden exposure | Negative suite |
| 54 | Rollback untested | Paper plan | Long outage | Restore drill |
| 55 | DR ignores DMS/ERP/PSA | CRM-only recovery | Service not restored | End-to-end DR |

## 20. ADR

| ADR | Решение | Альтернатива | Обоснование | Последствие |
|---|---|---|---|---|
| ADR-CONSLT-001 | Deal and Consulting Project are separate | One pipeline | Sales and delivery have different owners/grains | Handover relation |
| ADR-CONSLT-002 | Service Catalog is effective-versioned | Free-text services | Repeatable scope/pricing governance | Catalog owner required |
| ADR-CONSLT-003 | Expert Profile is minimal HR/PSA projection | Duplicate employee dossier | Prevents drift and excess data | Integration/freshness |
| ADR-CONSLT-004 | Proposal, Contract and Deliverable are versioned refs | File copies in comments | Scope and acceptance evidence | DMS/repository dependency |
| ADR-CONSLT-005 | Rights are client/project/purpose scoped | Organization-wide read | Confidentiality and least privilege | More negative tests |
| ADR-CONSLT-006 | Integrations use governed contracts/middleware | Point-to-point webhooks | Idempotency, security and monitoring | Integration owner |
| ADR-CONSLT-007 | Finance facts come from ERP/1С | Deal amount as finance | Reliable invoice/payment/margin | Reconciliation |
| ADR-CONSLT-008 | BI uses conformed project/expert/finance grains | Raw CRM counters | Explainable utilization/profitability | Metric/DQ ownership |

Each ADR has owner, status proposed/accepted/superseded, date, context, alternatives, consequences, portal/version and evidence.

## 21. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Нет Discovery | Сразу настраивают CRM | Wrong scope | AS IS/stakeholders |
| 2 | Cloud/On-Premise неизвестен | Template design | Unsupported solution | Capability matrix |
| 3 | Тариф не проверен | Assumption | Missing function | Verify target portal |
| 4 | Нет process owners | IT-only project | No decisions | RACI |
| 5 | Все входы — Leads | Universal lead | Support mixed with sales | Classification |
| 6 | Leads disabled without ADR | Fashion | Unqualified demand mixed | Lead decision |
| 7 | Contact/Company mixed | Flat client record | Broken B2B relations | Native entities |
| 8 | Business unit becomes duplicate Company | No hierarchy | Fragmented history | Account model |
| 9 | Name is integration key | No external ID | Wrong match | Stable IDs |
| 10 | One Deal for all client work | Convenience | No sales cycles | Deal per opportunity |
| 11 | Deal equals Consulting Project | One object easier | Sales/delivery conflict | Separate lifecycles |
| 12 | Project is a long task | Under-modeling | No phases/risks/acceptance | Project entity |
| 13 | Task equals deliverable | Checkbox thinking | No result quality | Deliverable object/gate |
| 14 | Expert is only free-text field | Quick assignment | No capacity/history | Expert relation |
| 15 | Expert profile copies HR dossier | Overintegration | Privacy/drift | Minimal projection |
| 16 | Service is free-text | No catalog | Inconsistent scope | Versioned catalog |
| 17 | Service price has no currency/version | Simplification | Wrong proposal | Governed price |
| 18 | Old catalog overwrites sold scope | No snapshot | Historical mismatch | Effective versions |
| 19 | Stages describe tasks | Every action as stage | Ambiguity | State-based stages |
| 20 | Too many stages | Micromanagement | Low adoption | Tasks/checklists |
| 21 | Mandatory fields too early | Maximum control | Fake values | Stage requiredness |
| 22 | No next step | Stage-only control | Stalled sale | Activity/task policy |
| 23 | Diagnostic brief in comments | No structure | Scope gaps | Versioned brief |
| 24 | Proposal overwritten | One file | Commercial dispute | Immutable versions |
| 25 | Assumptions not recorded | Sales optimism | Scope creep | Proposal checklist |
| 26 | Estimate has no owner | Collective input | Uncontrolled price | Approval owner |
| 27 | Capacity not checked before sale | Revenue pressure | Missed delivery | Capacity gate |
| 28 | Contract stage set manually | No DMS source | False readiness | Authoritative status |
| 29 | Handover has no acceptance | Sales closes early | Lost requirements | Handover checklist |
| 30 | Project created twice | Retry/re-entry | Double delivery | Unique key |
| 31 | Baseline changes silently | No change control | No plan/fact | Versioned baseline |
| 32 | Scope change only in chat | Informality | Margin loss | Change Request |
| 33 | Client dependencies absent | Internal focus | Delays blamed wrongly | Client Task |
| 34 | Project closes when tasks close | Completion shortcut | No acceptance | Closure gate |
| 35 | Deliverable version not stored | File-name process | Wrong delivery | Version ID |
| 36 | Quality reviewer is author by default | No segregation | Weak control | Review role |
| 37 | Robot duplicates tasks | Re-entry ignored | Fake load | Idempotency marker |
| 38 | Robots loop stages | Mutual rules | Automation storm | Loop guard |
| 39 | Robot approves competence | Overautomation | Wrong assignment | Human/HR gate |
| 40 | Robot confirms acceptance | Notification equals result | Dispute | Client/owner gate |
| 41 | Capacity stored in percent without calendar | Simplistic field | False availability | Period/capacity model |
| 42 | Planned and actual effort mixed | One field | No variance | Separate grains |
| 43 | Timesheet entered twice | Two masters | Drift | One source |
| 44 | Utilization ignores non-billable duties | KPI pressure | Overload | Governed formula |
| 45 | Expert ranking ignores complexity | Simplistic BI | Harmful decisions | Context |
| 46 | KP stored as uncontrolled copies | Email workflow | Version conflict | DMS reference |
| 47 | Contract stored in open task | Convenience | Data leak | Protected DMS |
| 48 | Act status manual | No 1С/ЭДО sync | False finance | Source status |
| 49 | Invoice equals Deal amount | CRM-only finance | Wrong receivable | ERP source |
| 50 | Payment marked by sales | Convenience | Financial error | Finance authority |
| 51 | Margin uses no actual costs | Revenue focus | False profitability | Reconciled costs |
| 52 | Consultant sees all clients | Broad role | Confidentiality breach | Project scope |
| 53 | Sales sees all deliverables | Shared record | Excess access | Commercial summary |
| 54 | Expert sees finance | Common rights | Excess disclosure | Functional minimum |
| 55 | Admin equals business reader | Role confusion | Uncontrolled access | Separate privileges |
| 56 | External access never expires | Manual sharing | Residual access | Expiry/revoke |
| 57 | Rights only positive-tested | Happy path | Hidden cross-access | Negative matrix |
| 58 | Export unrestricted | Defaults | Mass leak | Approval/minimum |
| 59 | Secret in README | Real example | Compromise | Placeholder/secret store |
| 60 | Webhook committed | Quick integration | Unauthorized calls | Environment/rotation |
| 61 | One broad integration token | Simplicity | Large blast radius | Purpose scopes |
| 62 | Raw payload logged | Debugging | Client data leak | Masking |
| 63 | Infinite retry | No DLQ | Load/outage | Bounded retry |
| 64 | Retry duplicates Project/invoice ref | No idempotency | Double records | Source event key |
| 65 | Out-of-order ignored | Async not designed | Old status wins | Version/order guard |
| 66 | Schema not validated | Trust source | Corrupt data | Contract validation |
| 67 | No correlation ID | Fragmented logs | Slow RCA | Trace ID |
| 68 | Free bidirectional edits | No master | Ping-pong | Field ownership |
| 69 | Connector invented | Guessing | Blocked release | Official verification |
| 70 | REST method invented | No reference check | Broken integration | API docs |
| 71 | Batch ignores limits | Premature optimization | Partial failure | Official check/report |
| 72 | Migration has no inventory | Unknown sources | Missing history | Source/volume report |
| 73 | Import without dry-run | Time pressure | Mass corruption | Sample/dry-run |
| 74 | Real client data in UAT | Prod copy | Confidentiality breach | Synthetic data |
| 75 | No relation reconciliation | HTTP success trusted | Orphan records | Counts/sample report |
| 76 | BI mixes Deal/Project/Task/Time/Invoice | Wrong grain | False KPI | Metric contract |
| 77 | Satisfaction inferred from silence | Biased method | False quality | Explicit survey methodology |
| 78 | Release without smoke/rollback | Deadline pressure | Long incident | Checklist/drill |
| 79 | Cloud assumes DB/custom runtime | On-Premise habit | Impossible design | External boundary |
| 80 | On-Premise core modified | Quick fix | Upgrade/security failure | Extensions only |

## 22. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Подтверждённый AS IS/scope | Stakeholders, services, systems, tariff, volumes | Facts/open questions approved |
| 2. Архитектура | Утверждённый TO BE | Entity/grain model, master matrix, rights, ADR, NFR | Architecture/security review |
| 3. CRM настройка | Governed client and sales model | Fields, stages, roles, automation register | Synthetic sales paths pass |
| 4. Каталог услуг | Versioned offerings and pricing projection | Service dictionary, versions, owners | Proposal test passes |
| 5. Проекты | Controlled delivery/resource model | Projects, assignments, tasks, changes, acceptance | End-to-end UAT passes |
| 6. Интеграции | Reliable secure exchange | Contracts, secrets, retry, idempotency, monitoring | Failure/replay/reconcile pass |
| 7. Тестирование | Evidence-based quality | Test plan, synthetic data, rights/capacity/rollback tests | Critical defects closed/sign-off |
| 8. Запуск | Controlled release | Cutover, backup/export, communication, fallback, rollback | Smoke/reconciliation pass |
| 9. Оптимизация | Measured improvement | Baseline, DQ, incidents and change backlog | KPI change verified |

### Критерии готовности

- подтверждены Cloud/On-Premise, тариф/редакция, версия, регион, права and test environment;
- утверждены Lead decision, client model, Deal–Project boundary, service versions, expert master, rights and ADR;
- DMS/ЭДО, HR/PSA, 1С/ERP and finance sources не подменяются CRM-полями;
- выполнены synthetic UAT, negative rights, capacity, version, failure, migration dry-run and reconciliation tests;
- secrets are outside Git; logs/alerts contain no deliverables, contracts, payment details or unnecessary PII;
- approved cutover, rollback, DR, degraded mode, monitoring and support RACI exist;
- sales, delivery, practice, HR/resource, finance, data and security owners accepted criteria.

### Официальные источники

Использованы только записи из knowledge/SOURCES.md. Перед реализацией URL, дату обновления, тарифную и версионную применимость проверяют повторно.

- [Официальная пользовательская документация Bitrix24](https://helpdesk.bitrix24.ru/) — CRM, задачи, права, automation and Cloud.
- [Официальная документация REST API](https://apidocs.bitrix24.ru/) — интеграции, авторизация и ограничения.
- [REST API reference](https://apidocs.bitrix24.ru/api-reference/) — методы, параметры and scopes.
- [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/index.html) — официальный инструментарий.
- [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) — создание and configuration Smart Processes.
- [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) — штатная automation.
- [Бизнес-процессы](https://helpdesk.bitrix24.ru/open/23951294/) — выбор типа процесса.
- [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) — официальные способы подключения.
- [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) — datasets and limitations.
- [Маркетплейс Bitrix24](https://www.bitrix24.ru/apps/) — приложения с отдельной оценкой поставщика и безопасности.
- [Документация On-Premise](https://dev.1c-bitrix.ru/docs/portal.php) — коробочная разработка без изменения ядра.

### Ограничения и открытые вопросы

1. Cloud или On-Premise, тариф/редакция, версия, регион and actual permissions.
2. Practices, services, billing models, lead usage, client hierarchy and account ownership.
3. Master client, service, skill, capacity, time, project, document, invoice, payment and cost.
4. HR/PSA, 1С/ERP, DMS/ЭДО, site, telephony and BI APIs, volumes, limits and support.
5. Capacity calendar, allocation units, actual-time process, utilization formula and conflict rules.
6. Diagnostic, estimate, proposal, handover, change, acceptance, support and repeat-sale policies.
7. Client/project rights, external access, confidentiality, export, retention and legal hold.
8. Finance grain, revenue recognition boundary, currency, rate versions and reconciliation.
9. BI grain, metric owners, satisfaction methodology, freshness, control totals and DQ thresholds.
10. Test portal, release window, downtime, rollback, degraded mode, DR and support RACI.

Документ не подтверждает юридическую допустимость конкретного договора, методики, внешнего доступа или обработки клиентских материалов. Такие решения требуют отдельной правовой, договорной и информационно-безопасностной проверки.
