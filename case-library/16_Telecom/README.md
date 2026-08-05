# Telecom Customer Experience CRM Architecture Case

Архитектурный кейс для синтетической телеком-компании, продающей и сопровождающей услуги связи для B2C- и B2B-клиентов. Документ задаёт референсную инженерную модель CRM, подключений, поддержки, оборудования и аналитики; это не готовая конфигурация и не проект телеком-сети.

> **Граница безопасности:** реальные операторы, абоненты, сотрудники, адреса подключения, номера, сетевые идентификаторы, договоры, платежи, CRM ID, SIP-логины, webhook URL и токены не используются. Все данные и идентификаторы синтетические.

> **Архитектурная граница:** Bitrix24 отвечает за CRM, продажи, обращения и оркестрацию клиентского цикла. Billing и BSS остаются master тарифных назначений, лицевых счетов, начислений и статуса подписки; OSS/network inventory/NMS — ресурсов, активации и технического состояния; 1С/ERP — финансового учёта. CRM не заменяет billing, OSS/BSS, NMS или ITSM/NOC.

Обозначения:

- **факт Bitrix24** — подтверждён официальным источником из `knowledge/SOURCES.md`;
- **решение архитектора** — целевая рекомендация, утверждаемая на Discovery;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа/редакции, версии, прав, региона, приложения или внешней системы.

## 1. Business Context

### Модель телеком-бизнеса

| Аспект | Референсная модель |
|---|---|
| B2C | Массовые подключения, стандартные тарифы, самообслуживание и поддержка |
| B2B | Компании, площадки, несколько услуг, согласования и SLA |
| Каналы | Сайт, contact center, офисы, партнёры, агенты и разрешённые сообщения |
| Продажа | Opportunity от заявки до подтверждённой активации |
| Исполнение | Service order/activation в BSS/OSS, CRM хранит управляемую проекцию |
| Поддержка | Клиентский case в CRM, техническая диагностика/incident — в профильном контуре |

### Цели CRM

1. Не терять заявки и обеспечить владельца следующего шага.
2. Создать допустимый единый профиль взаимодействий клиента.
3. Связать сделку, договор, service order, услугу, оборудование и обращения stable IDs.
4. Передавать клиента из продаж в подключение и поддержку без разрыва контекста.
5. Контролировать SLA и эскалации, не подменяя технический мониторинг.
6. Получать сверяемую аналитику продаж, активаций, нагрузки и оттока.

```text
Channels -> Enquiry -> Deal -> Contract -> Service Order -> Activation
                                      |                         |
                                      v                         v
Client <------------------------- Subscription <------------ Billing/BSS
  |                                                             |
  +-> Support Case -> Technical Incident -> OSS/NMS/ITSM -------+
  +-> Renewal/Upsell
```

## 2. AS IS Problems

| Проблема | Проявление | Причина | Целевой контроль |
|---|---|---|---|
| Разрозненные клиенты | Продажи, billing и support создают дубли | Нет identity policy | Stable IDs/manual review |
| Нет единой истории | Каналы не связаны | Нет governed relations | Client–service–case model |
| Теряются заявки | Нет ack/owner/SLA | Intake не унифицирован | Routing/reconciliation |
| Разрыв sales/support | Подключение не передаёт контекст | Нет handoff gate | Activation confirmation/handoff |
| Слабая аналитика | Сделки смешаны с услугами | Не определён grain | Metric contracts |
| Неверный статус услуги | Ручное CRM-поле | Нет BSS integration | Read-only projection |
| Ошибочный тариф | CRM-копия устарела | Нет version/effective dates | Product catalog master |
| Оборудование теряется | Выдача в комментариях | Нет asset relation | OSS/WMS asset ID |
| SLA недостоверен | Нет pause/calendar rules | Упрощённая модель | Governed SLA dictionary |
| Дубли интеграций | Retry создаёт объекты | Нет idempotency | Source event ID/upsert |

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Лид | Неквалифицированное обращение, если lead model утверждена | Bitrix24 |
| CRM | Контакт | Физическое лицо | Bitrix24/MDM по ADR |
| CRM | Компания | B2B-клиент/партнёр | Bitrix24/MDM/ERP |
| CRM | Сделка | Отдельная коммерческая возможность | Bitrix24 |
| CRM | Обращение | Клиентский вопрос/жалоба | Bitrix24 |
| Smart Process | Подключение услуги | Service order coordination/projection | BSS/OSS |
| Smart Process | Тариф | Минимальная версия каталога | Product catalog/BSS |
| Smart Process | Договор | Согласование и ссылка на authoritative version | ЭДО/ERP/BSS |
| Smart Process | Оборудование | CRM-проекция выданного asset | OSS/WMS/ERP |
| Smart Process | Заявка поддержки | Structured support lifecycle | Bitrix24/ITSM по ADR |
| Smart Process | Технический инцидент | Customer-impact projection | NMS/OSS/ITSM |

