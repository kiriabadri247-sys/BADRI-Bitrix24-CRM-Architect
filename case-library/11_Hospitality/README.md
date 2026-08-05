# Hospitality CRM Architecture Case

Архитектурный кейс для синтетической гостиничной компании, которая принимает запросы, управляет бронированиями, размещением, дополнительными услугами, обращениями, отзывами и повторными продажами. Документ задаёт референсную CRM-модель, а не готовую конфигурацию для копирования.

> **Граница безопасности:** реальные гостиницы, гости, телефоны, email, адреса, документы, номера комнат, платёжные сведения, CRM ID, webhook URL, токены и выгрузки не используются. Все данные синтетические.

> **Архитектурная граница:** Bitrix24 отвечает за CRM-профиль, допустимые коммуникации, продажи, сервис и минимальную проекцию бронирования. PMS остаётся master номерного фонда, доступности, брони, размещения и операционного статуса; channel manager/booking engine — канального распределения; POS/ERP/1С — услуг, начислений и финансовых фактов; платёжный провайдер — статуса транзакции.

Обозначения:

- **факт Bitrix24** — подтверждается официальным источником из `knowledge/SOURCES.md`;
- **решение архитектора** — целевой выбор кейса, который утверждается на Discovery;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа/редакции, версии, прав, региона, приложения или внешней системы.

## 1. Business Context

### Модель гостиничного бизнеса

Референсный сценарий — одна или несколько гостиничных площадок с прямыми и партнёрскими продажами. Гость выбирает размещение, бронирует категорию или конкретную единицу по правилам PMS, получает подтверждение, заезжает, пользуется услугами, выезжает и может вернуться.

| Аспект | Целевая интерпретация |
|---|---|
| Типы гостей | Индивидуальный, корпоративный, групповой, постоянный — синтетические сегменты |
| Каналы | Сайт, booking engine, телефон, email, разрешённые сообщения, партнёр/OTA |
| Коммерческая единица | Одно подтверждённое бронирование/сделка по принятой модели |
| Номерной фонд | PMS master по категориям, единицам, статусам и доступности |
| Сервис | Обращение/задача/претензия, связанные с гостем и пребыванием |
| Финансы | POS/PMS/ERP/1С и платёжный провайдер |

### Цели CRM

1. Сформировать допустимую единую историю взаимодействий гостя.
2. Не терять запросы и обращения.
3. Сделать подтверждение, изменения и исключения бронирования наблюдаемыми.
4. Координировать сервисные задачи и SLA без подмены PMS.
5. Управлять повторными коммуникациями по утверждённым правилам.
6. Получать сверяемые показатели бронирований, повторных визитов и качества сервиса.

```text
Channel -> CRM request -> Booking engine/PMS -> Reservation
   -> Pre-arrival -> Check-in -> Stay/Service -> Check-out
   -> Feedback -> Repeat visit
```

## 2. AS IS Problems

| Проблема AS IS | Проявление | Корневая причина | Целевой контроль |
|---|---|---|---|
| Нет единой истории гостя | Профили дублируются по каналам | Нет identity/master policy | Нормализация и conservative matching |
| Теряются заявки | Нет owner и первого ответа | Каналы не объединены | Intake, routing, fallback |
| Нет контроля бронирований | CRM/PMS показывают разные статусы | Нет master/mapping/reconciliation | PMS-authoritative flow |
| Коммуникации несогласованы | Гость получает дубли | Несколько систем отправляют сообщения | Communication ownership/idempotency |
| Нет аналитики повторных визитов | Брони не связаны с guest ID | Нестабильная идентичность | Stable external guest ID |
| Доступность устаревает | Продан занятый номер/категория | CRM кэшируют inventory | Real-time/controlled PMS confirmation |
| Перенос создаёт вторую бронь | Нет версии и компенсации | Системы обновляются независимо | PMS transaction/reconcile |
| Оплата отмечается вручную | Статус недостоверен | CRM подменяет finance | Provider/ERP confirmation |
| Запросы сервиса остаются в чатах | Нет owner/SLA | Нет service lifecycle | Case/task with outcome |
| Предпочтения записывают свободно | Избыточные/чувствительные данные | Нет data classification | Minimal structured preferences |

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Лид | Неквалифицированный запрос, если lead-based процесс подтверждён | Bitrix24 |
| CRM | Контакт | Минимальный профиль гостя | PMS/CRM/MDM по ADR |
| CRM | Компания | Корпоративный клиент/партнёр | ERP/MDM/Bitrix24 |
| CRM | Сделка | Коммерческая возможность/проекция брони | Bitrix24 |
| Service | Обращение | Запрос или проблема гостя | Bitrix24 |
| Domain | Smart Process «Бронирование» | Минимальная CRM-проекция reservation | PMS |
| Inventory | Smart Process «Номер» | Разрешённая проекция категории/единицы и статуса | PMS |
| Identity | Smart Process «Гость» | Дополнительный hospitality-profile, если Contact недостаточно | PMS/CRM by ADR |
| Service | Smart Process «Дополнительная услуга» | Заказ/исполнение услуги | PMS/POS/service system |
| Quality | Smart Process «Претензия» | Формальное несоответствие и решение | Bitrix24/PMS/quality system |
| Loyalty | Smart Process «Программа лояльности» | Минимальная проекция membership/status | Loyalty/PMS system |

**Факт Bitrix24:** CRM, Smart Processes, роботы, триггеры и бизнес-процессы описаны официальными материалами. Конкретные поля, связи, права, automation и доступность проверяются на целевом портале.

