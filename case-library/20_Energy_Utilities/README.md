# Energy and Utilities Customer Service CRM Architecture Case

Архитектурный кейс для синтетической энергетической/коммунальной компании, обслуживающей B2C- и B2B-клиентов, подключения, обращения и сервисные работы. Документ задаёт CRM-архитектуру, а не готовую конфигурацию, диспетчерскую систему или проект промышленной автоматизации.

> **Граница безопасности:** реальные компании, потребители, сотрудники, адреса, объекты инфраструктуры, схемы сети, показания, лицевые счета, договоры, платежи, телефоны, email, CRM ID, webhook URL и токены не используются. Все примеры синтетические.

> **Архитектурная граница:** Bitrix24 отвечает за CRM, обращения, коммерческие возможности и координацию клиентского сервиса. CIS/billing остаётся master лицевого счёта, начислений и платежного статуса; GIS/EAM — объектов и активов; OMS/SCADA/NMS — технического/аварийного состояния; WFM/ERP — нарядов и работ; 1С — финансов. CRM не подключается напрямую к OT и не отправляет управляющие команды в SCADA.

Обозначения:

- **факт Bitrix24** — подтверждён официальным источником из `knowledge/SOURCES.md`;
- **решение архитектора** — целевая рекомендация, утверждаемая на Discovery;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа/редакции, версии, прав, региона, приложения или внешней системы.

## 1. Business Context

### Модель энергетического бизнеса

| Контур | Назначение | Master |
|---|---|---|
| B2C | Физлица, premises/service points, договоры и обращения | CIS/billing + CRM interaction |
| B2B | Компании, площадки, несколько договоров/точек | CIS/ERP + CRM |
| Подключения | Проверка возможности, условия, работы и активация | GIS/EAM/WFM/CIS |
| Эксплуатация | Активы, техническое состояние и обслуживание | EAM/SCADA/OMS/WFM |
| Клиентский сервис | Contact center, requests, complaints, notifications | Bitrix24 |
| Финансы | Начисления, счета, оплаты и корректировки | CIS/billing/1С |

### Цели CRM

1. Создать допустимую единую историю взаимодействий клиента.
2. Не терять обращения и назначать владельца следующего шага.
3. Связать customer, premise/site, service point, contract, connection and case stable IDs.
4. Передавать продажи/сервис профильным системам без ручной подмены технических фактов.
5. Контролировать SLA, эскалации, коммуникации и качество сервиса.
6. Получать сверяемую аналитику обращений, работ и инцидентов.

```text
Channels -> Customer/Company -> Premise/Site -> Contract/Service Point
                 |                    |                  |
                 v                    v                  v
             CRM Case          Connection Request -> Work Order -> CIS/GIS/EAM
                 |                                       |
                 +-> Incident Relation <- OMS/SCADA ----+
                 +-> Billing Question <--- CIS/1C
```

## 2. AS IS Problems

| Проблема | Проявление | Причина | Целевой контроль |
|---|---|---|---|
| Нет единой истории | Контакты разбросаны по каналам | Нет identity/case model | Stable IDs and activity policy |
| Теряются обращения | Нет ack/owner/SLA | Intake не унифицирован | Routing/reconciliation |
| Нет контроля заявок | Статус в таблицах/звонках | Нет service lifecycle | Structured stages/milestones |
| Разрыв сервиса и contact center | Клиент повторяет данные | Нет case–work relation | Correlation and handoff |
| Слабая аналитика | Cases/incidents/orders смешаны | Grain не определён | Metric contracts |
| Неверный аварийный статус | CRM обновляется вручную | Нет OMS projection | Read-only source status |
| Адрес равен клиенту | Плоская модель | Нет premise/service point | Separate entities/relations |
| Дубли работ | Retry создаёт наряды | Нет idempotency | Source request/order ID |
| Оплата отмечается вручную | CRM считается billing | Нет reconciliation | CIS/1С authority |
| OT данные попадают в CRM | Scope creep | Нет IT/OT boundary | DMZ/middleware/minimization |

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Лид | Неквалифицированный коммерческий запрос, если lead model утверждена | Bitrix24 |
| CRM | Контакт | Физическое лицо/представитель | Bitrix24/MDM |
| CRM | Компания | B2B-клиент/контрагент | Bitrix24/CIS/ERP |
| CRM | Сделка | Коммерческая возможность/подключение по утверждённой модели | Bitrix24 |
| CRM | Обращение | Вопрос, service request или complaint | Bitrix24 |
| Smart Process | Объект инфраструктуры | Минимальная business projection | GIS/EAM |
| Smart Process | Подключение | Coordination/status projection | CIS/GIS/WFM |
| Smart Process | Договор | Согласование и ссылка на версию | CIS/ERP/DMS |
| Smart Process | Техническая заявка | Customer-facing technical case/work request | Bitrix24/ITSM/WFM |
| Smart Process | Аварийный случай | Customer-impact projection | OMS/SCADA/NMS |
| Smart Process | Сервисная работа | Work order projection | EAM/WFM |

