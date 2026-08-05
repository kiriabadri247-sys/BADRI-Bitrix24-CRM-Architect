# Logistics CRM Architecture Case

Архитектурный кейс для синтетической логистической компании, которая принимает заявку, рассчитывает перевозку, подтверждает заказ, назначает ресурсы, контролирует выполнение, доставку, документы, оплату и сервис. Это референсная модель, а не готовая конфигурация для копирования.

> **Граница безопасности:** реальные клиенты, водители, номера транспорта, маршруты, координаты, адреса, телефоны, документы, CRM ID, webhook URL, токены и выгрузки не используются. Все примеры обезличены или синтетические.

> **Архитектурная граница:** Bitrix24 отвечает за CRM, клиентский контекст, коммерческий lifecycle, межфункциональную координацию и сервис. TMS остаётся master планирования и исполнения перевозки; GPS/телематическая система — фактической телеметрии; ERP/1С — договорных, бухгалтерских и финансовых фактов. Bitrix24 не объявляется TMS или системой мониторинга транспорта без отдельного подтверждённого проекта.

Обозначения:

- **факт Bitrix24** — подтверждается официальным источником из `knowledge/SOURCES.md`;
- **решение архитектора** — целевой выбор кейса, подлежащий согласованию на Discovery;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа/редакции, версии, прав, приложения, региона или внешнего провайдера.

## 1. Business Context

Референсный сценарий — B2B-перевозки автомобильным транспортом с возможностью собственного и привлечённого парка. Заявка клиента проходит расчёт, согласование, назначение ресурсов, исполнение, подтверждение доставки и финансовое закрытие. Для мультимодальных, международных, опасных, температурных и иных регулируемых перевозок вводятся отдельные требования и системы контроля.

| Аспект | Целевая интерпретация |
|---|---|
| Модель продаж | Отдельная коммерческая возможность на заказ/рамочный объём по утверждённому правилу |
| Операционная единица | Рейс или перевозка с устойчивым внешним ID |
| Планирование | TMS/диспетчерский контур |
| Телеметрия | GPS/телематический провайдер |
| Клиентский сервис | CRM-обращение или претензия, связанная с перевозкой |
| Финансовый факт | ERP/1С/учётная система |

### Участники

- заказчик, грузоотправитель и грузополучатель;
- менеджер продаж и руководитель;
- логист и диспетчер;
- водитель и перевозчик/партнёр;
- склад и терминал;
- бухгалтерия и юрист;
- служба качества и клиентского сервиса;
- интеграционная и аналитическая команды.

### Цели CRM

1. Не терять заявки и следующий шаг.
2. Связать клиента, расчёт, коммерческую сделку и фактическую перевозку.
3. Сделать сроки, ресурсы, исключения и ответственность наблюдаемыми.
4. Сократить потери данных между продажами, логистами, диспетчерами и бухгалтерией.
5. Управлять обращениями и претензиями по SLA.
6. Получать сверяемые показатели перевозок, сроков, загрузки и прибыльности.

```text
Demand -> Sales -> Calculation -> Dispatch -> Transport execution
  -> Delivery confirmation -> Documents -> Finance -> Service
```

## 2. AS IS Problems

| Проблема AS IS | Проявление | Корневая причина | Целевой контроль |
|---|---|---|---|
| Заявки теряются | Нет owner и первого действия | Каналы не объединены | Intake, routing, fallback и alert |
| Клиентская история разорвана | Заказы и обращения в разных системах | Нет stable customer/order keys | Relations и identity policy |
| Расчёт не связан с заказом | Несколько версий в почте | Нет version control | Calculation ID/revision |
| Транспорт назначается вручную | Конфликты и двойное назначение | Нет TMS/master policy | Resource confirmation из master |
| Нет контроля сроков | Просрочка видна после звонка клиента | Нет plan/fact и milestone monitoring | ETA/status/exception controls |
| GPS и CRM показывают разное | Статусы выводятся независимо | Нет mapping/reconciliation | TMS/GPS contracts |
| Документы теряются | Файлы в чатах и почте | Нет lifecycle и актуальной версии | Metadata, owner, status |
| Оплата отмечается менеджером | Неверная задолженность | CRM подменяет учётную систему | Finance confirmation |
| Претензии хранятся комментариями | Нет SLA и RCA | Нет отдельного объекта | Claim lifecycle |
| Аналитика расходится | Разные единицы и grain | Нет metric dictionary | Data contracts и reconciliation |

## 3. TO BE Architecture

### Целевые контуры

| Контур | Сущность/система | Назначение | Master |
|---|---|---|---|
| CRM | Лид, контакт, компания, сделка | Клиент, продажа и коммерческое обязательство | Bitrix24/MDM по policy |
| Intake | Smart Process «Заявка на перевозку» | Требования и handoff в расчёт | Bitrix24 либо TMS по ADR |
| Planning | Smart Process «Маршрут» | CRM-проекция плана точек и сроков | TMS |
| Execution | Smart Process «Рейс» | Наблюдаемость исполнения перевозки | TMS |
| Fleet | Smart Process «Транспорт» | Минимальная разрешённая проекция ресурса | TMS/FMS |
| Workforce | Smart Process «Водитель» | Минимальная проекция назначения и доступности | HR/TMS/FMS |
| Service | Smart Process «Претензия» | Клиентское несоответствие, SLA и решение | Bitrix24, с фактами из TMS/ERP |
| Telemetry | GPS/телематика | Координаты и события движения | GPS provider/FMS |
| Finance | ERP/1С | Счёт, акт, оплата, себестоимость | ERP/1С |

**Факт Bitrix24:** CRM, Smart Processes, роботы, триггеры и бизнес-процессы описаны официальными материалами. Конкретные поля, связи, права, automation и доступность проверяются на целевом портале.

