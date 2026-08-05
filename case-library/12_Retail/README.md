# Retail Omnichannel CRM Architecture Case

Архитектурный кейс для синтетической розничной сети, объединяющей магазины, интернет-продажи, программу лояльности, маркетинг и клиентский сервис. Документ задаёт референсную CRM-модель, а не готовую конфигурацию для копирования.

> **Граница безопасности:** реальные компании, покупатели, сотрудники, адреса магазинов, телефоны, email, чеки, платёжные данные, CRM ID, webhook URL, токены и выгрузки не используются. Все примеры синтетические.

> **Архитектурная граница:** Bitrix24 отвечает за CRM-профиль, допустимые коммуникации, обращения, сервис и минимальную проекцию продаж. POS/кассовая система и ERP/1С остаются master чеков и финансовых фактов; PIM/ERP — товара и цены; WMS/ERP — остатков; e-commerce platform — онлайн-заказа; loyalty engine — бонусов, уровней и правил программы.

Обозначения:

- **факт Bitrix24** — подтверждается официальным источником из `knowledge/SOURCES.md`;
- **решение архитектора** — целевой выбор кейса, утверждаемый на Discovery;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа/редакции, версии, прав, региона, приложения или внешней системы.

## 1. Business Context

### Модель розничного бизнеса

Референсный сценарий — сеть физических магазинов и интернет-магазин с единым брендом. Покупатель может начать взаимодействие в рекламе или онлайн, получить консультацию в магазине, купить через POS или сайт, оформить доставку, обратиться за сервисом и вернуться повторно.

| Аспект | Целевая интерпретация |
|---|---|
| Каналы продаж | Магазин/POS, сайт, contact center, разрешённые сообщения, партнёрские каналы |
| Офлайн-взаимодействие | Консультация, чек, возврат и сервис в точке |
| Онлайн-взаимодействие | Заявка, заказ, оплата, доставка и поддержка |
| Клиентская идентичность | CRM/MDM/loyalty profile linked by stable external IDs |
| Продажа | Чек/заказ в POS/e-commerce/ERP; CRM stores allowed projection |
| Финансы | POS/ERP/1С/payment provider |

### Цели CRM

1. Создать допустимый единый профиль взаимодействий покупателя.
2. Связать магазины и онлайн-канал без двойного учёта продаж.
3. Не терять заявки, обращения, возвраты и сервисные задачи.
4. Управлять разрешёнными повторными коммуникациями и лояльностью.
5. Разделить права магазина, региона, маркетинга и поддержки.
6. Получать сверяемые omnichannel-показатели.

```text
Advertising -> Enquiry -> Consultation -> POS/E-commerce sale
 -> Delivery/Pickup -> Product use -> Service/Return -> Repeat purchase
```

## 2. AS IS Problems

| Проблема AS IS | Проявление | Корневая причина | Целевой контроль |
|---|---|---|---|
| Нет единого профиля | Офлайн/онлайн создают дубли | Нет identity/master policy | Conservative identity resolution |
| Разрыв магазинов и online | Каналы видят разные статусы | Нет cross-system IDs/contracts | Omnichannel relations |
| Нет истории покупок | Чеки не связаны с клиентом | Нет stable customer key | POS/loyalty integration |
| Слабая аналитика маркетинга | Источники и покупки не сопоставлены | Нет attribution/metric dictionary | Governed datasets |
| Теряются обращения | Нет owner и SLA | Каналы не объединены | Intake/routing/escalation |
| Продажи дублируются | POS и сайт оба создают CRM Deal | Нет business event semantics | Source sale ID/idempotency |
| Возвраты хранятся комментариями | Нет lifecycle и строк | Нет Return entity | Structured return process |
| Бонусы меняют вручную | CRM подменяет loyalty engine | Нет field ownership | Read-only loyalty projection |
| Остатки устаревают | Продавец обещает недоступный товар | CRM cache used as WMS | Freshness/master check |
| Доступы магазинов слишком широки | Общая роль | Нет store/region scope | Segmented rights model |

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Лид | Неквалифицированный запрос, если lead-based process подтверждён | Bitrix24 |
| CRM | Контакт | Минимальный профиль физлица | CRM/MDM/loyalty system by ADR |
| CRM | Компания | Корпоративный покупатель/партнёр | ERP/MDM/Bitrix24 |
| CRM | Сделка | Консультация/коммерческая возможность или sale projection | Bitrix24 |
| Service | Обращение | Запрос покупателя | Bitrix24 |
| Identity | Smart Process «Клиент» | Дополнительный retail-profile if Contact insufficient | CRM/MDM by ADR |
| Organization | Smart Process «Магазин» | Store hierarchy and CRM operational context | ERP/MDM |
| Loyalty | Smart Process «Программа лояльности» | Membership/status projection | Loyalty engine |
| Service | Smart Process «Возврат» | Return request/status coordination | POS/OMS/ERP master by channel |
| Service | Smart Process «Сервисное обращение» | SLA and resolution lifecycle | Bitrix24 |
| Marketing | Smart Process «Маркетинговая кампания» | Campaign governance/results reference | Marketing/BI system |

**Факт Bitrix24:** CRM, Smart Processes, роботы, триггеры и бизнес-процессы описаны официальными материалами. Конкретные поля, связи, права, automation and availability проверяются на целевом портале.