**Факт Bitrix24:** CRM, задачи, Smart Processes, роботы, триггеры и бизнес-процессы описаны официальными материалами. Конкретные поля, связи, права и automation проверяются на целевом портале.

**Решение архитектора:** каждая проекция содержит `source_system`, `external_id`, `source_updated_at`, `sync_status` и correlation ID. CRM не редактирует свободно network resource, activation fact, subscription status, balance, charge или payment.

```text
Site/Calls/Chat -> Lead/Enquiry -> Contact/Company -> Deal -> Contract
                         |                              |
                         v                              v
                    Support Case                Connection Request
                         |                              |
                         v                              v
                   Technical Incident <-> OSS/NMS <- Service Order
                         |                              |
                         +---------- Subscription -----+----> Billing/BSS
                                         |
                                      Equipment <-> OSS/WMS

BI <- reconciled datasets <- Bitrix24 + BSS/OSS/1C
```

### Правила целостности

- master назначается для клиента, адреса/площадки, тарифа, услуги, service order, ресурса, equipment, charge и payment;
- B2B company, site, contact и service не смешиваются в одной карточке;
- повтор события не создаёт дубликат заявки, подключения, subscription, incident или equipment relation;
- изменение тарифа/услуги создаёт versioned event/order, а не переписывает историю;
- bulk-операции используют dry-run, sample, журнал, reconciliation и rollback;
- автоматизация не объявляет активацию или устранение аварии без authoritative event.

## 4. Customer Journey

| Этап | Цель | Минимальные данные | Контроль |
|---|---|---|---|
| Реклама | Создать спрос | Campaign/channel metadata | Attribution/consent |
| Заявка | Зарегистрировать запрос | Channel, product interest, minimum contact | Dedup/owner/SLA |
| Консультация | Уточнить потребность | Segment, service class, site/address reference | Next step |
| Выбор тарифа | Выбрать актуальную версию | Tariff/version/options | Catalog validity |
| Подключение | Создать и проследить service order | Order ID, slot, resource requirements | OSS/BSS status |
| Использование | Отражать subscription projection | Service ID/status/plan | Freshness/reconciliation |
| Поддержка | Решить customer case | Service relation, category, priority | SLA/escalation |
| Продление | Сохранить/расширить услуги | Eligibility, contract term, consent | Current-state check |

```text
Реклама -> Заявка -> Консультация -> Выбор тарифа -> Подключение
 -> Использование -> Поддержка -> Продление/Дополнительная услуга

Branches: no coverage / technical survey / rejected / cancelled / activation failed /
incident / suspension / termination.
```

## 5. Telecom Data Model

| Сущность | Минимальные поля | Контроль |
|---|---|---|
| Клиент | External customer ID, type, segment, permitted contacts, preferences | Data minimization/dedup |
| Площадка/адрес | External site ID, synthetic location reference, serviceability status | Restricted access/source timestamp |
| Услуга | Service/subscription ID, tariff version, status, start/end | BSS master/read-only |
| Подключение | Service order ID, site, plan/fact date, owner, status | BSS/OSS master |
| Тариф | Product/tariff/version ID, effective dates, options | Catalog/BSS master |
| Оборудование | Asset ID/type/status, service/customer relation, handover/return refs | OSS/WMS/ERP master |
| Обращение | Case ID, service relation, category, priority, SLA, outcome | Bitrix24/ITSM |
| Incident projection | Incident ID, affected service/scope, status, timestamps | NMS/OSS/ITSM master |

```text
Customer 1 ---- N Contract 1 ---- N Subscription N ---- 1 Tariff Version
   |                                      |    |
   +---- N Site 1 ---- N Connection Order |    +---- N Equipment Relation
   |                                      |
   +---- N Support Case ------------------+---- N Incident Relation
```

### Кардинальности

| Связь | Кардинальность | Правило |
|---|---:|---|
| Customer → Deal | 1:N | Opportunity per sale/change where approved |
| B2B Company → Site | 1:N | Site is not a duplicate Company |
| Site → Subscription | 1:N | Multiple services per site |
| Subscription → Tariff Version | N:1 | Effective version retained |
| Subscription → Equipment | 1:N over time | Issue/return history preserved |
| Incident ↔ Subscription | M:N | One incident can affect many services |

## 6. Sales Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новая заявка | Зафиксировать спрос | Channel, service interest, owner | Routing/SLA/dedup |
| Квалификация | Подтвердить сегмент и потребность | B2C/B2B, site, need, next step | Task/escalation |
| Подбор тарифа | Выбрать актуальную версию | Tariff/version/options | Catalog validation |
| Предложение | Зафиксировать условия | Offer ID/version/validity/amount | Approval/follow-up |
| Согласование | Закрыть проверки | Open items, decision owner | Structured route |
| Договор | Подтвердить документ | Contract ID/version/status | Source check |
| Подключение | Передать/контролировать order | Service order ID, plan, status | BSS/OSS sync/escalation |
| Успешно | Получить activation fact | Subscription ID, actual activation | Reconciliation/handoff |

