# Recruitment Agency CRM and Candidate Lifecycle Architecture Case

Архитектурный кейс для синтетического рекрутингового агентства, которое продаёт услуги B2B-клиентам, ведёт вакансии, кандидатов, подборы, интервью, предложения и договоры. Документ задаёт референсную архитектуру Bitrix24, а не готовую конфигурацию, кадровое решение, юридическое заключение или замену специализированной ATS, job board, HRIS, ЭДО/DMS и 1С.

> **Граница данных:** реальные работодатели, кандидаты, сотрудники, резюме, фотографии, контакты, оценки, зарплаты, документы, CRM ID, webhook URL и токены не используются. Все примеры полностью синтетические и обезличенные.

> **Архитектурная граница:** Bitrix24 отвечает за B2B CRM, коммерческие возможности, координацию вакансий, подборов, задач и коммуникаций. Утверждённая ATS/HRIS может оставаться master резюме, consent/legal-basis evidence, кадрового статуса и расширенного профиля; ЭДО/DMS — документов; 1С/ERP — счетов, оплат и финансовых фактов. Конкретная применимость определяется на Discovery.

Обозначения:

- **факт Bitrix24** — подтверждается официальным источником из knowledge/SOURCES.md;
- **решение архитектора** — рекомендация, утверждаемая владельцами процесса и данных;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа или редакции, версии, региона, прав и приложения;
- **чувствительные данные** — сведения кандидата, доступ к которым ограничен purpose, role and retention policy.

## 1. Business Context

### Модель рекрутингового бизнеса

Агентство получает B2B-запрос работодателя, квалифицирует потребность, согласовывает условия, регистрирует вакансию и ведёт несколько кандидатов через независимые подборы. Коммерческая сделка, вакансия и конкретное представление кандидата работодателю имеют разные владельцев, сроки и результаты.

| Услуга | Коммерческая модель | Delivery object | Контроль результата |
|---|---|---|---|
| Подбор одного специалиста | Success fee/fixed package | Vacancy + Recruitments | Confirmed hire/start |
| Массовый подбор | Volume/milestone | Campaign + Vacancies | Cohort and source controls |
| Executive search | Retained/milestone | Confidential Vacancy | Restricted access |
| Recruitment process support | Time/material/retainer | Project/Requests | Agreed SLA/deliverables |
| Candidate mapping | Research deliverable | Project/Candidate projections | Scope and lawful basis |

Каналы привлечения: сайт, referrals, job boards, email, calls, approved databases and events. Наличие готового коннектора к конкретному каналу не предполагается.

Цели CRM:

- единый реестр B2B-клиентов и коммерческих возможностей;
- отсутствие потерь вакансий и кандидатов;
- отдельный статус кандидата и статус кандидата в конкретном подборе;
- управляемые интервью, feedback, offers and start confirmation;
- прозрачная загрузка рекрутеров и сроки вакансий;
- контролируемая передача данных работодателю;
- достоверная аналитика источников, конверсий и прибыльности;
- аудит прав, сроков хранения и интеграционных операций.

**Решение архитектора:** до настройки фиксируются Cloud/On-Premise, тариф, роли, ATS/HRIS, job boards, volumes, retention, legal basis, sensitive categories, integrations, downtime and test environment.

## 2. AS IS Problems

| Проблема | Симптом | Причина | Целевой контроль |
|---|---|---|---|
| Кандидаты теряются | Нет следующего шага | Таблицы и личные сообщения | Owner/task/SLA |
| Нет единой базы | Профиль дублируется по каналам | Нет stable identity policy | Conservative dedup |
| Вакансии не контролируются | Срок и owner неизвестны | Нет lifecycle | Vacancy stages |
| Коммуникации разрознены | История в личной почте/телефоне | Нет activity policy | Governed interaction history |
| Статус кандидата неверен | Один статус на все вакансии | Нет junction entity | Recruitment Assignment |
| Feedback теряется | Оценки в чатах | Нет structured decision | Versioned feedback |
| Данные передаются без контроля | Резюме отправляется вручную | Нет release gate | Approved disclosure |
| Аналитика не сходится | Candidate, recruitment and vacancy mixed | Wrong grain | Metric contracts |

AS IS должен включать client sales, vacancy intake, sourcing, identity, candidate lifecycle, submissions, interviews, feedback, offers, hiring/start, guarantees, documents, communications, security, retention, integrations and reports.

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Lead | Неквалифицированный B2B-запрос, если lead model утверждена | Bitrix24 |
| CRM | Contact | Представитель работодателя или партнёра | Bitrix24/MDM |
| CRM | Company | Компания-заказчик, партнёр или подрядчик | Bitrix24/ERP |
| CRM | Deal | Продажа отдельной recruitment service | Bitrix24 |
| CRM/Smart Process | Request | Вопрос клиента или кандидата вне продажи | Bitrix24 |
| Smart Process | Vacancy | Требование клиента и vacancy lifecycle | Bitrix24/ATS |
| Smart Process | Candidate | Минимальный профиль кандидата | ATS/Bitrix24 by ADR |
| Smart Process | Recruitment | Связь Candidate–Vacancy и selection status | Bitrix24/ATS |
| Smart Process | Interview | Встреча, participants, feedback and decision | Bitrix24/calendar/ATS |
| Smart Process | Candidate Offer | Условия и статус предложения | ATS/HRIS/DMS |
| Smart Process | Contract | Согласование и authoritative reference | ЭДО/DMS |