**Решение архитектора:** PMS создаёт/подтверждает бронь и назначение номерного фонда. Bitrix24 хранит external IDs, гостевой/корпоративный контекст, разрешённые коммуникации, нормализованный статус и сервисные действия.

```text
Lead/Request -> Contact/Company -> Deal
                     |              |
                     v              v
                   Guest <---- Reservation <----> PMS
                                  /   |   \
                                 v    v    v
                              Room  Payment Stay
                                      |      |
                                      v      v
                                  POS/ERP  Service/Claim
                                              |
                                              v
                                         Loyalty/BI
```

### Правила целостности

- один профиль гостя имеет устойчивый `external_guest_id`, но объединение не выполняется по одному слабому полю;
- одна бронь имеет `external_reservation_id` PMS;
- доступность и назначение номера подтверждаются только PMS;
- повторное событие не создаёт вторую бронь, услугу, оплату или сообщение;
- плановый заезд/выезд не считается фактом размещения;
- для каждого поля определены master, direction, classification, freshness и conflict rule;
- массовые операции имеют dry-run, sample, report, reconciliation and rollback.

## 4. Guest Journey Architecture

| Этап | Цель | Данные | Контроль |
|---|---|---|---|
| Запрос гостя | Зарегистрировать потребность | Channel, minimal contact, dates, occupancy, owner | Validation/dedup |
| Подбор размещения | Найти доступную категорию | PMS availability, category, tariff/version | No manual availability promise |
| Бронирование | Создать reservation | Guest ID, dates, category, tariff, channel | PMS atomic confirmation |
| Подтверждение | Зафиксировать authoritative status | Reservation ID/version/status | Idempotent message/event |
| Заезд | Получить минимальный факт | PMS check-in status/time | No document details in CRM |
| Проживание | Координировать сервис | Stay ID, allowed context, cases/tasks | PMS/service relations |
| Выезд | Получить факт завершения | Check-out status/time, open cases | Finance/service reconcile |
| Отзыв | Собрать допустимую обратную связь | Survey/reference, consent context | No public response automation without review |
| Повторный визит | Создать новую бронь | Existing guest/new reservation | Profile not duplicated |

```text
Запрос гостя -> Подбор размещения -> Бронирование -> Подтверждение
 -> Заезд -> Проживание -> Выезд -> Отзыв -> Повторный визит

Exceptions: no availability / change / cancellation / no-show / service claim
```

## 5. Hospitality Data Model

### Сущности и поля

| Сущность | Минимальные поля | Контроль качества |
|---|---|---|
| Гость | External ID, permitted contacts, segment, communication preferences, aggregate stay history | No identity documents/payment details in CRM |
| Предпочтение | Controlled category/value/source/date | Minimal, non-sensitive, owner-approved taxonomy |
| Бронирование | Reservation ID, property, dates, occupancy, category, tariff/version, amount summary, status | PMS source and version |
| Номер/категория | Room/category ID, characteristics, status/availability projection | PMS master and freshness |
| Пребывание | Stay ID, reservation, check-in/out facts, status | PMS fact, not planned dates |
| Услуга | Service ID, category, quantity, status, fulfillment owner, charge reference | POS/PMS master where applicable |
| Обращение/претензия | Case ID, stay/reservation, category, priority/severity, owner, result | Symptom, cause and resolution separated |
| Лояльность | Membership ID, tier/status, validity, points summary if required | Loyalty master; no manual balance edit |

```text
Contact/Company 1 ---- N Reservation 1 ---- 0..1 Stay
       |                     |  |  |
       +---- N Service Case  |  |  +---- N Payment/Charge reference
       +---- 0..1 Loyalty    |  +------- N Additional Service
                             +---------- 1 Room Category / 0..1 Room
```

### Кардинальность

| Отношение | Кардинальность | Правило |
|---|---:|---|
| Guest → Reservation | 1:N | Повторный визит — новая бронь |
| Company → Reservation | 1:N | Corporate relation separated from individual guest |
| Reservation → Stay | 1:0..1 or versioned by PMS model | Planned reservation ≠ actual stay |
| Reservation → Room assignment | N:0..1 at a moment | Category sale and room assignment distinguished |
| Stay → Additional service | 1:N | Service fulfillment and charge are separate facts |
| Stay → Case/Claim | 1:N | Service context is traceable |

## 6. Booking Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новый запрос | Зарегистрировать enquiry | Channel, dates, occupancy, property, owner | Routing and first-action task |
| Квалификация | Подтвердить потребность | Guest type, dates flexibility, category/service requirements | SLA/missing-data control |
| Подбор номера | Получить PMS availability | Category IDs, tariff/conditions, timestamp | Availability request; no local inventory |
| Предложение | Отправить versioned offer | Category/tariff version, amount, validity | Follow-up/expiry control |
| Подтверждение | Получить reservation ID/status | Guest, dates, category, policy references | PMS create/idempotent sync |
| Предоплата | Получить подтверждённый payment state | Payment ID, amount, currency, status | Provider/finance event/reconcile |
| Заезд | Получить PMS check-in fact | Stay ID, status, time | Pre-arrival tasks closed/updated |
| Проживание | Координировать service cases | Stay relation, open requests, SLA | Task routing/escalation |
| Закрытие | Подтвердить check-out/service/finance criteria | PMS fact, open claim/payment references | Completeness and feedback eligibility |

```text
Новый запрос -> Квалификация -> Подбор номера -> Предложение
 -> Подтверждение -> Предоплата -> Заезд -> Проживание -> Закрытие

Branches: unavailable / expired / cancelled / no-show / payment failed / claim
```

CRM-стадия отражает клиентский процесс. PMS status remains authoritative for confirmation, check-in, stay and check-out.

## 7. Room Management

