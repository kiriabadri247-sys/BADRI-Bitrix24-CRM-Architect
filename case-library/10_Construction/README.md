# Construction CRM Architecture Case

Архитектурный кейс для синтетической строительной компании, которая продаёт проекты, участвует в тендерах, координирует строительство, подрядчиков, договоры, сроки, документы и платежи. Документ задаёт референсную CRM-архитектуру, а не готовую конфигурацию для копирования.

> **Граница безопасности:** реальные компании, сотрудники, подрядчики, адреса объектов, сметы, чертежи, договоры, реквизиты, CRM ID, webhook URL, токены и выгрузки не используются. Все данные синтетические.

> **Архитектурная граница:** Bitrix24 отвечает за CRM, тендерный и договорный lifecycle, межфункциональную координацию, задачи верхнего уровня и клиентский сервис. BIM/CDE/PDM хранит проектную модель и версии инженерной документации; ERP/1С — бюджеты, затраты, обязательства и платежи; специализированная PM/строительная система — детальный календарно-сетевой график, объёмы и исполнение. Bitrix24 не объявляется BIM, ERP или системой сметного расчёта.

Обозначения:

- **факт Bitrix24** — подтверждается официальным источником из `knowledge/SOURCES.md`;
- **решение архитектора** — целевой выбор кейса, подлежащий согласованию на Discovery;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа/редакции, версии, прав, приложения или внешней системы.

## 1. Business Context

### Модель строительного бизнеса

Референсный сценарий — подрядная/генподрядная компания, которая продаёт строительство или комплекс работ для B2B-заказчика. Проект проходит коммерческую оценку, смету, договор, мобилизацию, этапы строительства, приёмку и закрытие. Для девелопмента, долевого строительства, инфраструктурных и государственных контрактов модель уточняется отдельно.

| Аспект | Целевая интерпретация |
|---|---|
| Типы проектов | Новое строительство, реконструкция, ремонт, инженерные работы — синтетические категории |
| Коммерческая единица | Отдельная сделка/тендер на конкретный scope |
| Объект | Физический/организационный объект со стабильным ID, не только адресом |
| Проект | Управляемое обязательство после подтверждённого заказа |
| Исполнение | PM/ERP/BIM/CDE и координационная проекция в Bitrix24 |
| Финансовый факт | ERP/1С |

### Участники

- заказчик, инвестор и технический заказчик;
- менеджер продаж и тендерная команда;
- руководитель проекта;
- инженеры, сметчики и снабжение;
- подрядчики и субподрядчики;
- юристы, бухгалтерия и финансовый контроль;
- технический надзор, качество и служба эксплуатации;
- интеграционная и аналитическая команды.

### Цели CRM

1. Объединить историю клиента, тендера, сделки, объекта и проекта.
2. Не терять требования и решения при передаче из продаж в исполнение.
3. Сделать этапы, владельцев, сроки и блокировки наблюдаемыми.
4. Управлять подрядчиками, договорами и претензиями по единой модели.
5. Разделить коммерческий прогноз, утверждённый бюджет и фактические затраты.
6. Получать сверяемые показатели продаж, сроков, бюджета и качества.

```text
Market/Tender -> Sales -> Estimate -> Contract -> Project mobilisation
 -> Construction milestones -> Acceptance -> Financial close -> Service
```

## 2. AS IS Problems

| Проблема AS IS | Проявление | Корневая причина | Целевой контроль |
|---|---|---|---|
| Разрозненные данные | Клиент, объект и договор в разных таблицах | Нет data model/master policy | Stable IDs и relations |
| Нет контроля объектов | Статус зависит от устного отчёта | Нет owner/lifecycle | Object/project register |
| Информация теряется между отделами | Исполнение получает старый scope | Нет formal handoff/version | Release package |
| Нет прозрачности проектов | Руководство видит только общий процент | Нет milestones/plan-fact | Project dashboard with source |
| Сроки контролируются вручную | Просрочка выявляется поздно | Нет baseline/exception monitoring | Plan/fact и escalation |
| Сметы пересылаются файлами | Несколько итоговых версий | Нет estimate master/version | Estimate ID/revision |
| Подрядчики дублируются | Каждый проект создаёт карточку заново | Нет external supplier ID | MDM/ERP relation |
| Документы теряются | Файлы в почте/чатах | Нет CDE/ECM governance | Metadata and secure reference |
| Платежи отмечаются менеджером | Неверный cash-flow status | CRM подменяет ERP | Finance confirmation |
| Аналитика расходится | Смешаны deal/project/stage/event grains | Нет metric dictionary | Governed datasets |

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Лид | Неквалифицированный запрос, если lead-based процесс подтверждён | Bitrix24 |
| CRM | Контакт | Физическое лицо и роль у клиента/подрядчика | Bitrix24/MDM |
| CRM | Компания | Заказчик, подрядчик, партнёр | ERP/MDM/Bitrix24 по ADR |
| CRM | Сделка | Коммерческая возможность/заказ | Bitrix24 |
| Service | Обращение | Вопрос, исключение или сервисный запрос | Bitrix24 |
| Domain | Smart Process «Строительный объект» | Карточка объекта и его lifecycle | PM/ERP/BIM master or Bitrix24 by ADR |
| Execution | Smart Process «Проект» | Координационная карточка проекта | PM/ERP; Bitrix24 orchestration view |
| Sales | Smart Process «Тендер» | Сроки, версии, решения и результат участия | Bitrix24/tender system |
| Supply | Smart Process «Подрядчик» | Relationship/qualification view | ERP/MDM |
| Contract | Smart Process «Договор» | Metadata, version и согласование | ECM/ЭДО/ERP |
| Execution | Smart Process «Этап строительства» | Верхнеуровневый milestone | PM system |
| Quality | Smart Process «Претензия» | Несоответствие, ответственность и решение | Bitrix24/QMS/PM по ADR |

