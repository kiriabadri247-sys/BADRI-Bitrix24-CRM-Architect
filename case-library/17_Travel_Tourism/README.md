# Travel and Tourism Customer Journey CRM Architecture Case

Архитектурный кейс для синтетической туристической компании, продающей туры B2C- и B2B-клиентам и сопровождающей бронирования и поездки. Документ задаёт референсную CRM-архитектуру, а не готовую конфигурацию, систему бронирования или туристическую рекомендацию.

> **Граница безопасности:** реальные туристические компании, путешественники, семьи, партнёры, адреса, маршруты, документы, паспортные данные, билеты, брони, платежи, телефоны, email, CRM ID, webhook URL и токены не используются. Все примеры синтетические.

> **Архитектурная граница:** Bitrix24 отвечает за CRM, продажи, обращения и клиентское сопровождение. Booking/GDS/channel/product systems остаются master наличия, цены, брони и supplier confirmation; payment provider и 1С/ERP — платежей, возвратов и финансов; ЭДО/DMS — подписанных документов. CRM не заменяет booking engine, GDS, ERP или emergency management platform.

Обозначения:

- **факт Bitrix24** — подтверждён официальным источником из `knowledge/SOURCES.md`;
- **решение архитектора** — проектная рекомендация, утверждаемая на Discovery;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа/редакции, версии, прав, региона, приложения или внешней системы.

## 1. Business Context

### Модель туристического бизнеса

| Аспект | Референсная модель |
|---|---|
| B2C | Индивидуальные туристы и группы/семьи с контекстными ролями участников |
| B2B | Корпоративные клиенты, организаторы и согласующие лица |
| Направления продаж | Пакетные/индивидуальные поездки, деловые поездки и дополнительные услуги |
| Каналы | Сайт, телефон, почта, офис, партнёры и разрешённые сообщения |
| Продажа | Отдельная opportunity от запроса до подтверждённой брони |
| Исполнение | Booking/trip facts приходят из профильных систем и партнёров |

### Цели CRM

1. Не терять запросы и назначать владельца следующего шага.
2. Создать допустимый единый профиль взаимодействий клиента без ложного merge.
3. Связать запрос, предложение, booking, trip, payment и support case stable IDs.
4. Разделить traveler, purchaser, company, partner и participant roles.
5. Контролировать сроки, подтверждения, изменения и обращения без подмены booking master.
6. Получать сверяемую аналитику продаж, направлений и повторных покупок.

```text
Channels -> Enquiry -> Deal -> Offer -> Booking Request -> Supplier Confirmation
                                      |                         |
                                      v                         v
Client/Travelers <---------------- Booking -----------------> Trip
       |                              |                         |
       +-> Support/Complaint          +-> Payment/Refund        +-> Feedback
```

## 2. AS IS Problems

| Проблема | Проявление | Причина | Целевой контроль |
|---|---|---|---|
| Теряются заявки | Нет owner/SLA | Intake по разным каналам | Ack/routing/reconciliation |
| Нет истории клиента | Поездки и контакты разрознены | Нет stable identity | Governed customer/trip relations |
| Нет контроля брони | Статус меняют вручную | Нет booking integration | Source status/freshness |
| Проблемы коммуникации | Сообщения без цели/версии | Нет communication policy | Transactional/marketing separation |
| Слабая аналитика | Deal, booking и traveler смешаны | Grain не определён | Metric contracts |
| Устаревшая цена | Копия предложения | Нет validity/version | Booking/product master |
| Двойное бронирование | Retry без idempotency | Нет source request ID | Unique booking key |
| Семья — один контакт | Упрощённая модель | Роли не выделены | Group/participant relation |
| Документы в комментариях | Нет защищённого хранилища | Быстрый обмен | DMS refs/access control |
| Отмена не сверяется | CRM верит локальному статусу | Нет payment/supplier reconciliation | Source-confirmed lifecycle |

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Лид | Неквалифицированный запрос, если lead model утверждена | Bitrix24 |
| CRM | Контакт | Физическое лицо | Bitrix24/MDM по ADR |
| CRM | Компания | Корпоративный клиент/партнёр | Bitrix24/ERP/partner system |
| CRM | Сделка | Коммерческая возможность | Bitrix24 |
| CRM | Обращение | Вопрос, support request или complaint | Bitrix24 |
| Smart Process | Туристический запрос | Structured travel requirements | Bitrix24 |
| Smart Process | Тур | Minimal product/version projection | Product/booking system |
| Smart Process | Бронирование | Coordination/read-only status projection | Booking/GDS/channel system |
| Smart Process | Поездка | Operational itinerary/service projection | Booking/operations system |
| Smart Process | Партнёр | Governed partner projection/relationship | ERP/partner master |
| Smart Process | Претензия | Restricted complaint lifecycle | Bitrix24/legal system |