| Компонент | Master | CRM-проекция | Контроль |
|---|---|---|---|
| Номер | PMS | Room ID, category, operational status if needed | Minimal data/freshness |
| Категория | PMS/catalog | Category ID, public characteristics | Versioned mapping |
| Доступность | PMS/channel manager | Availability result/timestamp, not free edit | No stale promise |
| Статус | PMS/housekeeping module | Normalized state summary | Unknown state remains exception |
| Обслуживание | PMS/maintenance/housekeeping system | Service task/reference/status | Operational owner |

```text
Inventory -> Category availability -> Reservation
 -> Room assignment -> Clean/Ready -> Check-in
 -> Occupied -> Check-out -> Cleaning/Maintenance -> Ready
```

Загрузка номерного фонда не рассчитывается по CRM-сделкам. Категория и конкретный номер различаются; продажа категории не означает раннее закрепление конкретной единицы, если PMS работает иначе.

## 8. Reservation Management

| Операция | Preconditions | Результат | Контроль |
|---|---|---|---|
| Создание | Valid PMS availability and guest context | One reservation ID/version | Idempotency/concurrency |
| Изменение | Current version and allowed status | Updated PMS reservation/new version | Price/policy re-evaluation |
| Отмена | Authority and policy context | Cancelled status and related actions | Release inventory, payment/service sync |
| Подтверждение | PMS-confirmed state | Confirmation event/message | Message ownership/idempotency |
| Уведомление | Eligible event/channel/template | Delivery result | Minimal data/no duplicates |

```text
Availability -> Create hold/reservation in PMS -> Confirm
       |                   |                    |
       +-> unavailable     +-> expire/cancel    +-> modify/check-in
```

Изменение дат, категории или тарифа требует версии и повторного расчёта условий согласно PMS. Нельзя обновлять только CRM, оставляя PMS без изменения.

## 9. Guest Service Architecture

| Компонент | Назначение | Данные | Exit criteria |
|---|---|---|---|
| Запрос гостя | Информационное/операционное действие | Category, stay, priority, owner | Result delivered/confirmed |
| Сервисная задача | Конкретная работа персонала | Location reference, deadline, assignee, outcome | Completed with result, not only status |
| Уборка | Room service coordination | Room/stay reference, schedule/status | PMS/housekeeping fact |
| Дополнительная услуга | Order and fulfillment | Service ID, quantity, owner, status, charge reference | Fulfilled and reconciled |
| Претензия | Formal dissatisfaction/impact | Severity, stay, evidence reference, decision | Resolution communicated and recorded |
| Контроль качества | Service experience review | Allowed feedback and case metrics | No unsupported public promise |

```text
Guest request -> Identify reservation/stay -> Classify -> Assign
 -> Fulfil -> Confirm result -> Close
             |
             +-> Delay/escalation -> Claim -> RCA/corrective action
```

SLA defines start, calendar, allowed pauses, priority, reaction/resolution targets and escalation. Implementation depends on target portal capabilities.

## 10. Loyalty Management

| Механизм | Назначение | Master | Контроль |
|---|---|---|---|
| Повторный визит | New reservation for existing guest | PMS/CRM identity link | No contact duplication |
| Сегментация | Approved grouping by service/business criteria | CRM/BI/loyalty system | Data minimization/freshness |
| Программа лояльности | Membership/tier/rules | Loyalty/PMS system | External ID/version |
| Баланс/статус | Points/status projection if needed | Loyalty system | Read-only/reconcile |
| Коммуникация | Permitted offer/service message | CRM/approved channel | Eligibility, suppression, frequency cap |

История предпочтений и лояльности не должна раскрывать чувствительную информацию или свободные субъективные заметки. Ручное изменение баланса/уровня в CRM запрещается, если master — внешняя система.

## 11. Rights Model

| Роль | Гости | Бронирования | Услуги | Документы | Аналитика |
|---|---|---|---|---|---|
| Менеджер бронирования | Assigned/queue minimum | Create/change through approved PMS flow | Commercial add-ons | Offer/contract metadata | Personal/team sales |
| Ресепшен | Current/upcoming stay minimum | Check-in/out projection and service context | Stay-related requests | Only required operational references | Operational shift view |
| Управляющий | Property scope | Property reservations/exceptions | Service/claims oversight | Necessary management scope | Property KPI |
| Сервисный персонал | Minimal guest/room context | No general booking access | Assigned tasks/services | No guest documents | Own workload/status |
| Бухгалтер | Requisites/finance context | Payment/charge references | Charge context | Invoices/acts/contracts metadata | Finance |
| Директор | Management scope | Aggregated portfolio/property view | Service quality overview | Only necessary documents | Management KPI |

Точная реализация CRM-ролей, Smart Process rights, отделов, групп, наследования и экспорта требует проверки на целевом портале. Роль персонала не должна открывать полную гостевую историю, документы или платёжные данные.

Контроли: least privilege, property/shift scope, service-account minimum scope, offboarding, temporary staff expiry, access review and UI/direct URL/search/REST/export testing.

## 12. Automation Architecture

| Механизм | Сценарий | Защита | Ошибка/эскалация |
|---|---|---|---|
| Робот/задача | First response | One marker per request | Fallback owner |
| Уведомление | Booking confirmation | PMS reservation ID/version | Delivery result/no duplicates |
| Уведомление | Pre-arrival reminder | Eligibility, minimal template | Failed delivery route |
| Задача | Staff service request | Stay/service ID, deadline, outcome | Reassign/escalate |
| Робот | SLA control | Calendar/priority/pause rules | Tiered escalation |
| Робот | No-show/cancel follow-up | Authoritative PMS status | Approved communication only |
| Бизнес-процесс | Claim/compensation approval | Version/authority/limit | Reject/timeout path |