**Решение архитектора:** Bitrix24 хранит только необходимую проекцию оперативных данных. Детальные координаты, оптимизация маршрута, график водителя, техническое состояние и диспетчерские операции остаются в специализированных системах.

```text
Lead/Request -> Contact/Company -> Deal
                                  |
                                  v
                         Transportation Request
                                  |
                                  v
        TMS <-> Route <-> Trip <-> Vehicle/Driver
         |                 |
         v                 v
      ERP/1C         GPS/Telematics
         |                 |
         +------> Bitrix24 status view
                          |
                          v
                    Case / Claim / BI
```

### Правила целостности

- один подтверждённый заказ/рейс имеет устойчивые `external_request_id` и `external_trip_id`;
- маршрут, рейс и коммерческая сделка — разные сущности;
- GPS-событие не переводит коммерческую сделку напрямую без нормализации и проверки;
- транспорт и водитель назначаются только через систему-владельца;
- повторное событие не создаёт второй рейс, документ или оплату;
- для каждого поля определены master, direction, freshness и conflict rule;
- массовые изменения имеют dry-run, sample, audit report, reconciliation и rollback.

## 4. CRM Model

| Сущность | Когда использовать | Ключевые данные | Ограничение |
|---|---|---|---|
| Лид | Неквалифицированный запрос, если lead-based процесс подтверждён | Канал, контакт, потребность, owner | Не создавать автоматически для каждого TMS-события |
| Контакт | Физическое лицо клиента/партнёра | Роль, допустимые каналы, relation | Не использовать как карточку юрлица |
| Компания | Заказчик, перевозчик, партнёр | Segment, account owner, external ID | Реквизиты имеют назначенный master |
| Сделка | Самостоятельная коммерческая возможность | Request, amount, terms, stage, owner | Не подменяет рейс или маршрут |
| Обращение | Вопрос/операционное исключение клиента | Category, trip/order, priority, SLA | Претензия выделяется при формальном несоответствии |

```text
Company 1 ---- N Contact
   |
   +---- N Deal ---- 1..N Transportation Request
                         |
                         +---- N Trip ---- N Case / Claim
```

Лид или Deal-First выбираются до настройки каналов. Повторная перевозка существующего клиента создаёт новый заказ/сделку по принятой коммерческой модели, но не новый профиль клиента.

## 5. Logistics Data Model

### Smart Processes и поля

| Сущность | Ключевые поля | Контроль качества |
|---|---|---|
| Заявка на перевозку | Request ID, client, cargo class, quantity/weight/volume, origin/destination zones, dates, service level, status | Единицы, timezone и версия требований обязательны |
| Маршрут | Route ID, version, ordered stops, plan distance/time, constraints | Геоданные минимизированы; TMS — master |
| Рейс | Trip ID, request, route version, vehicle/driver refs, plan/fact milestones, status | State mapping и source timestamp |
| Транспорт | Vehicle ID, type, capacity, availability projection, owner/carrier | Не хранить реальные номера в кейсе; freshness обязательна |
| Водитель | Driver ID, qualification flags, availability projection, assigned trip | Минимизация ПДн и строгие права |
| Претензия | Claim ID, client, trip, category, severity, amount context, owner, decision | Симптом, причина и решение разделены |

```text
Клиент -> Заявка -> Расчёт -> Назначение транспорта
 -> Перевозка -> Доставка -> Закрытие
       |            |            |
       v            v            v
    Маршрут       Рейс       Документы/Оплата
                     |
                     +-> Обращение / Претензия
```

### Кардинальность

| Отношение | Кардинальность | Правило |
|---|---:|---|
| Company → Request | 1:N | Новый самостоятельный запрос не перезаписывает прошлый |
| Request → Calculation | 1:N revisions | Заказ ссылается на утверждённую версию |
| Request → Trip | 1:N | Разделение по плечам/рейсам определяется TMS-моделью |
| Route → Stop | 1:N ordered | Порядок и версия управляются master-системой |
| Trip → Vehicle | N:1 over assignment period | Конфликт назначения контролирует TMS |
| Trip → Driver | N:1 over assignment period | Квалификация и режим труда проверяются вне CRM по policy |
| Trip → Claim | 1:N | Претензия связана с фактическим контекстом |

## 6. Sales Pipeline

| Стадия | Цель | Обязательные данные | Действия | Автоматизация |
|---|---|---|---|---|
| Новая заявка | Зарегистрировать запрос | Канал, клиент, грузовой контекст, даты, owner | Validation/dedup | Назначить owner и первый контакт |
| Квалификация | Подтвердить выполнимость и scope | Точки, груз, объём, service level, ограничения | Уточнить требования | Контроль SLA и неполных полей |
| Расчёт перевозки | Получить управляемую стоимость | Revision, маршрутный контекст, cost result, validity | Рассчитать в TMS/ERP/approved process | Задача и контроль версии |
| Коммерческое предложение | Согласовать условия | Цена, валюта, срок, revision, approvals | Подготовить/отправить КП | Follow-up и expiry alert |
| Подтверждение заказа | Зафиксировать обязательство | Approved terms, contract/order reference | Release handoff | Идемпотентно создать/связать request в TMS |
| Назначение ресурсов | Подтвердить транспорт/водителя | Trip ID, vehicle/driver refs, plan | TMS allocation | Alert при отсутствии ресурса |
| Выполнение перевозки | Контролировать milestones/exceptions | Plan/fact, normalized status, blocker | Dispatch/service actions | Просрочка/exception escalation |
| Закрытие | Подтвердить доставку, документы и критерий успеха | Delivery fact, document/finance references | Service handoff | Completeness check и follow-up |

