# Bitrix24 Architecture Patterns Catalog

Инженерный каталог повторяемых решений для проектирования Bitrix24. Паттерны извлечены из 25 обезличенных кейсов Case Library и применяются только после Discovery целевого портала.

> Все примеры синтетические. Доступность CRM, Smart Processes, роботов, триггеров, бизнес-процессов, BI, телефонии, REST and Marketplace applications зависит от Cloud/On-Premise, тарифа или редакции, версии, региона и прав. Конкретная возможность подтверждается по [официальному реестру источников](../knowledge/SOURCES.md) и на целевом портале.

Стандарт карточки паттерна:

~~~text
Проблема -> Когда применять -> Решение -> Архитектура
 -> Контроль -> Ограничения и риски
~~~

Паттерн описывает способ принятия решения. Он не является готовой настройкой, набором полей или гарантией наличия функции.

## 1. Назначение архитектурных паттернов

Архитектурный паттерн — повторяемая модель решения типовой проблемы с известным контекстом, trade-offs, контролями и ограничениями. Он помогает обсуждать архитектуру на уровне бизнес-смысла, а не начинать с интерфейса или конкретного инструмента.

Паттерны нужны, чтобы:

- одинаково называть повторяющиеся решения;
- быстрее проводить Discovery and solution workshops;
- разделять business entity, process, automation and integration;
- повторно использовать проверенные controls, tests and ADR;
- сравнивать варианты и фиксировать причины выбора;
- не копировать отраслевой кейс целиком.

| Паттерн | Готовое решение |
|---|---|
| Описывает проблему, контекст, структуру и последствия | Описывает конкретную конфигурацию целевого портала |
| Требует адаптации | Применяется только к подтверждённому scope |
| Не содержит реальных IDs, секретов и клиентских данных | Может включать локальные реестры, которые не попадают в Git |
| Допускает альтернативы | Фиксирует принятый ADR |
| Проверяется на уровне принципов и acceptance criteria | Проверяется тестами конкретной реализации |

~~~text
Case Library -> Наблюдаемые повторения -> Pattern Catalog
      |                                      |
      +-> отраслевой контекст                +-> reusable decision
                                                   |
Discovery -> Pattern selection -> Adaptation -> ADR -> Implementation
~~~

Case Library показывает сочетания паттернов в отраслях. PATTERNS.md позволяет собрать новую архитектуру из совместимых решений, сохраняя единые правила data ownership, security, tests and operations.

## 2. CRM Core Patterns

### 2.1 CRM Sales Pipeline Pattern

- **Проблема:** коммерческие возможности ведутся в задачах, письмах или одной неуправляемой стадии.
- **Когда применять:** есть отдельные opportunities с owner, суммой или ценностью, этапами и итогом.
- **Решение:** одна Deal на одну коммерческую возможность; Contact/Company описывают клиента, activities and tasks — работу.
- **Архитектура:** Lead when justified → Contact/Company → Deal → outcome → repeat opportunity.
- **Автоматизация:** owner assignment, next-step task, stage gates, reminders, structured loss reason.
- **Ограничения и риски:** стадии не заменяют tasks; Deal не является order, project, subscription or payment master.

~~~text
Lead or Qualified Request -> Contact / Company -> Deal Pipeline
 -> Offer / Contract Gate -> Won or Lost -> New Opportunity
~~~

### 2.2 Multi Pipeline Pattern

- **Проблема:** разные бизнес-направления используют одну воронку с взаимоисключающими стадиями и полями.
- **Когда применять:** направления имеют разные lifecycle, owners, permissions, automation or analytics.
- **Решение:** отдельные направления сделок с общей customer identity and governed dictionaries.
- **Архитектура:** shared Contact/Company → direction-specific Deal pipelines → common account view.
- **Автоматизация:** правила назначения и обязательности по направлению; межвороночная передача только по явному contract.
- **Ограничения и риски:** нельзя создавать направление только ради отчёта; дублирование клиента между направлениями запрещено без основания.

### 2.3 Lead Qualification Pattern

- **Проблема:** неквалифицированный поток сразу создаёт Deals и искажает pipeline.
- **Когда применять:** требуется отделить inquiry, spam, duplicate and no-fit от подтверждённой opportunity.
- **Решение:** Lead служит временным qualification object; модель без лидов допускается как отдельный ADR.
- **Архитектура:** source → Lead → validate/dedup/qualify → Contact/Company/Deal or structured rejection.
- **Автоматизация:** intake acknowledgement, SLA, owner, duplicate review and qualification checklist.
- **Ограничения и риски:** Lead не используется для support cases, orders or отраслевых entities; conversion rules проверяются на портале.

### 2.4 Customer 360 Pattern

- **Проблема:** клиентская история раздроблена между отделами и системами.
- **Когда применять:** нужен единый навигационный профиль с продажами, сервисом, проектами и внешними facts.
- **Решение:** Contact/Company — identity anchor; связанные objects и минимальные projections формируют 360-view.
- **Архитектура:** identity → relationships → Deals/Smart Processes/activities → governed external summaries.
- **Автоматизация:** identity resolution, freshness checks, relationship updates and exception queue.
- **Ограничения и риски:** Customer 360 не означает копирование всех данных в CRM; sensitive data and raw logs stay in masters.

