# Bitrix24 Analytics Engineering Knowledge Base

Эталонная инженерная база знаний для архитектуры CRM-аналитики и BI. Любой показатель считается достоверным только при формальном определении, назначенном владельце, проверенном источнике и контролируемом качестве данных.

> **Статус:** факты основаны только на `knowledge/SOURCES.md`. BI Конструктор, наборы данных, внешние SQL/Power BI и коробочные возможности зависят от тарифа, версии, прав и конфигурации. Неподтверждённая функция помечается **«требует проверки»**.

> **Безопасность:** только обезличенные или синтетические данные. Реальные отчёты, суммы, ФИО, телефоны, токены и клиентские выгрузки в Git запрещены.

## 1. Архитектура аналитики Bitrix24

| Контракт | Практика / лучшая практика | Ошибка | Ограничение |
|---|---|---|---|
| Источник → модель → метрика → визуализация → решение | Назначить master, grain, owner, refresh и SLA | Dashboard без semantic layer | Доступные источники зависят от продукта |

```text
Bitrix24/External → ETL/BI Dataset → Dimensions+Measures → Dashboard → Action
```

## 2. CRM Analytics

| Инженерное описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Анализ лидов, сделок, клиентов, активностей и стадий | Единые определения дат, сумм и исходов | Считать карточки без дублей/качества | Штатный состав отчётов зависит от тарифа |

```text
CRM entities → lifecycle facts → conversion/time/value metrics
```

## 3. BI Конструктор

| Описание | Рекомендация | Ошибка | Ограничение |
|---|---|---|---|
| Официальный BI-инструмент с доступными наборами данных | Начать со схемы и grain; проверить права и лимиты | SQL до понимания данных | Доступность и объём зависят от тарифа |

```text
Official datasets → SQL/dataset model → chart → dashboard
```

Источник: [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/).

## 4. Источники данных

| Источник | Практика | Ошибка | Ограничение |
|---|---|---|---|
| CRM, задачи, телефония, процессы и подтверждённые внешние источники | Реестр source/owner/grain/refresh | Смешать master и копию | Набор источников зависит от BI и подключений |

```text
Operational sources → governed ingestion → analytical model
```

## 5. Наборы данных (Datasets)

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Набор фактов/атрибутов с документированным grain | Паспорт: поля, ключи, даты, PII, refresh | Join many-to-many без контроля | Поля меняются; сверять официальный каталог |

```text
Dataset = grain + keys + dimensions + measures + quality rules
```

## 6. SQL Dataset

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Производный набор через SQL в официально поддерживаемом BI-контексте | Версионировать query, тестировать row count | `SELECT *`, cartesian join | Применимость SQL и функции требуют проверки тарифа |

```text
Base datasets → validated SQL → virtual dataset → report
```

## 7. Пользовательские отчёты

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Отчёт под конкретное управленческое решение | Owner, audience, filter defaults, definitions | Один отчёт для всех ролей | Конструктор/поля зависят от инструмента |

```text
Question → metric specification → report → decision log
```

## 8. Стандартные отчёты CRM

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Штатные представления CRM | Проверить семантику и период до сравнения | Считать UI-цифру универсальной | Набор отчётов зависит от тарифа/версии |

```text
CRM configuration + data quality → standard report
```

## 9. Воронки продаж

| Метрика | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Количество/сумма и переходы по стадиям | Cohort входа, стабильные стадии, won/lost rules | Смешать snapshot и flow | История стадий должна быть доступна и качественна |

```text
Entry cohort → Stage A → Stage B → Won/Lost
```

## 10. Конверсии

| Формула | Практика | Ошибка | Ограничение |
|---|---|---|---|
| `Converted / Eligible base` | Зафиксировать numerator, denominator, window | Делить разные когорты | Возвраты и повторные переходы требуют policy |

```text
Eligible cohort → outcome count / cohort count → conversion
```

## 11. Сквозная аналитика

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Связь рекламного касания, CRM и подтверждённого результата | Attribution model, UTM policy, cost master | Приписать всю выручку last click | Возможности и источники зависят от подключений |

```text
Ad cost → visit/lead → deal → confirmed revenue → attribution
```

## 12. KPI отдела продаж — «Аналитика отдела продаж»

| Блок | Определение |
|---|---|
| План/факт/выполнение | Согласованный план и подтверждённый факт; `% = fact/plan` |
| Прогноз/pipeline | Взвешенная методика с проверенной вероятностью, не обещание |
| Конверсии/средний чек | По одной когорте, валюте и исходу |
| ROMI/источники | Только с подтверждёнными затратами и attribution |
| Повторы/когорты | Новая сделка существующего клиента и период первого события |
| SLA/скорость реакции | Точные события поступления и первого ответа |
| Активность менеджеров | Контекст, результат и качество, а не число кликов |

