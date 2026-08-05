# IT and SaaS Customer Lifecycle CRM Architecture Case

Архитектурный кейс для синтетической IT-компании, которая продаёт SaaS-продукт, ведёт B2B/B2C клиентов, подписки, onboarding, Customer Success, поддержку и расширение. Документ задаёт референсную архитектуру Bitrix24, а не готовую конфигурацию, billing engine, product telemetry platform, identity provider, helpdesk или юридическую модель подписки.

> **Граница данных:** реальные компании, пользователи, tenant IDs, домены, договоры, подписки, usage events, обращения, суммы, телефоны, email, CRM ID, webhook URL, токены и API keys не используются. Все примеры синтетические и обезличенные.

> **Архитектурная граница:** Bitrix24 отвечает за CRM, коммерческие возможности, onboarding coordination, Customer Success, задачи и управленческие проекции. Product platform остаётся master SaaS accounts, product users, entitlement, configuration and usage facts; billing — plans, subscriptions, invoices, payments, credits and revenue events; helpdesk — technical cases, если он утверждён. CRM не заменяет эти системы.

Обозначения:

- **факт Bitrix24** — подтверждается официальным источником из knowledge/SOURCES.md;
- **решение архитектора** — целевая рекомендация, принимаемая на Discovery;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа или редакции, версии, региона, прав и приложения;
- **authoritative fact** — факт, подтверждённый master-системой, а не рассчитанный вручную в CRM.

## 1. Business Context

### Модель SaaS-бизнеса

SaaS-компания привлекает потенциальных клиентов, продаёт доступ к продукту, создаёт tenant/account, проводит onboarding, сопровождает использование, решает обращения и управляет продлением или расширением. B2B customer может включать юридическое лицо, несколько buyer/admin contacts, SaaS accounts and product users. B2C customer чаще связан с Contact and one or several product accounts.

| Модель | Коммерческий объект | Product object | Finance master |
|---|---|---|---|
| B2B subscription | Deal/Contract | SaaS Account + Subscription | Billing/ERP |
| B2C self-service | Order/payment projection | Product Account | Billing |
| Trial/freemium | Lead/Account projection | Trial Account | Product platform |
| Enterprise contract | Deal + negotiated terms | Tenant/entitlements | Billing/DMS |
| Services/onboarding | Deal/Project | Onboarding | ERP/Project system |

Цели CRM:

- единый account view без смешения CRM Company и SaaS tenant;
- управляемая квалификация, demo, proposal and contract;
- контролируемая передача продажи в onboarding;
- прозрачные subscription lifecycle and renewal signals;
- единый контекст sales, Customer Success and support;
- выявление expansion opportunities без перезаписи действующей подписки;
- достоверные MRR, ARR, churn, retention and LTV на сверенных данных;
- аудит прав, интеграций и client-data minimization.

**Решение архитектора:** до настройки подтверждаются Cloud/On-Premise, tariff, product architecture, billing model, B2B/B2C flows, support tooling, volumes, regions, data classification, integrations, downtime and test environments.

## 2. AS IS Problems

| Проблема | Симптом | Причина | Целевой контроль |
|---|---|---|---|
| Нет единого профиля | Company, tenant and payer не связаны | Нет identity/master model | Stable cross-system IDs |
| Sales оторваны от support | Инженер не видит commercial context | Нет governed projection | Account context with least privilege |
| Onboarding не контролируется | Нет owner/milestones | Deal закрыт слишком рано | Separate Onboarding lifecycle |
| Expansion теряется | Upgrade обсуждается в чатах | Нет signal-to-opportunity rule | Expansion Deal/task |
| Renewals просрочены | Нет current subscription dates | Billing не интегрирован | Authoritative renewal projection |
| Usage трактуется вручную | Raw events copied to CRM | Нет metric layer | Aggregated governed signals |
| Health score спорный | Одна непрозрачная цифра | Нет model/version/owner | Explainable components |
| Аналитика слабая | Deal amount считается MRR | CRM used as billing | Finance reconciliation |

AS IS включает acquisition, identity, products/plans, trial, sales, contract, provisioning, onboarding, usage, Customer Success, support, billing, renewal, churn, security, integrations and analytics.

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Lead | Неквалифицированный demand, если lead model утверждена | Bitrix24 |
| CRM | Contact | Buyer, admin, champion or B2C customer | Bitrix24/MDM |
| CRM | Company | B2B legal/commercial account | Bitrix24/ERP/MDM |
| CRM | Deal | New sale, renewal or expansion opportunity by policy | Bitrix24 |
| CRM/Smart Process | Request | Business/customer question outside sale | Bitrix24/helpdesk |
| Smart Process | SaaS Account | Minimal product tenant/account projection | Product platform |
| Smart Process | Subscription | Billing subscription projection | Billing |
| Smart Process | Plan | Effective plan/catalog projection | Billing/product catalog |
| Smart Process | Onboarding | Implementation and activation lifecycle | Bitrix24/project system |
| Smart Process | Client Integration | Customer-specific integration lifecycle | Product/integration platform |
| Smart Process | Support Request | CRM projection or governed case | Helpdesk/Bitrix24 |

~~~text
Lead -> Contact / Company -> Deal -> Contract / Order
                                 |
                                 v
                       SaaS Account -> Subscription -> Plan
                            |              |
                            v              +-> Billing / Renewal
                        Onboarding
                            |
            Product Usage <-+-> Client Integrations
                            |
                   Customer Success <-> Support Requests
                            |
                 Expansion / Renewal / Churn outcome