```text
Event -> Authenticate -> Validate source/version -> Eligibility
 -> Execute -> Record result -> Bounded retry -> Escalate/Reconcile
```

Каждая automation имеет owner, trigger, preconditions, data classification, idempotency key, timeout, retry budget, error route, audit trail and test case. Робот не подтверждает availability/payment/stay fact без master event.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| PMS | Guest ID, room/category, reservation, stay, service status by allowlist | REST/OAuth/application/event/batch contract if supported | Auth, idempotency, version, ordering, reconcile |
| Сайт | Enquiry and permitted contact/context | CRM form/REST after verification | Validation, dedup, spam control |
| Online booking/channel manager | Availability request, tariff, booking/channel reference | Provider API/application/event | Concurrency, mapping, latency, reconcile |
| Payment provider | Payment/refund state | Provider API/event through controlled integration | Authenticity, idempotency, amount reconcile |
| Телефония | Call event and CRM relation | Official Bitrix24 telephony/verified provider | Rights, unmatched calls, recording policy |
| 1С/ERP/POS | Counterparty, charges, invoices, acts, payment/accounting facts | Verified connector/application or contract integration | External IDs, mapping, queue, totals |
| BI/DWH | Approved datasets | BI Constructor/controlled export | Access, freshness, identity and metric reconcile |

```text
Source -> Authenticate -> Minimize -> Validate -> Map -> Idempotency -> Target
  |                                                                  |
  +-> source audit                                       target result
                 \                                        /
                  +------- retry / reconciliation --------+
```

Контракт фиксирует source of truth, direction, keys, schema/version, property/timezone, currency, tariff version, event ordering, create/change/cancel semantics, timeout, retry budget, security, monitoring, recovery and owner.

Наличие готовой интеграции с конкретной PMS, channel manager, payment provider or 1С configuration не предполагается. Приложение проверяется в официальном Маркетплейсе, внешний API — в документации поставщика отдельным scope.

## 14. Analytics

| Отчёт | Метрика | Grain | Источник сверки |
|---|---|---|---|
| Загрузка номерного фонда | Occupied/available room nights | Property/category/date | PMS, not CRM deals |
| Количество бронирований | Created/confirmed/cancelled/no-show | Reservation/cohort | PMS |
| Средний чек | Eligible confirmed revenue / stays | Stay/reservation cohort | PMS/POS/ERP |
| Повторные гости | Guests with repeated completed stays | Guest cohort | Stable PMS/CRM identity |
| Эффективность каналов | Confirmed stays/revenue/cancellations | Channel/cohort | PMS/channel attribution policy |
| Качество сервиса | SLA, cases, claims, permitted feedback | Stay/case period | CRM/PMS service data |
| Дополнительные услуги | Ordered/fulfilled/charged | Service/stay | PMS/POS |
| Интеграции | Error/lag/reconciliation delta | Flow-period | Technical monitoring |

```text
CRM + PMS + Channel Manager + POS/ERP + Payments
 -> Data contracts -> Quality/privacy gate -> Dataset/DWH
 -> Metric dictionary -> Dashboard -> Decision
                         ^                |
                         +-- reconcile ---+
```

BI Constructor, datasets and fields depend on target portal. Metric dictionary includes formula, grain, cohort, property, timezone, currency, eligible statuses, attribution, owner, freshness and reconciliation source.

## 15. Security Architecture

### 1. Role Based Access Control

Роли reservation agent, front office, guest service, sales/marketing, finance, property manager и administrator получают минимальный доступ по объекту размещения и операции. Данные гостя, оплата, лояльность, экспорт и настройки разделяются; shared accounts не используются как архитектурная норма.

| Роль | Рабочий scope | Ограничение |
|---|---|---|
| Reservation agent | Запросы и бронирования | Только назначенный property/channel scope |
| Front office | Заезд, проживание, сервис | Минимальный guest context без admin settings |
| Guest service | Обращения и задачи | Только связанная бронь/гость |
| Marketing | Сегменты и коммуникации | Минимизированные данные и подтверждённые разрешения |
| Finance | Оплаты и документы | Без управления guest lifecycle |

### 2. Department Isolation

Properties, front office, reservations, service, marketing и finance разделяются организационной структурой и ролями. Межобъектный доступ предоставляется централизованным функциям по утверждённому scope. Передача гостя или брони не создаёт дубликаты Customer 360.

### 3. CRM Permissions

| Объект | Правило доступа |
|---|---|
| Лиды | Sales/reservation intake, если лиды используются |
| Контакты | По guest relationship и property scope |
| Компании | Корпоративные клиенты и партнёры по account ownership |
| Сделки | По booking/sales pipeline, ответственному и property |
| Smart Processes | По роли в бронировании, номере, сервисе или loyalty lifecycle |

Просмотр, изменение, удаление, export, mass communication и API проверяются отдельно. PMS остаётся authoritative для согласованных reservation/room facts.

### 4. Document Security

Бронирования, счета, подтверждения, договоры, обращения и guest attachments доступны по need-to-know. Платёжные credentials и лишние идентифицирующие данные не помещаются в CRM. Документы хранятся в утверждённом защищённом контуре с отдельными правами просмотра, изменения и retention.

### 5. Audit

Контролируются изменения гостя, брони, property/room, дат, суммы, статуса оплаты, скидки, ответственного и коммуникационного статуса. CRM history сопоставляется с masked PMS/payment/integration logs по correlation ID. Property и security owners проводят review и RCA конфликтов или несанкционированных изменений.

### 6. Cloud Limitations