**Факт Bitrix24:** CRM, задачи, Smart Processes, роботы, триггеры и бизнес-процессы описаны официальными материалами. Конкретные поля, связи, права и automation проверяются на целевом портале.

**Решение архитектора:** интеграционные объекты содержат `source_system`, `external_id`, `source_updated_at`, `sync_status` и correlation ID. CRM не редактирует свободно availability, supplier confirmation, ticket/voucher status, payment or refund fact.

```text
Site/Calls/Mail -> Lead/Enquiry -> Contact/Company -> Deal -> Offer
                         |                              |
                         v                              v
                   Travel Request                Booking Request
                         |                              |
                         +---- Travelers/Group          v
                                                   Booking <-> GDS/Booking System
                                                      |
                         +----------------------------+-------------------+
                         v                            v                   v
                       Trip                    Payment/Refund         Documents
                         |                            |                   |
                  Support/Complaint                1C/PSP              DMS/EDO

BI <- reconciled datasets <- Bitrix24 + Booking/1C
```

### Правила целостности

- master назначается для клиента, traveler role, product, offer price, availability, booking, document, payment and refund;
- purchaser, traveler и corporate approver не смешиваются в одном поле;
- повтор события не создаёт duplicate Deal, booking, traveler relation, payment or complaint;
- изменение брони/поездки создаёт version/change event, а не переписывает историю;
- паспортные и иные избыточно чувствительные данные не копируются в CRM без обоснованной модели;
- bulk operations используют dry-run, sample, отчёт, reconciliation and rollback.

## 4. Customer Journey

| Этап | Цель | Минимальные данные | Контроль |
|---|---|---|---|
| Реклама | Зафиксировать источник спроса | Campaign/channel | Attribution/consent |
| Запрос клиента | Зарегистрировать потребность | Dates range, direction category, participants count, budget band | Dedup/owner/SLA |
| Подбор тура | Получить актуальные варианты | Product/version/availability timestamp | Source freshness |
| Предложение | Зафиксировать версию условий | Offer ID/version/validity/currency | Pre-send validation |
| Бронирование | Создать request и получить подтверждение | Booking request/source IDs, participants refs | Idempotency/source confirmation |
| Оплата | Получить authoritative financial status | Invoice/payment refs | PSP/1С reconciliation |
| Поездка | Сопровождать подтверждённые услуги | Trip/service refs, dates, support owner | Controlled communication |
| Отзывы | Получить добровольный feedback | Trip relation/result | Consent/minimization |
| Повторная покупка | Создать новую opportunity | Approved eligibility/context | No overwrite of prior trip |

```text
Реклама -> Запрос клиента -> Подбор тура -> Предложение -> Бронирование
 -> Оплата -> Поездка -> Отзывы -> Повторная покупка

Branches: no availability / supplier declined / payment failed / change /
cancellation / refund / disruption / complaint.
```

## 5. Travel Data Model

| Сущность | Минимальные поля | Контроль |
|---|---|---|
| Турист | External customer/traveler ID, role, permitted contacts, preferences if justified | No passport data in general CRM profile |
| Группа/семья | Group ID, trip context, participant relations | Not a shared Contact |
| Компания | Company external ID, contract/approver relations | Name is not sole key |
| Тур | Product/version ID, direction, date window, category, price projection, terms ref | Product/booking master |
| Бронирование | Booking ID/version, client/group, product/services, status, amount/currency | Booking master/read-only status |
| Поездка | Trip ID, booking, service segments, plan/status | Operations/booking source |
| Партнёр | Partner ID/type/status, contract ref | ERP/partner master |
| Платёж | Payment/refund external ID, booking/invoice, amount/currency/status | PSP/1С master |

```text
Customer N ---- M Traveler Role M ---- 1 Travel Group ---- 1 Booking
   |                                                   |       |
   +---- N Deal/Request ---- N Offer                   |       +---- N Payment/Refund
                                                       |       +---- N Document Ref
                                                       v
                                                    Trip ---- N Service Segment ---- 1 Partner
                                                       |
                                                       +---- N Support/Complaint
```

### Кардинальности

| Связь | Кардинальность | Правило |
|---|---:|---|
| Customer → Deal | 1:N | Каждая opportunity отдельна |
| Booking ↔ Traveler | M:N | Контекстная роль участника обязательна |
| Deal → Offer | 1:N | Offer versions immutable |
| Booking → Trip | 1:1..N | Depends on approved segmentation |
| Booking → Service/Partner | 1:N | Each supplier segment has source ref |
| Booking → Payment/Refund | 1:N | Financial events never overwritten |