### 2.5 Account Management Pattern

- **Проблема:** B2B-клиент сводится к одной сделке и теряется после продажи.
- **Когда применять:** есть несколько contacts, opportunities, sites, contracts, services or renewals.
- **Решение:** Company is account anchor; roles, account owner, plan and opportunities are explicit.
- **Архитектура:** Company → Contacts/roles → Deals → delivery/service → account plan → repeat Deal.
- **Автоматизация:** inactivity review, relationship gap task, renewal/expansion review and risk escalation.
- **Ограничения и риски:** account plan does not overwrite transactional history; ranking requires context and data quality.

## 3. Smart Process Patterns

### 3.1 Master Data Pattern

- **Проблема:** объекты, оборудование, товары или услуги хранятся в Deal fields and comments.
- **Когда применять:** entity has stable identity, reusable attributes and links to many transactions.
- **Решение:** separate Smart Process as governed CRM projection or master when formally approved.
- **Архитектура:** Master entity → version/status → relations to Company, Deal, service and external ID.
- **Контроль:** stable key, owner, field dictionary, lifecycle, archive, permissions and reconciliation.
- **Ограничения:** CRM must not replace ERP, PIM, EAM or product platform without scope and capacity proof.

### 3.2 Transaction Entity Pattern

- **Проблема:** order, booking or contract events overwrite each other in one client card.
- **Когда применять:** every transaction needs its own amount, dates, state, participants and outcome.
- **Решение:** one Smart Process item per business transaction.
- **Архитектура:** Customer → Transaction → lines/resources/documents → completion/cancellation.
- **Контроль:** business key, idempotency, status transitions, authoritative finance/document facts.
- **Ограничения:** do not duplicate Deal when it already expresses the exact transaction semantics.

### 3.3 Lifecycle Management Pattern