~~~

### Правила целостности

1. CRM Company, SaaS Account, payer, Subscription and Product User are separate grains.
2. Product users are not automatically created as CRM Contacts; only approved customer-facing roles are projected.
3. Subscription status, term, entitlement and payment come from billing/product masters.
4. Raw product events do not flow into CRM; only documented aggregates/signals with freshness are used.
5. Deal describes commercial opportunity, not current entitlement.
6. Integration projections contain source system, external ID, source timestamp, sync status and correlation ID.
7. Retry does not duplicate Account, Subscription, Onboarding, case or notification.

**Cloud:** no server/PHP/DB access; integration runtime, secrets, event processing and telemetry aggregation remain external. **On-Premise:** additionally design OS, web server, PHP, DB, modules, files, cron, backup, updates, performance and security; core is not modified.

## 4. Customer Lifecycle

~~~text
Лид -> Продажа -> Договор -> Onboarding -> Использование продукта
 -> Поддержка -> Расширение -> Продление

Exception paths:
Trial expired / onboarding blocked / payment failed / downgrade
 / suspension / churn -> governed outcome -> recovery or closure
~~~

| Этап | Цель | Обязательный результат | Gate |
|---|---|---|---|
| Лид | Зафиксировать demand | Source, identity, owner, next step | Dedup |
| Продажа | Подтвердить fit and value | Qualified opportunity | Stakeholders/use case |
| Договор | Зафиксировать terms | Contract/order reference | Authority/readiness |
| Onboarding | Достичь agreed activation | Milestones, owners, dependencies | Handover accepted |
| Использование | Получать governed adoption signals | Metric version/freshness | Product source |
| Поддержка | Решать cases by priority/SLA | Resolution/source evidence | Entitlement |
| Расширение | Создать separate opportunity | Expansion need and owner | Eligibility/no duplicate |
| Продление | Подтвердить subscription continuation | Billing renewal fact | Authoritative source |

Lifecycle does not imply a single linear CRM stage. Sale, onboarding, subscription, support and renewal remain related but independently auditable lifecycles.

## 5. SaaS Data Model

| Сущность | Основные данные | Связи | Master |
|---|---|---|---|
| Клиент | Company/Contact, segment, account owner, lifecycle tier | Accounts, Deals, contacts | CRM/MDM |
| SaaS Account | Tenant/account ID, environment, lifecycle status, region | Company/Contact, Subscriptions | Product platform |
| Product User | Product user ID, role, active aggregate | SaaS Account | Product/IdP |
| Subscription | ID, Plan version, term, quantity, status, renewal date | Account, payer, contract | Billing |
| Plan | Product family, edition, limits/entitlements, effective dates | Subscriptions | Billing/catalog |
| Onboarding | Scope, milestones, owner, target/actual dates | Account, Deal, integrations | Bitrix24/Project |
| Client Integration | Type, environment, owner, status, health | Account, onboarding/support | Integration platform |
| Support Request | Category, priority, SLA, owner, resolution | Account, contact, incident | Helpdesk/Bitrix24 |

~~~text
Company 1 ---- N SaaS Account 1 ---- N Product User
   |                  |
   |                  +---- N Subscription N ---- 1 Plan Version
   |                  |
   |                  +---- 0..N Onboarding / Client Integration
   |
   +---- N Deal / Contact / Support Context
~~~

### Identity and master rules

- SaaS Account uses stable product ID, not company name or domain alone;
- payer and user organization may differ and are related explicitly;
- subscription quantity, entitlement and status are not manually edited in CRM;
- plan versions are effective-dated; past subscriptions retain their version;
- usage aggregate has metric ID, window, source time and freshness state;
- B2C and B2B identity rules are separate and reviewable.

## 6. Sales Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новый лид | Зарегистрировать demand | Source, contact/account, owner | Ack/task/dedup |
| Квалификация | Проверить ICP/use case/timing | Segment, need, authority, next step | SLA/reminder |
| Discovery | Зафиксировать requirements and risks | Users, integrations, security, success criteria | Checklist/tasks |
| Демо | Подтвердить use case | Scenario, attendees, outcome | Follow-up |
| Предложение | Зафиксировать plan/scope/price version | Plan, quantity, term, assumptions | Review |
| Переговоры | Согласовать commercial/security terms | Decision owners, changes, blockers | Approval/escalation |
| Договор | Подтвердить authoritative commitment | DMS/order ref, dates | Completeness gate |
| Onboarding | Контролировать handover | Account/onboarding IDs, owner, milestones | Monitoring |
| Успешный клиент | Подтвердить agreed activation | Acceptance/activation source | CS handoff |

~~~text
Новый лид -> Квалификация -> Discovery -> Демо -> Предложение
 -> Переговоры -> Договор -> Onboarding -> Успешный клиент

Closed-lost -> Structured reason
Expansion / Renewal -> Separate Deal by approved commercial policy
~~~

Onboarding stage in Deal is a commercial control point; detailed work lives in Onboarding. Deal success does not create billing facts or product entitlement by itself.

## 7. Subscription Management

