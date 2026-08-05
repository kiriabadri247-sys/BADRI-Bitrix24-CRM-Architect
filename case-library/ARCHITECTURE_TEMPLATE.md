# Bitrix24 Solution Architecture Template

Универсальный шаблон архитектурного документа, создаваемого после завершённого Discovery и до конфигурации, разработки, миграции и запуска решения Bitrix24.

> **Статус:** шаблон не подтверждает наличие функций на конкретном портале. CRM, Smart Processes, роботы, триггеры, бизнес-процессы, документы, BI, телефония, REST и приложения проверяются по официальной документации и на целевом Cloud/On-Premise окружении с учётом тарифа или редакции, версии, региона и прав.

> **Безопасность:** в Git хранится только пустой шаблон или полностью синтетический пример. Заполненный документ с названием клиента, ФИО, телефонами, email, доменами, CRM ID, коммерческими условиями, схемами инфраструктуры, выгрузками, паролями, токенами или webhook URL хранится только в разрешённой защищённой системе.

## Правила применения

| Маркер | Значение |
|---|---|
| **Факт** | Подтверждён Discovery evidence, настройкой целевого портала или официальным источником |
| **Требование** | Имеет ID, владельца, приоритет и acceptance criteria |
| **Решение архитектора** | Зафиксировано в ADR с альтернативами и последствиями |
| **Требует проверки** | Зависит от тарифа, редакции, версии, региона, прав, приложения или внешней системы |
| **Ограничение** | Условие, которое архитектура обязана соблюдать |
| **Вне scope** | Осознанно исключено из текущего релиза и сохранено в backlog |

Каждый компонент трассируется как `Discovery fact → Requirement → Architecture decision → Implementation item → Test → Operational control`.

---

## 1. Назначение документа

### Цель архитектуры

Архитектура задаёт проверяемую целевую модель решения: границы, CRM-объекты, процессы, данные, права, автоматизацию, интеграции, аналитику, эксплуатацию, риски и план перехода. Документ является контрактом проектирования, а не инструкцией по немедленному изменению production.

### Участники

| Роль | Ответственность | Approves |
|---|---|---|
| Sponsor | Цели, scope, бюджет, остаточный бизнес-риск | Project charter и roadmap |
| Solution Architect | Целостность решения, ADR, NFR, границы систем | Architecture baseline |
| Business Analyst | Требования, процессы, acceptance criteria, traceability | Requirements baseline |
| Process Owner | Процесс, роли, SLA, исключения | Process design |
| Data Owner | Семантика, качество, master data, retention | Data model и migration rules |
| Security Owner | Классификация, доступ, audit, risk treatment | Security architecture |
| Technical Owner | Платформа, интеграции, эксплуатация | Technical design и runbooks |
| QA/UAT Lead | Стратегия проверки и evidence | Test/UAT results |
| Delivery Lead | Релизы, ресурсы, зависимости, cutover | Implementation plan |

~~~text
Discovery
   ↓
Architecture
   ↓
Implementation
   ↓
Support
~~~

**Architecture gate:** нет блокирующих вопросов без владельца и срока; Must-требования трассируются до решения и теста; критические риски приняты уполномоченными владельцами.

---

## 2. Project Overview

> Заполнять идентифицирующие сведения только в защищённой рабочей копии.

| Параметр | Значение |
|---|---|
| Клиент | `[Заполнить в защищённой системе]` |
| Отрасль | `[Заполнить]` |
| Цели | `[BG/REQ IDs]` |
| Scope | `[Процессы, подразделения, системы]` |
| Out of scope | `[Явные исключения]` |
| Масштаб | `[Объёмы, география, рост]` |
| Пользователи | `[Количество и роли]` |
| Версия Bitrix24 | `[Тариф/редакция и версия; требует проверки]` |
| Cloud / On-Premise | `[Выбрать и подтвердить]` |
| Регион | `[Заполнить]` |
| Test environment | `[Портал/воронка/стенд]` |
| Target release | `[Дата/волна]` |
| Допустимый downtime | `[Заполнить]` |
| Architecture owner | `[Роль]` |
| Статус документа | `[Draft/Review/Approved/Superseded]` |
| Версия документа | `[0.1]` |

| Источник | Версия/дата | Что подтверждает | Владелец |
|---|---|---|---|
| Discovery package | `[Версия]` | `[Facts/requirements/risks]` | `[Роль]` |
| Official source | `[URL из SOURCES.md, дата]` | `[Применимость функции]` | `[Роль]` |
| Portal evidence | `[Ссылка в защищённой системе]` | `[Настройка/ограничение]` | `[Роль]` |

---

## 3. Business Context

### Бизнес-модель

`[Описать продукт/услугу, клиента, каналы, модель выручки, единицу продажи и жизненный цикл без идентифицирующих данных.]`

### Процессы и подразделения

| Процесс | Подразделения | Trigger | Результат | KPI/SLA | Владелец |
|---|---|---|---|---|---|
| `[Заполнить]` | `[Роли/отделы]` | `[Событие]` | `[Измеримый результат]` | `[Метрика]` | `[Роль]` |

### Ключевые проблемы

| Problem ID | Наблюдаемый симптом | Root cause | Влияние | Связанная цель | Requirement IDs |
|---|---|---|---|---|---|
| PP-001 | `[Заполнить]` | `[Подтверждена/гипотеза]` | `[Деньги/время/качество/риск]` | `[BG-ID]` | `[REQ-IDs]` |

### Архитектурные драйверы

- `[Business value]`;
- `[Data integrity]`;
- `[Security/compliance]`;
- `[Continuity and recovery]`;
- `[Performance and scale]`;
- `[Maintainability and operability]`.

---

## 4. AS IS Architecture

| Область | Текущее состояние | Проблема |
|---|---|---|
| Business | `[Модель и ограничения]` | `[Problem IDs]` |
| Processes | `[Фактические маршруты]` | `[Узкие места/исключения]` |
| CRM | `[Объекты, стадии, поля]` | `[Дубли/неоднозначность]` |
| Automation | `[Роботы/BP/tasks]` | `[Циклы/ошибки/owner]` |
| Integrations | `[Системы и обмены]` | `[Конфликты/нет мониторинга]` |
| Data | `[Masters, объёмы, качество]` | `[Пробелы/дубли]` |
| Rights | `[Роли и фактический доступ]` | `[Избыточность/недоступность]` |
| Analytics | `[Reports/KPI/sources]` | `[Недостоверность]` |
| Operations | `[Support/change/backup]` | `[Нет SLA/runbook]` |

~~~text
Channels → Current CRM → Manual operations → External systems
              │                  │                   │
           duplicates         delays             reconciliation gaps
~~~

### Ограничения AS IS