## 6. Sales Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новая заявка | Зафиксировать спрос | Channel, request summary, owner | Routing/SLA/dedup |
| Квалификация | Подтвердить формат поездки | B2C/B2B, participants, dates, budget, next step | Task/escalation |
| Подбор направления | Найти актуальные варианты | Direction/product refs, timestamp | Source lookup/freshness |
| Подготовка предложения | Версионировать условия | Offer ID/version/validity/currency | Approval/pre-send check |
| Согласование | Закрыть вопросы | Open items, decision owner/date | Follow-up/escalation |
| Бронирование | Получить supplier confirmation | Booking request/booking IDs/status | Idempotent async handling |
| Оплата | Получить authoritative payment | Invoice/payment IDs/status | PSP/1С reconciliation |
| Поездка | Передать сопровождению | Trip/services/support owner | Handoff/checklist |
| Повторная продажа | Создать новую opportunity | Consent/eligibility/new scope | New Deal/duplicate check |

```text
Новая заявка -> Квалификация -> Подбор направления -> Подготовка предложения
 -> Согласование -> Бронирование -> Оплата -> Поездка -> Повторная продажа
```

Стадии отражают подтверждённое состояние. Сделка не закрывается успешно по локальной отметке без supplier/booking confirmation согласно принятому gate. Re-entry не создаёт повторные задачи/сообщения.

## 7. Tour Product Management

| Объект | Модель | Контроль |
|---|---|---|
| Направление | Governed geography/category reference | No sensitive inference |
| Тур | Stable product ID | Product system master |
| Версия | Effective dates/status | Immutable version |
| Категория | Package/individual/corporate/other approved | Data owner |
| Цена | Amount/currency/validity projection | Booking/pricing source |
| Сезонность | Availability window/reference | Source timestamp |
| Дополнительная услуга | Compatible service/product relation | Eligibility/version |

CRM хранит минимальную проекцию для продажи и аналитики. Availability, dynamic price, inventory, taxes/fees and supplier conditions проверяются в профильной системе перед предложением и бронью.

## 8. Booking Management

| Операция | CRM | Booking/GDS/channel system | Контроль |
|---|---|---|---|
| Создание | Request/projection/task | Authoritative booking | Idempotency/source ID |
| Подтверждение | Read-only status | Supplier confirmation/master | Timestamp/version |
| Изменение | Change request/history | Authoritative amendment | Old/new refs/price impact |
| Отмена | Cancellation request/status | Supplier cancellation | Terms/refund linkage |
| Статус | Operational projection | Master | Freshness/reconciliation |
| Завершение | Trip/customer follow-up | Service completion facts | Open exception check |

```text
Offer -> Booking Request -> Pending Supplier -> Confirmed Booking -> Trip
                 |                 |                    |
                 v                 v                    v
              Rejected          Timeout              Change/Cancel -> Refund workflow
```

Нельзя считать бронь подтверждённой, изменённой или отменённой только по CRM-stage. Booking reference, version, source timestamp and supplier result обязательны.

## 9. Partner Management

| Тип партнёра | Данные | Master | Контроль |
|---|---|---|---|
| Отель | Partner/property refs, service status | Booking/partner system | Version/availability |
| Перевозчик | Partner/service segment refs | GDS/booking system | Confirmation/status |
| Гид/оператор | Partner/service order ref | Partner/operations system | Assignment/acceptance |
| Агентский партнёр | Company, channel, contract/referral IDs | ERP/partner system | Attribution/commission source |
| Договор | Document ID/version/status | DMS/EDO | Access/effective dates |

Партнёр не получает прямой CRM-доступ без отдельного security design. Коммерческие условия, комиссии и расчёты подтверждаются ERP/contract source; partner rating не формируется из непроверенных комментариев.

## 10. Customer Support

| Тип | Назначение | Приоритет/маршрут | Контроль |
|---|---|---|---|
| Вопрос | Информация до/после брони | Sales/support queue | Verified booking context |
| Сопровождение | Изменение/разъяснение по trip | Trip support owner | Supplier confirmation |
| Претензия | Formal complaint | Restricted escalation | Separate lifecycle/audit |
| Экстренное обращение | Time-critical assistance request | Approved emergency route | Human ownership/alternate channel |
| Refund question | Status communication | Finance/booking relation | No unconfirmed promise |

```text
Customer Contact -> Verify identity/context -> Classify
 -> Normal Case / Complaint / Emergency Route
 -> Partner/Booking/Finance coordination -> Customer update -> Confirmed closure
```

CRM не является единственным emergency channel. Резервные контакты, часы, ownership and escalation должны быть утверждены организацией и проверены отдельным runbook.

## 11. Rights Model

