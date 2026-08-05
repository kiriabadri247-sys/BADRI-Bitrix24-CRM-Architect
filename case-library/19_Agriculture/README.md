# Agriculture and Agribusiness CRM Architecture Case

Архитектурный кейс для синтетической агропромышленной компании, объединяющей производство, продажи, снабжение, качество и поставки. Документ задаёт референсную CRM-архитектуру и не является агрономической рекомендацией, системой точного земледелия или готовой конфигурацией.

> **Граница безопасности:** реальные компании, поля, координаты, сотрудники, клиенты, поставщики, договоры, урожаи, маршруты, платежи, телефоны, email, CRM ID, webhook URL и токены не используются. Все данные и идентификаторы синтетические.

> **Архитектурная граница:** Bitrix24 отвечает за CRM, коммерческие возможности, обращения и координацию заказов/поставок. ERP/FMIS/GIS остаются master производственных планов, полей и агроопераций; WMS/ERP — партий и остатков; LIMS/quality system — результатов качества; fleet/telematics — техники; 1С/ERP — финансов. CRM не заменяет эти профильные контуры.

Обозначения:

- **факт Bitrix24** — подтверждён официальным источником из `knowledge/SOURCES.md`;
- **решение архитектора** — целевая рекомендация, утверждаемая на Discovery;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа/редакции, версии, прав, региона, приложения или внешней системы.

## 1. Business Context

### Модель агробизнеса

| Контур | Назначение | Предпочтительный master |
|---|---|---|
| Производство | Поля, сезоны, культуры, операции, урожай | FMIS/GIS/ERP |
| Продажи | Запросы, предложения, договоры, заказы | Bitrix24 + ERP order facts |
| Снабжение | Поставщики, материалы, условия и поставки | ERP/1С/procurement system |
| Склад | Партии, остатки, размещение и отгрузка | WMS/ERP |
| Качество | Пробы, спецификации и результаты | LIMS/quality system |
| Логистика | Рейсы, транспорт и delivery facts | TMS/fleet/ERP |

### Цели CRM

1. Создать управляемый профиль клиента и поставщика без дублей.
2. Не терять запросы и следующий шаг продажи.
3. Связать сделку, customer order, product/lot, договор и поставку stable IDs.
4. Передавать доступность и качество из master-систем без ручного переопределения.
5. Контролировать сроки, исключения, рекламации и коммуникации.
6. Получать сверяемую аналитику продаж, поставок и plan/fact.

```text
Production Plan -> Field/Season -> Harvest Lot -> Quality -> Warehouse
                                                      |
Customer Enquiry -> Deal -> Contract -> Customer Order+-> Delivery -> Customer
                                                      |
Supplier -> Procurement/Inbound ----------------------+
```

## 2. AS IS Problems

| Проблема | Проявление | Причина | Целевой контроль |
|---|---|---|---|
| Нет единой базы клиентов | Дубли по подразделениям | Нет identity policy | Stable IDs/dedup review |
| Теряются заявки | Нет owner/SLA | Каналы не унифицированы | Intake/routing/escalation |
| Нет контроля поставок | Статус в звонках/таблицах | Нет delivery lifecycle | Source status/milestones |
| Разрыв производства и продаж | Обещается непроизведённый объём | Нет availability contract | ERP/WMS projection |
| Слабая аналитика | Разные единицы/grain | Нет metric dictionary | Conformed units/datasets |
| План смешан с фактом | Одно поле объёма | Нет version/scenario | Separate plan/forecast/actual |
| Нет traceability | Партия в комментарии | Нет lot relation | Stable lot/batch ID |
| Качество вводится вручную | Нет LIMS integration | CRM считается master | Read-only quality projection |
| Поставщики дублируются | Имя используется как ключ | Нет external ID | Supplier identity policy |
| Retry создаёт заказы | Нет idempotency | Системные дубли | Source event ID/upsert |

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Лид | Неквалифицированный запрос, если lead model утверждена | Bitrix24 |
| CRM | Контакт | Представитель клиента/поставщика | Bitrix24/MDM |
| CRM | Компания | Клиент или поставщик с контекстной ролью | Bitrix24/ERP/MDM |
| CRM | Сделка | Отдельная коммерческая возможность | Bitrix24 |
| CRM | Обращение | Вопрос, сервисный запрос или рекламация | Bitrix24 |
| Smart Process | Сельскохозяйственное поле | Минимальная operational projection | FMIS/GIS |
| Smart Process | Урожай | Season/crop/lot projection | FMIS/ERP/WMS |
| Smart Process | Продукция | Catalog/specification projection | ERP/PIM |
| Smart Process | Поставка | Delivery coordination/projection | TMS/ERP |
| Smart Process | Поставщик | Supplier lifecycle projection | ERP/procurement |
| Smart Process | Договор | Согласование и ссылка на версию | DMS/ЭДО/ERP |
| Smart Process | Заказ клиента | Order coordination/projection | ERP |

