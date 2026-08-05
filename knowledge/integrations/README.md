# Bitrix24 Integrations Engineering Knowledge Base

Инженерная база знаний для проектирования, внедрения, миграции, аудита и сопровождения интеграций Bitrix24. Контракт конкретного метода, события, приложения и внешнего сервиса всегда проверяется по официальной документации и на целевом портале.

> **Статус:** факты обобщены только по источникам из `knowledge/SOURCES.md`. «Рекомендация архитектора» — проектное решение. Возможность, зависящая от тарифа, редакции, версии, прав, приложения или провайдера, помечается как **требующая проверки**.

> **Безопасность:** примеры только синтетические. Реальные домены, endpoint, токены, webhook URL, персональные данные и клиентские выгрузки в Git запрещены.

## 1. Архитектура интеграций Bitrix24

Интеграционный слой связывает Bitrix24 с сайтами, 1С/ERP, телефонией, каналами коммуникации, платёжными и документными системами. Он отвечает за преобразование данных, надёжность, безопасность, наблюдаемость и разрешение конфликтов.

```text
Источник → API/Connector → Интеграционный слой → REST/App → Bitrix24
Источник ← callback/polling ← Queue/Mapping/Rules ← event/webhook ← Bitrix24
```

| Компонент | Назначение | Контроль архитектора |
|---|---|---|
| REST API | Опубликованный контракт чтения/изменения | Версия, scope, права, лимиты и ошибки |
| Webhooks/events | Вызов REST или событийный сигнал | Секреты, повторы, очередь, идемпотентность |
| OAuth/app | Управляемый доступ приложения | Token lifecycle, tenant isolation, отзыв |
| Middleware/ESB | Mapping, маршрутизация и правила | SLA, HA, ownership и мониторинг |
| Queue | Асинхронность и backpressure | Порядок, retry, DLQ, recovery |
| ETL/batch | Массовый или периодический обмен | Dry-run, pagination, checkpoint, сверка |
| Polling | Выборка изменений по расписанию | Cursor, перекрытие окна, дедупликация |

Cloud ограничен опубликованными настройками, приложениями и REST в пределах тарифа и прав. Только On-Premise даёт контроль серверной инфраструктуры, PHP, БД, cron и модулей; ядро изменять нельзя. Конкретный коннектор всегда зависит от приложения и внешней системы.

## 2. Сбор требований

До выбора технологии архитектор фиксирует бизнес-цель, системы и владельцев, мастер-системы, объекты и направление обмена, частоту, объём, задержку, SLA, конфликты, безопасность, журналирование, поддержку и откат.

| Блок интервью | Вопросы |
|---|---|
| Цель | Какой измеримый результат и что произойдёт без интеграции |
| Системы | Какие версии, владельцы, среды и ограничения |
| Данные | Объекты, поля, объём, качество, персональные данные |
| Поток | Кто создаёт/изменяет, частота, допустимое отставание |
| Надёжность | Повторы, частичный результат, RPO/RTO, SLA |
| Эксплуатация | Кто мониторит, исправляет, ротирует секреты и принимает изменения |

```text
Business goal → Data ownership → Contract → Non-functional requirements
              → Architecture decision → Test/rollback/support plan
```

Чек-лист: [ ] владельцы назначены; [ ] мастер определён; [ ] mapping согласован; [ ] объём измерен; [ ] права минимальны; [ ] ошибки классифицированы; [ ] мониторинг и откат имеют владельца.

## 3. Мастер-система

Master data — авторитетное значение конкретного типа данных. Разные типы могут иметь разных мастеров; свободное двустороннее редактирование без ownership создаёт циклы и перезапись верных данных.

| Тип данных | Типичный мастер | Допустимое изменение | Правило конфликта |
|---|---|---|---|
| Контакты | CRM или профильная система | Только назначенные поля | Field ownership + timestamp/version |
| Компании | CRM/ERP | CRM — коммуникации, ERP — учётные атрибуты | Разрешение по полю, не whole-record |
| Реквизиты | 1С/ERP | В CRM чтение/запрос исправления | ERP побеждает после валидации |
| Товары | 1С/ERP/PIM | CRM использует | Внешний ID, мастер перезаписывает |
| Цены | ERP | Только согласованный прайс | Версия прайса и валюта |
| Заказы | Система оформления/ERP | Статусы по ownership | State machine и version |
| Оплаты | Банк/платёжная/ERP | CRM только отображает | Подтверждение мастера обязательно |
| Документы | ECM/ERP или CRM по типу | Версионное изменение | Зафиксированная версия |
| Статусы | Владелец жизненного цикла | Только его переходы | Недопустимый переход → manual |
| Остатки | ERP/WMS | Read-only в CRM | Последняя подтверждённая версия |