~~~text
Employer Request -> Lead / Deal -> Company + Client Contact -> Contract
                                            |
                                            v
                                         Vacancy
                                      /      |      \
                               Recruitment Recruitment Recruitment
                                  |            |          |
                              Candidate A  Candidate B Candidate C
                                  |
                             Interview(s) -> Candidate Offer -> Start confirmation
~~~

### Ключевое решение модели

Candidate хранит person-level profile and global eligibility state. Recruitment хранит vacancy-specific stage, source attribution, owner, submission, interview, client decision and outcome. Один кандидат может участвовать в нескольких подборах без перезаписи истории.

### Правила целостности

1. Deal, Vacancy, Candidate and Recruitment имеют разные grains.
2. Contact работодателя не используется как карточка кандидата.
3. Candidate identity не строится только на имени, телефоне или email.
4. Resume binary и расширенный профиль хранятся в утверждённой защищённой системе или как минимально необходимая защищённая копия по policy.
5. Передача профиля клиенту требует подтверждённых purpose, access and current version.
6. Integration projection содержит source system, external ID, source timestamp, sync status and correlation ID.
7. Retry не создаёт дубликаты Candidate, Recruitment, Interview or Offer.

**Cloud:** отсутствует server/PHP/DB access; сложный matching, file processing, secrets and extended logs выносятся во внешний защищённый контур. **On-Premise:** дополнительно проектируются OS, web server, PHP, DB, modules, files, cron, backup, updates and security; core is not modified.

## 4. Recruitment Customer Journey

~~~text
Запрос клиента -> Анализ вакансии -> Поиск кандидатов -> Отбор
 -> Интервью -> Предложение -> Выход кандидата -> Закрытие вакансии

Exception paths:
Vacancy paused / candidate withdrew / client rejected / offer declined
 / no-show / replacement guarantee -> structured outcome -> next action
~~~

| Этап | Цель | Обязательный результат | Gate |
|---|---|---|---|
| Запрос клиента | Зафиксировать commercial need | Owner, Company, contact, next step | Dedup |
| Анализ вакансии | Согласовать requirements and constraints | Approved vacancy brief | Completeness/authority |
| Поиск кандидатов | Создать governed candidate pool | Source and eligibility | Lawful purpose/data minimum |
| Отбор | Проверить fit | Recruitment status and evidence | Criteria/version |
| Интервью | Получить structured feedback | Decision and next step | Participant/access |
| Предложение | Зафиксировать current terms/status | Offer version | Authority/confidentiality |
| Выход кандидата | Подтвердить event from authoritative source | Start status/date | Source confirmation |
| Закрытие вакансии | Зафиксировать outcome | Reason, winner, open obligations | Closure criteria |

Закрытие вакансии не удаляет rejected/withdrawn Recruitments. Retention and future-contact eligibility применяются отдельно к Candidate profile and related records.

## 5. Recruitment Data Model

| Сущность | Основные данные | Связи | Ограничение |
|---|---|---|---|
| Клиент | Industry, segment, needs, order history | Contacts, Deals, Vacancies | No duplicate by vacancy |
| Вакансия | Role, requirements, location/format, deadline, owner | Deal, Company, Recruitments | Requirements versioned |
| Кандидат | Synthetic profile ID, skills, experience summary, availability | Recruitments, communications | Minimal data |
| Recruitment | Vacancy, Candidate, stage, source, owner, outcome | Interviews, Offer | Vacancy-specific status |
| Interview | Type, date, participants, feedback status | Recruitment | No uncontrolled notes |
| Offer | Version, status, valid dates, conditions ref | Recruitment | Restricted fields |
| Resume | Version, source, protected link, expiry | Candidate/Recruitment | ATS/DMS master preferred |

~~~text
Company 1 ---- N Vacancy
                    |
                    +---- N Recruitment N ---- 1 Candidate
                               |
                               +---- N Interview
                               |
                               +---- 0..N Candidate Offer Version
~~~

### Identity, consent and retention

- identity matching is conservative and reviewable;
- source, collection date, purpose, permitted uses and retention class are recorded as governed metadata;
- current availability and communication eligibility are not inferred from old activity;
- candidate global status does not replace Recruitment outcome;
- deletion, restriction, archive and legal hold follow approved policy outside this template;
- sensitive and free-text fields are minimized and access-scoped.

## 6. Sales Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новый клиент | Зафиксировать commercial request | Source, Company/contact, owner | Ack/task/dedup |
| Квалификация | Проверить fit and authority | Service need, stakeholders, next step | SLA/reminder |
| Выявление потребности | Сформировать vacancy/service brief | Roles, volume, timeline, constraints | Checklist/task |
| Подготовка предложения | Зафиксировать scope and model | Proposal version, price, assumptions | Review |
| Договор | Подтвердить terms and readiness | DMS ref/status, effective dates | Completeness gate |
| Подбор | Контролировать delivery | Vacancy IDs, owners, milestones | Vacancy monitoring |
| Закрытие вакансии | Подтвердить outcome | Hire/start source, obligations | Finance/guarantee task |
| Повторный заказ | Создать новую opportunity | New need and account plan | No duplicate |

~~~text
Новый клиент -> Квалификация -> Потребность -> Предложение -> Договор
 -> Подбор -> Закрытие вакансии -> Повторный заказ

Closed-lost -> Structured reason
Vacancy expansion -> New Vacancy / Deal change by approved rule
~~~

Deal отражает коммерческий контракт и не заменяет Vacancy lifecycle. Несколько вакансий в одном договоре связываются явно, а не хранятся одной многострочной записью.

## 7. Vacancy Management

