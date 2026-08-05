# Bitrix24 Monitoring Engineering Knowledge Base

Инженерная методология постоянного контроля решений Bitrix24 после внедрения. Документ предназначен для Senior Bitrix24 Solution Architect, Monitoring Engineer, DevOps Engineer и Technical Operations Lead.

> **Статус сведений:** продуктовые факты основаны исключительно на официальных источниках из `knowledge/SOURCES.md`. Доступность отчётов, журналов, REST-данных, BI Конструктора и административных функций зависит от Cloud/On-Premise, тарифа или редакции, версии, модулей и прав. Если возможность не подтверждена, она помечается **«требует проверки»**.

> **Рекомендация архитектора:** мониторинг начинается с критических бизнес-сценариев и качества результата. Технический `HTTP 200`, доступность портала или отсутствие жалоб сами по себе не доказывают работоспособность процесса.

> **Безопасность:** метрики и журналы не должны содержать пароли, токены, полные webhook URL, записи разговоров, персональные данные или клиентские выгрузки. Примеры используют только синтетические значения.

## 1. Основы мониторинга Bitrix24

Мониторинг непрерывно или периодически собирает сигналы, сравнивает их с baseline/SLO и инициирует реакцию до того, как скрытая деградация станет потерей данных или остановкой бизнеса.

| Практика | Назначение | Результат |
|---|---|---|
| Monitoring | Постоянно обнаруживать отклонения | Signal, alert, trend |
| Troubleshooting | Найти причину конкретного симптома | Подтверждённая гипотеза и восстановление |
| Audit | Периодически оценить архитектуру и соответствие | Findings и roadmap |

```text
Внедрение -> Наблюдение -> Обнаружение -> Анализ
    ^                                      |
    +--------- Улучшение <- Исправление <--+
```

Мониторинг должен отвечать: что измеряется, почему это важно, где источник, кто владелец, какая норма, что считается отклонением и какой runbook запускается.

## 2. Архитектура мониторинга

```text
Пользователь -> Bitrix24 -> Автоматизация -> Интеграции
      -> Внешние системы -> Мониторинг -> Alert/Incident/Improvement
```

| Уровень | Сигналы | Риск без контроля |
|---|---|---|
| Пользовательский | Доступность critical journey, ошибки, latency | Проблему первой видит клиент |
| Функциональный | Успешное завершение сценария | Технически доступный, но сломанный процесс |
| CRM | Поток, стадии, SLA, качество данных | Потерянные продажи |
| Автоматизация | Запуски, ошибки, длительность, зависания | Тихий отказ роботов/БП |
| Интеграции | Success/error, lag, backlog, reconciliation | Дубли и рассинхронизация |
| API | Auth, status, response time, error class | Скрытая деградация обмена |
| Инфраструктура | Server/PHP/DB/files/cron для Box | Простой On-Premise |
| Безопасность | Пользователи, права, credentials, apps | Несанкционированный доступ |
| Данные | Полнота, точность, уникальность, freshness | Недостоверные решения |

```text
Signals -> Collection -> Validation -> Metrics -> SLO evaluation
       -> Alert routing -> Runbook -> Incident -> RCA -> Improvement
```

## 3. Cloud мониторинг

В Cloud пользователь контролирует опубликованные интерфейсы, доступную историю, отчёты, BI, настройки, права, приложения и собственные интеграционные компоненты. Внутренние серверы, PHP, БД и платформенные журналы поставщика недоступны.

| Что проверять | Возможный официальный источник | Ограничение |
|---|---|---|
| Critical journey | Синтетическая проверка доступного сценария | Не выполнять разрушительные действия |
| CRM flow/data quality | Доступные CRM и аналитические представления | Состав зависит от тарифа |
| Automation | Доступная история/состояние | Детализация требует проверки |
| REST | Ответы собственного клиента | Не раскрывать credential |
| BI | Доступные datasets и dashboards | Тариф, права, refresh |
| Security | Пользователи, роли, приложения | Набор controls зависит от портала |

**Ограничение:** нельзя приписывать Cloud доступ к CPU, RAM, server logs, PHP workers, DB locks, filesystem или внутренним очередям Bitrix24. Для вопросов внутри managed boundary используется официальная поддержка с обезличенными доказательствами.

## 4. On-Premise мониторинг

