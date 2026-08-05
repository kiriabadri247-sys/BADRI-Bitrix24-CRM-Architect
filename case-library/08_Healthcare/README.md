# Healthcare CRM Architecture Case

Архитектурный кейс для синтетической медицинской организации, которая принимает обращения, координирует записи, информирует о сервисе, оформляет договорный контур, контролирует оплату и повторный контакт. Документ задаёт референсную CRM-модель, а не готовую конфигурацию и не медицинскую информационную систему.

> **Граница безопасности:** реальные клиники, пациенты, врачи, телефоны, email, даты рождения, адреса, диагнозы, жалобы, результаты исследований, назначения, документы, CRM ID, webhook URL, токены и выгрузки не используются. Все данные в примерах синтетические.

> **Архитектурная граница:** медицинская информационная система (МИС) остаётся master расписания, медицинской услуги, факта приёма и медицинской документации. Bitrix24 отвечает за допустимый клиентский сервис, обращения, коммуникации и минимальную CRM-проекцию статуса записи. CRM не используется для постановки диагноза, назначения лечения, ведения медицинской карты или принятия клинических решений.

> **Не юридическая гарантия:** требования к согласию, основаниям обработки, локализации, срокам хранения, медицинской тайне, уведомлениям, записи разговоров и доступу определяются уполномоченными специалистами для конкретной юрисдикции и организации. Кейс фиксирует инженерные контроли, но не заменяет правовую, медицинскую и информационно-безопасностную экспертизу.

Обозначения:

- **факт Bitrix24** — подтверждается официальным источником из `knowledge/SOURCES.md`;
- **решение архитектора** — целевой выбор кейса, подлежащий согласованию;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа/редакции, версии, прав, региона, приложения или внешней системы.

## 1. Business Context

### Модель медицинского бизнеса

Референсный сценарий — частная амбулаторная организация с платными консультационными и диагностическими услугами. CRM сопровождает путь до и после визита, но клиническое содержание приёма остаётся в МИС. Для стационара, лаборатории, телемедицины, страховой модели и регулируемых программ требуется отдельное проектирование.

| Аспект | Целевая интерпретация |
|---|---|
| Типы услуг | Первичные/повторные консультации, диагностика, пакеты и сервисные программы — без клинического содержания в CRM |
| Каналы обращения | Сайт, форма, телефония, разрешённые сообщения, личное обращение |
| Основная единица CRM | Обращение/пациентский запрос и связанная проекция записи |
| Расписание | МИС/система записи |
| Медицинская карта | Только МИС/назначенная медицинская система |
| Финансовый факт | 1С/ERP/учётная или платёжная система |

### Цели внедрения CRM

1. Не терять обращения и первый ответ.
2. Связать допустимую историю коммуникаций без дублирования пациента.
3. Управлять записью, переносом и отменой через интеграцию с master-расписанием.
4. Разделить операторский, медицинский, финансовый и аналитический доступ.
5. Контролировать сервисный SLA и повторный контакт без клинических обещаний.
6. Получать сверяемые отчёты по обращениям, записи, визитам и сервису.

```text
Channel -> CRM request -> Appointment projection -> MIS schedule
              |                  |                    |
              v                  v                    v
          Communication       Reminder          Medical visit
              |                                       |
              +-> Service case <----------------------+
                              |
                              v
                         Finance / BI
```

## 2. AS IS Problems

| Проблема AS IS | Проявление | Корневая причина | Целевой контроль |
|---|---|---|---|
| Потеря обращений | Нет owner и первого ответа | Каналы не объединены | Intake, routing и fallback |
| Нет единой допустимой истории | Повторные профили | Нет identity policy | Нормализация и dedup |
| Нет контроля записи | Переносы и отмены расходятся | CRM и МИС редактируют свободно | Master schedule и event contract |
| Разрыв между отделами | Оператор не видит допустимый статус | Нет минимальной проекции | Role-based service view |
| Нет аналитики | Показатели считают вручную | Нет grain и reconciliation | Metric dictionary |
| Клинические сведения попадают в комментарии | Свободный текст без границ | CRM используют как медкарту | Data classification и запрет клинического содержания |
| Напоминания дублируются | Несколько систем отправляют сообщения | Нет owner коммуникации | Message ownership и idempotency |
| Оплата отмечается вручную | Статус недостоверен | CRM подменяет учётную систему | Finance confirmation |
| Документы доступны широко | Общая роль/папка | Нет classification | Least privilege и secure master |
| Качество сервиса смешано с клиническим результатом | Не определены KPI | CRM пытается оценивать лечение | Разделить service и clinical metrics |

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Лид | Неквалифицированное обращение, если lead-based процесс подтверждён | Bitrix24 |
| CRM | Контакт | Минимальный профиль физлица для разрешённой коммуникации | Bitrix24/МИС/MDM по ADR |
| CRM | Компания | Корпоративный заказчик/партнёр | Bitrix24/ERP/MDM |
| CRM | Сделка | Коммерческая возможность или договорный пакет | Bitrix24 |
| Service | Обращение | Вопрос или сервисное действие | Bitrix24 |
| Domain | Smart Process «Пациентский запрос» | Потребность в записи/услуге без клинической истории | Bitrix24 |
| Schedule | Smart Process «Запись» | Минимальная CRM-проекция appointment | МИС |
| Catalog | Smart Process «Медицинская услуга» | Разрешённая коммерческая проекция каталога | МИС/ERP/catalog master |
| Contract | Smart Process «Договор» | Metadata и статус договорного процесса | ECM/ERP/разрешённое хранилище |
| Follow-up | Smart Process «Повторный визит» | Сервисный контроль следующей записи | МИС master факта; Bitrix24 coordination |
| Service | Smart Process «Обращение после лечения» | Обратная связь/сервисный вопрос | Bitrix24, без клинических выводов |