```text
Новая заявка -> Квалификация -> Расчёт перевозки -> КП
 -> Подтверждение заказа -> Назначение ресурсов
 -> Выполнение перевозки -> Закрытие

Exception: отказ / пауза / отмена / инцидент / претензия
```

Стадии основаны на результате, а не на факте активности. Плановая дата не считается фактической доставкой; GPS-координата не считается подтверждением юридического закрытия.

## 7. Transportation Management

### Ресурсы и исполнение

| Объект | Master | Проекция в Bitrix24 | Контроль |
|---|---|---|---|
| Транспорт | TMS/FMS | ID, type, capacity class, assignment/status | Freshness и minimal data |
| Водитель | HR/TMS/FMS | ID, qualification/availability flags, trip relation | Privacy и role scope |
| Маршрут | TMS | Version, stops summary, plan dates/distance | Route version |
| Рейс | TMS | Plan/fact milestones, normalized status, exceptions | Event ordering/reconciliation |
| Перевозчик | ERP/MDM/TMS | Company relation, qualification status | External ID и ownership |

```text
Confirmed request -> TMS planning -> Route version
       -> Vehicle + Driver assignment -> Dispatch release
       -> Milestones -> Delivery confirmation -> Close
```

Рекомендуемые нормализованные состояния CRM-проекции: `Draft`, `Planned`, `Assigned`, `Released`, `In Transit`, `At Stop`, `Delivered`, `Closed`, `Cancelled`, `Exception`. Точный список и mapping утверждаются по TMS; неизвестное внешнее состояние не преобразуется в успех.

## 8. Dispatching Architecture

| Элемент | Ответственность диспетчера | Контроль |
|---|---|---|
| Назначение | Проверить подтверждение TMS, а не свободно перезаписать ресурс | Assignment ID/version |
| Выпуск рейса | Проверить готовность пакета | Release checklist |
| Milestone | Следить за plan/fact и исключениями | Source timestamp/freshness |
| Задача | Выполнить конкретное действие с результатом | Owner, deadline, outcome |
| Эскалация | Передать blocker по severity/age | Tiered matrix и audit |
| Коммуникация | Дать клиенту подтверждённый статус | Минимум данных, допустимый канал |

```text
Event -> Normalize -> Compare with plan -> No deviation -> Observe
                                      |
                                      +-> Exception -> Task -> Escalate
                                                       -> Resolve -> RCA
```

Задачи Bitrix24 не подменяют диспетчерскую очередь TMS. Автоматизация не должна отправлять клиенту сырую GPS-телеметрию или неподтверждённый ETA.

## 9. Customer Service

| Тип | Назначение | Ключевые данные | Exit criteria |
|---|---|---|---|
| Вопрос | Информационный запрос | Client, request/trip, topic, owner | Подтверждённый ответ и next step |
| Обращение | Требуется операционное действие | Category, priority, SLA, owner, result | Действие выполнено или обоснованно закрыто |
| Претензия | Формальное несоответствие | Trip, severity, evidence ref, impact, decision | Решение, компенсация/отказ и RCA связаны |
| Инцидент доставки | Операционное отклонение | Milestone, event time, scope, containment | Сервис восстановлен и влияние оценено |

```text
Inbound -> Identify client/trip -> Classify -> Assign -> Investigate
 -> Resolve -> Communicate -> Confirm -> Close
                   |
                   +-> Claim / Finance / Legal escalation
```

SLA определяет старт, календарь, допустимые паузы, reaction/resolution targets, priority и escalation. Реализация SLA в конкретном Bitrix24 требует проверки доступных механизмов.

## 10. Documents Architecture

| Документ | Владелец | Master/хранилище | Контроль |
|---|---|---|---|
| Договор | Юрист/коммерческий owner | ECM/ERP/разрешённое хранилище | Версия, статус, полномочия |
| Заявка | Sales/логист | CRM/TMS по ADR | Revision и подтверждение |
| Путевой лист | Операционный/регулируемый контур | TMS/ERP/специализированная система | Реальные требования проверяются отдельно |
| Акт | Бухгалтерия/ответственный сдачи | ERP/ECM | Основание, версия, статус |
| Накладные | Логистика/склад | ERP/WMS/TMS/ECM | Shipment/trip relation |
| Подтверждение доставки | TMS/ECM | Разрешённый evidence reference | Source, timestamp, access |

Bitrix24 хранит metadata, lifecycle, связи и ссылку на актуальный документ. Юридические оригиналы, водительские данные, геоданные и регулируемые документы не копируются без классификации, основания и матрицы доступа.

## 11. Rights Model

| Роль | CRM | Перевозки | Документы | Аналитика |
|---|---|---|---|---|
| Менеджер | Свои/командные клиенты и сделки | Статус связанных перевозок | Коммерческие документы по scope | Личные/командные продажи |
| Диспетчер | Минимальный клиентский контекст | Назначенные/дежурные рейсы и исключения | Операционные ссылки | Plan/fact и exceptions |
| Водитель | Только необходимое задание | Назначенный рейс и допустимые действия | Только нужные инструкции | Собственный операционный минимум |
| Логист | Разрешённые заказы | Планирование, маршруты, ресурсы по scope | Заявки/маршрутные документы | Загрузка и сроки |
| Бухгалтер | Реквизиты по необходимости | Финансовый контекст рейса | Акты/счета/накладные | Оплата и задолженность |
| Директор | Управленческий scope | Агрегированный обзор и решения | Только необходимые документы | Управленческие KPI |

Точная реализация CRM-ролей, Smart Process rights, отделов, групп, наследования и экспорта требует проверки на целевом портале. Геолокация и персональные данные водителя доступны только тем ролям и системам, которым это действительно необходимо.