| ID | Ограничение | Тип | Влияние | Можно изменить | Решение/эскалация |
|---|---|---|---|---|---|
| CON-001 | `[Заполнить]` | `[Business/technical/legal/time]` | `[Заполнить]` | `[Да/нет]` | `[ADR/measure]` |

---

## 5. TO BE Architecture

### Целевая модель

~~~text
Бизнес
   ↓
Процессы
   ↓
CRM
   ↓
Автоматизация
   ↓
Интеграции
   ↓
Аналитика
~~~

~~~text
Каналы → Bitrix24 CRM / Smart Processes → Исполнение
                 │             │             │
                 ├─ Automation ├─ Documents  ├─ Tasks
                 │             │             │
                 └──── Integration Layer ────┘
                              │
                    ERP / 1C / DMS / BI
                              │
                     Monitoring + Support
~~~

### Принципы

1. CRM-first для клиентского контекста и коммерческого процесса.
2. Процесс и ответственность проектируются до автоматизации.
3. Для каждого класса данных назначается одна master-система.
4. Штатные инструменты рассматриваются до приложений и разработки.
5. Интеграции проектируются с идемпотентностью, наблюдаемостью и восстановлением.
6. Least privilege и data minimization применяются по умолчанию.
7. Массовые изменения имеют dry-run, test sample, reconciliation и rollback.
8. Cloud и On-Premise рассматриваются как разные эксплуатационные варианты.

| Layer | Компонент | Responsibility | Owner | Requirement IDs | ADR | NFR |
|---|---|---|---|---|---|---|
| CRM | `[Заполнить]` | `[Назначение]` | `[Роль]` | `[REQ]` | `[ADR]` | `[NFR]` |

---

## 6. CRM Architecture

### Лиды

- **Назначение:** `[Неквалифицированное обращение либо не используется]`.
- **Источники:** `[Каналы и правила attribution]`.
- **Обработка:** `[Owner, SLA, qualification criteria, conversion outcome]`.
- **Не использовать:** `[Условия прямого создания сделки]`.
- **Проверка:** режим лидов, права, поля и автоматизация подтверждаются на портале.

### Контакты

- **Модель клиента:** физическое лицо и его роли в отношениях.
- **Связи:** `[Компания, сделки, Smart Processes, активности]`.
- **Идентификация:** `[Matching keys; не использовать ФИО как единственный ключ]`.
- **Ownership:** `[Роль/правило]`.

### Компании

- **B2B-модель:** юридическое лицо/организация, единый профиль и отношения.
- **Сегментация:** `[Справочники и источник]`.
- **Связи:** `[Контакты, сделки, договоры, сервисные объекты]`.
- **Правило:** не создавать отдельные компании по направлениям продаж без бизнес-обоснования.

### Сделки

| Воронка | Назначение | Entry | Exit | Owner | KPI |
|---|---|---|---|---|---|
| `[Заполнить]` | `[Коммерческая возможность]` | `[Критерий]` | `[Критерий]` | `[Роль]` | `[Метрика]` |

| Стадия | Цель | Обязательные данные | Допустимый переход | Действия | Автоматизация | SLA |
|---|---|---|---|---|---|---|
| `[Заполнить]` | `[Результат стадии]` | `[Поля]` | `[Следующие стадии]` | `[Роли]` | `[Кандидат]` | `[Срок]` |

### Пользовательские поля

| Поле | Тип | Назначение |
|---|---|---|
| `[Business name / internal code]` | `[Подтверждённый тип]` | `[Решение/аналитика/integration]` |

| Сущность | Код | Множественность | Обязательность | Values/range | Owner | Права | Dependencies | Test |
|---|---|---|---|---|---|---|---|---|
| `[Объект]` | `[UF_* placeholder]` | `[Да/нет]` | `[Стадия/условие]` | `[Справочник]` | `[Роль]` | `[Роли]` | `[Robot/API/BI/doc]` | `[TC-ID]` |

### CRM traceability

| Object | Source creation | Owner | Duplicate rule | Archive/delete rule | Requirement | Test |
|---|---|---|---|---|---|---|
| `[Lead/contact/company/deal]` | `[Канал/процесс]` | `[Роль]` | `[Правило]` | `[Политика]` | `[REQ-ID]` | `[TC-ID]` |

---

## 7. Smart Process Architecture

Smart Process выбирается для самостоятельной сущности с собственным lifecycle, ролями, связями и аналитической ценностью. Конкретные настройки, поля, стадии, права и автоматизация подтверждаются на целевом портале.

### Реестр Smart Processes

| Название | Цель | Поля | Статусы | Связи | Автоматизация |
|---|---|---|---|---|---|
| Объект | `[Управляемый актив/предмет]` | `[Минимальный набор]` | `[Lifecycle]` | `[Клиент/сделка/сервис]` | `[Кандидат]` |
| Договор | `[Lifecycle договора]` | `[Номер без реальных значений, тип, даты, статус]` | `[Draft→active→closed]` | `[Компания/сделка]` | `[Approval/reminder]` |
| Сервис | `[Сервисное обращение]` | `[Type/priority/SLA]` | `[Lifecycle]` | `[Клиент/объект]` | `[Assignment/escalation]` |
| Проект | `[Delivery lifecycle]` | `[Scope/dates/owner]` | `[Lifecycle]` | `[Сделка/задачи]` | `[Milestones/control]` |
| Заявка | `[Запрос на действие]` | `[Source/category/priority]` | `[Lifecycle]` | `[Contact/company]` | `[Routing/notification]` |

> Примеры выше являются архетипами, а не требованием создать пять процессов.

### Карточка решения по каждому Smart Process

| Атрибут | Значение |
|---|---|
| Название | `[Заполнить]` |
| Цель | `[Business outcome]` |
| Почему не CRM/задача/внешняя система | `[Обоснование]` |
| Entity type ID | `[Назначается системой; не придумывать]` |
| Categories/pipelines | `[Требует проверки]` |
| Stages | `[Entry/exit criteria]` |
| Fields | `[Field registry reference]` |
| Relations | `[Objects and cardinality]` |
| Rights | `[Role matrix reference]` |
| Automation | `[AUT IDs]` |
| REST | `[Точные методы проверяются в API reference]` |
| Volume/retention | `[Оценка/политика]` |
| Analytics | `[Metrics/source]` |
| Tests | `[TC IDs]` |

~~~text
Client → Deal → Smart Process item → Task/Document
                  │
                  └→ Related item / External master
~~~

---

## 8. Business Process Architecture

| Process ID | Цель | Trigger | Steps | Conditions | Approvals | Roles | Result | Error path |
|---|---|---|---|---|---|---|---|---|
| BP-001 | `[Результат]` | `[Событие]` | `[Ссылка на flow]` | `[Правила]` | `[Матрица]` | `[RACI]` | `[Выход]` | `[Escalation]` |

~~~text
Trigger → Validate → Execute → Approve? → Complete
             │          │          │
             └─ Reject  └─ Retry   └─ Return / Escalate