| Роль | Клиенты | Туры | Брони | Документы | Аналитика |
|---|---|---|---|---|---|
| Менеджер продаж | Assigned/team | Current sales projection | Assigned minimum | Sales refs | Personal/team |
| Специалист сопровождения | Verified trip participants | Required service view | Assigned active trips | Need-to-know | Operations |
| Руководитель | Governed unit/portfolio | Read/approve | Team scope | As role requires | Unit dashboard |
| Партнёрский менеджер | Partner context/minimal clients | Product/partner refs | Aggregate/service exceptions | Partner contracts | Partner aggregates |
| Бухгалтер | Billing identity minimum | Product refs | Invoice/booking refs | Financial docs | Finance |
| Директор | Governed portfolio | Read | Aggregate/oversight | Approved | Executive |
| Администратор | Technical configuration | No business entitlement by default | No booking content entitlement | No automatic content | Config/dataset as needed |

Принципы: deny-by-default, least privilege, trip/portfolio/function scopes, time-bound access, recertification, export controls and negative tests. Паспортные и платёжные данные не должны быть доступны ролям по умолчанию.

## 12. Automation Architecture

| Автоматизация | Trigger | Действие | Защита |
|---|---|---|---|
| Intake SLA | First valid request | Assign/task/timer | Idempotency marker |
| Offer expiry | Validity approaching | Review/follow-up task | Current version/status |
| Booking status | Authoritative callback/poll | Update projection/task | Event order/idempotency |
| Payment status | PSP/1С event | Update read-only projection | Amount/currency/reconcile |
| Trip reminder | Confirmed current booking | Permitted notification/task | Timezone/change/cancel check |
| Emergency case | Approved category | Route/escalate immediately | Human owner/ack |
| Repeat sale | Eligible prior trip | New Deal/task | Consent/suppression/duplicate check |

```text
Event -> Validate source/version/rights -> Check idempotency -> Evaluate rule
 -> Bounded action -> Audit result -> Exception queue/escalation
```

Автоматизация не подтверждает availability, booking, payment, refund or safe travel outcome без authoritative source и утверждённого human/system gate.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| Сайт | Enquiries/search context | CRM form/official app/webhook/REST | Validation, consent, dedup |
| Booking/GDS | Products, availability, booking/change/cancel statuses | Documented API/middleware | IDs, ordering, idempotency, DLQ |
| Платёжная система | Payment/refund events | Provider API/middleware | Signature/auth, amount/currency/reconcile |
| Телефония | Call events/CRM linkage | Bitrix24 telephony/SIP/REST app | Rights, missed-call queue, correlation |
| Почта | Authorized correspondence | Built-in connection/approved integration | Mailbox ownership/retention |
| 1С | Invoices/payments/partner accounting refs | Supported integration/middleware | Stable IDs/control totals |
| BI | Governed datasets/aggregates | BI Constructor or approved ETL | Grain, masking, freshness |

```text
Source Event -> AuthN/AuthZ -> Schema validation -> Correlation/idempotency
 -> Master ownership -> Bitrix24 write or quarantine -> Reconciliation/monitoring
```

Секреты хранятся вне Git; retry ограничен; неизвестные значения карантинируются; двустороннее редактирование требует field ownership/conflict policy. REST methods/scopes/limits сверяются с официальной документацией.

## 14. Marketing Automation

| Сценарий | Условие | Действие | Контроль |
|---|---|---|---|
| Сегментация | Approved minimal preference/history data | Governed audience | No passport/health inference |
| Персональное предложение | Current product/eligibility | New Deal/campaign | Consent/frequency/current price |
| Сезонная кампания | Product version/availability window | Approved communication | Sample/suppression |
| Лояльность | Authoritative membership/status if used | Permitted benefit message | Loyalty master/reconciliation |
| Повторная покупка | Completed eligible trip | New opportunity | No overwrite/duplicate |

Персонализация не должна раскрывать маршрут другим участникам или делать чувствительные выводы. Перед отправкой проверяются consent, suppression, product validity, current booking state and communication purpose.

## 15. Analytics

| Отчёт | Grain | Показатели | Контроль качества |
|---|---|---|---|
| Продажи туров | Confirmed booking | Count/revenue projection by product/channel | Booking/1С reconcile |
| Эффективность менеджеров | Deal/owner/period | Conversion, cycle time, next-step discipline | Assignment changes/context |
| Популярность направлений | Booking/service segment | Confirmed travelers/bookings by direction | Cancelled excluded by rule |
| Повторные покупки | Customer + confirmed bookings | Repeat rate/interval/cohort | Stable identity |
| Прибыльность | Booking/product/period | Reconciled revenue/cost/margin | Finance methodology |
| Эффективность рекламы | Qualified enquiry/booking | Conversion/cost/attributed revenue | Attribution window |

```text
CRM + Booking/GDS + PSP/1C -> Conformed IDs -> DQ/Reconciliation
 -> Governed Metric Layer -> Role-based Dashboard
```

Метрика имеет owner, formula, grain, time window, timezone, currency/FX rule, exclusions, source, freshness SLA and control total. Deal amount не заменяет финансовый факт.