CRM может быть мастером продаж и коммуникаций; 1С — товаров, цен, заказов, оплат и реквизитов; сайт — исходной web-сессии/корзины; специализированный сервис — своего предметного статуса.

## 4. Архитектурные паттерны

| Паттерн | Когда применять / не применять | Плюсы | Риски, эксплуатация и масштаб |
|---|---|---|---|
| Point-to-point | 2 системы, простой контракт / не при росте связей | Быстро | Сильная связанность; низкая масштабируемость |
| Middleware | Несколько потоков и rules | Централизованный mapping | Требует HA, owner и monitoring |
| ESB | Большая корпоративная экосистема / не для одной связи | Governance и маршрутизация | Высокая сложность и стоимость |
| API gateway | Единая граница API | Auth, policy, routing | Не заменяет очередь и бизнес-mapping |
| Webhook-first | Есть официальные события | Низкая задержка | Повторы; нужна reconciliation |
| Event-driven | Независимые подписчики | Масштабирование | Порядок, schema evolution, eventual consistency |
| Polling | Нет события или нужна сверка | Предсказуемость | Задержка и нагрузка |
| Batch | Много коротких REST-команд | Меньше round trips | Не транзакция; partial errors |
| ETL | Миграция/аналитика | Контролируемый pipeline | Устаревание и большие объёмы |
| Двусторонний | Обе системы владеют разными полями | Доступность данных | Конфликты и циклы |
| Односторонний | Явный master→consumer | Простота | Consumer не исправляет master |
| Hybrid | Events + polling/reconcile | Скорость и полнота | Больше компонентов |

## 5. Интеграция с сайтами

| Сценарий | Объекты | Контроль |
|---|---|---|
| Лендинг | Форма → лид/сделка | Источник, UTM, согласие, spam, дубли |
| Корпоративный сайт | Заявки, формы, контент запроса | Маршрутизация и SLA |
| Интернет-магазин | Клиент, корзина, товары, заказ, оплата, статусы | Master order, цены, валюта, idempotency |
| Личный кабинет | Профиль, обращения, документы, статусы | Авторизация и разграничение данных |
| Custom web app | Published API contract | OAuth/webhook, queue, monitoring |

```text
Browser → Website backend → validate/consent/deduplicate → Bitrix24
Browser ← Website backend ← safe status subset ← master system
```

Cookies и идентификаторы применяются только при правовом основании и подтверждённом сценарии. Ошибки: REST из браузера с секретом, потеря UTM, дубль каждой отправки, доверие клиентской цене, отсутствие статуса ошибки и обратной сверки.

## 6. Интеграция с 1С

Нельзя считать интеграции с 1С одинаковыми: контракт зависит от конфигурации 1С, версии, приложения, модулей и доработок и **требует проверки**.

| Объект | Типичный мастер | Направление | Ключ | Риск |
|---|---|---|---|---|
| Клиент | По процессу CRM/1С | Одно- или двустороннее по полям | GUID + реквизиты | Дубли и конфликты |
| Товар | 1С | 1С → CRM | GUID/SKU | Дубли вариантов |
| Цена | 1С | 1С → CRM | Product GUID + price type | Валюта/НДС |
| Заказ | По месту оформления | В мастер исполнения | External ID | Повторное создание |
| Счёт | 1С/ERP | 1С → CRM | Document GUID | Разные модели счёта |
| Оплата | 1С/банк | 1С → CRM | Payment ID | Ложный paid status |
| Остаток | 1С/WMS | 1С → CRM | Product+warehouse | Устаревание |
| Статус | Владелец этапа | Master → consumer | Object ID+version | Обратные циклы |

Mapping учитывает реквизиты, единицы измерения, валюты, налоги, справочники и version. Официальный механизм и приложение проверяются на конкретном проекте.

## 7. Телефония