| Область | Данные | Контроль |
|---|---|---|
| Требования | Role, skills, level, must-have/nice-to-have | Approved version/owner |
| Условия | Work format, location, schedule, compensation range by policy | Restricted access |
| Срок | Open date, target date, priority | Calendar and pause rules |
| Ответственность | Account owner, lead recruiter, team | RACI |
| Статус | Draft, active, paused, filled, cancelled | Transition criteria |
| Capacity | Open positions, candidate demand | No silent overfill |
| Closure | Outcome, confirmed hire/start, obligations | Authoritative evidence |

~~~text
Draft -> Approved -> Active -> Sourcing / Selection -> Offer -> Filled
             |          |             |          |
             +-> Pause  +-> Reopen    +-> No hire +-> Guarantee/Replacement
~~~

Vacancy requirement changes use a version or approved change record. Existing candidate decisions remain linked to the criteria version used at the time.

## 8. Candidate Management

| Контур | Разрешённые данные | Контроль |
|---|---|---|
| Identity | Minimal synthetic ID and permitted contacts | Conservative dedup |
| Experience | Structured summary relevant to purpose | Source/date |
| Skills | Governed dictionary and evidence date | No unsupported inference |
| Resume | Versioned protected link | Access/retention |
| Availability | Current timestamped projection | Freshness |
| Communication | Channel, date, purpose, result | Eligibility/suppression |
| History | Recruitments and structured outcomes | Role-limited |

Candidate profile is not an unrestricted dossier. Health, family, beliefs, protected characteristics and unrelated personal content are not collected for convenience. Candidate decisions require documented job-relevant criteria and accountable human review.

Duplicate resolution never silently merges histories. Potential matches are quarantined or reviewed according to policy, with reversible mapping and audit.

## 9. Interview Management

| Элемент | Назначение | Обязательные данные | Gate |
|---|---|---|---|
| Interview | Specific meeting in Recruitment | Type, date, participants, owner | Availability/access |
| Scorecard | Job-relevant criteria | Criteria version, rating scale | No prohibited criteria |
| Feedback | Structured evidence and decision | Author, timestamp, visibility | Completeness |
| Reschedule | Preserve old slot and reason | New version/date | Notification control |
| No-show | Structured outcome | Who, date, next step | No automatic rejection without rule |
| Client decision | Vacancy-specific result | Decision owner/reason | Authority |

~~~text
Recruitment -> Interview Plan -> Meeting -> Structured Feedback
                                  |              |
                                  +-> Reschedule +-> Next Interview / Reject / Offer
~~~

Calendar event, Interview lifecycle and feedback are distinct. Recording, transcription and external meeting tooling require separate verification, access policy and legal basis; capability is not assumed.

## 10. HR Analytics

| Метрика | Grain | Формула/смысл | Контроль |
|---|---|---|---|
| Time to fill | Vacancy | Approved open-to-filled interval | Pause/calendar/source |
| Time to hire | Recruitment/Candidate event | Approved first-contact-to-start interval | Identity/event source |
| Recruiter throughput | Recruitment/period | Managed outcomes by role | Case mix/capacity |
| Candidate quality | Recruitment/cohort | Job-relevant accepted outcomes | Defined window/no sensitive proxy |
| Source effectiveness | Recruitment/source/cohort | Qualified/interview/offer/start conversion | Attribution version |
| Interview conversion | Recruitment stage transition | Pass rates by criteria version | Comparable process |
| Vacancy aging | Vacancy/date | Open duration and bottleneck | Status freshness |

Analytics must not turn unreviewed correlations into hiring decisions. Reports are for process quality and capacity; automated exclusion or protected-attribute inference is outside the solution unless separately authorized and validated.

## 11. Rights Model

| Роль | Клиенты | Вакансии | Кандидаты | Документы | Аналитика |
|---|---|---|---|---|---|
| Рекрутер | Assigned minimum | Assigned/team | Assigned Recruitments and minimum profile | Resume refs by purpose | Own/team operations |
| Аккаунт-менеджер | Own/team clients | Commercial summary | Approved client-facing profile only | Contract/proposal refs | Sales/client |
| Руководитель отдела | Governed team | Team portfolio | Team scope | Approved | Team analytics |
| HR/client reviewer | Assigned client/vacancy | Assigned | Submitted candidate minimum | Approved package | Vacancy |
| Бухгалтер | Counterparty minimum | Billing context | Нет | Financial docs | Finance |
| Директор | Governed aggregate | Portfolio | Restricted/aggregate | Approved | Executive |
| Администратор | Нет automatic business right | Configuration | Нет automatic candidate access | Нет automatic content | Dataset config as needed |

Principles: deny-by-default, least privilege, purpose limitation, vacancy/team/client scopes, segregation of sales/recruitment/finance, named accounts, temporary client access expiry, recertification, export controls and negative tests.

Передача кандидата клиенту uses an approved minimum profile and logged disclosure. Доступ работодателя к одному Recruitment не должен раскрывать другие вакансии кандидата или внутренние notes.

## 12. Automation Architecture

| Автоматизация | Trigger | Действие | Защита |
|---|---|---|---|
| B2B intake SLA | New valid source event | Owner/task/ack | Source key/idempotency |
| Vacancy activation | Approved brief/contract gate | Assign recruiter/checklist | One activation/version |
| Candidate follow-up | Eligible current Recruitment | Task/reminder | Purpose/current status |
| Interview coordination | Confirmed participants | Create task/invite request | Timezone/version |
| Feedback control | Interview completed | Reviewer task/escalation | One task/interview |
| Offer approval | Specific offer version ready | Approval route | Version/authority |
| Vacancy aging | Threshold reached | Escalate/review | Pause/calendar |
| Integration failure | Error/timeout | Queue/alert/bounded retry | Correlation/DLQ |