## 16. Security Architecture

| Контроль | Реализация | Проверка |
|---|---|---|
| Минимизация | Only necessary client/trip projections | Field/file inventory |
| Trip/group access | Explicit assignment/need-to-know | Cross-trip negative tests |
| Sensitive documents | Protected DMS refs, not general CRM fields | Unauthorized/expired access |
| Payment data | Tokenized/reference-only where applicable | No card data scan |
| Аудит | CRM history + masked integration logs | Correlation trace |
| Export | Approval/minimum fields/protected target | Export test |
| Secrets | Environment/secret store/rotation | Scan/revoke drill |
| Retention | Approved archive/delete/hold policy | Periodic review |

**Cloud:** нет server/PHP/DB access; используются штатные права/история и внешние логи. **On-Premise:** дополнительно защищаются сервер, PHP, БД, файлы, cron, backup и журналы; ядро не изменяется.

## 17. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Заявка с сайта | Valid synthetic payload | One request, owner/SLA |
| 2 | Повтор webhook | Same event ID | No duplicate |
| 3 | Невалидный контакт | Missing required value | Controlled error/quarantine |
| 4 | Семейная поездка | Three synthetic travelers | Separate roles/group, no shared Contact |
| 5 | B2B request | Company/approver/travelers | Correct contextual relations |
| 6 | Подбор тура | Current product/version | Fresh options only |
| 7 | Expired offer | Validity passed | Booking blocked/refreshed |
| 8 | Booking confirmation | Supplier event | One confirmed projection |
| 9 | Booking retry | Accepted then timeout | Idempotent result |
| 10 | Booking change | New version/price | History preserved/reconciled |
| 11 | Cancellation/refund | Source events | Statuses linked; no false promise |
| 12 | Payment mismatch | Different synthetic total | Incident/reconciliation |
| 13 | Emergency case | Approved category | Immediate owner/ack/escalation |
| 14 | Cross-trip rights | Unauthorized user | Client/booking/docs unavailable |
| 15 | Trip reminder after change | Changed dates | Current date used/no stale message |
| 16 | Consent withdrawal | Suppression event | Marketing stops |
| 17 | Repeat sale | Completed trip | New Deal; history preserved |
| 18 | BI reconciliation | Synthetic totals | Threshold pass or incident |
| 19 | Cloud/On-Premise | Capability matrix | Unsupported assumption detected |
| 20 | Rollback | Failed release | Verified previous state restored |

Тесты используют только synthetic travelers, groups, products, bookings and payments. Проверяются positive, negative, boundary, concurrency, retry, rights, audit and recovery.

## 18. Deployment

| Этап | Результат | Gate |
|---|---|---|
| Discovery | AS IS, systems, tariff, volumes, data classes | Business/security approval |
| Архитектура | TO BE, master matrix, rights, ADR, NFR | Architecture review |
| CRM настройка | Entities, stages, roles, automation in test scope | Configuration review |
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
| Необработанная заявка | Intake SLA/no ack | Assign/escalate | Sales operations |
| Ошибка бронирования | Failed/timeout/DLQ | Incident/replay after fix | Booking integration owner |
| Просроченная задача | Deadline breached | Owner/manager escalation | Process owner |
| Payment/refund mismatch | PSP/1С/booking difference | Finance incident | Finance owner |
| Обращение без owner | Unassigned/SLA | Route/escalate | Support owner |
| Stale booking | Freshness exceeded | Mark stale/stop unsafe action | Booking owner |
| Emergency ack absent | Short approved threshold | Alternate route/escalate | Duty manager |
| BI stale/mismatch | Dataset SLA/control total | Data incident | BI owner |

Alert содержит system, object reference, correlation ID, time, severity, owner and runbook; без маршрута, документов, секретов и лишних персональных данных.

