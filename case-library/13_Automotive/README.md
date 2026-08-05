# Automotive Customer Lifecycle CRM Architecture Case

Архитектурный кейс для синтетической автомобильной компании, которая продаёт новые автомобили и автомобили с пробегом, организует тест-драйвы, сервис, гарантийные обращения, запчасти и повторные продажи. Документ является референсной инженерной моделью, а не готовой конфигурацией для переноса без Discovery.

> **Граница безопасности:** реальные дилерские сети, клиенты, сотрудники, VIN, государственные номера, телефоны, email, адреса, CRM ID, webhook URL, токены, договоры и выгрузки не используются. Примеры и идентификаторы синтетические.

> **Архитектурная граница:** Bitrix24 отвечает за CRM-профиль, коммуникации, коммерческие возможности, обращения и оркестрацию работ. DMS/ERP/1С остаются master-system автомобилей, складских, ценовых, финансовых и сервисных фактов согласно утверждённой матрице владения. CRM не подменяет DMS, склад, бухгалтерский или диагностический контур.

Обозначения:

- **факт Bitrix24** — возможность подтверждена официальным источником из `knowledge/SOURCES.md`;
- **решение архитектора** — предлагаемая модель кейса, требующая утверждения на Discovery;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа/редакции, версии, прав, региона, приложения или внешней системы.

## 1. Business Context

### Модель бизнеса

Референсная организация включает несколько дилерских центров, отделы продаж новых автомобилей и автомобилей с пробегом, сервисные зоны, склад запчастей и централизованный маркетинг. Клиент может перейти между каналами и направлениями, но сохраняет единый допустимый профиль взаимодействий.

| Направление | Коммерческий объект | Основной результат | Предпочтительный master |
|---|---|---|---|
| Новые автомобили | Автомобиль/комплектация и сделка | Продажа и передача | DMS/ERP — stock/vehicle; Bitrix24 — opportunity |
| Автомобили с пробегом | Конкретный экземпляр | Оценка, резерв, продажа | DMS/used-car system |
| Сервис | Сервисный заказ | Выполненные работы | DMS/service system |
| Запчасти | Номенклатура и заказ | Выдача/установка | ERP/1С/WMS |
| Гарантия | Гарантийный случай | Решение и закрытие | DMS/warranty system; CRM — коммуникация |
| Маркетинг | Кампания и отклик | Квалифицированный спрос | Marketing source + Bitrix24 |

### Цели CRM

1. Не терять заявки и обеспечить владельца следующего шага.
2. Создать разрешённый единый профиль клиента без неконтролируемого объединения.
3. Связать продажу, автомобиль, тест-драйв, сервис и повторное обращение устойчивыми ID.
4. Разделить доступ дилерских центров и функциональных ролей.
5. Автоматизировать контроль сроков без циклов и скрытых изменений фактов.
6. Получать сверяемую аналитику продаж, тест-драйвов и сервиса.

```text
Marketing channels -> Enquiry -> Sales opportunity -> Vehicle handover
                               |                       |
                               v                       v
                          Test drive              Ownership cycle
                                                       |
                                                       v
                               Repeat sale <- Service / Warranty
```

## 2. AS IS Problems

| Проблема | Проявление | Причина | Целевой контроль |
|---|---|---|---|
| Нет единого профиля | Продажи и сервис создают разные карточки | Нет identity policy | Stable external IDs и ручная очередь спорных совпадений |
| Теряются заявки | Нет владельца и срока реакции | Каналы не нормализованы | Intake, routing, SLA и эскалация |
| Разрыв продаж и сервиса | Сервис не знает контекст владения | Нет связи customer–vehicle | Governed vehicle relation |
| Нет истории автомобиля | События хранятся комментариями | Нет vehicle/service model | Структурированные сущности и source IDs |
| Слабая аналитика | Метрики считаются по-разному | Нет словаря и grain | Metric dictionary и reconciliation |
| Дубли сделок | Повторный webhook создаёт запись | Нет идемпотентности | Source event ID и upsert policy |
| Недостоверный склад | CRM трактуется как DMS | Не назначен master | Read-only projection с freshness timestamp |
| Просроченные тест-драйвы | Расписание ведётся вручную | Нет конфликта ресурсов | Availability check и подтверждение |
| Неуправляемая гарантия | Обращение смешано с ремонтом | Нет lifecycle | Отдельный warranty case и service order |
| Избыточные права | Центры видят чужих клиентов | Общие роли | Scope by center/function и negative tests |

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Лид | Неквалифицированное обращение, если lead-based model утверждена | Bitrix24 |
| CRM | Контакт | Физическое лицо и минимальные разрешённые контакты | Bitrix24/MDM по ADR |
| CRM | Компания | Корпоративный покупатель, партнёр или страховая организация | Bitrix24/ERP |
| CRM | Сделка | Отдельная возможность продажи автомобиля/услуги | Bitrix24 |
| CRM/Service | Обращение | Клиентский вопрос или жалоба | Bitrix24 |
| Smart Process | Автомобиль | Управляемая CRM-проекция экземпляра | DMS |
| Smart Process | Тест-драйв | Запись, ресурс, результат | Bitrix24/DMS scheduling по ADR |
| Smart Process | Сервисный заказ | CRM-проекция обслуживания | DMS/service system |
| Smart Process | Ремонт | Координация ремонтного цикла при необходимости | DMS/service system |
| Smart Process | Гарантийный случай | Коммуникация и контроль решения | Warranty/DMS |
| Smart Process | Запчасть | Только допустимая проекция номенклатуры | ERP/1С/WMS |

**Факт Bitrix24:** CRM, смарт-процессы, роботы, триггеры и бизнес-процессы описаны официальной документацией. Доступность конкретных настроек, полей, связей, прав и автоматизации проверяется на целевом портале.

**Решение архитектора:** все интеграционные записи получают `source_system`, `external_id`, `source_updated_at`, `sync_status` и correlation ID. Финансовые, складские, диагностические и гарантийные факты не редактируются свободно в CRM.