```text
Plan + Pipeline + Activity + SLA + Conversion + Revenue/ROMI → Sales management
```

## 13. KPI руководителя

| KPI | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Forecast accuracy, team conversion, SLA, data quality | Баланс результата и качества процесса | Только выручка | Зависит от полномочий и состава команды |

```text
Team facts → controllable KPIs → forecast/coach/resource decision
```

## 14. KPI менеджера

| KPI | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Реакция, next step, conversion, value, quality | Нормировать по потоку/сегменту | Сравнивать несопоставимые портфели | Team selling требует attribution policy |

```text
Assigned portfolio → actions/outcomes → normalized KPI
```

## 15. Финансовая аналитика

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Выручка, оплата, маржа, дебиторка по master-системе | Currency, tax, recognition date | Сумма сделки = деньги | Часто master — ERP/1С, не CRM |

```text
CRM opportunity ≠ invoice ≠ payment ≠ recognized revenue
```

## 16. ABC-анализ

| Расчёт | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Ранжирование по доле выбранной ценности | Задать metric/period/threshold policy | ABC по смешанным валютам | Порог — экспертная настройка, не функция Bitrix24 |

```text
Aggregate value → sort desc → cumulative share → A/B/C
```

## 17. XYZ-анализ

| Расчёт | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Стабильность спроса по вариативности периодов | Равные интервалы и достаточная история | Нули/сезонность игнорируются | Выполняется аналитической моделью при наличии данных |

```text
Time buckets → mean/variation → X/Y/Z
```

## 18. RFM-анализ

| Расчёт | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Recency, Frequency, Monetary подтверждённых покупок | Customer identity и scoring version | Сделки вместо оплат | Требуется дедупликация клиента |

```text
Customer purchases → R + F + M scores → segment
```

## 19. Cohort Analysis

| Расчёт | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Группа по первому событию и поведение по возрасту | Immutable cohort date | Календарный и cohort view смешаны | Нужна полная история |

```text
First event month → cohort → month N retention/value
```

## 20. LTV

| Формула | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Накопленная маржа/выручка клиента за определённый горизонт | Указать gross/net, horizon, cohort | Бессрочный прогноз без модели | Часто нужен DWH и финансовый master |

```text
Customer revenue/margin over time → retention model → LTV
```

## 21. CAC

| Формула | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Acquisition costs / new acquired customers | Полный cost scope и cohort | Только рекламный spend | Cost data обычно внешние |

```text
Sales+marketing acquisition cost / new customers → CAC
```

## 22. CPL

| Формула | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Marketing cost / valid leads | Валидный лид и attribution | Спам в denominator | Требуются затраты и качество лидов |

```text
Attributed spend / qualified lead base → CPL
```

## 23. CPO

| Формула | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Cost / confirmed orders | Определить order master и отмены | Сделка = заказ | Заказы могут быть внешней сущностью |

```text
Attributed cost / confirmed orders → CPO
```

## 24. ROMI

| Формула | Практика | Ошибка | Ограничение |
|---|---|---|---|
| `(Attributed return - marketing cost)/marketing cost` | Зафиксировать return и attribution | Revenue без маржи/затрат | Не штатная универсальная цифра |

```text
Attributed outcome − marketing cost → divide by marketing cost
```

## 25. ROAS

| Формула | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Attributed revenue / ad spend | Одна валюта и окно | Путать с ROMI | Нужны рекламные расходы |

```text
Ad-attributed revenue / ad spend → ROAS
```

## 26. Средний чек

| Формула | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Confirmed sales value / confirmed sales count | Возвраты, валюта, нулевые чеки | Среднее без распределения | Master результата должен быть определён |

```text
Valid value / valid transaction count → average check
```

## 27. Повторные продажи

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Новая подтверждённая продажа существующему клиенту | Customer key + gap/window policy | Перезаписанная старая сделка | Дубли клиентов искажают показатель |

```text
Customer first sale → later independent sale → repeat flag
```

## 28. Отчёты по менеджерам

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Портфель, поток, активность, SLA и результат | Нормировать по сегменту и ownership history | Текущий ответственный получает всю историю | История назначений должна быть доступна |

```text
Ownership periods + facts → fair manager attribution
```

## 29. Источники лидов

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Нормализованный канал появления обращения | Controlled dictionary и unknown bucket | Свободный текст/перезапись | Source зависит от канала и интеграции |