**Факт Bitrix24:** CRM, Smart Processes, роботы, триггеры и бизнес-процессы описаны официальными материалами. Конкретные поля, связи, права, automation и доступность проверяются на целевом портале.

**Решение архитектора:** CRM получает только минимально необходимый набор: устойчивые внешние ID, разрешённые контактные данные, код/категорию услуги, плановый/фактический сервисный статус и ссылки на master-записи. Диагноз, анамнез, результаты, назначения и клинические комментарии не входят в модель.

```text
Lead/Request -> Contact/Company -> Deal/Contract process
      |               |
      v               v
Patient Request -> Appointment projection <-> MIS
      |                      |                 |
      v                      v                 v
Service communication   Payment status    Medical record
      |                      |                 |
      +-> Follow-up / Post-treatment case     +-> NOT stored in CRM
```

### Правила целостности

- один пациентский профиль имеет устойчивый `external_patient_id`, но сам идентификатор не раскрывает клинические данные;
- одна запись имеет один `external_appointment_id` master-системы;
- CRM не создаёт второй слот при повторном событии;
- плановый статус не считается фактом визита;
- изменение МИС не перезаписывается свободной правкой CRM;
- финансовый факт приходит из назначенной системы;
- логирование, аналитика и тесты используют синтетические или минимизированные данные.

## 4. Patient Journey Architecture

| Этап | Цель | Допустимые CRM-данные | Контроль |
|---|---|---|---|
| Обращение | Принять запрос | Канал, контакт, сервисная категория, owner | Validation, dedup, consent context |
| Первичная консультация | Квалифицировать сервисную потребность | Нужный тип услуги/направление без диагноза | Оператор не даёт клинических рекомендаций |
| Запись | Подтвердить слот | Appointment ID, service code, specialist ID, date/status | МИС master, timezone/version |
| Приём | Получить минимальный статус | Planned/completed/no-show/cancelled по mapping | Без clinical payload |
| Оплата | Получить финансовый статус | External payment ID, amount/state if required | Finance reconciliation |
| Контроль качества | Оценить сервис | Contact experience, response/resolution | Не оценивать клинический результат CRM-метрикой |
| Повторный визит | Организовать допустимый follow-up | Follow-up due/category/status | Основание, канал и сообщение проверены |

```text
Обращение -> Первичная консультация -> Запись -> Приём
 -> Оплата -> Контроль качества -> Повторный визит

Exception: перенос / отмена / no-show / service complaint
```

## 5. Healthcare Data Model

### Сущности и поля

| Сущность | Минимальные поля | Запрещённое/ограниченное содержание |
|---|---|---|
| Пациент/Contact | External ID, разрешённые контакты, communication preferences, service segment | Диагнозы, результаты, назначения и медицинская карта |
| История обращений | Case/request IDs, channel, category, status, owner, dates | Свободное описание клинических симптомов |
| Запись | Appointment ID, date/time, specialist ID, service code, location, status | Протокол приёма и медицинское заключение |
| Услуга | Service ID, direction/category, commercial price projection, duration | Клиническая интерпретация результата |
| Специалист | External ID, public service category, schedule reference | Избыточные HR/персональные сведения |
| Договор | Contract ID, type, status, version, secure reference | Неконтролируемые копии документов |
| Оплата | External payment ID, status, amount/currency if necessary | Платёжные секреты и полные карточные данные |

```text
Contact 1 ---- N Patient Request 1 ---- N Appointment
   |                                      |
   +---- N Service Case                   +---- 1 Service
   |                                      +---- 1 Specialist
   +---- N Deal/Contract                  +---- 0..N Follow-up

Clinical record -> MIS only; CRM stores no clinical relation payload.
```

### Identity и связи

| Отношение | Кардинальность | Правило |
|---|---:|---|
| Contact → Request | 1:N | Новый запрос не перезаписывает прошлый |
| Request → Appointment | 1:N | Перенос хранится как версия/событие по контракту |
| Appointment → Service | N:1 | Service code управляется master-каталогом |
| Appointment → Specialist | N:1 | Назначение подтверждает МИС |
| Contact → Contract | 1:N | Актуальная версия определяется metadata |
| Appointment → Follow-up | 1:N | Follow-up не раскрывает причину лечения |

## 6. CRM Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новое обращение | Зарегистрировать запрос | Канал, минимальный контакт, category, owner | Routing и first-response task |
| Квалификация | Определить допустимый сервисный маршрут | Service direction, urgency route according to approved script, next step | SLA и escalation без клинического решения |
| Подбор услуги | Сопоставить запрос с каталогом/специалистом | Service code/category, location, constraints | Задача оператору, запрос availability |
| Запись создана | Получить подтверждённый appointment ID | Slot, specialist, service, status | Idempotent sync и confirmation |
| Ожидание визита | Поддержать явку | Appointment status/time, communication eligibility | Напоминание и delivery tracking |
| Визит завершён | Получить минимальный факт | Completed/no-show/cancelled mapping | Service follow-up, без clinical data |
| Повторный контакт | Выполнить допустимую коммуникацию | Reason category, due date, owner, channel | Task/notification according to policy |
| Закрыто | Подтвердить завершение CRM-обязательства | Result, status source, open cases | Completeness check |

```text
Новое обращение -> Квалификация -> Подбор услуги -> Запись создана
 -> Ожидание визита -> Визит завершён -> Повторный контакт -> Закрыто

Exception branches: Cancelled / Rescheduled / No-show / Service case
```

Стадия CRM описывает клиентский сервис, а не медицинский процесс. Ни один робот не назначает лечение и не интерпретирует медицинские данные.

## 7. Appointment Management