~~~

### Обязательная спецификация

- сущность и событие запуска;
- входные данные и валидация;
- шаги, ветвления, ожидания и сроки;
- исполнители, заместители и согласующие;
- результат и изменения данных;
- повторный запуск и защита от дублей;
- обработка ошибки, эскалация и ручное восстановление;
- история/audit evidence;
- позитивные, негативные и повторные тесты;
- применимость конкретных действий в Cloud/On-Premise.

---

## 9. Automation Architecture

### Роботы

Кандидаты: уведомления, постановка задач и другие подтверждённые действия. Для каждого робота фиксируются событие, условие, задержка, исполнитель, side effects и error owner.

### Триггеры

Кандидаты: подтверждённые события и изменения, которые переводят элемент или запускают действие. Точные доступные триггеры проверяются по официальной документации и на портале.

| ID | Событие | Автоматизация | Результат |
|---|---|---|---|
| AUT-001 | `[Подтверждённое событие]` | `[Robot/trigger/BP/task]` | `[Измеримый результат]` |

| Entity/stage | Condition | Delay/calendar | Actor | Idempotency | Error handling | Monitoring | Test |
|---|---|---|---|---|---|---|---|
| `[Заполнить]` | `[Условие]` | `[Правило]` | `[Роль]` | `[Ключ/guard]` | `[Retry/escalation]` | `[Metric]` | `[TC-ID]` |

~~~text
Событие → Автоматизация → Результат
              │
              ├→ Guard against repeat
              └→ Error → Log → Owner → Recovery
~~~

---

## 10. Rights Architecture

| Роль | Объект | Просмотр | Изменение | Полное управление |
|---|---|---|---|---|
| `[Заполнить]` | `[CRM/Smart Process/task/document/BI]` | `[Own/department/all/none]` | `[Scope]` | `[Да/нет + обоснование]` |

| Роль | Create | Read | Update | Delete | Export | Settings | Audit evidence |
|---|---|---|---|---|---|---|---|
| `[Роль]` | `[Scope]` | `[Scope]` | `[Scope]` | `[Scope]` | `[Да/нет]` | `[Да/нет]` | `[TC-ID]` |

### Правила

- отделы и руководители отражают подтверждённую ответственность;
- доступ выдаётся роли, а не отдельному человеку без исключительного обоснования;
- совмещение ролей и наследование проверяются эффективными правами;
- экспорт, удаление, массовые операции и настройки ограничиваются отдельно;
- интеграционные учётные записи имеют минимально необходимые права;
- joiner/mover/leaver и временный доступ имеют owner, срок и аудит;
- позитивные и негативные сценарии проверяются тестовыми пользователями.

---

## 11. Integration Architecture

| Система | Данные | Направление | Метод | Ответственный |
|---|---|---|---|---|
| 1С | `[Объекты/поля]` | `[→/←/↔]` | `[Штатная интеграция/app/REST/file; подтвердить]` | `[Business/technical roles]` |
| ERP | `[Объекты/поля]` | `[→/←/↔]` | `[Подтвердить]` | `[Роли]` |
| Телефония | `[Calls/events/records]` | `[→/←]` | `[Подтверждённый вариант]` | `[Роли]` |
| Сайт | `[Forms/orders/sources]` | `[→/←]` | `[Подтвердить]` | `[Роли]` |
| BI | `[Datasets/metrics]` | `[→]` | `[Подтвердить]` | `[Роли]` |
| External | `[Заполнить]` | `[→/←/↔]` | `[REST/webhook/OAuth/app]` | `[Роли]` |

REST, webhook и OAuth являются разными интеграционными механизмами. Точные методы, события, scopes, авторизация и лимиты не предполагаются — они фиксируются после проверки официального API reference.

| INT ID | Master | Contract/version | Key | Frequency | Volume | Timeout | Retry/idempotency | Reconciliation | Security | Runbook |
|---|---|---|---|---|---:|---|---|---|---|---|
| INT-001 | `[System]` | `[Reference]` | `[External ID]` | `[Sync/async/batch]` | `[Estimate]` | `[Value]` | `[Policy]` | `[Rule]` | `[Scopes/secret ref]` | `[Link]` |

~~~text
External system → Auth → Validate → Map → Bitrix24
       ↑                  │        │         ↓
   reconciliation ← error store ← retry ← response/event
~~~

### Integration Definition of Ready

- master и conflict resolution согласованы;
- контракт, scope, limits и payload подтверждены;
- секреты вынесены в защищённое хранилище;
- PII минимизированы и не попадают в логи;
- предусмотрены timeout, bounded retry, idempotency и deduplication;
- есть reconciliation, replay, alert и runbook;
- выполнены load estimate и negative tests.

---

## 12. Data Architecture

### Источники и владельцы

| Domain | Master system | Data owner | Consumer | Key | Quality rule | Retention |
|---|---|---|---|---|---|---|
| Client | `[System]` | `[Роль]` | `[Systems]` | `[Stable ID]` | `[Rules]` | `[Policy]` |
| Product | `[System]` | `[Роль]` | `[Systems]` | `[Stable ID]` | `[Rules]` | `[Policy]` |
| Price | `[System]` | `[Роль]` | `[Systems]` | `[Stable ID]` | `[Rules]` | `[Policy]` |
| Order/deal | `[System]` | `[Роль]` | `[Systems]` | `[Stable ID]` | `[Rules]` | `[Policy]` |
| Payment | `[System]` | `[Роль]` | `[Systems]` | `[Stable ID]` | `[Rules]` | `[Policy]` |
| Document | `[System]` | `[Роль]` | `[Systems]` | `[Stable ID]` | `[Rules]` | `[Policy]` |

### Data Flow Diagram

~~~text
[Source]
   │ extract
   ↓
[Validation] → [Reject/Hold + owner]
   │ valid
   ↓
[Transform + Mapping] → [Bitrix24]
                              │
                              ↓
                    [Reconciliation report]
~~~

### Mapping и качество

| Source field | Target object/field | Type | Transform | Validation | Default | Error action | Owner |
|---|---|---|---|---|---|---|---|
| `[Field]` | `[Object.field]` | `[Type]` | `[Rule]` | `[Rule]` | `[Safe placeholder]` | `[Reject/hold/fix]` | `[Роль]` |

| Metric | Formula | Baseline | Threshold | Owner | Control |
|---|---|---:|---:|---|---|
| Completeness | `[Defined formula]` | `[Value]` | `[Target]` | `[Роль]` | `[Report/test]` |
| Duplicates | `[Defined formula]` | `[Value]` | `[Target]` | `[Роль]` | `[Report/test]` |
| Referential integrity | `[Defined formula]` | `[Value]` | `[Target]` | `[Роль]` | `[Report/test]` |

---

## 13. Analytics Architecture