**Факт Bitrix24:** CRM, Smart Processes, роботы, триггеры и бизнес-процессы описаны официальными материалами. Конкретные поля, связи, права, automation и доступность проверяются на целевом портале.

**Решение архитектора:** Bitrix24 хранит коммерческий и координационный контекст. Детальная BIM-модель, рабочая документация, сметные позиции, график ресурсов, объёмы работ и финансовые проводки остаются в специализированных master-системах.

```text
Lead -> Contact/Company -> Deal/Tender
                              |
                              v
                    Construction Object
                              |
                              v
                           Project <----> PM/ERP
                          /   |   \
                         v    v    v
                    Milestone Contract Contractor
                         |      |       |
                         v      v       v
                     BIM/CDE  ECM/ЭДО  ERP/MDM
                         |
                         +-> Claim / Service / BI
```

### Правила целостности

- объект имеет устойчивый `external_object_id`; адрес не является единственным ключом;
- проект имеет `external_project_id`, связанный с утверждённой сделкой/scope version;
- смета, график, договор и проектная документация версионируются независимо;
- стадия сделки не подменяет факт выполнения строительства;
- подтверждённые plan/fact и финансовые статусы приходят из master-систем;
- повторное событие не создаёт второй проект, этап, договор или платёж;
- массовые изменения используют dry-run, sample, audit report, reconciliation и rollback.

## 4. Construction Data Model

### Сущности и поля

| Сущность | Минимальные поля | Контроль качества |
|---|---|---|
| Объект | Object ID, synthetic location/zone, type, area, lifecycle status, owner | Address is not key; source/freshness required |
| Проект | Project ID, object, scope revision, plan dates, approved budget reference, project manager, status | Baseline and current forecast separated |
| Этап | Milestone ID, project, type, plan/fact dates, owner, status, blocker | Fact confirmed by PM/source |
| Подрядчик | Contractor ID, specialization, qualification status, contracts, owner | Requisites/master in ERP/MDM |
| Тендер | Tender ID, client, object, deadline, package revision, decision/result | Submission version and timestamp |
| Договор | Contract ID, parties refs, project, version, status, secure link | Current version and authority |
| Претензия | Claim ID, project/stage/contractor, category, severity, owner, decision | Symptom, cause and remedy separated |

```text
Клиент -> Сделка -> Объект -> Проект -> Этапы -> Подрядчики -> Сдача
              |        |        |         |          |
              v        v        v         v          v
           Тендер    BIM/CDE   Смета    Договоры   Акты/Платежи
```

### Кардинальность

| Отношение | Кардинальность | Правило |
|---|---:|---|
| Company → Deal | 1:N | Самостоятельная возможность — отдельная сделка |
| Object → Project | 1:N | Фазы/проекты не перезаписывают друг друга |
| Project → Milestone | 1:N | Milestone — уровень управленческого контроля, не каждая операция |
| Project → Contractor | N:M | Назначение связано с contract/work package |
| Project → Estimate revision | 1:N | Approved baseline определён явно |
| Project → Contract | 1:N | Основной и подрядные договоры различаются |
| Stage/Contractor → Claim | 1:N | Ответственность и evidence reference трассируемы |

## 5. Sales Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новый запрос | Зарегистрировать opportunity/tender | Channel, client, object context, deadline, owner | Routing and first-action task |
| Квалификация | Подтвердить scope и fit | Project type, stakeholders, budget/timeline context, next step | SLA/missing-data control |
| Анализ объекта | Собрать исходные требования | Object ID, survey/reference, constraints, revision | Tasks to engineering/estimate roles |
| Подготовка предложения | Сформировать versioned offer | Scope revision, assumptions, exclusions, validity | Review/follow-up/expiry control |
| Смета | Получить утверждённую коммерческую смету | Estimate ID/revision, amount, currency, status | Approval route and version guard |
| Переговоры | Согласовать условия | Offer/estimate version, decisions, risks | Next-step task and escalation |
| Договор | Зафиксировать обязательства | Contract ID/version, scope, dates, approvals | Document workflow and handoff gate |
| Строительство | Передать проект в исполнение | Project ID, release package, PM owner | Idempotent project creation/projection |
| Завершение | Подтвердить приёмку и CRM closure | Acceptance/finance references, open claims | Completeness and service handoff |

```text
Новый запрос -> Квалификация -> Анализ объекта
 -> Подготовка предложения -> Смета -> Переговоры
 -> Договор -> Строительство -> Завершение

Branches: отказ / пауза / пересчёт / новый scope / претензия
```

Стадия отражает результат. Переход в «Строительство» не создаёт юридический или финансовый факт без подтверждённого release package и master-system IDs.

## 6. Project Management Architecture

### Уровни планирования

| Уровень | Инструмент/master | Содержание | CRM-проекция |
|---|---|---|---|
| Портфель | PM/ERP/BI | Проекты, приоритеты, capacity | Status/health/owner |
| Проект | PM system | Baseline, forecast, milestones, risks | Project Smart Process |
| Этап | PM system | Milestone plan/fact и acceptance | Stage Smart Process |
| Пакет работ | PM/BIM/ERP | Scope, quantities, contractor | Reference/status only |
| Задача | Bitrix24/PM | Конкретное действие и результат | Use according to ownership |
| Операция | Specialized planning | Detailed construction activity/resource | Not duplicated in CRM |

```text
Approved contract -> Project release package -> Baseline
 -> Milestones -> Work packages -> Execution facts
 -> Acceptance -> Financial/document close
```

Контроль задач включает owner, deadline, dependencies at the chosen level, expected result, evidence reference, repeat behavior and escalation. Процент готовности не вводится свободно без метода расчёта и источника.

