# Franchise Network and Multi-Unit CRM Architecture Case

Архитектурный кейс для синтетической франчайзинговой сети, объединяющей управляющую компанию, независимых франчайзи и точки продаж. Документ задаёт референсную CRM-модель партнёрского цикла, запуска точек, обучения, качества и аналитики; это не готовая конфигурация или юридическая модель франшизы.

> **Граница безопасности:** реальные бренды, франчайзи, сотрудники, клиенты, адреса точек, договоры, кассовые данные, телефоны, email, CRM ID, webhook URL и токены не используются. Все данные и идентификаторы синтетические.

> **Архитектурная граница:** Bitrix24 отвечает за partner CRM, развитие сети, задачи, обращения и координацию стандартов. POS/ERP/1С остаются master продаж, номенклатуры и финансов точек; LMS — прохождения обучения; DMS/ЭДО — версий стандартов и подписанных договоров. Право управляющей компании видеть данные франчайзи и клиентов не предполагается автоматически и утверждается отдельно.

Обозначения:

- **факт Bitrix24** — подтверждён официальным источником из `knowledge/SOURCES.md`;
- **решение архитектора** — целевая рекомендация, принимаемая на Discovery;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа/редакции, версии, прав, региона, приложения или внешней системы.

## 1. Business Context

### Модель франчайзинга

| Участник | Роль | Целевой результат CRM |
|---|---|---|
| Управляющая компания | Бренд, модель, стандарты, развитие и network support | Governed partner portfolio and network controls |
| Кандидат | Потенциальный партнёр | Прозрачная qualification pipeline |
| Франчайзи | Самостоятельный партнёр по утверждённой модели | Onboarding, support, obligations, development |
| Точка продаж | Операционная единица сети | Opening lifecycle, status and aggregate KPI projection |
| Клиент точки | Конечный покупатель/заказчик | Только допустимая CRM-модель и согласованный data scope |
| Поставщик/система | POS, ERP, 1С, LMS, DMS | Authoritative operational facts |

### Цели CRM

1. Управлять циклом кандидата от лида до работающей точки.
2. Поддерживать единый стандарт без смешения независимых владельцев данных.
3. Связать франчайзи, договор, территории, точки, обучение, аудиты и действия stable IDs.
4. Контролировать открытие, поддержку, корректирующие мероприятия и развитие.
5. Разделить права управляющей компании, франчайзи и сотрудников точки.
6. Получать сверяемую network analytics из согласованных агрегатов.

```text
Partner Lead -> Qualification -> Franchise Deal -> Contract -> Franchisee
                                                         |
                                                         v
                                                   Location Opening
                                                         |
                                  +----------------------+----------------+
                                  v                      v                v
                              Training               Audit/Quality     Support
                                  |                      |                |
                                  +-------------------- Location ----------+
                                                         |
                                                         v
                                              Aggregate POS/ERP Analytics
```

## 2. AS IS Problems

| Проблема | Проявление | Причина | Целевой контроль |
|---|---|---|---|
| Нет единого стандарта | Точки работают по-разному | Нет versioned standard register | DMS version + adoption control |
| Разрозненные данные | Partner/location duplicated | Нет identity/topology model | Stable IDs and relations |
| Нет контроля партнёров | Обязательства в переписке | Нет lifecycle/owner | Partner plan and review checkpoints |
| Проблемы обучения | Нельзя подтвердить прохождение | Нет LMS integration | Course/version/result projection |
| Нет общей аналитики | POS/ERP показатели несопоставимы | Нет metric contracts | Conformed aggregates/reconciliation |
| Смешение данных франчайзи | Общий портал без scope | Не определён data ownership | Segmented rights and tenancy ADR |
| Просрочки открытия | Нет dependencies/gates | Этапы ведутся вручную | Opening checklist and escalation |
| Аудиты в файлах | Нет status/remediation link | Нет audit entity | Structured finding/action lifecycle |
| Маркетинг не согласован | Локальные материалы без версии | Нет approval process | Campaign/material version workflow |
| Повторные интеграции | Retry создаёт дубли | Нет idempotency | Source event ID/upsert |

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Лид | Неквалифицированный кандидат/обращение, если lead model утверждена | Bitrix24 |
| CRM | Контакт | Представитель партнёра/клиент в допустимом scope | Bitrix24/MDM |
| CRM | Компания | Кандидат, франчайзи или контрагент | Bitrix24/ERP/MDM |
| CRM | Сделка | Продажа франшизы или отдельная коммерческая opportunity | Bitrix24 |
| CRM | Обращение | Support request партнёра/точки | Bitrix24 |
| Smart Process | Франчайзи | Partner lifecycle projection | Bitrix24/ERP by ADR |
| Smart Process | Точка продаж | Location/opening/status projection | ERP/POS/MDM by ADR |
| Smart Process | Договор франшизы | Согласование и ссылка на authoritative version | DMS/ЭДО |
| Smart Process | Обучение | Assignment/progress projection | LMS |
| Smart Process | Аудит | Проверка, findings and remediation | Bitrix24/audit system |
| Smart Process | Маркетинговая активность | Central/local campaign approval | Bitrix24/marketing system |

