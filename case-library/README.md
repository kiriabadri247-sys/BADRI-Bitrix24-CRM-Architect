# Bitrix24 Architecture Case Library

Case Library — каталог обезличенных референсных архитектур для проектирования, пресейла, внедрения и аудита решений Bitrix24. Каждый кейс показывает инженерный способ декомпозиции отраслевой задачи, но не является готовой конфигурацией для копирования на рабочий портал.

> Все данные и примеры синтетические. Перед применением кейса обязательно проверяются Cloud/On-Premise, тариф или редакция, версия, регион, права, доступные приложения, существующие процессы, интеграции и ограничения целевого портала.

## 1. Назначение Case Library

Библиотека создана для повторного использования архитектурных решений без повторного изобретения базовых моделей CRM, Smart Processes, automation, integrations, testing and operations.

| Роль | Как использовать библиотеку | Результат |
|---|---|---|
| Solution Architect | Выбрать близкий сценарий, проверить границы систем, сущности, master data, права, ADR и риски | Черновик TO BE и список решений для Discovery |
| Пресейл | Быстро определить scope, зависимости, открытые вопросы, этапы и сложность | Проверяемая структура предложения без обещания неподтверждённых функций |
| CRM-консультант | Сопоставить AS IS клиента с референсным lifecycle и checklist | Карта разрывов и план адаптации |
| Business Analyst | Использовать process, data and rights models как источник вопросов | Полные требования и acceptance criteria |
| Delivery Lead | Взять testing, deployment, monitoring and rollback patterns | Управляемый запуск и сопровождение |
| Аудитор | Сравнить действующий портал с reference controls | Findings, risks and remediation backlog |

Case Library применяется:

- как навигация по типовым отраслевым процессам;
- как источник архитектурных альтернатив, а не догма;
- как checklist полноты проектного решения;
- как основа для workshop, assessment and estimation;
- как портфель повторно используемых ADR, test and operational patterns.

Не допускается перенос стадий, полей, роботов, прав или integrations без обследования действующего портала, проверки тарифа и анализа зависимостей.

## 2. Архитектурный подход

### Основные принципы

| Принцип | Практическое значение |
|---|---|
| CRM-first | Customer identity, commercial opportunity and relationship context проектируются вокруг CRM, но CRM не объявляется master всех данных |
| Процессный подход | Для каждой сущности определяются owner, lifecycle, states, transitions, exceptions, SLA, outcome and archive |
| Cloud / On-Premise | Возможности, расширения, эксплуатация и безопасность рассматриваются отдельно для каждого варианта |
| No-code first | Сначала штатные настройки, поля, карточки, роботы, триггеры, бизнес-процессы и Smart Processes |
| Integration by contract | Внешние системы подключаются через явные contracts, master ownership, stable IDs, idempotency, retry and reconciliation |
| Security by design | Least privilege, data minimization, synthetic tests, secret isolation and audit проектируются до запуска |
| Evidence-based delivery | Готовность подтверждается tests, UAT, smoke, reconciliation, monitoring and rollback evidence |

~~~text
Business Context
      |
      v
AS IS -> Requirements -> Entity and Process Model -> Rights and Data Ownership
                                                |
                                                v
Native Bitrix24 -> Automation -> Verified Apps/REST -> External Systems
                                                |
                                                v
Testing -> Deployment -> Monitoring -> Controlled Improvement
~~~

CRM-first не означает CRM-only. Для product, finance, billing, warehouse, medical, HR, document, telemetry and other specialized facts назначается профильная master-система. Bitrix24 хранит только необходимую рабочую проекцию.

Приоритет реализации:

1. штатные настройки CRM;
2. пользовательские поля и карточки;
3. роботы и триггеры;
4. штатные бизнес-процессы;
5. Smart Processes;
6. формы, открытые линии и стандартные коннекторы;
7. проверенные приложения Маркетплейса;
8. webhooks and REST API;
9. внешний сервис;
10. расширение On-Premise без изменения ядра.

## 3. Каталог отраслевых решений