Контроли: least privilege, segregation of duties, minimal service-account scope, offboarding, access review, UI/direct URL/search/REST/export tests.

## 12. Automation Architecture

| Механизм | Сценарий | Защита | Обработка ошибки |
|---|---|---|---|
| Робот/задача | Первый ответ по заявке | Один marker на request | Fallback owner/alert |
| Робот | Запрос расчёта | Calculation revision | Timeout → логист/руководитель |
| Уведомление | Подтверждение заказа/статуса | Только подтверждённые данные | Delivery failure фиксируется |
| Робот | Контроль просрочки milestone | Plan/fact threshold | Tiered escalation |
| Задача | Разбор exception | Named owner, deadline, result | Reassign/incident route |
| Бизнес-процесс | Согласование цены/условий | Version ID и approver matrix | Reject/timeout path |
| Бизнес-процесс | Претензия/компенсация | Severity/authority | Legal/finance escalation |

```text
Trigger -> Validate -> Check source/version -> Execute -> Record result
              |                              |
              v                              v
         Reject safely                 Retry bounded
                                              |
                                              v
                                        Escalate / RCA
```

Каждая automation имеет owner, trigger, preconditions, used fields, idempotency key, timeout, retry budget, error route, audit trail и test case. Автоматическое перемещение по стадиям не строится на непроверенной GPS-точке.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| GPS/телематика | Position/event/ignition/arrival signals по минимальному scope | Provider API/event through controlled integration | Authentication, dedup, timestamp, privacy, lag |
| TMS | Request, calculation, route, trip, assignment, milestones | REST/OAuth/application/batch contract | External IDs, version, order, reconcile |
| FMS | Vehicle/driver availability and status | API/controlled exchange | Freshness, qualification mapping |
| 1С/ERP | Counterparty, order, documents, cost/payment facts | Verified connector/application or contract integration | Mapping, queue, retry, totals |
| Склад/WMS | Pickup/delivery/handling status | API/event/batch flow | Shipment ID, quantity, timezone |
| Телефония | Call event and CRM relation | Official Bitrix24 telephony/verified provider | Unmatched calls, rights, recording policy |
| BI/DWH | Allowed CRM/TMS/ERP datasets | BI Constructor/controlled export | Freshness, access, metric reconciliation |

```text
Source -> Authenticate -> Validate -> Normalize -> Idempotency -> Target
  |                                                            |
  +-> source event log                                target result
                \                                       /
                 +------ retry / reconciliation -------+
```

Контракт фиксирует source of truth, direction, keys, schema/version, units, timezone, geo precision/retention, event ordering, create/update/cancel semantics, timeouts, retry budget, rate limits, security, PII policy, monitoring, recovery и owner.

Наличие готовой интеграции с конкретной TMS, GPS, 1С, складом или BI не предполагается. Приложение проверяется в официальном Маркетплейсе, а внешний контракт — в документации поставщика отдельным scope.

## 14. Analytics

| Отчёт | Метрика | Grain | Источник сверки |
|---|---|---|---|
| Количество перевозок | Planned/released/delivered/closed | Trip/period | TMS |
| Прибыльность | Revenue, direct cost, margin | Order/trip | ERP/finance |
| Загрузка транспорта | Assigned/available utilization | Vehicle-period | TMS/FMS |
| Сроки | On-time pickup/delivery, cycle time | Trip/milestone | TMS/GPS normalized events |
| Эффективность сотрудников | SLA, throughput, exceptions resolved | Role/process-period | Approved metric sources |
| Продажи | Conversion, value, win/loss | Deal/cohort | CRM |
| Претензии | Claim rate, severity, resolution time | Claim/trip/cohort | CRM + TMS/ERP facts |
| Интеграции | Success/error/lag/reconcile delta | Flow-period | Integration monitoring |

```text
CRM + TMS/FMS + GPS + ERP/WMS
 -> Data contracts -> Quality checks -> Dataset/DWH
 -> Metric dictionary -> Dashboard -> Decision
                         ^                |
                         +-- reconcile ---+
```

BI Constructor, datasets и поля зависят от портала. Metric dictionary содержит formula, grain, cohort, filters, timezone, currency, distance/weight units, owner, freshness и reconciliation source.

## 15. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Новая заявка | Validation, dedup, routing | Один request и owner |
| 2 | Неполная заявка | Mandatory data gate | Расчёт не запущен, создано действие |
| 3 | Новая версия требований | Revision control | Прошлый расчёт не перезаписан |
| 4 | Расчёт перевозки | Version, amount, currency, validity | КП связано с актуальным расчётом |
| 5 | Подтверждение заказа | Release package | Один external request в TMS |
| 6 | Повтор события | Idempotency | Дубликат рейса не создан |
| 7 | Назначение транспорта | Availability/version | Конфликт не принят как успех |
| 8 | Назначение водителя | Qualification/access | Только допустимое назначение |
| 9 | Создание маршрута | Stops order/timezone/version | CRM-проекция совпадает с TMS |
| 10 | GPS arrival event | Mapping/timestamp/duplicate | Один нормализованный milestone |
| 11 | События не по порядку | Sequence/version | Статус не откатился |
| 12 | Просрочка доставки | Plan/fact threshold | Exception и эскалация созданы |
| 13 | Неизвестный TMS status | Safe mapping | Нет ложного «Доставлено» |
| 14 | Подтверждение доставки | Source/evidence/access | Fact записан корректно |
| 15 | Претензия | Trip relation/SLA/owner | Управляемый lifecycle запущен |
| 16 | Ошибка GPS/TMS | Timeout/retry/dead event | Нет потери/дубля, alert трассируем |
| 17 | Финансовая сверка | Order/trip/payment IDs | Статус совпадает с ERP |
| 18 | Проверка прав | UI/URL/REST/export | Роль видит только разрешённый scope |
| 19 | Миграция | Dry-run/sample/totals | Mapping подтверждён |
| 20 | Rollback | Trigger/runbook | Безопасное состояние восстановлено |