| Операция | Master | Preconditions | Контроль |
|---|---|---|---|
| Просмотр расписания | МИС | Разрешённый scope и актуальность | Не кэшировать лишние данные |
| Создание записи | МИС | Service, specialist, slot, patient ID | Idempotency и conflict response |
| Подтверждение | МИС/CRM notification owner | Appointment version/status | Одно сообщение по событию |
| Перенос | МИС | Current version и новый slot | Старый слот освобождён master-системой |
| Отмена | МИС | Полномочия, reason category | Связанные reminder/follow-up обновлены |
| Напоминание | Bitrix24/назначенная система | Confirmed appointment и допустимый канал | Минимум данных, delivery result |

```text
Request slot -> MIS availability -> Create appointment -> Confirm
      |                                  |
      +-> Conflict / unavailable         +-> Remind
                                             |-> Attend
                                             |-> Reschedule
                                             |-> Cancel / No-show
```

CRM не рассчитывает доступность врача самостоятельно, если МИС является master. Конкурентная запись, timezone, задержка синхронизации и повтор webhook обязательно тестируются.

## 8. Doctors and Staff Architecture

| Роль/объект | CRM-проекция | Источник | Ограничение |
|---|---|---|---|
| Врач | External ID, public service categories, appointment relation | МИС/HR | Не копировать медицинский профиль пациента |
| Специалист | External ID, разрешённые услуги, location | МИС/HR | Qualification facts остаются в master |
| Оператор | Queue, request, communication task | Bitrix24 | Не принимает клинические решения |
| Администратор клиники | Schedule/service coordination view | МИС + Bitrix24 | Минимальный patient scope |
| Руководитель направления | Агрегированные SLA/load indicators | BI/МИС | Не требует доступа ко всем карточкам |

Загрузка специалиста определяется расписанием и фактом МИС, а не количеством задач Bitrix24. Ответственность фиксируется отдельно для запроса, записи, медицинского приёма и сервисного обращения.

## 9. Service Management

| Компонент | Назначение | Master | CRM-использование |
|---|---|---|---|
| Медицинская услуга | Код и сервисное предложение | МИС/ERP/catalog system | Разрешённая коммерческая проекция |
| Категория/направление | Навигация и аналитика | МИС/catalog | Нормализованный справочник |
| Цена | Коммерческая стоимость и validity | ERP/МИС/pricing master | Timestamp/snapshot в договорном контексте |
| Длительность | Планирование слота | МИС | Только для appointment flow |
| Пакет услуг | Коммерческий состав | ERP/МИС | Package ID и status, без clinical result |
| Повторная программа | Сервисный follow-up | МИС/approved program master | Due date/category и communication policy |

Товарный каталог CRM или Smart Process выбирается после проверки возможностей и master-системы. Название услуги не используется как технический ключ.

## 10. Documents Architecture

| Документ | Владелец | Master/хранилище | CRM-проекция |
|---|---|---|---|
| Договор | Юрист/администратор | ECM/ERP/разрешённая система | ID, type, version, status, secure reference |
| Согласие | Уполномоченный процесс | Назначенная защищённая система | Только минимальный факт/версия, если допустимо |
| Счёт | Бухгалтерия | 1С/ERP | External ID, amount/status при необходимости |
| Акт | Бухгалтерия/ответственный | 1С/ERP/ECM | ID, version, status |
| Внутренний документ | Владелец процесса | Классифицированное хранилище | Metadata без лишнего содержимого |
| Медицинский документ | Медицинский специалист | МИС | Не копируется в CRM; только допустимая ссылка при обосновании |

Версия, owner, classification, retention, доступ и удаление определяются для каждого типа. Прикрепление файла к сделке не считается достаточной системой управления документами.

## 11. Rights Model

| Роль | Пациенты | Записи | Документы | Аналитика |
|---|---|---|---|---|
| Оператор | Минимальные данные очереди/обращения | Создание/перенос через разрешённый flow | Только сервисные шаблоны | Личная/операционная |
| Администратор клиники | Разрешённый scope подразделения | Координационная проекция | Договорные metadata по необходимости | Операционная |
| Врач | Только пациенты/записи, необходимые по роли | Свои подтверждённые записи | Медицинские документы — в МИС, не CRM | Собственный/агрегированный минимум |
| Руководитель направления | Минимальный case-level доступ | Подразделение/агрегаты | По полномочиям | Управленческая по направлению |
| Бухгалтер | Реквизиты/финансовый контекст | Минимальный факт услуги | Счета/акты/договорные metadata | Финансовая |
| Директор | Управленческий scope | Агрегированный обзор | Только необходимое | Управленческая |

Точная реализация CRM-ролей, Smart Process rights, отделов, групп, наследования и экспорта требует проверки на целевом портале. Руководящая роль не означает автоматический доступ ко всем карточкам и документам.

Контроли: deny-by-default для чувствительных данных, least privilege, segregation of duties, service-account minimum scope, offboarding, access review и тест UI/direct URL/search/REST/export.

## 12. Automation Architecture

| Механизм | Сценарий | Защита | Ошибка/эскалация |
|---|---|---|---|
| Робот/задача | Первый ответ | Один marker на request | Fallback owner |
| Уведомление | Подтверждение записи | Appointment version/status | Delivery failure recorded |
| Уведомление | Напоминание | Допустимый канал, минимум данных | No duplicate message |
| Робот | Контроль SLA | Approved calendar/priority | Tiered escalation |
| Задача | Повторный контакт | Due reason category без клинической детали | Reassign/expire policy |
| Бизнес-процесс | Договор/согласие | Version ID и полномочия | Reject/timeout route |
| Робот | No-show/service follow-up | Confirmed MIS status | Не делать медицинских выводов |

```text
Event -> Validate source/version -> Check eligibility -> Execute
  |             |                          |             |
  |             +-> reject safely          |             v
  |                                        |        Record result
  +----------------------------------------+-> bounded retry -> escalate
```