## 20. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Дубли клиентов | Multiple channels | Fragmented history | Stable IDs/dedup |
| 2 | False merge | Shared family contact | Wrong trip disclosure | Conservative review/context roles |
| 3 | Lost enquiry | No ack | Lost sale | Intake reconciliation |
| 4 | Duplicate Deal | Retry | Funnel distortion | Idempotency |
| 5 | Family stored as one Contact | Simplification | Mixed data | Group/participant model |
| 6 | B2B approver equals traveler | Flat role | Wrong communication | Contextual roles |
| 7 | Passport data in CRM | Convenience | Sensitive leak | Minimize/protected system |
| 8 | Route in alert/log | Full payload | Privacy exposure | Masking/minimal alert |
| 9 | Stale availability | Cached data | Failed booking | Freshness/source check |
| 10 | Stale price | No validity/version | Dispute | Offer version/expiry |
| 11 | Currency omitted | Local assumption | Wrong amount | Currency/FX rule |
| 12 | Duplicate booking | Retry | Double cost | Source request ID |
| 13 | CRM marks confirmed | Manual status | False promise | Supplier confirmation |
| 14 | Event order ignored | Async updates | Old status wins | Version/order guard |
| 15 | Change overwrites booking | Flat model | Lost history | Amendment/version |
| 16 | Cancel not confirmed | CRM-only state | Ongoing liability | Supplier reconciliation |
| 17 | Refund promised locally | No finance source | Client dispute | PSP/1С confirmation |
| 18 | Partner terms stale | No contract version | Margin/service error | Effective contract refs |
| 19 | Partner gets broad CRM access | Shortcut | Data leak | No direct access/default deny |
| 20 | Support case lacks booking link | Manual intake | Slow response | Stable relation |
| 21 | Emergency relies on one channel | No fallback | Missed urgent case | Alternate route/runbook |
| 22 | Emergency auto-closed | Status automation | Unresolved traveler | Human confirmation |
| 23 | SLA timezone wrong | Travel zones | Missed deadline | Explicit timezone |
| 24 | Trip reminder stale | Booking changed | Wrong guidance | Pre-send source check |
| 25 | Marketing without consent | Suppression ignored | Compliance risk | Consent check |
| 26 | Sensitive personalization | Uncontrolled inference | Harm/reputation | Approved minimal attributes |
| 27 | Cross-trip access | Broad role | Data disclosure | Trip/function scopes |
| 28 | Admin as super-reader | Role confusion | Uncontrolled access | Separate privileges |
| 29 | Export unrestricted | Default rights | Mass leak | Approval/minimum fields |
| 30 | Secret in Git | Real example | Compromise | Secret store/rotation |
| 31 | Broad token | One integration user | Large blast radius | Least privilege |
| 32 | Raw payload logged | Debugging | PII leak | Masking |
| 33 | Infinite retry | No DLQ | Load/outage | Bounded backoff |
| 34 | Schema drift | Booking API change | Corrupt data | Contract validation |
| 35 | No correlation ID | Fragmented logs | Slow RCA | End-to-end trace |
| 36 | Free bidirectional edits | No master | Ping-pong | Field ownership |
| 37 | Payment not reconciled | HTTP success trusted | Hidden mismatch | Control totals |
| 38 | Booking count equals travelers | Wrong grain | Inflated sales | Metric contract |
| 39 | Cancellations included in popularity | Bad filter | False trends | Status inclusion rules |
| 40 | Margin from Deal amount | Missing supplier costs | False KPI | Finance reconciliation |
| 41 | Real travelers in tests | Prod copy | Data breach | Synthetic data |
| 42 | Migration without dry-run | Urgency | Mass corruption | Sample/report/rollback |
| 43 | No count reconciliation | Success response trusted | Silent loss | Counts/sample checks |
| 44 | Tariff not checked | Assumption | Feature unavailable | Capability assessment |
| 45 | Cloud treated as server | DB/PHP expected | Impossible design | Cloud boundary |
| 46 | Core modified On-Premise | Quick fix | Upgrade/security failure | Extensions only |
| 47 | Release lacks rights tests | Happy path only | Exposure | Negative suite |
| 48 | No rollback | Success assumed | Long outage | Tested rollback |
| 49 | Alert lacks owner | Notification only | Unresolved incident | RACI/runbook |
| 50 | No baseline | AS IS unmeasured | Effect unprovable | Baseline/KPI |
| 51 | No change register | Oral changes | Poor support | Versioned register |
| 52 | Partner outage not monitored | External dependency | Silent booking failure | Dependency monitoring |
| 53 | Season load not tested | Average-volume design | SLA collapse | Peak/load testing |
| 54 | Loyalty status local | No loyalty master | Wrong benefit | Authoritative projection |
| 55 | Archive confused with backup | No restore design | Failed recovery | Separate policies |

## 21. ADR

| ADR | Решение | Альтернатива | Обоснование | Последствие |
|---|---|---|---|---|
| ADR-TRV-001 | Tourist request separated from Deal details where lifecycle needs it | Only Deal fields | Structured requirements/reuse | Link and rights needed |
| ADR-TRV-002 | Tour is versioned product projection | CRM as product master | Availability/price belong to booking system | Freshness integration |
| ADR-TRV-003 | Booking is authoritative external projection | CRM as booking engine | Supplier confirmation/change history | Idempotency/reconciliation |
| ADR-TRV-004 | Travelers linked by contextual group roles | One family Contact | Preserves identities and privacy | Participant model required |
| ADR-TRV-005 | Partners remain Company/master projections without broad portal access | Direct shared CRM access | Least privilege | Controlled integration/communication |
| ADR-TRV-006 | Documents stored as protected DMS refs | Files/comments in CRM | Version/access/retention | Repository integration |
| ADR-TRV-007 | Middleware/contracts govern integrations | Free point-to-point webhooks | Idempotency/monitoring | Platform owner required |
| ADR-TRV-008 | BI uses reconciled conformed datasets | Raw CRM reports | Booking/finance grains differ | Metric/DQ ownership |