**Решение архитектора:** CRM получает only minimal customer, store, purchase, loyalty and service projection. POS/e-commerce/ERP events retain stable IDs and source semantics; CRM does not recalculate fiscal, stock or loyalty facts.

```text
Lead/Request -> Contact/Company -> Deal
                    |               |
                    v               v
                 Customer <---- Sale projection <---- POS/E-commerce
                    |                    |                 |
                    v                    v                 v
                 Loyalty             Return/Service     ERP/WMS/PIM
                    |                    |                 |
                    +--------> Marketing/BI <-------------+
                                  |
                                Store
```

### Правила целостности

- customer identity uses stable `external_customer_id`; weak fields do not trigger unreviewed merge;
- each offline sale has `external_receipt_id`, online sale has `external_order_id`;
- return is linked to sale/order lines and cannot exceed allowed quantity/amount;
- loyalty balance/tier is authoritative only in loyalty engine;
- stock and price are authoritative in ERP/PIM/WMS/POS as assigned;
- repeated event does not create duplicate sale, return, points event or message;
- bulk operations use dry-run, sample, report, reconciliation and rollback.

## 4. Omnichannel Customer Journey

| Этап | Цель | Данные | Контроль |
|---|---|---|---|
| Реклама | Create awareness | Campaign/channel metadata | Approved attribution and communication policy |
| Заявка | Register explicit request | Minimal contact, channel, topic, owner | Validation/dedup |
| Консультация | Help select product | Request, product/category refs, store/channel | No false stock/price promise |
| Покупка | Receive authoritative sale event | Receipt/order ID, lines, totals, store/channel | POS/e-commerce master |
| Доставка | Track fulfillment if applicable | Shipment ID/status/plan/fact | OMS/WMS/carrier source |
| Использование продукта | No automatic personal inference | Product relation and warranty context if needed | Data minimization |
| Сервис | Resolve request/return/claim | Case, sale, line, SLA, owner | Controlled lifecycle |
| Повторная покупка | New sale for existing customer | Stable identity/new receipt/order | No contact duplication |

```text
Реклама -> Заявка -> Консультация -> Покупка
 -> Доставка -> Использование продукта -> Сервис -> Повторная покупка

Channel switches preserve customer/request/sale correlation IDs.
```

## 5. Retail Data Model

### Основные сущности

| Сущность | Минимальные поля | Контроль качества |
|---|---|---|
| Клиент | External ID, permitted contacts, segment, communication preferences, aggregate purchase history | No payment data/free subjective profile |
| Магазин | Store ID, synthetic location/region, status, manager, organizational relation | Address not used as sole key |
| Сотрудник | User/employee ID, store/region/role relation | HR system governs employment status |
| Продажа | Receipt/order ID, source/channel, store, lines, amount, currency, timestamp, responsible | POS/e-commerce/ERP fact |
| Товар | Product/SKU/variant IDs, category, attributes | PIM/ERP master |
| Возврат | Return ID, source sale/line, quantity, amount, reason, status | POS/OMS/ERP fact/reconcile |
| Loyalty membership | Membership ID, tier/status, validity, balance summary if required | Loyalty engine read-only projection |
| Service case | Case ID, customer/sale/item, category, priority, SLA, owner, result | Outcome and cause structured |

```text
Customer 1 ---- N Sale 1 ---- N Sale Line N ---- 1 Product/SKU
   |               |  |                          |
   |               |  +---- N Return Line       v
   |               +------- 1 Store/Channel    Warranty/Service
   +---- 0..1 Loyalty
   +---- N Service Case / Campaign Interaction
```

### Кардинальность и grains

| Отношение | Кардинальность | Правило |
|---|---:|---|
| Customer → Sale | 1:N | Repeat purchase is a new sale |
| Store → Sale | 1:N | Online channel represented separately or by virtual channel rule |
| Sale → Sale Line | 1:N | Analytics grain defined explicitly |
| Sale Line → Return Line | 1:N | Returned quantity aggregated/validated |
| Customer → Loyalty | 1:0..N by program | Program/version IDs required |
| Customer → Service Case | 1:N | Case does not overwrite sale |

## 6. Sales Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новый клиент | Register request/profile without duplicate | Channel, minimal contact, owner | Dedup/routing |
| Консультация | Confirm need and channel/store | Product category, constraints, next step | SLA/task |
| Подбор товара | Use current catalog/stock context | Product/SKU, store/warehouse, timestamp | PIM/WMS query/reference |
| Предложение | Present valid terms | SKU, price/version, validity, channel | Follow-up/expiry |
| Оформление покупки | Receive checkout/POS intent | Order/receipt context and source | Idempotent source event |
| Оплата | Receive financial fact | Payment/sale ID, amount, currency, status | Provider/POS/ERP reconcile |
| Доставка | Observe fulfillment | Shipment ID, plan/fact, status | Delay/exception route |
| Завершение | Confirm sale/service completion | Delivery/pickup/receipt fact, open cases | Completeness check |
| Повторная продажа | Start allowed next interaction | Eligibility, segment, channel, product context | Consent/suppression/frequency controls |

```text
Новый клиент -> Консультация -> Подбор товара -> Предложение
 -> Оформление покупки -> Оплата -> Доставка -> Завершение
 -> Повторная продажа

Branches: unavailable / abandoned / cancelled / return / service case
```

POS sale completed without prior consultation may enter CRM as a historical/analytical projection rather than traversing artificial stages. Pipeline semantics are defined per channel.

## 7. Store Management