**Факт Bitrix24:** CRM, задачи, Smart Processes, роботы, триггеры и бизнес-процессы описаны официальными материалами. Конкретные права, связи, поля и automation проверяются на целевом портале.

**Решение архитектора:** каждая интеграционная проекция содержит `source_system`, `external_id`, `source_updated_at`, `sync_status` и correlation ID. Detail-level customer/POS data централизуется только при подтверждённом основании и ownership model; по умолчанию используются необходимые агрегаты.

```text
Website/Referral -> Lead -> Contact/Company -> Deal -> Franchise Contract
                                 |                         |
                                 v                         v
                            Franchisee 1 -------- N Location
                                 |                    |   |
                                 |                    |   +-> Audit -> Corrective Action
                                 |                    +-----> Training Assignment -> LMS
                                 |                    +-----> Support Case
                                 |                    +-----> Local Campaign
                                 |
                          Partner Development

POS/ERP/1C -> governed aggregates -> BI <- Bitrix24 network processes
DMS/EDO -> contract/standard versions; LMS -> learning facts
```

### Правила целостности

- одна company identity на юридическое лицо; location не создаётся как duplicate Company;
- master назначается для договора, стандарта, точки, сотрудника, обучения, продажи, финансов и KPI;
- `franchisee_id`, `location_id`, `contract_id`, `standard_version_id` и source IDs обязательны для интеграции;
- повтор события не создаёт duplicate location, training assignment, audit, task or campaign;
- права и аналитика фильтруются по franchisee/location scope до выдачи данных;
- bulk operations используют dry-run, sample, журнал, reconciliation and rollback.

## 4. Franchise Lifecycle

| Этап | Цель | Минимальные данные | Gate/контроль |
|---|---|---|---|
| Лид партнёра | Зарегистрировать интерес | Channel, region interest, contact minimum | Dedup/owner/SLA |
| Квалификация | Подтвердить fit и capacity | Candidate type, territory, next step | Approved criteria |
| Переговоры | Согласовать модель | Open items, stakeholders, timeline | Decision owner |
| Договор | Получить authoritative agreement | Contract ID/version/status/effective dates | DMS/ЭДО confirmation |
| Открытие точки | Выполнить readiness plan | Location ID, territory, milestones, owners | Dependency/gate checklist |
| Запуск | Подтвердить operational readiness | Training/audit/integration statuses | Go/no-go |
| Поддержка | Обрабатывать вопросы и отклонения | Case, location, category, SLA | Routing/escalation |
| Развитие сети | Управлять улучшениями/новыми точками | KPI trend, plan, approved opportunity | New lifecycle, not overwrite |

```text
Лид партнёра -> Квалификация -> Переговоры -> Договор
 -> Открытие точки -> Запуск -> Поддержка -> Развитие сети

Branches: rejected / postponed / contract stopped / opening blocked /
pilot failed / corrective plan / controlled closure.
```

## 5. Franchise Data Model

| Сущность | Минимальные поля | Контроль |
|---|---|---|
| Франчайзи | External company/franchisee ID, territory, lifecycle status, owner, aggregate KPI refs | Company is not duplicated by location |
| Точка | Location ID, franchisee, synthetic territory/address ref, opening/status dates, responsible | Address/customer detail restricted |
| Договор | Contract ID/version, parties, term, status, protected DMS ref | DMS/ЭДО master |
| Сотрудник/роль | User/external employee ID, franchisee/location/role, effective dates | HR/LMS source; no shared login |
| Обучение | Assignment ID, course/version, person/role, status/date | LMS master |
| Аудит | Audit ID/type/version, location, plan/fact, result, reviewer | Evidence access/audit trail |
| Finding | Finding ID, severity/category, standard version, owner, deadline/status | Remediation relation |
| KPI projection | Location/period/metric/version/value | Aggregate/source/control total |

```text
Franchisee 1 ---- N Location 1 ---- N Employee Role
    |                 |                    |
    |                 +---- N Audit ---- N Finding ---- N Corrective Action
    |                 +---- N Training Assignment ---- 1 Course Version
    |                 +---- N KPI Aggregate <- POS/ERP
    +---- N Contract Version -> DMS/EDO
    +---- N Support/Development Plan
```

### Кардинальности

| Связь | Кардинальность | Правило |
|---|---:|---|
| Company → Franchisee | 1:0..N by agreement model | Legal identity remains stable |
| Franchisee → Location | 1:N | Location has unique stable ID |
| Location → Audit | 1:N | Each audit immutable and versioned |
| Audit → Finding | 1:N | Findings do not overwrite audit |
| Employee ↔ Location | M:N over time | Effective role dates required |
| Location → KPI | 1:N by metric/period | Metric version/grain explicit |