Каждая automation имеет owner, trigger, preconditions, data classification, idempotency key, timeout, retry budget, error route, audit trail и test case. Шаблон не содержит диагноз, название чувствительной процедуры или иной избыточный контекст без отдельного согласования.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| МИС | Patient external ID, service catalog, appointment/status, minimal visit fact | REST/OAuth/application/batch contract, если поддерживается | Auth, idempotency, version, privacy filter, reconcile |
| Сайт | Request and permitted contact/consent context | CRM form/REST after verification | Validation, dedup, spam control |
| Телефония | Call event and CRM relation | Official Bitrix24 telephony/verified provider | Rights, unmatched calls, recording policy |
| Платёжная система | Payment/refund status | Provider API/event through controlled integration | Authenticity, idempotency, amount reconcile |
| BI/DWH | Approved minimal datasets | BI Constructor/controlled export | Access, de-identification, freshness, reconcile |
| 1С/ERP | Contract, invoice, act, finance status | Verified connector/application or contract integration | External IDs, mapping, queue, totals |

```text
Source -> Authenticate -> Minimize -> Validate -> Map -> Idempotency -> Target
  |                                                                  |
  +-> secured audit                                      target result
                 \                                       /
                  +------ retry / reconciliation --------+
```

Контракт фиксирует source of truth, purpose, lawful/approved scope determined externally, direction, keys, schema version, field classification, retention, timezone, event ordering, timeout, retry budget, security, monitoring, recovery и owner.

Наличие готовой интеграции с конкретной МИС, платёжной системой или конфигурацией 1С не предполагается. Приложение проверяется в официальном Маркетплейсе, внешний API — в документации поставщика отдельным scope.

## 14. Analytics

| Отчёт | Метрика | Grain | Ограничение/сверка |
|---|---|---|---|
| Обращения | Count, first response, channel | Request/period | CRM, исключить дубли |
| Конверсия запись/визит | Requests → appointments → completed visits | Cohort/request/appointment | Факт визита только из МИС |
| Загрузка специалистов | Booked/available/completed slots | Specialist-period | МИС schedule; не CRM tasks |
| Выручка | Confirmed revenue/payment | Service/order-period | 1С/ERP/finance |
| Повторные визиты | Eligible repeat/completed appointments | Patient cohort | Stable identity и approved definition |
| Качество сервиса | Response, resolution, no-show handling, permitted feedback | Case/appointment-period | Не интерпретировать клинический исход |
| Интеграции | Success/error/lag/reconcile delta | Flow-period | Technical monitoring |

```text
CRM + MIS + ERP/1C + Payments
 -> Purpose-limited data contracts -> Quality/privacy gate
 -> Dataset/DWH -> Metric dictionary -> Dashboard -> Decision
                                      ^                |
                                      +-- reconcile ---+
```

BI Constructor, datasets и поля зависят от портала. Metric dictionary фиксирует formula, grain, cohort, filters, timezone, currency, data classification, owner, freshness и reconciliation source. Для BI используются минимизированные или обезличенные наборы, когда это соответствует задаче.

## 15. Security and Privacy

### Инженерные принципы

| Принцип | Реализация | Проверка |
|---|---|---|
| Data minimization | Только поля, необходимые CRM-процессу | Field inventory и owner review |
| Purpose limitation | Для каждого поля/потока определена бизнес-цель | Data register |
| Role separation | Оператор, врач, бухгалтер и аналитик имеют разный scope | Role tests |
| Document control | Оригиналы в защищённой master-системе | Links, versions, access review |
| Secret management | Tokens/webhooks не в CRM notes, Git или логах | Secret scan/rotation procedure |
| Auditability | Изменения и интеграционные события трассируемы | Audit sample/RCA |
| Retention | Сроки определяет утверждённая политика | Scheduled review/deletion process |
| Safe analytics | Минимизация/обезличивание и агрегирование | Dataset review |

```text
Collect minimum -> Classify -> Authorize -> Use for stated purpose
 -> Log safely -> Retain by policy -> Restrict/Delete by approved process
```

Кейс не утверждает соответствие законодательству или отраслевому стандарту. До внедрения обязательны отдельные решения юридической службы, ИБ, медицинского руководителя и владельца данных, включая Cloud/On-Premise, место хранения, резервные копии, экспорт, поддержку и субподрядчиков.

## 16. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Новое обращение | Validation, dedup, owner | Один request и первый шаг |
| 2 | Повторное обращение | Existing identity/new request | Нет дубля профиля |
| 3 | Создание записи | MIS availability/version | Один appointment external ID |
| 4 | Конкурентная запись | Два запроса на слот | Только master подтверждает результат |
| 5 | Повтор appointment event | Idempotency | Дубликат не создан |
| 6 | Перенос | Old/new slot/version | CRM и МИС согласованы |
| 7 | Отмена | Reason/status/reminders | Напоминание снято, история сохранена |
| 8 | No-show | Confirmed MIS status | Корректный service follow-up |
| 9 | Напоминание | Eligibility/template/delivery | Одно минимизированное сообщение |
| 10 | Ошибка доставки сообщения | Error route | Alert без раскрытия данных |
| 11 | Завершённый визит | Minimal status filter | Clinical payload не попал в CRM |
| 12 | Оплата | External ID/amount/state | Статус сверяется с finance source |
| 13 | Обращение после лечения | Classification/SLA/owner | Service case без клинической интерпретации |
| 14 | Документ | Version/rights/direct link | Только разрешённая роль получает доступ |
| 15 | Проверка прав | UI/URL/search/REST/export | Scope соответствует матрице |
| 16 | Ошибка МИС | Timeout/retry/dead event | Нет дубля/потери, alert трассируем |
| 17 | События не по порядку | Version/sequence | Статус не откатился |
| 18 | Миграция | Dry-run/sample/minimization/totals | Только разрешённые поля перенесены |
| 19 | BI dataset | Field-level review/access | Клинические/идентифицирующие данные исключены по design |
| 20 | Rollback | Trigger/runbook/reconcile | Безопасное состояние восстановлено |