| Компонент | Назначение | Master | CRM-проекция |
|---|---|---|---|
| Сеть/регион | Organization hierarchy | ERP/HR/MDM | Region/store IDs and owners |
| Магазин | Physical sales point | ERP/MDM | Status, manager, synthetic location |
| Продавец | Employee/store assignment | HR/ERP | User relation and allowed store scope |
| Показатель точки | Sales/service metrics | POS/ERP/BI | Aggregates, not manual fields |
| Задача | Specific improvement/service action | Bitrix24/operations system | Owner, deadline, result |

```text
Company -> Region -> Store -> Shift/Employee
                         |
                         +-> POS sales
                         +-> Service cases/tasks
                         +-> KPI/quality review
```

Магазин не создаётся как Company-клиент. Store, legal entity and warehouse are distinct concepts linked by stable IDs.

## 8. Loyalty Program Architecture

| Компонент | Master | CRM-проекция | Контроль |
|---|---|---|---|
| Membership | Loyalty engine | Membership ID/status/program | Stable ID/read-only |
| Уровень клиента | Loyalty engine | Tier/version/validity | No manual override |
| Бонусы | Loyalty engine | Balance summary or event reference if needed | Reconcile/no secrets |
| Сегментация | CRM/BI/loyalty by governance | Approved segment IDs | Owner/freshness |
| Персональное предложение | Marketing system/CRM process | Offer/campaign/version | Eligibility and suppression |
| Коммуникация | Approved channel owner | Message event/result | Consent, frequency, delivery |

```text
Sale event -> Loyalty engine -> Earn/Spend/Adjust decision
 -> Membership status projection -> Eligible segment -> Communication
```

CRM does not compute or change points if loyalty engine is master. Manual compensation requires approved external adjustment process and auditable reference.

## 9. Marketing Automation

| Практика | Назначение | Обязательные контроли |
|---|---|---|
| Рассылки | Campaign communication | Eligibility, suppression, frequency, channel owner |
| Сегменты | Group customers by governed criteria | Stable identity, refresh, data minimization |
| Акции | Time-bound offer | Campaign/offer version, stores/channels, validity |
| Триггерные коммуникации | Respond to confirmed business event | Event ID, consent policy, idempotency |
| Повторные продажи | Next-best operational campaign by approved rules | No sensitive inference, holdout/measurement |

Abandoned cart, product usage and purchase history do not automatically grant communication permission. Transactional and marketing messages are classified separately.

## 10. Customer Service Architecture

| Тип | Назначение | Ключевые данные | Exit criteria |
|---|---|---|---|
| Обращение | Question/action request | Customer, sale/item, category, priority, SLA | Resolved/confirmed result |
| Warranty case | Service under approved warranty context | Sale line, product, dates, service route | Authorized service result |
| Возврат | Reverse goods/finance process | Sale line, quantity, reason, status | Goods and refund facts reconciled |
| Претензия | Formal dissatisfaction/impact | Severity, evidence ref, owner, decision | Resolution communicated and recorded |
| Service task | Concrete work | Store/service center, assignee, deadline, outcome | Completed with result |

```text
Inbound -> Identify customer/sale/item -> Classify -> Assign
 -> Resolve/Return/Repair -> Reconcile -> Communicate -> Close
                     |
                     +-> Escalate / Claim / RCA
```

SLA defines start, calendars, permitted pauses, priority, response/resolution targets and escalation. Implementation requires target portal verification.

## 11. Rights Model

| Роль | Клиенты | Продажи | Магазины | Маркетинг | Аналитика |
|---|---|---|---|---|---|
| Продавец | Assigned/current interaction minimum | Own/store sales projection | Own store | No bulk campaign access | Own/store minimum |
| Руководитель магазина | Store customers by need | Store sales/issues | Own store/team/tasks | Local approved activities | Store KPI |
| Маркетолог | Segments/minimized audience | Aggregated purchase context | Allowed network scope | Campaigns/segments | Marketing/channel KPI |
| Оператор поддержки | Queue/assigned customer | Sale/item context | Relevant store references | No campaign administration | Service SLA |
| Региональный менеджер | Region scope | Regional sales/service | Region stores | Approved regional view | Regional KPI |
| Директор | Management scope | Aggregated network | Network overview | Management view | Executive KPI |
| Администратор | Technical minimum | Configuration, not business decisions | Technical mapping | Technical configuration | Diagnostics |

Точная реализация CRM-ролей, Smart Process rights, departments, groups, inheritance and export requires target portal verification. Store/region isolation must be tested via UI, search, direct URLs, REST and export.

Controls: least privilege, service-account minimum scope, separate admin account, offboarding, temporary staff expiry, store transfer process and periodic access review.

## 12. Automation Architecture

| Механизм | Сценарий | Защита | Ошибка/эскалация |
|---|---|---|---|
| Робот/задача | First response | One marker/request | Fallback owner |
| Робот | Service SLA | Calendar/priority/pause | Tiered escalation |
| Задача | Store/service action | Sale/item/store IDs, outcome | Reassign/escalate |
| Уведомление | Transactional event | Trusted sale/order status | Delivery result/no duplicate |
| Робот | Repeat-sale eligibility | Confirmed event, suppression | No unauthorized communication |
| Business process | Claim/compensation approval | Version/authority/limit | Reject/timeout route |
| Robot/integration | Loyalty projection | Trusted membership event | Unknown state → exception |