**Факт Bitrix24:** CRM, задачи, Smart Processes, роботы, триггеры и бизнес-процессы описаны официальными материалами. Конкретные права, поля, связи и automation проверяются на целевом портале.

**Решение архитектора:** проекции содержат `source_system`, `external_id`, `source_updated_at`, `sync_status` и correlation ID. Интеграция с OT идёт только через утверждённый защищённый IT/OT gateway/middleware; CRM не хранит команды, пароли или полные технологические схемы.

```text
Site/Calls -> Lead/Enquiry -> Contact/Company -> Deal/Contract
                         |              |              |
                         v              v              v
                      CRM Case     Premise/Site -> Connection Request
                         |              |              |
                         |              v              v
                         +---- Service Point <- Work Order <- WFM/EAM
                         |              |
                         v              v
                   Incident Link <- IT/OT Gateway <- OMS/SCADA/NMS
                         |
                     CIS/Billing/1C -> financial projection

BI <- reconciled datasets <- Bitrix24 + CIS/GIS/EAM/OMS/WFM
```

### Правила целостности

- customer, premise/site, service point, contract, asset, case, incident and work order имеют разные IDs;
- master назначается для адресного объекта, актива, подключения, аварии, работы, начисления и оплаты;
- incident не создаётся из каждого customer case; cases связываются с подтверждённым incident;
- repeated event не создаёт duplicate case, connection, work order or notification;
- аварийный/технический статус не редактируется свободно в CRM;
- bulk operations используют dry-run, sample, журнал, reconciliation and rollback.

## 4. Customer Journey

| Этап | Цель | Минимальные данные | Контроль |
|---|---|---|---|
| Обращение клиента | Зарегистрировать контакт | Channel, topic, minimum identity | Verification/dedup/owner |
| Регистрация заявки | Классифицировать запрос | Premise/service point ref, category, received time | SLA/routing |
| Проверка возможности | Получить authoritative feasibility | GIS/CIS request/result refs | Source/version |
| Подготовка решения | Согласовать scope/terms/work | Decision/offer version, dependencies | Human/system gate |
| Подключение | Создать и проследить work/order | Connection/order IDs, plan/status | WFM/GIS/CIS sync |
| Эксплуатация | Вести service relationship | Contract/service point/status projection | CIS master |
| Сервис | Решить case/work | Case/work/incident relations | SLA/escalation |
| Повторное обращение | Создать новый case/opportunity | New purpose/context | Preserve history/no duplicate |

```text
Обращение -> Регистрация заявки -> Проверка возможности -> Подготовка решения
 -> Подключение -> Эксплуатация -> Сервис -> Повторное обращение

Branches: not feasible / documents required / work delayed / incident linked /
billing dispute / complaint / controlled closure.
```

## 5. Energy Data Model

| Сущность | Минимальные поля | Контроль |
|---|---|---|
| Клиент | External customer ID, type, permitted contacts, verification status | Minimize PII |
| Компания | External company ID, representatives, contract refs | Name not sole key |
| Premise/site | External premise ID, synthetic address ref, customer relation | Address restricted |
| Service point | Service point ID, premise, contract/service refs, status | CIS master |
| Объект/актив | Asset ID/type/status/parent relation projection | GIS/EAM master |
| Договор | Contract ID/version/status/effective dates | CIS/ERP/DMS master |
| Заявка | Case ID/type/priority/status/owner/SLA | Bitrix24/ITSM |
| Работа | Work order ID, asset/site/case, plan/fact/status | WFM/EAM master |

```text
Customer N ---- M Premise/Site 1 ---- N Service Point N ---- 1 Contract
   |                    |                     |
   +---- N CRM Case ----+                     +---- N Billing Ref
                        |                     +---- N Connection/Work Order
                        v                                 |
                  Incident Relation N ---- M Asset <-----+
```

### Кардинальности

| Связь | Кардинальность | Правило |
|---|---:|---|
| Customer ↔ Premise | M:N over time | Effective relation/role required |
| Premise → Service Point | 1:N | Utilities/services remain distinct |
| Contract → Service Point | 1:N or M:N by approved model | CIS semantics preserved |
| Case ↔ Incident | M:N | One outage can affect many cases |
| Case → Work Order | 1:0..N | Work only when approved/needed |
| Asset → Work Order | 1:N | Asset hierarchy stays in EAM/GIS |

## 6. Service Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новая заявка | Зафиксировать обращение | Channel, topic, customer/premise refs, owner | Ack/routing/dedup |
| Квалификация | Проверить identity/type/impact | Category, service point, urgency, next step | SLA/task |
| Анализ | Собрать source context | CIS/OMS/GIS status refs, checks | Known incident matching/task |
| Назначение исполнителя | Передать accountable role/team | Assignee, due date, handoff package | Routing/escalation |
| Выполнение работ | Координировать action/work order | Work ID, plan/status, dependencies | WFM sync/reminder |
| Проверка | Подтвердить результат | Source completion, customer/service validation | Review task |
| Закрытие | Зафиксировать outcome | Resolution code, open items, timestamps | Completeness/reopen rule |