### Отчётность

- baseline vs current forecast;
- plan/fact milestones;
- blockers and decisions;
- change requests/scope revisions;
- document readiness;
- budget reference and approved forecast;
- claims, quality and acceptance status.

## 7. Tender Management

| Элемент | Данные | Контроль |
|---|---|---|
| Заявка | Tender ID, customer, object, channel, owner | Dedup and stable external ID |
| Конкурсная документация | Package ID/version, secure reference | CDE/ECM master and access |
| Срок подачи | Due date/timezone, source, changes | Deadline monitoring and freeze |
| Решение участвовать | Version, approvers, result | Authority and audit |
| Согласования | Legal, estimate, technical, management routes | Object version/timeout |
| Результат | Submitted/won/lost/cancelled + reason | Source and final evidence |

```text
Tender intake -> Completeness -> Bid/No-bid decision
 -> Technical/estimate/legal preparation -> Final approval
 -> Submission confirmation -> Result -> Handoff or archive
```

CRM не подменяет электронную торговую площадку и не считается доказательством подачи. Факт подачи и результат имеют external reference/source timestamp.

## 8. Budget and Estimate Management

| Компонент | Master | CRM-проекция | Контроль |
|---|---|---|---|
| Смета | Specialized estimate/ERP system | Estimate ID, revision, status, summary amount | Version and validity |
| Бюджет baseline | ERP/project controls | Budget ID/version, approved amount | Approval reference |
| Forecast | ERP/project controls | Current forecast summary/date | Source timestamp |
| Факт затрат | ERP/1С | Aggregated confirmed cost | Reconciliation |
| Изменение | Change-control system | Change ID, impact, decision/status | No silent overwrite |
| Согласование расходов | ERP/approval system | Request/decision reference | Authority matrix |

```text
Scope revision -> Estimate revision -> Approval -> Budget baseline
 -> Commitments -> Actual costs -> Forecast -> Variance -> Decision
```

Bitrix24 не рассчитывает сметные позиции и не становится бухгалтерской системой. План, baseline, forecast, commitment, actual and cash payment — разные показатели и не сводятся в одно поле «Бюджет».

## 9. Contractor Management

| Объект | Данные | Master | Контроль |
|---|---|---|---|
| Подрядчик | External ID, specialization, relationship owner, qualification status | ERP/MDM | Stable ID and freshness |
| Договор | Contract ID/version/status/work package | ECM/ERP/ЭДО | Current version/access |
| Назначение | Project/stage/work package relation | PM/ERP | No duplicate/conflict |
| Документы | Checklist/status/secure links | ECM/CDE | Classification and expiry |
| Качество | Inspection/defect/claim summary | QMS/PM | Evidence source |
| Ответственность | RACI and named owner | Project governance | Escalation and acceptance |

```text
Qualification -> Selection -> Contract -> Mobilisation
 -> Work package execution -> Quality/acceptance -> Close/Review
                     |
                     +-> Defect / Claim / Corrective action
```

Карточка Company не должна дублироваться под каждый проект. Реквизиты, финансовый статус и квалификационные доказательства имеют назначенные master-системы.

## 10. Document Architecture

| Документ | Владелец | Master/хранилище | CRM-проекция |
|---|---|---|---|
| Договор | Legal/commercial owner | ECM/ЭДО/ERP | ID, version, status, secure link |
| Проектная документация | Design/BIM owner | BIM/CDE/PDM | Package/revision/status reference |
| Смета | Estimator/project controls | Estimate/ERP system | Estimate ID/revision/summary |
| Акт выполненных работ | Project/finance owner | ERP/ECM/ЭДО | Act ID, period, status, amount summary |
| Закрывающие документы | Finance | ERP/1С/ЭДО | External ID/status |
| Исполнительная документация | Construction/quality owner | CDE/ECM | Package readiness/reference |

Версия, status, owner, classification, retention and access определяются для каждого типа. Bitrix24 хранит metadata и ссылку; файлы не копируются в сделки без обоснования.

## 11. Rights Model

| Роль | CRM | Объекты | Проекты | Документы | Аналитика |
|---|---|---|---|---|---|
| Менеджер продаж | Свои/командные клиенты и сделки | Коммерческий scope | Status view after handoff | Commercial metadata | Sales/portfolio by scope |
| Руководитель проекта | Клиентский минимум | Assigned objects | Assigned projects/milestones | Project documents by need | Project plan/fact |
| Инженер | Minimal client context | Assigned object | Assigned stages/tasks | Technical links by role | Technical progress minimum |
| Сметчик | Commercial/project context | Required object data | Estimate tasks/status | Estimate packages | Estimate cycle/variance summary |
| Подрядчик | No general CRM access | Only assigned scope if approved | Assigned work package/status | Explicitly shared documents | Own delivery minimum |
| Бухгалтер | Requisites/financial context | Minimal | Financial status | Acts/invoices/closing metadata | Finance |
| Директор | Management scope | Portfolio overview | Portfolio and decisions | Only necessary documents | Management KPI |

Точная реализация CRM-ролей, Smart Process rights, отделов, групп, наследования и экспорта требует проверки на целевом портале. Внешний подрядчик не получает внутренний доступ автоматически; применяется отдельный approved collaboration model.

Контроли: least privilege, segregation of duties, service-account minimum scope, offboarding, contractor access expiry, periodic access review и UI/direct URL/search/REST/export tests.

## 12. Automation Architecture

| Механизм | Сценарий | Защита | Ошибка/эскалация |
|---|---|---|---|
| Робот/задача | Первый контакт/тендерный срок | One marker, timezone | Fallback owner |
| Задача | Техническая/сметная оценка | Scope revision and expected result | Timeout/escalation |
| Робот | Контроль milestone | Source plan/fact and threshold | PM owner alert |
| Уведомление | Blocker/decision | Minimal context and secure link | Delivery result recorded |
| Бизнес-процесс | Согласование документа | Object/version/authority | Reject/timeout/new cycle |
| Бизнес-процесс | Change request | Impact/version/approver matrix | No baseline overwrite |
| Робот | Незакрытая задача | Due date/result/status | Tiered escalation |