| Объект/событие | CRM projection | Authoritative source | Control |
|---|---|---|---|
| Plan | Name/version/effective dates/summary | Billing/catalog | Version/freshness |
| Subscription | ID, account, plan, term, quantity, status | Billing | Reconciliation |
| Renewal | Renewal date/status/quote ref | Billing/contract system | Current version |
| Upgrade | Requested and confirmed change | Billing | Effective date/entitlement |
| Downgrade | Requested and confirmed change | Billing | Impact/retention rule |
| Suspension | Status/reason projection | Billing/product | Access/current state |
| Cancellation | Effective date/reason/outcome | Billing | No manual churn fact |

~~~text
Plan Version -> Subscription -> Billing Periods
                    |
                    +-> Upgrade / Downgrade / Renewal / Cancel
                    |
                    +-> Entitlement Projection -> Product Account
~~~

CRM may coordinate tasks and opportunities, but billing controls money, term and entitlement. A Deal cannot directly overwrite Subscription. Any change uses an approved command/workflow and authoritative confirmation.

## 8. Customer Success Architecture

| Контур | Данные | Решение |
|---|---|---|
| Onboarding | Milestones, blockers, owner, success criteria | Separate lifecycle |
| Adoption | Governed usage aggregates | Product analytics source |
| Relationship | Champion/admin/decision roles | CRM Contacts with purpose |
| Health | Explainable component model | Version, owner, thresholds |
| Risk | Signal, evidence, severity, action | Human review and task |
| Value review | Outcomes and agreed metrics | Periodic plan |
| Retention | Renewal readiness, blockers, plan | Billing dates + CS actions |

~~~text
Product/Billing/Support Signals -> Freshness and Quality Checks
 -> Explainable Health Components -> CSM Review -> Playbook/Task
 -> Outcome -> Model feedback and renewal/expansion opportunity
~~~

Health score is a recommendation, not a fact. It includes model version, components, timestamps and owner. Missing or stale telemetry cannot automatically mean churn risk or success.

## 9. Support Management

| Элемент | Назначение | Обязательные данные | Control |
|---|---|---|---|
| Case | Customer issue/request | Account, category, owner, status | Stable case ID |
| Priority | Impact and urgency | Approved matrix | No free-text priority |
| SLA | Operational response/resolution rules | Calendar, pause, entitlement | Versioned policy |
| Escalation | Transfer to higher support/product/incident | Reason, owner, deadline | Ack/handoff |
| Incident relation | Link cases to shared incident | Incident ID and scope | Incident master |
| Resolution | Structured outcome and evidence | Source/version | Reopen rule |

~~~text
Customer Request -> Entitlement / Identity -> Case -> Triage
 -> Resolve or Escalate -> Verification -> Close / Reopen
                    |
                    +-> Product Incident / Problem / Feedback relation
~~~

Support Request in Bitrix24 can be master only if the approved process supports required SLA, security and volume. Otherwise Bitrix24 receives a minimal helpdesk projection. Technical incident, support case and product feedback are separate objects.

## 10. Product Management

| Объект | CRM role | Master | Control |
|---|---|---|---|
| Product module | Catalog/interest projection | Product catalog | Version |
| Feature | Customer-facing reference | Product system | No entitlement assumption |
| Release/version | Supported customer-relevant status | Release system | Compatibility/source |
| Feedback | Structured problem/outcome/context | Product feedback system/Bitrix24 | Dedup and access |
| Feature request | Evidence and account impact | Product backlog | No promise |
| Entitlement | Read-only customer projection | Product/billing | Current source |

~~~text
Support / CSM / Sales Feedback -> Normalize and Deduplicate
 -> Product Review -> Backlog Reference -> Decision / Release Source
 -> Customer-facing update by approved rule
~~~

CRM feedback does not commit a roadmap or release date. Product module, feature, plan and entitlement remain different concepts. Sensitive logs and telemetry are not copied into feedback text.

## 11. Rights Model

| Роль | Клиенты | Продажи | Подписки | Поддержка | Аналитика |
|---|---|---|---|---|---|
| Sales manager | Own/team accounts | Own/team Deals | Commercial summary | Limited context | Sales |
| Account manager | Assigned accounts | Expansion/renewal context | Read projection | Customer context | Account |
| Customer Success manager | Assigned accounts | Opportunity signals | Read current projection | Assigned summary | Adoption/health |
| Support engineer | Need-to-know account | No pricing unless needed | Entitlement minimum | Assigned cases | Support |
| Product manager | Pseudonymized/approved account context | Aggregate | Aggregate | Feedback/incident aggregate | Product |
| Директор | Governed portfolio | Portfolio | Aggregate/approved | Aggregate | Executive |
| Администратор | No automatic business entitlement | Configuration | No automatic data access | Configuration | Dataset config as needed |

Principles: deny-by-default, least privilege, tenant/account/team scopes, segregation of sales/support/finance/product, named accounts, temporary access expiry, recertification, export controls and negative tests.

Product administration and Bitrix24 administration are separate privileges. CRM access never grants product tenant access, support logs, secrets or raw telemetry.

## 12. Automation Architecture

| Автоматизация | Trigger | Действие | Защита |
|---|---|---|---|
| Lead scoring | Approved data and rules | Priority suggestion/task | Explainable/versioned/no auto-reject |
| Sales SLA | Valid new Lead/Deal | Owner/task/reminder | Source key/idempotency |
| Onboarding start | Contract/account readiness | Create/link Onboarding and tasks | Unique Deal/account key |
| Milestone control | Due threshold | Reminder/escalation | One event/threshold |
| Renewal preparation | Authoritative renewal window | CSM/account task | Current subscription state |
| Churn signal | Governed fresh components | Review task/playbook | Human validation |
| Expansion signal | Approved usage/business rule | Account review/new Deal | No duplicate/eligibility |
| Integration failure | Error/timeout | Queue/alert/bounded retry | Correlation/DLQ |