## 6. Franchise Sales Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новый партнёр | Зафиксировать кандидата | Channel, contact/company, territory, owner | Routing/SLA/dedup |
| Квалификация | Проверить базовый fit | Criteria result, resources band, next step | Review task |
| Презентация модели | Подтвердить понимание | Meeting result, questions, follow-up | Task/reminder |
| Проверка кандидата | Выполнить утверждённые проверки | Checklist/version/result/owner | Restricted approval route |
| Коммерческое предложение | Зафиксировать условия | Offer ID/version/validity | Approval/pre-send check |
| Договор | Подтвердить agreement | Contract ID/version/status | DMS/ЭДО check |
| Подготовка открытия | Создать location/readiness plan | Location ID, milestones, owners | Tasks/dependency control |
| Запуск точки | Выполнить go-live gates | Training, audit, systems readiness | Go/no-go checklist |
| Сопровождение | Передать partner owner | Support route, review cadence, open items | Handoff/review tasks |

```text
Новый партнёр -> Квалификация -> Презентация модели -> Проверка кандидата
 -> Коммерческое предложение -> Договор -> Подготовка открытия
 -> Запуск точки -> Сопровождение
```

Стадии описывают состояние, действия — задачи. Повторный вход не создаёт duplicate location/task/message. Успех подтверждается signed/effective contract and approved opening gates, а не ручной отметкой.

## 7. Partner Management

| Область | Модель | Контроль |
|---|---|---|
| Профиль | Company/franchisee IDs, territory, status, owner | Stable identity |
| Коммуникации | Authorized channels and partner cases | Purpose/retention/access |
| Поддержка | Structured category/SLA/owner | Separate from client support |
| Показатели | Approved aggregate KPI set | Source/grain/version/reconciliation |
| Развитие | Improvement/new-location opportunity | New plan/Deal, not overwrite history |
| Governance | Review cadence, obligations and decisions | Versioned action log |

Управляющая компания не должна свободно редактировать операционные факты франчайзи. Для каждого data domain фиксируются owner, controller/processor roles where applicable, allowed purpose, retention and dispute process.

## 8. Location Management

| Объект/этап | Данные | Gate | Master |
|---|---|---|---|
| Территория | Territory ID/version/status | Conflict/approval rule | Network planning/ERP |
| Кандидатная точка | Location ID, synthetic area ref, owner | Feasibility checklist | Bitrix24/planning system |
| Подготовка | Milestones, dependencies, plan/fact | Readiness reviews | Bitrix24 |
| Запуск | Training/audit/system statuses | Go/no-go | Approved committee/process |
| Работа | Operational status and aggregate KPIs | Monitoring/reviews | ERP/POS projection |
| Закрытие/передача | Effective date, asset/data obligations | Closure checklist | ERP/legal systems |

```text
Territory -> Candidate Location -> Readiness Plan -> Pilot/Open
                                       |                |
                          Training ----+---- Audit       v
                          Systems -----+---- Contract  Operating Location
```

## 9. Training Management

| Компонент | Назначение | Master/контроль |
|---|---|---|
| Программа | Role-based curriculum | LMS/catalog owner |
| Курс | Course/version/effective dates | LMS master |
| Назначение | Employee/location/course relation | Unique assignment ID |
| Прохождение | Status/result/completion projection | LMS source |
| Повторное обучение | New assignment for new standard/version | No overwrite |
| Материалы | Protected/versioned reference | LMS/DMS |

CRM координирует назначения, просрочки и readiness gates. Факт прохождения, результат и версия курса подтверждаются LMS; одна отметка в CRM не является доказательством компетентности.

## 10. Quality Control

| Элемент | Содержание | Контроль |
|---|---|---|
| Стандарт | Standard ID/version/effective dates | DMS master |
| План аудита | Location, scope/version, date, reviewer | Conflict/independence check |
| Проверка | Structured checklist and evidence refs | Need-to-know access |
| Finding | Category/severity/source criterion | Governed dictionary |
| Corrective action | Owner, result, deadline, status | SLA/escalation/evidence |
| Closure | Reviewer confirmation | No auto-close by task completion |

```text
Standard Version -> Audit Plan -> Audit -> Finding -> Corrective Action
                                           |                |
                                           +-> Recheck <----+
                                                  |
                                               Closure
```

Quality score не рассчитывается из свободных комментариев. Методика, веса, evidence, exceptions and appeal process утверждаются владельцем качества.

## 11. Rights Model

| Роль | Партнёры | Клиенты | Документы | Обучение | Аналитика |
|---|---|---|---|---|---|
| Управляющая компания | Governed network scope | Only approved/minimized scope | Standards/contracts by role | Aggregate/status | Network aggregates |
| Менеджер франшизы | Assigned portfolio | Minimum support context | Assigned partner refs | Portfolio progress | Portfolio |
| Франчайзи | Own company/locations | Own permitted data only | Own contracts/standards | Own employees | Own locations |
| Сотрудник точки | Own location minimum | Assigned operational clients | Working materials only | Own assignments | Personal/operational minimum |
| Маркетолог | Approved segments/campaigns | Minimized/consented audience | Approved materials | None by default | Campaign aggregates |
| Директор | Governed network portfolio | Aggregate/approved | Approved | Aggregate | Executive |
| Администратор | Technical configuration | No automatic business entitlement | No automatic content | Config only | Dataset/config as needed |

Принципы: deny-by-default, franchisee/location isolation, least privilege, separation of duties, temporary access expiry, periodic recertification, export controls and negative tests. Общая роль «все франчайзи» запрещена без field/record scope design.

## 12. Automation Architecture