**Факт Bitrix24:** CRM, задачи, Smart Processes, роботы, триггеры и бизнес-процессы описаны официальными материалами. Конкретные связи, права, поля и automation проверяются на целевом портале.

**Решение архитектора:** проекции содержат `source_system`, `external_id`, `source_updated_at`, `sync_status` и correlation ID. CRM не редактирует свободно урожайность, остаток, quality result, inventory allocation, route fact или payment.

```text
Site/Calls -> Lead/Enquiry -> Contact/Company -> Deal -> Contract
                                      |                    |
                                      v                    v
Supplier -> Inbound Supply -> ERP/WMS <- Customer Order <- Offer
                                  |
FMIS/GIS -> Field -> Harvest Lot -> Quality/LIMS -> Allocation -> Delivery
                                                               |
                                                               v
                                                            Customer

BI <- reconciled datasets <- Bitrix24 + FMIS/ERP/WMS/LIMS/TMS
```

### Правила целостности

- master назначается для field, season, crop, harvest lot, product, stock, quality, order, delivery and finance;
- plan, forecast, commitment, allocation and actual хранятся раздельно;
- единицы измерения, валюта, timezone and conversion version обязательны;
- repeated event не создаёт duplicate order, lot, delivery, task or complaint;
- изменения партии/поставки сохраняют историю и effective dates;
- bulk operations используют dry-run, sample, журнал, reconciliation and rollback.

## 4. Agriculture Data Model

| Сущность | Минимальные поля | Контроль |
|---|---|---|
| Клиент | External customer ID, segment, permitted contacts, purchase aggregates | No unnecessary personal data |
| Поставщик | Supplier ID, category, status, contract refs, quality aggregate | ERP/procurement master |
| Поле | Field ID, synthetic zone reference, area/unit, status | FMIS/GIS; no real coordinates |
| Урожай/партия | Season ID, crop/product, lot ID, quantity/unit, status | FMIS/ERP/WMS |
| Продукция | Product/SKU/specification/version IDs | ERP/PIM master |
| Заказ | Order ID, customer, lines, amount/currency, dates/status | ERP master |
| Поставка | Delivery ID, order/lot refs, plan/fact, carrier/vehicle refs, status | TMS/ERP |
| Quality result | Test/result ID, lot/spec version, status, timestamp | LIMS/quality master |

```text
Field 1 ---- N Season/Crop ---- N Harvest Lot N ---- 1 Product/Spec
                                              |             |
Customer 1 ---- N Deal ---- N Customer Order -+---- N Allocation
                                                            |
                                                            v
Supplier -> Inbound Lot -> Warehouse -----------------> Delivery
                                                            |
                                                        Quality Result
```

### Кардинальности

| Связь | Кардинальность | Правило |
|---|---:|---|
| Customer → Deal/Order | 1:N | Opportunity/order separate |
| Field → Season | 1:N over time | Season ID required |
| Season → Harvest Lot | 1:N | Lot never represented only by product name |
| Order → Order Line | 1:N | Analytics grain explicit |
| Order Line ↔ Lot | M:N | Allocation table with quantity/unit |
| Order → Delivery | 1:N | Partial deliveries preserved |

## 5. Sales Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новый запрос | Зафиксировать спрос | Channel, product interest, owner | Routing/SLA/dedup |
| Квалификация | Подтвердить клиента и потребность | Segment, quantity/unit, timing, next step | Task/escalation |
| Определение потребности | Уточнить specification/logistics | Product/spec, volume, delivery basis | Completeness check |
| Расчёт предложения | Получить цену/availability | Offer ID/version, source refs, currency | ERP query/approval |
| Согласование | Закрыть разногласия | Open items, decision owner/date | Follow-up/escalation |
| Договор | Подтвердить условия | Contract ID/version/status | DMS/ERP gate |
| Поставка | Контролировать confirmed order/delivery | Order/delivery IDs, plan/fact/status | Source sync/exception tasks |
| Закрытие | Подтвердить исполнение | Delivery/acceptance/finance refs | Reconciliation/open-item check |

```text
Новый запрос -> Квалификация -> Определение потребности -> Расчёт предложения
 -> Согласование -> Договор -> Поставка -> Закрытие

Branches: no stock/forecast only / specification mismatch / quality hold /
partial delivery / cancellation / complaint.
```

Стадия отражает подтверждённое состояние. Deal не закрывается успешно только по CRM-флагу без утверждённого delivery/acceptance gate. Re-entry не дублирует задачи и сообщения.

## 6. Production Management