Тестовые данные только синтетические. UAT подписывают Operations, Medical-system owner, Clinical governance representative, Finance, Legal/Privacy, Information Security и Integration owners. Негативные, повторные, конкурентные, role и recovery-сценарии обязательны.

## 17. Deployment

| Этап | Действия | Gate |
|---|---|---|
| Discovery | AS IS, portal, MIS, data classification, owners, constraints | Inventory и scope подписаны |
| Архитектура | Data model, master matrix, privacy/security review, ADR | TO BE утверждён |
| Настройка | Test portal/funnel, fields, stages, rights | Peer/security review и smoke |
| Миграция | Minimize, map, dry-run, sample, reconcile | Data owner acceptance |
| Интеграции | Contract, privacy filter, failure/recovery tests | End-to-end успешен |
| Обучение | Role-based and prohibited-data guidance | UAT/readiness sign-off |
| Запуск | Freeze, backup/export, cutover, smoke | Go/no-go |
| Hypercare | Monitoring, incident route, daily reconcile | Нет critical defects, support принял систему |

```text
Discovery -> Data classification -> Test configuration -> Contract tests
 -> Sample migration -> UAT -> Go/No-Go -> Cutover -> Smoke -> Hypercare
                                      |
                                      +-> Rollback by approved trigger
```

Cloud и On-Premise оцениваются отдельно юридической службой и ИБ. Для коробочной версии дополнительно проверяются версия продукта/модулей, PHP, БД, файлы, cron, backup/restore, инфраструктура и безопасность. Ядро продукта не изменяется.

## 18. Monitoring

| Контроль | Сигнал | Owner | Реакция |
|---|---|---|---|
| Необработанные обращения | Нет owner/первого ответа | Operations lead | Assign и RCA |
| Пропущенные записи | No-show/unknown status threshold | Clinic operations | Verify MIS and follow-up policy |
| Расхождение CRM/МИС | Appointment status/version delta | Integration/data owner | Reconcile, source correction |
| Ошибки интеграций | Error/retry/dead event | Integration owner | Recover/reconcile/incident |
| Напоминания не доставлены | Delivery failure | Communication owner | Approved alternative route |
| Обращение без ответа | SLA threshold | Service lead | Reassign/escalate |
| Клинические данные в CRM | Detection/review finding | Privacy/Security owner | Contain, restrict, incident process |
| Финансовое расхождение | CRM ≠ 1С/ERP | Finance owner | Reconciliation |
| Аномальный доступ/экспорт | Security signal | Security owner | Contain, audit, revoke |

SLI: success/error rate, latency, backlog age, retry count, last successful sync, stale age и reconciliation delta. Логи маскируют идентификаторы и не содержат медицинские сведения, документы, секреты или полные контактные данные.