### Уровни отчётности

| Уровень | Audience | Purpose | Frequency | Source | Access |
|---|---|---|---|---|---|
| Операционный | `[Исполнители]` | Очереди, SLA, ошибки, next actions | `[Real-time/daily]` | `[Dataset]` | `[Role]` |
| Управленческий | `[Руководители]` | Воронка, план/факт, производительность | `[Daily/weekly]` | `[Dataset]` | `[Role]` |
| Стратегический | `[Leadership]` | Тренды, unit economics, portfolio | `[Monthly]` | `[DWH/BI]` | `[Role]` |

### Метрики

| Metric | Definition | Grain | Filters | Source | Owner | Quality limitation |
|---|---|---|---|---|---|---|
| Продажи | `[Формула]` | `[Deal/order]` | `[Period/status]` | `[System]` | `[Роль]` | `[Known issue]` |
| Конверсия | `[Numerator/denominator]` | `[Cohort]` | `[Period/pipeline]` | `[System]` | `[Роль]` | `[Known issue]` |
| SLA | `[Start/stop/calendar]` | `[Item]` | `[Type/priority]` | `[System]` | `[Роль]` | `[Known issue]` |
| Эффективность | `[Outcome/input]` | `[User/team]` | `[Period]` | `[System]` | `[Роль]` | `[Known issue]` |

**Ограничение:** доступность BI Конструктора, наборов данных и внешней BI-интеграции зависит от тарифа, версии и источника. Метрика не утверждается до проверки data quality и единой формулы.

---

## 14. Security Architecture

### Общие контроли

| Control | Scope | Owner | Evidence | Test | Frequency |
|---|---|---|---|---|---|
| Least privilege | Users/roles/integrations | `[Роль]` | `[Access matrix]` | `[Negative TC]` | `[Review cycle]` |
| Secret management | Tokens/webhooks/keys | `[Роль]` | `[Secret reference]` | `[Rotation test]` | `[Cycle]` |
| Data minimization | Fields/logs/exports | `[Роль]` | `[Data inventory]` | `[Inspection]` | `[Cycle]` |
| Auditability | Critical changes/events | `[Роль]` | `[Available logs/history]` | `[Review]` | `[Cycle]` |
| Backup/recovery | Configuration/data | `[Роль]` | `[Report]` | `[Restore exercise]` | `[Cycle]` |

### Cloud

- нет доступа к ядру, серверному PHP-коду и базе данных;
- customer controls пользователей, роли, CRM-настройки, приложения, integrations и данные в пределах доступных функций;
- тариф, регион, доступные security controls и audit evidence проверяются на портале и по официальной документации;
- секреты не хранятся в полях CRM, документах, коде или Git;
- vendor/platform controls не приписываются клиенту без подтверждения.

### On-Premise

- дополнительно проектируются ОС, web server, PHP, DB, network, TLS, firewall, file permissions, cron/agents, cache, logs, backup и patching;
- фиксируются версия продукта и модулей, topology и custom modules;
- ядро продукта не изменяется;
- административный и инфраструктурный доступ разделяются;
- restore, update и security tests выполняются на изолированном окружении до production.

### Threat and access review

| Asset | Threat | Existing control | Gap | Treatment | Owner | Test |
|---|---|---|---|---|---|---|
| `[Data/process/integration]` | `[Threat]` | `[Control]` | `[Gap]` | `[Measure]` | `[Роль]` | `[TC-ID]` |

---

## 15. Deployment Plan

| Этап | Цель | Результат | Ответственный |
|---|---|---|---|
| 1. Подготовка | Зафиксировать baseline, scope, backup, access и окно | Approved release package | `[Release lead]` |
| 2. Настройка | Выполнить изолированную конфигурацию | Reviewed configuration | `[Implementation lead]` |
| 3. Интеграции | Развернуть contracts, secrets, monitoring | Tested integrations | `[Integration lead]` |
| 4. Тестирование | Smoke, functional, integration, security, UAT | Signed evidence | `[QA/UAT lead]` |
| 5. Обучение | Подготовить роли, инструкции и support | Ready users/support | `[Change lead]` |
| 6. Запуск | Выполнить cutover, smoke и hypercare | Operational solution | `[Release lead]` |

| Activity | Environment | Prerequisite | Window | Validation | Rollback trigger | Owner |
|---|---|---|---|---|---|---|
| `[Заполнить]` | `[Test/Pilot/Prod]` | `[Gate]` | `[Time]` | `[Check]` | `[Condition]` | `[Роль]` |

~~~text
Prepare → Change freeze → Deploy → Smoke → Go/No-Go → Hypercare
                                  │
                                  └→ Rollback → Validate → Report
~~~

---

## 16. Testing Strategy

| Test | Проверка | Результат |
|---|---|---|
| Smoke Testing | Критический путь после изменения | `[Pass/fail evidence]` |
| Functional Testing | Requirement и business rules | `[Expected vs actual]` |
| Integration Testing | Contract, auth, mapping, retry, errors | `[Evidence/reconciliation]` |
| UAT | Пользовательский outcome и acceptance criteria | `[Business sign-off]` |

| TC ID | Requirement | Preconditions | Synthetic data | Steps | Expected | Negative/repeat | Role | Evidence |
|---|---|---|---|---|---|---|---|---|
| TC-001 | `[REQ-ID]` | `[State]` | `[Dataset ID]` | `[Steps]` | `[Result]` | `[Case]` | `[Test role]` | `[Protected link]` |

### Обязательные уровни

- positive, negative, boundary и повторный запуск;
- права под реальными ролями, но тестовыми пользователями;
- история, аналитика и side effects;
- ошибки интеграций, retry, duplicates и recovery;
- объём/нагрузка пропорционально риску;
- rollback и post-rollback smoke;
- Cloud/On-Premise-specific checks;
- только синтетические или обезличенные тестовые данные.

---

## 17. Migration Strategy

| Dataset | Source | Target | Volume | Mapping | Clean-up | Key | Owner | Validation | Rollback |
|---|---|---|---:|---|---|---|---|---|---|
| `[Заполнить]` | `[System]` | `[Object]` | `[Count]` | `[Map ID]` | `[Rules]` | `[External ID]` | `[Роль]` | `[Counts/sample]` | `[Plan]` |

### Последовательность

~~~text
Inventory → Profile → Clean → Map → Dry-run → Test import
                                                ↓
Backup ← Go/No-Go ← Reconcile ← Full load ← Approve sample
   │
   └→ Rollback if trigger met
~~~

### Правила

- исходные данные не изменяются молча;
- объём, качество, дубли и зависимости измеряются до переноса;
- телефоны, email, реквизиты, даты и справочники нормализуются по согласованным правилам;
- название компании не является единственным надёжным ключом;
- тестовая выборка предшествует полной загрузке;
- после каждой пачки формируется отчёт и reconciliation;
- backup и rollback проверяются до cutover;
- история и неподдерживаемые объекты явно помечаются как ограничение.