| Объект | Данные | Master/контроль |
|---|---|---|
| Поле | ID, area/unit, zone, status | FMIS/GIS |
| Сезон | Season/crop plan version | FMIS/ERP |
| План | Scenario/version/quantity/unit/date | Planning system |
| Операция | Work type, plan/fact, resources | FMIS/work management |
| Урожай | Lot/quantity/unit/harvest date | FMIS/ERP/WMS |
| Forecast | Version/date/confidence/range | Approved planning source |
| Контроль | Deviations, owner, action | Governed exception process |

```text
Field -> Season Plan -> Agricultural Operations -> Harvest Event
                   |                                |
                   v                                v
              Forecast Versions               Harvest Lots -> Quality/Warehouse
```

CRM получает только необходимую проекцию для sales promise и coordination. Weather, agronomy, telemetry and yield calculations не реализуются роботами Bitrix24 без отдельной подтверждённой системы и методологии.

## 7. Supplier Management

| Область | Модель | Контроль |
|---|---|---|
| Поставщик | Company + supplier external ID/status | No duplicate Company per material |
| Договор | Contract/version/effective dates | DMS/ERP master |
| Условия | Price/delivery/payment/spec refs | Version/source |
| Поставка | Inbound order/delivery/lot refs | ERP/WMS fact |
| Качество | Approved aggregate/result refs | LIMS master |
| Развитие | Corrective plan/review | Structured owner/deadline |

Supplier score имеет formula, period, source, sample/volume context and owner. Он не строится по субъективным комментариям и не заменяет procurement decision.

## 8. Logistics Architecture

| Этап | Данные | Контроль | Master |
|---|---|---|---|
| Планирование | Delivery ID, order/lines, quantity/unit, window | Capacity/route approval | TMS/ERP |
| Назначение | Carrier/vehicle/driver refs | Rights/availability | TMS/fleet |
| Погрузка | Lot allocation, quantity, timestamp | WMS confirmation | WMS/ERP |
| Перевозка | Status/milestones/exception | Source timestamp | TMS/telematics |
| Доставка | Plan/fact, accepted quantity, evidence ref | Customer/order reconciliation | ERP/TMS |
| Возврат/расхождение | Reason, quantity/unit, claim relation | Structured exception | ERP/quality |

```text
Customer Order -> Lot Allocation -> Pick/Load -> Transport -> Delivery
        |                |              |            |          |
        +------------ Reconciliation/Exceptions ----------------+
```

CRM не является системой маршрутизации или телематики. Реальные координаты и персональные данные водителей не включаются в общие CRM-поля без утверждённой необходимости.

## 9. Quality Management

| Элемент | Назначение | Контроль |
|---|---|---|
| Спецификация | Product/spec/version and limits | ERP/PIM/LIMS master |
| Проба | Sample ID, lot, time, procedure/version | Chain-of-custody ref |
| Результат | Test/result ID/status/timestamp | LIMS authoritative |
| Quality hold | Lot/status/reason/owner | Blocks unsafe allocation by source rule |
| Рекламация | Customer/order/lot, category, owner, SLA | Restricted case lifecycle |
| Corrective action | Cause/action/deadline/evidence | Reviewer closure |

```text
Specification Version -> Lot -> Sample/Test -> Result
                              |                  |
                              +-> Hold/Release <-+
                                      |
                         Delivery or Complaint/Corrective Action
```

CRM не вычисляет лабораторный результат и не снимает hold. Решение приходит из quality/LIMS master; автоматизация только координирует задачи и коммуникации.

## 10. Rights Model

| Роль | Клиенты | Производство | Поставки | Документы | Аналитика |
|---|---|---|---|---|---|
| Менеджер продаж | Assigned/team | Availability/forecast minimum | Customer deliveries | Sales/contract refs | Personal/team |
| Агроном | No client detail by default | Assigned fields/seasons/operations | Input context minimum | Production docs | Production |
| Логист | Customer/site minimum | Lot/allocation context | Assigned deliveries | Logistics docs | Logistics |
| Снабжение | Supplier scope | Material demand minimum | Inbound deliveries | Supplier contracts | Procurement |
| Бухгалтер | Billing identity minimum | No agronomy detail | Invoice/order refs | Financial docs | Finance |
| Директор | Governed portfolio | Aggregate/read | Aggregate/read | Approved | Executive |
| Администратор | Technical configuration | No automatic business entitlement | No content entitlement | No automatic content | Config/dataset as needed |

Принципы: deny-by-default, department/field/location scopes, least privilege, separation of duties, temporary access expiry, recertification, export controls and negative tests.

## 11. Automation Architecture