```text
Event -> Authenticate -> Validate source/version -> Eligibility/Authority
 -> Execute -> Record result -> Bounded retry -> Escalate/Reconcile
```

Each automation defines owner, trigger, preconditions, data classification, idempotency key, timeout, retry budget, error route, audit trail and test case. Robot cannot invent sale, refund, stock or bonus facts.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| POS/кассы | Receipt/sale/return lines, store, seller, totals | Event/API/batch contract through controlled layer | IDs, signatures/auth, ordering, reconcile |
| ERP/1С | Customers by scope, products, prices, finance and accounting facts | Verified connector/application or contract integration | Mapping, queue, retry, totals |
| PIM | Product/SKU/category/attributes | API/controlled batch | Version, inactive SKU, mapping |
| Internet store/OMS | Online order, payment/delivery context | REST/OAuth/application/event | Idempotency, status mapping, reconcile |
| WMS | Stock/reservation/fulfillment | API/event/batch | Location, unit, timestamp, freshness |
| Payment provider | Payment/refund status | Provider API/event | Authenticity, idempotency, amount reconcile |
| Телефония | Call event and CRM relation | Official Bitrix24 telephony/verified provider | Rights, unmatched calls, recording policy |
| BI/DWH | Approved datasets | BI Constructor/controlled export | Access, identity, freshness, metric reconcile |

```text
Source -> Authenticate -> Minimize -> Validate -> Map -> Idempotency -> Target
  |                                                                  |
  +-> source audit                                       target result
                 \                                        /
                  +------- retry / reconciliation --------+
```

Contract records source of truth, direction, business/event keys, schema/version, store/channel, units, currency, timezone, create/update/return semantics, ordering, timeout, retry budget, security, monitoring, recovery and owner.

No ready integration with a particular POS, loyalty engine, WMS or 1C configuration is assumed. Marketplace apps and external APIs are assessed separately.

## 14. Analytics

| Отчёт | Метрика | Grain | Источник сверки |
|---|---|---|---|
| Продажи по магазинам | Revenue/units/returns | Store-sale-line-period | POS/ERP |
| Эффективность продавцов | Sales/service outcomes by approved attribution | Seller-period | POS/HR/store assignment |
| Средний чек | Eligible net revenue / sales | Sale cohort | POS/ERP, defined returns rule |
| Повторные покупки | Customers with repeated eligible sales | Customer cohort | Stable identity/POS/OMS |
| Эффективность маркетинга | Incremental/attributed sales and cost-aware return | Campaign/cohort | Marketing/POS/OMS, governed attribution |
| Клиентская ценность | Approved value metric over period | Customer cohort | Finance/BI; formula documented |
| Лояльность | Active members/tier movement/earn-spend activity | Membership-period | Loyalty engine |
| Сервис | SLA, returns, warranty, claims | Case/sale-line-period | CRM/POS/service systems |

```text
CRM + POS + ERP/1C + OMS/WMS + Loyalty + Marketing
 -> Data contracts -> Identity/quality gate -> Dataset/DWH
 -> Metric dictionary -> Dashboard -> Decision
                         ^                |
                         +-- reconcile ---+
```

BI Constructor, datasets and fields depend on target portal. Metric dictionary includes formula, grain, cohort, store/channel, eligible statuses, timezone, currency, attribution, owner, freshness and reconciliation source.

## 15. Security Architecture

| Принцип | Реализация | Проверка |
|---|---|---|
| Store/region isolation | Scope by organizational assignment and approved roles | Cross-store role tests |
| Data minimization | Field allowlist for POS/OMS/loyalty flows | Data inventory review |
| Least privilege | Entity/field/export/API minimum | UI/URL/REST/export tests |
| Admin separation | Dedicated technical accounts | Access and activity review |
| Secret management | Tokens/webhooks outside Git/notes/logs | Secret scan/rotation process |
| Auditability | Source events, versions, decisions traceable | Audit sample/RCA |
| Retention | Approved period by data class | Scheduled review/process |
| Safe analytics | Aggregated/minimized datasets | Dataset review |

```text
Collect minimum -> Classify -> Authorize by store/role
 -> Process for stated purpose -> Audit safely -> Retain/restrict by policy
```

No legal guarantee is made. Privacy, fiscal, payment, loyalty, recording, retention and marketing requirements are determined by authorized Legal, Finance, Security and Data owners.

## 16. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Offline POS sale | Receipt/store/lines/totals | One authoritative sale projection |
| 2 | Online order | Order/customer/channel relation | One order/deal projection |
| 3 | Repeated sale event | Idempotency | No duplicate sale/points/message |
| 4 | Unknown SKU | Mapping exception | Sale preserved, alert created |
| 5 | Existing offline+online customer | Identity resolution | One reviewed customer profile |
| 6 | Ambiguous identity | Conservative matching | No automatic wrong merge |
| 7 | Price/stock lookup | Store/location/timestamp | Authoritative current result |
| 8 | Payment event | Authenticity/amount/currency | Correct reconciled state |
| 9 | Return | Sale line/quantity/amount/reason | Controlled return lifecycle |
| 10 | Repeated refund event | Refund ID/idempotency | No duplicate finance effect |
| 11 | Loyalty earn/spend | Source event/membership/balance | Read-only projection reconciled |
| 12 | Trigger communication | Eligibility/suppression/message key | One permitted message |
| 13 | Service case | Sale/item/store/SLA/owner | Correct routing/outcome |
| 14 | Store transfer employee | Old/new scope | Access changes correctly |
| 15 | Cross-store rights | UI/URL/search/REST/export | No unauthorized customer/sale access |
| 16 | Integration failure | Timeout/retry/dead event | No loss/duplicate, alert traceable |
| 17 | Out-of-order events | Sequence/version | State does not roll back |
| 18 | Migration | Dry-run/identity sample/totals | Mapping and dedup accepted |
| 19 | BI reconciliation | POS/CRM/ERP sample | Delta explained or corrected |
| 20 | Rollback | Trigger/runbook/reconcile | Safe prior state restored |