```text
Новая заявка -> Квалификация -> Анализ -> Назначение исполнителя
 -> Выполнение работ -> Проверка -> Закрытие

Known incident path: Case -> Incident relation -> Updates -> Service restored source
 -> Case verification/closure rule.
```

Re-entry не создаёт повторные задачи/уведомления. Case не закрывается только по SCADA/OMS status, если утверждённая customer verification/reopen policy требует дополнительной проверки.

## 7. Infrastructure Management

| Область | CRM projection | Master/контроль |
|---|---|---|
| Объект | Asset/site ID, type, business status | GIS/EAM |
| Иерархия | Parent/system/service relation | GIS/EAM; no full network topology in CRM |
| Технические параметры | Только необходимый read-only subset | EAM/SCADA |
| Состояние | Timestamped summarized status | OMS/SCADA/NMS |
| Обслуживание | Work plan/status references | EAM/WFM |
| История | Linked work/incident refs | Source history, not CRM overwrite |

```text
GIS/EAM Asset Hierarchy -> IT/OT Gateway -> Minimal CRM Projection
          |                                      |
          +-> OMS/SCADA State                    +-> Case/Work relation
          +-> WFM Work Orders                    +-> Customer communication
```

CRM не хранит control commands, engineering credentials, full topology, protection settings or unrestricted telemetry.

## 8. Connection Management

| Этап | Данные | Gate | Master |
|---|---|---|---|
| Запрос | Customer/premise/service type | Identity/completeness | CRM/CIS |
| Техническая проверка | Feasibility request/result/version | Authorized engineering result | GIS/EAM/planning |
| Решение/условия | Scope, dependencies, validity | Approval/contract | CIS/ERP/DMS |
| Планирование | Connection/work order, plan date | Resource/permit readiness | WFM/EAM |
| Выполнение | Work status/evidence ref | Safety/quality process outside CRM | WFM/EAM |
| Активация | Service point/contract status | Authoritative activation | CIS/GIS |

Нельзя обещать feasibility, срок или activation по локальному CRM-полю. Изменения сроков и условий сохраняются версиями с owner/reason/source timestamp.

## 9. Incident Management

| Элемент | Модель | Контроль |
|---|---|---|
| Incident | External incident ID, status, impact scope projection | OMS/SCADA/ITSM master |
| Приоритет | Approved impact/urgency/severity | No free-text decision |
| Customer case | Individual request/communication | Separate SLA/lifecycle |
| Связь | Case–incident/service point relation | Correlation evidence |
| Эскалация | Role/time/threshold/runbook | Human acknowledgement |
| Closure | Source restoration + approved case closure rule | Reopen monitoring |

```text
Technical Signal -> OMS/SCADA/ITSM Incident -> Impact Scope
                              |                    |
                              v                    v
                       Dispatch/Work Orders    Linked Customer Cases
                              |                    |
                              +-> Restoration ----+-> Communication/Verification
```

Emergency process имеет резервные каналы и не зависит только от Bitrix24. CRM получает минимальную projection через защищённую IT/OT boundary.

## 10. Customer Support Architecture

| Уровень | Ответственность | Вход | Выход |
|---|---|---|---|
| Contact center/L1 | Verify, classify, known information | Customer/service context | Resolve or structured handoff |
| L2/service specialist | Analyze CIS/known incident/work context | Diagnostic package | Resolution/work/escalation |
| Dispatcher | Coordinate incident/resources | Approved incident/work data | Assignment/status update |
| Engineer | Execute authorized work | WFM order and safety process | Work fact to master |
| Quality/supervisor | SLA, reopen, complaint review | Cases/outcomes | Corrective action |

Перед передачей фиксируются case/service point IDs, timestamps, category, performed checks, expected result, priority basis and correlation ID. Лишние персональные и OT-данные исключаются.

## 11. Rights Model

| Роль | Клиенты | Объекты | Заявки | Документы | Аналитика |
|---|---|---|---|---|---|
| Оператор | Verified assigned/minimum | Service point summary | Create/update assigned | Service refs minimum | Queue minimum |
| Инженер | Customer minimum | Assigned asset/work subset | Assigned technical work | Work docs | Operational |
| Диспетчер | Impact/customer aggregate | Incident/resource projection | Incident-linked cases | Runbook refs | Dispatch |
| Руководитель направления | Governed unit/portfolio | Unit scope | Team/SLA/escalations | Approved | Unit dashboard |
| Бухгалтер | Billing identity minimum | Service point ref | Billing cases only | Financial docs | Finance |
| Директор | Governed aggregate/read scope | Aggregate | Oversight | Approved | Executive |
| Администратор | Technical configuration | No automatic OT/content entitlement | No business entitlement | No automatic content | Config/dataset as needed |