~~~text
Event -> Validate source / version / freshness / rights -> Idempotency
 -> Explainable business rule -> Bounded action -> Audit
 -> Exception queue / human review / escalation
~~~

Automation does not create entitlement, confirm payment, cancel subscription, block product access or promise product functionality without authoritative and approved gates. Bulk actions use dry-run, synthetic sample, report and rollback.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| Product platform | Account, user aggregates, entitlement, usage signals | Documented API/event/middleware | Auth, schema, order, freshness |
| Billing | Plans, subscriptions, invoices, payments, credits | API/event/middleware | Amount, currency, idempotency, reconcile |
| Bitrix24 CRM | Leads, accounts, Deals, onboarding/CS projection | Native configuration/REST as verified | Rights, IDs, audit |
| Website | Leads, trials, requests | CRM form/approved app/webhook/REST | Validation, consent, dedup |
| Telephony | Calls and CRM relation | Bitrix24 telephony/SIP/REST app | Rights, unmatched calls, recording policy |
| BI/DWH | Governed facts and dimensions | BI Constructor or approved ETL | Grain, masking, freshness, DQ |
| Helpdesk | Cases, SLA, incidents, resolution projection | Documented API/middleware | Case IDs, order, reconciliation |

~~~text
Product / Billing / Helpdesk Events -> Integration Layer
 -> AuthN/AuthZ -> Schema/version -> Identity mapping -> Idempotency
 -> Bitrix24 projection or quarantine -> Reconciliation / Monitoring
~~~

No ready connector to a particular product, billing or helpdesk platform is assumed. REST methods, scopes, limits and Marketplace applications are verified against official sources and the target portal. Secrets remain outside Git and payload logs.

## 14. Marketing Automation

| Сценарий | Segment | Eligibility | Control |
|---|---|---|---|
| Lead nurture | Qualified/engaged prospects | Approved basis and current segment | Frequency/suppression |
| Trial activation | Valid trial account | Transactional purpose/current state | No expired-account messages |
| Education | Customer role + relevant module | Approved communication policy | Role/channel |
| Upsell | Account with reviewed need | CSM/account approval | Separate Deal |
| Cross-sell | Relevant product family | Entitlement and fit check | No incompatible offer |
| Win-back | Churned eligible cohort | Retention/communication rule | Reason/time window |

~~~text
Governed Data -> Segment Version -> Eligibility / Consent / Suppression
 -> Campaign Version -> Send -> Delivery / Response -> Measurement
~~~

Transactional product notices and marketing are separate. Usage alone is not blanket permission to communicate. Anonymous visitors and product users are not automatically converted into CRM Contacts.

## 15. Analytics

| Отчёт | Grain | Показатели | Control |
|---|---|---|---|
| MRR | Active subscription component/month | Recurring normalized amount | Billing formula/currency/reconcile |
| ARR | Approved MRR annualization or contract basis | Annual recurring value | Documented method |
| Sales conversion | Lead/Deal cohort | Conversion, cycle, win/loss | Stage/segment |
| Churn | Subscription/account cohort | Logo/revenue churn | Denominator/event rules |
| Retention | Cohort/period | Gross/net retention | Expansion/contraction rules |
| LTV | Customer cohort/model | Expected/realized value | Model assumptions/version |
| Customer Success | Account/period | Onboarding, adoption, health actions | Signal freshness/context |
| Team effectiveness | Role/work item/period | SLA, throughput, quality | Workload/case mix |

~~~text
Billing Facts + Product Usage + CRM + Helpdesk
 -> Conformed Account / Subscription / Plan / Period IDs
 -> DQ / Reconciliation -> Metric Layer -> Role-based Dashboards
~~~

Metric contract includes owner, formula, grain, cohort, period, timezone, currency/FX, exclusions, source, freshness and control total. MRR/ARR/LTV/churn formulas are recommendations until finance and data owners approve them.

## 16. Security Architecture

| Control | Implementation | Verification |
|---|---|---|
| Tenant isolation | Stable tenant scope and least privilege | Cross-tenant negative test |
| Product users | Minimal approved projection | Field inventory |
| Entitlement | Read-only authoritative source | Unauthorized-change test |
| Support data | Need-to-know and redacted logs | Cross-role test |
| Integrations | Purpose-specific credentials and rotation | Scope/revoke drill |
| Logs | Masked payload and correlation ID | Secret/PII scan |
| Export | Approval/minimum/protected destination | Export test |
| Retention | Approved keep/restrict/delete/hold | Periodic evidence |
| Audit | CRM plus integration/security logs | End-to-end trace |

**Cloud:** platform layer is provider-managed; customer controls users, rights, configuration, integrations, devices and data. **On-Premise:** customer additionally secures OS, web server, PHP, DB, modules, files, cron, backup, updates and logs; core is not modified.

Customer credentials, access tokens, API keys, raw telemetry and confidential support artifacts never enter CRM fields, Git, alerts or general logs.