ADR получает owner, status `proposed/accepted/superseded`, date, context, alternatives, consequences, portal/version and evidence.

## 22. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Нет Discovery | Immediate setup | Wrong scope | AS IS/stakeholders |
| 2 | Deployment type unknown | Template design | Unsupported solution | Cloud/On-Premise check |
| 3 | Tariff not verified | Assumption | Blocked release | Verify portal |
| 4 | CRM declared booking/GDS | Scope confusion | False booking facts | Architecture boundary |
| 5 | All requests are Leads | Universal lead | Support mixed with sales | Classification |
| 6 | Leads disabled without ADR | Fashion | Unqualified demand mixed | Lead decision |
| 7 | Contact/Company mixed | Flat client field | Wrong B2B relations | Native entities |
| 8 | Family is one Contact | Shortcut | Mixed identities | Traveler group relations |
| 9 | Child/participant stored in comment | No model | Missing rights/history | Contextual participant |
| 10 | Name is integration key | No external ID | Wrong match | Stable IDs |
| 11 | Phone is sole dedup key | Convenience | False merge | Conservative matching |
| 12 | Passport copied to CRM | Easy access | Sensitive leak | Protected specialist system |
| 13 | One Deal for all trips | Entity saving | No repeat analytics | Deal per opportunity |
| 14 | Booking only in Deal fields | No lifecycle/version | Lost changes | Booking projection |
| 15 | Tour and version mixed | No effective dates | Wrong offer | Versioned product |
| 16 | Price copied indefinitely | Cache | Client dispute | Validity/freshness |
| 17 | Availability guessed | No source query | Failed booking | Authoritative check |
| 18 | Stages describe tasks | Process mistake | Ambiguous state | State-based stages |
| 19 | Too many stages | Every action is stage | Low adoption | Tasks/checklists |
| 20 | Fields mandatory too early | Maximum control | Fake data | Stage requiredness |
| 21 | Loss reason in comment | No dictionary | No analytics | Governed codes |
| 22 | No next step | Stage-only control | Stalled Deal | Activity policy |
| 23 | Deal success before confirmation | KPI pressure | False sale | Booking gate |
| 24 | Re-entry duplicates tasks | Stateless robot | Fake workload | Idempotency marker |
| 25 | Robots loop stages | Mutual transitions | Automation storm | Loop guard |
| 26 | Offer overwritten | One document | Wrong conditions | Immutable versions |
| 27 | Currency/validity absent | Local assumption | Wrong quote | Required dimensions |
| 28 | Booking has no request ID | Local title | Duplicate booking | Source idempotency key |
| 29 | Confirmation manual | No integration | False promise | Supplier event |
| 30 | Amendment overwrites original | Flat status | Audit loss | Change/version model |
| 31 | Cancellation local only | No supplier sync | Continuing liability | Reconciliation |
| 32 | Refund equals cancellation | Concepts mixed | Wrong finance status | Separate refund lifecycle |
| 33 | Payment checkbox manual | Convenience | False payment | PSP/1С source |
| 34 | Card/payment details stored | Overcollection | Security incident | Token/reference only |
| 35 | Partner duplicated by service | No identity | Fragmented contracts | Stable partner ID |
| 36 | Partner sees full CRM | Collaboration shortcut | Data leak | Controlled access/integration |
| 37 | Commission calculated in robot | Missing ERP | Finance mismatch | ERP/accounting master |
| 38 | Documents uploaded to comments | Convenience | No version/access | Protected DMS |
| 39 | Document link public | Easy sharing | Disclosure | Authenticated expiring access |
| 40 | Trip dates use portal timezone | No travel-zone model | Wrong reminders | Explicit timezone |
| 41 | Reminder ignores amendment | No current-state check | Wrong message | Source validation |
| 42 | Support has no booking link | Manual intake | Slow resolution | Stable relation |
| 43 | Complaint mixed with question | One queue | Wrong SLA | Separate type/route |
| 44 | Emergency uses normal SLA | No critical route | Delayed help | Emergency runbook |
| 45 | Emergency auto-response considered ack | Automation optimism | No human owner | Human acceptance |
| 46 | Manager sees other trips | Broad role | Privacy leak | Assignment scope |
| 47 | Accountant sees itinerary | Shared entity rights | Excess access | Finance projection |
| 48 | Admin equals business reader | Role confusion | Uncontrolled access | Separate privileges |
| 49 | Temporary rights never expire | Manual grant | Residual risk | Expiry/recertification |
| 50 | Rights only positive-tested | Happy path | Hidden cross-access | Negative tests |
| 51 | Export unrestricted | Defaults | Mass leak | Approval/minimum fields |
| 52 | Secret in README | Real example | Compromise | Placeholder/secret store |
| 53 | Webhook committed | Convenience | Unauthorized calls | Environment/rotation |
| 54 | Token overprivileged | One credential | Large blast radius | Least privilege |
| 55 | Raw payload logged | Debugging | Traveler data leak | Masking |
| 56 | Retry duplicates booking | No idempotency | Double cost | Source event key |
| 57 | Infinite retry | No DLQ | Load/outage | Bounded retry |
| 58 | Out-of-order ignored | Async not designed | Stale status | Version/order guard |
| 59 | Schema not validated | Trust source | Corrupt data | Contract validation |
| 60 | No correlation ID | Fragmented logs | Slow RCA | Trace ID |
| 61 | Bidirectional edits free | No master | Ping-pong | Field ownership |
| 62 | Batch ignores limits | Premature optimization | Partial errors | Official checks/report |
| 63 | Migration has no inventory | Unknown sources | Missing objects | Source/volume report |
| 64 | Import without dry-run | Urgency | Mass corruption | Sample/dry-run |
| 65 | Real travelers in UAT | Prod copy | Privacy breach | Synthetic data |
| 66 | No count/sample reconciliation | HTTP success trusted | Silent loss | Control report |
| 67 | BI counts travelers as bookings | Wrong grain | Inflated sales | Metric contract |
| 68 | Cancelled trips in repeat metric | Bad filter | False retention | Status/cohort rule |
| 69 | Margin from Deal amount | Missing supplier costs | False profit | Finance reconciliation |
| 70 | Attribution window absent | Last-touch assumption | False channel KPI | Metric definition |
| 71 | Old fields removed immediately | Cleanup | Broken dependencies | Deprecation scan |
| 72 | Release without smoke | Time pressure | Production defect | Checklist |
| 73 | Rollback untested | Paper plan | Long outage | Restore drill |
| 74 | Monitoring only HTTP uptime | Technical focus | Bookings lost | Business signals |
| 75 | Alert includes itinerary/passport | Full payload | Disclosure | Minimal alert |
| 76 | Alert has no owner | Notification only | Unresolved incident | RACI/runbook |
| 77 | Cloud assumes DB/PHP access | On-Premise habit | Impossible design | Cloud boundary |
| 78 | On-Premise core modified | Quick fix | Upgrade/security failure | Extensions |
| 79 | No hypercare/change register | Immediate closure | Incident/config drift | Stabilization/docs |
| 80 | Done means configured | No evidence | Disputed result | Tested acceptance criteria |