Тестовые данные синтетические. UAT подписывают Sales, Logistics, Dispatch, Fleet, Drivers representative, Finance, Service и Integration owners. Обязательны positive, negative, repeat, concurrency, rights и recovery cases.

## 16. Deployment

| Этап | Действия | Gate |
|---|---|---|
| Discovery | AS IS, портал, системы, объёмы, master data, риски геоданных | Inventory и scope подписаны |
| Архитектура | Data model, lifecycle, RACI, mappings, ADR | TO BE утверждён |
| Настройка | Test portal/funnel, fields, stages, rights | Peer review и smoke |
| Миграция | Clean, mapping, dry-run, sample, reconcile | Acceptance report |
| Интеграции | Contract, failure, lag, recovery tests | End-to-end успешен |
| Обучение | Role-based и сменные сценарии | UAT/readiness sign-off |
| Запуск | Freeze, backup/export, cutover, smoke | Go/no-go |
| Hypercare | Monitoring, incident route, daily reconcile | Нет critical defects, support принял систему |

```text
Discovery -> Test configuration -> Contract tests -> Sample migration
 -> UAT -> Go/No-Go -> Cutover -> Smoke -> Reconcile -> Hypercare
                              |
                              +-> Rollback by approved trigger
```

Cloud и On-Premise планируются отдельно. Для коробочной версии дополнительно проверяются версия продукта/модулей, PHP, БД, файлы, cron, backup/restore, инфраструктура и безопасность. Ядро продукта не изменяется.

## 17. Monitoring

| Контроль | Сигнал | Owner | Реакция |
|---|---|---|---|
| Необработанные заявки | Нет owner/первого действия | Sales lead | Назначить и провести RCA |
| Рейс без ресурса | Release приближается, assignment отсутствует | Logistics/dispatch | Эскалация и план |
| Просрочка milestone | Plan breached | Dispatcher | Verify source, exception workflow |
| GPS/TMS stale | Freshness threshold | Integration owner | Mark stale, provider incident |
| Ошибка интеграции | Error/retry/dead event | Integration owner | Recover/reconcile |
| Претензия без ответа | SLA threshold | Service lead | Reassign/escalate |
| Документы не готовы | Close gate breached | Document owner | Task/escalation |
| Финансовое расхождение | CRM/TMS ≠ ERP | Finance/data owner | Reconciliation, source correction |
| Аномальный доступ к геоданным | Security signal | Security owner | Contain/audit/revoke |

SLI: success/error rate, latency, backlog age, retry count, last successful sync, stale age и reconciliation delta. Логи не содержат секреты, реальные координаты, адреса или лишние персональные данные.