```text
Event -> Validate source/version -> Check authority -> Execute coordination
  |             |                         |                |
  |             +-> reject safely         |                v
  +---------------------------------------+-> Record -> Retry bounded/Escalate
```

Каждая automation имеет owner, trigger, preconditions, used fields, idempotency key, timeout, retry budget, error route, audit trail and test case. Робот не подтверждает факт строительства, бюджета или оплаты без master-event.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| 1С | Counterparties, contracts, acts, invoices, payment/accounting facts | Verified connector/application or contract integration | External IDs, mapping, queue, totals |
| ERP | Projects, budgets, commitments, costs, procurement statuses | REST/OAuth/application/event/batch contract | Version, idempotency, reconcile |
| BIM/CDE | Object/model/document package IDs, revisions, statuses | API/application/secure references | Access, revision, no large uncontrolled copies |
| PM system | Project, baseline, milestone plan/fact, blockers | API/event/batch flow | Event order, version, freshness |
| Сайт | Lead/tender request | CRM form/REST after verification | Validation, dedup, spam control |
| ЭДО/ECM | Contract/act/document status and version | API/application/secure link | Authenticity, version, access, retry |
| BI/DWH | Approved datasets | BI Constructor/controlled export | Freshness, access, metric reconciliation |
| Телефония | Call event and CRM relation | Official Bitrix24 telephony/verified provider | Unmatched calls, rights, recording policy |

```text
Source -> Authenticate -> Validate -> Map -> Version/Idempotency -> Target
  |                                                             |
  +-> source audit                                      target result
               \                                         /
                +-------- retry / reconciliation --------+
```

Контракт фиксирует source of truth, direction, keys, schema/version, document revision, units, currency, timezone, create/update/cancel semantics, event ordering, timeout, retry budget, security, monitoring, recovery and owner.

Наличие готовой интеграции с конкретной BIM, ERP, ЭДО или конфигурацией 1С не предполагается. Приложение проверяется в официальном Маркетплейсе, внешний API — в документации поставщика отдельным scope.

## 14. Analytics

| Отчёт | Метрика | Grain | Источник сверки |
|---|---|---|---|
| Продажи объектов/проектов | Pipeline, wins, contract value | Deal/project cohort | CRM + ERP/core contract fact |
| Эффективность менеджеров | Response, qualified pipeline, wins, cycle | Manager-period | CRM with quality gates |
| Прибыльность проектов | Revenue, actual/forecast cost, margin | Project-period | ERP/1С/project controls |
| Сроки строительства | Baseline/current/fact milestones, delay | Project/milestone | PM system |
| Загрузка подрядчиков | Assigned work packages/capacity context | Contractor-period | PM/ERP, not CRM task count |
| План/факт бюджета | Baseline, commitments, actual, forecast, variance | Project-period | ERP/project controls |
| Тендеры | Submission, win/loss, cycle, rework | Tender/cohort | CRM/tender source |
| Документы | Readiness, approval cycle, overdue | Document package/version | CDE/ECM |

```text
CRM + PM + ERP/1C + BIM/CDE + ECM
 -> Data contracts -> Quality gate -> Dataset/DWH
 -> Metric dictionary -> Dashboard -> Decision
                         ^                |
                         +-- reconcile ---+
```

BI Constructor, datasets and fields depend on target portal. Metric dictionary фиксирует formula, grain, baseline/version, cohort, filters, timezone, currency, units, owner, freshness and reconciliation source.

## 15. Security Architecture

### 1. Role Based Access Control

Роли sales, project manager, estimator, engineer, contractor coordinator, finance, site/department head и administrator получают доступ по проекту и функции. Сметы, договоры, тендеры, финансовые показатели и техническая конфигурация разделяются; внешнему подрядчику не предоставляется общий CRM-доступ без отдельного подтверждённого механизма.

| Роль | Scope | Ограничение |
|---|---|---|
| Sales | Клиенты и сделки | Без изменения project/estimate facts |
| Project manager | Назначенные проекты, задачи, документы | Без глобального CRM administration |
| Estimator / engineer | Сметы и технические данные | Только назначенные объекты и версии |
| Finance | Бюджет, договоры, оплаты | Без управления execution lifecycle |
| Director / head | Разрешённый portfolio и analytics | Без автоматического admin access |

### 2. Department Isolation

Sales, project office, engineering, procurement, site teams и finance разделяются подразделениями, проектными командами и ролями. Межпроектный и межфилиальный доступ разрешается по назначению и сроку. Общие контрагенты не дублируются ради изоляции проектов.

### 3. CRM Permissions

| Объект | Правило доступа |
|---|---|
| Лиды | Sales qualification, если используется lead model |
| Контакты | По клиенту, подрядчику и проектной роли |
| Компании | Account ownership; клиенты и подрядчики различаются атрибутом/ролью |
| Сделки | Sales team и утверждённые руководители |
| Smart Processes | По роли в объекте, проекте, тендере, смете или подрядчике |

Проверяются CRUD, export, REST, массовые операции и доступ через связанные объекты. Финансовые/ERP facts не редактируются свободно в CRM-проекции.

### 4. Document Security

Договоры, сметы, чертежи, акты, тендерные материалы и исполнительная документация имеют классификацию, владельца и версию. Конфиденциальные файлы хранятся в защищённом DMS/project repository; CRM содержит необходимые метаданные или ссылку. Права просмотра, изменения, скачивания и согласования разделены.