```text
Website/Calls/Forms
        |
        v
Lead/Enquiry -> Contact/Company -> Deal -> Vehicle handover
                     |              |             |
                     |              v             v
                     |          Test Drive     Ownership
                     |                            |
                     +---- Service Case <---- Service Order <---- DMS
                                      |              |
                                      v              v
                                  Warranty       Parts <- ERP/1C/WMS

BI <- governed datasets <- Bitrix24 + DMS/ERP reconciliation
```

### Правила целостности

- один источник истины назначается для клиента, автомобиля, цены, резерва, оплаты, сервиса, гарантии и остатка;
- VIN рассматривается как чувствительный идентификатор и хранится только при обоснованной необходимости и с ограниченными правами;
- повтор события не создаёт дубликат клиента, сделки, тест-драйва, автомобиля или сервисного заказа;
- смена владельца автомобиля не переписывает историческую связь продажи;
- bulk-операции имеют dry-run, тестовую выборку, отчёт, reconciliation и план отката;
- Cloud/On-Premise и тарифные различия фиксируются до дизайна.

## 4. Customer Journey

| Этап | Цель | Минимальные данные | Контроль |
|---|---|---|---|
| Реклама | Зафиксировать источник спроса | Campaign/channel/UTM where lawful | Attribution dictionary |
| Заявка | Зарегистрировать явный интерес | Канал, тема, минимальный контакт | Dedup, consent, owner, SLA |
| Консультация | Уточнить потребность | Тип автомобиля, бюджетный диапазон, срок | Next action |
| Подбор автомобиля | Сопоставить потребность и актуальный stock | Vehicle/offer external ID, timestamp | DMS freshness |
| Тест-драйв | Проверить интерес и ресурс | Slot, center, vehicle, manager | Conflict and confirmation check |
| Переговоры | Согласовать условия | Offer version, validity, approvals | Approval trail |
| Покупка | Зафиксировать успешный коммерческий факт | Contract/order/payment references | DMS/ERP reconciliation |
| Сервис | Поддержать владение | Vehicle relation, request, service order | SLA/status feedback |
| Повторная продажа | Создать новую возможность | Eligibility, consent, lifecycle signal | Frequency/suppression rules |

```text
Реклама -> Заявка -> Консультация -> Подбор автомобиля
 -> Тест-драйв -> Переговоры -> Покупка -> Сервис -> Повторная продажа

Exception branches: no contact / no stock / no-show / lost / cancellation /
warranty review / complaint / consent withdrawal
```

## 5. Automotive Data Model

### Сущности и поля

| Сущность | Минимальные поля | Запрещённая практика |
|---|---|---|
| Клиент | External customer ID, тип, разрешённые контакты, center scope, communication preferences | Субъективный профиль и лишние персональные данные |
| Автомобиль | External vehicle ID, model/variant refs, status, source, customer relation if justified | Использовать свободное название как ключ |
| Сделка | Type, center, source, vehicle/offer ref, amount/currency, stage, owner, next step | Хранить все покупки клиента в одной вечной сделке |
| Тест-драйв | Slot, center, vehicle, manager, status, result code | Бронирование без проверки конфликта |
| Сервисный заказ | External service ID, vehicle/customer refs, plan/fact, status | Подменять DMS заказ текстовым комментарием |
| Работа | Work code, service order ID, status, plan/fact | Несогласованные свободные справочники |
| Запчасть | Part external ID, order relation, quantity/status projection | Ручное изменение stock в CRM |
| Гарантийный случай | Case ID, vehicle/service refs, category, owner, decision status | Хранить решение без source/evidence reference |

```text
Customer 1 ---- N Deal N ---- 1 Vehicle/Offer
    |              |                  |
    |              +---- 0..N Test Drive
    |
    +---- N Ownership Relation N ---- 1 Vehicle
    |                                  |
    +---- N Service Case ---- 1 Service Order ---- N Work/Part
                                           |
                                           +---- 0..N Warranty Case
```

### Ключи и кардинальности

| Связь | Кардинальность | Правило |
|---|---:|---|
| Customer → Deal | 1:N | Каждая возможность отдельна |
| Customer ↔ Vehicle | M:N во времени | Нужны периоды владения/пользования |
| Deal → Vehicle | N:1 или N:0..1 | До выбора может быть только offer/segment |
| Vehicle → Service Order | 1:N | История не перезаписывается |
| Service Order → Part/Work | 1:N | Строки имеют source IDs |
| Test Drive → Vehicle | N:1 | Конфликты слота запрещены |

**Рекомендация архитектора:** не использовать VIN, телефон или email как единственный интеграционный ключ. Нужны стабильные source IDs, нормализованные значения и консервативная дедупликация.

## 6. Sales Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новая заявка | Зарегистрировать спрос | Канал, продуктовый интерес, центр, owner | Routing, SLA timer, dedup |
| Квалификация | Подтвердить клиента и намерение | Need, timeframe, contact result, next step | Task/escalation |
| Подбор автомобиля | Найти актуальный вариант | Model/variant or vehicle ID, availability timestamp | DMS query/projection |
| Тест-драйв | Запланировать и провести | Slot, vehicle, manager, confirmation | Reminder/conflict check |
| Коммерческое предложение | Зафиксировать версию условий | Offer ID/version, validity, amount/currency | Approval/follow-up |
| Переговоры | Управлять разногласиями | Decision criteria, next action, loss risk | Manager escalation |
| Договор | Подтвердить согласование | Contract reference, approvals, vehicle/order ID | Completeness check |
| Продажа | Получить подтверждённый факт | Order/payment/handover refs | Reconciliation; no manual financial override |
| Постпродажное сопровождение | Передать в ownership/service cycle | Handover date, vehicle relation, service owner | Welcome/service tasks with consent |

```text
Новая заявка -> Квалификация -> Подбор автомобиля -> Тест-драйв
 -> Коммерческое предложение -> Переговоры -> Договор -> Продажа
 -> Постпродажное сопровождение

Closed branches require structured reason: duplicate / no contact / no stock /
competitor / terms / postponed / invalid request.
```