| Автоматизация | Trigger | Действие | Защита |
|---|---|---|---|
| Intake SLA | First valid request | Assign/task/timer | Idempotency marker |
| Offer validation | Before send/expiry | Refresh price/availability task | Source/version check |
| Delivery control | Milestone approaching/breached | Task/escalation | Current delivery version |
| Quality hold | Authoritative hold event | Block/flag related coordination | No CRM release decision |
| Contract approval | Defined threshold/change | Structured route | Version/audit |
| Supplier exception | Late/quality event | Corrective task/review | Unique event key |
| Integration failure | Error | Queue/alert/bounded retry | Correlation/DLQ |

```text
Event -> Validate source/version/unit/rights -> Check idempotency -> Rule
 -> Bounded action -> Audit result -> Exception queue/escalation
```

Автоматизация не создаёт производственный факт, не обещает stock, не меняет LIMS result и не подтверждает доставку/оплату без authoritative event.

## 12. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| ERP | Products, orders, contracts, plans/facts | Documented API/middleware | IDs, units, versions, reconcile |
| 1С | Invoices/payments/accounting refs | Supported integration/middleware | Amount/currency/control totals |
| WMS/склад | Lots, stock, allocations, shipments | API/event/middleware | Lot IDs/freshness/idempotency |
| FMIS/GIS | Fields, seasons, operations, forecast/harvest refs | API/ETL/middleware | Version/unit/source timestamp |
| Техника/fleet | Asset/work/telemetry summary | API/middleware | Asset IDs/minimization/freshness |
| BI | Governed datasets/aggregates | BI Constructor or approved ETL | Grain/units/masking/DQ |
| Телефония | Call events/CRM linkage | Bitrix24 telephony/SIP/REST app | Rights/missed-call/correlation |

```text
Source Event -> AuthN/AuthZ -> Schema + unit validation
 -> Correlation/idempotency -> Master ownership
 -> Bitrix24 write or quarantine -> Reconciliation/monitoring
```

Секреты хранятся вне Git; retry ограничен; unknown product/lot/unit mappings карантинируются; двусторонняя запись требует field ownership. REST methods/scopes/limits подтверждаются официальной документацией.

## 13. Analytics

| Отчёт | Grain | Показатели | Контроль качества |
|---|---|---|---|
| Продажи продукции | Confirmed order/delivery line | Quantity, revenue, product/customer | ERP reconcile/units |
| Прибыльность | Product/customer/season/period | Reconciled revenue/cost/margin | Finance methodology |
| Выполнение поставок | Delivery/order line | On-time/in-full, delay reasons | Plan/fact/status rules |
| Эффективность менеджеров | Deal/owner/period | Conversion, cycle, forecast discipline | Assignment/context |
| Plan/fact | Product/season/location/period | Plan, forecast, actual, variance | Scenario/version/unit |
| Качество | Lot/test/complaint | Holds, pass/reject, claims | LIMS/quality source |

```text
CRM + FMIS + ERP/WMS + LIMS + TMS/1C -> Conformed IDs/Units
 -> DQ/Reconciliation -> Governed Metric Layer -> Role-based Dashboard
```

Метрика имеет owner, formula, grain, period/season, unit/conversion, currency, exclusions, source, freshness SLA and control total. Plan, forecast and actual never combine without an explicit dimension.

## 14. Security Architecture

| Контроль | Реализация | Проверка |
|---|---|---|
| Минимизация | Only required client/field/lot projections | Field/file inventory |
| Field/location data | Restricted operational scope | Cross-field negative tests |
| Quality data | Role-based results/evidence refs | Unauthorized result test |
| Документы | Protected DMS refs | Expired/revoked access |
| Аудит | CRM history + masked integration logs | Correlation trace |
| Export | Approval/minimum fields/protected target | Export test |
| Secrets | Environment/secret store/rotation | Scan/revoke drill |
| Retention | Approved archive/delete/hold policy | Periodic review |

**Cloud:** нет server/PHP/DB access; используются штатные rights/history and external logs. **On-Premise:** дополнительно защищаются server, PHP, DB, files, cron, backup and logs; core is not modified.

## 15. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Customer enquiry | Valid synthetic payload | One request, owner/SLA |
| 2 | Duplicate event | Same source ID | No duplicate |
| 3 | Quantity/unit | Valid product and unit | Correct normalized requirement |
| 4 | Unsupported unit | Unknown conversion | Quarantine/error, no silent conversion |
| 5 | Offer availability | Current ERP/WMS projection | Valid offer/source timestamp |
| 6 | Stale stock | Freshness exceeded | Offer blocked/reviewed |
| 7 | Contract gate | Missing authoritative status | Order/delivery transition blocked |
| 8 | Partial delivery | Two delivery events | Correct plan/fact and remaining quantity |
| 9 | Lot allocation | Two synthetic lots | Traceable quantities sum to order line |
| 10 | Quality hold | LIMS hold event | Related unsafe action blocked/flagged |
| 11 | Complaint | Delivered lot relation | Case/lot/order linked, SLA assigned |
| 12 | Supplier late delivery | Threshold breached | Corrective action/escalation once |
| 13 | Rights | Sales user opens restricted field data | Access denied/minimized |
| 14 | Integration timeout | Accepted then timeout | Retry idempotent |
| 15 | Payment mismatch | Synthetic 1С total differs | Finance incident/reconcile |
| 16 | Plan/forecast/actual | Same period, three scenarios | Values not overwritten/mixed |
| 17 | BI units | Mixed source units | Approved conversion/version applied |
| 18 | Migration dry-run | Synthetic source sample | Counts/mapping report without write |
| 19 | Cloud/On-Premise | Capability matrix | Unsupported assumption detected |
| 20 | Rollback | Failed release | Verified previous state restored |