## 19. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | CRM используется как МИС | Неверная граница | Клинические данные в неподходящем контуре | System context/MIS master |
| 2 | Диагноз хранится в CRM | Свободное поле | Высокий privacy/security risk | Запрет field/content review |
| 3 | Результат исследования в комментарии | Удобство | Утечка/неуправляемая копия | MIS only |
| 4 | Медицинский документ приложен к сделке | Нет classification | Избыточный доступ | Secure master/reference |
| 5 | Дубли пациентов | Нет identity policy | Разорванная история/ошибка записи | Matching and review |
| 6 | Ошибочное объединение пациентов | Слабый ключ | Чужая история/коммуникация | Conservative resolution |
| 7 | Email/телефон — единственный ключ | Shared/changed contact | Неверная идентичность | Multi-factor approved matching |
| 8 | Повторное событие создаёт запись | Нет idempotency | Двойной слот | Appointment external ID |
| 9 | Конкурентная запись | CRM кэширует availability | Overbooking | MIS atomic confirmation |
| 10 | Перенос не освобождает слот | Нет transaction/saga | Потеря ёмкости | MIS operation/reconcile |
| 11 | Отмена не снимает напоминание | Контуры не связаны | Некорректное сообщение | Compensating event |
| 12 | Плановый визит считается завершённым | Неверная семантика | Ложная аналитика | MIS fact status |
| 13 | События не по порядку | Async integration | Откат статуса | Sequence/version |
| 14 | Неизвестный статус считается успехом | Оптимистичный mapping | Ошибочное закрытие | Exception state |
| 15 | Расписание устарело | Sync lag | Неверная запись | Freshness/MIS check |
| 16 | Услуга сопоставлена неверно | Разные справочники | Ошибка записи/цены | Service ID mapping |
| 17 | Цена устарела | Нет timestamp | Спор/ошибка счёта | Pricing master/validity |
| 18 | Оплата отмечена вручную | CRM подменяет finance | Ложный paid state | Finance confirmation |
| 19 | Повтор payment event | Нет event ID | Дублированный эффект | Idempotency |
| 20 | Платёжные данные в CRM | Избыточный payload | Security risk | Token/reference only |
| 21 | Напоминание раскрывает услугу | Избыточный шаблон | Privacy incident | Neutral/minimal template |
| 22 | Маркетинговое сообщение без eligibility | Нет consent/suppression | Жалоба/правовой риск | Approved communication policy |
| 23 | Запись разговора без проверки | Настройка по умолчанию | Legal/privacy risk | Legal/IB review |
| 24 | Оператор даёт клинический совет | Нет границы script | Риск для пациента | Escalation to qualified process |
| 25 | Робот интерпретирует симптомы | Неверная automation | Опасное решение | Запрет clinical decision automation |
| 26 | Follow-up раскрывает причину визита | Персонализированный текст | Утечка | Minimal category/message |
| 27 | Врач видит всех пациентов | Широкая роль | Excess access | Need-to-know |
| 28 | Бухгалтер видит обращения | Скопирована роль | Excess access | Field/entity scope |
| 29 | Маркетолог видит документы | Нет segregation | Privacy breach | Aggregated segment only |
| 30 | Администратор работает как бизнес-пользователь | Удобство | Неконтролируемые изменения | Separate roles/accounts |
| 31 | Service account — администратор | Быстрая интеграция | Большой blast radius | Least scope |
| 32 | Уволенный сотрудник сохраняет доступ | Нет offboarding | Data breach | Revoke/reassign/rotate |
| 33 | Webhook URL хранится в README | Секрет считают ссылкой | Компрометация | Secret store/env |
| 34 | Payload МИС пишется в лог | Debug целиком | Medical data leak | Allowlist/masking |
| 35 | Ошибка API возвращает данные в alert | Небезопасное логирование | Утечка в канале уведомления | Error sanitization |
| 36 | Retry бесконечный | Нет budget | Нагрузка/backlog | Bounded backoff |
| 37 | Retry неидемпотентен | Нет event key | Дубли | Idempotency |
| 38 | REST limits/timeout | Пиковая нагрузка | Lag записи | Queue/backoff/verified batch |
| 39 | Свободная двусторонняя правка | Нет field owner | Расхождение CRM/МИС | Master matrix |
| 40 | Импорт без минимизации | Переносят всё | Excess sensitive data | Field allowlist |
| 41 | Полный импорт без dry-run | Спешка | Массовая ошибка | Sample/report/rollback |
| 42 | Нет reconciliation | HTTP success принят за итог | Тихие пропуски | Totals/sample/delta |
| 43 | Backup содержит лишние данные | Нет inventory | Утечка/retention risk | Classified backup policy |
| 44 | Backup не восстанавливается | Нет restore drill | Rollback невозможен | Test restore |
| 45 | Тариф не поддерживает права/процессы | Capability check пропущен | Нереализуемая модель | Portal verification |
| 46 | Marketplace app не оценено | Доверие карточке | Vendor/security risk | Assessment/exit plan |
| 47 | On-Premise устарел | Старые модули/PHP | Ошибки/уязвимости | Supported update plan |
| 48 | BI dataset содержит identifiers | Нет privacy review | Excess analytics access | Minimize/de-identify |
| 49 | Конверсия считает плановые визиты | Неверный факт | Завышенный KPI | MIS completed status |
| 50 | Загрузка считается по задачам | Неверный grain | Ошибочное планирование | MIS schedule source |
| 51 | Качество сервиса смешано с лечением | Нет metric boundary | Некорректные выводы | Service-only KPIs |
| 52 | Dashboard без freshness | Старые данные | Ошибочное решение | Last refresh/SLA |
| 53 | Нет owner интеграции | Проект завершён | Ошибки без реакции | Support RACI |
| 54 | Нет privacy/security incident route | Процесс не определён | Запоздалая реакция | Approved incident plan |
| 55 | Нет hypercare | Система передана сразу | Инциденты накапливаются | Support window |

## 20. ADR

### ADR-HC-001: Модель пациента

- **Контекст:** CRM нужна идентичность для сервиса, но медицинская карта должна оставаться в МИС.
- **Решение:** Contact хранит минимальный сервисный профиль и внешний patient ID; clinical data запрещены.
- **Альтернативы:** полный профиль МИС в CRM; только анонимный request.
- **Последствия:** conservative matching, minimization и strict rights.
- **Статус:** Proposed; требует Legal/Privacy/Medical/IB approval.

### ADR-HC-002: Модель записи

- **Контекст:** слот и назначение управляются МИС.
- **Решение:** Smart Process «Запись» — минимальная CRM-проекция; создание/перенос/отмена подтверждаются МИС.
- **Альтернативы:** CRM как master расписания; только ссылка без статуса.
- **Последствия:** external ID, version, concurrency и reconciliation.
- **Статус:** Proposed.

### ADR-HC-003: Каталог услуг

- **Контекст:** услуга имеет код, цену, длительность и доступность.
- **Решение:** МИС/ERP/catalog — master; CRM хранит разрешённую коммерческую проекцию.
- **Альтернативы:** CRM-каталог как master; свободный текст.
- **Последствия:** mapping, validity и snapshot договорного контекста.
- **Статус:** Proposed.

### ADR-HC-004: Документы

- **Контекст:** договорные, согласия, финансовые и медицинские документы имеют разную чувствительность.
- **Решение:** оригиналы хранятся в назначенных защищённых системах; CRM содержит metadata/reference по allowlist.
- **Альтернативы:** все файлы в сделке.
- **Последствия:** classification, version, retention и access review.
- **Статус:** Proposed.

### ADR-HC-005: Модель прав

- **Контекст:** оператор, врач, бухгалтер и руководитель требуют разный scope.
- **Решение:** deny-by-default и role-based least privilege с тестами UI/URL/REST/export.
- **Альтернативы:** общая роль; доступ только по ответственному без field review.
- **Последствия:** test accounts, offboarding и periodic access review.
- **Статус:** Proposed.

### ADR-HC-006: Интеграция МИС

- **Контекст:** МИС содержит расписание и клинический контур.
- **Решение:** purpose-limited allowlist contract, минимальные события, idempotency и reconciliation.
- **Альтернативы:** полная двусторонняя синхронизация; ручной CSV.
- **Последствия:** privacy filter, monitoring, incident route и data owner.
- **Статус:** Proposed.

### ADR-HC-007: Аналитическая архитектура