Правила переходов:

- стадия отражает подтверждённое состояние, а не список действий;
- обязательность полей вводится в момент, когда информация реально возникает;
- перенос назад не должен повторно отправлять сообщения или создавать задачи;
- роботы используют idempotency marker и различают первый вход и повторный;
- причины проигрыша имеют управляемый справочник и owner качества данных.

## 7. Vehicle Management

| Область | Модель | Контроль |
|---|---|---|
| Каталог | Model/variant/option reference | DMS/PIM/ERP master; versioned identifiers |
| Новый автомобиль | Конкретный экземпляр после появления stock ID | Status and price freshness |
| Автомобиль с пробегом | Отдельный экземпляр с appraisal/status reference | DMS used-car master; restricted attributes |
| Статус | Available/reserved/sold/service states as projection | Source timestamp and reconciliation |
| Связь с клиентом | Ownership/use relation with effective dates | No historical overwrite |
| Резерв | Reservation external ID and expiry | Atomic source action outside CRM if required |

**Решение архитектора:** справочник моделей и конкретный автомобиль разделяются. CRM может хранить ссылочную проекцию, но наличие, резерв, цену и юридический статус подтверждает DMS/ERP.

```text
Model -> Variant -> Vehicle Instance -> Stock/Reservation State
                                  |-> Deal
                                  |-> Test Drive
                                  |-> Ownership History
                                  +-> Service History
```

## 8. Test Drive Management

| Шаг | Данные | Автоматизация | Исключение |
|---|---|---|---|
| Запрос | Deal/customer, center, preferred time/model | Create request | No available vehicle |
| Подбор слота | Calendar/resource reference | Availability check | Concurrent booking |
| Подтверждение | Slot, vehicle, manager, channel | Reminder subject to consent | Client not confirmed |
| Проведение | Actual time, synthetic result code | Manager task | No-show/cancellation |
| Результат | Outcome, next step, feedback | Move deal only by validated rule | Rebooking required |

Контроли: уникальный booking ID, timezone, buffer time, водительские/организационные требования только по утверждённому процессу, запрет чувствительных документов в комментариях, журнал переноса и отмены.

## 9. Service Management

| Процесс | Назначение | Master | CRM-контроль |
|---|---|---|---|
| ТО | Плановое обслуживание | DMS/service system | Reminder eligibility, request, status projection |
| Ремонт | Диагностика и работы | DMS | Customer communication and overdue escalation |
| Гарантия | Рассмотрение допустимости | Warranty/DMS | Case ownership and decision status |
| Обращение | Вопрос/жалоба | Bitrix24 | Intake, category, SLA, outcome |
| SLA | Время реакции/решения по договорённой модели | Bitrix24/service policy | Pause/escalation rules documented |

```text
Customer Request -> Service Case -> Appointment -> DMS Service Order
       |                                  |              |
       |                                  v              v
       +--------------------------- Communication <- Work/Part status
                                                       |
                                                       v
                                                Closure/Feedback
```

**Ограничение:** медицинские, диагностические или иные чувствительные файлы и документы клиента не должны попадать в CRM без правового основания, утверждённой модели доступа и срока хранения. Конкретные требования определяет организация и применимое право.

## 10. Spare Parts Management

| Объект | Bitrix24 | ERP/1С/WMS | Правило |
|---|---|---|---|
| Номенклатура | Reference/projection | Master | Stable part ID |
| Остаток | Optional timestamped projection | Master | Never promise stale stock |
| Цена | Approved display value/version | Master | Currency and validity required |
| Заявка | Customer/service intent | Fulfillment document | Correlation ID |
| Резерв | Status reference | Atomic operation/master | Idempotent request |
| Списание | No independent fact | Master | Reconciliation only |

**Рекомендация архитектора:** Smart Process «Запчасть» создаётся только при доказанной процессной необходимости. Для крупного каталога предпочтительна интеграционная ссылка на master, а не полная неконтролируемая копия.

## 11. Rights Model

| Роль | Клиенты | Продажи | Автомобили | Сервис | Аналитика |
|---|---|---|---|---|---|
| Продавец | Назначенные/разрешённые | Свои и разрешённая команда | Read permitted center projection | Limited history needed for sale | Personal/team minimum |
| Руководитель продаж | Center/team scope | Team control and approvals | Read center stock projection | Limited cross-sell context | Center/team |
| Сервисный консультант | Assigned service customers | Read minimum sale context | Read vehicle/service relation | Create/update assigned cases | Service team |
| Мастер | Minimum identity | No commercial edit | Technical reference only | Assigned work status | Operational minimum |
| Маркетолог | Segments/minimized export | Aggregates/campaign attribution | Aggregate product interest | Aggregate service indicators | Approved aggregate dashboards |
| Директор | Governed business scope | Read/approve | Read | Read | Governed cross-center |
| Администратор | Technical administration | Configuration, not business ownership | Configuration | Configuration | Dataset/config access as needed |

### Принципы

- deny-by-default, least privilege и separation of duties;
- центр, подразделение, роль и назначение проверяются совместно;
- сервис не получает коммерческие или финансовые поля без необходимости;
- маркетинг работает с разрешёнными сегментами и минимальными выгрузками;
- администраторская роль не означает право использовать клиентские данные;
- позитивные и негативные сценарии прав обязательны.

**Требует проверки:** фактическая модель прав CRM и Smart Processes, наследование, доступные scopes и аудит зависят от версии, тарифа/редакции и конфигурации портала.

## 12. Automation Architecture

| Автоматизация | Trigger | Действие | Защита |
|---|---|---|---|
| Контроль новой заявки | First entry to new stage | Назначить owner, task и SLA | Idempotency marker |
| Напоминание тест-драйва | Confirmed future booking | Допустимое уведомление/задача | Consent, timezone, cancellation check |
| Эскалация контакта | SLA threshold reached | Notify manager | One escalation per threshold |
| Контроль КП | Offer validity approaching | Follow-up task | Current offer version check |
| Передача в сервис | Confirmed handover | Create governed service onboarding task | Sale fact required |
| Напоминание ТО | Eligible service signal | Create review/communication task | Source date, consent, suppression |
| Повторная продажа | Approved lifecycle rule | New opportunity/task | Frequency and duplicate check |