```text
Raw channel → normalized source → lead/deal attribution
```

## 30. UTM

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Campaign metadata первого/последнего касания по policy | Normalize case, preserve raw, bot filter | UTM = доказанная причина продажи | Cookies/identifiers требуют правового основания |

```text
UTM raw → normalization → touchpoint → attribution model
```

## 31. Каналы продаж

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Управляемая классификация источника и способа продажи | Channel dimension, owner и mapping | Смешать source/channel/product | Внешние каналы зависят от коннектора |

```text
Source → channel grouping → funnel/value comparison
```

## 32. Аналитика бизнес-процессов

| Метрики | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Starts, outcomes, duration, waits, errors, SLA | Process/version/outcome dimensions | Все активные = зависшие | Доступность dataset зависит от BI/версии |

```text
BP instances → activity timeline → duration/outcome/error metrics
```

## 33. Аналитика роботов

| Метрики | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Запуск, успех, ошибка, эффект на процесс | Automation registry и correlation | Число запусков = ценность | Детальность журналов зависит от продукта |

```text
Robot trigger → execution result → business outcome linkage
```

## 34. Аналитика Smart Processes

| Метрики | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Items, stages, duration, outcome, owner | `entityTypeId`, category/stage mapping | Смешать разные типы | Dataset зависит от BI и настройки |

```text
Smart Process items → stage history → SLA/conversion/value
```

## 35. Аналитика телефонии

| Метрики | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Calls, answered/missed, duration, wait, CRM result | Normalize number, route/team dimensions | Duration = quality | Данные зависят от подключения и прав |

```text
Telephony facts → CRM relation → SLA/outcome by route/team
```

## 36. Аналитика SLA

| Формула | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Within SLA / eligible events; breach duration | Business calendar, pause policy | Calendar time вместо business time | Механизм календаря может требовать внешней модели |

```text
Start → allowed pauses/calendar → completion → SLA status
```

## 37. Аналитика задач

| Метрики | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Created/completed/overdue, duration, workload, result | Project/team/type dimensions | Количество = производительность | Наборы полей зависят от BI/версии |

```text
Task lifecycle → timeliness/workload/result metrics
```

## 38. Dashboards

| Контракт | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Audience + decisions + refresh + owner | 5–9 ключевых сигналов, definitions link | «Стена графиков» | Refresh и права зависят от BI |

```text
Strategic → tactical → operational dashboards
```

## 39. Визуализация данных

| Выбор | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Line=time, bar=compare, funnel=flow, table=detail | Честные axes, units, denominator | 3D/pie overload | Chart types зависят от инструмента |

```text
Analytical question → visual encoding → annotation → action
```

## 40. Drill Down

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Переход по иерархии детализации | Defined hierarchy и row-level access | Несогласованные levels | Поддержка зависит от BI-инструмента |

```text
Year → Quarter → Month → Day
```

## 41. Drill Through

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Переход из агрегата к связанному detail context | Filter context и PII control | Раскрыть закрытые записи | Поддержка требует проверки BI |

```text
KPI cell → filtered detail page → governed record set
```

## 42. Dimension

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Атрибут группировки: время, источник, менеджер, продукт | Conformed dimensions и surrogate key в DWH | Много разных справочников | Историзация требует модели |

```text
Fact ─many-to-one→ Dimension
```

## 43. Measure

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Агрегируемый расчёт с формальной семантикой | Additivity, currency, grain | Sum of ratios | Expression support зависит от BI |

```text
Row facts → aggregation context → measure
```

## 44. ETL

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Extract, transform, load с checkpoint и audit | Incremental, idempotent, schema version | Full reload всегда | REST-лимиты и внешняя инфраструктура |

```text
Extract → validate/normalize → stage → load → reconcile
```

## 45. Data Warehouse

| Описание | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Историческое аналитическое хранилище | Facts, dimensions, SCD, lineage | CRM как DWH | Внешний DWH не является штатной функцией Bitrix24 |

```text
Sources → staging → core DWH → marts → BI
```

## 46. Data Quality

| Измерения | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Completeness, validity, uniqueness, consistency, timeliness | Rule owner + threshold + remediation | Средний score скрывает critical field | Качество зависит от процесса ввода |

```text
Data → quality rules → score/issues → owner remediation
```

## 47. Проверка качества данных

| Проверка | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Null, enum, duplicates, dates, relations, totals | Automated profile + sample | Проверять только до отчёта | Некоторые правила внешние |

```text
Profile → validate → exceptions → fix → recheck
```

## 48. Очистка данных