Принципы: deny-by-default, least privilege, region/unit/asset scopes, IT/OT separation, named accounts, temporary access expiry, recertification, export controls and negative tests.

## 12. Automation Architecture

| Автоматизация | Trigger | Действие | Защита |
|---|---|---|---|
| Auto-routing | First valid classified case | Assign queue/owner/task | Deterministic rules/idempotency |
| SLA control | Threshold reached | Escalate/notify | One event per threshold |
| Known incident | Confirmed correlation | Link case/show approved update | No incident creation/closure |
| Work order status | Authoritative WFM event | Update projection/task | Version/order check |
| Customer update | Approved current status | Transactional notification | Identity/channel/current-state check |
| Reopen | New contact/failed verification | Reopen/escalate per rule | Preserve history |
| Integration failure | Error | Queue/alert/bounded retry | Correlation/DLQ |

```text
Event -> Validate source/version/rights/IT-OT boundary -> Idempotency check
 -> Bounded CRM action -> Audit result -> Exception queue/escalation
```

Автоматизация не отправляет SCADA commands, не создаёт technical fact, не подтверждает safety, restoration, charge or payment без authoritative/human gate.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| CIS/Billing | Customers, contracts, service points, charges/payment projections | Documented API/middleware | IDs, amounts, freshness, reconciliation |
| SCADA/OMS/NMS | Minimal incident/impact/status projection | Segmented IT/OT gateway/middleware | Read-only, allowlist, no commands, audit |
| GIS/EAM | Premises/assets/hierarchy/work refs | API/middleware | Stable IDs/version/minimization |
| ERP/WFM | Connection/work orders, resources, plan/fact | API/event/middleware | Idempotency/order/reconciliation |
| 1С | Invoices/payments/accounting refs | Supported integration/middleware | Amount/currency/control totals |
| Телефония | Call events/CRM linkage | Bitrix24 telephony/SIP/REST app | Rights/missed-call/correlation |
| BI | Governed datasets/aggregates | BI Constructor or approved ETL | Grain/masking/freshness/DQ |

```text
OT Systems -> One-way/approved IT-OT Gateway -> Integration Layer
                                                   |
CIS/GIS/EAM/ERP -> AuthN/AuthZ + Schema + Idempotency
                                                   |
                                                   v
                                         Bitrix24 projection/cases
                                                   |
                                         Monitoring/Reconciliation
```

Секреты хранятся вне Git; OT credentials never enter CRM; methods, data flows and direction are allowlisted; retry bounded; unknown mappings quarantined. REST methods/scopes/limits проверяются по официальной документации.

## 14. Analytics

| Отчёт | Grain | Показатели | Контроль качества |
|---|---|---|---|
| Обращения | Case/category/channel/period | Volume/backlog/reopen | Duplicate/status rules |
| Скорость обработки | Case/milestone | First response/resolution/wait | Calendar/pause rules |
| SLA | Case/threshold | Compliance/breach/reason | Approved denominator |
| Загрузка инженеров | Work order/engineer/period | Assigned/plan/fact/overdue | WFM/capacity context |
| Аварийность | Incident/asset/period | Incidents/duration/affected projection | OMS master/grain |
| Качество сервиса | Case/outcome/sample | Repeat, complaint, quality review | No context-free ranking |

```text
CRM + CIS/Billing + GIS/EAM + OMS/WFM -> Conformed IDs
 -> DQ/Reconciliation -> Governed Metric Layer -> Role-based Dashboard
```

Метрика имеет owner, formula, grain, period, calendar/timezone, exclusions, source, freshness SLA and control total. Cases, incidents, assets and work orders не суммируются как один grain.

## 15. Security Architecture

| Контроль | Реализация | Проверка |
|---|---|---|
| IT/OT boundary | Segmented gateway, allowlisted read projection | No direct CRM-to-SCADA path |
| Customer minimization | Only required identity/service data | Field/file inventory |
| Asset detail | Need-to-know summarized projection | Unauthorized asset test |
| Documents | Protected DMS refs | Expired/revoked access |
| Audit | CRM history + masked integration/security logs | Correlation trace |
| Export | Approval/minimum fields/protected target | Export test |
| Secrets | Secret store/rotation, none in Git/logs | Scan/revoke drill |
| Retention | Approved archive/delete/hold policy | Periodic review |

**Cloud:** нет server/PHP/DB access; интеграционный boundary и логи находятся во внешнем защищённом контуре. **On-Premise:** дополнительно защищаются server, PHP, DB, files, cron, backup and logs; core is not modified.