## 18. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Bitrix24 используется как TMS | Неверная граница | Неполное планирование | System context и TMS master |
| 2 | Дубли заявок | Повтор события | Двойной расчёт/рейс | Idempotency key |
| 3 | Дубли клиентов | Нет identity policy | Разорванная история | Normalization/matching |
| 4 | Сделка равна рейсу | Сущности смешаны | Неверная аналитика | Separate lifecycle |
| 5 | Маршрут без версии | Правки перезаписывают план | Исполнение по старым данным | Route revision |
| 6 | Точки маршрута в свободном тексте | Нет структуры | Ошибка планирования | Structured stops in TMS |
| 7 | Неверная единица веса/объёма | Mapping не определён | Неверный транспорт/цена | Unit dictionary |
| 8 | Срок без timezone | Локальные даты | Ложная просрочка | Explicit timezone |
| 9 | Расчёт не связан с revision | Несколько версий | Убыточное КП | Calculation ID/version |
| 10 | КП истекло | Нет validity | Неверные условия | Validity control |
| 11 | Рейс запущен без подтверждения | Нет gate | Финансовый риск | Release approval |
| 12 | Двойное назначение транспорта | Нет concurrency control | Срыв рейса | TMS assignment master |
| 13 | Двойное назначение водителя | Нет availability check | Срыв/нарушение | TMS/FMS validation |
| 14 | Неверная квалификация водителя | Данные устарели | Риск безопасности | Qualification master/freshness |
| 15 | Транспортный статус устарел | Sync lag | Ошибочное решение | Freshness marker |
| 16 | GPS недоступен | Provider outage | Нет наблюдаемости | Safe degradation/TMS fallback |
| 17 | GPS-точка ошибочно означает доставку | Неверная семантика | Ложное закрытие | Confirmed milestone rule |
| 18 | События приходят не по порядку | Async поток | Откат статуса | Sequence/version control |
| 19 | Неизвестный статус считается успехом | Оптимистичный mapping | Неверное закрытие | Exception state |
| 20 | Просрочка не эскалируется | Нет threshold | Клиент узнаёт первым | SLA/alert |
| 21 | ETA сообщается без подтверждения | Сырая телеметрия | Ложное обещание | Validated ETA source |
| 22 | Геоданные доступны широко | Широкие роли | Privacy/security incident | Need-to-know |
| 23 | ПДн водителя дублируются в CRM | Нет minimization | Утечка | ID/flags only |
| 24 | Путевой документ хранится без ограничений | Удобство | Compliance risk | Secure master/reference |
| 25 | Секрет в webhook URL опубликован | Неверная классификация | Компрометация | Secret store/rotation |
| 26 | Payload с координатами в логах | Debug целиком | Утечка | Masking/minimal logs |
| 27 | Повтор retry создаёт рейс | Нет idempotency | Двойное исполнение | Event/external ID |
| 28 | Retry бесконечный | Нет budget | Нагрузка/backlog | Bounded backoff/dead letter |
| 29 | REST limits/timeout | Пиковая нагрузка | Lag | Queue/backoff/verified batch |
| 30 | Свободная двусторонняя правка | Нет field owner | Расхождение систем | Master matrix |
| 31 | Документ неверной версии | Файлы в почте | Юридический спор | Current version metadata |
| 32 | Доставка закрыта без документа | Нет close gate | Не готово закрытие | Completeness rule |
| 33 | Оплата отмечена вручную | CRM подменяет ERP | Ложная задолженность | ERP confirmation |
| 34 | Себестоимость вводится приблизительно | Нет finance source | Неверная маржа | ERP reconciliation |
| 35 | Претензия без рейса | Нет relation | Слабый RCA | Required trip link |
| 36 | SLA претензии не учитывает паузу | Нет policy | Ложный KPI | Calendar/pause definition |
| 37 | Компенсацию утверждает исполнитель | Нет segregation | Финансовый риск | Approval matrix |
| 38 | Массовый импорт без dry-run | Спешка | Порча данных | Sample/report/rollback |
| 39 | Нет reconciliation | HTTP success принят за итог | Тихие пропуски | Totals/sample/delta |
| 40 | Тариф не поддерживает модель | Capability check пропущен | Нереализуемый scope | Portal verification |
| 41 | Marketplace app изменилось | Vendor dependency | Сбой интеграции | Assessment/exit plan |
| 42 | On-Premise устарел | Старые модули/PHP | Ошибки/уязвимости | Supported update plan |
| 43 | Backup не восстанавливается | Нет restore drill | Rollback невозможен | Test restore |
| 44 | Права тестирует администратор | Нет role QA | Production access defects | Role-based UAT |
| 45 | Service account — администратор | Простая настройка | Большой blast radius | Least scope |
| 46 | Уволенный owner/водитель остаётся активным | Нет offboarding | Доступ/ошибочное назначение | Revoke/reassign sync |
| 47 | Загрузка считается по числу задач | Неверный grain | Ошибочное планирование | TMS/FMS source |
| 48 | Прибыльность без фактических затрат | Нет ERP data | Ложная маржа | Finance source |
| 49 | KPI без metric dictionary | Разные формулы | Конфликт отделов | Governed definitions |
| 50 | Dashboard без freshness | Старые статусы | Ошибочное решение | Last refresh/SLA |
| 51 | Нет owner интеграции | Проект завершён | Ошибки не разбираются | Support RACI |
| 52 | Нет safe degradation | TMS/GPS outage | Порча статусов | Stale/unknown state |
| 53 | Нет concurrency test | Только последовательный QA | Double assignment | Parallel tests |
| 54 | Нет обучения смен | Обучили офис | Обход процесса | Role/shift training |
| 55 | Нет hypercare | Система передана сразу | Инциденты накапливаются | Support window |

## 19. ADR

### ADR-LOG-001: Модель перевозки

- **Контекст:** коммерческая сделка и операционный рейс имеют разные статусы и владельцев.
- **Решение:** сделка отражает коммерческое обязательство, Smart Process «Рейс» — CRM-проекцию TMS execution.
- **Альтернативы:** одна сделка на всё; только TMS без CRM-проекции.
- **Последствия:** нужны external ID, state mapping и reconciliation.
- **Статус:** Proposed.

### ADR-LOG-002: Модель маршрута

- **Контекст:** маршрут имеет упорядоченные точки, ограничения и версии.
- **Решение:** TMS — master; Bitrix24 хранит summary и relation к версии маршрута.
- **Альтернативы:** адреса в полях сделки; Smart Process как единственный planner.
- **Последствия:** version ID, timezone и минимизация геоданных.
- **Статус:** Proposed.

### ADR-LOG-003: Транспорт и водитель

- **Контекст:** ресурсы имеют доступность, квалификацию и чувствительные данные.
- **Решение:** TMS/FMS/HR — master; Bitrix24 получает минимальные IDs/status flags.
- **Альтернативы:** полный реестр в CRM.
- **Последствия:** freshness, privacy, least privilege.
- **Статус:** Proposed.

### ADR-LOG-004: Модель прав

- **Контекст:** продажи, диспетчер, водитель и бухгалтерия требуют разный scope.
- **Решение:** role-based least privilege с проверкой UI/URL/REST/export.
- **Альтернативы:** общая роль; администратор для операционной работы.
- **Последствия:** test accounts, access review и offboarding.
- **Статус:** Proposed.

### ADR-LOG-005: Документы

- **Контекст:** договорные, перевозочные и финансовые документы различаются по owner и требованиям.
- **Решение:** Bitrix24 хранит metadata/lifecycle/reference; оригинал — в назначенной защищённой системе.
- **Альтернативы:** все файлы в сделке; только внешняя система без CRM-связи.
- **Последствия:** classification, current version, retention.
- **Статус:** Proposed.

### ADR-LOG-006: Интеграционная архитектура

- **Контекст:** TMS, GPS, ERP и WMS передают асинхронные события.
- **Решение:** versioned contracts, idempotency, event ordering, retry budget и reconciliation.
- **Альтернативы:** свободная двусторонняя синхронизация; ручной CSV.
- **Последствия:** integration monitoring и support ownership.
- **Статус:** Proposed.

### ADR-LOG-007: Аналитическая архитектура

- **Контекст:** CRM не содержит все факты маршрута, затрат и загрузки.
- **Решение:** datasets строятся по data contracts, производственные показатели сверяются с TMS/ERP.
- **Альтернативы:** только CRM-отчёты; ручные таблицы.
- **Последствия:** metric dictionary, grain, freshness и reconciliation.
- **Статус:** Proposed.