Only synthetic data are used. UAT sign-off by Store Operations, E-commerce, Marketing/Loyalty, Support, Finance, Security, Data and Integration owners. Negative, repeat, concurrency, role and recovery cases are mandatory.

## 17. Deployment

| Этап | Действия | Gate |
|---|---|---|
| Discovery | AS IS, stores/channels, systems, volumes, identity and owners | Inventory/scope signed |
| Architecture | Data model, masters, state mappings, rights, ADR | TO BE approved |
| CRM configuration | Test portal/funnel, fields, stages, roles | Peer/security review and smoke |
| Store integrations | POS/ERP/PIM/OMS/WMS contracts and recovery | End-to-end accepted |
| Customer migration | Minimize, normalize, dry-run, identity sample, reconcile | Data owner acceptance |
| Training | Role/store/region/shift scenarios | UAT/readiness sign-off |
| Launch | Freeze, backup/export, cutover, smoke | Go/no-go |
| Hypercare | Monitoring, incident route, daily reconcile | No critical defects, support accepted |

```text
Discovery -> Identity design -> Test configuration -> Store integration pilot
 -> Sample migration -> UAT -> Go/No-Go -> Rollout waves -> Hypercare
                                  |
                                  +-> Rollback by approved trigger
```

Cloud and On-Premise planned separately. On-Premise additionally requires product/modules, PHP, database, files, cron, backup/restore, infrastructure and security verification. Core product files are not modified.

## 18. Monitoring

| Контроль | Сигнал | Owner | Реакция |
|---|---|---|---|
| Необработанные обращения | No owner/first action | Support/store lead | Assign/RCA |
| Ошибка интеграции | Error/retry/dead event | Integration owner | Recover/reconcile |
| Потерянная продажа | POS/OMS sale missing in target reconcile | Data/integration owner | Replay/source correction |
| Просроченная задача | SLA/deadline breached | Functional owner | Reassign/escalate |
| Качество сервиса | Return/claim/repeat contact threshold | Service/region owner | Containment/RCA/action |
| Loyalty mismatch | CRM projection != loyalty master | Loyalty/data owner | Reconcile, no manual overwrite |
| Duplicate customer/sale | Identity/business key alert | Data owner | Contain/review rule |
| Store access anomaly | Cross-store/export signal | Security owner | Contain/audit/revoke |
| Data freshness | POS/WMS/loyalty sync stale | Integration owner | Mark stale/incident |

SLI: success/error rate, latency, backlog age, retry count, last successful sync, stale age and reconciliation delta. Logs contain no secrets, payment payloads, full receipts or unnecessary personal data.