---

## 18. Monitoring Plan

| Область | Контроль | Метрика/условие | Порог | Частота | Owner | Alert | Runbook |
|---|---|---|---|---|---|---|---|
| CRM | Зависшие/без owner/без next step | `[Definition]` | `[Threshold]` | `[Frequency]` | `[Роль]` | `[Channel]` | `[Link]` |
| Integrations | Ошибки, latency, retry, backlog | `[Definition]` | `[Threshold]` | `[Frequency]` | `[Роль]` | `[Channel]` | `[Link]` |
| SLA | Нарушение и риск нарушения | `[Definition]` | `[Threshold]` | `[Frequency]` | `[Роль]` | `[Channel]` | `[Link]` |
| Data quality | Missing/duplicate/conflict | `[Definition]` | `[Threshold]` | `[Frequency]` | `[Роль]` | `[Channel]` | `[Link]` |
| Automation | Failed/stuck/repeated outcome | `[Definition]` | `[Threshold]` | `[Frequency]` | `[Роль]` | `[Channel]` | `[Link]` |
| Performance | Response/queue/resource | `[Cloud-visible or server metric]` | `[Threshold]` | `[Frequency]` | `[Роль]` | `[Channel]` | `[Link]` |

Cloud monitoring ограничен доступной пользовательской, административной и интеграционной наблюдаемостью. On-Premise дополнительно включает server, PHP, DB, disk, network, web server, cron/agents, cache, queues и backup в соответствии с подтверждённой инфраструктурой.

~~~text
Signal → Detect → Classify → Assign → Mitigate → Recover → RCA
~~~

---

## 19. Risks

| № | Риск | Вероятность | Влияние | Меры |
|---:|---|---|---|---|
| 1 | Scope не согласован | High | High | Baseline и change control |
| 2 | Архитектура опирается на гипотезу | Medium | High | Traceability и evidence gate |
| 3 | Не подтверждён тариф/редакция | Medium | High | Проверка портала и официального источника |
| 4 | Функция недоступна в регионе | Low | High | Region applicability check |
| 5 | Недостаточны права команды | Medium | Medium | Access plan до реализации |
| 6 | Нет test environment | Medium | High | Изолированный портал/воронка/набор |
| 7 | Изменение затронет production | Medium | High | Pilot, freeze, backup, rollback |
| 8 | Потеря данных при миграции | Medium | Critical | Backup, dry-run, reconciliation |
| 9 | Искажение связей объектов | Medium | High | Mapping и referential tests |
| 10 | Дубли после импорта | High | High | Matching keys и idempotency |
| 11 | Низкое качество source data | High | High | Profiling, clean-up, reject flow |
| 12 | Master-система не определена | Medium | High | Master data matrix |
| 13 | Конфликт двустороннего обмена | Medium | High | Ownership и conflict resolution |
| 14 | API contract изменится | Low | High | Version check, monitoring, regression |
| 15 | REST limits превышены | Medium | High | Load model, throttling, bounded retry |
| 16 | Повтор события создаст дубль | Medium | High | Idempotency key и duplicate guard |
| 17 | Ошибка интеграции останется незаметной | Medium | High | Metrics, alerts, reconciliation |
| 18 | Секрет попадёт в Git/лог | Low | Critical | Secret store, scanning, log policy |
| 19 | Избыточные права раскроют данные | Medium | Critical | Least privilege и negative tests |
| 20 | Уволенный пользователь сохранит доступ | Low | High | Joiner/mover/leaver control |
| 21 | Экспорт доступен лишним ролям | Medium | High | Export permission review |
| 22 | Поле изменено без dependency audit | Medium | High | Field dependency register |
| 23 | Стадия изменена при активных сделках | Medium | High | Active item inventory и migration plan |
| 24 | Automation образует цикл | Medium | High | Guards и repeat tests |
| 25 | Robot зависит от конкретного сотрудника | High | Medium | Role/group assignment и substitution |
| 26 | Согласование остановится на отсутствии | Medium | High | Deputies, timeout, escalation |
| 27 | Документ содержит неверные данные | Medium | High | Source ownership и validation |
| 28 | KPI имеют разные определения | High | High | Metric catalog и owner approval |
| 29 | BI построен на грязных данных | High | High | Data quality gate |
| 30 | SLA не учитывает календарь | Medium | Medium | Explicit calendar definition |
| 31 | Производительность не оценена | Medium | High | Volume/load model и test |
| 32 | Массовая операция необратима | Low | Critical | Dry-run, batch report, rollback |
| 33 | Пользователи обходят процесс | High | Medium | UX review, training, monitoring |
| 34 | Обязательных полей слишком много | High | Medium | Stage-based minimum data |
| 35 | У сущности нет владельца | Medium | High | Ownership matrix |
| 36 | Нет причины закрытия/отказа | High | Medium | Controlled reason catalog |
| 37 | Источник лида недостоверен | High | Medium | Attribution rules и validation |
| 38 | Телефония зависит от провайдера | Medium | High | Provider validation и fallback |
| 39 | Marketplace app прекращает поддержку | Low | High | Vendor assessment и exit plan |
| 40 | On-Premise module несовместим | Medium | High | Version matrix и test update |
| 41 | Изменено ядро On-Premise | Low | Critical | Extension policy и code review |
| 42 | Backup не восстанавливается | Low | Critical | Restore exercises |
| 43 | Недостаточно disk/DB capacity | Medium | High | Capacity plan и alerts |
| 44 | Нет release owner | Medium | High | Release RACI |
| 45 | UAT не отражает реальные роли | Medium | High | Role-based synthetic scenarios |
| 46 | Нет rollback trigger | Medium | High | Measurable Go/No-Go criteria |
| 47 | Support не принял решение | Medium | Medium | Handover, runbook, training |
| 48 | Документация устаревает | High | Medium | Owner, version, review cadence |
| 49 | Критический риск не принят | Low | Critical | Formal risk acceptance |
| 50 | Клиентские данные попадут в репозиторий | Low | Critical | Protected workspace и sanitization review |

| Risk ID | Trigger | Owner | Preventive action | Contingency | Residual risk | Due | Status |
|---|---|---|---|---|---|---|---|
| R-001 | `[Signal]` | `[Роль]` | `[Measure]` | `[Plan]` | `[L/M/H]` | `[Date]` | `[Open]` |

---

## 20. ADR Decision Records

### ADR template

| Поле | Значение |
|---|---|
| ADR № | `[ADR-XXX]` |
| Решение | `[Краткий заголовок]` |
| Контекст | `[Facts, requirements, constraints]` |
| Альтернативы | `[Включая do nothing]` |
| Выбранное решение | `[Decision and rationale]` |
| Последствия | `[Positive/negative/trade-offs]` |
| Status | `[Proposed/Accepted/Deprecated/Superseded]` |
| Owner/date | `[Роль/YYYY-MM-DD]` |
| Requirement/test links | `[REQ/TC IDs]` |