```text
Новая заявка -> Квалификация -> Подбор тарифа -> Предложение
 -> Согласование -> Договор -> Подключение -> Успешно

Closed branches require structured reason and source context.
```

Стадии отражают подтверждённое состояние. Повторный вход не создаёт повторную задачу/уведомление. Сделка закрывается успешно только после согласованного authoritative activation/handoff event.

## 7. Service Management

| Тип | Назначение | Источник/owner | SLA-контроль |
|---|---|---|---|
| Клиентское обращение | Вопрос/изменение/жалоба | CRM service owner | Response/resolution calendar |
| Технический запрос | Диагностика одной услуги | Support/ITSM | Priority/assignment/escalation |
| Массовый incident | Common technical event | NMS/OSS/ITSM | Affected services/status projection |
| Service order | Подключение/изменение/отключение | BSS/OSS | Plan/fact/exception |
| Претензия | Formal complaint | Restricted route | Separate SLA/audit |

```text
Customer Case -> L1 Triage -> Known Incident?
       |               | yes -> Link to Incident -> Status communication
       | no            v
       +----------> L2/Engineer -> Diagnosis/Work Order -> Resolution -> Close
```

Приоритет выводится из утверждённых impact/urgency rules, а не свободного текста. Pause/reopen/calendar rules документируются; customer case и network incident не смешиваются.

## 8. Tariff Management

| Объект | Модель | Контроль |
|---|---|---|
| Тариф | Stable ID/name/category | BSS/catalog master |
| Версия | Effective-from/to/status | Immutable version |
| Услуга | Subscription relation | BSS master |
| Опция | Compatible product relation | Eligibility/version |
| Цена | Amount/currency/validity projection | Billing/catalog source |
| Изменение условий | Change order/event | No overwrite of history |

CRM хранит минимальную проекцию для sales/service. Rating, discounts, taxes, charging and eligibility остаются в BSS/billing/approved catalog; роботы не копируют тарифную логику.

## 9. Equipment Management

| Операция | CRM | OSS/WMS/ERP | Контроль |
|---|---|---|---|
| Выбор | Type/model reference | Availability/compatibility master | Freshness timestamp |
| Выдача | Handover task/status projection | Authoritative asset movement | Asset ID/document ref |
| Установка | Work task/result | Activation/resource fact | Engineer/order correlation |
| Замена | Replacement request/history | Asset movement/master | Old/new IDs |
| Возврат | Return task/status | Warehouse receipt/master | Completeness/reconciliation |
| Списание | No independent fact | ERP/WMS master | Read-only status |

Оборудование идентифицируется stable asset ID. Серийные/сетевые идентификаторы минимизируются и защищаются; CRM не считается системой складского остатка.

## 10. Support Architecture

| Уровень | Ответственность | Вход | Выход/эскалация |
|---|---|---|---|
| L1 | Идентификация, классификация, известное решение | Customer case/service context | Resolved or structured L2 handoff |
| L2 | Техническая диагностика | Diagnostic package, incident/service IDs | Resolution/work order/L3 |
| Инженер | Field/network work | Approved order, site/resource context | Fact/evidence to OSS/ITSM |
| Supervisor | Queue/SLA/quality | Breach/risk/complaint | Assignment/escalation review |
| Quality | Sample/reopen/root-cause trends | Closed cases | Improvement backlog |

Перед передачей фиксируются service ID, symptoms category, timestamps, checks already performed, expected result, priority basis and correlation ID; секреты и лишние данные исключаются.

## 11. Rights Model

| Роль | Клиенты | Услуги | Поддержка | Документы | Аналитика |
|---|---|---|---|---|---|
| Менеджер продаж | Assigned/team | Sales projection | Minimum handoff context | Sales/contract refs | Personal/team |
| Оператор поддержки | Verified assigned cases | Required service view | Create/update assigned | Service docs minimum | Queue minimum |
| Инженер | Minimum identity/site | Technical service context | Assigned work/incident | Work docs only | Operational minimum |
| Руководитель | Governed unit/portfolio | Team scope | Queue/escalations | As role requires | Unit dashboard |
| Бухгалтер | Billing identity minimum | Invoice/subscription refs | No technical narrative | Financial docs | Finance |
| Директор | Governed portfolio | Aggregate/read scope | Aggregate/oversight | Approved | Executive dashboard |
| Администратор | Technical configuration | No automatic business entitlement | No automatic case content | No automatic content | Config/dataset as needed |

Принципы: deny-by-default, least privilege, B2B portfolio/site scopes, functional separation, time-bound access, periodic recertification, export controls and negative tests.

## 12. Automation Architecture

| Автоматизация | Trigger | Действие | Защита |
|---|---|---|---|
| Intake SLA | First valid request | Assign/task/timer | Idempotency marker |
| Connection control | Authoritative order update | Update projection/task | Version/order check |
| SLA escalation | Threshold reached | Notify supervisor/route | One event per threshold |
| Known incident link | Approved correlation rule | Link case/incident | No automatic closure |
| Renewal/upsell | Eligible service signal | Deal/task/permitted message | Consent/current-state/duplicate check |
| Equipment return | Termination/replace order | Controlled task | Current asset relation |
| Integration error | Failure | Queue/alert/bounded retry | Correlation/DLQ |