```text
Event -> Validate source/version -> Check idempotency -> Evaluate condition
     -> Perform one bounded action -> Record outcome -> Retry/exception queue
```

Робот, триггер или бизнес-процесс фиксирует сущность, запуск, условие, исполнителя, срок, поля, повторный запуск, обработку ошибки и эскалацию. Массовые коммуникации не запускаются без сегментной выборки и контрольного sample.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| DMS дилера | Vehicles, stock, reservations, orders, service facts | Official connector/app or documented REST/middleware | Idempotency, timestamps, reconciliation, DLQ |
| ERP | Prices, financial/order projection | Connector/middleware/API | Version, currency, source total reconciliation |
| 1С | Counterparties, catalog, documents where approved | Supported integration or governed middleware | Stable IDs, conflict policy, batch report |
| Сайт | Enquiries, configurations, test-drive requests | CRM forms/official app/webhook/REST | Signature/auth, dedup, source metadata |
| Каталог автомобилей | Model/variant/vehicle projection | API/middleware | Schema/version/freshness checks |
| Телефония | Call events and CRM linkage | Bitrix24 telephony/SIP/REST app as applicable | Rights, missed-call queue, event correlation |
| BI | Governed datasets and aggregates | BI Constructor or approved ETL | Grain, freshness, totals, access |

```text
External Event
   -> Authentication/authorization
   -> Schema validation
   -> Correlation + idempotency
   -> Master ownership decision
   -> Bitrix24 write / exception queue
   -> Reconciliation + monitoring
```

Интеграционные правила:

- webhook URL, OAuth secret и токены хранятся вне Git и логов;
- retry ограничен, использует backoff и не повторяет необратимое действие без idempotency key;
- входные данные валидируются, неизвестные справочники карантинируются;
- двустороннее редактирование разрешено только с формальной conflict policy;
- Cloud не даёт доступа к ядру/БД; On-Premise расширяется без изменения ядра;
- REST-методы и лимиты проверяются по актуальной официальной документации до реализации.

## 14. Marketing Automation

| Сценарий | Сегмент | Условие допуска | Контроль |
|---|---|---|---|
| Акция | Approved audience | Valid consent/legal basis and current offer | Suppression/frequency cap |
| Напоминание ТО | Owner + vehicle/service due signal | Confirmed ownership and source date | Opt-out and completed service check |
| Новая модель | Relevant interest/ownership segment | Explicit campaign eligibility | No sensitive inference |
| Trade-in/повторная продажа | Lifecycle signal | Approved rule and no active duplicate | New Deal, not overwrite old sale |
| Сервисное сообщение | Active case/order | Transactional purpose | Separate from marketing consent |

**Рекомендация архитектора:** транзакционные и маркетинговые коммуникации разделяются. Сегмент, основание, suppression, частота, канал и результат фиксируются; автоматизация не делает выводов о человеке по неподтверждённым данным.

## 15. Analytics

| Отчёт | Grain | Показатели | Контроль качества |
|---|---|---|---|
| Продажи автомобилей | One authoritative sale/handover | Count, amount, model, center, channel | Reconcile with DMS/ERP |
| Эффективность менеджеров | Deal + activity window | Conversion, cycle time, next-step discipline | Role/territory normalization |
| Конверсия тест-драйвов | One booking/result | Booked, held, no-show, sale after drive | Defined attribution window |
| Сервисная загрузка | Service order/resource period | Appointments, work plan/fact, delay | DMS/service master |
| Повторные покупки | Customer + authoritative sales | Repeat rate, interval, cohort | Stable identity and ownership history |
| Потери | Closed-lost deal | Structured reasons, stage, center | Mandatory governed reason |

```text
Operational Sources -> Validation -> Conformed IDs -> Governed Dataset
                  -> Reconciliation -> Metric Layer -> Role-based Dashboard
```

Каждая метрика имеет владельца, формулу, grain, окно времени, timezone, валюту, исключения, источник, freshness SLA и сверочный итог. Доступность BI Конструктора и наборов данных зависит от тарифа и текущей версии.

## 16. Security Architecture

| Контроль | Реализация | Проверка |
|---|---|---|
| Изоляция центров | Roles/scopes by department/center | Cross-center negative tests |
| Минимизация данных | Only required fields and projections | Field inventory review |
| Секреты | Environment/secret storage outside repository | Secret scan and rotation drill |
| Аудит | CRM history, integration logs without secrets/PII | Sample trace by correlation ID |
| Экспорт | Role approval, minimum columns, protected destination | Export rights and sample inspection |
| Интеграции | Least-privilege credentials and allowlisted operations | Revocation and unauthorized call tests |
| Retention | Approved retention/archive/delete policy | Periodic evidence review |

**Cloud:** проверяются доступные штатные журналы, история CRM, права и интеграционные логи; доступа к серверу, ядру и БД нет. **On-Premise:** дополнительно контролируются сервер, PHP, БД, файлы, cron, журналы, резервные копии и обновления; ядро не изменяется.