## 17. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | New Lead | Valid synthetic web request | One Lead/Deal, owner and source |
| 2 | Duplicate trial event | Same external ID | No duplicate |
| 3 | B2B account | Company with two admins/one tenant | Correct relations |
| 4 | B2C account | Contact with product account | No fake Company |
| 5 | Plan version | Existing and new Subscription | Historical version preserved |
| 6 | Contract handover | Readiness complete | One Onboarding linked |
| 7 | Onboarding blocker | Client integration delayed | Risk/task/escalation once |
| 8 | Product activation | Authoritative event | Current projection/CS handoff |
| 9 | Usage stale | Freshness threshold exceeded | Health component marked unknown |
| 10 | Support entitlement | Valid Subscription | Correct support route |
| 11 | Cross-tenant rights | Engineer from another tenant | Access denied |
| 12 | Upgrade | Billing-confirmed plan change | Subscription projection updated |
| 13 | Payment failure | Billing event | Review task, no CRM-made suspension |
| 14 | Renewal | Authoritative renewal | New term/current status reflected |
| 15 | Churn | Confirmed cancellation effective | Correct cohort/outcome |
| 16 | Integration timeout | Accepted then timeout | Idempotent bounded retry |
| 17 | Out-of-order event | Older subscription version | New state preserved |
| 18 | BI reconciliation | Billing MRR vs CRM | Billing totals win/discrepancy logged |
| 19 | Migration dry-run | Synthetic sample | Report, no production write |
| 20 | Rollback | Failed release | Previous state restored |

Tests use synthetic companies, accounts, users, plans, subscriptions, cases and usage signals. Positive, negative, repeated, rights, billing, freshness, failure, recovery and reconciliation scenarios are required.

## 18. Deployment

| Этап | Результат | Gate |
|---|---|---|
| Discovery | AS IS, products, systems, volumes, regions | Business/data/security approval |
| Архитектура | TO BE, masters, grains, rights, ADR, NFR | Architecture/threat review |
| Настройка | Isolated CRM/Smart Processes/automation | Configuration review |
| Интеграции | Contracts, secrets, retry, monitoring | Failure/security/replay tests |
| Миграция | Inventory, mapping, clean, dry-run | Reconciliation/sign-off |
| Обучение | Role-based synthetic scenarios | UAT |
| Запуск | Cutover, communication, fallback, rollback | Go/no-go |
| Stabilization | Hypercare, baseline and incidents | Exit criteria |

~~~text
Inventory -> Dependency / Identity / Threat Review -> Test Configuration
 -> Synthetic UAT -> Migration Dry-run -> Go/No-Go
 -> Release -> Smoke / Reconciliation -> Hypercare or Rollback
~~~

Existing fields, stages, roles, automations and integrations are not removed or renamed before dependency analysis, backup/export, pilot and approved rollback. Product and billing production writes use isolated test tenants first.

## 19. Monitoring

| Signal | Rule | Action | Owner |
|---|---|---|---|
| Lost Lead | Source count/no ack | Incident/replay | Sales operations |
| Overdue onboarding | Milestone threshold | Review/escalate | Onboarding owner |
| Churn signal | Approved fresh components | CSM review/playbook | CSM |
| Renewal at risk | Billing window/blocker | Account task/escalation | Account manager |
| Support SLA breach | Calendar/priority threshold | Escalate | Support lead |
| Integration error | Error/retry/DLQ | Investigate/replay | Integration support |
| Stale usage | Freshness exceeded | Mark unknown/block score | Data/product owner |
| Billing mismatch | Control totals differ | Incident/reconcile | Finance owner |
| Access anomaly | Unexpected view/export | Security incident | Security owner |

Alert includes system, tenant/account reference, correlation ID, timestamp, severity, owner and runbook; without secrets, raw usage, credentials, payment details or unnecessary personal data.

## 20. Risks