```text
Event -> Validate source/version/rights -> Check idempotency -> Evaluate rule
 -> Bounded action -> Audit result -> Exception queue/escalation
```

Автоматизация не объявляет coverage, activation, restoration, charge или payment без authoritative source. Для каждого робота фиксируются repeat behavior, error handling and test case.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| Billing | Accounts, charges, balances/payment projections | Documented API/middleware | IDs, currency, reconciliation |
| OSS/BSS | Orders, subscriptions, resources, statuses | API/event bus/middleware | Ordering, idempotency, DLQ |
| 1С | Invoices/payments/accounting refs | Supported integration/middleware | Stable IDs/control totals |
| Сайт | Enquiries/serviceability requests | CRM form/official app/webhook/REST | Validation, consent, dedup |
| Телефония | Call events/CRM linkage | Bitrix24 telephony/SIP/REST app | Rights, missed-call queue, correlation |
| Чат | Authorized dialogue/case intake | Supported channel/official app | Identity, routing, retention |
| BI | Governed datasets/aggregates | BI Constructor or approved ETL | Grain, masking, freshness |

```text
Source Event -> AuthN/AuthZ -> Schema validation -> Correlation/idempotency
 -> Master ownership -> Bitrix24 write or quarantine -> Reconciliation/monitoring
```

Секреты и webhook URL находятся вне Git; retry ограничен; неизвестные справочники карантинируются; двусторонняя запись требует field ownership/conflict policy. REST methods/scopes/limits подтверждаются официальной документацией.

## 14. Marketing Automation

| Сценарий | Условие | Действие | Контроль |
|---|---|---|---|
| Сегментация | Approved minimal service/customer attributes | Governed audience | No sensitive/network inference |
| Дополнительная услуга | Compatibility/eligibility from BSS | New Deal/task | Current state/consent |
| Удержание | Approved churn-risk rule | Service task/offer | Explainable data/frequency cap |
| Кампания | Product/version/segment approved | Permitted communication | Suppression/sample/approval |
| Service message | Active case/incident purpose | Transactional update | Separate from marketing |

CRM не обещает техническую возможность, цену или восстановление без проверки master. Перед отправкой проверяются eligibility, consent, suppression, current service state and tariff version.

## 15. Analytics

| Отчёт | Grain | Показатели | Контроль качества |
|---|---|---|---|
| Продажи тарифов | Activated subscription/order | Count, revenue projection, product/channel | BSS/billing reconcile |
| Подключения | Service order | Planned/actual, lead time, failures | OSS/BSS status |
| Нагрузка поддержки | Case/agent/period | Intake, backlog, handle/resolution time | Reopen/transfer rules |
| SLA | Case/threshold/calendar | Response/resolution compliance | Pause/calendar dictionary |
| Отток | Subscription/customer/cohort | Terminations, retained, reasons | Denominator/effective dates |
| Эффективность сотрудников | Role/team/period | Conversion, task discipline, SLA | Context/capacity/assignment |

```text
CRM + BSS/Billing + OSS/ITSM + 1C -> Conformed IDs -> DQ/Reconciliation
 -> Governed Metric Layer -> Role-based Dashboard
```

Метрика имеет owner, formula, grain, time window, timezone, currency, exclusions, source, freshness SLA and control total. Доступность BI datasets зависит от тарифа и версии.

## 16. Security Architecture

| Контроль | Реализация | Проверка |
|---|---|---|
| Минимизация | Only required client/service projections | Field/file inventory |
| Разделение | Sales/support/engineering/finance scopes | Cross-unit negative tests |
| Site/network data | Restricted need-to-know fields | Unauthorized access test |
| Документы | Protected repository links | Revoked/expired access |
| Аудит | CRM history + masked integration logs | Trace by correlation ID |
| Export | Approval/minimum fields/protected target | Export test |
| Secrets | Environment/secret store/rotation | Scan/revoke drill |
| Retention | Approved archive/delete/hold policy | Periodic review |

**Cloud:** нет server/PHP/DB access; используются штатные права/история и внешние логи. **On-Premise:** дополнительно защищаются сервер, PHP, БД, файлы, cron, backup и журналы; ядро не изменяется.