### ADR-LOG-008: Лиды или Request-First

- **Контекст:** заявка постоянного клиента может быть уже квалифицированным заказом.
- **Решение:** лид применяется только для неквалифицированного intake; подтверждённый запрос создаётся в request/deal flow.
- **Альтернативы:** лид на каждую перевозку; Deal-First для всего.
- **Последствия:** выбор фиксируется до настройки каналов и аналитики.
- **Статус:** Open до Discovery.

## 20. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Bitrix24 объявлен TMS | Неверный scope | Нет полноценного планирования | TMS master/system context |
| 2 | Сделка и рейс — одна сущность | Желание одной карточки | Смешаны продажи и исполнение | Separate lifecycle |
| 3 | Задача заменяет рейс | Work item подменяет domain object | Нет статуса/аналитики | Trip entity + tasks |
| 4 | Лид создаётся на каждый GPS event | Автоматизация без модели | Мусор CRM | Business-event boundary |
| 5 | Новый заказ создаёт новую компанию | Нет dedup | Дубли клиентов | Company matching |
| 6 | Название компании — ключ | Нет stable ID | Ошибочный merge | External/internal ID |
| 7 | Один клиент — одна сделка навсегда | Клиент и заказ смешаны | Нельзя считать перевозки | Deal/order rule |
| 8 | Заявка перезаписывается при изменении | Нет versioning | Расчёт по старым требованиям | Request revision |
| 9 | Точки маршрута хранятся строкой | Быстрая настройка | Нельзя валидировать порядок | TMS structured stops |
| 10 | Адрес — технический ключ | Адрес меняется/неуникален | Неверные связи | Stop/location ID |
| 11 | Маршрут без версии | Нет change control | Водитель получает старый план | Route revision |
| 12 | Плановая дата считается фактом | Semantics не определена | Ложное SLA | Plan/fact separation |
| 13 | Время без timezone | Локальные даты | Ошибка сроков | Explicit timezone |
| 14 | Вес без единицы | Неполная модель | Неверный расчёт | Quantity + unit |
| 15 | Объём и вес смешаны | Одно поле | Неверный ресурс | Separate measures |
| 16 | Расчёт в комментарии | Нет structured result | Нельзя сверить КП | Calculation entity/reference |
| 17 | КП не связано с revision | Версии отдельно | Неверная цена | Version relation |
| 18 | КП без validity | Нет срока | Убыточная перевозка | Validity date |
| 19 | Валюта не указана | Быстрая настройка | Неверная аналитика | Amount + currency |
| 20 | Срок обещан до TMS-проверки | KPI продаж | Невыполнимое обязательство | Feasibility/capacity gate |
| 21 | Рейс запускается по стадии сделки | Нет release package | Неполные данные | Formal handoff |
| 22 | Повтор release создаёт рейс | Нет idempotency | Двойное исполнение | External/event ID |
| 23 | Номер рейса вводят в двух системах | Нет master | Коллизии | Master-generated ID |
| 24 | Транспорт назначают в CRM и TMS | Свободная двусторонняя правка | Конфликт ресурсов | TMS ownership |
| 25 | Водитель назначается без проверки | Нет qualification/availability | Срыв/риск | TMS/FMS validation |
| 26 | Реальный номер транспорта в тестах | Не обезличили данные | Утечка | Synthetic values |
| 27 | Водитель видит всю клиентскую базу | Широкая роль | Privacy breach | Minimal trip view |
| 28 | Диспетчер видит финансовые документы | Скопирована роль | Избыточный доступ | Need-to-know |
| 29 | GPS-координаты хранятся бессрочно | Нет retention | Privacy/security risk | Minimal retention/reference |
| 30 | GPS payload пишется в лог | Debug целиком | Утечка | Masking |
| 31 | GPS-точка автоматически закрывает доставку | Неверная семантика | Ложный факт | Confirmed delivery source |
| 32 | Неизвестный статус = delivered | Оптимистичный fallback | Ошибочное закрытие | Exception state |
| 33 | События применяются не по порядку | Async | Откат статуса | Sequence/version |
| 34 | Дубликат события создаёт milestone | Нет event key | Искажённая история | Idempotency |
| 35 | ETA отправляется без проверки | Сырая телеметрия | Ложное обещание | Validated ETA rule |
| 36 | Просрочка не создаёт exception | Нет monitor | Клиент жалуется первым | Threshold/alert |
| 37 | Эскалация уведомляет всех | Нет matrix | Alert fatigue | Tiered route |
| 38 | Задача создаётся повторно | Нет marker | Дубли работы | Idempotent automation |
| 39 | Роботы циклически меняют статус | Взаимные triggers | Нагрузка/хаос | Loop guard |
| 40 | Ошибка automation скрыта | Нет error route | Рейс завис | Alert/escalation |
| 41 | Обращение закрыто отправкой сообщения | Activity = result | Проблема не решена | Exit criteria |
| 42 | Претензия хранится комментарием | Нет entity | Нет SLA/RCA | Claim lifecycle |
| 43 | Претензия не связана с рейсом | Нет relation | Слабое расследование | Required trip link |
| 44 | Severity и Priority смешаны | Нет методики | Неверная реакция | Separate definitions |
| 45 | Пауза SLA бесконтрольна | Нет reason/owner | Искусственно хороший KPI | Allowed pauses/audit |
| 46 | Компенсацию одобряет исполнитель | Нет segregation | Финансовый риск | Approval matrix |
| 47 | RCA заменено симптомом | Спешка | Повтор проблемы | Structured RCA |
| 48 | Документ хранится без версии | Файлы в чате | Неверный оригинал | Current version metadata |
| 49 | Путевой лист копируется в CRM без оценки | Удобство | Compliance/privacy risk | Secure master/reference |
| 50 | Доставка закрыта без документов | Нет gate | Финансовое закрытие задержано | Completeness check |
| 51 | Оплата отмечена менеджером | Нет ERP sync | Неверная задолженность | Finance master |
| 52 | Себестоимость — ручное поле | Нет source | Ложная прибыльность | ERP cost result |
| 53 | Callback/webhook URL в README | Секрет считают ссылкой | Компрометация | Secret store/env |
| 54 | Service account — администратор | Быстрая настройка | Большой blast radius | Least scope |
| 55 | Нет timeout | Happy path | Зависшие workers | Explicit timeout |
| 56 | Retry бесконечный | Нет budget | Нагрузка/backlog | Bounded backoff |
| 57 | Retry создаёт дубль | Нет idempotency | Двойной рейс | Event key |
| 58 | Ошибка просто пропускается | Нет dead-letter | Потеря события | Failed-event register |
| 59 | Batch без проверки метода | Желание ускорить | Частичный сбой | Verify limits/semantics |
| 60 | Статусы CRM и TMS редактируются свободно | Нет field owner | Расхождение | Master matrix |
| 61 | Полный импорт без dry-run | Спешка | Массовая порча | Sample first |
| 62 | Название транспорта — ключ импорта | Нет ID | Неверный merge | Vehicle external ID |
| 63 | Нет source-target журнала | Миграция непрозрачна | Нельзя откатить | Mapping/report |
| 64 | Нет сверки totals | HTTP success принят за итог | Тихие пропуски | Reconciliation |
| 65 | Права тестирует администратор | Нет role QA | Production defects | Role accounts |
| 66 | Прямые ссылки/REST/export не проверены | Проверили UI | Обход прав | Multi-channel test |
| 67 | Уволенный сотрудник остаётся owner | Нет offboarding | Бесхозные объекты | Reassign/revoke |
| 68 | Уволенный водитель доступен для назначения | HR/TMS lag | Ошибочный рейс | Offboarding sync |
| 69 | Загрузка считается по задачам CRM | Неверный grain | Ошибочное планирование | TMS/FMS source |
| 70 | Количество перевозок считают по status events | Event grain | Завышенный KPI | Trip grain |
| 71 | Прибыльность без затрат ERP | Нет finance contract | Ложная маржа | Reconciliation |
| 72 | On-time без timezone/calendar | Нет metric definition | Спорный KPI | Metric dictionary |
| 73 | GPS availability принимают за качество сервиса | Подмена результата активностью | Неверный KPI | Outcome metrics |
| 74 | Dashboard без freshness | Старые данные | Ошибочное решение | Last refresh/SLA |
| 75 | Нет owner интеграции | Проект завершён | Ошибки не разбираются | Support RACI |
| 76 | Marketplace app не оценено | Доверие карточке | Vendor/security risk | Assessment/exit plan |
| 77 | Backup не тестировали | Формальная проверка | Rollback не работает | Restore drill |
| 78 | Запуск без go/no-go | Нет owner решения | Неконтролируемый cutover | Release criteria |
| 79 | Обучили только офисную смену | Неполный rollout | Ночная смена обходит процесс | Shift training |
| 80 | Нет hypercare | Проект закрыт сразу | Инциденты без owner | Support window |