## 17. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Заявка с сайта | Валидный synthetic request | Одна заявка/сделка, owner и SLA |
| 2 | Повтор события | Тот же source event ID | Дубликат не создан |
| 3 | Некорректный контакт | Пустой/невалидный обязательный атрибут | Карантин/понятная ошибка без потери payload |
| 4 | Квалификация | Заполнение next step | Переход разрешён, задача создана один раз |
| 5 | Подбор автомобиля | Fresh DMS status | Показывается актуальная проекция |
| 6 | Устаревший stock | Истёк freshness SLA | Продажа/обещание блокируется или требует проверки |
| 7 | Тест-драйв | Свободный слот и ресурс | Booking создан и связан со сделкой |
| 8 | Конфликт тест-драйва | Тот же vehicle/time | Второе бронирование отклонено/эскалировано |
| 9 | No-show | Result code no-show | Сделка не закрывается успешно; создаётся допустимый next step |
| 10 | КП и согласование | Discount beyond threshold | Маршрут approval и полный audit trail |
| 11 | Продажа | Authoritative handover event | Deal closed once; vehicle/customer relation created |
| 12 | Сервисная заявка | Existing vehicle/customer | Case linked without duplicating identity |
| 13 | Гарантийный случай | Missing required source reference | Decision not fabricated; exception assigned |
| 14 | Запчасть | ERP unavailable | Controlled retry/queue; no false stock status |
| 15 | Повторная продажа | Eligible synthetic customer | New Deal; previous sale unchanged |
| 16 | Отзыв согласия | Suppression event | Marketing stopped; transactional policy preserved |
| 17 | Права центра | User from another center | Unauthorized record/field unavailable |
| 18 | Retry integration | Timeout after accepted write | Idempotency prevents duplicate |
| 19 | BI reconciliation | Compare governed sample totals | Difference within approved threshold or incident raised |
| 20 | Rollback | Failed release step | Documented reversal restores verified state |

### Наборы тестовых данных

- только synthetic customers, centers, vehicles and IDs;
- позитивные, негативные, граничные, повторные и конкурентные события;
- Cloud/On-Premise and rights variants where applicable;
- no real VIN, phone, email, contract, plate, SIP credential or webhook.

## 18. Deployment

| Этап | Результат | Gate |
|---|---|---|
| Discovery | AS IS, owners, master matrix, volumes, constraints | Business and IT sign-off |
| Архитектура | TO BE, data model, rights, ADR, NFR | Architecture review |
| Настройка | CRM/Smart Processes/automation in test scope | Configuration review |
| Интеграции | Contracts, idempotency, retry, reconciliation | Integration tests |
| Миграция | Map, cleanse, dry-run, sample report | Count and sample reconciliation |
| Обучение | Role-based instructions and exercises | UAT readiness |
| Запуск | Approved release window and support roster | Go/no-go |
| Stabilization | Incident monitoring and KPI baseline | Exit criteria |

```text
Inventory -> Backup/Export -> Test Configuration -> Synthetic Tests -> UAT
 -> Dry-run Migration -> Go/No-Go -> Release -> Smoke -> Reconciliation
 -> Hypercare -> Closure or Rollback
```

Нельзя удалять/переименовывать действующие поля, стадии, роботов или связи до dependency analysis, backup/export, пилота и утверждённого rollback. Для рабочей воронки предпочтительна параллельная настройка.

## 19. Monitoring

| Сигнал | Порог/правило | Действие | Владелец |
|---|---|---|---|
| Необработанные заявки | SLA by channel | Alert and escalation | Sales operations |
| Просроченный контакт | No next activity by rule | Manager task | Sales manager |
| Зависшие сделки | No movement/activity | Review queue | Pipeline owner |
| Ошибки интеграций | Error/retry/DLQ threshold | Incident and replay after fix | Integration support |
| Задержка проекции DMS | Freshness SLA exceeded | Mark stale, stop unsafe action | DMS owner |
| Тест-драйв без результата | End time passed | Completion task | Center coordinator |
| Сервисная задержка | SLA/plan breach | Escalation and client update | Service manager |
| Расхождение BI | Control total mismatch | Data-quality incident | BI owner |

Мониторинг не содержит секретов и избыточных персональных данных. Alert связывается с correlation ID, системой, объектом, временем, severity и runbook.