| № | Risk | Cause | Consequence | Mitigation |
|---:|---|---|---|---|
| 1 | Lead lost | No ack | Lost revenue | Intake reconciliation |
| 2 | Duplicate Lead/Deal | Retry/multiple channels | Double work | Source key/dedup |
| 3 | Company equals SaaS Account | Flat model | Broken tenant history | Separate entities |
| 4 | Domain is only account key | Weak identity | Wrong tenant merge | Stable IDs |
| 5 | Product user becomes Contact automatically | Overprojection | Data bloat/privacy | Role-based projection |
| 6 | Payer equals user organization | Simplification | Wrong billing relation | Explicit roles |
| 7 | Deal equals Subscription | Sales-centric model | Wrong entitlement | Billing master |
| 8 | Deal close provisions access | Unsafe automation | Unauthorized tenant | Product gate |
| 9 | Plan overwritten | No versioning | Historical mismatch | Effective versions |
| 10 | Subscription status manual | Convenience | False lifecycle | Billing projection |
| 11 | Upgrade edits old Deal | One-record habit | No expansion history | New Deal/change event |
| 12 | Renewal overwrites term | No versions | Audit loss | Billing term history |
| 13 | Cancellation before effective source event | Premature CRM action | Wrong churn | Authoritative state |
| 14 | Onboarding stored in Deal stages only | Under-modeling | No milestones | Onboarding entity |
| 15 | Handover incomplete | Sales closes early | Failed activation | Readiness gate |
| 16 | Raw usage copied to CRM | Overintegration | Load/privacy risk | Aggregates |
| 17 | Usage signal stale | No freshness | Wrong CS action | Freshness state |
| 18 | Health score opaque | One magic number | Untrusted decisions | Explainable model |
| 19 | Missing data means unhealthy | Bad default | False churn risk | Unknown state |
| 20 | Churn action fully automated | Automation optimism | Harmful outreach | Human review |
| 21 | Support case and incident merged | One object | Wrong SLA/grain | Separate relation |
| 22 | Priority free-text | No matrix | Wrong response | Impact/urgency |
| 23 | Entitlement unavailable | Helpdesk disconnected | Wrong support | Source check/fallback |
| 24 | Product feedback promises roadmap | No product gate | Customer commitment | Decision status |
| 25 | Payment equals Deal success | CRM-only finance | False MRR | Billing facts |
| 26 | MRR from Deal amount | Wrong grain | False reporting | Subscription components |
| 27 | ARR formula undefined | Shortcut | Incomparable KPI | Metric contract |
| 28 | Churn denominator unstable | No cohort rule | False trend | Governed formula |
| 29 | LTV presented as fact | Model assumptions hidden | Bad investment | Versioned model |
| 30 | Cross-tenant CRM access | Broad rights | Data breach | Tenant/account scopes |
| 31 | Support sees commercial secrets | Shared record | Excess access | Role minimum |
| 32 | Admin is business super-reader | Role confusion | Uncontrolled access | Separate privileges |
| 33 | Export unrestricted | Defaults | Mass leak | Approval/minimum |
| 34 | Secret in Git | Real example | Compromise | Secret store/rotation |
| 35 | Broad integration token | One account | Large blast radius | Least privilege |
| 36 | Raw payload logged | Debugging | Customer-data leak | Masking |
| 37 | Infinite retry | No DLQ | Load/outage | Bounded backoff |
| 38 | Retry duplicates Account/Subscription | No idempotency | Double lifecycle | Source key |
| 39 | Out-of-order event | Async ignored | Old status wins | Version/order guard |
| 40 | Schema drift | Source change | Corrupt projection | Contract validation |
| 41 | No correlation ID | Fragmented logs | Slow RCA | End-to-end trace |
| 42 | Free bidirectional edit | No master | Ping-pong | Field ownership |
| 43 | Connector assumed | Unverified feature | Blocked release | Official/vendor check |
| 44 | Migration without inventory | Unknown scope | Missing relations | Source report |
| 45 | Import without dry-run | Deadline pressure | Mass corruption | Sample/report/rollback |
| 46 | Real tenant data in UAT | Prod copy | Confidentiality breach | Synthetic tenants |
| 47 | No reconciliation | HTTP success trusted | Silent loss | Counts/control totals |
| 48 | Peak event load untested | Average design | Queue collapse | Load/degraded mode |
| 49 | Tariff not checked | Assumption | Feature unavailable | Portal assessment |
| 50 | Cloud assumes DB/event runtime | On-Premise habit | Impossible design | External integration |
| 51 | On-Premise core modified | Quick fix | Upgrade/security failure | Extensions only |
| 52 | Release lacks tenant rights tests | Happy path | Hidden exposure | Negative suite |
| 53 | Rollback untested | Paper plan | Long outage | Restore drill |
| 54 | Alert has no owner | Notification only | Incident unresolved | RACI/runbook |
| 55 | DR ignores product/billing/helpdesk | CRM-only recovery | Service not restored | End-to-end DR |

## 21. ADR

| ADR | Decision | Alternative | Rationale | Consequence |
|---|---|---|---|---|
| ADR-SAAS-001 | SaaS Account is separate from Company/Contact | Store tenant on Company | Supports B2B/B2C and multiple tenants | Identity mapping |
| ADR-SAAS-002 | Subscription is read-oriented billing projection | Deal as Subscription | Billing owns term/status/entitlement | Reconciliation required |
| ADR-SAAS-003 | Plan is effective-versioned | Mutable tariff field | Historical correctness | Catalog governance |
| ADR-SAAS-004 | Onboarding is separate lifecycle | Deal stages only | Milestones/dependencies need delivery object | Handover relation |
| ADR-SAAS-005 | Support master chosen by SLA/volume | Always CRM or always helpdesk | Avoids tool-driven design | Projection contract |
| ADR-SAAS-006 | Integrations use governed middleware/contracts | Point-to-point webhooks | Idempotency/security/monitoring | Platform owner |
| ADR-SAAS-007 | Rights are tenant/account/purpose scoped | Organization-wide read | Least privilege | Negative tests |
| ADR-SAAS-008 | BI uses reconciled billing/product grains | Raw CRM counters | Reliable recurring metrics | Metric/DQ ownership |

Each ADR has owner, status proposed/accepted/superseded, date, context, alternatives, consequences, portal/version and evidence.