| Автоматизация | Trigger | Действие | Защита |
|---|---|---|---|
| Candidate SLA | First valid lead | Assign/task/timer | Idempotency marker |
| Opening control | Milestone approaching/breached | Task/escalation | Current plan/version |
| Training assignment | Approved role/location event | Create LMS assignment request/task | Unique assignment key |
| Audit scheduling | Approved cadence/event | Audit plan/task | Standard version/conflict check |
| Corrective action | Finding approved | Owner/deadline/escalation | No automatic finding closure |
| Standard update | New effective version | Review/adoption tasks | Target scope/version |
| Integration failure | Error | Queue/alert/bounded retry | Correlation/DLQ |

```text
Event -> Validate source/version/tenant/rights -> Check idempotency -> Rule
 -> Bounded action -> Audit result -> Exception queue/escalation
```

Автоматизация не подписывает договор, не подтверждает обучение, не закрывает audit finding и не утверждает открытие точки без authoritative/human gate.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| Сайт | Candidate leads/location enquiries | CRM form/official app/webhook/REST | Validation, consent, dedup |
| ERP | Franchisee/location/product/order projections | Documented API/middleware | Stable IDs/field ownership/reconcile |
| Кассы/POS | Location sales aggregates | API/ETL/middleware | Grain/period/control totals |
| 1С | Contracts/invoices/payments/royalty refs | Supported integration/middleware | Amount/currency/reconciliation |
| BI | Governed datasets/aggregates | BI Constructor or approved ETL | Tenant filters/freshness/DQ |
| Телефония | Call events/CRM linkage | Bitrix24 telephony/SIP/REST app | Rights/missed-call/correlation |
| LMS | Courses/assignments/completion facts | API/middleware/approved connector | Version/idempotency/reconciliation |

```text
Source Event -> AuthN/AuthZ -> Tenant + schema validation
 -> Correlation/idempotency -> Master ownership
 -> Bitrix24 write or quarantine -> Reconciliation/monitoring
```

Секреты хранятся вне Git; token scopes ограничены franchisee/function where possible; retry bounded; unknown mappings quarantined. REST methods/scopes/limits подтверждаются официальной документацией.

## 14. Marketing Architecture

| Контур | Назначение | Workflow | Контроль |
|---|---|---|---|
| Централизованный маркетинг | Brand campaigns/materials | Version/approval/distribution | Brand owner/effective dates |
| Локальная кампания | Location-specific activity | Request/review/approve/run/close | Franchisee/location scope |
| Материал | Template/asset version | DMS link/adoption status | No unapproved editing |
| Audience | Approved segment | Eligibility/consent/suppression | No cross-franchisee data leak |
| Аналитика | Campaign/location aggregates | Attribution/reconciliation | Defined window/grain |

Центральная и локальная кампании имеют разные owners, budgets, audiences and metric contracts. Customer-level exchange между франчайзи не допускается без утверждённой цели и модели прав.

## 15. Analytics

| Отчёт | Grain | Показатели | Контроль качества |
|---|---|---|---|
| Эффективность франчайзи | Franchisee + period | Approved sales/quality/training/support metrics | Comparable metric version |
| Продажи точек | Location + period | Reconciled aggregate sales/orders | POS/ERP totals |
| Открытие точек | Location/milestone | Lead time, gates, delays | Plan/fact/source |
| Качество | Audit/finding/location | Score, findings, closure time | Standard/version/methodology |
| Обучение | Assignment/person-role/course | Completion/overdue/readiness | LMS reconciliation |
| Прибыльность сети | Franchisee/location/period | Reconciled revenue/cost/margin | Finance-owned formula |

```text
CRM + POS/ERP/1C + LMS + Audits -> Conformed Franchisee/Location IDs
 -> Tenant-safe DQ/Reconciliation -> Metric Layer -> Role-based Dashboards
```

Метрика имеет owner, formula, grain, period, currency, exclusions, source, freshness SLA, metric version and control total. Ranking не используется без сопоставимости и контекста.

## 16. Security Architecture

| Контроль | Реализация | Проверка |
|---|---|---|
| Franchisee isolation | Record/field scope by franchisee/location | Cross-tenant negative tests |
| HQ access | Explicit purpose/role/minimized fields | HQ unauthorized-detail test |
| Customer data | Local ownership/default minimization | Data inventory/export test |
| Документы | Protected DMS refs | Expired/revoked access |
| Аудит действий | CRM history + masked integration logs | Correlation trace |
| Export | Approval/minimum columns/protected destination | Export permission test |
| Secrets | Environment/secret store/rotation | Scan/revoke drill |
| Retention | Approved archive/delete/hold policy | Periodic review |

**Cloud:** нет server/PHP/DB access; доступны штатные rights/history and external logs. **On-Premise:** дополнительно защищаются server, PHP, DB, files, cron, backup and logs; core is not modified.