### 5. Audit

Аудиту подлежат изменения бюджета, сметы, срока, подрядчика, project status, договора и ответственного. CRM history и masked ERP/DMS/integration logs связываются по ID/correlation ID. PMO, Finance и Security owners отвечают за периодический review и RCA существенных отклонений.

### 6. Cloud Limitations

- права, журналы и доступность проектных/Smart Process controls зависят от тарифа и конфигурации;
- возможности проверяются на целевом Cloud-портале и в официальной документации;
- отсутствует доступ к server, PHP, core и DB;
- DMS/ERP/project repository controls проектируются как внешние границы.

### 7. On-Premise Considerations

Администратор защищает OS, web server, PHP, DB, modules, files, cron/agents, network, logs, backups и updates. Контролируются файловые права и capacity, backup/restore тестируются, infrastructure access отделён от business roles, ядро Bitrix24 не изменяется.

---

## 16. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Новый запрос | Validation, dedup, owner | One opportunity and first action |
| 2 | Новый тендер | External ID/deadline/timezone | One tender and controlled due date |
| 3 | Новая версия документации | Package revision | Previous version remains traceable |
| 4 | Bid/no-bid approval | Authority/version | Valid decision and audit |
| 5 | Смета | Estimate ID/revision/currency | Offer linked to exact estimate |
| 6 | Изменение scope | New revision/change process | Baseline not silently overwritten |
| 7 | Договор | Current version/rights | Only approved version is current |
| 8 | Создание проекта | Release package/idempotency | One project with external ID |
| 9 | Этап строительства | PM plan/fact mapping | Milestone status matches source |
| 10 | Просрочка этапа | Threshold/escalation | Blocker task and alert created once |
| 11 | Назначение подрядчика | Qualification/contract relation | Assignment is valid and traceable |
| 12 | Претензия | Project/stage/contractor/SLA | Controlled claim lifecycle |
| 13 | Акт и платёж | External IDs/status | CRM projection reconciles with ERP |
| 14 | Ошибка BIM/ERP/PM | Timeout/retry/dead event | No duplicate/loss, alert traceable |
| 15 | Events out of order | Version/sequence | Status does not roll back |
| 16 | Проверка прав | UI/URL/search/REST/export | Role sees only allowed scope |
| 17 | Contractor access expiry | Date/offboarding | Access revoked, objects retained |
| 18 | Миграция | Dry-run/sample/totals | Mapping and counts accepted |
| 19 | BI reconciliation | CRM/PM/ERP sample | Delta explained or corrected |
| 20 | Rollback | Trigger/runbook/reconcile | Safe prior state restored |

Все данные синтетические. UAT подписывают Sales, Tender, Project, Engineering, Estimate, Contractor management, Legal, Finance, Security and Integration owners. Positive, negative, repeat, concurrency, rights and recovery cases обязательны.

## 17. Deployment

| Этап | Действия | Gate |
|---|---|---|
| Discovery | AS IS, portal, projects, systems, data, owners | Inventory/scope signed |
| Архитектура | Data model, master matrix, integrations, rights, ADR | TO BE approved |
| CRM настройка | Test portal/funnel, fields, stages, roles | Peer review and smoke |
| Миграция | Clean, map, dry-run, sample, reconcile | Acceptance report |
| Интеграции | Contract, failure, recovery tests | End-to-end successful |
| Обучение | Role-based and contractor scenarios | UAT/readiness sign-off |
| Запуск | Freeze, backup/export, cutover, smoke | Go/no-go |
| Hypercare | Monitoring, incident route, daily reconcile | No critical defects, support accepted |

```text
Discovery -> Architecture -> Test configuration -> Contract tests
 -> Sample migration -> UAT -> Go/No-Go -> Cutover -> Smoke -> Hypercare
                                      |
                                      +-> Rollback by approved trigger
```

Cloud and On-Premise planned separately. For On-Premise additionally verify product/modules, PHP, database, files, cron, backup/restore, infrastructure and security. Core product files are not modified.

## 18. Monitoring

| Контроль | Сигнал | Owner | Реакция |
|---|---|---|---|
| Просрочка этапа | Baseline/current threshold breached | Project manager | Verify PM source, recovery plan |
| Project health/problem | Blocker/severity/age | PMO/project owner | Escalate and decision log |
| Ошибка интеграции | Error/retry/dead event | Integration owner | Recover/reconcile |
| Незакрытая задача | Deadline/result missing | Functional owner | Reassign/escalate |
| Отклонение бюджета | Forecast/actual vs baseline threshold | Project controls/finance | Change/decision process |
| Document not ready | Required package/version overdue | Document owner | Task/escalation |
| Contractor issue | Quality/delay/access event | Contractor manager | Containment/claim/review |
| CRM/master delta | Project/stage/contract/payment mismatch | Data owner | Reconciliation/source correction |
| Anomalous access/export | Security signal | Security owner | Contain/audit/revoke |

SLI: success/error rate, latency, backlog age, retry count, last successful sync, stale age and reconciliation delta. Logs contain no secrets, drawings, estimates, documents or personal data.