| Процесс | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Normalize, merge candidates, correct, archive | Backup, dry-run, sample, approval, rollback | Массовое исправление original | Дубли не объединять без согласования |

```text
Original copy → proposed changes → approval → apply → reconcile
```

## 49. Типичные ошибки аналитики

| № | Ошибка | Последствие |
|---:|---|---|
| 1 | Нет владельца метрики | Споры |
| 2 | Нет определения numerator | Разные цифры |
| 3 | Нет denominator | Ложная конверсия |
| 4 | Смешаны когорты | Неверная динамика |
| 5 | Snapshot выдан за flow | Ошибка воронки |
| 6 | Сумма сделки = выручка | Финансовая ошибка |
| 7 | Нет master оплаты | Ложный факт |
| 8 | Смешаны валюты | Неверная сумма |
| 9 | Игнорируются возвраты | Завышение |
| 10 | Нет attribution policy | Двойной credit |
| 11 | UTM считается причиной | Ложный вывод |
| 12 | Спам включён в лиды | Искажён CPL |
| 13 | Дубли клиентов | Искажён RFM/LTV |
| 14 | Название компании — ключ | Ошибочные joins |
| 15 | Текущий ответственный получает историю | Нечестный KPI |
| 16 | Нет timezone policy | Сдвиг периодов |
| 17 | Неполная pagination | Пропуски |
| 18 | ETL не идемпотентен | Дубли фактов |
| 19 | Нет checkpoint | Невосстановимость |
| 20 | Full reload без причины | Нагрузка |
| 21 | `SELECT *` | Нагрузка/PII |
| 22 | Cartesian join | Размножение сумм |
| 23 | Many-to-many без bridge | Double counting |
| 24 | Sum of ratios | Неверный итог |
| 25 | Среднее скрывает распределение | Ложная картина |
| 26 | ABC без периода/метрики | Невоспроизводимость |
| 27 | XYZ игнорирует сезонность | Неверный класс |
| 28 | RFM по сделкам, не покупкам | Ложный сегмент |
| 29 | LTV без горизонта | Непроверяемость |
| 30 | CAC без полных затрат | Занижение |
| 31 | ROMI смешан с ROAS | Ошибка решения |
| 32 | SLA без business calendar | Ложный breach |
| 33 | Активность = эффективность | Gaming |
| 34 | Dashboard без audience | Не используется |
| 35 | Слишком много KPI | Нет фокуса |
| 36 | Нет data dictionary | Разные смыслы |
| 37 | Нет lineage | Нельзя доказать цифру |
| 38 | Нет refresh SLA | Устаревшие данные |
| 39 | Нет quality gates | Плохие данные публикуются |
| 40 | Ошибка скрывается нулём | Ложное спокойствие |
| 41 | PII в BI без нужды | Утечка |
| 42 | Права detail слабее CRM | Раскрытие данных |
| 43 | Test/prod смешаны | Загрязнение |
| 44 | Реальные данные в Git | Утечка |
| 45 | BI строится до процесса | Нечего измерять |
| 46 | Стадии меняются без version | Сломана история |
| 47 | Closed-lost без причины | Нет диагностики |
| 48 | Нет reconciliation с ERP | Финансовые расхождения |
| 49 | Внешний SQL считается штатным | Неверная архитектура |
| 50 | Power BI предполагается доступным | Неподтверждённое решение |
| 51 | Тариф не проверен | Недоступная функция |
| 52 | Cloud/Box смешаны | Невыполнимое ТЗ |
| 53 | Нет нагрузочного теста | Медленный BI |
| 54 | Нет query limits | Деградация |
| 55 | Нет индексов DWH | Медленные запросы |
| 56 | Индексы Cloud предполагаются управляемыми | Ошибка платформы |
| 57 | Нет архивной политики | Рост данных |
| 58 | Нет changelog | Регрессия необъяснима |
| 59 | Нет приёмочных контрольных чисел | Нельзя принять |
| 60 | Нет плана исправления качества | Dashboard остаётся ложным |
| 61 | Drill-through раскрывает PII | Утечка |
| 62 | Прогноз выдан за факт | Ошибка управления |
| 63 | План меняется задним числом | Манипуляция KPI |
| 64 | Ручной Excel — скрытый master | Нет governance |

```text
Bad definition + bad data + attractive chart = confident wrong decision
```

## 50. Производительность BI

| Контроль | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Grain, select, filters, joins, refresh, concurrency | Aggregate marts, incremental load, query review | Оптимизировать только chart | Cloud limits не управляются клиентом |

```text
Source load → model size → query plan → dashboard latency
```

## 51. Ограничения облака