### ADR-001 — Модель квалификации лидов

- **Решение:** `[Использовать лиды / direct-to-deal]`.
- **Контекст:** `[Каналы, объём, критерии квалификации]`.
- **Альтернативы:** `[Варианты]`.
- **Выбранное решение:** `[Заполнить]`.
- **Последствия:** `[Data/process/analytics]`.

### ADR-002 — Границы CRM-объектов

- **Решение:** `[Контакт/компания/сделка responsibilities]`.
- **Контекст:** `[Customer and opportunity model]`.
- **Альтернативы:** `[Варианты]`.
- **Выбранное решение:** `[Заполнить]`.
- **Последствия:** `[Связи, дубли, ownership]`.

### ADR-003 — Использование Smart Process

- **Решение:** `[Entity and lifecycle]`.
- **Контекст:** `[Почему отдельная сущность]`.
- **Альтернативы:** `[Deal/task/list/external system]`.
- **Выбранное решение:** `[Заполнить]`.
- **Последствия:** `[Rights, REST, analytics]`.

### ADR-004 — Master data ownership

- **Решение:** `[Master по domain]`.
- **Контекст:** `[Systems and conflicts]`.
- **Альтернативы:** `[Варианты]`.
- **Выбранное решение:** `[Заполнить]`.
- **Последствия:** `[Edit rules/reconciliation]`.

### ADR-005 — Automation mechanism

- **Решение:** `[Robot/trigger/BP/external service]`.
- **Контекст:** `[Complexity, failure modes]`.
- **Альтернативы:** `[Варианты]`.
- **Выбранное решение:** `[Заполнить]`.
- **Последствия:** `[Support/test/limits]`.

### ADR-006 — Integration authorization

- **Решение:** `[Webhook/OAuth/app; подтвердить]`.
- **Контекст:** `[Users, distribution, scopes, lifecycle]`.
- **Альтернативы:** `[Варианты]`.
- **Выбранное решение:** `[Заполнить]`.
- **Последствия:** `[Security/rotation/support]`.

### ADR-007 — Rights isolation

- **Решение:** `[Role/department model]`.
- **Контекст:** `[Data classification and org structure]`.
- **Альтернативы:** `[Варианты]`.
- **Выбранное решение:** `[Заполнить]`.
- **Последствия:** `[UX/admin/audit]`.

### ADR-008 — Analytics platform

- **Решение:** `[Standard reports/BI/external DWH]`.
- **Контекст:** `[Metrics, volumes, freshness]`.
- **Альтернативы:** `[Варианты]`.
- **Выбранное решение:** `[Заполнить]`.
- **Последствия:** `[Cost/data quality/access]`.

### ADR-009 — Migration and cutover

- **Решение:** `[Big bang/phased/pilot]`.
- **Контекст:** `[Volume, downtime, dependencies]`.
- **Альтернативы:** `[Варианты]`.
- **Выбранное решение:** `[Заполнить]`.
- **Последствия:** `[Duration/reconciliation/rollback]`.

### ADR-010 — Cloud or On-Premise deployment boundary

- **Решение:** `[Confirmed platform and extensions]`.
- **Контекст:** `[Requirements, constraints, operations]`.
- **Альтернативы:** `[Варианты]`.
- **Выбранное решение:** `[Заполнить]`.
- **Последствия:** `[Security, support, upgrade, cost]`.

---

## 21. Implementation Roadmap

~~~text
MVP
 ↓
Phase 1
 ↓
Phase 2
 ↓
Optimization
~~~

| Phase | Outcome | Scope | Files/configuration | Dependencies | Exit criteria | Owner |
|---|---|---|---|---|---|---|
| MVP | `[Minimum valuable outcome]` | `[Must requirements]` | `[Registers/design references]` | `[Dependencies]` | `[Acceptance metrics]` | `[Роль]` |
| Phase 1 | `[Outcome]` | `[Scope]` | `[Artifacts]` | `[Dependencies]` | `[Criteria]` | `[Роль]` |
| Phase 2 | `[Outcome]` | `[Scope]` | `[Artifacts]` | `[Dependencies]` | `[Criteria]` | `[Роль]` |
| Optimization | `[Measured improvement]` | `[Backlog]` | `[Artifacts]` | `[Production evidence]` | `[Target KPI]` | `[Роль]` |

### Increment rules

- каждый этап имеет business outcome, owner и rollback boundary;
- изменения проходят test/pilot до production;
- следующий этап не скрывает дефекты предыдущего;
- конфигурация, поля, стадии, automation и integrations имеют версии и changelog;
- фактические метрики после запуска сравниваются с baseline.

---

## 22. Architecture Review Checklist

Основной checklist содержит **105 пунктов**. Пункт закрывается только evidence, ADR или записью `[Не применимо: причина]`.

### Scope and traceability — 10 пунктов

- [ ] Discovery package утверждён и имеет версию.
- [ ] Business goals измеримы и имеют владельцев.
- [ ] Scope и exclusions согласованы.
- [ ] Stakeholders и decision owners назначены.
- [ ] Must requirements имеют acceptance criteria.
- [ ] Requirements связаны с problems и goals.
- [ ] Architecture decisions связаны с requirements.
- [ ] Tests связаны с requirements и decisions.
- [ ] Открытые вопросы имеют owners и сроки.
- [ ] Изменения scope проходят change control.

### CRM — 15 пунктов

- [ ] Решение об использовании лидов обосновано.
- [ ] Назначение контакта и компании однозначно.
- [ ] Сделка представляет отдельную коммерческую возможность.
- [ ] Направления разделены только по процессному основанию.
- [ ] Каждая стадия имеет цель, entry и exit criteria.
- [ ] Обязательные данные определены по стадиям.
- [ ] Владельцы CRM-объектов назначены.
- [ ] Источники создания объектов перечислены.
- [ ] Правила дублей и matching keys утверждены.
- [ ] Стандартные поля не дублируются без причины.
- [ ] Пользовательские поля имеют коды, типы и владельцев.
- [ ] Зависимости полей и стадий зафиксированы.
- [ ] Связи CRM-объектов проверены.
- [ ] Archive/delete rules согласованы.
- [ ] CRM model проверена на релевантных кейсах и адаптирована.

### Processes and automation — 15 пунктов