## 16. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | B2C case | Valid synthetic customer/service point | One case, owner/SLA |
| 2 | B2B multi-site | Company with two premises | Correct relations/no company duplicate |
| 3 | Duplicate event | Same source ID | No duplicate |
| 4 | Invalid identity | Missing verification | Sensitive action blocked/routed |
| 5 | Connection feasibility | Authoritative result | Correct status/version |
| 6 | Connection delayed | WFM milestone breach | Escalation once/customer task |
| 7 | Work completion | Authoritative WFM fact | Projection updated, verification gate |
| 8 | Known outage | Two synthetic cases/one incident | Both linked, no duplicate incident |
| 9 | Incident restore | OMS status restored | Approved updates; no unsafe auto-close |
| 10 | Reopen | Customer reports unresolved | Case reopened/history preserved |
| 11 | Billing mismatch | CIS vs 1С synthetic totals | Incident/reconciliation |
| 12 | Cross-unit rights | Unauthorized user | Customer/asset/case unavailable |
| 13 | OT boundary | Attempted CRM control flow | Rejected; no command path |
| 14 | Integration timeout | Accepted then timeout | Retry idempotent |
| 15 | Alert masking | Incident generated | No address/OT secret/full PII |
| 16 | SLA pause | Approved waiting state | Correct clock/history |
| 17 | BI grain | Cases/incidents/work orders | Separate correct metrics |
| 18 | Migration dry-run | Synthetic sample | Report without production write |
| 19 | Cloud/On-Premise | Capability matrix | Unsupported assumption detected |
| 20 | Rollback | Failed release | Verified previous state restored |

Тесты используют только synthetic customers, premises, service points, assets, cases and incidents. OT проверяется через безопасные mocks/test gateways; production control systems не используются.

## 17. Deployment

| Этап | Результат | Gate |
|---|---|---|
| Discovery | AS IS, systems, tariff, volumes, criticality, IT/OT boundary | Business/security/OT approval |
| Архитектура | TO BE, master matrix, rights, ADR, NFR | Architecture/threat review |
| Настройка | CRM/Smart Processes/automation in isolated scope | Configuration review |
| Интеграции | Contracts/gateway/secrets/retry/monitoring | Failure/security/replay tests |
| Обучение | Role-based synthetic scenarios/runbooks | UAT/sign-off |
| Запуск | Window/cutover/support/rollback | Go/no-go |
| Stabilization | Incidents/baseline/operational observation | Exit criteria |

```text
Inventory -> Threat/Dependency Review -> Test Gateway/Config -> Synthetic Tests
 -> UAT + Failure/DR Drills -> Go/No-Go -> Release -> Smoke/Reconciliation
 -> Hypercare or Rollback
```

Никакая CRM-настройка не внедряется напрямую в OT. Рабочие поля, стадии, роли и интеграции не удаляются/переименовываются до dependency analysis, backup/export, pilot and approved rollback.

## 18. Monitoring

| Сигнал | Правило | Действие | Владелец |
|---|---|---|---|
| Необработанная заявка | Intake SLA/no ack | Assign/escalate | Contact center owner |
| SLA breach | Threshold/calendar | Escalate/review | Service supervisor |
| Аварийный процесс | Incident/impact/freshness threshold | Incident runbook | Dispatcher/OMS owner |
| Integration error | Error/retry/DLQ | Incident/replay after fix | Integration support |
| Просроченная задача | Deadline breached | Owner/manager escalation | Process owner |
| Stale CIS/OMS projection | Freshness exceeded | Mark stale/stop unsafe message | Source owner |
| IT/OT anomaly | Unexpected direction/access | Security incident/isolation | OT security owner |
| BI mismatch | Control total difference | Mark stale/investigate | BI owner |

Alert содержит system, object reference, correlation ID, time, severity, owner and runbook; без OT credentials, topology, полного адреса и лишних клиентских данных.