## 23. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Confirmed AS IS/scope | Stakeholders, systems, tariff, volumes, data classes | Facts/open questions approved |
| 2. Архитектура | Approved TO BE | Entity model, master matrix, rights, NFR, ADR | Architecture/security review |
| 3. CRM настройка | Governed intake/sales | Fields, stages, roles, automation register | Synthetic paths pass |
| 4. Туристические процессы | Product/booking/trip/support projections | IDs, versions, participant roles, SLA | End-to-end UAT passes |
| 5. Интеграции | Reliable exchange | Contracts, secrets, retry, idempotency, monitoring | Failure/replay/reconcile pass |
| 6. Тестирование | Evidence-based quality | Test plan, synthetic data, rights suite, UAT | Critical defects closed/sign-off |
| 7. Запуск | Controlled release | Cutover, backup/export, rollback, support | Smoke/reconciliation pass |
| 8. Оптимизация | Measured improvement | Baseline, DQ/incident/change backlog | KPI change verified |

### Критерии готовности

- подтверждены Cloud/On-Premise, тариф/редакция, версия, регион и права;
- утверждены master-system matrix, entity/participant model, rights and ADR;
- проверены права ролей позитивными и негативными тестами;
- выполнены synthetic tests, UAT, migration dry-run and reconciliation;
- секреты вынесены из Git, logs/alerts не содержат чувствительных данных;
- утверждены release, rollback, emergency, monitoring, retention and support procedures;
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
2. B2C/B2B model, channels, lead usage, product types and responsible owners.
3. Master клиента, traveler roles, product, availability, price, booking, payment/refund and documents.
4. Booking/GDS/PSP/1С/DMS APIs, stable IDs, event order, volumes, SLA and replay.
5. Допустимый состав персональных, паспортных, маршрутных и платёжных данных в CRM.
6. Consent/suppression, retention/delete, export, audit and emergency policies.
7. Booking/change/cancel/refund gates, calendars, timezone and escalation rules.
8. Partner access, contracts, commissions, service confirmation and fallback procedures.
9. BI grain, currency/FX, metric owners, control totals and DQ thresholds.
10. Test portal/pipeline, release window, downtime, rollback and support RACI.