## 19. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Bitrix24 used as BIM/CDE | Wrong system boundary | Lost revisions/model integrity | BIM/CDE master |
| 2 | Bitrix24 used as ERP | No master policy | Wrong budget/payments | ERP/1C master |
| 3 | CRM task list used as detailed schedule | Oversimplification | Invalid critical path | PM system master |
| 4 | Duplicate clients | No identity policy | Fragmented history | Normalization/matching |
| 5 | Duplicate objects | Address used as key | Wrong relations | Stable object ID |
| 6 | Duplicate projects | Repeated event | Double execution/reporting | Idempotency key |
| 7 | Deal equals project | Lifecycles mixed | Wrong sales/project metrics | Separate entities |
| 8 | Object and project mixed | One card desire | Phases overwrite each other | Object 1:N projects |
| 9 | Scope overwritten | No revision | Work against old requirements | Version/change control |
| 10 | Estimate not linked to scope revision | Separate files | Wrong price | Revision relation |
| 11 | Estimate outdated | No validity | Loss/dispute | Validity/status control |
| 12 | Contract wrong version | Multiple copies | Legal dispute | ECM current version |
| 13 | Project starts without release package | Pressure on dates | Incomplete inputs | Formal gate |
| 14 | Milestone fact entered manually | No PM integration | False progress | Master event |
| 15 | Percent complete is subjective | No method/source | Misleading report | Approved calculation |
| 16 | Baseline overwritten | No change process | Delay hidden | Immutable baseline/new forecast |
| 17 | Forecast and actual mixed | One date/amount field | Wrong decisions | Separate measures |
| 18 | Budget/estimate/cash mixed | Weak data model | Financial errors | Separate concepts |
| 19 | Actual costs estimated in CRM | No ERP facts | Wrong profitability | ERP reconciliation |
| 20 | Contractor duplicated per project | No MDM ID | Inconsistent status | One company/external ID |
| 21 | Contractor assigned without qualification | No gate | Quality/schedule risk | Qualification master |
| 22 | Contractor access never expires | No offboarding | Data leak | Time-bound access |
| 23 | External contractor sees all projects | Broad role | Confidentiality breach | Explicit sharing/least privilege |
| 24 | Drawings attached to Deal | Convenience | Wrong version/access | CDE secure reference |
| 25 | Estimate payload logged | Debug full data | Commercial leak | Masking/allowlist |
| 26 | Secret in webhook URL published | Misclassification | Integration compromise | Secret store/rotation |
| 27 | Repeat event creates contract/stage | No event ID | Duplicates | Idempotency |
| 28 | Events arrive out of order | Async integration | Status rollback | Sequence/version |
| 29 | Infinite retry | No budget | Load/backlog | Bounded backoff |
| 30 | REST limit/timeout | Peak operations | Lag | Queue/backoff/verified batch |
| 31 | Free bidirectional editing | No field ownership | CRM/master conflict | Master matrix |
| 32 | Tender deadline timezone wrong | Local date | Missed submission | Explicit timezone/source |
| 33 | CRM status considered proof of submission | Wrong semantics | Lost tender | External confirmation |
| 34 | Approval not linked to document version | Generic process | Wrong file approved | Version ID |
| 35 | Author approves own change | No segregation | Unauthorized budget/scope | Authority matrix |
| 36 | Timeout means approval | Unsafe fallback | Unauthorized action | Exception/escalation |
| 37 | Claim not linked to contractor/stage | No relation | Weak RCA/recovery | Required links |
| 38 | Quality defect only in comments | No taxonomy | No analytics | Structured claim/defect |
| 39 | Act/payment entered manually | No ERP sync | False close | Finance master |
| 40 | Full migration without dry-run | Rush | Mass corruption | Sample/report/rollback |
| 41 | No reconciliation | HTTP success treated as result | Silent omissions | Totals/sample/delta |
| 42 | Backup not restored in test | Formal control | Rollback fails | Restore drill |
| 43 | Tariff lacks needed rights/processes | Capability check missed | Infeasible scope | Portal verification |
| 44 | Marketplace app not assessed | Trust listing | Vendor/security risk | Review/exit plan |
| 45 | On-Premise outdated | Old modules/PHP | Errors/vulnerabilities | Supported update plan |
| 46 | Rights tested as administrator | No role QA | Production access defects | Role-based UAT |
| 47 | Service account is administrator | Fast integration | Large blast radius | Least scope |
| 48 | Terminated employee remains owner | No offboarding | Orphaned projects/access | Reassign/revoke |
| 49 | Profitability uses expected deal amount | Wrong source | Wrong margin | ERP/project controls |
| 50 | Contractor load based on CRM tasks | Wrong grain | Bad planning | PM/ERP source |
| 51 | KPI has no baseline/version | Definition gap | Disputed performance | Metric dictionary |
| 52 | Dashboard lacks freshness | Stale data | Wrong decision | Last refresh/SLA |
| 53 | No integration owner | Project ended | Errors unhandled | Support RACI |
| 54 | No concurrency test | Sequential QA only | Duplicate project/approval | Parallel scenarios |
| 55 | No hypercare | Immediate handoff | Incidents accumulate | Support window |

## 20. ADR

### ADR-CON-001: Модель строительного объекта

- **Контекст:** объект живёт дольше одной сделки/фазы и имеет устойчивую идентичность.
- **Решение:** отдельный Smart Process/projected domain object with stable external ID; address is attribute, not key.
- **Альтернативы:** company/deal field; external system only.
- **Последствия:** object lifecycle, owner and 1:N projects.
- **Статус:** Proposed.

### ADR-CON-002: Модель проекта

- **Контекст:** project execution differs from sales opportunity.
- **Решение:** Smart Process «Проект» as orchestration view; PM/ERP master detailed execution.
- **Альтернативы:** deal through construction; only PM system.
- **Последствия:** release package, external ID, state mapping and reconcile.
- **Статус:** Proposed.

### ADR-CON-003: Этапы строительства

- **Контекст:** management needs milestones, while PM has detailed tasks.
- **Решение:** only governed milestones projected to Bitrix24; operations remain in PM.
- **Альтернативы:** every operation as CRM stage/task; single percent field.
- **Последствия:** baseline/current/fact separation and source rules.
- **Статус:** Proposed.

### ADR-CON-004: Подрядчики

- **Контекст:** contractor works across projects and has qualification/contracts.
- **Решение:** Company + contractor process view linked by ERP/MDM external ID; assignment via work package/contract.
- **Альтернативы:** duplicate company per project.
- **Последствия:** qualification, access expiry and contractor relation governance.
- **Статус:** Proposed.