## 19. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Дубли клиентов | Multiple sources | Fragmented service | Stable IDs/dedup |
| 2 | False merge | Weak identifiers | Wrong contract disclosure | Conservative review |
| 3 | Premise equals customer | Flat model | Wrong history | Separate relations |
| 4 | Service point omitted | Address-only model | Wrong service context | Stable service-point ID |
| 5 | Lost case | No ack | Service failure | Intake reconciliation |
| 6 | Duplicate case/work | Retry | Double dispatch | Idempotency |
| 7 | CRM controls SCADA | Scope/security failure | OT incident | No command path/segmentation |
| 8 | OT credentials in CRM | Convenience | Critical compromise | Secret isolation |
| 9 | Full topology copied | Overintegration | Security exposure | Minimal projection |
| 10 | Stale incident status | Delayed integration | Wrong customer message | Freshness/source check |
| 11 | Every case creates incident | No correlation | Dispatcher overload | Known-incident linking |
| 12 | Incident auto-closes cases | Simplification | Unresolved customer problem | Verification/reopen rule |
| 13 | Priority free-text | No matrix | Wrong response | Impact/urgency rules |
| 14 | SLA calendar absent | Simplification | False breach | Governed calendar |
| 15 | Emergency relies on CRM only | No fallback | Missed critical event | Redundant channels/runbook |
| 16 | Feasibility guessed | No GIS/planning source | Invalid promise | Authoritative check |
| 17 | Connection success manual | KPI pressure | False activation | CIS/GIS gate |
| 18 | Work closes without evidence | Task status trusted | Unsafe/incomplete service | WFM/reviewer gate |
| 19 | Asset ID unstable | Name/address key | Wrong work relation | EAM asset ID |
| 20 | Asset hierarchy duplicated | CRM as EAM | Drift | Source projection |
| 21 | Billing status manual | Convenience | Financial error | CIS/1С authority |
| 22 | Charge/payment mixed | Flat field | Wrong communication | Separate finance facts |
| 23 | Customer receives stale outage update | No pre-send check | Trust damage | Current-state validation |
| 24 | Cross-region access | Broad role | Data leak | Region/unit scopes |
| 25 | Engineer sees finance | Shared record | Excess access | Functional minimum |
| 26 | Operator sees OT details | Broad projection | Security risk | Summarized fields |
| 27 | Admin as super-reader | Role confusion | Uncontrolled access | Separate privileges |
| 28 | Export unrestricted | Defaults | Mass leak | Approval/minimum fields |
| 29 | Secret in Git | Real example | Compromise | Secret store/rotation |
| 30 | Broad token | One integration user | Large blast radius | Least privilege |
| 31 | Raw logs contain address/OT | Debugging | Data/security leak | Masking |
| 32 | Infinite retry | No DLQ | Load/outage | Bounded backoff |
| 33 | Out-of-order event | Async ignored | Old state wins | Version/order guard |
| 34 | Schema drift | Source change | Corrupt projection | Contract validation |
| 35 | No correlation ID | Fragmented logs | Slow RCA | End-to-end trace |
| 36 | Bidirectional edits free | No master | Ping-pong | Field ownership |
| 37 | Real utility data in tests | Prod copy | Data exposure | Synthetic data |
| 38 | Migration without dry-run | Urgency | Mass corruption | Sample/report/rollback |
| 39 | No reconciliation | HTTP success trusted | Silent loss | Counts/control totals |
| 40 | BI mixes case/incident/work | Wrong grain | False KPI | Metric contract |
| 41 | Engineer ranking lacks job complexity | Simplistic KPI | Harmful decisions | Context/capacity |
| 42 | Incident duration uses CRM timestamps | Wrong source | False reliability | OMS source |
| 43 | Peak outage load untested | Average design | CRM/SLA collapse | Load/degradation tests |
| 44 | Tariff not checked | Assumption | Feature unavailable | Capability assessment |
| 45 | Cloud treated as OT/on-prem server | DB/PHP expectation | Impossible/unsafe design | Cloud boundary |
| 46 | Core modified On-Premise | Quick fix | Upgrade/security failure | Extensions only |
| 47 | Release lacks rights/security tests | Happy path only | Exposure | Negative/threat suite |
| 48 | Rollback absent | Success assumed | Long outage | Tested rollback |
| 49 | Alert has no owner | Notification only | Unresolved incident | RACI/runbook |
| 50 | No baseline | AS IS unmeasured | Effect unprovable | Baseline/KPI |
| 51 | No change register | Oral changes | Configuration drift | Versioned register |
| 52 | Source outage hidden | No dependency monitoring | Stale silence | Health/freshness monitor |
| 53 | Customer verification weak | Speed | Unauthorized disclosure/change | Approved identity process |
| 54 | Archive confused with backup | No restore design | Failed recovery | Separate policies |
| 55 | DR ignores integration gateway | CRM-only restore | Broken service after recovery | End-to-end DR test |

## 20. ADR

| ADR | Решение | Альтернатива | Обоснование | Последствие |
|---|---|---|---|---|
| ADR-ENU-001 | Contact/Company linked to separate Premise and Service Point | Address fields on client only | Supports B2B/moves/multiple services | More relation governance |
| ADR-ENU-002 | Infrastructure object is minimal GIS/EAM projection | CRM as asset registry | Engineering hierarchy belongs to EAM/GIS | Integration/freshness required |
| ADR-ENU-003 | Customer Case separated from technical Incident and Work Order | One Smart Process for all | Different owners/grains/SLA | Correlation model needed |
| ADR-ENU-004 | CRM has no direct OT control path | Direct SCADA integration | Reduces cyber/operational risk | Gateway/read-only projections |
| ADR-ENU-005 | Connection lifecycle references CIS/GIS/WFM source gates | Deal stages as technical truth | Feasibility/work/activation need masters | More async integration |
| ADR-ENU-006 | Rights split by function/region/asset scope | Broad utility-wide roles | Least privilege/IT-OT separation | Negative tests required |
| ADR-ENU-007 | Middleware/contracts govern integrations | Free point-to-point webhooks | Security/idempotency/monitoring | Platform owner required |
| ADR-ENU-008 | BI uses reconciled conformed grains | Raw CRM reports | Cases/incidents/work/finance differ | Metric/DQ ownership |