## 17. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Заявка с сайта | Valid synthetic payload | One request, owner/SLA |
| 2 | Повтор webhook | Same event ID | No duplicate |
| 3 | Невалидный контакт | Missing required field | Controlled error/quarantine |
| 4 | B2B site | Company with two sites | Correct relations/no company duplicate |
| 5 | Выбор тарифа | Current version | Valid tariff projection |
| 6 | Устаревший тариф | Expired version | Offer blocked/refreshed |
| 7 | Подключение | Valid service order | Linked projection/tasks once |
| 8 | Activation retry | Accepted event then timeout | Idempotent success |
| 9 | Activation failure | OSS error | Deal not falsely successful; exception assigned |
| 10 | Equipment issue | Available synthetic asset | Relation/history created |
| 11 | Equipment return | Terminated service | Return task/fact reconciled |
| 12 | Support case | Existing subscription | Correct SLA and service link |
| 13 | Mass incident | Two affected services | Cases linked; no automatic false close |
| 14 | Cross-unit rights | Unauthorized user | Client/service/incident unavailable |
| 15 | Billing mismatch | Different synthetic total | Incident/reconciliation |
| 16 | Consent withdrawal | Suppression event | Marketing stops |
| 17 | Renewal/upsell | Eligible service | New Deal once; old history preserved |
| 18 | BI reconciliation | Synthetic control totals | Threshold pass or incident |
| 19 | Cloud/On-Premise | Capability matrix | Unsupported assumption detected |
| 20 | Rollback | Failed release | Verified prior state restored |

Тесты используют только synthetic clients, sites, services, equipment and events. Проверяются positive, negative, boundary, concurrency, retry, rights, audit and recovery.

## 18. Deployment

| Этап | Результат | Gate |
|---|---|---|
| Discovery | AS IS, systems, tariff, volumes, data classes | Business/security approval |
| Архитектура | TO BE, master matrix, rights, ADR, NFR | Architecture review |
| Настройка | CRM/Smart Processes/automation in test scope | Configuration review |
| Интеграции | Contracts/secrets/retry/monitoring | Failure/replay tests |
| Перенос данных | Inventory/map/cleanse/dry-run/report | Count/sample reconciliation |
| Обучение | Role-based synthetic exercises | UAT sign-off |
| Запуск | Window/runbook/support/rollback | Go/no-go |
| Stabilization | Incidents and baseline | Exit criteria |

```text
Inventory -> Dependency review -> Backup/export -> Test configuration
 -> Synthetic tests -> UAT -> Migration dry-run -> Go/No-Go
 -> Release -> Smoke/reconciliation -> Hypercare or Rollback
```

Рабочие поля, стадии, роли, роботы и интеграции не удаляются/переименовываются до dependency analysis, backup/export, pilot and approved rollback.

## 19. Monitoring

| Сигнал | Правило | Действие | Владелец |
|---|---|---|---|
| Потерянная заявка | Intake SLA/no ack | Assign/escalate | Sales operations |
| SLA breach | Threshold/calendar | Route/escalate | Support supervisor |
| Billing mismatch | Control total/status difference | Finance/billing incident | Billing owner |
| Stale service projection | Freshness exceeded | Mark stale/stop unsafe action | BSS owner |
| Integration error | Error/retry/DLQ | Incident/replay after fix | Integration support |
| Просроченная задача | Deadline breached | Owner/manager escalation | Process owner |
| Unlinked incident | Affected service without relation | Correlation review | NOC/support |
| BI stale/mismatch | Dataset SLA/control total | Data incident | BI owner |

Alert содержит system, object reference, correlation ID, time, severity, owner and runbook; без секретов, адресов и лишних клиентских данных.