| № | Отрасль / сценарий | Папка | Основные процессы |
|---:|---|---|---|
| 01 | EdTech | [01_EdTech](01_EdTech/README.md) | Лиды, продажи обучения, продукты, оплаты, повторные продажи |
| 02 | B2B Sales | [02_B2B_Sales](02_B2B_Sales/README.md) | Длинные продажи, ЛПР, КП, согласования, account management |
| 03 | Service Business | [03_Service_Business](03_Service_Business/README.md) | Заявки, услуги, проекты, SLA, повторное обслуживание |
| 04 | Real Estate | [04_Real_Estate](04_Real_Estate/README.md) | Объекты, показы, бронирования, сделки, договоры и оплаты |
| 05 | Manufacturing | [05_Manufacturing](05_Manufacturing/README.md) | Заказы, производство, снабжение, качество, отгрузка |
| 06 | E-commerce | [06_Ecommerce](06_Ecommerce/README.md) | Каталог, заказы, оплаты, доставка, возвраты, support |
| 07 | Logistics | [07_Logistics](07_Logistics/README.md) | Заявки, маршруты, рейсы, транспорт, доставка, претензии |
| 08 | Healthcare | [08_Healthcare](08_Healthcare/README.md) | Пациентский путь, записи, услуги, персонал, privacy |
| 09 | Finance | [09_Finance](09_Finance/README.md) | Финансовые продукты, заявки, approvals, risks, documents |
| 10 | Construction | [10_Construction](10_Construction/README.md) | Объекты, проекты, тендеры, сметы, подрядчики |
| 11 | Hospitality | [11_Hospitality](11_Hospitality/README.md) | Бронирования, номерной фонд, гостевой сервис, loyalty |
| 12 | Retail | [12_Retail](12_Retail/README.md) | Омниканальные продажи, магазины, loyalty, marketing, service |
| 13 | Automotive | [13_Automotive](13_Automotive/README.md) | Продажи автомобилей, тест-драйвы, сервис, гарантия, запчасти |
| 14 | Legal Services | [14_Legal_Services](14_Legal_Services/README.md) | Клиенты, matters, договоры, документы, lawyers, billing |
| 15 | Insurance | [15_Insurance](15_Insurance/README.md) | Продукты, полисы, claims, support, renewals |
| 16 | Telecom | [16_Telecom](16_Telecom/README.md) | Тарифы, услуги, оборудование, support, customer lifecycle |
| 17 | Travel and Tourism | [17_Travel_Tourism](17_Travel_Tourism/README.md) | Туры, бронирования, партнёры, support, repeat sales |
| 18 | Franchise | [18_Franchise](18_Franchise/README.md) | Продажа франшизы, партнёры, точки, обучение, quality |
| 19 | Agriculture | [19_Agriculture](19_Agriculture/README.md) | Продажи, производство, поставщики, логистика, качество |
| 20 | Energy and Utilities | [20_Energy_Utilities](20_Energy_Utilities/README.md) | Подключения, обращения, инфраструктура, incidents, support |
| 21 | Government | [21_Government](21_Government/README.md) | Обращения граждан, документы, поручения, проекты, контроль |
| 22 | Media and Entertainment | [22_Media_Entertainment](22_Media_Entertainment/README.md) | Реклама, медиапроекты, контент, мероприятия, партнёры |
| 23 | Consulting | [23_Consulting](23_Consulting/README.md) | Продажи услуг, проекты, эксперты, загрузка, deliverables |
| 24 | Recruitment | [24_Recruitment](24_Recruitment/README.md) | Клиенты, вакансии, кандидаты, интервью, подборы |
| 25 | IT and SaaS | [25_IT_SaaS](25_IT_SaaS/README.md) | Подписки, onboarding, Customer Success, support, renewals |

Каждый кейс содержит только референсную модель. Фактические сущности, названия стадий, поля, роли, automation and integrations утверждаются отдельно.

## 4. Как использовать кейсы

~~~text
Discovery
   |
   v
Выбор отраслевого сценария
   |
   v
Адаптация архитектуры
   |
   v
Проектирование CRM
   |
   v
Настройка и интеграции
   |
   v
Тестирование -> Запуск -> Monitoring
~~~

| Шаг | Действия | Выход |
|---:|---|---|
| 1. Discovery | Зафиксировать AS IS, stakeholders, цели, боли, данные, объёмы, тариф, права and systems | Подтверждённые факты и open questions |
| 2. Выбор кейса | Найти ближайший отраслевой и функциональный сценарий | Reference case и применимые patterns |
| 3. Gap analysis | Сопоставить процессы клиента с reference model | Fit/gap matrix |
| 4. Адаптация | Выбрать entities, masters, lifecycle, exceptions, rights and NFR | Draft TO BE |
| 5. Проектирование | Подготовить fields, stages, automation, integrations, analytics and ADR | Approved architecture |
| 6. Реализация | Настроить сначала изолированный test scope | Версионируемая конфигурация |
| 7. Проверка | Выполнить positive, negative, repeat, rights, failure and recovery tests | Test evidence and UAT |
| 8. Запуск | Применить cutover, smoke, reconciliation and rollback plan | Controlled production release |
| 9. Сопровождение | Контролировать business and technical signals | Incident/change backlog |