- **Контекст:** CRM, МИС и finance содержат разные факты и уровни чувствительности.
- **Решение:** минимизированные datasets, metric dictionary, access segmentation и reconciliation.
- **Альтернативы:** только CRM-отчёты; выгрузка всех полей.
- **Последствия:** dataset review, stable identity и freshness monitoring.
- **Статус:** Proposed.

### ADR-HC-008: Лиды или Request-First

- **Контекст:** не каждое обращение является коммерческой возможностью.
- **Решение:** лид применяется только для подтверждённого неквалифицированного intake; пациентский запрос ведёт сервисный lifecycle.
- **Альтернативы:** лид на каждый визит; Deal-First для всех обращений.
- **Последствия:** выбор фиксируется до настройки каналов и KPI.
- **Статус:** Open до Discovery.

## 21. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | CRM объявлена МИС | Неверный scope | Медданные в неподходящем контуре | System context |
| 2 | Диагноз хранится в Contact | Желание единой карточки | Excess sensitive data | MIS only |
| 3 | Анамнез в комментарии | Свободный текст | Неуправляемая копия | Prohibited-content guidance |
| 4 | Результаты исследований прикреплены к сделке | Удобство | Excess access | Secure MIS/ECM |
| 5 | Назначение врача хранится в задаче | CRM подменяет clinical record | Риск ошибки | MIS only |
| 6 | Оператор записывает симптомы подробно | Нет script boundary | Sensitive data spread | Minimal service category |
| 7 | Робот рекомендует услугу по симптомам | Автоматизация без governance | Clinical decision risk | Prohibit/escalate |
| 8 | Новый визит создаёт новый Contact | Нет identity policy | Дубли пациента | Matching process |
| 9 | Телефон — единственный ключ | Shared/changed number | Wrong merge | Conservative multi-field match |
| 10 | Email — единственный ключ | Shared address | Wrong identity | Approved resolution |
| 11 | Автоматическое merge без review | Желание очистить дубли | Смешение людей | Human-controlled merge |
| 12 | Patient ID показан в уведомлении | Избыточный шаблон | Disclosure | Minimal neutral message |
| 13 | Каждый визит создаёт лид | Шаблон без анализа | Мусор CRM | Request/appointment model |
| 14 | Одна сделка на всю жизнь пациента | Смешение процессов | Неверная аналитика | Defined commercial unit |
| 15 | Запись хранится только задачей | Task replaces entity | Нет status/version | Appointment projection |
| 16 | CRM самостоятельно хранит свободные слоты | Нет MIS boundary | Double booking | MIS availability |
| 17 | Повтор webhook создаёт запись | Нет idempotency | Duplicate appointment | External ID |
| 18 | Перенос создаёт вторую активную запись | Нет version/compensation | Double slot | MIS transaction/reconcile |
| 19 | Отмена удаляет историю | Желание очистить список | Нет audit | Cancel status/reason |
| 20 | Отмена не снимает reminder | Контуры не связаны | Неверное сообщение | Compensating event |
| 21 | Plan date считается фактом визита | Semantics не разделены | Завышенная конверсия | MIS completed status |
| 22 | No-show определяется вручную оператором | Нет source | Неверный follow-up | MIS confirmation |
| 23 | События применяются не по порядку | Async | Status rollback | Version/sequence |
| 24 | Неизвестный status = completed | Оптимистичный mapping | Ложный факт | Exception state |
| 25 | Название услуги — ключ | Нет service ID | Wrong mapping | Stable code |
| 26 | Услуга в свободном тексте | Быстрое поле | Нельзя анализировать | Controlled catalog |
| 27 | Текущая цена переписывает историю | Нет snapshot | Неверный договор/отчёт | Price context snapshot |
| 28 | Цена без валюты/validity | Неполная модель | Финансовый спор | Amount/currency/date |
| 29 | Пакет услуг хранится строкой | Нет composition model | Неверный остаток пакета | Package ID/master |
| 30 | Повторный визит создаётся без основания | Growth automation | Нежелательная коммуникация | Approved eligibility |
| 31 | Follow-up текст раскрывает процедуру | Избыточная персонализация | Privacy incident | Neutral template |
| 32 | Напоминание отправляют из двух систем | Нет owner | Дубли/жалобы | Communication ownership |
| 33 | Не проверяется delivery result | Fire-and-forget | Пациент не уведомлён | Delivery monitoring |
| 34 | Retry отправляет сообщение повторно | Нет message key | Спам | Idempotency |
| 35 | Маркетинг смешан с транзакционным сообщением | Один шаблон | Consent risk | Message classification |
| 36 | Отписка не учитывается | Списки расходятся | Нежелательная рассылка | Suppression master |
| 37 | Запись разговора включена без review | Default setting | Privacy/legal risk | Legal/IB decision |
| 38 | Врач видит все CRM-карточки | Широкая роль | Excess access | Need-to-know |
| 39 | Оператор видит медицинские документы | Файлы в CRM | Excess access | MIS/secure reference |
| 40 | Бухгалтер видит переписку | Скопирована роль | Privacy breach | Financial-only scope |
| 41 | Маркетолог экспортирует Contacts | Нет restrictions | Data leak | Aggregated segments/access control |
| 42 | Руководитель автоматически администратор | Role confusion | Uncontrolled changes | Separate admin role |
| 43 | Service account — администратор | Быстрая настройка | Large blast radius | Least scope |
| 44 | Уволенный сотрудник сохраняет доступ | Нет offboarding | Breach | Revoke/reassign/rotate |
| 45 | Прямые URL/REST/export не тестируют | Проверяют UI | Rights bypass | Multi-channel tests |
| 46 | Согласие хранится checkbox без версии | Упрощение | Нельзя подтвердить контекст | Versioned evidence in master |
| 47 | Документ копируется в несколько сделок | Нет master | Разные версии | Metadata/reference |
| 48 | Медицинский документ индексируется CRM-поиском | Неверное хранилище | Wider exposure | MIS only |
| 49 | Нет retention policy | Хранят всё | Excess long-term data | Approved retention process |
| 50 | Удаляют вручную без dependency check | Спешка | Broken history/audit | Controlled procedure |
| 51 | Webhook URL в README | Secret misclassification | Compromise | Secret store/env |
| 52 | MIS payload логируется целиком | Debug | Sensitive data leak | Allowlist/masking |
| 53 | Error alert содержит payload | Нет sanitization | Leak into messenger/email | Sanitized errors |
| 54 | Нет timeout | Happy path | Hung integration | Explicit timeout |
| 55 | Retry бесконечный | Нет budget | Load/backlog | Bounded backoff |
| 56 | Retry неидемпотентен | Нет event ID | Duplicates | Idempotency |
| 57 | Ошибки silently ignored | Нет dead-letter | Missing appointments | Failed-event register |
| 58 | Свободная двусторонняя правка | Нет field owner | CRM/MIS conflict | Master matrix |
| 59 | Batch без проверки метода | Желание ускорить | Partial failure | Verify limits/semantics |
| 60 | Импортирует всю МИС | Нет minimization | Sensitive overcollection | Field allowlist |
| 61 | Полный импорт без dry-run | Спешка | Mass corruption | Sample first |
| 62 | Реальные пациенты используются в тесте | Нет synthetic dataset | Privacy risk | Synthetic-only QA |
| 63 | Нет source-target журнала | Migration opaque | No rollback/reconcile | Mapping report |
| 64 | Нет проверки totals/sample | HTTP success принят за результат | Silent omissions | Reconciliation |
| 65 | Backup содержит лишние данные | Нет classification | Exposure | Classified backup policy |
| 66 | Backup не восстановлен тестово | Formal control | Rollback fails | Restore drill |
| 67 | SLA не учитывает календарь | Простая разница дат | False breach | Calendar definition |
| 68 | Пауза SLA без reason | KPI manipulation | False performance | Controlled pauses |
| 69 | Обращение закрыто отправкой ответа | Activity = outcome | Issue unresolved | Exit criteria |
| 70 | Жалоба на сервис смешана с клинической претензией | Нет classification | Wrong route | Governance taxonomy |
| 71 | Service KPI трактуется как clinical outcome | Metric boundary absent | Misleading conclusion | Separate domains |
| 72 | Загрузка врача считается по CRM tasks | Wrong source/grain | Bad planning | MIS schedule |
| 73 | Revenue берётся из stage | CRM status not finance fact | Overstated report | ERP/finance reconcile |
| 74 | Repeat visit определяется по совпадению email | Weak identity | False repeat rate | Stable patient ID |
| 75 | BI выгружает identifiers без необходимости | Convenience | Excess access | De-identify/minimize |
| 76 | Dashboard не показывает freshness | Stale data | Wrong decision | Last refresh/SLA |
| 77 | Marketplace app не оценено | Trust listing | Vendor/security risk | Assessment/exit plan |
| 78 | Запуск без Legal/IB/Medical review | Focus on configuration | Unaccepted risk | Formal gates |
| 79 | Обучили только администраторов | No role training | Staff enter prohibited data | Role guidance/UAT |
| 80 | Нет hypercare/incident RACI | Project closed immediately | Incidents unowned | Support window |