~~~text
Event -> Validate source / rights / purpose / version -> Idempotency
 -> Bounded action -> Audit result -> Exception queue / escalation
~~~

Robot cannot decide candidate suitability, infer protected attributes, confirm consent/legal basis, send a profile to a client, sign an offer or confirm hire/start without authoritative or accountable human gate. Bulk operations require dry-run, synthetic sample, report and rollback.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| Job boards | Vacancy refs, response IDs, candidate projection where authorized | Official vendor API/app/middleware | Scope, consent, dedup, limits |
| Email | Approved recruitment/client communication | Supported connector/integration | Thread mapping, access, suppression |
| Телефония | Call events and CRM linkage | Bitrix24 telephony/SIP/REST app | Rights, unmatched calls, recording policy |
| Calendars | Interview availability and event refs | Supported integration/API | Timezone, reschedule, participant privacy |
| 1С/ERP | Clients, contracts, invoices, payments | Supported integration/middleware | Stable IDs, currency, totals |
| BI/DWH | Governed datasets and aggregates | BI Constructor or approved ETL | Masking, grain, DQ, freshness |
| ATS/HRIS/DMS | Candidate/resume/consent/status/document refs | Documented API/middleware | Purpose, version, retention, reconcile |

~~~text
Source -> AuthN/AuthZ -> Purpose and minimization -> Schema/version validation
 -> Stable ID mapping -> Idempotency -> Bitrix24 projection
 -> Masked result log -> Bounded retry/quarantine -> Reconciliation
~~~

No connector to a particular job board, calendar, ATS or HRIS is assumed. REST methods, scopes, limits and Marketplace applications are verified against official documentation and the target portal. Secrets remain outside Git and payload logs.

## 14. Communication Architecture

| Канал | Business object | Mandatory metadata | Control |
|---|---|---|---|
| Звонок | Candidate/Recruitment or Company/Deal | Purpose, owner, result | Rights/recording policy |
| Email | Specific Recruitment/Vacancy/Deal | Thread, version, recipients | No cross-candidate leak |
| Встреча | Interview | Participants, timezone, outcome | Access and reschedule history |
| Client submission | Recruitment | Approved profile version, recipient, timestamp | Disclosure log |
| Notification | Current stage event | Template/version/channel | Eligibility/suppression |

~~~text
Communication Event -> Identify purpose and object -> Verify recipient/access
 -> Send/record minimal content -> Delivery result -> Next action/audit
~~~

Contact history is not a free-text dumping ground. Sensitive assessments stay in restricted structured fields; client-facing and internal feedback are separated.

## 15. Analytics

| Отчёт | Grain | Показатели | Контроль качества |
|---|---|---|---|
| Количество вакансий | Vacancy/period | Opened, active, paused, filled | Status/source reconciliation |
| Скорость закрытия | Vacancy/cohort | Time to fill, percentiles, aging | Calendar/pause |
| Конверсия этапов | Recruitment/stage | Sourced, screened, interview, offer, start | Stage semantics |
| Эффективность рекрутеров | Recruitment/recruiter/period | Throughput, aging, quality | Workload/case mix |
| Источники | Recruitment/source/cohort | Qualified, offer, start conversion | Attribution version |
| Прибыльность клиентов | Company/project/period | Revenue, cost, margin | ERP/1С reconciliation |
| Качество процесса | Vacancy/Recruitment | Feedback SLA, declines, reopen | Structured outcomes |

~~~text
CRM + ATS/HRIS + Job Boards + 1C/ERP
 -> Conformed Company / Vacancy / Candidate / Recruitment IDs
 -> DQ / Reconciliation -> Governed Metric Layer -> Role-based Dashboards
~~~

Metric contract includes owner, formula, grain, cohort window, timezone, exclusions, source, freshness, access and control total. Candidate, Recruitment, Interview, Vacancy, Deal, Invoice and Hire Event are separate grains.

## 16. Security Architecture

| Контроль | Реализация | Проверка |
|---|---|---|
| Candidate minimization | Only job-relevant approved fields | Field/file inventory |
| Purpose limitation | Use linked to Vacancy/Recruitment | Cross-purpose test |
| Client disclosure | Minimum approved profile/version | Recipient audit |
| Resume protection | Protected ATS/DMS link | Unauthorized download |
| Role scopes | Vacancy/team/client segmentation | Negative matrix |
| Integrations | Least-privilege credentials and rotation | Scope/revoke drill |
| Logs | Masked payload and correlation ID | Secret/PII scan |
| Export | Approval/minimum/protected destination | Export test |
| Retention | Approved keep/restrict/delete/hold lifecycle | Periodic evidence |
| Audit | CRM history plus security/integration logs | End-to-end trace |

**Cloud:** platform-level controls are provider-managed; customer controls users, configuration, data, integrations and devices. **On-Premise:** customer additionally secures OS, web server, PHP, DB, modules, files, cron, backup, updates and logs; core is not modified.

Specific legal bases, consent rules, cross-border transfers, recording rules and retention periods are not invented in this case and require current legal/security review.