## 20. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Дубли клиентов | Multiple channels | Fragmented history | Stable IDs/dedup |
| 2 | False merge | Weak identifiers | Wrong service disclosure | Conservative review |
| 3 | Lost enquiry | No ack | Lost sale | Intake reconciliation |
| 4 | Duplicate Deal | Retry | Funnel distortion | Event idempotency |
| 5 | Site mixed with Company | Flat B2B model | Wrong service relation | Separate site entity |
| 6 | Stale tariff | No version sync | Wrong offer | Effective version/freshness |
| 7 | Price edited in CRM | No master | Billing dispute | Billing/catalog authority |
| 8 | False serviceability | CRM guess | Failed installation | OSS authoritative check |
| 9 | Deal closed before activation | KPI pressure | False success | Activation gate |
| 10 | Duplicate service order | Retry | Duplicate work/resource | Source order ID |
| 11 | Out-of-order status | Async events | Old state wins | Version/order guard |
| 12 | Subscription overwritten | One field | Lost history | Effective-dated versions |
| 13 | Equipment lost | No asset ID | Financial/operational loss | OSS/WMS relation |
| 14 | Equipment stock stale | CRM copy | Promise unavailable asset | Freshness/master check |
| 15 | Return not recorded | No closure gate | Asset loss | Return reconciliation |
| 16 | Case and incident mixed | One object | Wrong SLA/impact | Separate lifecycles |
| 17 | Mass incident creates duplicates | Per-case detection | Support overload | Shared incident relation |
| 18 | Automatic incident close | Source status assumed | Unresolved customer issue | Customer confirmation/rule |
| 19 | Priority from free text | No matrix | Wrong escalation | Impact/urgency dictionary |
| 20 | SLA has no pause rules | Simplification | False KPI | Governed calendar |
| 21 | Sales/support handoff absent | Separate teams | Repeated questions | Handoff checklist |
| 22 | B2B portfolio access broad | Common role | Data leak | Portfolio/site scopes |
| 23 | Engineer sees finance | Shared entity | Excess access | Field/function minimum |
| 24 | Admin is super-reader | Role confusion | Uncontrolled access | Separate privileges |
| 25 | Temporary access persists | No expiry | Residual access | Recertification |
| 26 | Export unrestricted | Default rights | Mass leak | Approval/minimum fields |
| 27 | Address/network data in logs | Raw payload | Sensitive leak | Masking |
| 28 | Secret in Git | Real example | Compromise | Secret store/rotation |
| 29 | Broad integration token | One user | Large blast radius | Least privilege |
| 30 | Retry not idempotent | Timeout ambiguity | Duplicates | Source event ID |
| 31 | Infinite retry | No DLQ | Load/outage | Bounded backoff |
| 32 | Schema drift | OSS/BSS change | Corrupt fields | Contract validation |
| 33 | No correlation ID | Fragmented logs | Slow RCA | End-to-end trace |
| 34 | Two status masters | Bidirectional editing | Ping-pong | Field ownership |
| 35 | Billing total not reconciled | HTTP success trusted | Hidden mismatch | Control totals |
| 36 | Chat identity wrong | Weak verification | Data disclosure | Identity/authorization policy |
| 37 | Marketing without consent | Suppression ignored | Compliance risk | Eligibility/consent check |
| 38 | Churn score opaque | Uncontrolled model | Harmful targeting | Explainable governance |
| 39 | Service message mixed with marketing | Purpose confused | Missed/illegal message | Purpose separation |
| 40 | Migration without dry-run | Urgency | Mass corruption | Sample/report/rollback |
| 41 | Real subscribers in test | Prod copy | Data breach | Synthetic data |
| 42 | Wrong analytics grain | Orders/services mixed | Double count | Metric contract |
| 43 | Churn denominator wrong | Suspended/ineligible included | False KPI | Cohort definition |
| 44 | Tariff not checked | Assumption | Feature unavailable | Capability assessment |
| 45 | Cloud treated as server | DB/PHP expected | Impossible design | Cloud boundary |
| 46 | Core modified On-Premise | Quick fix | Upgrade/security failure | Extensions only |
| 47 | Release lacks rights tests | Happy path only | Exposure | Negative suite |
| 48 | No rollback | Success assumed | Long outage | Tested rollback |
| 49 | Alert contains full client data | Full payload | Disclosure | Minimal alert |
| 50 | No support owner | Missing RACI | Long incident | Runbook/on-call |
| 51 | No baseline | AS IS unmeasured | Effect unprovable | Baseline/KPI |
| 52 | No change register | Oral changes | Poor support | Versioned register |
| 53 | Capacity spike ignored | No volume model | SLA collapse | Load forecast/testing |
| 54 | NMS outage hides CRM status | Single dependency | False silence | Dependency monitoring |
| 55 | Archive confused with backup | No restore design | Failed recovery | Separate policies |

## 21. ADR

| ADR | Решение | Альтернатива | Обоснование | Последствие |
|---|---|---|---|---|
| ADR-TEL-001 | Contact/Company plus Site relation | Universal customer object | Native CRM semantics and B2B topology | Identity/site policy required |
| ADR-TEL-002 | Subscription is BSS projection | CRM as service master | Activation/status belong to BSS | Freshness/reconciliation needed |
| ADR-TEL-003 | Connection order separated from Deal | Track activation only in stages | Different owner/lifecycle | Reliable Deal–order link |
| ADR-TEL-004 | Support Case separated from technical Incident | One Smart Process for both | Customer SLA differs from network event | Correlation model required |
| ADR-TEL-005 | Equipment is OSS/WMS projection | CRM inventory | Asset movement needs specialist master | Stable asset IDs/integration |
| ADR-TEL-006 | Middleware/contracts govern integrations | Free point-to-point webhooks | Central idempotency/monitoring | Platform owner required |
| ADR-TEL-007 | Rights split by function/portfolio/site | Broad department access | Least privilege | Explicit scopes/negative tests |
| ADR-TEL-008 | BI uses reconciled conformed datasets | Raw CRM reports | OSS/BSS/billing facts differ | Metric/DQ ownership |

ADR получает owner, status `proposed/accepted/superseded`, date, context, alternatives, consequences, portal/version and evidence.