Тесты используют synthetic companies, fields, lots, products, orders and deliveries. Проверяются positive, negative, boundary, concurrency, retry, rights, audit and recovery.

## 16. Deployment

| Этап | Результат | Gate |
|---|---|---|
| Discovery | AS IS, systems, tariff, volumes, seasons, units | Business/security approval |
| Архитектура | TO BE, master matrix, rights, ADR, NFR | Architecture review |
| CRM настройка | Entities, pipeline, roles, automation in test scope | Configuration review |
| Производственный контур | Minimal FMIS/ERP projections and mappings | Source/version/unit tests |
| Интеграции | Contracts/secrets/retry/monitoring | Failure/replay tests |
| Обучение | Role-based synthetic exercises | UAT sign-off |
| Запуск | Window/runbook/support/rollback | Go/no-go |
| Stabilization | Incidents/baseline/season observation | Exit criteria |

```text
Inventory -> Dependency review -> Backup/export -> Test configuration
 -> Synthetic tests -> Pilot season/product flow -> UAT -> Go/No-Go
 -> Release -> Smoke/reconciliation -> Hypercare or Rollback
```

Рабочие поля, стадии, роли, роботы и интеграции не удаляются/переименовываются до dependency analysis, backup/export, pilot and approved rollback.

## 17. Monitoring

| Сигнал | Правило | Действие | Владелец |
|---|---|---|---|
| Просроченная поставка | Plan/fact milestone breach | Escalate/replan | Logistics owner |
| Integration error | Error/retry/DLQ | Incident/replay after fix | Integration support |
| Потерянная заявка | Intake SLA/no ack | Assign/escalate | Sales operations |
| Plan deviation | Approved threshold/scenario | Review/action | Production owner |
| Stale stock/forecast | Freshness SLA | Mark stale/stop unsafe promise | ERP/FMIS owner |
| Quality hold conflict | Allocated lot becomes held | Incident/block/reconcile | Quality owner |
| Unit/data mismatch | Validation/DQ rule | Quarantine/data incident | Data owner |
| BI mismatch | Control total difference | Mark stale/investigate | BI owner |

Alert содержит system, object reference, correlation ID, time, severity, owner and runbook; без реальных координат, персональных данных и секретов.