## 17. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | New B2B request | Valid synthetic Company/contact | One Deal/owner/ack |
| 2 | Duplicate intake | Same source event | No duplicate |
| 3 | Vacancy creation | Approved synthetic brief | One Vacancy with owner/deadline |
| 4 | Requirement version | Brief updated | Old assessment keeps old criteria ref |
| 5 | Candidate import | Valid minimal synthetic profile | Candidate and source metadata created |
| 6 | Potential duplicate | Similar identifiers | Review/quarantine, no silent merge |
| 7 | Same candidate/two vacancies | Two Recruitments | Independent stages |
| 8 | Interview schedule | Valid participants/timezone | One Interview and tasks |
| 9 | Interview reschedule | New slot | History retained/no duplicate |
| 10 | Client submission | Approved profile version | Logged minimum disclosure |
| 11 | Unauthorized submission | Missing purpose/access | Blocked/escalated |
| 12 | Offer version | v2 replaces v1 | Only current approved version active |
| 13 | Candidate withdrawal | One Recruitment | Other Recruitments unchanged |
| 14 | Confirmed start | Authoritative synthetic event | Vacancy outcome/finance task updated |
| 15 | Cross-client rights | Client reviewer from another vacancy | Access denied |
| 16 | Retention expiry | Synthetic expired record | Approved restriction/delete workflow |
| 17 | Integration timeout | Accepted then timeout | Idempotent bounded retry |
| 18 | BI grain | Candidate vs Recruitment vs Vacancy | Correct separate metrics |
| 19 | Migration dry-run | Synthetic sample | Report, no production write |
| 20 | Rollback | Failed release | Previous state restored |

Testing uses only synthetic candidates, resumes, employers, vacancies, interviews and offers. Positive, negative, repeated, rights, disclosure, retention, failure, recovery and reconciliation scenarios are mandatory.

## 18. Deployment

| Этап | Результат | Gate |
|---|---|---|
| Discovery | AS IS, systems, volumes, legal/data rules | Owners/security/legal approval |
| Архитектура | TO BE, master matrix, grains, rights, ADR | Architecture/privacy review |
| CRM настройка | Isolated CRM/Smart Processes/automation | Configuration review |
| Миграция данных | Inventory, mapping, minimize, dry-run | Reconciliation/sign-off |
| Интеграции | Contracts, secrets, retry, monitoring | Failure/security/disclosure tests |
| Обучение | Role-based synthetic scenarios | UAT |
| Запуск | Cutover, communication, fallback, rollback | Go/no-go |
| Stabilization | Hypercare, baseline and incidents | Exit criteria |

~~~text
Inventory -> Data/Dependency/Threat Review -> Test Configuration
 -> Synthetic UAT -> Migration Dry-run -> Go/No-Go
 -> Release -> Smoke/Reconciliation -> Hypercare or Rollback
~~~

Existing fields, stages, rights, automations and integrations are not removed or renamed before dependency analysis, backup/export, pilot and approved rollback. Production candidate data is not copied into test.

## 19. Monitoring

| Сигнал | Правило | Действие | Владелец |
|---|---|---|---|
| Просроченная вакансия | Target/aging threshold | Review/replan/escalate | Recruitment lead |
| Потерянный кандидат | No next action/current Recruitment | Assign/review | Recruiter |
| Task without owner | Empty/inactive assignee | Reassign/escalate | Process owner |
| Feedback overdue | Interview feedback SLA | Remind/escalate | Hiring process owner |
| Stale availability | Freshness exceeded | Mark unknown/verify | Recruiter |
| Integration error | Error/retry/DLQ | Investigate/replay | Integration support |
| Disclosure anomaly | Unexpected access/export | Security incident | Security owner |
| Retention breach | Record beyond policy | Restrict/remediate | Data owner |
| BI mismatch | Control totals differ | Mark stale/investigate | BI owner |

Alert contains system, object reference, correlation ID, time, severity, owner and runbook; without full resume, feedback body, compensation, secret or unnecessary personal data.