- [ ] Каждый процесс имеет trigger и результат.
- [ ] Шаги, роли и исключения описаны.
- [ ] SLA и рабочие календари определены.
- [ ] Согласования имеют decision rules.
- [ ] Замещение и absence path предусмотрены.
- [ ] Механизм automation выбран по сложности.
- [ ] Каждый robot/trigger/BP имеет owner.
- [ ] Условия запуска однозначны.
- [ ] Повторный запуск спроектирован.
- [ ] Циклы изменения стадий исключены.
- [ ] Idempotency или duplicate guard предусмотрены.
- [ ] Ошибка имеет logging и escalation.
- [ ] Ручное восстановление документировано.
- [ ] Доступность действий подтверждена на портале.
- [ ] Automation покрыта positive/negative/repeat tests.

### Smart Processes and documents — 10 пунктов

- [ ] Для Smart Process доказана самостоятельная сущность.
- [ ] Lifecycle и stages имеют критерии.
- [ ] Categories/pipelines обоснованы.
- [ ] Fields и relations минимальны и документированы.
- [ ] Rights и ownership определены.
- [ ] Volume, retention и analytics учтены.
- [ ] Документы имеют master и источник полей.
- [ ] Approval/signing/storage boundaries подтверждены.
- [ ] Версии и сроки хранения документов определены.
- [ ] Неприменимые функции помечены, а не предполагаются.

### Integrations — 15 пунктов

- [ ] Все внешние системы инвентаризированы.
- [ ] Master определён для каждого data domain.
- [ ] Направление обмена и conflict policy согласованы.
- [ ] Contract, version и exact methods подтверждены.
- [ ] Authorization и scopes минимальны.
- [ ] Секреты вынесены из Git и документа.
- [ ] Stable external IDs определены.
- [ ] Mapping и validation rules утверждены.
- [ ] Timeout и bounded retry заданы.
- [ ] Idempotency и deduplication предусмотрены.
- [ ] Rate/load limits учтены.
- [ ] Ошибки, backlog и latency наблюдаемы.
- [ ] Reconciliation и replay предусмотрены.
- [ ] Runbook и technical owner назначены.
- [ ] Contract, negative и recovery tests определены.

### Data and analytics — 15 пунктов

- [ ] Data owners назначены.
- [ ] Volumes и growth оценены.
- [ ] Data quality baseline измерен.
- [ ] Required fields и completeness targets определены.
- [ ] Duplicate rules согласованы.
- [ ] Referential integrity проверяется.
- [ ] Source-to-target mapping создан.
- [ ] Clean-up и reject flow определены.
- [ ] Migration dry-run предусмотрен.
- [ ] Reconciliation criteria утверждены.
- [ ] Retention/archive/delete policy согласована.
- [ ] Metrics имеют единые определения.
- [ ] Metrics связаны с authoritative sources.
- [ ] BI availability и datasets подтверждены.
- [ ] Ограничения качества отражаются в отчётах.

### Security — 10 пунктов

- [ ] Data classification согласована.
- [ ] Role matrix следует least privilege.
- [ ] Effective access проверяется тестовыми ролями.
- [ ] Export/delete/admin rights ограничены.
- [ ] Integration identities имеют минимальные scopes.
- [ ] Joiner/mover/leaver process определён.
- [ ] Secrets rotation и revocation определены.
- [ ] PII не попадают в технические логи.
- [ ] Cloud и On-Premise threat boundaries разделены.
- [ ] Critical security controls имеют evidence и tests.

### Testing, deployment and operations — 15 пунктов

- [ ] Test environment и synthetic data готовы.
- [ ] Smoke, functional, integration и UAT спроектированы.
- [ ] Rights и negative tests включены.
- [ ] Performance test пропорционален риску.
- [ ] Migration test sample согласован.
- [ ] Backup выполнен до рискованного изменения.
- [ ] Restore/rollback проверены.
- [ ] Release window и freeze согласованы.
- [ ] Go/No-Go и rollback triggers измеримы.
- [ ] Post-deploy smoke определён.
- [ ] Monitoring metrics и alerts настроены в доступном контуре.
- [ ] Incident owners и escalation назначены.
- [ ] Runbooks и support handover готовы.
- [ ] Training и adoption plan согласованы.
- [ ] Architecture review завершён формальным решением.

---