## 18. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Дубли клиентов | Multiple sources | Fragmented sales | Stable IDs/dedup |
| 2 | False merge | Weak identifiers | Wrong order disclosure | Conservative review |
| 3 | Lost enquiry | No ack | Lost sale | Intake reconciliation |
| 4 | Duplicate order | Retry | Double allocation | Idempotency |
| 5 | Supplier duplicated as client | Role confusion | Wrong rights/report | Contextual company roles |
| 6 | Field identified by name | Mutable value | Wrong production link | Stable field ID |
| 7 | Real coordinates exposed | Broad fields/logs | Security risk | Minimize/restrict/mask |
| 8 | Season overwritten | One current field | Lost history | Season/version entity |
| 9 | Plan equals fact | Flat metric | False decisions | Scenario dimension |
| 10 | Forecast version absent | Update in place | No audit | Immutable forecast version |
| 11 | Stock stale | Cached CRM value | Oversell | Freshness/master check |
| 12 | Production forecast promised as stock | Semantics mixed | Contract failure | State definitions/gate |
| 13 | Product/spec mismatch | Free text | Quality dispute | Versioned SKU/spec |
| 14 | Unit conversion silent | No version/rule | Quantity error | Governed conversions |
| 15 | Lot absent | Product-only model | No traceability | Lot/batch ID |
| 16 | Allocation exceeds lot | No balance check | Impossible delivery | WMS/ERP atomic allocation |
| 17 | Quality result manual | No LIMS sync | Unsafe release | Read-only source result |
| 18 | Hold ignored | CRM stage trusted | Nonconforming shipment | Authoritative hold gate |
| 19 | Sample/result relation lost | Text notes | No evidence | Stable sample/test IDs |
| 20 | Partial delivery overwritten | One delivery field | Wrong plan/fact | Delivery lines/events |
| 21 | Route status stale | No TMS sync | Missed delay | Freshness/milestones |
| 22 | Driver/location data broad | Overcollection | Privacy/security risk | Need-to-know |
| 23 | Supplier score subjective | Comments | Biased procurement | Governed metrics |
| 24 | Contract version stale | Local file | Wrong terms | DMS/ERP ref |
| 25 | Price edited in two systems | No master | Margin dispute | Field ownership |
| 26 | Currency missing | Local assumption | Wrong amount | Currency/date required |
| 27 | Payment manual | Convenience | False financial status | 1С source |
| 28 | Sales sees sensitive production detail | Broad role | Commercial leak | Minimum projection |
| 29 | Agronomist sees customer finance | Shared record | Excess access | Functional scopes |
| 30 | Admin as super-reader | Role confusion | Uncontrolled access | Separate privileges |
| 31 | Export unrestricted | Defaults | Mass leak | Approval/minimum fields |
| 32 | Secret in Git | Real example | Compromise | Secret store/rotation |
| 33 | Broad token | One integration user | Large blast radius | Least privilege |
| 34 | Raw payload logged | Debugging | Data leak | Masking |
| 35 | Infinite retry | No DLQ | Load/outage | Bounded backoff |
| 36 | Out-of-order event | Async ignored | Stale status | Version/order guard |
| 37 | Schema drift | Source changed | Corrupt fields | Contract validation |
| 38 | No correlation ID | Fragmented logs | Slow RCA | End-to-end trace |
| 39 | Bidirectional edits free | No master | Ping-pong | Field ownership |
| 40 | Real farm data in tests | Prod copy | Data leak | Synthetic data |
| 41 | Migration without dry-run | Urgency | Mass corruption | Sample/report/rollback |
| 42 | No reconciliation | HTTP success trusted | Silent loss | Counts/control totals |
| 43 | BI wrong grain | Lots/orders/lines mixed | Double count | Metric contract |
| 44 | Profit from Deal amount | Missing costs | False margin | Finance methodology |
| 45 | Tariff not checked | Assumption | Feature unavailable | Capability assessment |
| 46 | Cloud treated as server | DB/PHP expected | Impossible design | Cloud boundary |
| 47 | Core modified On-Premise | Quick fix | Upgrade/security failure | Extensions only |
| 48 | Seasonal peak untested | Average load | SLA collapse | Capacity/load test |
| 49 | Rollback absent | Success assumed | Long outage | Tested rollback |
| 50 | Alert has no owner | Notification only | Unresolved incident | RACI/runbook |
| 51 | No baseline | AS IS unmeasured | Effect unprovable | Baseline/KPI |
| 52 | No change register | Oral changes | Configuration drift | Versioned register |
| 53 | Weather dependency treated as CRM fact | Scope confusion | Wrong forecast | External source/methodology |
| 54 | Equipment telemetry copied in full | Overintegration | Load/security risk | Aggregate projection |
| 55 | Archive confused with backup | No restore design | Failed recovery | Separate policies |

## 19. ADR

| ADR | Решение | Альтернатива | Обоснование | Последствие |
|---|---|---|---|---|
| ADR-AGR-001 | Product is ERP/PIM projection with specification version | CRM product as sole master | Operational catalog/quality authority | Integration/freshness required |
| ADR-AGR-002 | Delivery is separate order-linked lifecycle | Delivery fields in Deal | Partial shipments/exceptions need history | More relations/status mappings |
| ADR-AGR-003 | Production/field/harvest remain FMIS/ERP master | Full farm management in CRM | Specialist planning and telemetry | Minimal projections only |
| ADR-AGR-004 | Lot allocation links order lines and harvest lots | Product-level shipment only | Traceability and quantities | Allocation grain required |
| ADR-AGR-005 | Rights split by function/field/portfolio | Broad company access | Least privilege | Negative tests/role matrix |
| ADR-AGR-006 | Contracts/documents stored as DMS/ERP references | Authoritative files in CRM | Version/access/retention | Repository integration |
| ADR-AGR-007 | Middleware/contracts govern integrations | Free point-to-point webhooks | Units/idempotency/monitoring | Platform owner required |
| ADR-AGR-008 | BI uses reconciled conformed units and scenarios | Raw CRM reports | Production/sales grains differ | Metric/DQ ownership |

ADR получает owner, status `proposed/accepted/superseded`, date, context, alternatives, consequences, portal/version and evidence.