On-Premise добавляет полную эксплуатационную ответственность владельца. Применимость средств и параметры сверяются с [официальной документацией коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php).

### Bitrix24 уровень

- CRM, задачи, Smart Processes, роботы и БП;
- REST, приложения, права и события;
- версии продукта и модулей;
- штатные проверки и обновления в доступном составе.

### Серверный уровень

| Компонент | Контроль | Сигнал риска |
|---|---|---|
| Web/PHP | Error rate, latency, workers, совместимость версии | Ошибки/исчерпание workers |
| Database | Connections, latency, locks, growth | Блокировки/рост времени |
| Filesystem | Свободное место, права, latency, целостность | Заполнение/ошибка записи |
| Cache | Размер/эффективность только подтверждёнными средствами | Шторм перестроения |
| Agents/cron | Последний запуск, длительность, backlog | Пропуск/двойной запуск |
| Modules | Версия, зависимость, update status | Несовместимость |

### Инфраструктурный уровень

- CPU, RAM, disk I/O, network и clock synchronization;
- резервное копирование, возраст последней копии и restore test;
- capacity trend и прогноз исчерпания;
- доступность зависимых сервисов;
- security events и изменение конфигурации.

```text
Users -> Load balancer/web -> PHP -> Bitrix24 modules -> DB/files/cache
                                  -> agents/cron -> external systems
```

Не менять ядро, БД, индексы, кеш или cron по одному алерту: сначала подтверждаются причина, совместимость, backup и rollback.

## 5. CRM Monitoring

| Метрика | Что показывает | Источник | Риск | Действие |
|---|---|---|---|---|
| Новые лиды | Входящий поток | CRM/доступная аналитика | Падение источника | Сравнить каналы и формы |
| Новые сделки | Коммерческий поток | CRM | Потеря создания | Проверить sources/automation |
| Элементы по стадиям | Распределение pipeline | CRM | Накопление bottleneck | Проверить aging/SLA |
| Stage age | Зависшие элементы | Даты переходов при доступности | Потерянные сделки | Назначить owner/action |
| Просроченные активности | Дисциплина follow-up | CRM activities | Нет контакта с клиентом | Эскалировать по процессу |
| Заполнение полей | Data quality | CRM/BI | Неверная автоматизация/BI | Исправить источник заполнения |
| Дубли | Identity quality | Доступные инструменты CRM | Разрыв истории | Анализировать ключи, не сливать автоматически |
| Сделки без активности | Потерянный follow-up | CRM | Потеря выручки | Создать контролируемый список |
| Reaction time | Скорость обработки | Согласованные timestamps | Нарушение SLA | Найти источник задержки |
| Конверсия | Качество процесса | CRM Analytics/BI | Ложный вывод при плохих данных | Проверить формулу и данные |

```text
Lead source -> Lead/Deal created -> First activity -> Qualification
 -> Stage progression -> Won/Lost -> Repeat sale
```

Метрика имеет formal definition: entity, фильтры, timestamp, timezone, denominator, exclusions и owner.

## 6. Monitoring автоматизации

| Объект | Контроль | Аномалия | Реакция |
|---|---|---|---|
| Робот | Число ожидаемых/фактических запусков | Не запущен или повторён | Проверить trigger/condition/history |
| БП | Start/success/error/duration | Зависание/ошибка/рост времени | Локализовать activity и actor |
| Переходы | Допустимая sequence | Цикл или пропуск | Проверить transition graph |
| Задачи | Создание, deadline, completion | Дубли/просрочка | Проверить automation owner |
| Уведомления | Delivery intent и получатель | Шум/нет уведомления | Исправить правило/escalation |

- мониторить не только ошибки, но и отсутствие ожидаемого запуска;
- отделять waiting state от зависания;
- использовать synthetic canary только в изолированной области;
- не перезапускать массово процессы до проверки идемпотентности;
- точный состав истории зависит от портала.

## 7. Monitoring REST API