## 22. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Нет Discovery | Сразу настраивают CRM | Wrong scope | AS IS/stakeholders |
| 2 | Cloud/On-Premise неизвестен | Template design | Unsupported solution | Capability matrix |
| 3 | Тариф не проверен | Assumption | Missing function | Verify target portal |
| 4 | Нет system/data owners | IT-only project | No decisions | RACI |
| 5 | All inputs are Leads | Universal pattern | Support/users mixed with sales | Classification |
| 6 | Trial account is Lead only | Sales-only view | Product lifecycle invisible | SaaS Account |
| 7 | Company equals tenant | Flat identity | Multi-account failure | Separate Account |
| 8 | Product user equals CRM Contact | Auto-sync everything | Data bloat/privacy | Approved roles only |
| 9 | Domain is unique key | Convenience | Wrong merges | Stable product IDs |
| 10 | Payer relation omitted | One-company assumption | Billing mismatch | Explicit payer |
| 11 | Deal equals Subscription | CRM-first model | Wrong status/term | Billing projection |
| 12 | One Deal for all expansions | Convenience | No revenue history | Separate opportunities |
| 13 | Renewal overwrites original Deal | One-record habit | Audit loss | Renewal policy/new Deal |
| 14 | Plan is free text | No catalog | Inconsistent analytics | Versioned Plan |
| 15 | Plan update changes history | Mutable dictionary | Wrong entitlement | Effective versions |
| 16 | Quantity and users mixed | Wrong grain | Billing error | Separate measures |
| 17 | Subscription status manual | No integration | False lifecycle | Billing master |
| 18 | Deal robot provisions product | Overautomation | Unauthorized access | Product command gate |
| 19 | Payment marked by sales | Convenience | Financial error | Billing/finance source |
| 20 | Invoice status copied manually | No reconcile | Stale debt | Authoritative sync |
| 21 | Onboarding lives only in Deal | Under-modeling | No delivery control | Onboarding entity |
| 22 | Handover has no acceptance | Sales closes early | Lost requirements | Readiness checklist |
| 23 | Client integration stored in comments | Quick setup | No ownership/health | Integration entity |
| 24 | Milestones are Deal stages | Process mixing | False sales cycle | Delivery lifecycle |
| 25 | No success criteria | Activation equals value | CS disputes | Agreed outcomes |
| 26 | Raw telemetry copied to CRM | Overintegration | Load/privacy | Aggregates |
| 27 | Usage metric has no window | Simplification | False trend | Metric contract |
| 28 | Usage freshness hidden | No timestamp | Wrong action | Freshness state |
| 29 | Missing usage means zero | Bad default | False churn signal | Unknown state |
| 30 | Health score is magic number | No components/version | No trust | Explainable model |
| 31 | Health auto-cancels or discounts | Automation optimism | Financial/customer harm | Human approval |
| 32 | Support case equals incident | One object | Wrong SLA | Separate relation |
| 33 | Priority subjective | No impact matrix | Wrong response | Governed priority |
| 34 | SLA ignores entitlement | Generic support | Contract breach/cost | Entitlement check |
| 35 | SLA calendar absent | Simplification | False breach | Calendar/pause |
| 36 | Case closes on notification | Send equals resolve | Unresolved issue | Verification gate |
| 37 | Product feedback is roadmap promise | Sales pressure | False commitment | Product decision state |
| 38 | Feature name is entitlement | Concept mixing | Unauthorized use | Product source |
| 39 | Marketing and transactional messages mixed | One automation | Wrong communication | Separate purposes |
| 40 | Upsell fires from one raw event | No review | Spam/wrong offer | Governed signal |
| 41 | Cross-sell ignores compatibility | Catalog gap | Impossible sale | Eligibility |
| 42 | Anonymous users become Contacts | Tracking habit | Data bloat | Aggregated audience |
| 43 | MRR is Deal amount divided by 12 | Shortcut | False KPI | Billing components |
| 44 | ARR formula changes silently | No version | Incomparable periods | Metric version |
| 45 | Churn uses wrong denominator | Dashboard default | False retention | Cohort contract |
| 46 | Expansion included inconsistently | No NRR rule | Misleading metric | Finance-approved formula |
| 47 | LTV shown without assumptions | Model as fact | Bad decisions | Versioned assumptions |
| 48 | Currency/FX omitted | Single-currency assumption | Wrong totals | FX policy |
| 49 | Sales sees support secrets | Shared record | Excess disclosure | Summary projection |
| 50 | Support sees pricing/contracts broadly | Common access | Commercial leak | Functional minimum |
| 51 | Product manager sees raw PII | Analytics shortcut | Privacy risk | Pseudonymization |
| 52 | Admin equals business reader | Role confusion | Uncontrolled access | Separate privileges |
| 53 | Cross-tenant rights not tested | Happy path | Data breach | Negative matrix |
| 54 | Export unrestricted | Defaults | Mass leak | Approval/minimum |
| 55 | Secret in README | Real example | Compromise | Placeholder/secret store |
| 56 | Webhook committed | Quick integration | Unauthorized calls | Environment/rotation |
| 57 | One broad integration token | Simplicity | Large blast radius | Purpose scopes |
| 58 | Raw payload logged | Debugging | Customer data leak | Masking |
| 59 | Infinite retry | No DLQ | Load/outage | Bounded retry |
| 60 | Retry duplicates tenant/subscription | No idempotency | Double lifecycle | Source event key |
| 61 | Out-of-order ignored | Async not designed | Old status wins | Version/order guard |
| 62 | Schema not validated | Trust source | Corrupt data | Contract validation |
| 63 | No correlation ID | Fragmented logs | Slow RCA | Trace ID |
| 64 | Free bidirectional edits | No master | Ping-pong | Field ownership |
| 65 | Connector invented | Guessing | Blocked release | Official/vendor verification |
| 66 | REST method invented | No reference check | Broken integration | API docs |
| 67 | Batch ignores limits | Premature optimization | Partial failure | Official check/report |
| 68 | Migration has no inventory | Unknown sources | Missing relations | Source/volume report |
| 69 | Import without dry-run | Time pressure | Mass corruption | Sample/dry-run |
| 70 | Real tenant/user data in UAT | Prod copy | Confidentiality breach | Synthetic data |
| 71 | Migration ignores account hierarchy | Flat mapping | Wrong tenant/customer link | Identity map |
| 72 | No reconciliation | HTTP success trusted | Silent loss | Counts/control totals |
| 73 | BI mixes Account/Subscription/Invoice | Wrong grain | False recurring metrics | Metric contract |
| 74 | Onboarding duration uses Deal dates | Wrong source | False CS KPI | Milestone events |
| 75 | Team ranking ignores account complexity | KPI pressure | Harmful management | Context/capacity |
| 76 | Peak event load untested | Average design | Queue collapse | Load/degraded mode |
| 77 | Old fields deleted immediately | Cleanup | Broken dependencies | Deprecation scan |
| 78 | Release without smoke/rollback | Deadline pressure | Long incident | Checklist/drill |
| 79 | Cloud assumes DB/event runtime | On-Premise habit | Impossible design | External boundary |
| 80 | On-Premise core modified | Quick fix | Upgrade/security failure | Extensions only |