- безопасность, права, history и ограничения коммуникаций зависят от тарифа, региона и конфигурации портала;
- конкретные возможности проверяются на целевом Cloud-портале;
- нет доступа к server, PHP, core и DB;
- PMS, payment и внешние security controls не считаются функциями Bitrix24 без подтверждённой интеграции.

### 7. On-Premise Considerations

Администратор дополнительно отвечает за OS, web server, PHP, DB, modules, files, cron/agents, network, TLS, logs, backup и updates. Доступ к инфраструктуре и приложению разделяется, восстановление проверяется, сервисные учётные записи ограничены, ядро не изменяется.

---

## 16. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | New guest request | Validation, dedup, owner | One request and first action |
| 2 | Existing guest/new stay | Identity/new reservation | Guest profile not duplicated |
| 3 | PMS availability | Category/dates/property/timestamp | Only authoritative result shown |
| 4 | Concurrent booking | Two requests for last inventory | PMS confirms only valid reservation |
| 5 | Reservation create | External ID/version | One CRM projection |
| 6 | Repeated event | Idempotency | No duplicate reservation/message |
| 7 | Date/category change | Current version/reprice | PMS/CRM stay consistent |
| 8 | Cancellation | Policy/status/inventory/reminders | History retained and related actions updated |
| 9 | Payment success | Authenticity/amount/currency | Confirmed status reconciled |
| 10 | Payment failure/refund | Error/repeat/amount | No false paid/refunded state |
| 11 | Pre-arrival notification | Eligibility/template/delivery | One minimal message |
| 12 | Check-in/check-out | PMS fact status | Planned dates not treated as fact |
| 13 | Additional service | Order/fulfillment/charge relation | Statuses remain distinct/reconciled |
| 14 | Guest request | Stay relation/SLA/owner | Correct task and escalation |
| 15 | Claim | Severity/authority/result | Controlled lifecycle |
| 16 | PMS/API failure | Timeout/retry/dead event | No loss/duplicate, alert traceable |
| 17 | Out-of-order events | Version/sequence | Status does not roll back |
| 18 | Rights | UI/URL/search/REST/export | Role sees only allowed scope |
| 19 | Migration | Dry-run/sample/totals | Mapping and identity accepted |
| 20 | BI reconciliation | PMS/CRM/POS sample | Delta explained or corrected |
| 21 | Rollback | Trigger/runbook/reconcile | Safe prior state restored |

Тестовые данные только синтетические. UAT подписывают Reservations, Front Desk, Housekeeping/Service, Property Management, Finance, Marketing/Loyalty, Security and Integration owners. Negative, repeat, concurrency, role and recovery cases обязательны.

## 17. Deployment

| Этап | Действия | Gate |
|---|---|---|
| Discovery | AS IS, properties, PMS/channels, volumes, data and owners | Inventory/scope signed |
| Архитектура | Data model, masters, state mappings, rights, ADR | TO BE approved |
| Настройка | Test portal/funnel, fields, stages, roles | Peer/security review and smoke |
| Интеграции | Contract, concurrency, failure/recovery tests | End-to-end successful |
| Миграция | Minimize, clean, map, dry-run, sample, reconcile | Data owner acceptance |
| Обучение | Role/property/shift scenarios | UAT/readiness sign-off |
| Запуск | Freeze, backup/export, cutover, smoke | Go/no-go |
| Hypercare | Monitoring, incident route, daily reconcile | No critical defects, support accepted |

```text
Discovery -> Architecture -> Test configuration -> PMS contract tests
 -> Sample migration -> UAT -> Go/No-Go -> Cutover -> Smoke -> Hypercare
                                      |
                                      +-> Rollback by approved trigger
```

Cloud and On-Premise planned separately. For On-Premise additionally verify product/modules, PHP, database, files, cron, backup/restore, infrastructure and security. Core product files are not modified.

## 18. Monitoring

| Контроль | Сигнал | Owner | Реакция |
|---|---|---|---|
| Необработанные заявки | No owner/first action | Reservations lead | Assign/RCA |
| Ошибка бронирования | PMS reject/unknown/reconcile delta | Reservations/integration owner | Contain/recover/reconcile |
| Проблема оплаты | Failed/unknown/delta | Payments/finance owner | Verify provider/incident |
| Просроченная задача | SLA/deadline breached | Service lead | Reassign/escalate |
| Качество сервиса | Claim/severity/repeat threshold | Property manager | Containment/RCA/action |
| PMS status stale | Freshness threshold | Integration owner | Mark stale/provider incident |
| Duplicate guest/reservation | Identity/key alert | Data owner | Contain/review rules |
| Unsent communication | Delivery failure | Communication owner | Approved alternative route |
| Anomalous access/export | Security signal | Security owner | Contain/audit/revoke |

SLI: success/error rate, latency, backlog age, retry count, last successful sync, stale age and reconciliation delta. Logs contain no secrets, identity documents, room/stay details beyond necessary IDs, payment payloads or full contact data.