## 20. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Нет Discovery | Immediate setup | Wrong scope | AS IS/stakeholders |
| 2 | Cloud/On-Premise unknown | Template design | Unsupported solution | Capability check |
| 3 | Tariff not verified | Assumption | Blocked release | Verify portal |
| 4 | CRM declared FMIS/ERP/WMS | Scope confusion | False production facts | Architecture boundary |
| 5 | All requests are Leads | Universal lead | Support mixed with sales | Classification |
| 6 | Leads disabled without ADR | Fashion | Unqualified demand mixed | Lead decision |
| 7 | Contact/Company mixed | Flat client field | Wrong B2B relations | Native entities |
| 8 | Supplier duplicated per category | Shortcut | Fragmented history | Company + supplier role |
| 9 | Name is integration key | No external ID | Wrong match | Stable IDs |
| 10 | Phone is sole dedup key | Convenience | False merge | Conservative matching |
| 11 | One Deal for all seasons/orders | Entity saving | No sales history | Deal per opportunity |
| 12 | Order stored only in Deal fields | No ERP relation | Wrong fulfillment | Order projection |
| 13 | Field stored as Company | Entity misuse | Wrong topology/rights | Field Smart Process/projection |
| 14 | Field name is key | No stable ID | Wrong history | Field ID |
| 15 | Coordinates copied everywhere | Convenience | Security exposure | Minimize/restrict |
| 16 | Season is one text field | Flat model | History lost | Season entity/version |
| 17 | Plan/forecast/fact in one field | Simplification | Overwrite/false KPI | Scenario dimension |
| 18 | Yield calculated in robot | Overautomation | Wrong forecast | FMIS/methodology |
| 19 | Product and specification mixed | No version | Quality dispute | Versioned spec |
| 20 | Lot stored in comment | No traceability | Recall/claim failure | Lot ID/relation |
| 21 | Units are free text | No dictionary | Quantity errors | Governed UOM |
| 22 | Conversion factor unversioned | Manual formula | Inconsistent totals | Versioned conversion |
| 23 | Stock copied without timestamp | Cache | Oversell | Freshness SLA |
| 24 | Forecast sold as available | State confusion | Failed delivery | Explicit availability states |
| 25 | Deal closes before delivery | KPI pressure | False revenue/service | Delivery gate |
| 26 | Stages describe tasks | Process mistake | Ambiguous state | State-based stages |
| 27 | Too many stages | Every action is stage | Low adoption | Tasks/checklists |
| 28 | Fields mandatory too early | Maximum control | Fake data | Stage requiredness |
| 29 | Loss reason in comment | No dictionary | No analytics | Governed codes |
| 30 | No next step | Stage-only control | Stalled Deal | Activity policy |
| 31 | Robot duplicates task | Re-entry ignored | Fake workload | Idempotency marker |
| 32 | Robots loop stages | Mutual transitions | Automation storm | Loop guard |
| 33 | Contract overwritten | No versioning | Wrong terms | Immutable refs |
| 34 | Price edited in CRM and ERP | No master | Margin conflict | Field ownership |
| 35 | Currency absent | Local assumption | Wrong offer | Currency/date |
| 36 | Partial supply overwrites plan | One status | Lost remainder | Delivery events/lines |
| 37 | Delivery has no lot allocation | Product only | No traceability | Order-line/lot relation |
| 38 | Transport stored as comment | No entity/ref | No control | TMS/fleet reference |
| 39 | Route telemetry copied raw | Overintegration | Load/security risk | Aggregates only |
| 40 | LIMS result manually entered | No integration | Unsafe release | Source projection |
| 41 | Quality hold ignored | Sales priority | Nonconforming delivery | Authoritative gate |
| 42 | Complaint lacks lot/order | Weak intake | No RCA | Required relations |
| 43 | Finding closes by task status | Automation shortcut | Issue unresolved | Reviewer/evidence gate |
| 44 | Supplier score from comments | Subjectivity | Bad procurement | Metric contract |
| 45 | Sales sees all field details | Broad role | Commercial leak | Minimum projection |
| 46 | Agronomist sees finance | Shared access | Excess disclosure | Functional scope |
| 47 | Accountant sees production notes | Common rights | Excess disclosure | Finance projection |
| 48 | Admin equals business reader | Role confusion | Uncontrolled access | Separate privileges |
| 49 | Temporary access never expires | Manual grant | Residual risk | Expiry/recertification |
| 50 | Rights only positive-tested | Happy path | Hidden cross-access | Negative tests |
| 51 | Export unrestricted | Defaults | Mass leak | Approval/minimum fields |
| 52 | Secret in README | Real example | Compromise | Placeholder/secret store |
| 53 | Webhook committed | Convenience | Unauthorized calls | Environment/rotation |
| 54 | Token overprivileged | One credential | Large blast radius | Least privilege |
| 55 | Raw payload logged | Debugging | Data leak | Masking |
| 56 | Retry duplicates order | No idempotency | Double allocation | Source event key |
| 57 | Infinite retry | No DLQ | Load/outage | Bounded retry |
| 58 | Out-of-order ignored | Async not designed | Stale status | Version/order guard |
| 59 | Schema not validated | Trust source | Corrupt data | Contract validation |
| 60 | No correlation ID | Fragmented logs | Slow RCA | Trace ID |
| 61 | Bidirectional edits free | No master | Ping-pong | Field ownership |
| 62 | Batch ignores limits | Premature optimization | Partial errors | Official checks/report |
| 63 | Migration has no inventory | Unknown sources | Missing objects | Source/volume report |
| 64 | Import without dry-run | Urgency | Mass corruption | Sample/dry-run |
| 65 | Real field/client data in UAT | Prod copy | Data exposure | Synthetic data |
| 66 | No count/sample reconciliation | HTTP success trusted | Silent loss | Control report |
| 67 | BI counts lots as orders | Wrong grain | Inflated sales | Metric contract |
| 68 | Plan and actual summed | Scenario ignored | False output | Scenario dimension |
| 69 | Profit from Deal amount | Costs/actuals missing | False margin | Finance model |
| 70 | Seasons compared without normalization | Context ignored | False trend | Cohort/unit/context |
| 71 | Old fields removed immediately | Cleanup | Broken dependencies | Deprecation scan |
| 72 | Release without smoke | Time pressure | Production defect | Checklist |
| 73 | Rollback untested | Paper plan | Long outage | Restore drill |
| 74 | Monitoring only HTTP uptime | Technical focus | Deliveries/orders missed | Business signals |
| 75 | Alert includes coordinates/client detail | Full payload | Disclosure | Minimal alert |
| 76 | Alert has no owner | Notification only | Unresolved incident | RACI/runbook |
| 77 | Cloud assumes DB/PHP access | On-Premise habit | Impossible design | Cloud boundary |
| 78 | On-Premise core modified | Quick fix | Upgrade/security failure | Extensions |
| 79 | No hypercare/change register | Immediate closure | Incident/config drift | Stabilization/docs |
| 80 | Done means configured | No evidence | Disputed result | Tested acceptance criteria |