- **Проблема:** object state is represented by unrelated tasks or free text.
- **Когда применять:** entity has meaningful states, allowed transitions, exceptions and SLA.
- **Решение:** Smart Process stages represent states; tasks represent work needed to change state.
- **Архитектура:** create → validate → active states → review → terminal outcome → archive/reopen.
- **Контроль:** transition gate is an architectural requirement; required fields by stage, re-entry and loop protection.
- **Ограничения:** gate implementation is selected after portal verification from available required fields, permissions, robots, business processes or external validation; excessive stages reduce adoption, and lifecycle depends on tariff/version. Официальная проверка обязательности по стадиям: [пользовательские поля в счетах и смарт-процессах](https://helpdesk.bitrix24.ru/open/22088566/) (проверено 2026-08-10).

### 3.4 Relationship Pattern

- **Проблема:** client, asset, sale and service context are flattened into repeated fields.
- **Когда применять:** one client owns or uses many objects, and each object has many Deals or service cases.
- **Решение:** explicit relations preserve identity and history.
- **Архитектура:** Customer → Object → Deal/Transaction → Service Case.
- **Контроль:** cardinality, effective dates, relationship roles, delete/archive behavior and access inheritance.
- **Ограничения:** available relation mechanisms and permissions must be confirmed; names are not reliable keys.

~~~text
Contact / Company 1 ---- N Object
                           |
                           +---- N Deal / Transaction
                           |
                           +---- N Service Case / Task / Document
~~~

## 4. Service Management Patterns

### 4.1 SLA Management Pattern

- **Проблема:** response and resolution times are informal or measured from wrong timestamps.
- **Процесс:** intake → classification → assignment → work → verification → closure.
- **Стадии:** New, Triaged, Assigned, In Work, Waiting by approved reason, Review, Closed.
- **Автоматизация:** deadline calculation, threshold reminders, escalation and pause audit.
- **Контроль:** SLA version, calendar, priority, entitlement, source timestamps and breach reason.
- **Ограничения:** internal SLA is not automatically a statutory or contractual deadline.

### 4.2 Ticket Management Pattern

- **Проблема:** questions, incidents, complaints and sales requests are mixed.
- **Процесс:** identify request type → create case → triage → resolve/escalate → verify.
- **Стадии:** New, Classified, Assigned, In Progress, Resolved, Closed/Reopened.
- **Автоматизация:** dedup, routing, task creation, customer update and integration error queue.
- **Контроль:** one case ID, owner, priority matrix, next action, outcome and related incident.
- **Ограничения:** Bitrix24 master or external helpdesk projection is selected by volume, SLA, security and feature fit.

### 4.3 Escalation Pattern

- **Проблема:** overdue or high-impact work produces notifications without accountable action.
- **Процесс:** threshold/event → validate severity → assign escalation owner → acknowledge → resolve.
- **Стадии:** Detected, Assigned, Acknowledged, Action, Resolved, Reviewed.
- **Автоматизация:** one escalation per threshold, tier progression and fallback owner.
- **Контроль:** RACI, acknowledgement time, runbook, correlation ID and closure evidence.
- **Ограничения:** escalation does not replace incident management or emergency channels.

### 4.4 Customer Support Pattern

- **Проблема:** customer service lacks shared context, ownership and feedback loop.
- **Процесс:** identity/entitlement → request → resolve → verify → learn.
- **Стадии:** Intake, Qualification, Service, Verification, Closed/Reopened.
- **Автоматизация:** routing, SLA, approved transactional updates and repeat-contact linkage.
- **Контроль:** least-privilege customer view, resolution codes, reopen and quality sampling.
- **Ограничения:** support and sales remain separate grains even when shown in Customer 360.

~~~text
Customer -> Support Case -> SLA / Owner -> Resolution
                     |          |
                     v          v
              Incident/Problem  Escalation -> Review -> Knowledge/Change
~~~

## 5. Project Management Patterns

### 5.1 Project Delivery Pattern

- **Проблема:** successful Deal is treated as completed delivery.
- **Когда применять:** contract leads to multi-step work, milestones and deliverables.
- **Решение:** separate Project or Smart Process linked to Deal.
- **Архитектура:** Deal/contract → handover → project baseline → milestones → deliverables → acceptance.
- **Контроль:** scope version, RACI, dependencies, risk/change registers and closure gate.
- **Ограничения:** CRM project coordination may not replace specialized PM/ERP master.

### 5.2 Task Execution Pattern

- **Проблема:** stages are overloaded with detailed actions.
- **Когда применять:** work has assignee, deadline, input and expected result.
- **Решение:** task represents executable work; lifecycle stage represents business state.
- **Архитектура:** state entry/event → task → result/evidence → reviewed transition.
- **Контроль:** unique task key, re-entry behavior, overdue escalation and acceptance.
- **Ограничения:** task completion alone may not prove transaction or project completion.

### 5.3 Approval Workflow Pattern

- **Проблема:** price, contract, document or decision approvals happen in chats.
- **Когда применять:** approval has defined authority, version, thresholds and rejection paths.
- **Решение:** business process or governed automation route on a specific object version.
- **Архитектура:** submit version → reviewer sequence/parallel route → approve/reject/rework → audit.
- **Контроль:** authority, substitution, deadline, immutable version, reason and timeout escalation.
- **Ограничения:** automation cannot provide legal signature or authority not present outside the workflow.

### 5.4 Document Workflow Pattern

- **Проблема:** multiple uncontrolled file copies are called the final document.
- **Когда применять:** proposal, contract, report, act or regulated document needs version and approval.
- **Решение:** CRM stores metadata and protected link; DMS/EDO remains master when required.
- **Архитектура:** document card → version → approval → signature/status source → retention/archive.
- **Контроль:** access, version, checksum/status, effective dates, retention and revoked links.
- **Ограничения:** a CRM file or checkbox is not automatically legally significant.

## 6. Integration Patterns

### 6.1 REST Integration Pattern

- **Проблема:** systems need controlled read/write exchange beyond native settings.
- **Система:** Bitrix24 and an authorized external application/service.
- **Данные:** minimal contract fields with stable business and external IDs.
- **Метод:** documented REST methods and scopes verified in official API reference.
- **Контроль ошибок:** timeout, HTTP errors, bounded retry, idempotency, pagination/batch rules and reconciliation.
- **Ограничения:** methods, limits and fields are never inferred; Cloud has no direct DB/core access.

### 6.2 Webhook Event Pattern

- **Проблема:** external system must react to a Bitrix24 event or call a limited REST operation.
- **Система:** event producer → protected endpoint → consumer.
- **Данные:** event ID, object reference, version/timestamp and minimal payload.
- **Метод:** verified inbound/outbound webhook scenario or application event.
- **Контроль ошибок:** authentication, dedup, ordering, replay, DLQ and correlation.
- **Ограничения:** webhook URL is a secret. The handler is designed defensively for duplicate delivery, unknown outcomes and changed ordering; this is an architectural safeguard, not a universal Bitrix24 guarantee of at-least-once or unordered delivery. The exact event contract is verified on its official page, for example [`event.bind`](https://apidocs.bitrix24.ru/api-reference/events/event-bind.html) (checked 2026-08-10).

### 6.3 External System Master Pattern

- **Проблема:** CRM users edit finance, product, inventory or document facts that belong elsewhere.
- **Система:** authoritative ERP/1C/PIM/DMS/product platform → integration layer → CRM projection.
- **Данные:** read-only or controlled subset, external ID, source timestamp and freshness.
- **Метод:** approved API, application, middleware or supported connector.
- **Контроль ошибок:** ownership matrix, stale-state marker, conflict policy and reconciliation.
- **Ограничения:** no free bidirectional editing; source outage must be visible.

### 6.4 Data Synchronization Pattern

- **Проблема:** replicated records drift or duplicate across systems.
- **Система:** source and target with explicit direction per field/object.
- **Данные:** identity keys, version, changed attributes and operation type.
- **Метод:** event, scheduled incremental sync or controlled batch.
- **Контроль ошибок:** idempotency, ordering, retry budget, quarantine, counts and sample reconciliation.
- **Ограничения:** two-way sync requires a conflict policy; last-write-wins is not a default.

### 6.5 1C Integration Pattern

- **Проблема:** CRM needs counterparties, products, invoices, payments or accounting status.
- **Система:** Bitrix24 ↔ approved 1C configuration through supported integration or middleware.
- **Данные:** explicitly owned entities, stable IDs, currency, amounts and document statuses.
- **Метод:** verified standard integration/application or documented contract.
- **Контроль ошибок:** mapping, duplicate keys, partial batch report, amount/control-total reconciliation.
- **Ограничения:** compatibility depends on editions, configurations and versions; 1C finance facts are not manually replaced in CRM.

~~~text
Source System -> Authenticate -> Validate -> Map -> Idempotency
 -> Bitrix24 Projection -> Result / Retry / Quarantine
 -> Reconciliation and Monitoring
~~~

## 7. Data Architecture Patterns

### 7.1 Data Mapping Pattern

- **Проблема:** source and target fields have different semantics, types and dictionaries.
- **Когда применять:** migration or integration transforms records.
- **Решение:** versioned mapping register with source, target, conversion, owner and validation.
- **Архитектура:** source schema → transform rules → target schema → rejected-record queue.
- **Контроль:** types, mandatory values, units, currency, timezone, dictionaries and default prohibition.
- **Ограничения:** mapping cannot repair undefined business semantics.

### 7.2 Deduplication Pattern

- **Проблема:** the same person, company or transaction appears several times.
- **Когда применять:** multi-channel intake, imports and cross-system identity.
- **Решение:** conservative matching with strong keys, candidate matches and human review.
- **Архитектура:** normalize → exact rules → probable-match queue → merge/link decision → audit.
- **Контроль:** reversible map, protected fields, dependency scan and false-merge tests.
- **Ограничения:** name, phone or email alone may be insufficient; automatic destructive merge is unsafe.

### 7.3 Data Quality Pattern

- **Проблема:** automation and reports operate on incomplete, stale or contradictory data.
- **Когда применять:** every operational and analytical solution.
- **Решение:** data contracts and measurable completeness, validity, uniqueness, consistency and freshness.
- **Архитектура:** source → validation → quality flags/quarantine → usable dataset → remediation.
- **Контроль:** owner, thresholds, score components, trend and blocking rules.
- **Ограничения:** a single quality percentage can hide critical field failures.

### 7.4 Migration Pattern

- **Проблема:** legacy data must move without loss or corruption.
- **Когда применять:** portal consolidation, system replacement or initial import.
- **Решение:** inventory → cleanse → map → synthetic/sample dry-run → controlled batches → reconcile.
- **Архитектура:** source snapshot → transformation → test import → acceptance → production cutover.
- **Контроль:** counts, totals, samples, rejects, idempotency, backup/export and rollback.
- **Ограничения:** HTTP success is not migration success; history and relations need separate validation.

### 7.5 Archive Pattern

- **Проблема:** active CRM is overloaded, while records must remain controlled.
- **Когда применять:** completed or expired objects have approved retention rules.
- **Решение:** distinguish active, archived, backup and legally held states.
- **Архитектура:** lifecycle outcome → retention decision → restricted archive/export → verified restore/delete.
- **Контроль:** owner, retention class, access, legal hold, restore test and deletion evidence.
- **Ограничения:** archive is not backup; deletion and retention depend on applicable policy.

## 8. Security Patterns

### 8.1 Role Based Access Pattern

- **Проблема:** access follows convenience rather than responsibility.
- **Когда применять:** always.
- **Решение:** permissions based on role, purpose, entity and operation.
- **Архитектура:** identity → role → scope → allowed actions → audit.
- **Контроль:** least privilege, named accounts, recertification, negative tests and temporary expiry.
- **Ограничения:** platform administrator is not automatically business-data owner.

### 8.2 Department Isolation Pattern

- **Проблема:** teams, regions, tenants or clients can see each other’s records.
- **Когда применять:** segmented organizations and confidential projects.
- **Решение:** explicit department/team/project/tenant scopes with controlled shared views.
- **Архитектура:** user role + organizational scope → record access → approved cross-scope handoff.
- **Контроль:** cross-unit negative tests, export restrictions and relation inheritance review.
- **Ограничения:** exact rights behavior must be tested on target portal and may depend on entity/settings.

### 8.3 Audit Pattern

- **Проблема:** changes and integration actions cannot be reconstructed.
- **Когда применять:** business-critical fields, rights, automation, documents and integrations.
- **Решение:** combine CRM history with masked integration/security logs using correlation IDs.
- **Архитектура:** actor/event → before/after or status → timestamp → source → result.
- **Контроль:** clock/timezone, retention, tamper protection, access and trace sampling.
- **Ограничения:** available native history differs by feature/version; external logs may be required.

### 8.4 Sensitive Data Protection Pattern

- **Проблема:** personal, financial, medical, HR or commercial data spreads into fields, files and logs.
- **Когда применять:** any restricted data.
- **Решение:** minimize, classify, segregate, encrypt/protect in approved systems and mask operational traces.
- **Архитектура:** purpose → minimum dataset → access scope → protected storage/link → retention.
- **Контроль:** field/file inventory, secret scan, export approval, revoked access and incident response.
- **Ограничения:** legal basis and retention are external requirements and are not invented by CRM design.

## 9. Automation Patterns

### 9.1 Notification Pattern

- **Проблема:** stakeholders do not learn about a relevant state change.
- **Когда применять:** a confirmed event requires information, not work.
- **Решение:** send one role-appropriate notification based on current state.
- **Архитектура:** event → eligibility/recipient/version check → notify → delivery result.
- **Контроль:** dedup, channel, timezone, template version, suppression and masking.
- **Ограничения:** notification is not acknowledgement, task or business result.

### 9.2 Reminder Pattern

- **Проблема:** planned action approaches a deadline without progress.
- **Когда применять:** owner and due date exist.
- **Решение:** scheduled reminder with escalation only after defined thresholds.
- **Архитектура:** due date/calendar → pre-threshold → remind → overdue review.
- **Контроль:** one reminder per threshold, completed-state check and pause rules.
- **Ограничения:** repeated messages cannot compensate for missing owner or broken process.

### 9.3 Assignment Pattern

- **Проблема:** new object or task has no accountable owner.
- **Когда применять:** routing rules can be deterministic and governed.
- **Решение:** assign by queue, department, territory, workload or explicit business rule.
- **Архитектура:** classify → select eligible owner → assign → acknowledge → fallback.
- **Контроль:** active-user check, capacity, deterministic rule version and no-owner monitor.
- **Ограничения:** automated assignment does not prove competence or availability without source data.

### 9.4 Approval Pattern

- **Проблема:** action requires authority before execution.
- **Когда применять:** discount, document, scope, access or exception decisions.
- **Решение:** approval bound to one immutable object/version.
- **Архитектура:** request → approver(s) → approve/reject/rework → authorized action.
- **Контроль:** limits, substitution, timeout, reason, separation of duties and audit.
- **Ограничения:** approval does not create legal signature or authoritative external status.

### 9.5 Escalation Pattern

- **Проблема:** exception remains unresolved after initial owner action.
- **Когда применять:** breach, high impact, repeated failure or no acknowledgement.
- **Решение:** tiered escalation with named owner, runbook and stop condition.
- **Архитектура:** trigger → validate → L1 → L2/manager → incident/change.
- **Контроль:** severity, one escalation per level, acknowledgement and closure review.
- **Ограничения:** avoid loops and alert storms; critical operations need alternate channels.

## 10. Analytics Patterns

### 10.1 Sales Dashboard Pattern

- **Проблема:** sales report mixes leads, Deals, payments and forecasts.
- **Когда применять:** pipeline performance and forecast management.
- **Решение:** separate grains and governed stage/cohort metrics.
- **Архитектура:** Lead/Deal events → normalized stages → conversion/cycle/amount dashboard.
- **Контроль:** owner, period, currency, loss reasons, duplicates and source reconciliation.
- **Ограничения:** Deal amount is not automatically revenue or payment.

### 10.2 Operational Dashboard Pattern

- **Проблема:** teams discover backlog and SLA issues too late.
- **Когда применять:** service, projects, production, onboarding and integrations.
- **Решение:** role-based current-state dashboard with age, owner and actionable exception.
- **Архитектура:** operational events → freshness/DQ → backlog/aging/SLA views.
- **Контроль:** update time, thresholds, drill path, owner and runbook.
- **Ограничения:** dashboard does not replace monitoring alerts or source-system truth.

### 10.3 Management Reporting Pattern

- **Проблема:** executives receive unrelated operational counts without business context.
- **Когда применять:** portfolio and trend decisions.
- **Решение:** reconciled KPI layer with targets, actuals, trends and explanations.
- **Архитектура:** conformed dimensions/facts → metric contracts → governed executive view.
- **Контроль:** grain, formula, owner, exclusions, baseline, freshness and commentary.
- **Ограничения:** aggregation can hide long-tail failures; drill-down access remains role-based.

### 10.4 KPI Pattern

- **Проблема:** KPI is chosen before process and data quality are defined.
- **Когда применять:** only after outcome, controllability and source are approved.
- **Решение:** KPI contract with formula, grain, window, owner and guardrails.
- **Архитектура:** authoritative facts → DQ/reconciliation → metric → target → review.
- **Контроль:** anti-gaming measures, context, cohorts, confidence and revision history.
- **Ограничения:** a metric is not a performance judgement without complexity and capacity context.

~~~text
Operational Sources -> Conformed IDs -> Data Quality / Reconciliation
 -> Metric Contracts -> Role-based Dashboards -> Decision / Action
~~~

## 11. Deployment Patterns

### 11.1 Discovery Pattern

- **Проблема:** configuration starts before facts and constraints are known.
- **Когда применять:** before every material change.
- **Решение:** structured AS IS inventory, stakeholders, scope, systems, data, rights, risks and open questions.
- **Архитектура:** evidence collection → process/data maps → gap analysis → decision backlog.
- **Контроль:** source, owner, confirmed/assumed status and approval.
- **Ограничения:** interview statements alone are not system evidence.

### 11.2 MVP Implementation Pattern

- **Проблема:** project attempts to implement the entire target ecosystem at once.
- **Когда применять:** core value can be tested with limited scope and reversible configuration.
- **Решение:** smallest end-to-end lifecycle with essential entities, rights, automation and metrics.
- **Архитектура:** one segment/process → test data → UAT → measured result → backlog.
- **Контроль:** explicit out-of-scope, no architectural dead-end, rollback and acceptance criteria.
- **Ограничения:** MVP is not permission to omit security, data integrity or error handling.

### 11.3 Pilot Rollout Pattern

- **Проблема:** direct organization-wide release has high operational risk.
- **Когда применять:** working process can be isolated by team, pipeline, portal or test scope.
- **Решение:** limited cohort with production-like controls and defined exit criteria.
- **Архитектура:** configuration → synthetic tests → pilot users/data → observation → expand/rollback.
- **Контроль:** baseline, support channel, daily review, defect threshold and change freeze.
- **Ограничения:** pilot data and users must be representative; do not hide critical integrations.

### 11.4 Enterprise Rollout Pattern

- **Проблема:** multi-unit deployment causes inconsistent configuration and uncontrolled change.
- **Когда применять:** several departments, regions or systems share a governed target model.
- **Решение:** waves, template baseline, local fit/gap, change control and central observability.
- **Архитектура:** foundation → wave 1 → evidence → wave N → stabilization.
- **Контроль:** version register, RACI, training, migration reconciliation, rollback and support capacity.
- **Ограничения:** local legal/process differences require ADR; one-size-fits-all is unsafe.

~~~text
Discovery -> MVP -> Pilot -> Wave Rollout
    |          |       |          |
    +-> ADR    +-> UAT +-> Review +-> Monitoring / Change Control
~~~

## 12. Monitoring Patterns

### 12.1 CRM Health Monitoring Pattern

- **Проблема:** stuck objects, missing owners and automation failures remain invisible.
- **Когда применять:** all operational CRM lifecycles.
- **Решение:** monitor business invariants, age, owners, next actions, duplicates and data quality.
- **Архитектура:** CRM state/events → rules → alert/dashboard → owner/runbook.
- **Контроль:** severity, freshness, dedup, acknowledgement and resolution.
- **Ограничения:** status counts alone do not show end-to-end service health.

### 12.2 Integration Monitoring Pattern

- **Проблема:** HTTP availability hides lost, duplicated or stale business events.
- **Когда применять:** every external exchange.
- **Решение:** technical and business monitoring with queue, retry, DLQ and reconciliation.
- **Архитектура:** request/event → correlation → result/latency → retry/quarantine → control totals.
- **Контроль:** success rate, age, lag, schema errors, duplicate rate and source/target counts.
- **Ограничения:** alerts must not expose secrets or full payloads.

### 12.3 SLA Monitoring Pattern

- **Проблема:** breaches are reported after closure or calculated inconsistently.
- **Когда применять:** service, onboarding, projects and approvals with governed deadlines.
- **Решение:** real-time threshold monitoring on versioned calendar and pause rules.
- **Архитектура:** start event → deadline engine → warning/breach → escalation → RCA.
- **Контроль:** source timestamp, priority, entitlement, pause reason, owner and outcome.
- **Ограничения:** different SLA types cannot share one formula without semantic proof.

## 13. Cloud vs On-Premise Patterns

| Паттерн | Cloud | On-Premise |
|---|---|---|
| CRM Sales Pipeline | Штатные CRM settings; tariff/rights verification | Same plus version/module governance |
| Smart Process Lifecycle | Availability and options depend on tariff/version | Depends on edition/modules/version |
| REST Integration | External runtime; no core/DB access | External or local module/API without core changes |
| Webhook Event | External protected endpoint and secret store | Same; local infrastructure may host endpoint |
| External System Master | API/middleware projection | API/module/middleware; DB write is not default |
| Data Synchronization | External queue/log/retry layer | External or controlled local services |
| Document Workflow | Cloud capabilities/app/DMS integration verified | Additional local DMS/module options by version |
| Role Based Access | Test actual CRM/entity rights | Test rights plus server/file/DB permissions |
| Sensitive Data Protection | Provider platform boundary; customer data/config responsibility | Customer also secures OS, PHP, DB, files and backups |
| BI and Dashboards | BI availability/datasets depend on tariff/version | Built-in/external BI and DB access by approved architecture |
| MVP/Pilot | Separate pipeline/test portal where available | Separate environment/module/configuration |
| Enterprise Rollout | Waves within portal(s), external integration capacity | Waves plus infrastructure capacity and deployment governance |
| Monitoring | External integration/security logs often required | Application, server, DB, cron and infrastructure monitoring |
| Backup/Rollback | Available product mechanisms plus config/data export strategy | Full application/data/infrastructure recovery design |

Cloud and On-Premise are architecture variants, not feature superiority labels. Every pattern is verified against the exact portal, edition, version and operational RACI.

## 14. Pattern Selection Matrix

| Задача клиента | Рекомендуемый паттерн |
|---|---|
| Построить управляемые продажи | CRM Sales Pipeline + Lead Qualification + Sales Dashboard |
| Разделить разные направления продаж | Multi Pipeline + Customer 360 |
| Организовать B2B account development | Account Management + Customer 360 |
| Создать отраслевую сущность | Master Data or Transaction Entity + Lifecycle Management |
| Организовать сервис | Ticket Management + SLA Management + Escalation |
| Управлять проектами | Project Delivery + Task Execution + Approval Workflow |
| Согласовывать документы | Document Workflow + Approval + Audit |
| Подключить внешнюю систему | External System Master + REST Integration + Integration Monitoring |
| Реагировать на события | Webhook Event + Idempotency controls + Notification/Assignment |
| Синхронизировать 1С | 1C Integration + Data Mapping + Reconciliation |
| Выполнить миграцию | Migration + Data Mapping + Deduplication + Data Quality |
| Защитить подразделения/клиентов | Role Based Access + Department Isolation + Audit |
| Построить аналитику | Data Quality + KPI + Management Reporting |
| Запустить MVP | Discovery + MVP Implementation + CRM Health Monitoring |
| Выполнить масштабный rollout | Pilot Rollout + Enterprise Rollout + Integration Monitoring |

~~~text
Client Task -> Context / Constraints -> Candidate Patterns
 -> Compatibility and Risk Review -> Selected Set -> ADR -> Testable Design
~~~

## 15. Pattern Anti-patterns

| № | Антипаттерн | Почему плохо | Как исправить |
|---:|---|---|---|
| 1 | CRM-first interpreted as CRM-only | Specialized facts drift | Assign master systems and projections |
| 2 | One entity for every process | Lifecycles and grains collide | Separate business objects |
| 3 | Deal used as project/order/subscription | Sales analytics and states become false | Link dedicated entity |
| 4 | One Deal for all repeat sales | Commercial cycles disappear | Deal per opportunity |
| 5 | Lead used for support or transactions | Qualification semantics break | Classify intake |
| 6 | Leads disabled without ADR | Unqualified demand may pollute pipeline | Decide from business process |
| 7 | Pipeline per manager | Duplicated process and bad analytics | Pipelines by lifecycle |
| 8 | Stage for every task | Adoption falls and state becomes ambiguous | Keep work in tasks |
| 9 | Too many stages | Users skip or fake states | Use minimum business states |
| 10 | Mandatory fields before data is known | Fake values and poor quality | Stage-based requiredness |
| 11 | Customer name as unique key | False matches and duplicates | Stable external/business IDs |
| 12 | Automatic destructive dedup | Wrong records merge | Conservative reviewable matching |
| 13 | Contact/Company copied per direction | Customer 360 breaks | Shared identity and relations |
| 14 | Smart Process without owner/lifecycle | Becomes another uncontrolled list | Define governance first |
| 15 | Smart Process replaces ERP/EAM/PIM by default | Scope, load and truth drift | Minimal governed projection |
| 16 | Free bidirectional synchronization | Conflicts and ping-pong | Field ownership and conflict policy |
| 17 | Last-write-wins as default | Older or unauthorized value may win | Version/source rules |
| 18 | Webhook treated as exactly-once | Duplicates create actions | Idempotency and dedup |
| 19 | Infinite retry | Load storm and repeated side effects | Bounded backoff and DLQ |
| 20 | No correlation ID | End-to-end incident cannot be traced | Trace every exchange |
| 21 | HTTP 200 equals business success | Partial or wrong data is hidden | Reconciliation and acceptance |
| 22 | Invented REST method/field | Integration fails | Verify official API reference |
| 23 | Batch before volume/method review | Partial failures and limits | Test sample and batch report |
| 24 | Secrets in README or webhook URL in Git | Credential compromise | Secret store and rotation |
| 25 | Full payload logging | Personal/commercial data leak | Mask and minimize |
| 26 | Platform admin equals data owner | Excessive access | Separate technical/business roles |
| 27 | Rights tested only positively | Hidden cross-access remains | Negative matrix tests |
| 28 | Organization-wide export by default | Mass leakage risk | Approval and minimum fields |
| 29 | Notification considered acknowledgement | No accountable action | Require task/ack state |
| 30 | Reminder storm | Noise hides real breaches | Threshold dedup and escalation |
| 31 | Robot approves its own result | No independent control | Human/authoritative gate |
| 32 | Automation loops stages | Storm and data corruption | Re-entry and loop guards |
| 33 | Document copied into comments/tasks | Version and access lost | Protected DMS reference |
| 34 | Checkbox treated as legal signature | False legal status | Authoritative EDO/DMS confirmation |
| 35 | Deal amount treated as revenue/payment | Financial metrics false | ERP/billing facts |
| 36 | Dashboard built before data quality | Attractive but false report | DQ and metric contracts first |
| 37 | Mixed analytical grains | Double counting | Separate facts and dimensions |
| 38 | Average used without distribution | Long-tail problems disappear | Percentiles/cohorts |
| 39 | KPI without owner/formula | Metric changes silently | Versioned KPI contract |
| 40 | Ranking without complexity/capacity | Harmful management decisions | Context and guardrails |
| 41 | Migration without inventory | Objects and relations are missed | Source/volume report |
| 42 | Production import without dry-run | Mass corruption | Synthetic/sample test |
| 43 | Real client data in UAT | Privacy/confidentiality breach | Synthetic datasets |
| 44 | No rollback | Failed release becomes outage | Tested rollback plan |
| 45 | Pilot excludes critical integrations | False confidence | Representative end-to-end pilot |
| 46 | One-shot enterprise rollout | Large blast radius | Waves and exit criteria |
| 47 | Monitoring only uptime | Lost business events remain hidden | Business invariants and reconciliation |
| 48 | Alert without owner/runbook | Incident remains unresolved | RACI and actionable alert |
| 49 | Alert contains secrets or personal data | Secondary data leak | Minimal masked context |
| 50 | Archive treated as backup | Restore may be impossible | Separate archive/backup policies |

## 16. Связь с Case Library

| Паттерн / семейство | Какие кейсы используют |
|---|---|
| CRM Sales Pipeline and Lead Qualification | 01 EdTech, 02 B2B Sales, 04 Real Estate, 06 E-commerce, 12 Retail, 13 Automotive, 23 Consulting, 25 IT and SaaS |
| Multi Pipeline and Account Management | 02 B2B Sales, 05 Manufacturing, 10 Construction, 15 Insurance, 16 Telecom, 18 Franchise |
| Customer 360 | 01 EdTech, 03 Service Business, 06 E-commerce, 08 Healthcare, 11 Hospitality, 12 Retail, 13 Automotive, 17 Travel, 25 IT and SaaS |
| Master Data | 04 Real Estate, 05 Manufacturing, 07 Logistics, 10 Construction, 13 Automotive, 19 Agriculture, 20 Energy and Utilities |
| Transaction Entity | 04 Real Estate, 06 E-commerce, 07 Logistics, 09 Finance, 11 Hospitality, 15 Insurance, 17 Travel |
| Lifecycle and Relationship | Все кейсы 01–25 |
| SLA, Ticket and Customer Support | 03 Service Business, 06 E-commerce, 08 Healthcare, 11 Hospitality, 13 Automotive, 15 Insurance, 16 Telecom, 20 Energy and Utilities, 21 Government, 25 IT and SaaS |
| Project Delivery and Task Execution | 03 Service Business, 05 Manufacturing, 10 Construction, 14 Legal Services, 19 Agriculture, 21 Government, 22 Media and Entertainment, 23 Consulting |
| Approval and Document Workflow | 02 B2B Sales, 09 Finance, 10 Construction, 14 Legal Services, 21 Government, 22 Media and Entertainment, 23 Consulting, 24 Recruitment |
| REST, Webhook and External Master | Все кейсы 01–25; особенно 05, 06, 07, 13, 16, 20, 25 |
| 1C Integration | 04 Real Estate, 05 Manufacturing, 06 E-commerce, 09 Finance, 10 Construction, 12 Retail, 19 Agriculture, 22 Media and Entertainment, 23 Consulting, 24 Recruitment |
| Mapping, Deduplication and Migration | Все кейсы 01–25 |
| Role Based Access and Sensitive Data Protection | Все кейсы 01–25; усиленно 08 Healthcare, 09 Finance, 14 Legal, 15 Insurance, 21 Government, 24 Recruitment |
| Automation patterns | Все кейсы 01–25 |
| Analytics and KPI | Все кейсы 01–25; recurring metrics in 16 Telecom and 25 IT and SaaS |
| MVP, Pilot and Enterprise Rollout | Все кейсы 01–25; network/wave focus in 12 Retail, 16 Telecom, 18 Franchise, 20 Energy, 21 Government |
| CRM, Integration and SLA Monitoring | Все кейсы 01–25 |
| Booking patterns through Transaction/Lifecycle | 04 Real Estate, 11 Hospitality, 13 Automotive, 17 Travel |
| Production patterns through Project/Master Data | 05 Manufacturing, 10 Construction, 19 Agriculture, 22 Media and Entertainment |
| Learning patterns through Lifecycle/Tasks | 01 EdTech, 18 Franchise, 23 Consulting |
| Recruitment relationship pattern | 24 Recruitment: Candidate → Recruitment → Vacancy |
| SaaS account/subscription pattern | 25 IT and SaaS: Company/Contact → SaaS Account → Subscription |

Покрытие означает наличие применимого архитектурного решения, а не идентичную реализацию. При использовании нескольких паттернов проверяются их совместимость, common identity, master ownership, rights inheritance, automation loops and analytical grains.

### Официальные источники

Для проверки возможностей используются только записи из knowledge/SOURCES.md:

- [Поддержка24](https://helpdesk.bitrix24.ru/) — пользовательские возможности, интерфейс, тарифные и облачные оговорки;
- [Bitrix24 REST API](https://apidocs.bitrix24.ru/) — интеграционные принципы, авторизация and limits;
- [REST API reference](https://apidocs.bitrix24.ru/api-reference/) — точные методы, поля, scopes and events;
- [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/) — официальный инструментарий;
- [Smart Processes](https://helpdesk.bitrix24.ru/open/18913884/) — создание и конфигурация;
- [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) — штатная automation;
- [Бизнес-процессы](https://helpdesk.bitrix24.ru/open/23951294/) — выбор process type;
- [BI Constructor datasets](https://helpdesk.bitrix24.ru/open/24355590/) — доступные datasets and limitations;
- [On-Premise documentation](https://dev.1c-bitrix.ru/docs/portal.php) — коробочная разработка без изменения ядра.

Дата актуализации каталога: **2026-08-06**.