### ADR-CON-005: Документы

- **Контекст:** contracts, BIM packages, estimates and acts have different masters.
- **Решение:** originals in ECM/CDE/ERP; Bitrix24 stores metadata/reference/version.
- **Альтернативы:** all files in Deal.
- **Последствия:** classification, access, retention and direct-link tests.
- **Статус:** Proposed.

### ADR-CON-006: Модель прав

- **Контекст:** sales, project, engineering, finance and contractors need different scopes.
- **Решение:** role-based least privilege; external access only via approved collaboration model.
- **Альтернативы:** common role/project group for all.
- **Последствия:** test accounts, contractor expiry, access review and offboarding.
- **Статус:** Proposed.

### ADR-CON-007: Аналитическая архитектура

- **Контекст:** CRM lacks detailed schedule, budget and actual cost facts.
- **Решение:** governed datasets joining CRM/PM/ERP/CDE by stable IDs and versions.
- **Альтернативы:** only CRM reports; manual spreadsheets.
- **Последствия:** metric dictionary, grain, freshness and reconciliation.
- **Статус:** Proposed.

### ADR-CON-008: Сметы и бюджет

- **Контекст:** estimate, baseline, forecast, actual and payment are different facts.
- **Решение:** estimate/ERP/project controls remain masters; CRM stores references and approved summaries.
- **Альтернативы:** user fields in Deal as sole record.
- **Последствия:** versioned summaries and no manual actual-cost editing.
- **Статус:** Proposed.

## 21. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | CRM объявлена BIM/CDE | Wrong scope | Lost model/document control | BIM/CDE master |
| 2 | CRM объявлена ERP | Desire for one system | Wrong budgets/payments | ERP/1C master |
| 3 | Deal used as construction project | One-card design | Sales/execution mixed | Separate Project |
| 4 | Task used as project entity | Work item replaces domain | No lifecycle/analytics | Project Smart Process |
| 5 | Object and project are one record | Concepts mixed | Phases overwrite | Object 1:N Project |
| 6 | Address is object key | Easy identifier | Duplicates/wrong merge | Stable object ID |
| 7 | New project creates new client company | No identity policy | Duplicate customers | Company matching |
| 8 | Contractor duplicated per project | Local convenience | Fragmented history | One external ID |
| 9 | Company name is import key | Mutable/nonunique | Wrong merges | MDM ID |
| 10 | One Deal for all client projects | Customer/process mixed | Wrong sales analytics | Opportunity unit rule |
| 11 | Every work operation becomes CRM stage | Copy PM schedule | Unusable pipeline | Milestone-level model |
| 12 | Only one stage «Строительство» | Oversimplification | No visibility | Governed milestones |
| 13 | Percent complete entered manually | No method | Misleading progress | Source/calculation rule |
| 14 | Plan and fact in one date | Weak model | Delay hidden | Separate fields/events |
| 15 | Baseline overwritten with forecast | No change control | No variance history | Immutable baseline |
| 16 | Scope change edits original description | No revision | Wrong obligations | Change/revision entity |
| 17 | Project starts from Deal stage alone | No release gate | Incomplete handoff | Release package |
| 18 | Repeat event creates project | No idempotency | Duplicate execution | External/event ID |
| 19 | Project number entered in two systems | No master | Collisions | Master-generated ID |
| 20 | Status editable freely in CRM and PM | No field owner | Conflicts | Master matrix |
| 21 | Estimate stored as attachment only | No metadata/version | Wrong price | Estimate ID/revision |
| 22 | Estimate lines copied to Deal fields | One-window desire | Stale duplicated data | Estimate system/reference |
| 23 | Estimate not linked to scope revision | Artifacts separate | Wrong commercial offer | Revision relation |
| 24 | Estimate has no validity | No date | Stale price | Validity/status |
| 25 | Budget, estimate and payment mixed | One «Amount» field | Wrong plan/fact | Separate concepts |
| 26 | Actual cost entered manually in CRM | No ERP integration | False profitability | ERP fact |
| 27 | Forecast not timestamped | Latest number only | No trend/audit | Forecast date/version |
| 28 | Change approved without impact | Incomplete workflow | Budget/schedule surprise | Impact fields/gate |
| 29 | Author approves own change | No segregation | Unauthorized scope/cost | Authority matrix |
| 30 | Approval not linked to version | Generic checkbox | Wrong artifact approved | Object version |
| 31 | Timeout treated as approval | Unsafe default | Unauthorized action | Escalation/hold |
| 32 | Tender deadline lacks timezone | Local date | Missed submission | Explicit timezone |
| 33 | CRM status proves tender submission | Wrong semantics | Lost opportunity | External receipt/reference |
| 34 | Bid package overwritten | No versioning | Cannot reproduce submission | Immutable submitted version |
| 35 | Bid/no-bid decision stays in chat | No formal process | No audit | Approval record |
| 36 | Contractor selected without qualification | Schedule pressure | Quality/risk issue | Qualification gate |
| 37 | Contractor assignment lacks contract | Relations incomplete | Responsibility dispute | Contract/work-package link |
| 38 | Contractor access is permanent | No expiry | Data leak | Time-bound access |
| 39 | Contractor sees other projects | Broad group/role | Confidentiality breach | Explicit sharing |
| 40 | Contractor status stored in text | No classifier | Cannot monitor | Structured status |
| 41 | Defect only in comment | No quality entity | No RCA/analytics | Claim/defect record |
| 42 | Claim not linked to stage/contractor | No relation | Weak responsibility | Required links |
| 43 | Corrective action closes without verification | Task equals result | Repeat defect | Effectiveness check |
| 44 | Contract stored unversioned in Deal | Convenience | Wrong legal document | ECM/version metadata |
| 45 | BIM file copied into CRM | One-window desire | Wrong revision/storage load | CDE reference |
| 46 | Drawing link accessible after role change | No access lifecycle | IP leak | Access review/expiry |
| 47 | Act/payment marked by project manager | No finance sync | False closure | ERP confirmation |
| 48 | Closing documents have no checklist | No gate | Delayed payment/close | Document package status |
| 49 | Webhook URL in README | Secret misclassification | Compromise | Secret store/env |
| 50 | Estimate/document payload in logs | Debug full data | Commercial leak | Allowlist/masking |
| 51 | Integration account is admin | Fast setup | Large blast radius | Least scope |
| 52 | No timeout | Happy path | Hung sync | Explicit timeout |
| 53 | Infinite retry | No budget | Load/backlog | Bounded backoff |
| 54 | Retry non-idempotent | No event key | Duplicate project/stage | Idempotency |
| 55 | Out-of-order events applied | Async | Status rollback | Sequence/version |
| 56 | Errors silently ignored | No dead letter | Stale status | Failed-event register |
| 57 | Batch used without method check | Speed focus | Partial failure | Verify limits/semantics |
| 58 | Full import without dry-run | Rush | Mass corruption | Sample first |
| 59 | Object address used as import key | No ID | Wrong merge | Stable key |
| 60 | No source-target report | Opaque migration | No rollback/reconcile | Mapping log |
| 61 | No counts/sample validation | HTTP success assumed | Silent omissions | Acceptance report |
| 62 | Existing project users not consulted | Technical-only discovery | Process disruption | Stakeholder inventory |
| 63 | Rights tested as administrator | No role QA | Production access defects | Role accounts |
| 64 | Direct URL/REST/export not tested | UI-only test | Access bypass | Multi-channel tests |
| 65 | Engineer sees full client base | Broad role | Excess access | Need-to-know |
| 66 | Sales sees confidential contractor docs | Copied access | Confidentiality risk | Entity/document scope |
| 67 | Terminated employee remains owner | No offboarding | Orphaned projects/access | Reassign/revoke |
| 68 | Expired contractor retains access | No review | Data leak | Automated/manual expiry control |
| 69 | Sales KPI counts project stage as revenue | Wrong fact | Overstated sales | Contract/ERP source |
| 70 | Profitability uses Deal amount | No actual cost | Wrong margin | ERP/project controls |
| 71 | Contractor load based on CRM tasks | Wrong grain | Bad planning | PM/ERP capacity |
| 72 | Delay metric ignores baseline version | No definition | Disputed KPI | Metric dictionary |
| 73 | Milestones counted as projects | Wrong grain | Inflated portfolio | Project grain |
| 74 | Dashboard has no freshness | Stale data | Wrong decision | Last refresh/SLA |
| 75 | Marketplace app not assessed | Trust listing | Vendor/security risk | Review/exit plan |
| 76 | On-Premise version unsupported | Upgrade ignored | Errors/vulnerabilities | Supported plan |
| 77 | Backup never restored | Formal control | Rollback fails | Restore drill |
| 78 | Launch without go/no-go | No decision owner | Uncontrolled cutover | Release criteria |
| 79 | No integration/support owner | Project handoff gap | Incidents unhandled | Support RACI |
| 80 | No hypercare | Immediate closure | Issues accumulate | Support window |