## 20. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Candidate lost | No next action | Lost placement | Task/SLA monitor |
| 2 | Vacancy lost | No owner/ack | Client failure | Intake reconciliation |
| 3 | Duplicate Candidate | Multiple sources | Fragmented history | Conservative dedup |
| 4 | False merge | Weak identifiers | Wrong disclosure | Reviewable matching |
| 5 | Name as candidate key | No stable ID | Identity collision | External ID policy |
| 6 | Candidate stored as employer Contact | Entity shortcut | Rights/history leak | Separate Candidate |
| 7 | One Candidate status | No junction entity | Cross-vacancy overwrite | Recruitment entity |
| 8 | Deal equals Vacancy | One pipeline | Sales/delivery distortion | Separate lifecycle |
| 9 | Vacancy equals Recruitment | Under-modeling | No candidate funnel | Separate grains |
| 10 | Requirement changes silently | No versioning | Unfair/inconsistent decisions | Criteria version |
| 11 | Resume copied everywhere | Convenience | Data exposure | Protected master link |
| 12 | Excess candidate dossier | Collect-all habit | Privacy/security risk | Minimization |
| 13 | Old availability trusted | No freshness | Bad outreach | Timestamp/verify |
| 14 | Consent/legal basis assumed | No governance | Unauthorized use | Approved evidence |
| 15 | Client gets full profile too early | No gate | Disclosure risk | Minimum submission |
| 16 | Client sees other Recruitments | Broad rights | Confidentiality breach | Vacancy scope |
| 17 | Internal notes sent to client | Same field/template | Reputation/privacy harm | Separate views |
| 18 | Interview feedback free-text only | No structure | Bias/no analytics | Scorecard |
| 19 | Protected traits inferred | Model/notes | Discrimination risk | Job-relevant criteria |
| 20 | AI/autofilter treated as decision | Automation optimism | Unreviewed rejection | Human accountability |
| 21 | Interview recording assumed | Tool convenience | Legal/privacy risk | Separate approval |
| 22 | Offer version uncontrolled | Email files | Wrong conditions | Version gate |
| 23 | Hire confirmed manually | KPI pressure | False closure | Authoritative event |
| 24 | Candidate withdrawal closes global profile | One status | Other opportunities lost | Recruitment outcome |
| 25 | Guarantee case overwrites vacancy | No relation | History lost | Related lifecycle |
| 26 | Source attribution overwritten | Last-touch only | Wrong ROI | Versioned attribution |
| 27 | Recruiter ranking ignores vacancy complexity | Simplistic KPI | Harmful management | Case mix/capacity |
| 28 | Client profitability uses Deal amount | CRM-only finance | False margin | ERP/1С reconciliation |
| 29 | Admin is candidate super-reader | Role confusion | Uncontrolled access | Separate privileges |
| 30 | Export unrestricted | Defaults | Mass data leak | Approval/minimum |
| 31 | Temporary client access persists | No expiry | Residual access | Expiry/review |
| 32 | Secret in Git | Real example | Compromise | Secret store/rotation |
| 33 | Broad integration token | One account | Large blast radius | Least privilege |
| 34 | Raw resume payload logged | Debugging | Personal-data leak | Masking |
| 35 | Infinite retry | No DLQ | Load/outage | Bounded backoff |
| 36 | Retry duplicates Candidate/Recruitment | No idempotency | Double process | Business key |
| 37 | Out-of-order event | Async ignored | Old stage wins | Version/order guard |
| 38 | Schema drift | Job board change | Corrupt profile | Contract validation |
| 39 | No correlation ID | Fragmented logs | Slow RCA | End-to-end trace |
| 40 | Free bidirectional edit | No master | Ping-pong | Field ownership |
| 41 | Job-board connector assumed | Unverified claim | Blocked release | Official/vendor check |
| 42 | Migration without inventory | Unknown scope | Missing/excess data | Source report |
| 43 | Import without dry-run | Deadline pressure | Mass corruption | Sample/report/rollback |
| 44 | Real resumes in UAT | Prod copy | Privacy breach | Synthetic data |
| 45 | No reconciliation | HTTP success trusted | Silent loss | Counts/control totals |
| 46 | Retention not implemented | Store forever | Compliance risk | Lifecycle owner |
| 47 | Deletion breaks audit | Hard delete only | Evidence loss | Governed restriction/delete |
| 48 | BI mixes grains | Dashboard-first | False funnel | Metric contract |
| 49 | Time to fill ignores pauses | Simplistic formula | False performance | Calendar/pause rules |
| 50 | Quality measured by subjective score | No window/method | Bias | Defined outcome/cohort |
| 51 | Peak campaign load untested | Average design | Queue collapse | Load/degraded mode |
| 52 | Tariff not checked | Assumption | Feature unavailable | Portal assessment |
| 53 | Cloud assumes DB/file processing | On-Premise habit | Impossible design | External boundary |
| 54 | On-Premise core modified | Quick fix | Upgrade/security failure | Extensions only |
| 55 | DR ignores ATS/job boards/DMS | CRM-only recovery | Process not restored | End-to-end DR |

## 21. ADR

| ADR | Решение | Альтернатива | Обоснование | Последствие |
|---|---|---|---|---|
| ADR-REC-001 | Vacancy is a separate Smart Process | Deal stage only | Independent delivery lifecycle and SLA | Deal–Vacancy relation |
| ADR-REC-002 | Candidate is separate from CRM Contact | Candidate as Contact | Protects semantics and rights | Identity mapping needed |
| ADR-REC-003 | Recruitment is Candidate–Vacancy junction | One status on Candidate | Supports many simultaneous selections | More records/grain governance |
| ADR-REC-004 | Resume is protected ATS/DMS ref where possible | File copies in CRM | Version, retention and access | Source availability |
| ADR-REC-005 | Communications link to Recruitment purpose | Global candidate timeline only | Prevents cross-vacancy disclosure | Strong linking discipline |
| ADR-REC-006 | Rights are vacancy/team/purpose scoped | Agency-wide read | Least privilege | Negative tests |
| ADR-REC-007 | Integrations use governed contracts/middleware | Point-to-point webhooks | Security/idempotency/monitoring | Integration owner |
| ADR-REC-008 | BI uses separate Candidate/Recruitment/Vacancy grains | Raw CRM counters | Explainable funnel and aging | Metric/DQ ownership |

Each ADR has owner, status proposed/accepted/superseded, date, context, alternatives, consequences, portal/version and evidence.