## 20. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Дубли клиентов | Разные каналы и слабые ключи | Разорванная история | Stable IDs, normalization, review queue |
| 2 | Ложное объединение | Совпадение телефона/email | Чужие данные в профиле | Conservative matching and manual review |
| 3 | Потеря заявки | Нет подтверждения intake | Упущенная продажа | Ack, SLA, reconciliation |
| 4 | Дубли сделок | Retry без idempotency | Искажённая воронка | Source event ID/upsert |
| 5 | Устаревший stock | Задержка DMS | Обещан недоступный автомобиль | Freshness mark and live verification |
| 6 | Двойной резерв | Конкурентные операции | Конфликт продажи | Reserve atomically in master |
| 7 | Неверная цена | CRM редактирует master fact | Финансовый спор | Versioned source price |
| 8 | Потеря истории автомобиля | Relation overwritten | Нет lifecycle trace | Effective-dated ownership |
| 9 | Избыточный VIN access | Общие права | Privacy/security incident | Field-level minimization/scope |
| 10 | Конфликт тест-драйва | Нет resource lock | Срыв встречи | Availability check and unique booking |
| 11 | Неверная timezone | Центры в разных зонах | Пропущенная встреча | Store timezone contract |
| 12 | Повтор напоминания | Re-entry stage | Spam | First-entry marker |
| 13 | Маркетинг без допуска | Consent ignored | Compliance/reputation risk | Consent/suppression controls |
| 14 | Смешение сообщений | Service message used as marketing | Нарушение ожиданий | Purpose separation |
| 15 | Разрыв продажи и сервиса | Нет vehicle link | Плохой клиентский опыт | Stable customer/vehicle relation |
| 16 | CRM заменяет DMS | Scope creep | Неверные факты | Master matrix and ADR |
| 17 | Запчасти без актуального остатка | Full catalog copy | Ошибка обещания | Timestamped projection/query master |
| 18 | Скрытая интеграционная ошибка | Нет DLQ | Потерянные события | Monitoring, queue, replay |
| 19 | Бесконечный retry | Нет лимита | Нагрузка/дубли | Bounded backoff |
| 20 | Логи с PII/secret | Raw payload logging | Утечка | Masking and log policy |
| 21 | Webhook в Git | Ошибка хранения | Компрометация | Secret storage and rotation |
| 22 | Широкий токен | Excess scopes | Масштабный ущерб | Least privilege credentials |
| 23 | Межцентровой доступ | Неверная роль | Утечка клиентских данных | Center scope negative tests |
| 24 | Администратор как оператор | Role confusion | Неконтролируемые изменения | Separate admin/business accounts |
| 25 | Неверный owner | Routing не учитывает центр | Просрочка | Governed routing table |
| 26 | Цикл стадий | Роботы перемещают друг друга | Шторм действий | Loop guard and event design |
| 27 | Потеря approval trail | Скидка согласована в чате | Спор условий | Structured approval |
| 28 | КП без версии | Условия перезаписаны | Неверный договор | Version and validity |
| 29 | Ручное закрытие продажи | Нет source fact | Ложная выручка | DMS/ERP confirmation |
| 30 | Гарантия признана CRM | Нет полномочий | Неверное обещание | DMS/warranty decision master |
| 31 | SLA без pause rule | Неясный расчёт | Недостоверный KPI | SLA dictionary |
| 32 | Жалоба потеряна | В комментариях | Репутационный риск | Structured case lifecycle |
| 33 | Миграция без dry-run | Непроверенная карта | Массовая порча | Sample, report, rollback |
| 34 | Название как ключ | Нестабильное значение | Дубли/ошибочные связи | External IDs |
| 35 | Импорт поверх production | Нет изоляции | Повреждение данных | Test scope and go/no-go |
| 36 | Нет rollback | Односторонняя настройка | Длительный простой | Versioned plan and backup |
| 37 | Плавающие справочники | Каждый вводит своё | Невозможна аналитика | Governed dictionaries |
| 38 | Разная валюта | Нет normalization | Неверная сумма | Currency/date rules |
| 39 | Неверный grain BI | Deals mixed with vehicles | Двойной счёт | Metric contracts |
| 40 | Нет reconciliation | CRM totals trusted blindly | Скрытые расхождения | Control totals with DMS/ERP |
| 41 | Тариф не проверен | Design by assumption | Недоступная функция | Portal capability assessment |
| 42 | Cloud treated as server | Ожидается DB/PHP access | Нереализуемое решение | Cloud boundary |
| 43 | Изменено ядро On-Premise | Быстрое исправление | Update/security failures | Extension mechanisms only |
| 44 | Нет UAT сервиса | Проверены только продажи | Срыв aftersales | End-to-end UAT |
| 45 | Реальные данные в тестах | Удобная копия | Privacy breach | Synthetic data |
| 46 | Не проверен revoke | Интеграция бессрочна | Остаточный доступ | Credential lifecycle test |
| 47 | Нет support owner | Инциденты без маршрута | Долгое восстановление | RACI/runbook/on-call |
| 48 | Нет change log | Неясно что внедрено | Сложный аудит | Versioned configuration register |
| 49 | Массовая рассылка ошибочному сегменту | Нет sample/approval | Репутационный ущерб | Dry-run, sample, approval |
| 50 | Оптимизация без baseline | Нет исходных метрик | Нельзя доказать эффект | Baseline and acceptance criteria |
| 51 | Удаление старой связи | Cleanup без dependency check | Потеря истории | Archive policy and dependency scan |
| 52 | События вне порядка | Async delivery | Старый статус перезаписывает новый | Source version/time ordering |
| 53 | Schema drift | DMS изменил контракт | Ошибочные записи | Contract validation/versioning |
| 54 | Один correlation ID отсутствует | Системы не трассируются | Долгая диагностика | End-to-end correlation |
| 55 | Персональные выгрузки BI | Широкий dataset | Утечка | Aggregation, scopes, export controls |

## 21. ADR

| ADR | Решение | Альтернатива | Обоснование | Последствие |
|---|---|---|---|---|
| ADR-AUTO-001 | Контакт — основной профиль физлица; отдельный MDM только при доказанной потребности | Smart Process «Клиент» для всех | Не дублировать CRM-сущность | Нужна identity policy и минимизация полей |
| ADR-AUTO-002 | Автомобиль в CRM — проекция DMS с stable ID | CRM как vehicle master | Stock/status требует профильного master | Интеграция и freshness обязательны |
| ADR-AUTO-003 | Отдельная сделка на каждую коммерческую возможность | Одна сделка на клиента | Сохраняет воронку и повторные продажи | Нужен duplicate/opportunity rule |
| ADR-AUTO-004 | Тест-драйв — отдельный lifecycle object | Поля сделки/календарь без связи | Ресурс, слот и результат требуют истории | Проверить Smart Process availability |
| ADR-AUTO-005 | Сервисный заказ master в DMS, CRM хранит service case/projection | Полное ведение ремонта в CRM | Не подменять DMS | Нужны source IDs/reconciliation |
| ADR-AUTO-006 | Интеграции через governed middleware/contracts | Свободные point-to-point webhooks | Централизует idempotency и monitoring | Появляется platform ownership |
| ADR-AUTO-007 | Права сегментируются по центру и функции | Общая дилерская роль | Least privilege | Требуются negative tests |
| ADR-AUTO-008 | BI использует conformed IDs и reconciled datasets | Отчёты только по raw CRM fields | Продажи/сервис имеют разные masters | Нужен metric owner/data-quality SLA |

Каждый ADR перед реализацией получает статус `proposed/accepted/superseded`, владельца, дату, портал/версию, ограничения и ссылку на проверку.