## 21. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Confirmed AS IS/scope | Stakeholders, systems, tariff, seasons, volumes, units | Facts/open questions approved |
| 2. Архитектура | Approved TO BE | Entity/lot model, master matrix, rights, NFR, ADR | Architecture/security review |
| 3. CRM настройка | Governed sales pipeline | Fields, stages, roles, automation register | Synthetic paths pass |
| 4. Производственные процессы | Minimal FMIS/ERP projections | Field/season/lot/spec mappings, scenarios | Source/unit/version tests pass |
| 5. Поставки | Order/allocation/delivery/quality lifecycle | Status mappings, SLA, exceptions, traceability | End-to-end UAT passes |
| 6. Интеграции | Reliable exchange | Contracts, secrets, retry, idempotency, monitoring | Failure/replay/reconcile pass |
| 7. Тестирование | Evidence-based quality | Test plan, synthetic data, rights suite, UAT | Critical defects closed/sign-off |
| 8. Запуск | Controlled release | Cutover, backup/export, rollback, support | Smoke/reconciliation pass |
| 9. Оптимизация | Measured improvement | Baseline, DQ/incident/change backlog | KPI change verified |

### Критерии готовности

- подтверждены Cloud/On-Premise, тариф/редакция, версия, регион и права;
- утверждены master-system matrix, field/season/lot/order grain, units, rights and ADR;
- проверены права ролей позитивными и негативными тестами;
- выполнены synthetic tests, pilot flow, migration dry-run and reconciliation;
- секреты вынесены из Git, logs/alerts не содержат координат и чувствительных данных;
- утверждены release, rollback, monitoring, retention and support procedures;
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
2. Business lines, crop/season model, sales channels, lead usage and process owners.
3. Master field, season, forecast, harvest lot, product/spec, stock, quality, order and delivery.
4. FMIS/GIS/ERP/WMS/LIMS/TMS/1С APIs, stable IDs, event order, volumes, SLA and replay.
5. Units/conversions, currencies, calendars/timezones, lot allocation and traceability rules.
6. Допустимый состав координат, telemetry, client/employee data, retention/export and audit policies.
7. Availability states, sales promise, quality hold/release, partial delivery and complaint gates.
8. Supplier governance, contract versions, quality metrics and corrective-action process.
9. BI grain, plan/forecast/actual scenarios, owners, control totals and DQ thresholds.
10. Test portal/pilot product-season flow, release window, rollback and support RACI.

# Related Documents

- [Case Library](../README.md)
- [Architecture Patterns](../PATTERNS.md)
- [Discovery Template](../DISCOVERY_TEMPLATE.md)
- [Architecture Template](../ARCHITECTURE_TEMPLATE.md)