## 22. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Нет Discovery | Сразу настраивают CRM | Wrong scope | AS IS/stakeholders |
| 2 | Cloud/On-Premise неизвестен | Template design | Unsupported solution | Capability matrix |
| 3 | Тариф не проверен | Assumption | Missing function | Verify target portal |
| 4 | Нет data/process owners | IT-only project | No decisions | RACI |
| 5 | Все входы — Leads | Universal pattern | Candidates and B2B mixed | Intake classification |
| 6 | Lead used as Candidate | Entity shortcut | Conversion/data loss | Candidate entity |
| 7 | Candidate stored as Contact | Native-only habit | Employer/candidate semantics mixed | Separate model |
| 8 | Company created for each Vacancy | Flat client model | Duplicates | Company–Vacancy relation |
| 9 | Name is dedup key | No stable ID | False merge | Conservative identity |
| 10 | Phone/email auto-merge without review | Aggressive dedup | Wrong person disclosure | Match policy |
| 11 | Deal equals Vacancy | One pipeline | Bad sales/delivery analytics | Separate entities |
| 12 | Vacancy equals Candidate status | Under-modeling | No funnel | Recruitment junction |
| 13 | Candidate has one global stage | Simplicity | Cross-vacancy overwrite | Per-Recruitment stage |
| 14 | Interview is task only | No event/feedback model | History lost | Interview entity |
| 15 | Offer stored in comment | Quick process | No version/access | Offer entity/ref |
| 16 | Resume copied to every Vacancy | Convenience | Storage/privacy risk | Protected source ref |
| 17 | Resume filename is identity | Weak key | Wrong version/person | Stable IDs |
| 18 | Candidate profile is unlimited dossier | Collect-all | Excess exposure | Data minimization |
| 19 | Sensitive traits in free text | No field governance | Bias/security risk | Prohibit/minimize |
| 20 | Availability has no timestamp | Static field | Wrong outreach | Freshness |
| 21 | Consent/legal basis is checkbox without source | Formalism | Unverifiable use | Evidence/source |
| 22 | Retention not defined | Store forever | Compliance risk | Lifecycle policy |
| 23 | Deletion is manual ad hoc | No workflow | Incomplete/unsafe action | Governed request |
| 24 | Vacancy brief in comments | No structure/version | Inconsistent selection | Versioned requirements |
| 25 | Must-have and nice-to-have mixed | No criteria model | Wrong filtering | Structured criteria |
| 26 | Requirement changes overwrite history | No versioning | Unfair decisions | Effective version |
| 27 | Stages describe tasks | Every action as stage | Ambiguity | State-based stages |
| 28 | Too many stages | Micromanagement | Low adoption | Tasks/checklists |
| 29 | Required fields too early | Maximum control | Fake values | Stage requiredness |
| 30 | No next action | Stage-only control | Lost candidate | Task policy |
| 31 | Recruiter assignment ignores capacity | Manual convenience | Delay/overload | Capacity check |
| 32 | Robot duplicates follow-up tasks | Re-entry ignored | Candidate spam | Idempotency |
| 33 | Robots loop stages | Mutual rules | Automation storm | Loop guard |
| 34 | Robot rejects candidate | Automation optimism | Unaccountable decision | Human review |
| 35 | Robot infers protected trait | Bad analytics | Discrimination/privacy risk | Prohibit |
| 36 | Profile sent before approval | Speed | Unauthorized disclosure | Submission gate |
| 37 | Full internal notes sent to client | Same template | Reputation/privacy harm | Separate data views |
| 38 | Email thread linked to wrong candidate | Weak mapping | Data leak | Correlation/review |
| 39 | Interview timezone omitted | Local assumption | Missed meeting | Explicit timezone |
| 40 | Reschedule overwrites old interview | One record | History lost | Version/event history |
| 41 | Feedback has no author/date | Anonymous notes | No accountability | Structured metadata |
| 42 | Scorecard uses subjective traits | No methodology | Bias | Job-related criteria |
| 43 | Client decision free-text only | No reason codes | No analytics | Governed outcomes |
| 44 | Offer version overwritten | One file | Wrong terms | Version lock |
| 45 | Offer decline closes Candidate globally | One status | Other paths lost | Recruitment outcome |
| 46 | Hire/start marked manually | KPI pressure | False closure | Authoritative source |
| 47 | Replacement guarantee overwrites old process | No related case | History loss | Separate linked lifecycle |
| 48 | Account manager sees all candidate notes | Broad role | Excess access | Client-facing projection |
| 49 | Client reviewer sees other vacancies | Shared rights | Confidentiality breach | Vacancy scope |
| 50 | Accountant sees resumes | Common record rights | Excess disclosure | Finance minimum |
| 51 | Admin equals candidate reader | Role confusion | Uncontrolled access | Separate privileges |
| 52 | Temporary client access never expires | Manual sharing | Residual access | Expiry/recertification |
| 53 | Rights only positive-tested | Happy path | Hidden cross-access | Negative matrix |
| 54 | Export unrestricted | Defaults | Mass leak | Approval/minimum |
| 55 | Secret in README | Real example | Compromise | Placeholder/secret store |
| 56 | Webhook committed | Quick integration | Unauthorized calls | Environment/rotation |
| 57 | One broad integration token | Simplicity | Large blast radius | Purpose scopes |
| 58 | Raw resume logged | Debugging | Personal-data leak | Masking |
| 59 | Infinite retry | No DLQ | Load/outage | Bounded retry |
| 60 | Retry duplicates Candidate/Interview | No idempotency | Double process | Source event key |
| 61 | Out-of-order ignored | Async not designed | Old status wins | Version/order guard |
| 62 | Schema not validated | Trust source | Corrupt data | Contract validation |
| 63 | No correlation ID | Fragmented logs | Slow RCA | Trace ID |
| 64 | Free bidirectional edits | No master | Ping-pong | Field ownership |
| 65 | Job-board connector invented | Guessing | Blocked release | Official/vendor verification |
| 66 | REST method invented | No reference check | Broken integration | API docs |
| 67 | Batch ignores limits | Premature optimization | Partial failure | Official check/report |
| 68 | Migration has no inventory | Unknown sources | Missing/excess data | Source/volume report |
| 69 | Import without dry-run | Time pressure | Mass corruption | Sample/dry-run |
| 70 | Real candidates in UAT | Prod copy | Privacy breach | Synthetic data |
| 71 | Migration ignores consent/retention metadata | Data-only map | Unlawful/stale processing | Governance mapping |
| 72 | No relation reconciliation | HTTP success trusted | Orphan Recruitments | Counts/sample report |
| 73 | BI counts Candidates as Recruitments | Wrong grain | False funnel | Metric contract |
| 74 | Time to fill ignores pauses | Simplistic formula | False KPI | Calendar rules |
| 75 | Source attribution is last-touch only | Default logic | Wrong investment | Versioned attribution |
| 76 | Recruiter ranking ignores vacancy difficulty | KPI pressure | Harmful management | Context/capacity |
| 77 | Old fields deleted immediately | Cleanup | Broken automation/integration | Deprecation scan |
| 78 | Release without smoke/rollback | Deadline pressure | Long incident | Checklist/drill |
| 79 | Cloud assumes DB/file parser | On-Premise habit | Impossible design | External boundary |
| 80 | On-Premise core modified | Quick fix | Upgrade/security failure | Extensions only |