## 22. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Нет Discovery | Immediate setup | Wrong scope | AS IS/stakeholders |
| 2 | Deployment type unknown | Template design | Unsupported solution | Cloud/On-Premise check |
| 3 | Tariff not verified | Assumption | Blocked release | Verify portal |
| 4 | CRM declared OSS/BSS | Scope confusion | Wrong service facts | Architecture boundary |
| 5 | All requests are Leads | Universal lead | Support mixed with sales | Classification |
| 6 | Leads disabled without ADR | Fashion | Unqualified demand mixed | Lead decision |
| 7 | Contact/Company mixed | Flat customer field | Wrong relationships | Native entities |
| 8 | Company duplicated by service/site | Shortcut | Fragmented history | Company–Site relation |
| 9 | Name is integration key | No external ID | Wrong match | Stable IDs |
| 10 | Phone is sole dedup key | Convenience | False merge | Conservative matching |
| 11 | B2B site stored in comment | No site model | Broken fulfillment | Structured Site |
| 12 | One Deal for all services | Entity saving | No sales history | Deal per opportunity |
| 13 | Service only in Deal fields | No subscription entity | Lost lifecycle | BSS projection |
| 14 | Tariff and version mixed | No effective dates | Wrong offer | Versioned catalog |
| 15 | Billing rules copied to robots | Fast prototype | Wrong charge | Billing master |
| 16 | Serviceability guessed | No OSS call | Failed install | Authoritative check |
| 17 | Stages describe tasks | Process mistake | Ambiguous state | State-based stages |
| 18 | Too many stages | Every action is stage | Low adoption | Tasks/checklists |
| 19 | Fields mandatory too early | Maximum control | Fake data | Stage requiredness |
| 20 | Loss reason in comment | No dictionary | No analytics | Governed codes |
| 21 | No next step | Stage-only control | Stalled Deal | Activity policy |
| 22 | Deal successful before activation | KPI shortcut | False conversion | Activation gate |
| 23 | Re-entry duplicates tasks | Stateless robot | Fake workload | Idempotency marker |
| 24 | Robots loop stages | Mutual transitions | Automation storm | Loop guard |
| 25 | Contract overwritten | No versioning | Wrong terms | Versioned reference |
| 26 | Service order lacks external ID | Local numbering | Duplicate activation | Source ID |
| 27 | Subscription status manual | No BSS sync | False service state | Read-only projection |
| 28 | Change tariff overwrites history | Flat field | Audit loss | Change order/version |
| 29 | Disconnect treated as Delete | Cleanup | Lost history | Termination state/archive |
| 30 | Equipment stored as text | No asset relation | Lost device | Stable asset ID |
| 31 | CRM maintains stock | No WMS integration | False availability | WMS/ERP master |
| 32 | Replacement drops old asset | One current field | Missing history | Old/new relation |
| 33 | Return has no reconciliation | Task only | Asset loss | Warehouse fact |
| 34 | Case equals incident | One queue | Wrong SLA | Separate lifecycles |
| 35 | Every case creates new incident | No correlation | NOC overload | Known-incident link |
| 36 | Incident closes all cases automatically | Simplification | Unresolved customers | Case closure rule |
| 37 | Priority chosen subjectively | No matrix | SLA distortion | Impact/urgency |
| 38 | SLA calendar absent | Simplification | False breaches | Calendar/pause rules |
| 39 | Reopen resets SLA silently | No policy | Misleading KPI | Reopen rule |
| 40 | L1 handoff lacks diagnostics | Weak template | Repeated work | Structured package |
| 41 | Sales sees technical secrets | Broad fields | Security risk | Field minimum |
| 42 | Engineer sees billing details | Shared access | Excess disclosure | Functional scope |
| 43 | Accountant sees support narrative | Common entity rights | Excess disclosure | Finance projection |
| 44 | Admin equals business reader | Role confusion | Uncontrolled access | Separate privileges |
| 45 | Temporary rights never expire | Manual grant | Residual risk | Expiry/recertification |
| 46 | Rights only positive-tested | Happy path | Hidden cross-access | Negative tests |
| 47 | Export unrestricted | Defaults | Mass leak | Approval/minimum fields |
| 48 | Secret in README | Real example | Compromise | Placeholder/secret store |
| 49 | Webhook committed | Convenience | Unauthorized calls | Environment/rotation |
| 50 | Token overprivileged | One credential | Large blast radius | Least privilege |
| 51 | Raw payload logged | Debugging | Client/network data leak | Masking |
| 52 | Retry duplicates order | No idempotency | Duplicate field work | Source event key |
| 53 | Infinite retry | No DLQ | Load/outage | Bounded retry |
| 54 | Out-of-order ignored | Async not designed | Stale status | Version/order guard |
| 55 | Schema not validated | Trust source | Corrupt data | Contract validation |
| 56 | No correlation ID | Fragmented logs | Slow RCA | Trace ID |
| 57 | Bidirectional edits free | No master | Ping-pong | Field ownership |
| 58 | Batch ignores limits | Premature optimization | Partial errors | Official checks/report |
| 59 | Migration has no inventory | Unknown sources | Missing objects | Source/volume report |
| 60 | Import without dry-run | Urgency | Mass corruption | Sample/dry-run |
| 61 | Real subscribers in UAT | Prod copy | Privacy breach | Synthetic data |
| 62 | No count/sample reconciliation | HTTP success trusted | Silent loss | Control report |
| 63 | BI counts orders as subscriptions | Wrong grain | Inflated sales | Metric contract |
| 64 | SLA ignores transfers/pause | Simple formula | False result | SLA dictionary |
| 65 | Churn includes planned migration | Bad reason codes | False KPI | Governed cohort/reasons |
| 66 | Staff ranked without queue context | Simplistic KPI | Harmful incentives | Contextual metrics |
| 67 | No data-quality controls | Dashboard-first | False confidence | DQ monitoring |
| 68 | Old fields removed immediately | Cleanup | Broken dependencies | Deprecation scan |
| 69 | Release without smoke | Time pressure | Production defect | Checklist |
| 70 | Rollback untested | Paper plan | Long outage | Restore drill |
| 71 | Monitoring only HTTP uptime | Technical focus | Requests still lost | Business signals |
| 72 | Alert includes address/full dialogue | Full payload | Disclosure | Minimal alert |
| 73 | Alert lacks owner | Notification only | Unresolved incident | RACI/runbook |
| 74 | Cloud assumes DB/PHP access | On-Premise habit | Impossible design | Cloud boundary |
| 75 | On-Premise core modified | Quick fix | Upgrade/security failure | Extensions |
| 76 | No change register | Oral changes | Poor support | Versioned docs |
| 77 | Generic training only | No role practice | Workarounds | Role-based UAT |
| 78 | No hypercare | Immediate closure | Incident backlog | Stabilization |
| 79 | Archive treated as backup | Terms confused | Failed recovery | Separate policies |
| 80 | Done means configured | No evidence | Disputed result | Tested acceptance criteria |