## 19. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Bitrix24 used as POS/ERP | Wrong boundary | Fiscal/finance errors | POS/ERP master |
| 2 | Duplicate customers | Offline/online identities separate | Fragmented history | Conservative resolution |
| 3 | Wrong customer merge | Weak key/auto merge | Other person's purchases/communications | Human review |
| 4 | Phone/email sole key | Shared/changed value | Wrong identity | Approved multi-attribute match |
| 5 | Duplicate sale | POS/OMS repeated event | Double revenue/loyalty | Idempotency/source sale ID |
| 6 | Receipt/order concepts mixed | One generic ID | Collisions/wrong channel metrics | Typed external keys |
| 7 | Product name used as key | No SKU/variant ID | Wrong mapping | Product/SKU ID |
| 8 | Variant lost | Product-level mapping | Wrong return/service | Variant ID |
| 9 | Price outdated | No version/freshness | Customer dispute | Price master/timestamp |
| 10 | Stock outdated | CRM cache | Lost sale/oversell | WMS/ERP master/freshness |
| 11 | Store/warehouse mixed | Weak location model | Wrong availability | Separate IDs |
| 12 | Sale stage used as fiscal fact | CRM semantics | Overstated revenue | POS/ERP confirmation |
| 13 | Payment marked manually | No provider integration | False paid state | Provider/POS fact |
| 14 | Repeated callback duplicates finance effect | No event ID | Financial error | Idempotency |
| 15 | Payment data stored in CRM | Excess payload | Security risk | Token/reference only |
| 16 | Return not linked to sale line | Order-level only | Wrong quantity/amount | Line relation |
| 17 | Returned quantity exceeds sale | No aggregate validation | Loss | Quantity control |
| 18 | Return request equals refund fact | Process mixed | Wrong reporting | Separate lifecycle |
| 19 | Loyalty balance edited in CRM | Wrong master | Incorrect benefits | Loyalty engine read-only |
| 20 | Repeated sale awards points twice | No idempotency | Loyalty loss | Event key/reconcile |
| 21 | Tier outdated | Sync lag | Wrong offer | Freshness/loyalty master |
| 22 | Compensation bypasses approval | Manual adjustment | Fraud/loss | Authority process |
| 23 | Marketing ignores suppression | Systems disconnected | Unwanted messages | Suppression master |
| 24 | Trigger sends duplicate message | Retry/multiple systems | Spam | Message ownership/key |
| 25 | Segment stale | No owner/refresh | Wrong audience | Segment governance |
| 26 | Sensitive preference inferred | Over-personalization | Privacy/discrimination risk | Minimal approved features |
| 27 | Attribution overwritten | No policy/version | Disputed effectiveness | Governed attribution |
| 28 | Store employee sees whole network | Broad role | Data leak | Store/region scope |
| 29 | Marketing sees receipts/customer rows | Convenience | Excess access | Aggregated/minimized data |
| 30 | Support sees payment details | Copied role | Excess access | Case-only finance reference |
| 31 | Admin used as seller | Convenience | Uncontrolled changes | Separate accounts |
| 32 | Service account is administrator | Fast integration | Large blast radius | Least scope |
| 33 | Transferred employee keeps old store access | No mobility process | Cross-store leak | Role reassignment review |
| 34 | Temporary staff access remains | No expiry | Data leak | Time-bound access |
| 35 | Webhook URL published | Secret misclassification | Compromise | Secret store/rotation |
| 36 | POS/payment payload logged | Full debug | Sensitive leak | Allowlist/masking |
| 37 | Error alert includes customer data | No sanitization | Notification leak | Sanitized errors |
| 38 | Infinite retry | No budget | Load/backlog | Bounded backoff |
| 39 | Retry non-idempotent | No key | Duplicate sales/messages | Event ID |
| 40 | Events out of order | Async | State rollback | Sequence/version |
| 41 | Full import without dry-run | Rush | Mass corruption | Sample/report/rollback |
| 42 | Identity matching not sampled | Totals only | Wrong merges | Identity QA sample |
| 43 | No reconciliation | HTTP success assumed | Missing sales/returns | Totals/sample/delta |
| 44 | Backup contains unnecessary customer data | No classification | Exposure | Classified backup |
| 45 | Backup not restore-tested | Formal control | Rollback fails | Restore drill |
| 46 | Portal tariff lacks required rights | Capability check missed | Store isolation infeasible | Target verification |
| 47 | Marketplace app not assessed | Trust listing | Vendor/security risk | Review/exit plan |
| 48 | On-Premise outdated | Old modules/PHP | Errors/vulnerabilities | Supported update plan |
| 49 | Average check includes returns/cancels inconsistently | No definition | Wrong KPI | Metric dictionary |
| 50 | Repeat rate uses email only | Weak identity | False result | Stable customer ID |
| 51 | LTV uses gross sales without policy | Formula gap | Misleading value | Governed formula/finance source |
| 52 | Dashboard has no freshness | Stale data | Wrong decision | Last refresh/SLA |
| 53 | No integration owner | Project ended | Errors unhandled | Support RACI |
| 54 | No concurrency/peak test | Only normal load | Holiday failure | Peak scenarios |
| 55 | No hypercare/wave control | Big-bang rollout | Network-wide incidents | Pilot/waves/support window |

## 20. ADR

### ADR-RTL-001: Клиентская модель

- **Контекст:** offline/online/POS/loyalty identities differ.
- **Решение:** Contact is minimal CRM profile linked to stable external IDs; optional Customer Smart Process only for justified independent lifecycle.
- **Альтернативы:** duplicate profile per channel; loyalty system only.
- **Последствия:** conservative matching, review queue and data minimization.
- **Статус:** Proposed.

### ADR-RTL-002: Омниканальная продажа

- **Контекст:** POS receipt and online order have different lifecycles.
- **Решение:** preserve typed source sale IDs and channel semantics; CRM projection is not fiscal master.
- **Альтернативы:** one generic Deal flow for every transaction.
- **Последствия:** channel-specific mapping, idempotency and reconciliation.
- **Статус:** Proposed.

### ADR-RTL-003: Модель магазина

- **Контекст:** store differs from customer company, warehouse and legal entity.
- **Решение:** Store Smart Process/reference linked to ERP/MDM ID and organizational hierarchy.
- **Альтернативы:** Company for every store; free-text field.
- **Последствия:** region/store rights, transfers and KPI grain.
- **Статус:** Proposed.

### ADR-RTL-004: Лояльность

- **Контекст:** points/tier require atomic rules and audit.
- **Решение:** loyalty engine remains master; Bitrix24 stores membership/status projection and campaign eligibility.
- **Альтернативы:** CRM user fields as balance.
- **Последствия:** event IDs, read-only fields, reconciliation.
- **Статус:** Proposed.

### ADR-RTL-005: Маркетинг

- **Контекст:** campaigns need identity, consent/suppression and attribution.
- **Решение:** governed segment/campaign versions, communication ownership and measured outcomes.
- **Альтернативы:** ad-hoc lists/manual exports.
- **Последствия:** eligibility, frequency, dataset access and attribution policy.
- **Статус:** Proposed.

### ADR-RTL-006: Модель прав

- **Контекст:** seller/store/region/support/marketing scopes differ.
- **Решение:** store/region role-based least privilege with cross-store UI/URL/REST/export tests.
- **Альтернативы:** network-wide common role.
- **Последствия:** transfer/offboarding process and periodic access review.
- **Статус:** Proposed.

### ADR-RTL-007: Аналитическая архитектура