## 21. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Подтверждены AS IS, объёмы и границы систем | Inventory, journeys, integrations, privacy assessment | Masters и owners определены |
| 2. Архитектура | Согласован TO BE | Data model, state mappings, RACI, ADR, risks | Владельцы подписали решения |
| 3. CRM настройка | Работает клиентский/коммерческий контур | Fields, stages, roles, dedup, tests | Smoke/rights tests успешны |
| 4. Логистические процессы | Работает request/route/trip/service orchestration | Smart Process registry, handoff и exception rules | Сквозной synthetic trip пройден |
| 5. Интеграции | Работают контролируемые TMS/GPS/ERP flows | Contracts, retries, monitoring, reconciliation | Failure/recovery tests успешны |
| 6. Тестирование | Подтверждена бизнес-готовность | Test report, defects, UAT sign-off | Нет critical/high defects |
| 7. Запуск | Выполнен управляемый cutover | Runbook, backup, rollback, communication | Smoke и reconcile успешны |
| 8. Оптимизация | Стабилизированы SLA и KPI | Baseline, backlog, lessons learned | Support model принят owners |

### Официальные источники

Используются только источники из `knowledge/SOURCES.md`; дата последней проверки реестра — 2026-08-05.

| Источник | Применение |
|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | Пользовательские возможности CRM и Cloud |
| [Документация REST API](https://apidocs.bitrix24.ru/) | Интеграционная архитектура и ограничения |
| [REST API reference](https://apidocs.bitrix24.ru/api-reference/) | Проверка контрактов методов и полей |
| [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/index.html) | Проверка вариантов интеграционного доступа |
| [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) | Проектирование request, route, trip, vehicle, driver и claim views |
| [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) | Проверка automation |
| [Как выбрать тип бизнес-процесса](https://helpdesk.bitrix24.ru/open/23951294/) | Выбор approval/orchestration механизма |
| [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) | Проверка вариантов телефонии |
| [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) | Проверка datasets и аналитики |
| [Маркетплейс Bitrix24](https://www.bitrix24.ru/apps/) | Проверка приложений и коннекторов |
| [Документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php) | On-Premise возможности и ограничения |

Если функция, поле, связь, событие, REST-метод, приложение или ограничение не подтверждены официальным источником и целевым порталом, они помечаются **«требует проверки»** и не включаются в обязательства перед заказчиком.