## 23. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Confirmed AS IS/scope | Stakeholders, systems, tariff, volumes, data classes | Facts/open questions approved |
| 2. Архитектура | Approved TO BE | Entity model, master matrix, rights, NFR, ADR | Architecture/security review |
| 3. CRM настройка | Governed intake/sales | Fields, stages, roles, automation register | Synthetic paths pass |
| 4. Услуги | Subscription/connection projections | IDs, lifecycle, tariff versions, equipment relations | BSS/OSS UAT passes |
| 5. Поддержка | Case/incident model | SLA, priority, routing, handoff, closure | End-to-end support tests pass |
| 6. Интеграции | Reliable exchange | Contracts, secrets, retry, idempotency, monitoring | Failure/replay/reconcile pass |
| 7. Тестирование | Evidence-based quality | Test plan, synthetic data, rights suite, UAT | Critical defects closed/sign-off |
| 8. Запуск | Controlled release | Cutover, backup/export, rollback, support | Smoke/reconciliation pass |
| 9. Оптимизация | Measured improvement | Baseline, DQ/incident/change backlog | KPI change verified |

### Критерии готовности

- подтверждены Cloud/On-Premise, тариф/редакция, версия, регион и права;
- утверждены master-system matrix, entity boundaries, rights and ADR;
- проверены права ролей позитивными и негативными тестами;
- выполнены synthetic tests, UAT, migration dry-run and reconciliation;
- секреты вынесены из Git, logs/alerts не содержат чувствительных данных;
- утверждены release, rollback, monitoring, incident, retention and support procedures;
- бизнес-владельцы приняли измеримые критерии результата.

### Официальные источники

Использованы только записи из `knowledge/SOURCES.md`; перед реализацией URL, дата обновления и применимость проверяются повторно.

- [Официальная пользовательская документация Bitrix24](https://helpdesk.bitrix24.ru/) — CRM, задачи, права, automation and Cloud.
- [Официальная документация REST API](https://apidocs.bitrix24.ru/) — интеграции, авторизация и ограничения.
- [REST API reference](https://apidocs.bitrix24.ru/api-reference/) — методы, параметры and scopes.
- [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/) — официальный инструментарий.
- [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) — создание and configuration Smart Processes.
- [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) — штатная automation.
- [Бизнес-процессы](https://helpdesk.bitrix24.ru/open/23951294/) — выбор типа процесса.
- [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) — официальные способы подключения.
- [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) — официальные datasets and limitations.
- [Маркетплейс Bitrix24](https://www.bitrix24.ru/apps/) — приложения с отдельной оценкой поставщика и безопасности.
- [Документация On-Premise](https://dev.1c-bitrix.ru/docs/portal.php) — коробочная разработка без изменения ядра.

### Ограничения и открытые вопросы

1. Cloud или On-Premise, тариф/редакция, версия, регион и права.
2. B2C/B2B segments, channels, lead model, portfolios and service geography.
3. Master клиента, site/address, product/tariff, subscription, order, asset, charge and payment.
4. Billing/OSS/BSS/NMS/ITSM/1С APIs, stable IDs, event order, volumes, SLA and replay.
5. Допустимый состав клиентских, адресных и технических данных в CRM.
6. Consent/suppression, retention/delete, export and audit policies.
7. Service/incident SLA calendars, priority, pause/reopen and escalation rules.
8. Serviceability, activation, tariff change, equipment issue/return and termination gates.
9. BI grain, metric owners, control totals, churn methodology and DQ thresholds.
10. Test portal/pipeline, release window, downtime, rollback and support RACI.

# Related Documents

- [Case Library](../README.md)
- [Architecture Patterns](../PATTERNS.md)
- [Discovery Template](../DISCOVERY_TEMPLATE.md)
- [Architecture Template](../ARCHITECTURE_TEMPLATE.md)