## 23. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Confirmed AS IS/scope/data rules | Stakeholders, systems, tariff, volumes, retention | Facts/open questions approved |
| 2. Архитектура | Approved TO BE | Entity/grain model, master matrix, threat/privacy model, ADR | Architecture/security review |
| 3. CRM настройка | Governed B2B sales model | Companies, Contacts, Deals, roles, automation register | Synthetic sales paths pass |
| 4. Вакансии | Versioned Vacancy lifecycle | Brief, stages, owners, SLA, exceptions | Vacancy UAT passes |
| 5. Кандидаты | Secure Candidate/Recruitment lifecycle | Identity, resume refs, interviews, offers, retention | End-to-end and rights UAT pass |
| 6. Интеграции | Reliable secure exchange | Contracts, secrets, retry, idempotency, monitoring | Failure/disclosure/reconcile pass |
| 7. Тестирование | Evidence-based quality | Synthetic data, negative rights, migration/rollback tests | Critical defects closed/sign-off |
| 8. Запуск | Controlled release | Cutover, backup/export, communication, fallback, rollback | Smoke/reconciliation pass |
| 9. Оптимизация | Measured improvement | Baseline, DQ, incidents and change backlog | KPI change verified |

### Критерии готовности

- confirmed Cloud/On-Premise, tariff/edition, version, region, permissions and test environment;
- approved Lead decision, Candidate–Recruitment–Vacancy model, masters, retention, disclosures, rights and ADR;
- ATS/HRIS, job boards, DMS/ЭДО and 1С/ERP are not replaced by unsupported CRM fields;
- synthetic UAT, negative rights, disclosure, retention, failure, load, migration dry-run and reconciliation tests pass;
- secrets are outside Git; logs/alerts contain no resumes, full feedback, offers or unnecessary personal data;
- cutover, rollback, DR, degraded mode, monitoring and support RACI are approved;
- sales, recruitment, data/privacy, legal, security, finance and integration owners accept the result.

### Официальные источники

Использованы только записи из knowledge/SOURCES.md. Перед реализацией URL, дату обновления, тарифную и версионную применимость проверяют повторно.

- [Официальная пользовательская документация Bitrix24](https://helpdesk.bitrix24.ru/) — CRM, задачи, права, automation and Cloud.
- [Официальная документация REST API](https://apidocs.bitrix24.ru/) — интеграции, авторизация и ограничения.
- [REST API reference](https://apidocs.bitrix24.ru/api-reference/) — методы, параметры and scopes.
- [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/) — официальный инструментарий.
- [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) — создание and configuration Smart Processes.
- [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) — штатная automation.
- [Бизнес-процессы](https://helpdesk.bitrix24.ru/open/23951294/) — выбор типа процесса.
- [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) — официальные способы подключения.
- [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) — datasets and limitations.
- [Маркетплейс Bitrix24](https://www.bitrix24.ru/apps/) — applications with separate vendor/security review.
- [Документация On-Premise](https://dev.1c-bitrix.ru/docs/portal.php) — коробочная разработка без изменения ядра.

### Ограничения и открытые вопросы

1. Cloud or On-Premise, tariff/edition, version, region and actual permissions.
2. Recruitment services, commercial model, lead usage, vacancy types and client hierarchy.
3. Master Candidate, resume, consent/legal basis, Vacancy, Recruitment, Interview, Offer, hire/start and finance.
4. ATS/HRIS, job boards, calendars, DMS/ЭДО, 1С/ERP, telephony and BI APIs, limits and support.
5. Candidate identity, dedup, lawful purpose, communication eligibility, disclosure and retention policies.
6. Vacancy criteria versioning, pause, candidate withdrawal, client rejection, offer decline, hire/start and guarantee rules.
7. Team/client rights, external access, sensitive fields, files, exports and recertification.
8. Source attribution, time-to-fill calendars, recruiter capacity and outcome-quality methodology.
9. BI grains, metric owners, cohort windows, freshness, control totals and DQ thresholds.
10. Test portal, release window, downtime, rollback, degraded mode, DR and support RACI.

Документ не подтверждает юридическую допустимость конкретного способа поиска кандидатов, решения о найме, записи интервью, передачи резюме или срока хранения. Эти вопросы требуют отдельной актуальной правовой, privacy and information-security review.

# Related Documents

- [Case Library](../README.md)
- [Architecture Patterns](../PATTERNS.md)
- [Discovery Template](../DISCOVERY_TEMPLATE.md)
- [Architecture Template](../ARCHITECTURE_TEMPLATE.md)