- **Контекст:** sales, returns, loyalty and marketing facts span systems.
- **Решение:** governed datasets linked by stable IDs with identity and metric layers.
- **Альтернативы:** only CRM reports; manual spreadsheets.
- **Последствия:** grain, attribution, freshness and reconciliation.
- **Статус:** Proposed.

### ADR-RTL-008: Лиды или Sale-First

- **Контекст:** completed POS sale is already a business fact; consultation may be unqualified.
- **Решение:** lead only for unqualified enquiry; completed sale enters historical/projection flow directly.
- **Альтернативы:** lead for every receipt; Deal pipeline for all POS events.
- **Последствия:** per-channel lifecycle and analytics rules.
- **Статус:** Open until Discovery.

## 21. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | CRM declared POS/ERP | Wrong scope | Fiscal/finance errors | POS/ERP master |
| 2 | One customer per channel | No identity model | Duplicates | Stable external IDs |
| 3 | Phone as sole key | Shared/changed value | Wrong merge | Conservative resolution |
| 4 | Email as sole key | Shared address | Wrong identity | Approved matching |
| 5 | Auto-merge without review | Cleanup pressure | Mixed purchase histories | Review queue |
| 6 | Real customers used in tests | No synthetic data | Privacy risk | Synthetic-only QA |
| 7 | One Deal for all purchases | Customer/process mixed | Wrong repeat analytics | Sale/deal unit |
| 8 | Lead created per receipt | Template automation | CRM noise | Sale-first projection |
| 9 | Receipt and online order use same untyped key | Weak integration | Collisions | Typed IDs |
| 10 | Repeat POS event creates Deal | No idempotency | Double revenue | Source event ID |
| 11 | Product name is key | No SKU | Wrong mapping | Product/SKU ID |
| 12 | Variant not represented | Product-level model | Wrong service/return | Variant ID |
| 13 | Category stored as free text | No PIM mapping | Bad analytics | Controlled hierarchy |
| 14 | Current price overwrites sale history | No snapshot | Wrong receipts/report | Sale-line snapshot |
| 15 | Amount lacks currency | Incomplete model | Wrong KPI | Amount + currency |
| 16 | Discount lacks source/rule | Manual edit | Margin leakage | Discount metadata/approval |
| 17 | CRM stock treated as master | No WMS boundary | Lost sale/oversell | WMS/ERP master |
| 18 | Stock has no store/warehouse context | One quantity | Wrong availability | Location ID |
| 19 | Stock has no timestamp | No freshness | Stale promise | Freshness field/SLA |
| 20 | Store and warehouse are same entity | Simplification | Wrong logistics/KPI | Separate concepts |
| 21 | Store created as customer Company | Misused CRM entity | Polluted client base | Store entity/reference |
| 22 | Address used as store key | Mutable/nonunique | Wrong mapping | Store ID |
| 23 | Employee store assignment copied manually | No HR sync/process | Wrong access/KPI | Assignment master/workflow |
| 24 | Seller keeps old store access after transfer | No mobility control | Data leak | Access review |
| 25 | Payment marked manually | No source sync | False paid state | POS/provider fact |
| 26 | Payment payload stored in CRM | Excess integration | Security risk | Token/reference only |
| 27 | Repeat callback duplicates action | No event key | Financial/CRM duplicate | Idempotency |
| 28 | Return stored in comment | No entity | No SLA/analytics | Return lifecycle |
| 29 | Return not linked to line | Order-level only | Wrong quantity | Sale-line relation |
| 30 | Return quantity not aggregated | No validation | Excess refund | Quantity control |
| 31 | Return request equals refund fact | Processes mixed | Wrong finance report | Separate statuses |
| 32 | Reason is only free text | No taxonomy | No product insight | Controlled reason + note |
| 33 | Loyalty balance is CRM field | Wrong master | Wrong benefits | Loyalty engine projection |
| 34 | Points event lacks sale ID | No relation | No reconciliation | Source sale/event ID |
| 35 | Repeated sale awards points twice | No idempotency | Loyalty loss | Event key |
| 36 | Tier edited by seller | Broad rights | Unauthorized benefit | Read-only/approval |
| 37 | Segment has no owner/freshness | One-time setup | Wrong audience | Segment governance |
| 38 | Marketing uses full customer export | Convenience | Data leak | Controlled audience tool/minimization |
| 39 | Suppression ignored | Lists disconnected | Unwanted contact | Suppression master |
| 40 | Transactional and marketing messages mixed | One template | Communication risk | Message classification |
| 41 | Trigger sends duplicate communication | Retry/multiple systems | Spam | Ownership/message ID |
| 42 | No frequency cap | Many campaigns | Customer fatigue | Contact policy |
| 43 | Attribution overwritten retroactively | No policy/version | Disputed ROI | Governed attribution |
| 44 | Service case stored in chat | No owner/SLA | Lost issue | Case entity |
| 45 | Task closed without result | Activity=outcome | Issue unresolved | Exit criteria |
| 46 | Warranty inferred without sale line | No source relation | Wrong service decision | Sale/product reference |
| 47 | Claim and question same type | No classification | Wrong priority | Separate category/lifecycle |
| 48 | SLA ignores store calendars | Simple elapsed time | False breach | Calendar policy |
| 49 | Compensation approved by executor | No segregation | Loss/fraud risk | Authority matrix |
| 50 | Store seller sees network customers | Broad role | Privacy leak | Store scope |
| 51 | Marketing sees support conversations | Copied role | Excess access | Segmented view |
| 52 | Support sees payment details | No minimization | Excess access | Finance reference only |
| 53 | Administrator used for operations | Convenience | Uncontrolled changes | Separate account |
| 54 | Temporary staff access remains | No expiry | Data leak | Time-bound access |
| 55 | Direct URL/REST/export not tested | UI-only QA | Access bypass | Multi-channel tests |
| 56 | Webhook URL in README | Secret misclassification | Compromise | Secret store/env |
| 57 | POS/payment payload logged | Full debug | Sensitive leak | Allowlist/masking |
| 58 | Error alert includes customer/receipt | No sanitization | Notification leak | Sanitized errors |
| 59 | No timeout | Happy path | Hung sync | Explicit timeout |
| 60 | Infinite retry | No budget | Load/backlog | Bounded backoff |
| 61 | Retry non-idempotent | No key | Duplicate sales/messages | Event ID |
| 62 | Out-of-order events applied | Async | State rollback | Sequence/version |
| 63 | Full import without dry-run | Rush | Mass corruption | Sample first |
| 64 | Identity matching not sampled | Counts only | Wrong merges | Identity QA sample |
| 65 | No source-target report | Opaque migration | No rollback/reconcile | Mapping log |
| 66 | No totals reconciliation | HTTP success assumed | Missing sales/customers | Acceptance report |
| 67 | Backup includes unnecessary customer data | No classification | Exposure | Classified backup |
| 68 | Backup not restore-tested | Formal control | Rollback fails | Restore drill |
| 69 | Sales by store use CRM responsible instead of source store | Wrong dimension | Wrong store KPI | POS store ID |
| 70 | Seller efficiency ignores shift/store assignment | No temporal relation | Wrong attribution | Effective-dated assignment |
| 71 | Average check includes returns inconsistently | No metric definition | Wrong KPI | Metric dictionary |
| 72 | Repeat rate uses email only | Weak identity | False result | Stable customer ID |
| 73 | LTV ignores returns/cost policy | Simplistic formula | Misleading value | Governed formula |
| 74 | Dashboard has no freshness | Stale data | Wrong decision | Last refresh/SLA |
| 75 | Marketplace app not assessed | Trust listing | Vendor/security risk | Review/exit plan |
| 76 | Target tariff not checked | Discovery missed | Infeasible rights/process | Portal verification |
| 77 | On-Premise outdated | Upgrade ignored | Errors/vulnerabilities | Supported plan |
| 78 | Big-bang all stores | No pilot/waves | Network-wide incident | Pilot and rollout waves |
| 79 | No integration/support owner | Handoff gap | Incidents unhandled | Support RACI |
| 80 | No hypercare during peak | Immediate closure | Sales/service disruption | Support window |