## 17. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Candidate from site | Valid synthetic payload | One lead/Deal, owner/SLA |
| 2 | Duplicate event | Same source ID | No duplicate |
| 3 | Candidate qualification | Required synthetic criteria | Correct gate/task once |
| 4 | Contract not confirmed | Missing DMS status | Opening blocked/flagged |
| 5 | Location creation | Approved contract/location ID | One location and readiness plan |
| 6 | Opening dependency | Training incomplete | Go-live gate blocks |
| 7 | LMS completion | Authoritative event | Assignment updated/reconciled |
| 8 | Audit finding | Failed synthetic criterion | Finding/action/owner created |
| 9 | Finding recheck | Evidence accepted | Controlled closure/audit trail |
| 10 | Standard update | New version | Targeted adoption tasks; old history retained |
| 11 | Franchisee isolation | User A opens franchisee B record | Access denied |
| 12 | HQ minimum access | HQ role without detail permission | Only approved fields/aggregates |
| 13 | POS aggregate | Synthetic location totals | Correct location/period/reconcile |
| 14 | Integration timeout | Write accepted then timeout | Retry idempotent |
| 15 | Local campaign | Approved material/audience | Only own location campaign |
| 16 | Export | Unauthorized franchisee user | Denied/minimized |
| 17 | Support SLA | Breach threshold | Escalation once |
| 18 | BI reconciliation | Synthetic totals | Threshold pass or incident |
| 19 | Pilot rollback | Failed gate/release | Verified prior state restored |
| 20 | Scale onboarding | Second synthetic franchisee | Config template without data leakage |

Тесты используют только synthetic partners, locations, employees, clients and sales aggregates. Проверяются positive, negative, boundary, concurrency, retry, tenant rights, audit and recovery.

## 18. Deployment

| Этап | Результат | Gate |
|---|---|---|
| Discovery | AS IS, legal/data model, systems, tariff, volumes | Business/security approval |
| Архитектура | TO BE, tenant/master matrix, rights, ADR, NFR | Architecture review |
| Настройка | CRM/Smart Processes/automation in test scope | Configuration review |
| Пилотная точка | One synthetic/test partner-location flow | Pilot acceptance/rollback readiness |
| Обучение | Role-based exercises and support materials | UAT/sign-off |
| Масштабирование | Versioned rollout waves | Wave go/no-go/reconciliation |
| Stabilization | Incidents/baseline/adoption | Exit criteria |

```text
Inventory -> Architecture/Rights Tests -> Test Configuration -> Synthetic UAT
 -> Pilot Franchisee/Location -> Smoke/Reconciliation -> Go/No-Go
 -> Rollout Waves -> Hypercare -> Optimization or Rollback
```

Масштабирование выполняется волнами. Нельзя копировать portal/configuration вместе с partner data; перед каждой волной проверяются tenant filters, mappings, roles, secrets, training and rollback.

## 19. Monitoring

| Сигнал | Правило | Действие | Владелец |
|---|---|---|---|
| Просрочка открытия | Milestone/gate breached | Escalate/replan | Opening owner |
| Проблема партнёра | Support SLA/risk status | Assign/review | Franchise manager |
| Невыполнение стандарта | Finding overdue/repeat | Escalate/recheck | Quality owner |
| Training overdue | LMS assignment threshold | Notify/escalate | Training owner |
| Integration error | Error/retry/DLQ | Incident/replay after fix | Integration support |
| Cross-tenant anomaly | Unexpected access/filter failure | Security incident | Security owner |
| POS/ERP mismatch | Aggregate difference | Reconcile/data incident | Data owner |
| BI stale/mismatch | Freshness/control total | Mark stale/investigate | BI owner |

Alert содержит system, franchisee/location reference, correlation ID, time, severity, owner and runbook; без секретов и клиентских деталей.