## 22. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Подтверждены AS IS, данные, системы и ограничения | Inventory, data classification, stakeholders, capability check | Masters и owners определены |
| 2. Архитектура | Согласован TO BE | Data model, master matrix, rights, ADR, risks | Medical/Legal/Privacy/IB owners дали требуемые решения |
| 3. CRM настройка | Работает минимальный сервисный контур | Fields allowlist, stages, roles, dedup, tests | Smoke/rights tests успешны |
| 4. Записи и услуги | Работает appointment/service projection | Mapping, external IDs, reminder rules | Сквозной synthetic journey пройден |
| 5. Интеграции | Работают purpose-limited MIS/finance flows | Contracts, privacy filters, retries, monitoring, reconcile | Failure/recovery tests успешны |
| 6. Тестирование | Подтверждена готовность | Test report, security/privacy findings, UAT | Нет critical/high defects |
| 7. Запуск | Выполнен управляемый cutover | Runbook, backup, rollback, communications | Smoke/reconcile успешны |
| 8. Оптимизация | Стабилизированы SLA, доступ и KPI | Baseline, access review, backlog, lessons learned | Support model принят owners |

### Официальные источники

Используются только источники из `knowledge/SOURCES.md`; дата последней проверки реестра — 2026-08-05. Эти источники подтверждают возможности Bitrix24, но не заменяют отраслевые и юридические требования, которые находятся вне реестра и потому не цитируются в кейсе.

| Источник | Применение |
|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | Пользовательские возможности CRM и Cloud |
| [Документация REST API](https://apidocs.bitrix24.ru/) | Интеграционная архитектура и ограничения |
| [REST API reference](https://apidocs.bitrix24.ru/api-reference/) | Проверка контрактов методов и полей |
| [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/index.html) | Проверка вариантов интеграционного доступа |
| [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) | Проектирование request, appointment, service, contract и follow-up views |
| [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) | Проверка automation |
| [Как выбрать тип бизнес-процесса](https://helpdesk.bitrix24.ru/open/23951294/) | Выбор approval/orchestration механизма |
| [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) | Проверка вариантов телефонии |
| [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) | Проверка datasets и аналитики |
| [Маркетплейс Bitrix24](https://www.bitrix24.ru/apps/) | Проверка приложений и коннекторов |
| [Документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php) | On-Premise возможности и ограничения |

Если функция, поле, связь, событие, REST-метод, приложение или ограничение не подтверждены официальным источником и целевым порталом, они помечаются **«требует проверки»** и не включаются в обязательства перед заказчиком.