## 23. Implementation Roadmap

| Stage | Result | Minimal artifacts | Ready criterion |
|---:|---|---|---|
| 1. Discovery | Confirmed AS IS/scope | Stakeholders, product, billing, systems, tariff, volumes | Facts/open questions approved |
| 2. Архитектура | Approved TO BE | Entity/grain model, master matrix, threat model, rights, ADR | Architecture/security review |
| 3. CRM настройка | Governed identity/sales model | Fields, stages, roles, automation register | Synthetic sales paths pass |
| 4. Продажи | Controlled opportunity lifecycle | Discovery, demo, proposal, contract, handover | Sales UAT passes |
| 5. Подписки | Reliable billing projection | Account/Plan/Subscription mappings, reconcile | Billing lifecycle tests pass |
| 6. Customer Success | Onboarding/health/renewal operations | Milestones, signals, playbooks, cases | End-to-end UAT passes |
| 7. Интеграции | Secure reliable exchange | Contracts, secrets, retry, idempotency, monitoring | Failure/replay/reconcile pass |
| 8. Тестирование | Evidence-based quality | Synthetic data, rights/load/rollback tests | Critical defects closed/sign-off |
| 9. Запуск | Controlled release | Cutover, backup/export, fallback, rollback | Smoke/reconciliation pass |
| 10. Оптимизация | Measured improvement | Baseline, DQ, incidents and backlog | KPI change verified |

### Критерии готовности

- confirmed Cloud/On-Premise, tariff/edition, version, region, rights and test environments;
- approved B2B/B2C identity, Company–Account–Subscription model, masters, rights, metrics and ADR;
- product, billing, helpdesk, IdP and DWH facts are not replaced by manual CRM fields;
- synthetic UAT, negative tenant rights, billing, freshness, failure, load, migration dry-run and reconciliation tests pass;
- secrets are outside Git; logs/alerts have no credentials, raw telemetry, payment details or unnecessary PII;
- cutover, rollback, DR, degraded mode, monitoring and support RACI are approved;
- sales, Customer Success, support, product, billing/finance, data and security owners accept results.

### Официальные источники

Использованы только записи из knowledge/SOURCES.md. Перед реализацией URL, дату обновления, тарифную и версионную применимость проверяют повторно.

- [Официальная пользовательская документация Bitrix24](https://helpdesk.bitrix24.ru/) — CRM, tasks, rights, automation and Cloud.
- [Официальная документация REST API](https://apidocs.bitrix24.ru/) — integrations, authorization and limits.
- [REST API reference](https://apidocs.bitrix24.ru/api-reference/) — methods, parameters and scopes.
- [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/) — official integration tooling.
- [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) — Smart Process creation and configuration.
- [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) — штатная automation.
- [Бизнес-процессы](https://helpdesk.bitrix24.ru/open/23951294/) — process type selection.
- [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) — official connection options.
- [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) — datasets and limitations.
- [Маркетплейс Bitrix24](https://www.bitrix24.ru/apps/) — apps with separate vendor/security review.
- [Документация On-Premise](https://dev.1c-bitrix.ru/docs/portal.php) — box development without core changes.

### Ограничения и открытые вопросы

1. Cloud or On-Premise, tariff/edition, version, region and actual permissions.
2. B2B/B2C, sales-assisted/self-service, trial/freemium, contract and renewal models.
3. Master Company, payer, SaaS Account, Product User, Plan, Subscription, entitlement, usage, invoice and payment.
4. Product, billing, helpdesk, IdP, website, telephony and BI APIs, event volumes, limits and support.
5. Identity mapping, tenant isolation, regions, retention, communication eligibility and data classification.
6. Provisioning, onboarding, activation, support entitlement, upgrade/downgrade, suspension, cancellation and renewal rules.
7. Health model components, freshness, thresholds, owner, review and playbooks.
8. MRR, ARR, churn, GRR/NRR, retention, LTV, currency/FX and revenue-recognition boundaries.
9. Team/tenant rights, exports, logs, temporary access and recertification.
10. Test tenants, release window, downtime, rollback, degraded mode, DR and support RACI.

Документ не подтверждает юридическую модель конкретной подписки, revenue recognition, SLA, product access or marketing eligibility. Эти решения требуют отдельной договорной, финансовой, правовой and information-security review.