## 19. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Bitrix24 used as PMS | Wrong boundary | Overbooking/wrong operations | PMS master |
| 2 | Duplicate guests | No identity policy | Fragmented history | Conservative matching |
| 3 | Wrong guest merge | Weak key | Other guest data/communication | Human-controlled merge |
| 4 | Email/phone sole key | Shared/changed contact | Wrong identity | Approved multi-attribute resolution |
| 5 | Duplicate reservation | Repeated event | Double inventory/communication | Idempotency/external ID |
| 6 | Availability stale | Cached CRM state | Overbooking | PMS confirmation/freshness |
| 7 | Category and room mixed | Weak model | Wrong assignment/analytics | Separate entities |
| 8 | Planned room assignment treated as final | PMS model ignored | Operational conflict | PMS authoritative assignment |
| 9 | Concurrent last-room booking | No atomic PMS operation | Overbooking | PMS transaction/concurrency test |
| 10 | Change only in CRM | Free bidirectional edit | PMS mismatch | Master-controlled command/event |
| 11 | Cancellation does not release inventory | Incomplete compensation | Lost sales | PMS operation/reconcile |
| 12 | Cancellation leaves reminders | Systems disconnected | Wrong message | Compensating event |
| 13 | No-show inferred from date | No PMS fact | Wrong follow-up/report | PMS status |
| 14 | Check-in/out plan equals fact | Semantics mixed | False occupancy | Fact event |
| 15 | Events out of order | Async flow | Status rollback | Version/sequence |
| 16 | Unknown PMS state means success | Optimistic mapping | False closure | Exception/hold |
| 17 | Tariff outdated | No version/validity | Price dispute | PMS tariff revision |
| 18 | Currency missing | Incomplete model | Wrong total | Amount + currency |
| 19 | Payment marked manually | CRM replaces finance | False paid state | Provider/ERP fact |
| 20 | Repeated callback duplicates payment | No event ID | Financial error | Idempotency |
| 21 | Payment details stored in CRM | Excess payload | Security risk | Token/reference only |
| 22 | Refund request equals refund fact | Lifecycles mixed | False finance report | Separate statuses |
| 23 | Duplicate confirmation message | Multiple senders/retry | Guest dissatisfaction | Message ownership/key |
| 24 | Message reveals room/stay details | Excess template | Privacy/security risk | Minimal neutral content |
| 25 | Marketing without eligibility | No consent/suppression | Complaint/legal risk | Approved policy |
| 26 | Loyalty balance manually edited | CRM as master | Wrong benefits | Loyalty master/read-only |
| 27 | Preferences stored as free notes | No taxonomy | Sensitive/excess data | Minimal controlled values |
| 28 | Staff sees full guest history | Broad role | Privacy breach | Need-to-know |
| 29 | Housekeeping sees documents/payments | Copied role | Excess access | Task/room-only view |
| 30 | Temporary worker access never expires | No offboarding | Data leak | Time-bound access |
| 31 | Service account is administrator | Fast setup | Large blast radius | Least scope |
| 32 | Webhook URL published | Secret misclassification | Compromise | Secret store/rotation |
| 33 | PMS/payment payload in logs | Full debug | Sensitive leak | Allowlist/masking |
| 34 | Error alert contains guest data | No sanitization | Leak in notification | Sanitized errors |
| 35 | Call recording not reviewed | Default configuration | Legal/privacy risk | Legal/IB decision |
| 36 | Infinite retry | No budget | Load/backlog | Bounded backoff |
| 37 | Retry non-idempotent | No key | Duplicates/messages | Event ID |
| 38 | REST limit/timeout | Peak arrivals/booking | Lag | Queue/backoff/verified batch |
| 39 | Full migration without dry-run | Rush | Mass corruption | Sample/report/rollback |
| 40 | No reconciliation | HTTP success assumed | Silent omissions | Totals/sample/delta |
| 41 | Backup contains unnecessary guest data | No classification | Exposure/retention risk | Classified backup policy |
| 42 | Backup not restore-tested | Formal control | Rollback fails | Restore drill |
| 43 | Portal tariff lacks needed controls | Capability check missed | Infeasible design | Portal verification |
| 44 | Marketplace app not assessed | Trust listing | Vendor/security risk | Review/exit plan |
| 45 | On-Premise outdated | Old modules/PHP | Errors/vulnerabilities | Supported update plan |
| 46 | Rights tested as administrator | No role QA | Production access defects | Role-based UAT |
| 47 | Repeat guest metric uses email only | Weak identity | False repeat rate | Stable guest ID |
| 48 | Occupancy calculated from CRM deals | Wrong source/grain | Wrong planning | PMS room-night data |
| 49 | Average check includes cancellations | No eligibility definition | Wrong KPI | Metric dictionary |
| 50 | Channel attribution changes retroactively | No policy | Disputed performance | Versioned attribution |
| 51 | Dashboard has no freshness | Stale data | Wrong decision | Last refresh/SLA |
| 52 | No integration owner | Project ended | Errors unhandled | Support RACI |
| 53 | No concurrency test | Sequential QA only | Overbooking | Parallel scenarios |
| 54 | Shift staff not trained | Office-only rollout | Process bypass | Role/shift training |
| 55 | No hypercare | Immediate handoff | Incidents accumulate | Support window |

## 20. ADR

### ADR-HSP-001: Модель гостя

- **Контекст:** гостю нужна единая сервисная история, но PMS/CRM identities могут расходиться.
- **Решение:** Contact is minimal CRM profile linked to PMS external guest ID; separate Guest Smart Process only if justified by additional lifecycle.
- **Альтернативы:** Guest Smart Process instead of Contact; PMS-only profile.
- **Последствия:** conservative matching, minimization and dedup governance.
- **Статус:** Proposed.

### ADR-HSP-002: Модель бронирования

- **Контекст:** reservation has dates, tariff, inventory and PMS status.
- **Решение:** PMS is master; Smart Process/Deal is controlled CRM projection.
- **Альтернативы:** Deal as reservation master; PMS-only no CRM view.
- **Последствия:** external ID, version, idempotency, reconciliation.
- **Статус:** Proposed.

### ADR-HSP-003: Номерной фонд