## 20. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Дубли франчайзи | Multiple sources | Fragmented portfolio | Stable IDs/dedup |
| 2 | Location as Company duplicate | Flat model | Wrong topology | Separate Location entity |
| 3 | Cross-franchisee data leak | Shared roles | Security incident | Tenant scopes/negative tests |
| 4 | HQ sees excessive customer data | Undefined purpose | Privacy/commercial risk | Minimized explicit access |
| 5 | Franchisee sees network data | Broad analytics | Commercial disclosure | Own-scope dashboards |
| 6 | Shared employee account | Cost saving | No accountability | Named identities |
| 7 | Temporary access persists | No expiry | Residual risk | Recertification |
| 8 | Contract status manual | No DMS sync | False relationship state | Authoritative confirmation |
| 9 | Standard version stale | Local file copy | Noncompliance | DMS version/adoption |
| 10 | Candidate accepted by subjective notes | No criteria | Inconsistent decisions | Governed checklist |
| 11 | Opening without dependencies | Deadline pressure | Failed launch | Go/no-go gates |
| 12 | Duplicate opening tasks | Re-entry | Fake progress | Idempotency |
| 13 | Territory conflict | No master map | Partner dispute | Versioned territory approval |
| 14 | POS data assigned wrong location | Weak mapping | False KPI | Stable location ID |
| 15 | Detail POS data overcentralized | Dashboard demand | Data exposure/load | Aggregation/minimization |
| 16 | 1С/CRM financial conflict | Two masters | Wrong amounts | Finance ownership |
| 17 | LMS status manual | No integration | False readiness | LMS reconciliation |
| 18 | Course version overwritten | Flat catalog | Wrong training evidence | Immutable version |
| 19 | Audit uses obsolete standard | No version pinning | Invalid result | Standard version ID |
| 20 | Finding auto-closed | Task completion trigger | Unresolved issue | Reviewer gate |
| 21 | Auditor conflicts | No independence rule | Biased quality result | Assignment/approval control |
| 22 | Quality score from comments | Unstructured data | False ranking | Governed checklist/method |
| 23 | Corrective action no owner | Collective responsibility | Repeat issue | Accountable owner/deadline |
| 24 | Local campaign bypasses approval | Speed | Brand/reputation risk | Material workflow |
| 25 | Marketing crosses tenants | Shared audience | Data leak | Tenant-safe segmentation |
| 26 | Consent ignored | Central campaign | Compliance risk | Eligibility/suppression |
| 27 | Partner portal access broad | Convenience | Network exposure | Least privilege |
| 28 | Admin as super-reader | Role confusion | Uncontrolled access | Separate privileges |
| 29 | Export unrestricted | Defaults | Mass leak | Approval/minimum fields |
| 30 | Secret in Git | Real example | Compromise | Secret store/rotation |
| 31 | Broad integration token | One credential | Large blast radius | Per-scope least privilege |
| 32 | Raw logs contain customers | Debugging | Data leak | Masking |
| 33 | Retry without idempotency | Timeout ambiguity | Duplicates | Source event ID |
| 34 | Infinite retry | No DLQ | Load/outage | Bounded backoff |
| 35 | Event order ignored | Async update | Stale state | Version/order guard |
| 36 | Schema drift | Source change | Corrupt fields | Contract validation |
| 37 | No correlation ID | Fragmented logs | Slow RCA | End-to-end trace |
| 38 | Free bidirectional edits | No master | Ping-pong | Field ownership |
| 39 | Metric versions differ | Local formulas | Noncomparable network | Metric contract/version |
| 40 | Profitability from Deal amount | Missing costs | False result | Finance reconciliation |
| 41 | Ranking ignores maturity | New/old locations mixed | Harmful decisions | Cohort/context |
| 42 | Real partner data in test | Prod copy | Data breach | Synthetic data |
| 43 | Migration without dry-run | Urgency | Mass corruption | Sample/report/rollback |
| 44 | No reconciliation | HTTP success trusted | Silent loss | Counts/control totals |
| 45 | Tariff not checked | Assumption | Feature unavailable | Capability assessment |
| 46 | Cloud treated as server | DB/PHP expected | Impossible design | Cloud boundary |
| 47 | Core modified On-Premise | Quick fix | Upgrade/security failure | Extensions only |
| 48 | Pilot skipped | Big-bang pressure | Network-wide failure | Pilot/waves |
| 49 | Rollback absent | Success assumed | Long outage | Tested rollback |
| 50 | Alert has no owner | Notification only | Unresolved incident | RACI/runbook |
| 51 | No baseline | AS IS unmeasured | Effect unprovable | Baseline/KPI |
| 52 | No change register | Oral changes | Configuration drift | Versioned register |
| 53 | Franchisee closure not designed | Growth-only focus | Orphan data/access | Offboarding checklist |
| 54 | Scale load not tested | Pilot volumes only | SLA collapse | Capacity/load tests |
| 55 | Archive confused with backup | No restore design | Failed recovery | Separate policies |

## 21. ADR

| ADR | Решение | Альтернатива | Обоснование | Последствие |
|---|---|---|---|---|
| ADR-FRN-001 | Company represents legal partner; Franchisee stores relationship lifecycle | Separate Company per contract/location | Stable legal identity | Company–Franchisee relation required |
| ADR-FRN-002 | Location is separate Smart Process/projection | Location as Company | Preserves network topology | Stable location master needed |
| ADR-FRN-003 | Rights isolate franchisee/location; HQ gets explicit minimized access | One network-wide role | Independent data domains | More complex rights tests |
| ADR-FRN-004 | LMS is training master | Manual CRM checkbox | Versioned evidence/learning lifecycle | LMS integration required |
| ADR-FRN-005 | Audit/finding/action are separate linked lifecycles | Audit notes in Deal | Traceable remediation | More entities/dictionaries |
| ADR-FRN-006 | DMS/ЭДО masters contract/standard versions | Authoritative files in CRM | Version/signature/retention | Protected refs/integration |
| ADR-FRN-007 | Middleware/contracts govern integrations | Free point-to-point webhooks | Idempotency/tenant validation | Platform owner required |
| ADR-FRN-008 | BI uses tenant-safe reconciled aggregates | Central raw POS/client dataset | Minimization/comparability | Metric/DQ ownership |

ADR получает owner, status `proposed/accepted/superseded`, date, context, alternatives, consequences, portal/version and evidence.