## 22. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Нет Discovery по центрам | Сразу начата настройка | Конфликт процессов | AS IS and stakeholder map |
| 2 | Не определён Cloud/On-Premise | Шаблонный дизайн | Недоступные механизмы | Capability assessment |
| 3 | Не проверен тариф/редакция | Предположение | Блокировка запуска | Verify target portal |
| 4 | CRM объявлена DMS | Scope misunderstood | Недостоверный stock/service | Master-system matrix |
| 5 | Один профиль на семью/организацию | Упрощение | Смешение данных | Correct person/company model |
| 6 | Телефон — единственный ключ | Удобство | Ложные merge | Stable external IDs |
| 7 | VIN — открытое поле всем | Нет threat model | Утечка | Minimize and restrict |
| 8 | История владения перезаписывается | Одно текущее поле | Потеря lifecycle | Effective-dated relation |
| 9 | Модель и экземпляр смешаны | Нет data modeling | Неверный stock | Separate catalog/vehicle instance |
| 10 | Новые и used vehicles не различены | Одна воронка без признака | Неверные правила | Explicit type and pipeline decision |
| 11 | Все обращения становятся лидами | Lead used universally | Мусорная воронка | Classify sales/service cases |
| 12 | Лиды отключены без анализа | Модный шаблон | Неквалифицированный спрос смешан | ADR on lead model |
| 13 | Одна вечная сделка клиента | Экономия сущностей | Нет повторных продаж | Deal per opportunity |
| 14 | Сделка на каждое системное событие | Event/object confused | Дубли | Business event semantics |
| 15 | Стадии описывают действия | Task list as pipeline | Непонятное состояние | State-based stages |
| 16 | Слишком много стадий | Каждый шаг стал стадией | Низкая дисциплина | Tasks/checklists for actions |
| 17 | Обязательные поля с первой стадии | Максимальный контроль | Формальное заполнение | Stage-appropriate requiredness |
| 18 | Причина проигрыша — комментарий | Нет справочника | Нет аналитики | Governed reason codes |
| 19 | Нет next step | Фокус на stage | Зависшие сделки | Activity/next-action policy |
| 20 | Робот повторяет задачу | Не учтён re-entry | Дубли задач | First-entry/idempotency marker |
| 21 | Роботы создают цикл | Взаимные переходы | Automation storm | Loop guard and design review |
| 22 | Скидка согласуется вне CRM | Быстрее в чате | Нет audit trail | Structured approval |
| 23 | КП без версии/срока | Перезапись файла | Спор условий | Versioned offer metadata |
| 24 | Stock копируется без timestamp | Простая синхронизация | Устаревшие обещания | Freshness SLA |
| 25 | Резерв создаётся только в CRM | Нет DMS action | Двойная продажа | Reserve in master |
| 26 | Цена редактируется в двух системах | Нет ownership | Конфликт суммы | One price master |
| 27 | Продажа закрывается до подтверждения | KPI pressure | Ложная выручка | Authoritative completion event |
| 28 | Оплата хранится вручную | Нет ERP link | Неверный статус | Financial projection from master |
| 29 | Тест-драйв — текстовое поле | Нет entity | Потеря расписания | Separate booking lifecycle |
| 30 | Нет проверки конфликтов | Calendar not integrated | Двойной слот | Resource availability control |
| 31 | Не учтена timezone | Несколько регионов | Ошибочные reminders | Explicit timezone |
| 32 | No-show считается проведением | Нет result code | Завышенная конверсия | Structured outcome |
| 33 | Напоминание после отмены | Status not rechecked | Негативный опыт | Pre-send state validation |
| 34 | Сервис ведётся комментариями | Быстрый старт | Нет SLA/history | Service case/order model |
| 35 | CRM принимает гарантийное решение | Неверная ответственность | Ошибочное обещание | Warranty master and status projection |
| 36 | Работы и запчасти одной строкой | Нет grain | Нет себестоимости/аналитики | Separate line references |
| 37 | CRM ведёт остатки запчастей | DMS/ERP не интегрирован | Ложные остатки | ERP/WMS master |
| 38 | Ремонт и обращение смешаны | Один объект на всё | Нет коммуникационного SLA | Case vs service order separation |
| 39 | SLA без календаря | Только часы | Неверная просрочка | Calendar/pause policy |
| 40 | Сервис видит все финансы | Общая роль | Excess access | Field/function scopes |
| 41 | Маркетолог экспортирует raw PII | Нет ограничения | Утечка | Aggregates/minimum export |
| 42 | Центры имеют общую роль | Быстрая настройка | Cross-center leak | Center-based roles |
| 43 | Права проверены только позитивно | Happy path only | Скрытый доступ | Negative rights tests |
| 44 | Admin account used daily | Удобство | Неконтролируемые изменения | Separate privileged account |
| 45 | Нет access review | Роли устаревают | Остаточные права | Periodic recertification |
| 46 | Webhook записан в README | Игнорирование secret policy | Компрометация | Secret storage and rotation |
| 47 | Токен имеет все права | Один integration user | Excess blast radius | Least privilege |
| 48 | REST retry без idempotency | Timeout ambiguity | Дубли | Idempotency key/source ID |
| 49 | Бесконечный retry | Нет DLQ | Нагрузка | Bounded retry/queue |
| 50 | Ошибки интеграции скрыты | Empty catch | Потеря данных | Explicit errors and alerts |
| 51 | Raw payload в логах | Debug shortcut | PII/secret exposure | Masked structured logs |
| 52 | Нет schema validation | Trust source | Corrupt fields | Contract validation |
| 53 | События вне порядка игнорируются | Асинхронность не учтена | Старый status wins | Version/order control |
| 54 | Двустороннее редактирование свободно | Нет conflict policy | Ping-pong updates | Field ownership |
| 55 | Нет correlation ID | Разные журналы | Долгая диагностика | End-to-end trace ID |
| 56 | Batch без проверки лимитов | Оптимизация заранее | Partial failure | Official limits and batch report |
| 57 | Mass update без dry-run | Срочность | Массовая ошибка | Sample/dry-run/rollback |
| 58 | Миграция без карты полей | Нет inventory | Потеря значений | Mapping and transforms |
| 59 | Реальные данные в тесте | Копия production | Privacy incident | Synthetic dataset |
| 60 | Нет count reconciliation | Импорт завершился без ошибки | Тихая потеря | Counts and sample checks |
| 61 | BI считает строки вместо продаж | Wrong grain | Завышенный результат | Metric contract |
| 62 | Тест-драйв приписан любой продаже | Нет attribution window | Ложная конверсия | Defined window/rule |
| 63 | Выручка берётся из Deal без сверки | CRM trusted as finance | Расхождение с ERP | Reconciliation |
| 64 | Повторная продажа меняет старую | Нет new opportunity | Потеря истории | New Deal per sale |
| 65 | Нет baseline | Не измерено AS IS | Эффект недоказуем | Baseline KPI |
| 66 | Мониторинг только availability | Нет business controls | Теряются заявки | Technical + business monitoring |
| 67 | Alert без owner/runbook | Только уведомление | Инцидент не решается | RACI and runbook |
| 68 | Release без smoke test | Сроки | Ошибка попадает клиентам | Post-release checklist |
| 69 | Нет rollback | Успех предполагается | Долгий простой | Tested rollback plan |
| 70 | Удалены старые поля сразу | Cleanup before adoption | Broken automation/integration | Dependency scan and deprecation |
| 71 | Изменено ядро коробки | Quick customization | Update failures | Separate module/extensions |
| 72 | Cloud проектируется как коробка | Ожидание server access | Нереализуемость | Explicit deployment boundary |
| 73 | Документы не версионируются | Только устные решения | Неподдерживаемость | ADR/config/change register |
| 74 | Нет обучения по ролям | Общая демонстрация | Пользователи обходят процесс | Role-based training/UAT |
| 75 | Нет hypercare | Проект закрыт сразу | Инциденты копятся | Stabilization window |
| 76 | Нет владельца справочников | Everyone edits values | Data drift | Data steward |
| 77 | Удаление вместо архивирования | Clean-up goal | Потеря истории | Retention/archive policy |
| 78 | Все файлы хранятся в CRM | Нет classification | Security/storage risk | Document policy and links |
| 79 | Маркетинг не проверяет suppression | Segment only | Нежелательные сообщения | Pre-send eligibility check |
| 80 | Критерий готовности — настроено | Нет measurable acceptance | Спор о результате | Tested acceptance criteria |