| Ограничение | Практика | Ошибка | Лучшая практика |
|---|---|---|---|
| Нет доступа к БД/ядру; тарифные BI/REST-ограничения | Published datasets/API only | Проектировать direct SQL к CRM DB | Проверять тариф и документацию |

```text
Cloud tenant → official BI/REST boundary → external analytics if needed
```

## 52. Возможности коробки

| Возможность | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Контроль инфраструктуры и серверной эксплуатации | Stage, backup, supported modules, DWH separation | Direct DB write/report on OLTP | Версия, нагрузка и обновления критичны |

```text
On-Premise operational DB → supported extraction → analytical store
```

Источник: [документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php).

## 53. Cloud vs On-Premise

| Возможность | Облако | Коробка | Комментарий |
|---|---|---|---|
| Standard/BI reports | По тарифу | По редакции/версии | Verify target |
| Datasets/SQL | По BI Конструктору | Version/config dependent | Contract first |
| DB access | Нет | Owner-controlled | Не нагружать OLTP аналитикой |
| ETL/DWH/Power BI | Внешняя интеграция | Внешняя/owned infrastructure | Не считать штатным без проекта |
| Scaling/logging | Vendor boundary | Owner responsibility | Operations required |

```text
Same KPI specification → different extraction/infrastructure paths
```

## 54. Архитектурные рекомендации

| Решение | Практика | Ошибка | Ограничение |
|---|---|---|---|
| Semantic layer + data contracts + quality gates | Start small, reconcile, version | Dashboard-first | Tool capabilities differ |

```text
Definitions → data model → quality → reports → adoption → governance
```

## 55. Практические кейсы

| Отрасль | Главные факты/KPI | Риск |
|---|---|---|
| EdTech | Lead→enrollment, payment, cohort retention, repeat course | Плательщик ≠ студент |
| B2B | Account, opportunity, cycle, proposal, payment, stakeholders | Длинный цикл и attribution |
| B2C | Response, conversion, average check, repeats, channel | Дубли контактов |
| Производство | Opportunity→order→production→shipment→payment | ERP — master исполнения |
| Услуги | Lead→brief→proposal→project→acceptance→payment | Сделка ≠ delivery project |
| Интернет-магазин | Visit→cart→order→payment→return→repeat | Order/payment/return masters |

```text
Industry process → master systems → facts → cohort/KPI → decision
```

## 56. Чек-лист BI архитектора

| Область | Готовность |
|---|---|
| Governance | Owner, glossary, source, grain, lineage |
| Model | Facts, dimensions, measures, history |
| Pipeline | Incremental, idempotent, monitored, reconciled |
| Security | PII, RBAC, environments, retention |
| Quality | Rules, thresholds, remediation |
| Delivery | Audience, SLA, performance, acceptance |

```text
Specify → model → build → test → reconcile → publish → govern
```

## 57. Чек-лист руководителя отдела продаж

| Проверка | Вопрос |
|---|---|
| План/факт | Один период, валюта и определение? |
| Pipeline/forecast | Есть aging, next step и accuracy? |
| Conversion/SLA | Когорты и события корректны? |
| Team | Ownership и портфели сопоставимы? |
| Sources/ROMI | Затраты и attribution подтверждены? |
| Repeats/cohorts | Клиенты дедуплицированы? |

```text
Signal → manager action → owner/deadline → next review
```

## 58. Чек-лист аналитика

| Этап | Проверка |
|---|---|
| Request | Decision, audience, definition |
| Data | Source, grain, keys, PII, quality |
| Calculation | Formula, window, currency, nulls |
| Validation | Control totals, sample, edge cases |
| Visual | Honest scale, context, limitations |
| Release | Version, owner, refresh, feedback |

```text
Question → SQL/model → tests → peer review → publish → monitor
```

## 59. Официальные источники

| Источник | Использование | Проверено в реестре |
|---|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | CRM/analytics product context | 2026-08-05 |
| [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) | Official datasets and BI capabilities | 2026-08-05 |
| [Bitrix24 REST API](https://apidocs.bitrix24.ru/) | Extraction/integration architecture | 2026-08-05 |
| [API reference](https://apidocs.bitrix24.ru/api-reference/) | Methods, fields, pagination | 2026-08-05 |
| [On-Premise docs](https://dev.1c-bitrix.ru/docs/portal.php) | Box-specific context | 2026-08-05 |

```text
Official source → verify tariff/version → metric contract → tested report
```

Сторонние источники не использованы. Power BI, внешний SQL/DWH и конкретные BI-функции не считаются доступными без отдельного подтверждения архитектуры и лицензирования.