## 22. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Нет Discovery | Immediate setup | Wrong scope | AS IS/stakeholders |
| 2 | Cloud/On-Premise unknown | Template design | Unsupported solution | Capability check |
| 3 | Tariff not verified | Assumption | Blocked release | Verify portal |
| 4 | Legal/data model ignored | CRM-first design | Unlawful/unsafe sharing | Ownership/purpose review |
| 5 | All candidates are Companies only | No lifecycle | Weak pipeline | Lead/Deal/Company roles |
| 6 | Leads disabled without ADR | Fashion | Unqualified demand mixed | Lead decision |
| 7 | One Company per location | Shortcut | Partner duplicates | Location entity |
| 8 | One Company per contract | Contract-centric model | Fragmented identity | Contract relation |
| 9 | Name is integration key | No external ID | Wrong match | Stable IDs |
| 10 | Phone is sole dedup key | Convenience | False merge | Conservative matching |
| 11 | Franchisee and customer mixed | One type field | Wrong rights/process | Explicit entity roles |
| 12 | One Deal for whole partnership | Entity saving | No sales history | Deal per opportunity |
| 13 | Opening tracked only in Deal stages | No operational entity | Lost milestones | Location/readiness lifecycle |
| 14 | Stages describe tasks | Process mistake | Ambiguous state | State-based stages |
| 15 | Too many stages | Every action is stage | Low adoption | Tasks/checklists |
| 16 | Fields mandatory too early | Maximum control | Fake data | Stage requiredness |
| 17 | Loss reason in comment | No dictionary | No analytics | Governed codes |
| 18 | No next step | Stage-only control | Stalled candidate | Activity policy |
| 19 | Deal success before contract | KPI pressure | False partner count | DMS/ЭДО gate |
| 20 | Robot duplicates opening tasks | Re-entry ignored | Fake progress | Idempotency marker |
| 21 | Robots loop stages | Mutual transitions | Automation storm | Loop guard |
| 22 | Contract overwritten | No versioning | Wrong terms | Immutable version refs |
| 23 | Territory stored as free text | No master | Conflicts/poor analytics | Governed territory ID |
| 24 | Location address is only key | Mutable value | Duplicate/mismatch | Stable location ID |
| 25 | No opening dependency map | Checklist only | Premature launch | Gates/dependencies |
| 26 | Pilot skipped | Big-bang ambition | Network disruption | Pilot location |
| 27 | Pilot uses real customer data | Convenience | Data breach | Synthetic/pilot-safe data |
| 28 | Training checkbox manual | No LMS link | False readiness | LMS source |
| 29 | Course version absent | One title | Obsolete training | Version/effective dates |
| 30 | Employee uses shared account | Cost saving | No audit | Named identity |
| 31 | Employee offboarding missing | HR gap | Residual access | Effective dates/revoke |
| 32 | Standard file sent by email | Easy distribution | Stale copies | DMS version/adoption |
| 33 | Audit in spreadsheet attachment | Legacy habit | No lifecycle | Audit entity |
| 34 | Finding stored in comment | Quick note | No remediation | Finding/action relation |
| 35 | Finding closes with task | Automation shortcut | Issue unresolved | Reviewer/recheck gate |
| 36 | Audit severity subjective | No dictionary | Inconsistent score | Governed criteria |
| 37 | Auditor conflict ignored | Resource shortage | Biased audit | Independence check |
| 38 | Partner support mixed with client support | One queue | Wrong SLA/access | Case classification |
| 39 | HQ sees all client details | Centralization | Data exposure | Minimized aggregate model |
| 40 | Franchisee sees other partners | Common role | Cross-tenant leak | Tenant scopes |
| 41 | Employee sees entire franchisee | Broad inheritance | Excess access | Location/role scope |
| 42 | Marketing exports raw clients | Campaign convenience | Mass leak | Approved segments/minimum |
| 43 | Admin equals business reader | Role confusion | Uncontrolled access | Separate privileges |
| 44 | Temporary access never expires | Manual grant | Residual risk | Expiry/recertification |
| 45 | Rights only positive-tested | Happy path | Hidden cross-access | Negative tests |
| 46 | Export unrestricted | Defaults | Mass leak | Approval/minimum fields |
| 47 | Secret in README | Real example | Compromise | Placeholder/secret store |
| 48 | Webhook committed | Convenience | Unauthorized calls | Environment/rotation |
| 49 | Token covers all franchisees | One credential | Network-wide blast radius | Per-scope least privilege |
| 50 | Raw payload logged | Debugging | Partner/client leak | Masking |
| 51 | Retry duplicates location | No idempotency | Double onboarding | Source event key |
| 52 | Infinite retry | No DLQ | Load/outage | Bounded retry |
| 53 | Out-of-order ignored | Async not designed | Stale status | Version/order guard |
| 54 | Schema not validated | Trust source | Corrupt data | Contract validation |
| 55 | No correlation ID | Fragmented logs | Slow RCA | Trace ID |
| 56 | Bidirectional edits free | No master | Ping-pong | Field ownership |
| 57 | POS totals uploaded manually | No integration control | Wrong KPI | Automated/reconciled aggregate |
| 58 | Location mapping by name | Weak key | Misassigned sales | Location ID |
| 59 | Metric formulas local | Autonomy without contract | Noncomparable network | Versioned metric dictionary |
| 60 | Sales and profit mixed | Incomplete costs | False economics | Finance methodology |
| 61 | Training counts people not assignments | Wrong grain | False readiness | Assignment grain |
| 62 | Quality ranks unlike cohorts | Context ignored | Harmful actions | Cohort/maturity controls |
| 63 | Migration has no inventory | Unknown sources | Missing objects | Source/volume report |
| 64 | Import without dry-run | Urgency | Mass corruption | Sample/dry-run |
| 65 | No count/sample reconciliation | HTTP success trusted | Silent loss | Control report |
| 66 | Config copied with tenant data | Clone shortcut | Cross-partner leak | Sanitized templates |
| 67 | Scaling is one wave | Schedule pressure | Network failure | Controlled rollout waves |
| 68 | Old fields removed immediately | Cleanup | Broken dependencies | Deprecation scan |
| 69 | Release without smoke | Time pressure | Production defect | Checklist |
| 70 | Rollback untested | Paper plan | Long outage | Restore drill |
| 71 | Monitoring only HTTP uptime | Technical focus | Openings/issues missed | Business signals |
| 72 | Alert includes client/financial detail | Full payload | Disclosure | Minimal alert |
| 73 | Alert has no owner | Notification only | Unresolved incident | RACI/runbook |
| 74 | Cloud assumes DB/PHP access | On-Premise habit | Impossible design | Cloud boundary |
| 75 | On-Premise core modified | Quick fix | Upgrade/security failure | Extensions |
| 76 | No change register | Oral changes | Configuration drift | Versioned docs |
| 77 | Generic training only | No role practice | Workarounds | Role-based UAT |
| 78 | Franchisee offboarding absent | Growth-only design | Orphan access/data | Closure plan |
| 79 | Archive treated as backup | Terms confused | Failed recovery | Separate policies |
| 80 | Done means configured | No evidence | Disputed result | Tested acceptance criteria |