ADR получает owner, status `proposed/accepted/superseded`, date, context, alternatives, consequences, portal/version and evidence.

## 21. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Нет Discovery | Immediate setup | Wrong scope | AS IS/stakeholders |
| 2 | Cloud/On-Premise unknown | Template design | Unsupported solution | Capability check |
| 3 | Tariff not verified | Assumption | Blocked release | Verify portal |
| 4 | CRM declared CIS/SCADA/EAM | Scope confusion | False/unsafe facts | Architecture boundary |
| 5 | All requests are Leads | Universal lead | Service mixed with sales | Classification |
| 6 | Leads disabled without ADR | Fashion | Unqualified demand mixed | Lead decision |
| 7 | Contact/Company mixed | Flat customer field | Wrong B2B relations | Native entities |
| 8 | Premise stored only in address field | No topology | Wrong service history | Premise entity |
| 9 | Service point omitted | Address equals service | Contract/case mismatch | Stable service-point relation |
| 10 | Name/address is integration key | No external ID | Wrong match | Stable IDs |
| 11 | One Deal for all connections | Entity saving | No lifecycle analytics | Deal/request per opportunity |
| 12 | Connection stored only in Deal | No work/source relation | False activation | Connection projection |
| 13 | Case/incident/work merged | One object easier | Wrong SLA/owner/grain | Separate lifecycles |
| 14 | Every call creates incident | No correlation | Incident flood | Known-incident link |
| 15 | Incident created from free text | Overautomation | False outage | OMS authority |
| 16 | CRM sends SCADA command | Direct integration | Critical OT risk | No control path |
| 17 | OT credentials stored in webhook | Convenience | Compromise | Segmented secret store |
| 18 | Full telemetry copied to CRM | Overintegration | Load/security risk | Minimal aggregates |
| 19 | Asset hierarchy recreated manually | No EAM sync | Drift | GIS/EAM projection |
| 20 | Asset identified by name | Weak key | Wrong work | Asset ID |
| 21 | Stages describe tasks | Process mistake | Ambiguous state | State-based stages |
| 22 | Too many stages | Every action is stage | Low adoption | Tasks/checklists |
| 23 | Fields mandatory too early | Maximum control | Fake values | Stage requiredness |
| 24 | Resolution in comment | No dictionary | No analytics | Governed codes |
| 25 | No next step | Stage-only control | Stalled case | Activity policy |
| 26 | Robot duplicates tasks | Re-entry ignored | Fake load | Idempotency marker |
| 27 | Robots loop stages | Mutual transitions | Automation storm | Loop guard |
| 28 | Feasibility marked manually | No GIS/planning | Invalid promise | Source result |
| 29 | Connection date promised locally | No WFM capacity | Missed commitment | Source plan/version |
| 30 | Activation equals task completion | Wrong gate | False service status | CIS/GIS confirmation |
| 31 | Work closes without verification | Task trusted | Unresolved problem | Evidence/reviewer gate |
| 32 | Incident closes all cases | Simplification | Customer unresolved | Case closure policy |
| 33 | Reopen resets history/SLA | No rule | False metrics | Reopen policy |
| 34 | Priority subjective | No impact matrix | Wrong response | Impact/urgency |
| 35 | SLA calendar absent | Simplification | False breach | Calendar/pause rules |
| 36 | Emergency uses normal queue | No runbook | Delayed response | Dedicated route/fallback |
| 37 | Auto-response treated as human ack | Automation optimism | No owner | Human acknowledgement |
| 38 | Billing status manual | Convenience | Financial error | CIS/1С source |
| 39 | Charge/payment mixed | One field | Wrong communication | Separate facts |
| 40 | Contract overwritten | No versioning | Wrong terms | Immutable source refs |
| 41 | Operator sees all regions | Broad role | Data leak | Region/queue scope |
| 42 | Engineer sees customer finance | Shared record | Excess access | Functional minimum |
| 43 | Accountant sees OT narrative | Common rights | Excess disclosure | Finance projection |
| 44 | Admin equals business/OT reader | Role confusion | Uncontrolled access | Separate privileges |
| 45 | Temporary access never expires | Manual grant | Residual risk | Expiry/recertification |
| 46 | Rights only positive-tested | Happy path | Hidden cross-access | Negative tests |
| 47 | Export unrestricted | Defaults | Mass leak | Approval/minimum fields |
| 48 | Secret in README | Real example | Compromise | Placeholder/secret store |
| 49 | Webhook committed | Convenience | Unauthorized calls | Environment/rotation |
| 50 | Token overprivileged | One credential | Large blast radius | Least privilege |
| 51 | Raw payload logged | Debugging | Client/OT leak | Masking |
| 52 | Retry duplicates work order | No idempotency | Double dispatch | Source event key |
| 53 | Infinite retry | No DLQ | Load/outage | Bounded retry |
| 54 | Out-of-order ignored | Async not designed | Stale status | Version/order guard |
| 55 | Schema not validated | Trust source | Corrupt data | Contract validation |
| 56 | No correlation ID | Fragmented logs | Slow RCA | Trace ID |
| 57 | Bidirectional edits free | No master | Ping-pong | Field ownership |
| 58 | Batch ignores limits | Premature optimization | Partial errors | Official checks/report |
| 59 | Migration has no inventory | Unknown sources | Missing objects | Source/volume report |
| 60 | Import without dry-run | Urgency | Mass corruption | Sample/dry-run |
| 61 | Real customer/asset data in UAT | Prod copy | Security/privacy breach | Synthetic data |
| 62 | No count/sample reconciliation | HTTP success trusted | Silent loss | Control report |
| 63 | BI counts cases as incidents | Wrong grain | False reliability | Metric contract |
| 64 | Incident duration from CRM | Wrong source | False KPI | OMS timestamps |
| 65 | Engineer ranking ignores job complexity | Simplistic KPI | Harmful decisions | Context/capacity |
| 66 | No data-quality monitor | Dashboard-first | False confidence | DQ/freshness controls |
| 67 | Peak outage not load-tested | Average design | CRM collapse | Load/degraded-mode test |
| 68 | Old fields removed immediately | Cleanup | Broken dependencies | Deprecation scan |
| 69 | Release without smoke | Time pressure | Production defect | Checklist |
| 70 | Rollback untested | Paper plan | Long outage | Restore drill |
| 71 | DR tests CRM only | Integration ignored | Broken recovery | End-to-end DR |
| 72 | Monitoring only HTTP uptime | Technical focus | Cases/incidents missed | Business signals |
| 73 | Alert contains address/topology | Full payload | Disclosure | Minimal alert |
| 74 | Alert has no owner | Notification only | Unresolved incident | RACI/runbook |
| 75 | Cloud assumes DB/OT access | On-Premise habit | Impossible/unsafe design | Cloud boundary |
| 76 | On-Premise core modified | Quick fix | Upgrade/security failure | Extensions |
| 77 | No change register | Oral changes | Configuration drift | Versioned docs |
| 78 | Generic training only | No role/emergency drills | Unsafe workaround | Role-based UAT/runbook |
| 79 | Archive treated as backup | Terms confused | Failed recovery | Separate policies |
| 80 | Done means configured | No evidence | Disputed result | Tested acceptance criteria |