## 23. Common Architecture Mistakes

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Архитектура начата без утверждённого Discovery | Давление срока | Решение строится на догадках | Ввести Discovery gate |
| 2 | Нет связи с бизнес-целями | Технологический фокус | Функции без ценности | Traceability goal→requirement→decision |
| 3 | Scope не зафиксирован | Желание охватить всё | Бесконечные изменения | Baseline и exclusions |
| 4 | Пожелание названо требованием | Нет анализа | Лишняя конфигурация | Acceptance criteria и owner |
| 5 | Предположение названо фактом | Нет evidence | Неприменимая архитектура | Явные статусы и проверки |
| 6 | Нет Architecture Decision Records | Решения устные | Нельзя понять trade-off | Вести ADR |
| 7 | ADR не содержит альтернатив | Формальное заполнение | Выбор необоснован | Сравнить варианты и do nothing |
| 8 | Не учтены последствия ADR | Фокус на выгоде | Скрытый технический долг | Записать positive/negative consequences |
| 9 | Скопирован отраслевой кейс без адаптации | Шаблонный подход | Несоответствие процессу | Проверить каждую гипотезу Discovery |
| 10 | Использован паттерн как готовая настройка | Нет context fit | Ошибочная реализация | Pattern selection matrix и ADR |
| 11 | Не разделены AS IS и TO BE | Смешанное описание | Непонятен переход | Отдельные baseline/target sections |
| 12 | Не определены границы Bitrix24 | Желание хранить всё | CRM становится монолитом | System context и responsibilities |
| 13 | Не назначены master-системы | Нет data ownership | Конфликты данных | Master data matrix |
| 14 | Свободное двустороннее редактирование | Удобство пользователей | Перезапись фактов | Ownership и conflict policy |
| 15 | Лид используется без квалификационной задачи | Привычка | Лишний объект и дубли | Обосновать lead model |
| 16 | Лид исключён без анализа каналов | Упрощение | Нет контроля квалификации | Оценить источники и SLA |
| 17 | Контакт смешан со сделкой | Нет entity model | Дубли клиента | Разделить person и opportunity |
| 18 | Компания создаётся на каждое направление | Локальная удобность | Разорванный Customer 360 | Единая компания и связанные сделки |
| 19 | Сделка используется как постоянный клиент | Нет lifecycle boundaries | Искажён pipeline | Разделить account и opportunity |
| 20 | Стадии описывают действия вместо результата | Task-oriented thinking | Субъективные переходы | Entry/exit criteria |
| 21 | Воронки разделены по менеджерам | Права подменяют процесс | Сложная аналитика | Pipeline только для разных lifecycle |
| 22 | Слишком много обязательных полей | Желание собрать всё | Пользователи вводят мусор | Minimum data by stage |
| 23 | Дублируются стандартные поля | Нет field audit | Конфликт значений | Reuse standard field |
| 24 | Поля создаются без internal codes/register | Настройка в UI | Интеграции хрупкие | Field registry |
| 25 | Изменяется поле без dependency review | Локальная правка | Ломаются BP/API/BI/docs | Dependency matrix |
| 26 | Smart Process создаётся для любой таблицы | Feature-driven design | Лишняя сложность | Проверить самостоятельный lifecycle |
| 27 | Smart Process дублирует сделку | Нечёткие границы | Двойной учёт | Entity responsibility ADR |
| 28 | Связи Smart Process не спроектированы | Фокус на карточке | Потерян контекст | Relationship model |
| 29 | Задача используется как master data | Удобство | Нет устойчивой модели | Выбрать подходящую сущность/систему |
| 30 | Процесс автоматизирован до оптимизации | Automation-first | Ошибки масштабируются | Сначала TO BE process |
| 31 | Робот зависит от конкретного сотрудника | Быстрая настройка | Сбой при отсутствии | Role/group и substitution |
| 32 | Нет защиты от повторного запуска | Только happy path | Дубли задач/сообщений | Idempotency guard |
| 33 | Автоматизация образует цикл | Несвязанные роботы | Нагрузка и хаос стадий | Event map и repeat tests |
| 34 | Ошибки automation скрыты | Нет owner/monitoring | Тихая потеря процесса | Error path и escalation |
| 35 | Сложный процесс реализован одним непрозрачным блоком | Нет decomposition | Трудная поддержка | Разделить responsibilities |
| 36 | Согласование не имеет timeout | Happy-path design | Процесс зависает | Deadline, deputy, escalation |
| 37 | Документ не имеет master | Нет document architecture | Разные версии | Source/storage/status rules |
| 38 | В документ подставляются непроверенные поля | Нет validation | Юридически неверный результат | Data owner и pre-generation checks |
| 39 | Права проектируются по сотрудникам | Нет role model | Ручное сопровождение | RBAC |
| 40 | Руководителю даются права администратора | Удобство | Избыточный риск | Separate business/admin roles |
| 41 | Не проверяется эффективное наследование | Матрица только на бумаге | Скрытый доступ | Test users и evidence |
| 42 | Экспорт не рассматривается отдельно | Фокус на CRUD | Утечка данных | Export control review |
| 43 | Интеграция не имеет master | API-first | Конфликты и циклы | Data ownership before contract |
| 44 | REST method придуман по памяти | Нет source check | Нерабочий интерфейс | Official API reference |
| 45 | Webhook URL записан в архитектуру | Смешение design и secret | Компрометация | Secret reference only |
| 46 | OAuth и webhook считаются взаимозаменяемыми | Упрощение auth | Неверный lifecycle доступа | Authorization ADR |
| 47 | Не учтены rate limits | Нет load model | Throttling и потери | Volume model и backoff |
| 48 | Retry не ограничен | Попытка надёжности | Шторм запросов | Bounded retry и jitter policy |
| 49 | Нет idempotency интеграции | Stateless design | Дубли | Stable key и operation state |
| 50 | Ошибки логируются с персональными данными | Полный payload в log | Утечка | Redaction и minimal logging |
| 51 | Нет reconciliation | Доверие HTTP success | Расхождения незаметны | Periodic/business reconciliation |
| 52 | Нет runbook интеграции | Проектный фокус | Долгое восстановление | Operational handover |
| 53 | Миграция проектируется без profiling | Доверие source | Импорт плохих данных | Data quality assessment |
| 54 | Название компании — единственный ключ | Нет stable ID | Ошибочное сопоставление | External ID/composite rule |
| 55 | Полная миграция без dry-run | Давление срока | Массовая ошибка | Sample, report, approval |
| 56 | Нет rollback миграции | Оптимизм | Необратимый cutover | Backup и rollback trigger |
| 57 | История обещана без проверки переносимости | Маркетинговое ожидание | Невыполнимый scope | Object-by-object feasibility |
| 58 | KPI не имеют формулы | Каждому всё понятно | Разные отчёты | Metric catalog |
| 59 | BI строится до data quality | Визуальный приоритет | Красивые неверные цифры | Quality gate |
| 60 | Не указан grain метрики | Нет data modeling | Double counting | Define grain and dimensions |
| 61 | Cloud и On-Premise смешаны | Универсальный шаблон | Неприменимые контроли | Separate variants |
| 62 | Для Cloud предполагается доступ к DB/core | On-Premise привычка | Нереализуемый дизайн | Использовать доступные interfaces |
| 63 | Для On-Premise не учтена инфраструктура | CRM-only view | Сбои и уязвимости | Server/PHP/DB/cron architecture |
| 64 | Изменяется ядро On-Premise | Быстрый workaround | Обновления ломаются | Extension mechanisms/modules |
| 65 | Тариф/редакция не подтверждены | Демонстрационный портал | Функция недоступна | Portal and source evidence |
| 66 | Тестирование сведено к smoke | Сжатый срок | Скрытые ошибки/права | Risk-based test strategy |
| 67 | UAT выполняет только администратор | Удобный доступ | Реальные роли не проверены | Role-based UAT |
| 68 | Deployment не имеет Go/No-Go | Нет release governance | Запуск при дефектах | Measurable gates |
| 69 | Monitoring добавляется после инцидента | Delivery-only thinking | Тихие сбои | Monitoring in design |
| 70 | Архитектура не имеет owner и review cadence | Документ как разовая сдача | Быстрое устаревание | Ownership, versioning, periodic review |

### Официальные источники

Использовать только [реестр официальных источников](../knowledge/SOURCES.md). Перед утверждением решения повторно проверить доступность URL, дату материала, Cloud/On-Premise, тариф или редакцию, версию и регион.

- [Поддержка24](https://helpdesk.bitrix24.ru/) — штатные возможности, интерфейс и тарифные оговорки.
- [Bitrix24 REST API](https://apidocs.bitrix24.ru/) — архитектура интеграций и авторизации.
- [Справочник REST API](https://apidocs.bitrix24.ru/api-reference/) — точные методы, параметры, события и scopes.
- [Инструменты локальных интеграций](https://apidocs.bitrix24.ru/local-integrations/) — официальный интеграционный инструментарий.
- [Битрикс24 Маркетплейс](https://www.bitrix24.ru/apps/) — наличие приложений; качество, безопасность и поддержку оценивать отдельно.
- [Документация On-Premise](https://dev.1c-bitrix.ru/docs/portal.php) — модули, API и безопасное расширение коробочной версии.
- [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) — проверка штатной автоматизации.
- [Smart Processes](https://helpdesk.bitrix24.ru/open/18913884/) — проверка сущностей, стадий, связей, прав и automation.
- [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) — проверка доступных datasets и ограничений.

**Дата проверки локального реестра:** 2026-08-05. В рабочем Architecture package зафиксировать новую дату проверки применённых источников.