## 22. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Confirmed AS IS, volumes and system boundaries | Inventory, stakeholders, integrations, capability check | Masters and owners defined |
| 2. Архитектура | Approved TO BE | Data model, lifecycle, RACI, ADR, risks | Owners signed decisions |
| 3. CRM настройка | Working sales/tender contour | Fields, stages, roles, dedup, tests | Smoke/rights tests pass |
| 4. Объекты и проекты | Working object/project/milestone orchestration | Smart Process registry, release/change rules | Synthetic end-to-end project passes |
| 5. Документы | Governed contract/estimate/CDE references | Document matrix, versions, access and close checklist | Version/right tests pass |
| 6. Интеграции | Controlled PM/ERP/BIM/ECM flows | Contracts, retries, monitoring, reconciliation | Failure/recovery tests pass |
| 7. Тестирование | Business readiness confirmed | Test report, defects, UAT sign-off | No critical/high defects |
| 8. Запуск | Controlled cutover completed | Runbook, backup, rollback, communication | Smoke/reconcile successful |
| 9. Оптимизация | SLA and KPI stabilized | Baseline, backlog, access review, lessons learned | Support model accepted |

### Официальные источники

Используются только источники из `knowledge/SOURCES.md`; дата последней проверки реестра — 2026-08-05.

| Источник | Применение |
|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | Пользовательские возможности CRM и Cloud |
| [Документация REST API](https://apidocs.bitrix24.ru/) | Интеграционная архитектура и ограничения |
| [REST API reference](https://apidocs.bitrix24.ru/api-reference/) | Проверка контрактов методов и полей |
| [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/) | Проверка вариантов интеграционного доступа |
| [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) | Проектирование object, project, tender, contractor, contract, stage and claim views |
| [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) | Проверка automation |
| [Как выбрать тип бизнес-процесса](https://helpdesk.bitrix24.ru/open/23951294/) | Выбор approval/orchestration механизма |
| [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) | Проверка вариантов телефонии |
| [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) | Проверка datasets и аналитики |
| [Маркетплейс Bitrix24](https://www.bitrix24.ru/apps/) | Проверка приложений и коннекторов |
| [Документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php) | On-Premise возможности и ограничения |

Если функция, поле, связь, событие, REST-метод, приложение или ограничение не подтверждены официальным источником и целевым порталом, они помечаются **«требует проверки»** и не включаются в обязательства перед заказчиком.