- **Контекст:** categories, rooms, availability and housekeeping statuses change frequently.
- **Решение:** PMS/channel manager remain masters; Bitrix24 stores only required projection/reference.
- **Альтернативы:** CRM product/Smart Process as inventory master.
- **Последствия:** freshness, category/room separation and no manual availability edit.
- **Статус:** Proposed.

### ADR-HSP-004: Дополнительные услуги

- **Контекст:** service order, fulfillment and charge are separate facts.
- **Решение:** Smart Process coordinates service; PMS/POS/ERP master fulfillment/charge as assigned.
- **Альтернативы:** free-text task; Deal product only.
- **Последствия:** service ID, outcome, charge reference and reconciliation.
- **Статус:** Proposed.

### ADR-HSP-005: Интеграция PMS

- **Контекст:** PMS generates authoritative high-frequency reservation/stay events.
- **Решение:** versioned allowlist contract with idempotency, ordering, retries and reconciliation.
- **Альтернативы:** free bidirectional sync; periodic full CSV.
- **Последствия:** integration monitoring and outage fallback.
- **Статус:** Proposed.

### ADR-HSP-006: Модель прав

- **Контекст:** reservations, front desk, service and finance need different guest scopes.
- **Решение:** role/property/shift-based least privilege with UI/URL/REST/export tests.
- **Альтернативы:** common role; admin for operations.
- **Последствия:** test accounts, access review, offboarding/expiry.
- **Статус:** Proposed.

### ADR-HSP-007: Аналитическая архитектура

- **Контекст:** PMS contains occupancy/stay facts, POS/ERP revenue and CRM service history.
- **Решение:** governed datasets linked by stable IDs with metric dictionary/reconciliation.
- **Альтернативы:** only CRM reports; manual spreadsheets.
- **Последствия:** identity, room-night grain, freshness and access controls.
- **Статус:** Proposed.

### ADR-HSP-008: Лиды или Reservation-First

- **Контекст:** confirmed online booking is already a qualified business event.
- **Решение:** lead only for unqualified enquiry; valid booking enters reservation/deal flow directly.
- **Альтернативы:** lead for every booking; Deal-First for all requests.
- **Последствия:** choice fixed before channels and analytics setup.
- **Статус:** Open until Discovery.