## 23. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Подтверждённый AS IS и scope | Stakeholders, portals, tariffs, volumes, pain points | Owners approve facts/open questions |
| 2. Архитектура | Утверждённый TO BE | Entity model, master matrix, rights, NFR, ADR | Architecture review passed |
| 3. CRM настройка | Базовый intake и sales pipeline | Fields, stages, roles, automation register | Synthetic happy/negative paths pass |
| 4. Автомобили | Governed vehicle/test-drive model | IDs, relations, status mapping, booking rules | DMS freshness/conflict tests pass |
| 5. Сервис | Связанный aftersales lifecycle | Case/order/warranty mappings, SLA | End-to-end sales-to-service UAT passes |
| 6. Интеграции | Надёжный обмен | Contracts, secrets model, retry, idempotency, monitoring | Failure/replay/reconciliation tests pass |
| 7. Тестирование | Evidence-based quality | Test plan, synthetic dataset, UAT, defects | Critical defects closed; sign-off recorded |
| 8. Запуск | Контролируемый production release | Runbook, backup/export, cutover, rollback, support roster | Smoke and reconciliation pass |
| 9. Оптимизация | Управляемое улучшение | KPI baseline, incident/change backlog | Improvement measured and approved |

### Критерии завершения кейса на целевом проекте

- подтверждены Cloud/On-Premise, тариф/редакция, версия и права;
- утверждены entity model, master-system matrix, integration contracts and ADR;
- проверены права каждого центра и роли позитивными и негативными тестами;
- выполнены synthetic tests, UAT, dry-run migration and reconciliation;
- секреты вынесены из репозитория, журналы не содержат PII;
- есть release, rollback, monitoring, incident and support procedures;
- бизнес-владелец принял измеримые критерии результата.

### Официальные источники

Использованы только записи из `knowledge/SOURCES.md`; перед реализацией URL, дата обновления и применимость к целевому порталу проверяются повторно.

- [Официальная пользовательская документация Bitrix24](https://helpdesk.bitrix24.ru/) — CRM, настройки, права, автоматизация, телефония и Cloud-возможности.
- [Официальная документация REST API](https://apidocs.bitrix24.ru/) — авторизация, методы и ограничения REST.
- [REST API reference](https://apidocs.bitrix24.ru/api-reference/) — подтверждение конкретных методов и параметров перед разработкой.
- [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/) — официальные варианты локальных интеграций.
- [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) — создание и базовая конфигурация Smart Processes.
- [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) — штатная CRM-автоматизация.
- [Бизнес-процессы](https://helpdesk.bitrix24.ru/open/23951294/) — выбор типа бизнес-процесса.
- [Телефония Bitrix24](https://helpdesk.bitrix24.ru/open/6450911/) — официальные способы подключения телефонии.
- [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) — доступные данные и ограничения аналитики.
- [Маркетплейс Bitrix24](https://www.bitrix24.ru/apps/) — поиск официально публикуемых приложений; применимость оценивается отдельно.
- [Документация коробочной версии](https://dev.1c-bitrix.ru/docs/portal.php) — особенности On-Premise и расширение без изменения ядра.

### Ограничения и открытые вопросы

До практического внедрения необходимо определить:

1. Cloud или On-Premise, тариф/редакцию, версию и регион портала.
2. DMS/ERP/1С, их API, owners, SLA, объёмы и поддерживаемые идентификаторы.
3. Нужны ли лиды и отдельные направления новых, used, fleet и service sales.
4. Где являются master клиент, автомобиль, резерв, цена, заказ, оплата, сервис и гарантия.
5. Требуется ли Smart Process для каждого предложенного объекта и доступен ли он.
6. Правила межцентрового обслуживания и допустимость единого клиентского профиля.
7. Правовые основания, сроки хранения, consent/suppression и правила удаления.
8. Сервисные SLA, календари, расписание ресурсов и источник availability.
9. Модель BI, grain, владельцев метрик, контрольные итоги и допустимую задержку.
10. Test portal/pipeline, release window, допустимый простой, rollback и support RACI.

# Related Documents

- [Case Library](../README.md)
- [Architecture Patterns](../PATTERNS.md)
- [Discovery Template](../DISCOVERY_TEMPLATE.md)
- [Architecture Template](../ARCHITECTURE_TEMPLATE.md)