## 22. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Confirmed AS IS/scope | Stakeholders, systems, criticality, tariff, volumes, IT/OT boundary | Facts/open questions approved |
| 2. Архитектура | Approved TO BE | Entity/grain model, master matrix, threat model, rights, ADR | Architecture/security/OT review |
| 3. CRM настройка | Governed intake/service pipeline | Fields, stages, roles, automation register | Synthetic paths pass |
| 4. Сервисные процессы | Case/connection/incident/work projections | SLA, routing, relations, runbooks | End-to-end UAT passes |
| 5. Интеграции | Secure reliable exchange | Gateway/contracts/secrets/retry/idempotency/monitoring | Security/failure/replay/reconcile pass |
| 6. Тестирование | Evidence-based quality | Test plan, synthetic data, threat/rights/load/DR tests | Critical defects closed/sign-off |
| 7. Запуск | Controlled release | Cutover, backup/export, rollback, support/fallback | Smoke/reconciliation pass |
| 8. Оптимизация | Measured improvement | Baseline, DQ/incident/change backlog | KPI change verified |

### Критерии готовности

- подтверждены Cloud/On-Premise, тариф/редакция, версия, регион и права;
- утверждены IT/OT boundary, master matrix, entity grains, rights and ADR;
- отсутствует direct CRM-to-SCADA control path; gateway design прошёл security review;
- выполнены synthetic UAT, negative rights, load/degraded-mode, migration dry-run and reconciliation;
- секреты вынесены из Git, logs/alerts не содержат topology/credentials/full addresses;
- утверждены release, rollback, DR, emergency fallback, monitoring and support procedures;
- бизнес- и OT-владельцы приняли измеримые критерии результата.

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
2. Utility types, B2C/B2B processes, lead usage, service regions and owners.
3. Master customer, premise, service point, contract, asset, incident, work, charge and payment.
4. CIS/GIS/EAM/OMS/SCADA/WFM/1С APIs, stable IDs, event order, volumes, SLA and replay.
5. IT/OT segmentation, gateway direction, allowlists, credentials, monitoring and DR.
6. Допустимый состав address/asset/customer data, retention/export/audit policies.
7. Identity verification, feasibility, activation, incident correlation, closure/reopen and emergency rules.
8. SLA calendars, priority/impact, degraded mode, alternate channels and peak-load targets.
9. BI grain, metric owners, source timestamps, control totals and DQ thresholds.
10. Test portal/gateway, release window, downtime, rollback and support/OT RACI.