## 23. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Confirmed AS IS/scope | Stakeholders, legal/data ownership, systems, tariff, volumes | Facts/open questions approved |
| 2. Архитектура | Approved TO BE | Entity/tenant model, master matrix, rights, NFR, ADR | Architecture/security review |
| 3. CRM настройка | Governed partner pipeline | Fields, stages, roles, automation register | Synthetic paths pass |
| 4. Пилотная точка | End-to-end opening/support/audit flow | Pilot plan, test data, rollback, acceptance | Pilot gates/UAT pass |
| 5. Обучение | Role-based readiness | LMS mappings, materials, support guide | Completion/reconciliation/sign-off |
| 6. Масштабирование | Controlled rollout waves | Sanitized template, wave plan, monitoring | Each wave smoke/reconcile pass |
| 7. Аналитика | Tenant-safe network dashboards | Metric contracts, aggregate datasets, DQ controls | Source totals/rights tests pass |
| 8. Оптимизация | Measured improvement | Baseline, DQ/incident/change backlog | KPI change verified |

### Критерии готовности

- подтверждены Cloud/On-Premise, тариф/редакция, версия, регион и права;
- утверждены data ownership, franchisee/location tenant model, master matrix and ADR;
- права HQ и каждого franchisee/location проверены позитивными и негативными тестами;
- выполнены synthetic tests, pilot UAT, migration dry-run and reconciliation;
- секреты вынесены из Git, logs/alerts не содержат partner/client details;
- утверждены rollout, rollback, monitoring, offboarding, retention and support procedures;
- бизнес-владельцы приняли измеримые критерии результата.

### Официальные источники

Использованы только записи из `knowledge/SOURCES.md`; перед реализацией URL, дата обновления и применимость проверяются повторно.

- [Официальная пользовательская документация Bitrix24](https://helpdesk.bitrix24.ru/) — CRM, задачи, права, automation and Cloud.
- [Официальная документация REST API](https://apidocs.bitrix24.ru/) — интеграции, авторизация и ограничения.
- [REST API reference](https://apidocs.bitrix24.ru/api-reference/) — методы, параметры and scopes.
- [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/index.html) — официальный инструментарий.
- [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) — создание and configuration Smart Processes.
- [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) — штатная automation.
- [Бизнес-процессы](https://helpdesk.bitrix24.ru/open/23951294/) — выбор типа процесса.
- [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) — официальные способы подключения.
- [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) — официальные datasets and limitations.
- [Маркетплейс Bitrix24](https://www.bitrix24.ru/apps/) — приложения с отдельной оценкой поставщика и безопасности.
- [Документация On-Premise](https://dev.1c-bitrix.ru/docs/portal.php) — коробочная разработка без изменения ядра.

### Ограничения и открытые вопросы

1. Cloud или On-Premise, тариф/редакция, версия, регион и права.
2. Юридическая/операционная модель сети и допустимый доступ HQ к данным франчайзи.
3. Master partner, location, employee role, contract, standard, course, sale and finance.
4. POS/ERP/1С/LMS/DMS APIs, stable IDs, volumes, SLA and replay capabilities.
5. Нужен ли единый портал, изолированные порталы или гибрид; tenant boundaries and support model.
6. Consent, customer data scope, retention/delete, export, audit and offboarding policies.
7. Opening gates, territory rules, standard adoption, audit/appeal and corrective-action SLA.
8. Central/local marketing ownership, material approval, audiences and attribution.
9. BI grain, aggregate detail, metric versions, owners, control totals and DQ thresholds.
10. Test portal/pilot location, rollout waves, downtime, rollback and support RACI.