Точный контракт методов и доступные признаки проверяются в [REST API reference](https://apidocs.bitrix24.ru/api-reference/).

```text
Request rate -> HTTP/API status -> Latency -> Retry -> Business outcome
                         |                    |
                      error class          reconciliation
```

| Метрика | Назначение | Ограничение |
|---|---|---|
| Availability | Доля корректных ответов собственного клиента | Не равна business success |
| Auth errors | Отзыв/права/credential lifecycle | Не логировать secret |
| Method errors | Contract/configuration changes | Классифицировать по методу |
| Response time | Деградация клиента/сети/API | Percentiles, не только average |
| Request count | Capacity и аномальный рост | Не задавать неподтверждённый лимит |
| Batch outcomes | Partial failures | Проверять каждую команду |
| Webhook events | Lag, duplicates, loss indicators | Delivery contract проверить |
| OAuth lifecycle | Ошибки авторизации/обновления | Flow и scopes требуют проверки |

Не публиковать универсальные числовые REST limits. Порог строится от официального актуального контракта и собственного baseline.

## 8. Monitoring Webhooks

| Контроль | Метрика/реестр | Реакция |
|---|---|---|
| Владелец | Named owner и backup owner | Эскалация при увольнении |
| Актуальность | Purpose, last review, consumer | Удалить/отозвать по процедуре |
| Безопасность | Scope, secret age без значения | Rotation/revoke |
| Ошибки | Error class/rate | Runbook по transient/permanent |
| Delivery | Lag, duplicates, unknown outcomes | Reconciliation |
| Журналирование | Safe IDs, timing, status | Маскировать URL/ПДн |

```text
Event -> Receiver -> Durable queue -> Worker -> REST enrichment -> Result
           |              |             |              |
        latency        depth/age      retry/DLQ     reconciliation
```

Смена сотрудника запускает проверку ownership, scopes, endpoints, credentials и отзыв неиспользуемого доступа. Не предполагать подпись или гарантированный порядок событий без официального подтверждения.

## 9. Monitoring интеграций

| Интеграция | Источник ошибки | Проверка | Возможное решение |
|---|---|---|---|
| 1С | Mapping, module/app, queue, master conflict | Версия механизма, last success, reconciliation | Исправить подтверждённый слой |
| Телефония | Provider, route, connector, CRM link | Boundary metrics и synthetic call | Joint runbook/provider escalation |
| Внешняя CRM | Contract, identity, ordering | External IDs, versions, lag | Idempotent replay/reconciliation |
| Платежи | Provider status, callback, matching | Transaction alias и business status | Controlled reconciliation |
| Сайт | Form, validation, spam, mapping | Synthetic submission | Исправить source/adapter |
| Приложение | Scope, owner, version, vendor | App register и permission review | Update/revoke/escalate |

```text
Source produced = delivered + queued + processed + rejected + unknown
```

Баланс должен сходиться за согласованное окно. «Unknown» требует reconciliation, а не слепого повтора.

## 10. Monitoring телефонии

Официальный продуктовый контекст: [варианты подключения телефонии](https://helpdesk.bitrix24.ru/open/6450911/).

| Область | Сигнал | Проверка |
|---|---|---|
| Доступность | Failed test route/provider status | Разделить provider и Bitrix24 |
| Входящие | Count, missed, route outcome | Рабочее время/очередь/fallback |
| Исходящие | Attempts/success | Права, способ подключения, provider |
| Запись | Expected vs available | Настройка, права, storage, compliance |
| Распределение | Queue wait/agent outcome | Route matrix |
| Очереди | Depth/wait/overflow | Capacity/fallback |
| CRM cards | Linked/unlinked/duplicate | Нормализация и duplicate policy |

Тестовые звонки выполняются по согласованному синтетическому маршруту. Номера, SIP-логины, записи и домены в документации не сохраняются.

## 11. Monitoring BI и аналитики

Доступность BI Конструктора и datasets зависит от тарифа, прав и конфигурации. Официальная отправная точка — [наборы данных BI](https://helpdesk.bitrix24.ru/open/24355590/).

- freshness: время последнего подтверждённого обновления;
- completeness: ожидаемые и фактические строки/периоды;
- validity: типы, диапазоны, обязательные значения;
- consistency: контрольные суммы между источниками;
- uniqueness: дубли по согласованным ключам;
- permissions: доступ к datasets, dashboards и публикации;
- publication: владелец, аудитория, версия и дата проверки.

| Аномалия | Проверка | Риск |
|---|---|---|
| Dashboard устарел | Refresh window/source update | Решение по старым данным |
| Суммы расходятся | Grain/formula/timezone/currency | Ложный KPI |
| Пропал период | Filters/dataset/load | Неполный отчёт |
| Видны лишние данные | Role/publication test | Утечка |

## 12. Monitoring безопасности

| Объект | Контроль | Событие реакции |
|---|---|---|
| Пользователи | Active/disabled, owner, department | Неактуальная учётная запись |
| Администраторы | Named list и review | Необоснованное добавление |
| Права | Role matrix и exceptions | Расширение доступа |
| Webhooks | Owner, scope, age, last use | Orphan credential |
| Приложения | Vendor, scopes, purpose | Изменение/избыточный scope |
| Внешние доступы | Named, time-bound | Истёкший договор/срок |
| Offboarding | User, sessions, tokens, ownership | Неотозванный доступ |

```text
Joiner -> Role assignment -> Periodic review -> Change -> Offboarding
                         \-> Exception register -> expiry
```

Доступность журналов входа, устройств, 2FA, SSO и иных controls **требует проверки** на конкретном портале.

## 13. Метрики эксплуатации

| Метрика | Цель | Частота проверки | Ответственный |
|---|---|---|---|
| Critical journey success | Работоспособность бизнеса | Непрерывно/регулярно по риску | Operations owner |
| CRM creation lag | Скорость поступления | Согласованное окно | CRM owner |
| Stage aging | Bottleneck процесса | Ежедневно | Sales/process owner |
| Automation error/absence | Надёжность автоматизации | Регулярно | Automation owner |
| API error/latency | Стабильность обмена | Непрерывно | Integration owner |
| Queue depth/age | Накопление работ | Непрерывно | Integration owner |
| Reconciliation mismatch | Целостность данных | По циклу обмена | Data owner |
| BI freshness | Актуальность аналитики | По refresh schedule | BI owner |
| Access exceptions | Безопасность | Еженедельно/месячно | Security owner |
| Backup/restore evidence (Box) | Восстановимость | По политике | Infrastructure owner |

Технические метрики описывают компоненты; бизнес-метрики — результат; process quality — правильность прохождения; integration stability — доставку и согласованность. Метрика без owner и действия не считается эксплуатационной.

## 14. SLA / SLO

| Термин | Значение | Пример без универсального порога |
|---|---|---|
| SLI | Измеримый индикатор | Доля успешно созданных сделок |
| SLO | Внутренняя цель для SLI | Согласованная доля за окно |
| SLA | Обязательство и последствия | Договорное время реакции |
| Availability | Возможность выполнить функцию | Synthetic critical journey |
| Reaction time | От обнаружения до начала работы | Timestamp incident/ack |
| Recovery time | От сбоя до восстановления | Timestamp restore-start |

```text
Business objective -> SLI definition -> SLO + window -> Alert policy
                                      -> SLA where contractually agreed
```

Универсальных значений нет. Цели зависят от критичности, рабочего времени, тарифа/редакции, provider contracts и стоимости контроля.

## 15. Alerting

Alert создаётся только для actionable отклонения с owner и runbook. Dashboard без alerting не обеспечивает своевременную реакцию.

| Severity | Пример | Реакция |
|---|---|---|
| Critical | Потеря/раскрытие данных, остановка critical journey | Немедленный incident и containment |
| High | Массовый отказ интеграции без обхода | Срочная диагностика и business notice |
| Medium | Рост backlog с запасом времени | Плановое устранение до нарушения SLO |
| Low | Тренд capacity/quality без текущего ущерба | Backlog и review |

- дедуплицировать одинаковые сигналы;
- группировать по причинной границе;
- использовать duration/hysteresis, где оправдано;
- иметь maintenance suppression с конечным сроком;
- измерять alert volume, false positives и time-to-ack;
- регулярно удалять неактуальные правила.

## 16. Dashboard архитектура

| Dashboard | Аудитория | Основные блоки |
|---|---|---|
| Операционный | Support/operations | Incidents, alerts, critical journeys, backlog |
| CRM | Sales/process owners | Flow, aging, activities, data quality, conversion |
| Интеграционный | Integration team | Errors, latency, queues, reconciliation, providers |
| Технический | DevOps/Box admins | Web/PHP/DB/files/cron/capacity |
| Руководительский | Business owners | SLO, impact, trends, risks, improvement |

```text
Executive: few outcome indicators
Operational: actionable service signals
Technical: component diagnostics
Raw logs: investigation only, protected access
```

Dashboard фиксирует definition, source, refresh, owner, audience и last review. Красно-зелёный индикатор без контекста и drill path недостаточен.

## 17. Incident Monitoring

```text
Monitoring -> Alert -> Incident -> Troubleshooting -> Recovery
                                         -> RCA -> Preventive monitoring
```

| Связь | Обязательные данные |
|---|---|
| Alert → Incident | Rule ID, start time, metric, threshold, scope |
| Incident → Troubleshooting | Timeline, environment, evidence, changes |
| Recovery → Monitoring | Success criteria, observation window |
| RCA → Improvement | New/changed SLI, alert, runbook, test |

Monitoring продолжает работать во время инцидента и доказывает восстановление. После RCA проверяется, мог ли сигнал обнаружить проблему раньше.

## 18. Capacity Monitoring

| Рост | Индикатор | Планирование |
|---|---|---|
| База/файлы | Volume и growth rate | Retention/archive/storage |
| Пользователи | Active users/concurrency | Licenses, roles, support load |
| Автоматизации | Starts/duration/backlog | Simplification и scheduling |
| Интеграции | Requests/events/queue | Throughput, batching, workers |
| BI | Dataset volume/refresh time | Grain, retention, query design |

```text
Current usage + growth trend + seasonality + planned changes
 -> forecast -> lead time -> capacity action -> verify
```

Cloud capacity контролируется через доступные прикладные сигналы и официальные ограничения; инфраструктурная capacity Cloud не управляется клиентом. On-Premise требует прогнозов CPU, RAM, disk, DB, files, network и backup window.

## 19. Регулярные проверки

### Ежедневный чек-лист

- [ ] critical journeys и активные incidents просмотрены;
- [ ] automation/API/integration errors проверены;
- [ ] queue depth/age и unknown outcomes контролируются;
- [ ] CRM flow, зависшие элементы и просроченные активности просмотрены;
- [ ] телефония и критические providers доступны;
- [ ] BI freshness соответствует расписанию;
- [ ] Critical/High alerts имеют владельца.

### Еженедельный чек-лист

- [ ] trends error rate, latency и backlog проанализированы;
- [ ] webhook/app ownership и неиспользуемые credentials просмотрены;
- [ ] data quality и reconciliation exceptions обработаны;
- [ ] noisy/stale alerts пересмотрены;
- [ ] capacity anomalies и повторные incidents обсуждены;
- [ ] runbooks для новых сигналов добавлены.

### Ежемесячный чек-лист

- [ ] SLI/SLO и business impact review выполнен;
- [ ] пользователи, администраторы и access exceptions проверены;
- [ ] CRM conversion/aging и process quality reviewed;
- [ ] интеграционные contracts/providers reviewed;
- [ ] dashboard definitions и audience актуальны;
- [ ] backup/restore evidence для Box проверено;
- [ ] monitoring coverage и blind spots зарегистрированы.

### Квартальный аудит

- [ ] критические процессы и ownership подтверждены бизнесом;
- [ ] architecture/dataflow и monitoring map актуальны;
- [ ] alert effectiveness и incident lessons оценены;
- [ ] Cloud/On-Premise, тариф/редакция, версии и модули сверены;
- [ ] capacity forecast и roadmap обновлены;
- [ ] security/offboarding/app reviews выполнены;
- [ ] synthetic checks и disaster/restore evidence пересмотрены;
- [ ] официальные источники и ограничения проверены повторно.

## 20. Типовые ошибки мониторинга

| № | Ошибка | Последствие / исправление |
|---:|---|---|
| 1 | Мониторинг появляется после аварии | Проектировать до запуска |
| 2 | Нет monitoring owner | Назначить ответственность |
| 3 | Нет владельца метрики | Назначить data/service owner |
| 4 | Нет baseline | Собрать нормальное поведение |
| 5 | Нет SLI definition | Формализовать источник и формулу |
| 6 | Нет SLO | Согласовать цель и окно |
| 7 | SLA назван без договора | Отделить internal SLO |
| 8 | Универсальные пороги скопированы | Настроить по риску/baseline |
| 9 | Мониторится только доступность портала | Добавить critical journeys |
| 10 | HTTP 200 считается business success | Проверить итоговое состояние |
| 11 | Нет контроля отсутствия событий | Добавить missing-event signal |
| 12 | Нет контроля качества данных | Ввести DQ metrics |
| 13 | Нет контроля freshness | Измерять last confirmed update |
| 14 | Нет reconciliation | Сводить source/target balance |
| 15 | Unknown outcomes игнорируются | Создать review/reconciliation |
| 16 | Нет владельца alert | Настроить routing |
| 17 | Нет runbook | Связать alert с действием |
| 18 | Слишком много алертов | Дедупликация и tuning |
| 19 | Шумные алерты игнорируются | Измерять false positives |
| 20 | Alert не тестируется | Провести synthetic trigger |
| 21 | Нет escalation path | Назначить уровни реакции |
| 22 | Maintenance suppression бессрочен | Задать expiry |
| 23 | Severity равна Priority | Разделить impact/urgency |
| 24 | Нет времени обнаружения | Сохранять timestamps/timezone |
| 25 | Dashboard заменяет alerting | Добавить actionable alerts |
| 26 | Dashboard без owner | Назначить поддержку |
| 27 | Dashboard без definition | Документировать формулы |
| 28 | Dashboard показывает среднее | Добавить percentiles/distribution |
| 29 | Нет drill path | Связать с безопасным evidence |
| 30 | Отчёт публикуется всем | Проверить права/audience |
| 31 | Cloud мониторят как собственный сервер | Соблюдать managed boundary |
| 32 | Придумывают внутренние Cloud metrics | Использовать published signals |
| 33 | Тарифные функции считаются гарантированными | Проверить портал |
| 34 | On-Premise мониторят только через UI | Добавить infrastructure layers |
| 35 | CPU без business correlation | Сопоставить с latency/outcome |
| 36 | Disk growth не контролируется | Forecast exhaustion |
| 37 | DB locks игнорируются | Наблюдать безопасно |
| 38 | PHP errors не агрегируются | Ввести error classes |
| 39 | cron контролируется только по process presence | Проверять last success/result |
| 40 | Агенты запускаются дважды | Контролировать concurrency |
| 41 | Кеш очищается по алерту | Сначала подтвердить причину |
| 42 | Версии модулей не инвентаризированы | Вести baseline |
| 43 | Update status не контролируется | Планировать review/staging |
| 44 | Backup проверяется только по наличию файла | Делать restore test |
| 45 | Backup age неизвестен | Метрика last successful backup |
| 46 | RPO/RTO не связаны с мониторингом | Добавить breach signals |
| 47 | Никто не проверяет ошибки БП | Назначить automation owner |
| 48 | Waiting БП считается ошибкой | Отличать допустимое ожидание |
| 49 | Зависший БП не обнаруживается | Контролировать duration/state |
| 50 | Робот мониторится только при ошибке | Контролировать expected starts |
| 51 | Повтор робота не замечается | Считать duplicate effects |
| 52 | Цикл стадий не обнаруживается | Анализировать transition frequency |
| 53 | Задачи создаются дублями | Мониторить key/count |
| 54 | Уведомления создают шум | Контролировать volume/response |
| 55 | CRM flow не мониторится | Добавить funnel inputs/stages |
| 56 | Зависшие сделки не определены | Формализовать stage aging |
| 57 | Просроченные активности игнорируются | Ввести owner/action |
| 58 | Дубли мониторятся без правил ключей | Согласовать identity policy |
| 59 | Потерянные сделки определяются субъективно | Задать формулу |
| 60 | Conversion без проверки данных | Добавить DQ prerequisite |
| 61 | REST auth errors смешаны со method errors | Классифицировать слои |
| 62 | Не измеряется API latency | Добавить percentiles |
| 63 | Request count без business context | Связать с operations |
| 64 | Batch считается одним успехом | Проверять subcommands |
| 65 | Публикуется неподтверждённый REST limit | Сверять official contract |
| 66 | OAuth lifecycle не контролируется | Ввести owner/expiry/error monitoring |
| 67 | Webhook не имеет владельца | Назначить owner/backup |
| 68 | Нет реестра webhook purpose/scope | Создать inventory без secret |
| 69 | Webhook URL попал в лог | Маскировать и ротировать |
| 70 | Нет контроля увольнений | Связать с offboarding |
| 71 | Duplicate delivery создаёт дубли | Идемпотентный consumer |
| 72 | Out-of-order не учитывается | Version/reconciliation |
| 73 | Queue depth без age | Измерять оба сигнала |
| 74 | Retry storm не обнаруживается | Retry rate/backoff metrics |
| 75 | Dead-letter не имеет owner | Настроить review SLA |
| 76 | 1С мониторится как единый универсальный механизм | Фиксировать конфигурацию/app |
| 77 | External ID не контролируется | Добавить identity quality |
| 78 | Mapping version неизвестна | Версионировать contract |
| 79 | Provider status принят за полный результат | Проверять end-to-end |
| 80 | Платёжный callback равен оплате | Сверять business status |
| 81 | Сайт проверяется только открытием страницы | Делать synthetic submission |
| 82 | Marketplace app без permission review | Контролировать scopes/vendor |
| 83 | Телефонию проверяет первый клиент | Использовать safe synthetic call |
| 84 | Caller ID считается уникальным | Контролировать normalization/duplicates |
| 85 | Записи разговоров мониторятся без compliance | Учитывать права/retention |
| 86 | BI freshness не измеряется | Контролировать update window |
| 87 | BI grain не документирован | Определить единицу строки |
| 88 | Timezone dashboards различается | Нормализовать время |
| 89 | Права BI не проверяются | Включить security monitoring |
| 90 | Инцидент закрывают без observation window | Подтвердить стабильность |

## 21. Практические сценарии

| № | Симптом | Проверка | Вероятная причина | Решение |
|---:|---|---|---|---|
| 1 | Перестали создаваться сделки | Source count, API/automation errors, last success | Изменение формы/mapping/прав | Локализовать слой и восстановить contract |
| 2 | Интеграция не обновляет данные | Queue, external ID, response, reconciliation | Backlog/ошибка mapping/auth | Исправить причину, controlled replay |
| 3 | Робот не запускается | Expected starts vs history, trigger/condition | Неверный вход/условие | Исправить и проверить synthetic entity |
| 4 | REST возвращает ошибку | HTTP/API class, auth, method, params | Credential/rights/contract | Исправить подтверждённый слой |
| 5 | Менеджеры не видят клиентов | Role, department, ownership, direct URL | Конфликт/изменение прав | Вернуть least privilege |
| 6 | Телефония недоступна | Provider boundary, route, connector | Provider/config/network | Joint runbook/fallback |
| 7 | BI показывает неверные данные | Grain, formula, refresh, timezone, sample | Data quality/filter/formula | Исправить metric contract/data |
| 8 | Растёт очередь webhook | Ingress, processing, retry, downstream | Slow consumer/retry storm | Throttle/fix consumer/reconcile |
| 9 | Создаются дубли контактов | Identity metrics, source keys | Нормализация/idempotency | Исправить key policy; отдельная очистка |
| 10 | Сделки зависли на стадии | Stage age, activities, automation | Нет owner/task/transition | Исправить процесс и эскалацию |
| 11 | БП долго выполняется | Instance duration/state/activity | Valid wait или stalled activity | Разделить ожидание/ошибку |
| 12 | OAuth-интеграция массово отказала | Auth error trend, owner/change | Отзыв/scope/lifecycle | Выполнить официальный recovery flow |
| 13 | Платежи есть, статусы не обновлены | Provider vs CRM balance | Callback/queue/matching | Reconciliation и controlled replay |
| 14 | Исчезла запись звонка | Expected call sample, rights/config/storage | Настройка/право/provider | Исправить подтверждённый слой |
| 15 | On-Premise стал медленным | User journey + PHP/DB/files/agents metrics | Resource/backlog/change | RCA и безопасная оптимизация |
| 16 | После увольнения работает интеграция | User/app/webhook ownership review | Orphan credential | Переназначить и отозвать доступ |

Каждый сценарий завершается verification, observation window и обновлением monitoring rule/runbook, если обнаружен новый blind spot.

## 22. Cloud vs On-Premise

| Область | Cloud | On-Premise | Ограничения |
|---|---|---|---|
| Platform internals | Управляет поставщик | Управляет владелец | В Cloud нет server/DB доступа |
| Application monitoring | Доступные UI/history/REST/BI | То же + server evidence | Состав зависит от версии/прав |
| Server metrics | Недоступны клиенту | CPU/RAM/disk/network | Нужен infra owner |
| PHP/web | Недоступны клиенту | Logs/workers/latency | Совместимость версии |
| Database | Недоступна клиенту | Growth/locks/connections | Не менять схему вслепую |
| Files/cache | Только опубликованные функции | Capacity/rights/latency | Cache action требует RCA |
| Agents/cron | Внутренности недоступны | Last run/duration/backlog | Повтор должен быть безопасен |
| Backup | Доступные официальные средства | Политика владельца + restore test | Export не равен полному backup |
| Updates | Vendor-managed | Owner-managed | Box: staging/regression/rollback |
| Escalation | Official support | Internal infra + official support | Evidence без секретов/ПДн |

## 23. Чек-лист Monitoring Architect

### Контекст и ownership

- [ ] Cloud/On-Premise, тариф/редакция, версия и права зафиксированы;
- [ ] critical services и business journeys определены;
- [ ] у каждой системы, метрики, alert и runbook есть owner;
- [ ] provider contracts и escalation contacts зарегистрированы;
- [ ] monitoring scope и blind spots приняты.

### CRM и пользователи

- [ ] lead/deal flow, stages, aging и activities контролируются;
- [ ] data quality, duplicates и lost-deal rules формализованы;
- [ ] users/admins/roles/exceptions регулярно reviewed;
- [ ] offboarding связан с apps/webhooks/integrations;
- [ ] negative access signals покрыты.

### Автоматизация

- [ ] expected starts, success, errors и duration измеряются;
- [ ] waiting и stalled states различаются;
- [ ] duplicate effects и stage loops обнаруживаются;
- [ ] tasks/notifications имеют quality metrics;
- [ ] automation runbook безопасен и идемпотентен.

### API, webhooks и интеграции

- [ ] HTTP и API errors разделены;
- [ ] auth, latency, volume и partial batch контролируются;
- [ ] webhook owner, purpose, scope и revoke procedure существуют;
- [ ] queue depth, age, retry и dead-letter наблюдаемы;
- [ ] external IDs, mapping versions и reconciliation контролируются;
- [ ] secrets отсутствуют в logs/dashboards/docs.

### Телефония и BI

- [ ] provider/Bitrix24 boundary и route metrics определены;
- [ ] calls, queues, distribution и CRM links контролируются;
- [ ] recording monitoring учитывает права и compliance;
- [ ] BI freshness, completeness, formulas и permissions проверяются;
- [ ] dashboard publication имеет owner и audience.

### Производительность и резервирование

- [ ] baseline, percentiles, workload и capacity trends существуют;
- [ ] Cloud managed boundary соблюдён;
- [ ] Box server/PHP/DB/files/cache/agents/cron покрыты;
- [ ] backup age, integrity и restore evidence контролируются;
- [ ] SLI/SLO и alert thresholds связаны с business impact.

### Эксплуатация и документация

- [ ] daily/weekly/monthly/quarterly routines назначены;
- [ ] alert noise и effectiveness регулярно reviewed;
- [ ] Monitoring → Incident → Troubleshooting → RCA связан IDs;
- [ ] technical и business success criteria определены;
- [ ] changelog, architecture, metric catalog и runbooks актуальны;
- [ ] официальные источники и применимость функций перепроверяются.

## 24. Официальные источники

Дата последней проверки источников хранится в `knowledge/SOURCES.md`; перед проектным решением актуальность проверяется повторно.

| Источник | Тип / приоритет | Использование в мониторинге |
|---|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | Официальная пользовательская документация / высокий | Доступные функции, настройки и ограничения Cloud |
| [Документация REST API](https://apidocs.bitrix24.ru/) | Официальная документация разработчика / критический | REST, webhooks, OAuth и интеграционные границы |
| [REST API reference](https://apidocs.bitrix24.ru/api-reference/) | Официальный справочник / критический | Методы, параметры и ошибки |
| [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/) | Официальная документация / критический | Webhook и локальные интеграционные сценарии |
| [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) | Официальная статья / высокий | Контроль Smart Processes |
| [Варианты подключения телефонии](https://helpdesk.bitrix24.ru/open/6450911/) | Официальная статья / высокий | Телефония и provider boundary |
| [Наборы данных BI](https://helpdesk.bitrix24.ru/open/24355590/) | Официальная статья / высокий | BI datasets и аналитика |
| [Документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php) | Официальная документация 1С-Битрикс / критический | On-Premise, модули и инфраструктура |

Сторонние блоги и форумы не используются как источник возможностей Bitrix24. Если официальный источник не подтверждает метрику, журнал, лимит, механизм alerting или системный интерфейс, он считается **требующим проверки**, а не дополняется предположением.