## 21. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | CRM declared PMS | Wrong scope | Overbooking/operational errors | PMS master |
| 2 | One guest profile per reservation | No identity policy | Duplicates | Stable guest ID |
| 3 | Phone is sole key | Shared/changed number | Wrong merge | Conservative resolution |
| 4 | Email is sole key | Shared address | Wrong identity | Approved multi-attribute match |
| 5 | Auto-merge without review | Cleanup pressure | Mixed guest histories | Human-controlled merge |
| 6 | Real passport data in test | No synthetic dataset | Data exposure | Synthetic-only QA |
| 7 | Identity documents attached to Deal | One-window desire | Excess access | Secure PMS/DMS reference |
| 8 | One Deal for all stays | Guest and reservation mixed | Wrong analytics | Reservation/deal unit |
| 9 | Lead created for every PMS event | No event boundary | CRM noise | Business-event model |
| 10 | Reservation is only a task | Task replaces entity | No version/status | Reservation projection |
| 11 | Repeated event creates reservation | No idempotency | Duplicate inventory/messages | External/event ID |
| 12 | Category and room are one field | Weak model | Wrong assignment/report | Separate concepts |
| 13 | Room number used as global key | Multi-property collision | Wrong relation | Property + room ID |
| 14 | Availability copied to CRM nightly | Stale architecture | Overbooking | PMS confirmation |
| 15 | Manager manually marks room available | No ownership | Inventory conflict | Read-only PMS projection |
| 16 | Planned room assignment treated as guaranteed | PMS logic ignored | Guest conflict | Authoritative assignment status |
| 17 | Change only in CRM | Free edit | PMS mismatch | Command to PMS/reconcile |
| 18 | Change overwrites tariff/conditions | No version | Price dispute | Reservation revision |
| 19 | Cancellation deletes record | Cleanup desire | No audit | Cancel status/reason |
| 20 | Cancellation does not release inventory | Incomplete process | Lost sales | PMS compensation |
| 21 | Cancellation leaves reminders | Systems disconnected | Wrong communication | Compensating event |
| 22 | No-show inferred by date | No PMS fact | False status | PMS event |
| 23 | Planned check-in equals fact | Semantics mixed | Wrong occupancy | Fact status |
| 24 | Unknown status means checked-in/out | Optimistic mapping | False stay state | Exception/hold |
| 25 | Events applied out of order | Async | Status rollback | Version/sequence |
| 26 | Tariff name is key | No tariff ID | Wrong mapping | Stable tariff/version |
| 27 | Current tariff overwrites booking history | No snapshot | Wrong amount | Reservation snapshot |
| 28 | Amount lacks currency | Incomplete model | Wrong finance analytics | Amount + currency |
| 29 | Offer has no validity | No expiry | Stale promise | Validity date |
| 30 | Payment marked by receptionist | No provider sync | False paid state | Provider/ERP fact |
| 31 | Repeated callback duplicates payment | No event ID | Financial error | Idempotency |
| 32 | Card/payment details stored in CRM | Excess payload | Security risk | Token/reference only |
| 33 | Refund request equals refund fact | Process mixed | Wrong report | Separate statuses |
| 34 | Confirmation sent by PMS and CRM | No communication owner | Duplicate message | Ownership/messaging key |
| 35 | Message contains room/stay details | Excess template | Privacy/security risk | Minimal neutral content |
| 36 | Delivery result not tracked | Fire-and-forget | Guest uninformed | Delivery monitoring |
| 37 | Retry sends duplicate reminder | No idempotency | Spam | Message key |
| 38 | Marketing ignores suppression | Lists disconnected | Unwanted contact | Suppression master |
| 39 | Loyalty balance edited in CRM | CRM treated as master | Wrong benefit | Loyalty system read-only |
| 40 | Preferences stored as free subjective notes | No taxonomy | Sensitive/bias risk | Minimal controlled values |
| 41 | Service request stored in chat | No entity/owner | Lost request | Case/task lifecycle |
| 42 | Task closed without outcome | Activity equals result | Service unresolved | Exit criteria |
| 43 | Housekeeping task not linked to room/stay | No relation | Wrong service | Required IDs |
| 44 | Additional service order equals fulfilled | States mixed | Wrong guest/charge status | Separate lifecycle |
| 45 | Charge entered manually from task | No POS sync | Finance mismatch | POS/ERP reference |
| 46 | Claim stored in comment | No lifecycle | No SLA/RCA | Claim entity |
| 47 | Severity and priority mixed | No method | Wrong response | Separate definitions |
| 48 | Compensation approved by executor | No segregation | Financial/service risk | Authority matrix |
| 49 | Staff sees all guest profiles | Broad role | Privacy breach | Property/shift scope |
| 50 | Housekeeping sees documents/payments | Copied role | Excess access | Task-only view |
| 51 | Accountant sees service conversations | No segregation | Excess access | Finance-only scope |
| 52 | Administrator used by front desk | Convenience | Uncontrolled changes | Separate accounts |
| 53 | Temporary staff access does not expire | No offboarding | Data leak | Time-bound access |
| 54 | Service account is administrator | Fast integration | Large blast radius | Least scope |
| 55 | Direct URL/REST/export not tested | UI-only QA | Access bypass | Multi-channel tests |
| 56 | Webhook URL in README | Secret misclassification | Compromise | Secret store/env |
| 57 | PMS/payment payload logged | Full debug | Sensitive leak | Allowlist/masking |
| 58 | Error notification contains guest data | No sanitization | Leakage | Sanitized errors |
| 59 | No timeout | Happy path | Hung sync | Explicit timeout |
| 60 | Infinite retry | No budget | Load/backlog | Bounded backoff |
| 61 | Retry non-idempotent | No key | Duplicates | Event ID |
| 62 | Errors silently ignored | No dead-letter | Missing reservations | Failed-event register |
| 63 | Full import without dry-run | Rush | Mass corruption | Sample first |
| 64 | Guest matching not sampled | Only totals checked | Wrong merges | Identity QA sample |
| 65 | No source-target report | Opaque migration | No rollback/reconcile | Mapping log |
| 66 | No counts/reconciliation | HTTP success assumed | Silent omissions | Acceptance report |
| 67 | Backup includes unnecessary guest data | No classification | Exposure | Classified backup |
| 68 | Backup never restore-tested | Formal control | Rollback fails | Restore drill |
| 69 | Occupancy computed from CRM deals | Wrong source/grain | Wrong KPI | PMS room nights |
| 70 | Booking count includes enquiries | No eligibility rule | Inflated metric | Confirmed reservation definition |
| 71 | Average check includes cancelled stays | Wrong denominator | Misleading KPI | Metric dictionary |
| 72 | Repeat guest based only on email | Weak identity | False repeats | Stable guest ID |
| 73 | Channel attribution overwritten | No policy/version | Disputed performance | Governed attribution |
| 74 | Dashboard has no freshness | Stale data | Wrong decision | Last refresh/SLA |
| 75 | Marketplace app not assessed | Trust listing | Vendor/security risk | Review/exit plan |
| 76 | Portal capability not checked | Discovery missed | Infeasible workflow/rights | Target verification |
| 77 | On-Premise version unsupported | Upgrade ignored | Errors/vulnerabilities | Supported plan |
| 78 | Training only office staff | Shift model ignored | Night shift bypasses CRM | Role/shift training |
| 79 | No integration/support owner | Handoff gap | Incidents unhandled | Support RACI |
| 80 | No hypercare | Immediate closure | Issues accumulate | Support window |

## 22. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Confirmed AS IS, properties, PMS and channels | Inventory, journeys, integrations, capability check | Masters and owners defined |
| 2. Архитектура | Approved TO BE | Data model, state mappings, RACI, ADR, risks | Owners signed decisions |
| 3. CRM настройка | Working enquiry/service contour | Fields, stages, roles, dedup, tests | Smoke/rights tests pass |
| 4. Бронирования | Controlled PMS reservation projection | IDs, versions, lifecycle, notification rules | Synthetic booking journey passes |
| 5. Интеграции PMS | Reliable PMS/channel/payment/ERP flows | Contracts, retries, monitoring, reconciliation | Concurrency/failure/recovery tests pass |
| 6. Тестирование | Business readiness confirmed | Test report, defects, UAT sign-off | No critical/high defects |
| 7. Запуск | Controlled cutover completed | Runbook, backup, rollback, communication | Smoke/reconcile successful |
| 8. Оптимизация | SLA, identity and KPI stabilized | Baseline, access review, backlog, lessons learned | Support model accepted |

### Официальные источники

Используются только источники из `knowledge/SOURCES.md`; дата последней проверки реестра — 2026-08-05.

| Источник | Применение |
|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | Пользовательские возможности CRM и Cloud |
| [Документация REST API](https://apidocs.bitrix24.ru/) | Интеграционная архитектура и ограничения |
| [REST API reference](https://apidocs.bitrix24.ru/api-reference/) | Проверка контрактов методов и полей |
| [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/) | Проверка вариантов интеграционного доступа |
| [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) | Проектирование reservation, room, guest, service, claim and loyalty views |
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