## 22. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Confirmed AS IS, stores/channels/systems and volumes | Inventory, journeys, identity assessment, capability check | Masters and owners defined |
| 2. Архитектура | Approved TO BE | Data model, identity/master matrix, rights, ADR, risks | Owners signed decisions |
| 3. CRM настройка | Working enquiry/service contour | Fields, stages, roles, dedup, tests | Smoke/rights tests pass |
| 4. Интеграции магазинов | Reliable POS/ERP/PIM/OMS/WMS flows | Contracts, IDs, retries, monitoring, reconciliation | Pilot store end-to-end passes |
| 5. Лояльность | Controlled membership/points projection | Mapping, event IDs, rights, reconciliation | Earn/spend/repeat tests pass |
| 6. Маркетинг | Governed segments/campaigns/communications | Eligibility, suppression, attribution and tests | Approved campaign pilot measured |
| 7. Тестирование | Business readiness confirmed | Test report, defects, UAT sign-off | No critical/high defects |
| 8. Запуск | Controlled wave rollout completed | Runbook, backup, rollback, communication | Smoke/reconcile per wave successful |
| 9. Оптимизация | Identity, SLA and KPI stabilized | Baseline, access review, backlog, lessons learned | Support model accepted |

### Официальные источники

Используются только источники из `knowledge/SOURCES.md`; дата последней проверки реестра — 2026-08-05.

| Источник | Применение |
|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | Пользовательские возможности CRM и Cloud |
| [Документация REST API](https://apidocs.bitrix24.ru/) | Интеграционная архитектура и ограничения |
| [REST API reference](https://apidocs.bitrix24.ru/api-reference/) | Проверка контрактов методов и полей |
| [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/) | Проверка вариантов интеграционного доступа |
| [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) | Проектирование customer, store, loyalty, return, service and campaign views |
| [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) | Проверка automation |
| [Как выбрать тип бизнес-процесса](https://helpdesk.bitrix24.ru/open/23951294/) | Выбор approval/orchestration механизма |
| [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) | Проверка вариантов телефонии |
| [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) | Проверка datasets и аналитики |
| [Маркетплейс Bitrix24](https://www.bitrix24.ru/apps/) | Проверка приложений и коннекторов |
| [Документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php) | On-Premise возможности и ограничения |

Если функция, поле, связь, событие, REST-метод, приложение или ограничение не подтверждены официальным источником и целевым порталом, они помечаются **«требует проверки»** и не включаются в обязательства перед заказчиком.

# Related Documents

- [Case Library](../README.md)
- [Architecture Patterns](../PATTERNS.md)
- [Discovery Template](../DISCOVERY_TEMPLATE.md)
- [Architecture Template](../ARCHITECTURE_TEMPLATE.md)