Интеграционный аспект включает встроенный сценарий, SIP-подключение и REST-приложения/коннекторы в подтверждённой конфигурации. Официальная основа: [варианты подключения телефонии](https://helpdesk.bitrix24.ru/open/6450911/).

| Поток | Контроль |
|---|---|
| Входящий звонок | Нормализация номера, поиск клиента, правило создания лида |
| Маршрутизация | Очередь, ответственные, рабочее время, fallback |
| Карточка звонка | Связь с корректной CRM-сущностью |
| Запись | Права, согласие/правовое основание, срок хранения |
| Пропущенный | Задача/SLA и дедупликация повторных событий |

```text
Operator/PBX → supported connector/SIP/REST app → Bitrix24 telephony → CRM
```

Не смешивать один номер через разные механизмы без официальной проверки.

## 8. Мессенджеры и открытые линии

| Аспект | Архитектурное правило |
|---|---|
| Канал | Поддержка WhatsApp, Telegram или иного канала зависит от официального коннектора/приложения |
| Распределение | Очередь, рабочее время, SLA и transfer |
| CRM | Формальное правило создания/поиска лида, контакта и сделки |
| Идентификация | Provider ID + подтверждённые контакты; display name не уникален |
| История | Срок хранения, права и персональные данные |
| Дубли | Повторное сообщение/переподключение не создаёт новый клиентский объект |

```text
Channel provider → Connector → Open Line → Operator → CRM context
```

Ограничения внешнего провайдера, шаблоны сообщений и регионы проверяются отдельно.

## 9. Почта и календарь

| Контур | Риски и контроль |
|---|---|
| Личный ящик | Владелец, доступ, увольнение, привязка писем |
| Общий ящик | Очередь, ответственный, дубли обработки |
| Email → CRM | Поиск клиента, правило создания лида, threading |
| Календарь | Направление синхронизации, timezone, повторные встречи |
| Приглашения | Участники, отмена/обновление, права и персональные данные |

```text
Mail/Calendar provider ↔ supported connector ↔ Bitrix24 ↔ CRM activity
```

Конкретная доступность синхронизации и провайдера зависит от продукта и приложения.

## 10. Платёжные системы

| Этап | Контроль |
|---|---|
| Создание | Amount, currency, order/payment ID из master |
| Ссылка | Генерируется только подтверждённым провайдером; не логировать чувствительные параметры |
| Callback | HTTPS, официальный механизм проверки, queue, deduplication |
| Статус | Не доверять redirect браузера; подтверждать master/provider |
| Возврат | Отдельный operation ID, права и reconciliation |
| Ошибка | Retry только временного; unknown → проверить статус |

```text
CRM/order → Payment provider → customer
CRM/order ← verified callback/reconcile ← provider
```

Конкретные провайдеры не фиксируются без официального источника.

## 11. Документы и ЭДО

| Область | Штатное / внешнее | Контроль |
|---|---|---|
| Генерация | Штатный инструмент при доступности | Шаблон, version, fields |
| Подпись | Штатная функция или приложение | Юридическая применимость и стороны |
| ЭДО | Внешняя система/приложение | Document ID, statuses, signatures |
| Хранение | Bitrix24 или ECM по master policy | Права, версии, архив и retention |
| CRM-связь | Deal/company/contact/process | Не хранить только имя файла |

```text
CRM data → document template → versioned document → sign/EDO → status back
```

Штатное, marketplace-приложение и custom integration документируются раздельно.

## 12. Интеграция с внешними API

| Компонент | Требование |
|---|---|
| Вход/выход | Published REST/webhook contract |
| Auth | OAuth/API key/token только по контракту, секрет вне Git |
| Middleware | Validation, mapping, routing, queue |
| Schema | Version, required fields, enum mapping, defaults |
| Reliability | Timeout, classified retry, idempotency, DLQ |
| Operations | Metrics, alerts, trace, revoke runbook |

```text
Bitrix24 ↔ Adapter ↔ Canonical model ↔ Adapter ↔ External API
```

API key внешнего сервиса не помещается в робот, поле CRM или клиентский браузер.

## 13. Карта данных

| Source field | Target field | Type/required | Transform/dictionary | Match/default/error | Owner |
|---|---|---|---|---|---|
| `source.customer_id` | `<CRM_EXTERNAL_ID>` | string / yes | Trim, validate | Unique / none / reject | Integration |
| `source.email` | Standard email | email / no | Lowercase, validate | Contact key candidate / empty / quarantine | CRM owner |
| `source.status` | `<CRM_STATUS_FIELD>` | enum / yes | Versioned mapping | No default / manual unknown | Process owner |

```text
Source schema → Mapping/version → Validation → Target schema
                         └──────→ quarantine/report
```

Для каждого поля фиксируются источник, target, тип, обязательность, transform, справочник, key, default, error policy и data owner.

## 14. Ключи сопоставления и дубли

| Ключ | Надёжность | Применение |
|---|---|---|
| External ID/GUID | Высокая в пределах источника | Основной upsert key |
| Source ID | Высокая с namespace | `source_system + source_id` |
| Email/телефон | Средняя | Нормализованный кандидат, не всегда уникален |
| ИНН/регистрационный ID | Высокая при валидации и юрисдикции | Компания/реквизиты |
| Составной ключ | По качеству компонентов | Когда нет одного устойчивого ID |
| Название компании | Низкая | Только поиск кандидатов, не unique key |

```text
Normalize → exact external key → candidate search → conflict policy
          → one match: update | none: create | many: manual review
```

## 15. Идемпотентность

| Ситуация | Решение |
|---|---|
| Повторный request/event | Тот же idempotency/operation key |
| Create | Upsert по external ID |
| Timeout | Read-after-timeout/reconciliation |
| Partial | Per-step state, compensation/manual review |
| Webhook replay | Dedup store + актуальное REST-чтение |
| Rollback | Компенсационное действие, не иллюзия общей транзакции |

```text
operation_id → planned → sent → succeeded / failed / unknown
                                      unknown → reconcile → final state
```

## 16. Очереди и асинхронная обработка

| Элемент | Назначение |
|---|---|
| Producer | Валидирует и публикует сообщение |
| Queue | Надёжно хранит и применяет backpressure |
| Consumer | Идемпотентно обрабатывает |
| Retry queue | Отложенный повтор временных ошибок |
| DLQ | Permanent/исчерпанные ошибки для разбора |
| Partition/order | Порядок только там, где он нужен |
| Monitoring | Depth, age, throughput, failures |

```text
Producer → Main queue → Consumer → success
                         ├→ Retry queue ─┘
                         └→ DLQ → manual/recovery
```

Технология выбирается после SLA; архитектура от бренда брокера не зависит.

## 17. Производительность

| Область | Правило |
|---|---|
| Объём/частота | Измерить average, peak и growth |
| REST limits | Проверять актуальную [документацию](https://apidocs.bitrix24.ru/), числа не придумывать |
| Batch | Только после проверки; per-command errors |
| Pagination | Полный `next`/cursor, stable order, checkpoint |
| Parallelism | Лимит на tenant и operation type |
| Cache | TTL для безопасных metadata, не вечные права |
| Incremental | Modified timestamp + ID tie-breaker + overlap |
| Night load | Не гарантия безопасности; учитывать роботов и пользователей |
| Load test | Test portal, synthetic volume, recovery |
| Degradation | Circuit breaker, queue, backoff, reconcile |

## 18. Безопасность

| Контроль | Требование |
|---|---|
| Права | Least privilege и отдельный integration identity |
| Secrets | Secret storage; `.env` локально и вне Git |
| TLS | Для всех внешних потоков |
| Personal data | Минимизация, маскирование, retention и legal basis |
| Logs | Без токенов и полных payload |
| Rotation/revoke | Проверенный runbook |
| IP control | Только если применимо и подтверждено; не единственная защита |
| Contractors | Временный минимальный доступ и аудит |
| Environments | Раздельные credentials/data/queues |
| Tests | Только синтетические данные |

## 19. Обработка ошибок

Точное число retry задаётся политикой проекта и официальными рекомендациями, не универсальным числом.

| Ошибка | Retry | Manual | Лог/уведомление |
|---|---|---|---|
| Network/timeout | Ограниченно; write после reconcile | Unknown result | operation ID, stage, latency |
| Auth | После refresh/rotation | Если доступ отозван | Без token |
| Permission | Нет | Согласовать права | tenant, operation, role |
| Validation/transform | Нет | Исправить mapping/data | Field code, category |
| Not found | Обычно нет | Если объект ожидался | Type, safe ID |
| Conflict/duplicate | После read и policy | Ambiguous match | Versions/candidate count |
| Temporary unavailable/limit | Backoff+jitter | При исчерпании SLA | attempt, response class |
| Partial | Только failed safe steps | Unknown dependency | Per-step status |
| External system | По её контракту | Permanent/business reject | System alias, category |

## 20. Наблюдаемость

| Сигнал | Содержание |
|---|---|
| Structured log | time, env, tenant alias, correlation/operation ID, result class |
| Exchange journal | source/target IDs, direction, version, safe status |
| Metrics | rate, latency, errors, retry, duplicates, queue depth/age, lag |
| Alerts | SLA breach, permanent errors, stale checkpoint, DLQ growth |
| Dashboard/trace | Сквозной путь без секретов |
| Manual retry | Preview, same operation ID, audit |

```text
correlation_id: receive → transform → queue → API → reconcile → report
```

## 21. Тестирование интеграций

| Тест | Проверка |
|---|---|
| Unit | Validation, transform, dedup, retry decision |
| Integration | Test portal + synthetic data |
| Contract | Request/response/schema/version |
| E2E | Полный бизнес-поток |
| Sandbox/mock | Изоляция и редкие ошибки; mock не заменяет contract |
| Rights/volume | Minimum role и representative load |
| Duplicate/replay | Повтор не меняет итог |
| Outage/recovery | Queue, backoff, checkpoint |
| Rollback | Compensation и reconciliation |
| Compatibility/regression | Старые/новые форматы и стабильность |

## 22. Миграция данных

```text
Inventory → Profile → Clean/Normalize → Mapping/Keys → Dry-run
          → Test import → Reconcile → Approval → Full import
          → Report/Rollback/Idempotent rerun
```

Этапы: инвентаризация; профиль качества; копия оригинала; очистка; нормализация телефонов/email/ИНН/дат/валют; mapping; ключи дублей; dry-run; тестовый импорт; выборочная проверка; согласованная freeze window; полный импорт пачками; сверка количества/сумм/связей; отчёт; откат; повтор по external ID.

| Контроль | Критерий |
|---|---|
| Dry-run | planned/skipped/invalid/duplicate |
| Test | Малая выборка и все побочные автоматизации |
| Full | Checkpoint и batch report |
| Reconcile | Counts, sample, sums, relations |
| Rollback | Заранее согласован и протестирован |

## 23. Документация интеграции

| Артефакт | Минимальное содержание |
|---|---|
| Архитектурная/потоковая схема | Системы, trust boundaries, направления, очереди |
| Data mapping | Поля, types, transform, keys, owner |
| Endpoint/event registry | Alias, contract, owner, environment; без secrets |
| Auth model | Flow, scope, rotation/revoke |
| Error journal/SLA | Categories, retry, alerts, support |
| Run/support/recovery | Deploy, start, diagnose, rerun, rollback |
| Changelog/contact matrix | Versions, decisions, owners, escalation |

## 24. Cloud vs On-Premise

| Возможность | Облако | Коробка | Комментарий архитектора |
|---|---|---|---|
| REST/webhooks | Published contract, tariff/rights | Version/module dependent | Verify target |
| Local apps | Supported portal capabilities | Version dependent | Contract first |
| Server code | Нет | Supported modules | No core edits |
| Cron/queue | External | External or owned infrastructure | Monitor |
| Database | Нет | Owner access | No direct write as integration API |
| Logs/infrastructure | Limited platform control | Owner control | Security/operations required |
| Custom modules | Нет | Supported extensions | Upgrade tests |
| Scaling | Service boundary | Owner responsibility | Load model |
| Updates | Vendor-managed | Owner-planned | Stage, backup, regression |

Источник On-Premise: [официальная документация](https://dev.1c-bitrix.ru/docs/portal.php).

## 25. Типовые ошибки

| № | Ошибка | Последствие |
|---:|---|---|
| 1 | Нет мастер-системы | Конфликты |
| 2 | Свободное двустороннее редактирование | Циклы |
| 3 | Токен в коде | Компрометация |
| 4 | Нет mapping | Искажение данных |
| 5 | Нет внешнего ID | Нет upsert |
| 6 | Синхронизация по названию | Ложные совпадения |
| 7 | Нет идемпотентности | Дубли |
| 8 | Нет очереди для долгого потока | Потеря/timeout |
| 9 | Нет retry временных ошибок | Потеря операций |
| 10 | Retry всех ошибок | Шторм |
| 11 | Нет timeout | Зависание |
| 12 | Нет мониторинга | Скрытый сбой |
| 13 | Нет журнала обмена | Нет аудита |
| 14 | Нет dry-run | Массовая порча |
| 15 | Нет тестового импорта | Ошибка в production |
| 16 | Нет отката | Невосстановимость |
| 17 | Игнорируются права | Сбой/утечка |
| 18 | PII в логах | Утечка |
| 19 | Test и prod смешаны | Перекрёстные данные |
| 20 | Нет SLA | Нет реакции |
| 21 | Нет владельца | Нет ответственности |
| 22 | Batch считается транзакцией | Partial loss |
| 23 | Неполная pagination | Пропуски |
| 24 | Timeout считается неуспехом | Дубли |
| 25 | Нет reconciliation | Тихие расхождения |
| 26 | Все поля запрашиваются | Нагрузка/PII |
| 27 | Mapping по display name | Поломка |
| 28 | Enum без mapping | Неверный статус |
| 29 | Валюта игнорируется | Финансовая ошибка |
| 30 | Timezone игнорируется | Пропуски окна |
| 31 | Роботы не учтены при импорте | Шторм действий |
| 32 | Webhook URL в логах | Утечка credential |
| 33 | Секрет в браузере | Компрометация |
| 34 | Нет ротации/отзыва | Вечный доступ |
| 35 | Нет DLQ | Ошибки теряются |
| 36 | Нет correlation ID | Нет трассировки |
| 37 | Предполагается порядок событий | Старое состояние |
| 38 | Нет schema version | Несовместимость |
| 39 | Один credential для всех систем | Нет изоляции |
| 40 | Нет negative/recovery tests | Сбой при первой ошибке |
| 41 | Реальные данные в тестах | Утечка |
| 42 | Нет freeze/migration window | Дельта теряется |
| 43 | Прямое удаление при сверке | Потеря истории |
| 44 | Коннектор выбран без аудита | Vendor/functional risk |
| 45 | Документация отсутствует | Зависимость от автора |

## 26. Чек-лист интеграционного архитектора

- [ ] Цель, системы, владельцы и SLA согласованы.
- [ ] Master определён по каждому типу/полю.
- [ ] Объекты, direction, frequency, volume и latency измерены.
- [ ] External ID, duplicate и conflict policy заданы.
- [ ] Auth, minimal rights, secrets, environments и PII проверены.
- [ ] Mapping, dictionaries, defaults и errors версионированы.
- [ ] Events/polling/batch/queue выбраны с обоснованием.
- [ ] Timeout, retry, idempotency, DLQ и reconciliation спроектированы.
- [ ] Logs, metrics, alerts, dashboard и manual retry имеют owner.
- [ ] Unit, contract, integration, E2E, rights, volume, replay, outage, recovery и regression tests подготовлены.
- [ ] Dry-run, pilot, acceptance, rollback и support runbooks согласованы.
- [ ] Architecture, flows, registry, SLA, changelog и contacts актуальны.

```text
Requirements → ADR → Contract/mapping → Build → Test → Pilot
             → Accept → Monitor/support → Change/recover
```

## 27. Официальные источники

| Источник | Использование | Проверено в реестре |
|---|---|---|
| [Bitrix24 REST API](https://apidocs.bitrix24.ru/) | REST, auth, events, limits, batch | 2026-08-05 |
| [Справочник API](https://apidocs.bitrix24.ru/api-reference/) | Методы, поля, scope, errors | 2026-08-05 |
| [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/) | Официальные local scenarios | 2026-08-06 |
| [Developer Hub](https://github.com/bitrix24/bitrix24-dev-hub) | Official SDK/examples | 2026-08-05 |
| [REST docs repository](https://github.com/bitrix24/b24restdocs) | Documentation history; not copied | 2026-08-05 |
| [PHP SDK](https://github.com/bitrix24/b24phpsdk) | Official SDK; API contract has priority | 2026-08-05 |
| [Marketplace](https://www.bitrix24.ru/apps/) | Проверка готовых приложений | 2026-08-05 |
| [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) | Integration variants | 2026-08-05 |
| [On-Premise docs](https://dev.1c-bitrix.ru/docs/portal.php) | Box-specific architecture | 2026-08-05 |

Сторонние материалы не использованы. Неподтверждённый provider, connector, method, limit или capability помечается **«требует проверки»**.