Если ни один кейс не покрывает задачу полностью, используются несколько кейсов как источники отдельных patterns. Результатом всё равно должна быть единая непротиворечивая архитектура, а не механическая склейка.

## 5. Матрица покрытия процессов

| Процесс | Какие кейсы покрывают |
|---|---|
| Продажи | Все кейсы; особенно 01, 02, 04, 06, 12, 13, 18, 22, 23, 25 |
| Сервис | 03, 07, 08, 11, 13, 16, 20, 21, 25 |
| Проекты | 03, 05, 10, 14, 19, 21, 22, 23 |
| Документы | 02, 04, 05, 09, 10, 14, 20, 21, 22, 23, 24 |
| Интеграции | Все 25 кейсов; глубина определяется master-системами отрасли |
| Поддержка | 03, 06, 08, 11, 13, 15, 16, 17, 20, 21, 25 |
| Подписки | 01, 15, 16, 25 |
| Бронирования | 04, 11, 13, 17 |
| Производство | 05, 10, 19, 22 |
| Обучение | 01, 18, 23; частично onboarding в 24 и 25 |

Матрица показывает наличие pattern, а не подтверждение готовой функции на конкретном тарифе. Для смежных процессов можно комбинировать кейсы: например, B2B Sales + Manufacturing или SaaS + Consulting.

## 6. Матрица CRM объектов

| Объект | Использование |
|---|---|
| Лид | Неквалифицированный входящий спрос, когда lead-based model обоснована; решение фиксируется ADR |
| Контакт | Физическое лицо или представитель организации; не заменяет отраслевую сущность вроде кандидата или пациента |
| Компания | Юридическое лицо, B2B-клиент, партнёр или контрагент; разные направления одного клиента не создают дубли без основания |
| Сделка | Отдельная коммерческая opportunity с суммой, owner, lifecycle and outcome |
| Smart Process | Самостоятельная сущность с собственными стадиями, правами, связями и automation |
| Задача | Исполнимая работа с ответственным, сроком, входом и ожидаемым результатом |
| Документ | Управляемая версия или защищённая ссылка на authoritative DMS/ЭДО; не неуправляемая копия |
| Роботы | Локальные детерминированные действия, reminders, tasks and transitions с защитой от повторного запуска |
| Бизнес-процессы | Многошаговые маршруты, approvals and stateful workflows с owner, errors and escalation |

~~~text
Contact / Company
       |
       +-> Lead qualification -> Deal
       |                         |
       |                         +-> Smart Process lifecycle
       |                                  |
       +-> Activities / Tasks             +-> Robots / Business Processes
                                                  |
                                                  +-> Documents / Integrations
~~~

Объект выбирается по бизнес-смыслу, lifecycle, владельцу, правам, связям, аналитическому grain and archive policy. Экономия сущностей не является достаточным архитектурным основанием.

## 7. Архитектурные ограничения

| Область | Cloud | On-Premise | Обязательная проверка |
|---|---|---|---|
| Доступ к платформе | Нет доступа к ядру, server PHP and DB | Доступ управляется заказчиком | Edition, version and support policy |
| Расширение | Settings, automation, apps, REST and external services | Дополнительно поддерживаемые modules/extensions | Не изменять ядро |
| Эксплуатация | Платформенный слой управляется провайдером | OS, web server, PHP, DB, files, cron, cache, backup and updates у заказчика | RACI and monitoring |
| Масштабирование | Tariff/REST/product limits and external integration capacity | Application and infrastructure capacity | Load profile and limits |
| Безопасность | Customer controls users, rights, config, data and integrations | Дополнительно infrastructure hardening | Threat model and negative tests |
| Backup/rollback | Доступные штатные механизмы и configuration rollback | Full infrastructure/data recovery design | Tested recovery |

Общие ограничения:

- доступность CRM, Smart Processes, robots, business processes, BI, telephony, REST methods and apps проверяется по официальной документации и на целевом портале;
- тарифные, региональные, версионные и правовые ограничения не переносятся из кейса как факт;
- REST methods, scopes, fields, limits and events не угадываются;
- приложение Маркетплейса оценивается по поставщику, permissions, data flow, support and exit plan;
- массовые операции используют dry-run, test sample, idempotency, batch report and rollback;
- секреты, webhook URL, tokens, real domains and client exports не хранятся в Git;
- внешние master-системы не подменяются удобными CRM-полями.

## 8. Методология работы архитектора

~~~text
Discovery -> Architecture -> Implementation -> Testing
 -> Deployment -> Monitoring -> Improvement
        ^                                 |
        +--------- Change Control --------+
~~~

| Этап | Основные действия | Gate |
|---|---|---|
| Discovery | AS IS, stakeholders, goals, scope, volumes, data quality, systems, risks and constraints | Facts and open questions approved |
| Architecture | TO BE, entity model, masters, rights, integrations, NFR, ADR and rollback concept | Architecture review |
| Implementation | Isolated configuration, registers, mappings, error handling and documentation | Configuration review |
| Testing | Positive, negative, repeat, permissions, integration failure, migration, recovery and UAT | Test evidence/sign-off |
| Deployment | Backup/export, cutover, communication, smoke, reconciliation and rollback | Go/no-go |
| Monitoring | Business SLA, stuck objects, integration health, data quality, access and capacity | Operational acceptance |
| Improvement | RCA, metrics, backlog, change impact and regression | Approved change |

Минимальные результаты архитектора:

1. AS IS and TO BE;
2. entity and relationship model;
3. lifecycle/stage register;
4. field and master-data register;
5. rights matrix;
6. automation register;
7. integration map and contracts;
8. analytics metric contracts;
9. ADR and risk register;
10. test, deployment, rollback, monitoring and support plans.

## 9. Связанные документы

| Документ | Ссылка | Статус и назначение |
|---|---|---|
| PATTERNS.md | [Архитектурные паттерны](PATTERNS.md) | Каталог повторяемых CRM, Smart Process, automation, integration, data, security and delivery patterns |
| DISCOVERY_TEMPLATE.md | [Шаблон Discovery-аудита](DISCOVERY_TEMPLATE.md) | Универсальный шаблон обследования AS IS, требований, данных, прав, интеграций и рисков |
| ARCHITECTURE_TEMPLATE.md | [Шаблон архитектурного проектирования](ARCHITECTURE_TEMPLATE.md) | Универсальный шаблон TO BE, CRM architecture, ADR, testing, deployment and monitoring |
| Ecosystem Architecture | [Целевая экосистема](../docs/ECOSYSTEM_ARCHITECTURE.md) | Уровни и roadmap развития репозитория |
| Official Sources | [Реестр источников](../knowledge/SOURCES.md) | Единственный разрешённый реестр внешних официальных источников |
| Testing | [База знаний по тестированию](../knowledge/testing/README.md) | QA strategy, checks and acceptance |
| Deployment | [База знаний по deployment](../knowledge/deployment/README.md) | Release, rollback and stabilization |
| Monitoring | [База знаний по monitoring](../knowledge/monitoring/README.md) | Operational signals and response |

Паттерны выбираются после Discovery, а утверждённые требования переносятся в архитектурный шаблон с трассировкой до ADR, реализации, тестов и эксплуатационного контроля.

## 10. Статистика библиотеки

| Показатель | Значение | Метод подсчёта |
|---|---:|---|
| Количество кейсов | 25 | Каталоги 01–25 с README.md |
| Количество отраслей и solution domains | 25 | Один основной домен на каталог |
| Количество архитектурных решений | 191 | ADR-записи во всех существующих кейсах |
| Нумерованных архитектурных разделов | 542 | Разделы формата ## N. во всех кейсах |
| Таблиц | 580 | Markdown table separator blocks |
| Markdown-схем | 272 | Fenced text diagrams |
| Дата обновления | 2026-08-06 | Актуализировано после создания Discovery и Architecture templates |

Статистика отражает текущее состояние файлов и пересчитывается при добавлении или структурном изменении кейсов. Число ADR зависит от единообразия их табличной маркировки.

### Правила сопровождения каталога

- новый кейс получает следующий двухзначный ID и отдельный README.md;
- каталог обновляется в том же изменении, что и новый кейс, после проверки ссылок;
- кейс использует только synthetic data and registered official sources;
- изменение общего pattern не переписывает автоматически уже принятые ADR;
- устаревший кейс помечается явно, а не удаляется без impact analysis;
- статистика пересчитывается командами read-only перед публикацией.
