# Government and Municipal Service CRM Architecture Case

Архитектурный кейс для синтетической государственной или муниципальной организации, которая регистрирует обращения граждан и организаций, координирует поручения, проекты, документы и взаимодействие подразделений. Документ задаёт референсную архитектуру Bitrix24, но не заменяет обследование, нормативную экспертизу, модель угроз, регламент делопроизводства или проект государственной информационной системы.

> **Граница данных:** реальные государственные органы, граждане, сотрудники, адреса, документы, номера дел, идентификаторы государственных систем, телефоны, email, CRM ID, webhook URL, токены и иные персональные либо служебные сведения не используются. Все примеры обезличены и синтетические.

> **Архитектурная граница:** Bitrix24 используется как контур регистрации, CRM-взаимодействия, маршрутизации, задач, согласований и управленческого контроля. Официальный реестр, юридически значимая СЭД/ЭДО, архив, система идентификации, отраслевые ГИС и нормативный источник сроков остаются master-системами. Возможность обмена подтверждается отдельно; наличие штатного коннектора не предполагается.

Обозначения:

- **факт Bitrix24** — подтверждается официальным источником из knowledge/SOURCES.md;
- **решение архитектора** — целевая рекомендация, утверждаемая на Discovery;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа или редакции, версии, региона, прав и действующего регламента;
- **вне CRM** — юридический, отраслевой или инфраструктурный факт, который нельзя объявлять возможностью Bitrix24.

## 1. Business Context

### Модель работы организации

Организация принимает запросы через сайт, электронную почту, телефон, личный приём и утверждённые государственные каналы. Обращение должно быть зарегистрировано, классифицировано, передано компетентному подразделению, исполнено, проверено и закрыто с контролируемым ответом.

| Участник | Ответственность | Ожидаемый результат |
|---|---|---|
| Гражданин или организация | Передать запрос и необходимые сведения | Подтверждение регистрации и результат |
| Оператор | Идентифицировать канал, проверить полноту, зарегистрировать | Единая карточка обращения |
| Специалист отдела | Рассмотреть и подготовить результат | Проверяемое исполнение |
| Руководитель | Распределить нагрузку и контролировать срок | Управляемая очередь |
| Контролёр | Проверить полноту, срок и основание закрытия | Независимый контроль |
| Аналитик | Поддерживать метрики и качество данных | Достоверная отчётность |
| Администратор | Настраивать платформу | Работоспособная конфигурация без автоматического доступа к содержанию |

Цели внедрения:

- единое окно регистрации без потери каналов;
- прозрачная ответственность и следующий шаг;
- разделение нормативного срока, внутреннего SLA и технических таймеров;
- контролируемая межведомственная или межподразделенческая передача;
- минимизация персональных данных;
- измеримая нагрузка, качество и причины просрочек;
- воспроизводимый аудит действий и интеграций.

**Решение архитектора:** до настройки фиксируются Cloud/On-Premise, тариф или редакция, права, действующие сущности, процессы, интеграции, объёмы, критичность, допустимый простой и наличие тестового портала.

## 2. AS IS Problems

| Проблема | Наблюдаемый симптом | Архитектурная причина | Целевой контроль |
|---|---|---|---|
| Нет единого окна | Каналы ведут разные журналы | Нет общего intake и source ID | Реестр входов и сверка |
| Запросы теряются | Нет подтверждения регистрации | Fire-and-forget интеграция | Ack, correlation ID, reconciliation |
| Нет контроля исполнения | Ответственность в письмах | Нет owner, deadline, next step | Lifecycle и задачи |
| Подразделения разобщены | Повторный ручной ввод | Нет handoff-контракта | Связанные сущности и пакет передачи |
| Сроки трактуются по-разному | Один таймер для всех случаев | Нормативный срок смешан с SLA | Версионируемый календарь и основание |
| Документы копируются | Несколько неуправляемых версий | CRM объявлена СЭД | Ссылка на master-документ |
| Права слишком широкие | Доступ по удобству | Нет классификации данных | Deny-by-default и scopes |
| Аналитика недостоверна | Отчёты не сходятся | Смешаны grains и статусы | Metric contracts и DQ |

Сначала строится AS IS: каналы, реестры, виды обращений, сроки, подразделения, исключения, документы, master-системы, интеграции, права, отчёты и болевые точки. Настройка по неподтверждённой схеме запрещена.

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Contact | Минимальный профиль физического лица | Утверждённый реестр/Bitrix24 по решению |
| CRM | Company | Организация-заявитель или контрагент | Утверждённый реестр |
| CRM | Deal | Только коммерческая/договорная возможность, если она реально существует | Bitrix24 |
| Smart Process | Citizen Request | Жизненный цикл обращения | Bitrix24 как процессный реестр либо внешний реестр |
| Smart Process | Project | Управленческий проект и контрольные точки | Bitrix24/PM-система |
| Smart Process | Approval | Экземпляр управленческого согласования | Bitrix24; юридическая значимость вне CRM |
| Smart Process | Document Card | Метаданные и ссылка на документ | СЭД/ЭДО/DMS |
| Smart Process | Control Assignment | Поручение с ответственным и сроком | Bitrix24/официальная СЭД |
| Smart Process | Event | Мероприятие, участники и результат | Bitrix24/календарная система |
| Tasks | Task | Исполнимая работа с конкретным результатом | Bitrix24 |

~~~text
Approved Channels -> Intake Gateway -> Citizen Request -> Classification
                                         |       |
                                         |       +-> Task / Control Assignment
                                         v
Citizen/Organization <-> CRM Profile -> Department Review -> Control -> Response
                                         |                 |
                                         +-> DMS Reference  +-> Audit / Analytics
~~~

### Правила целостности

1. Обращение, гражданин, документ, поручение и проект имеют разные lifecycle и не объединяются в одну карточку.
2. Source ID, correlation ID и версия источника отделены от внутреннего CRM ID.
3. Contact не превращается в медицинское, социальное или иное отраслевое досье.
4. Документ хранится в утверждённой СЭД; CRM хранит минимальные метаданные и защищённую ссылку.
5. Нормативный срок поступает из утверждённого правила, а не из свободного поля или робота.
6. Повторная доставка события не создаёт второе обращение, задачу или уведомление.
7. Закрытие требует результата, основания, времени и проверки обязательных данных.

**Cloud:** нет доступа к ядру, серверному PHP и базе данных; сложная интеграция, секреты и расширенные журналы размещаются во внешнем защищённом контуре. **On-Premise:** дополнительно проектируются сервер, PHP, DB, файлы, cron, резервное копирование, обновления и безопасность; ядро не изменяется.

## 4. Citizen Journey

~~~text
Обращение гражданина -> Регистрация -> Распределение -> Рассмотрение
 -> Исполнение -> Контроль -> Ответ -> Закрытие

Exception path:
Неполные данные / неверная компетенция / дубликат / отзыв / обжалование
 -> утверждённое решение -> уведомление -> сохранение истории
~~~

| Этап | Контрольная точка | Обязательный результат | Недопустимое действие |
|---|---|---|---|
| Обращение | Канал и source timestamp | Полученный пакет | Логирование полного payload без необходимости |
| Регистрация | Уникальность и полнота | ID и подтверждение | Создание без source ID |
| Распределение | Компетенция и очередь | Ответственный подразделения | Маршрутизация по свободному тексту без контроля |
| Рассмотрение | Основание и документы | План/следующий шаг | Подмена нормативного решения роботом |
| Исполнение | Задачи и зависимости | Проверяемый результат | Закрытие по факту уведомления |
| Контроль | Срок, качество, полномочия | Принято или возвращено | Самопроверка без правила |
| Ответ | Утверждённый канал и версия | Зафиксированная отправка | Передача лишних данных |
| Закрытие | Outcome и audit trail | Финальный статус | Удаление истории |

Journey допускает повторное обращение и обжалование как отдельные связанные случаи. Связь не означает автоматическое копирование всех персональных данных.

## 5. Public Sector Data Model

| Сущность | Минимальные данные | Связи | Запрещённая перегрузка |
|---|---|---|---|
| Гражданин | Synthetic profile ID, разрешённые контакты, preference | Requests, authorized relations | Полное отраслевое досье |
| Организация | External ID, type, approved requisites projection | Contacts, requests, projects | Копия официального реестра |
| Обращение | Type, category, source, owner, deadline basis, status, outcome | Citizen/company, tasks, docs | Документ целиком и секреты |
| Проект | Goal, owner, dates, milestones, status | Departments, tasks, docs | Портфельная система без обоснования |
| Документ | DMS ID, version, class, status, protected link | Request/project/approval | Неуправляемая файловая копия |
| Поручение | Owner, due date, control status, result ref | Request/project/document | Неограниченный комментарий с ПДн |
| Мероприятие | Type, date, owner, participants by minimum | Project/request | Скрытая система учёта посещаемости |

~~~text
Citizen 1 ---- N Request N ---- 1 Department
   |                 |  \
   |                 |   +---- N Task / Control Assignment
   |                 +-------- N Document Reference ---- 1 DMS
   |
Organization 1 ---- N Request / Project
Project 1 ---- N Milestone ---- N Task
~~~

Ключи и поля:

- внешний стабильный ID не заменяется ФИО, телефоном или email;
- поле содержит один бизнес-смысл, тип, владельца, источник и правило качества;
- множественные значения применяются только при доказанной кардинальности;
- обязательность включается на стадии, где данные действительно известны;
- справочники категорий, outcomes и причин возврата версионируются;
- файлы и комментарии получают отдельную классификацию доступа и retention.

## 6. Request Management

| Стадия | Цель | Обязательные данные | Автоматизация | Gate |
|---|---|---|---|---|
| Новое обращение | Зафиксировать поступление | Source ID, channel, received at | Ack, dedup, intake queue | Пакет принят |
| Классификация | Определить тип и компетенцию | Category, subject, sensitivity | Suggested route/task | Human/rule validation |
| Назначение исполнителя | Установить accountability | Department, owner, due basis | Assignment and notification | Accept/handoff |
| Работа | Подготовить результат | Next step, tasks, dependencies | Reminders and controls | Evidence complete |
| Проверка | Проверить срок и качество | Reviewer, checklist, result version | Review task/escalation | Accepted/returned |
| Ответ гражданину | Отправить утверждённый результат | Channel, template/version, authority | Transactional send | Identity/current version |
| Закрытие | Зафиксировать outcome | Outcome, dates, unresolved items | Completeness check | Closure criteria |

~~~text
New -> Classified -> Assigned -> In Work -> Review -> Response -> Closed
          |              |          |          |
          +-> Redirect   +-> Pause  +-> Return +-> Appeal/Reopen relation
~~~

**Рекомендация архитектора:** пауза срока, перенаправление, объединение дублей, отзыв и обжалование описываются отдельными transition rules. Автоматизация не определяет компетенцию или правовую позицию без утверждённого правила и ответственного.

## 7. Document Management

| Объект | В Bitrix24 разрешено | Master-система | Контроль |
|---|---|---|---|
| Карточка документа | ID, class, version, status, owner, protected link | СЭД/DMS | Access and freshness |
| Проект ответа | Управляемая рабочая версия при разрешении | СЭД/DMS | Version/approval |
| Подпись | Только статус/ссылка | Утверждённый ЭДО/подпись | Authoritative verification |
| Архив | Ссылка и retention class | Архивная система | Hold/retention |
| Вложение гражданина | Минимально необходимая защищённая копия либо ссылка | Intake/DMS | Malware, type, access, deletion |

~~~text
Request -> Document Card -> Protected DMS Link -> Version / Approval / Signature
                 |
                 +-> CRM shows minimum status; DMS remains legal record
~~~

Юридическая значимость, электронная подпись, регистрационный номер, номенклатура дел, сроки хранения и уничтожение определяются официальными системами и регламентами. Нельзя заявлять, что стандартный CRM-файл автоматически обеспечивает эти свойства.

## 8. Internal Process Management

| Процесс | Инструмент-кандидат | Условие применения | Контроль |
|---|---|---|---|
| Поручение | Task или Smart Process | Есть конкретный результат и owner | Deadline, acceptance, escalation |
| Согласование | Robot/business process | Утверждены маршрут и полномочия | Version, substitution, rejection |
| Проект | Project tasks/Smart Process | Есть scope, milestones, RACI | Baseline and change control |
| Межведомственный обмен | Approved integration | Есть соглашение и контракт | Ack, retries, reconciliation |
| Мероприятие | Smart Process/tasks/calendar | Определены участники и результат | Invitation, attendance minimum, outcome |

Перед автоматизацией фиксируются инициатор, условие запуска, исполнитель, срок, поля, повторный запуск, защита от дублей, исключения, эскалация и тест. Организационный процесс нельзя маскировать одной длинной воронкой.

## 9. Citizen Service Architecture

| Область | Решение архитектора | Ограничение |
|---|---|---|
| Intake | Единый реестр каналов с source ID | Каналы и тариф проверяются |
| Уведомления | Только transactional и по актуальной версии | Законность канала вне CRM |
| SLA | Внутренний operational SLA отдельно | Не равен нормативному сроку |
| Quality | Review outcome, reopen, complaint reason | Нельзя оценивать по скорости без контекста |
| Identity | Утверждённая проверка перед раскрытием | Bitrix24 не объявляется системой идентификации |
| Accessibility | Альтернативные каналы и ручной fallback | Конкретные требования утверждаются |

~~~text
Channel -> Identity/Consent Rule -> Request -> SLA/Deadline Engines
                                      |              |
                                      v              v
                                  Service Team -> Quality Control -> Response
~~~

Metric customer satisfaction применяется только при добровольной, понятной и методологически корректной выборке. Отсутствие ответа не считается положительной оценкой.

## 10. Rights Model

| Роль | Граждане | Обращения | Документы | Проекты | Аналитика |
|---|---|---|---|---|---|
| Оператор | Минимум подтверждённого профиля | Create/assigned intake | Только intake refs | Нет | Своя очередь |
| Специалист отдела | Need-to-know | Assigned/unit | По назначению | Участник | Операционная |
| Руководитель | Подразделение | Team/approve/escalate | По полномочию | Portfolio unit | Подразделение |
| Контролёр | Минимум для проверки | Review/read governed scope | Review refs | Контрольные точки | Quality/SLA |
| Аналитик | Псевдонимизированный минимум | Aggregate/read by policy | Нет содержания | Aggregate | Governed datasets |
| Администратор | Нет автоматического права | Конфигурация, не business content | Нет автоматического права | Config only | Dataset config по необходимости |

Принципы: deny-by-default, least privilege, purpose limitation, department and case scopes, named accounts, segregation of duties, временный доступ с expiry, periodic recertification, export approval and negative testing.

Наследование проверяется фактически на целевом портале. Роль администратора платформы не должна автоматически считаться полномочием на чтение всех обращений и документов.

## 11. Automation Architecture

| Автоматизация | Trigger | Действие | Защита |
|---|---|---|---|
| Registration | Новый валидный source event | Создать/связать обращение | Source key and idempotency |
| Routing | Подтверждённая категория | Назначить очередь/задачу | Deterministic rule and fallback |
| Deadline control | Утверждённый threshold | Напомнить/эскалировать | Calendar/version/one event |
| Handoff | Принято решение о передаче | Создать пакет и запросить ack | No close before acceptance |
| Approval | Готова конкретная версия | Запустить маршрут | Version lock and authority |
| Response | Одобрена текущая версия | Отправить по каналу | Identity/current-state check |
| Integration error | Ошибка/timeout | Queue, bounded retry, alert | Correlation and dead-letter handling |

~~~text
Event -> Validate source / rights / schema / version -> Idempotency
 -> Bounded action -> Audit result -> Exception queue -> Human escalation
~~~

Робот не принимает правовое решение, не подтверждает подпись и не закрывает обращение только по факту отправки уведомления. Для массовых изменений обязательны dry-run, тестовая выборка, пакетный отчёт и откат.

## 12. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| Государственная ИС | Approved IDs, statuses, minimal attributes | Документированный API через защищённый middleware | Auth, schema, ack, reconciliation |
| Сайт/форма | Request payload and consent evidence | Supported form/API/gateway | Validation, dedup, anti-abuse |
| Электронная почта | Message metadata/content by policy | Approved connector/integration | Thread mapping, attachment control |
| Телефония | Call event and CRM relation | Bitrix24 telephony/SIP/REST app | Rights, missed calls, correlation |
| СЭД/ЭДО | Document IDs, versions, statuses | Vendor API/middleware | Signature/status authority, version |
| BI/DWH | Governed events and dimensions | BI Constructor or approved ETL | Grain, masking, freshness, DQ |

~~~text
External Systems -> API Gateway / Middleware
 -> AuthN/AuthZ -> Schema -> Idempotency -> Mapping -> Bitrix24
                                            |
                                            +-> Audit / DLQ / Reconciliation
~~~

Наличие API или штатного коннектора для конкретной государственной системы не предполагается. REST methods, scopes, limits, тариф, регион и условия приложения проверяются по официальной документации и на целевом портале. Секреты находятся вне Git и payload logs.

## 13. Analytics

| Отчёт | Grain | Показатели | Проверка качества |
|---|---|---|---|
| Количество обращений | Request/category/channel/period | Received, closed, backlog | Dedup and completeness |
| Срок исполнения | Request/milestone | Median, percentiles, breach | Calendar/pause/version |
| Нагрузка | Assignment/employee/period | Opened, completed, WIP | Capacity and reassignment |
| Качество обслуживания | Request/outcome/review | Reopen, return, complaint | Sampling and context |
| Эффективность подразделений | Unit/request cohort | Throughput, age, quality | Comparable scope |
| Каналы | Intake event/channel | Volume, delivery, loss | Source reconciliation |

~~~text
CRM Events + DMS Status + External Registry Facts -> Conformed IDs
 -> DQ / Reconciliation -> Governed Metric Layer -> Role-based Dashboards
~~~

Для каждой метрики фиксируются owner, formula, grain, period, timezone, calendar, exclusions, source, freshness, access and control total. Граждане, обращения, задачи, документы и проекты не суммируются как единый grain.

## 14. Security Architecture

| Контроль | Реализация | Проверка |
|---|---|---|
| Data minimization | Только необходимые поля и projection | Field/file inventory |
| Segmentation | Контуры и роли по подразделению/назначению | Cross-unit negative test |
| Identity | Named accounts, MFA/SSO when available and approved | Access review |
| Documents | Protected DMS links, version and access | Revoked/expired access |
| Integration | Least-privilege credentials, allowlist, rotation | Scope and revoke drill |
| Logs | Masked payload, correlation ID | Secret/PII scan |
| Export | Approval, minimum fields, protected destination | Export test |
| Retention | Approved keep/archive/delete/hold rules | Periodic execution evidence |
| Audit | CRM history plus integration/security logs | End-to-end trace |

**Cloud:** провайдер управляет платформенным слоем; заказчик управляет конфигурацией, пользователями, правами, интеграциями, клиентскими устройствами и данными. **On-Premise:** заказчик дополнительно отвечает за OS, web server, PHP, DB, modules, files, cron, backup, updates, monitoring and hardening. Конкретные средства безопасности и сертификационные требования подтверждаются отдельно.

## 15. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Новое обращение с сайта | Валидный synthetic payload | Один request, source ID, ack |
| 2 | Повторная доставка | Тот же source event | Дубликат не создан |
| 3 | Неполные данные | Нет обязательного атрибута | Controlled exception queue |
| 4 | Классификация | Утверждённая категория | Верная очередь и owner |
| 5 | Неизвестная категория | Mapping отсутствует | Fallback, не потеря |
| 6 | Межотдельская передача | Новый отдел принимает | История и ack сохранены |
| 7 | Отказ принять | Нет компетенции | Возврат по правилу, owner остаётся |
| 8 | Нормативный срок | Synthetic calendar | Верный deadline basis |
| 9 | Пауза SLA | Утверждённое основание | Только operational timer изменён |
| 10 | Просрочка | Threshold достигнут | Одна эскалация нужной роли |
| 11 | Согласование ответа | Две версии | Отправлена только одобренная |
| 12 | Ошибка СЭД | Timeout after accept | Idempotent retry, no duplicate doc |
| 13 | Cross-unit rights | Чужой специалист | Доступ запрещён |
| 14 | Аналитик | Pseudonymized dataset | Нет содержания/лишних ПДн |
| 15 | Экспорт | Пользователь без полномочия | Экспорт запрещён/зафиксирован |
| 16 | Повторное обращение | Связанная тема | Новый case со ссылкой, история не перезаписана |
| 17 | Обжалование | Closed request | Separate related lifecycle |
| 18 | Migration dry-run | Synthetic sample | Report only, no production write |
| 19 | Cloud/On-Premise | Capability matrix | Unsupported assumptions found |
| 20 | Rollback | Failed release | Предыдущее состояние восстановлено |

Тесты используют synthetic citizens, organizations, requests, documents and projects. Обязательны позитивный, негативный, повторный, rights, failure, recovery and reconciliation paths.

## 16. Deployment

| Этап | Результат | Gate |
|---|---|---|
| Discovery | AS IS, systems, volumes, regulations, criticality | Owners and compliance approval |
| Архитектура | TO BE, master matrix, rights, threat model, ADR | Architecture/security review |
| Настройка | Isolated CRM/process configuration | Configuration review |
| Миграция | Map, clean, dry-run and reconciliation | Owner sign-off |
| Интеграции | Contracts, secrets, retries, monitoring | Failure/security/replay tests |
| Обучение | Role-based synthetic cases and runbooks | UAT |
| Запуск | Cutover, communication, fallback, rollback | Go/no-go |
| Stabilization | Hypercare and baseline observation | Exit criteria |

~~~text
Inventory -> Dependency/Threat Review -> Test Configuration -> Synthetic UAT
 -> Migration Dry-run -> Go/No-Go -> Release -> Smoke/Reconciliation
 -> Hypercare or Rollback
~~~

До изменения полей, стадий, прав или automation проверяются активные сущности, документы, отчёты, integrations and users. Для действующего процесса предпочтителен parallel pilot.

## 17. Monitoring

| Сигнал | Правило | Действие | Владелец |
|---|---|---|---|
| Нерегистрированное поступление | Source count отличается | Incident/replay | Intake owner |
| Необработанное обращение | Ack/assignment SLA | Assign/escalate | Service owner |
| Нарушение срока | Approved threshold | Escalate/review | Department head |
| Зависшая задача | No progress/overdue | Owner and manager action | Process owner |
| Integration error | Error/retry/DLQ | Investigate/replay | Integration support |
| Stale DMS status | Freshness exceeded | Mark stale, block unsafe action | DMS owner |
| Rights anomaly | Unexpected access/export | Security incident | Security owner |
| BI mismatch | Control total difference | Mark report stale | BI owner |

Alert содержит system, object reference, correlation ID, time, severity, owner and runbook; без полного текста обращения, документа, секрета и лишних персональных данных.

## 18. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Потеря входящего обращения | Нет ack | Нарушение процесса | Intake reconciliation |
| 2 | Дубликат обращения | Повторная доставка | Двойная работа | Source key/idempotency |
| 3 | False merge граждан | Слабый ключ | Раскрытие чужих данных | Conservative matching |
| 4 | ФИО как идентификатор | Нет stable ID | Ошибочная история | Approved external ID |
| 5 | CRM стала официальным реестром без решения | Scope drift | Юридический риск | Master matrix |
| 6 | Deal используется для любого обращения | Sales-centric template | Искажённый lifecycle | Smart Process |
| 7 | Contact содержит отраслевое досье | Удобство | Excessive data | Minimization |
| 8 | Нормативный срок равен SLA | Термины смешаны | Неверная просрочка | Separate clocks |
| 9 | Срок задан вручную | Нет rule source | Ошибка исполнения | Governed deadline basis |
| 10 | Пауза без основания | KPI pressure | Манипуляция сроком | Approved reason/audit |
| 11 | Маршрутизация ошибочна | Устаревший справочник | Потеря времени | Versioned mapping/fallback |
| 12 | Передача без ack | Fire-and-forget | Нет ответственного | Acceptance protocol |
| 13 | Закрытие по отправке | Notification equals outcome | Неполное исполнение | Closure gate |
| 14 | Робот принимает правовое решение | Overautomation | Неправомерный результат | Human authority |
| 15 | Документ скопирован в CRM | CRM as DMS | Version leak | Protected DMS link |
| 16 | Неактуальная версия ответа | Нет version gate | Ошибочный ответ | Current approved version |
| 17 | Подпись считается по флагу CRM | Нет authority | Ложная юридическая значимость | E-sign master check |
| 18 | Архив равен backup | Термины смешаны | Утрата документа | Separate policies |
| 19 | Broad operator access | Роль по удобству | Массовая утечка | Queue/department scope |
| 20 | Admin считается super-reader | Role confusion | Неконтролируемый доступ | Separate privileges |
| 21 | Аналитик видит полный текст | Raw dataset | Excess disclosure | Pseudonymization |
| 22 | Export unrestricted | Defaults | Массовая утечка | Approval/minimum |
| 23 | Temporary access permanent | Manual grant | Residual risk | Expiry/recertification |
| 24 | Secret в Git | Реальный пример | Компрометация | Secret store/rotation |
| 25 | Token overprivileged | One credential | Large blast radius | Least privilege |
| 26 | Raw payload в логах | Debug convenience | Data leak | Masking |
| 27 | Infinite retry | Нет DLQ | Load/outage | Bounded backoff |
| 28 | Out-of-order event | Async ignored | Старый статус побеждает | Version/order guard |
| 29 | Schema drift | Source change | Corrupt data | Contract validation |
| 30 | Нет correlation ID | Fragmented logs | Slow RCA | End-to-end trace |
| 31 | Free bidirectional edits | Нет master | Ping-pong | Field ownership |
| 32 | Государственная ИС недоступна | External outage | Stale/no service | Degraded mode |
| 33 | Коннектор предполагается | Непроверенная возможность | Blocked design | Official capability check |
| 34 | Канал уведомления не подтверждён | Assumption | Недоставка/нарушение | Channel policy |
| 35 | Identity check weak | Speed | Unauthorized disclosure | Approved verification |
| 36 | Attachment unsafe | No scanning/control | Malware/exposure | Secure intake |
| 37 | Retention отсутствует | Store forever | Compliance risk | Governed lifecycle |
| 38 | Hold нарушен | Auto-delete | Evidence loss | Legal/records hold |
| 39 | Real data in UAT | Prod copy | Privacy breach | Synthetic data |
| 40 | Migration без dry-run | Urgency | Mass corruption | Sample/report/rollback |
| 41 | Нет reconciliation | HTTP success trusted | Silent loss | Counts/control totals |
| 42 | BI смешивает grains | Dashboard-first | False KPI | Metric contract |
| 43 | Скорость = качество | Simplistic KPI | Harmful behavior | Balanced measures |
| 44 | Сотрудники ранжируются без сложности | Context ignored | Несправедливая оценка | Case mix/capacity |
| 45 | Peak load не проверен | Average design | Queue collapse | Load/degradation test |
| 46 | Тариф не проверен | Assumption | Feature unavailable | Portal assessment |
| 47 | Cloud ожидает DB/PHP | On-Premise habit | Impossible design | Cloud boundary |
| 48 | Core modified On-Premise | Quick fix | Upgrade/security failure | Extensions only |
| 49 | Release без rights tests | Happy path | Hidden exposure | Negative suite |
| 50 | Rollback не проверен | Paper plan | Long outage | Restore drill |
| 51 | Alert без owner | Notification only | Incident unresolved | RACI/runbook |
| 52 | Нет change register | Oral changes | Configuration drift | Versioned register |
| 53 | Training generic | No role scenarios | Workarounds/errors | Role-based UAT |
| 54 | DR проверяет только CRM | Integrations ignored | Service not restored | End-to-end DR |
| 55 | Нормативная применимость не подтверждена | Template copied | Неприемлемое решение | Legal/compliance review |

## 19. ADR

| ADR | Решение | Альтернатива | Обоснование | Последствие |
|---|---|---|---|---|
| ADR-GOV-001 | Citizen Request — отдельный Smart Process | Deal for every request | Несбытовой lifecycle и собственные сроки | Нужны связи с CRM profiles |
| ADR-GOV-002 | Contact/Company хранят минимальный профиль | Full citizen registry in CRM | Data minimization and reuse | External identity mapping |
| ADR-GOV-003 | Документ остаётся в СЭД/DMS | File copies in CRM | Version/legal record authority | Protected links/freshness |
| ADR-GOV-004 | Нормативный срок отделён от operational SLA | One deadline field | Разные основания и календари | Two governed clocks |
| ADR-GOV-005 | Integration through governed middleware | Point-to-point webhooks | Security/idempotency/monitoring | Platform owner required |
| ADR-GOV-006 | Права ограничены подразделением и назначением | Organization-wide read | Least privilege | More negative tests |
| ADR-GOV-007 | BI строится на reconciled events and grains | Raw CRM counters | Достоверность и explainability | Metric/DQ ownership |
| ADR-GOV-008 | Deal применяется только для реальной коммерческой возможности | Universal public-sector entity | Сохраняет семантику CRM | Discovery decision required |

Каждый ADR получает owner, status proposed/accepted/superseded, date, context, alternatives, consequences, portal/version and evidence.

## 20. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Нет Discovery | Сразу настраивают портал | Неверный scope | AS IS and stakeholders |
| 2 | Cloud/On-Premise не определён | Template design | Unsupported architecture | Capability matrix |
| 3 | Тариф/редакция не проверены | Assumption | Недоступная функция | Verify target portal |
| 4 | Не назначены process owners | Проект как IT-only | Нет решений | RACI |
| 5 | CRM названа официальным реестром | Scope confusion | Юридическая ошибка | Master-system decision |
| 6 | Все обращения — Leads | Universal lead pattern | Сервис смешан с квалификацией | Request Smart Process |
| 7 | Все обращения — Deals | Sales template | Искажённые стадии/KPI | Separate lifecycle |
| 8 | Контакт заменяет обращение | Экономия сущностей | История перезаписывается | Request per case |
| 9 | Компания создаётся на каждый отдел заявителя | Плохая дедупликация | Дубли | One organization plus relations |
| 10 | ФИО — единственный ключ | Нет external ID | False merge | Stable ID/review |
| 11 | Телефон — обязательный идентификатор | B2C habit | Исключение каналов | Approved identity model |
| 12 | Полное досье в Contact | Удобство | Excess data | Minimal profile |
| 13 | Source ID не хранится | UI-only setup | Дубли/нет сверки | External keys |
| 14 | Один статус для request/task/document | Flat model | Непонятный факт | Separate lifecycles |
| 15 | Стадии описывают действия | Every step as stage | Ambiguity | State-based stages |
| 16 | Слишком много стадий | Every task is stage | Low adoption | Tasks/checklists |
| 17 | Нет exception paths | Happy path only | Ручные обходы | Redirect/pause/reopen rules |
| 18 | Обжалование перезаписывает исходное обращение | One-card thinking | Утрата истории | Related case |
| 19 | Дубликаты удаляют автоматически | Aggressive matching | Evidence loss | Link/review policy |
| 20 | Обязательные поля слишком рано | Maximum control | Fake values | Stage requiredness |
| 21 | Категория свободным текстом | Нет справочника | Нет маршрутизации/BI | Governed dictionary |
| 22 | Причина закрытия в комментарии | No structured outcome | False analytics | Outcome codes |
| 23 | Нормативный срок — обычная дата | No source/rule | Неверная просрочка | Deadline basis/version |
| 24 | SLA и нормативный срок объединены | Terms confused | Bad escalation/reporting | Separate clocks |
| 25 | Пауза срока без основания | KPI gaming | Недостоверность | Approved reason/audit |
| 26 | Календарь не версионируется | Static assumption | Historical drift | Effective-dated calendar |
| 27 | Маршрутизация по ключевому слову без fallback | Overautomation | Wrong department | Controlled classifier |
| 28 | Передача закрывает ответственность отправителя | No acceptance | Lost request | Ack/handoff protocol |
| 29 | Нет следующего шага | Stage-only control | Stalled work | Task/activity policy |
| 30 | Робот дублирует задачи | Re-entry ignored | Fake workload | Idempotency marker |
| 31 | Роботы двигают стадии циклом | Mutual actions | Automation storm | Loop guard |
| 32 | Робот принимает правовое решение | Automation optimism | Wrongful response | Human gate |
| 33 | Ответ отправляется до approval | Race condition | Wrong version | Version lock |
| 34 | Закрытие по факту email | Delivery equals resolution | Incomplete case | Closure criteria |
| 35 | Уведомление содержит лишние ПДн | Template copy | Disclosure | Minimal template |
| 36 | Канал ответа не проверяется | Assumption | Недоставка | Channel preference/policy |
| 37 | Документы хранятся копиями | CRM as DMS | Version conflict | DMS references |
| 38 | Подпись — checkbox | No authoritative source | False legal status | E-sign verification |
| 39 | Регистрационный номер генерируется двумя системами | No master | Collision | Single authority |
| 40 | Вложение доступно всем участникам | Inherited rights | Data leak | File-specific policy |
| 41 | Вложения не контролируются | No security process | Malware/risk | Secure intake |
| 42 | Operator sees all departments | Broad role | Cross-unit leak | Queue scope |
| 43 | Analyst sees full request text | Raw BI | Excess disclosure | Pseudonymization |
| 44 | Admin equals content reader | Role confusion | Uncontrolled access | Separate privileges |
| 45 | Temporary access has no expiry | Manual grant | Residual access | Expiry/recertification |
| 46 | Rights tested only positively | Happy path | Hidden access | Negative matrix |
| 47 | Export unrestricted | Defaults | Mass leak | Approval/minimum |
| 48 | Real token in README | Convenient example | Compromise | Placeholder/secret store |
| 49 | Webhook committed | Quick integration | Unauthorized calls | Environment/rotation |
| 50 | One broad integration user | Simplicity | Large blast radius | Per-purpose scopes |
| 51 | Full payload logged | Debugging | PII/document leak | Masking |
| 52 | Retry has no limit | No failure design | Load storm | Bounded retry/DLQ |
| 53 | Retry creates duplicate request | No idempotency | Double work | Source event key |
| 54 | Event order ignored | Async not modeled | Old status wins | Version/order guard |
| 55 | Schema not validated | Trust source | Corrupt fields | Contract validation |
| 56 | No correlation ID | Fragmented tracing | Slow RCA | End-to-end ID |
| 57 | Free bidirectional editing | No field owner | Ping-pong | Ownership matrix |
| 58 | Government connector assumed | Marketing inference | Blocked release | Official verification |
| 59 | REST method invented | Guessing | Broken integration | API reference check |
| 60 | Batch used without method limits | Premature optimization | Partial failure | Official check/report |
| 61 | Migration has no inventory | Unknown scope | Missing records | Source/volume report |
| 62 | Import runs without dry-run | Deadline pressure | Mass corruption | Sample/dry-run |
| 63 | Real citizen data copied to UAT | Production clone | Privacy breach | Synthetic data |
| 64 | Mapping changes during cutover | No freeze | Inconsistent import | Approved version |
| 65 | No count/sample reconciliation | HTTP success trusted | Silent loss | Control report |
| 66 | BI counts tasks as requests | Wrong grain | False workload | Metric contract |
| 67 | Average hides long tail | Simplistic dashboard | Breaches invisible | Percentiles/cohorts |
| 68 | Speed is the only KPI | KPI pressure | Lower quality | Balanced measures |
| 69 | Rankings ignore case complexity | Context omitted | Harmful management | Segmentation/capacity |
| 70 | Dashboard has no freshness | Source outage hidden | False confidence | Freshness indicator |
| 71 | Peak intake not tested | Average load | Queue collapse | Load/degraded-mode test |
| 72 | Old fields deleted immediately | Cleanup | Broken dependencies | Deprecation scan |
| 73 | Release without smoke/rollback | Time pressure | Long incident | Checklist/drill |
| 74 | Cloud assumes server access | On-Premise habit | Impossible solution | External service boundary |
| 75 | On-Premise core modified | Quick fix | Upgrade/security failure | Extensions only |
| 76 | Monitoring checks only HTTP | Technical focus | Lost business events | Business signals |
| 77 | Alert has no owner/runbook | Notification-only | Incident unresolved | RACI/runbook |
| 78 | Alert contains request text | Full context copied | Disclosure | Minimal alert |
| 79 | No change register | Oral changes | Configuration drift | Versioned documentation |
| 80 | Done means configured | No evidence | Disputed result | Tested acceptance criteria |

## 21. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Подтверждённый AS IS и scope | Stakeholders, channels, systems, regulations, tariff, volumes | Facts/open questions approved |
| 2. Архитектура | Утверждённый TO BE | Entity model, master matrix, threat model, rights, ADR | Architecture/security review |
| 3. CRM настройка | Управляемые profiles and intake | Fields, stages, roles, automation register | Synthetic paths pass |
| 4. Обращения граждан | Request lifecycle and controls | Categories, deadlines, handoff, exceptions | End-to-end UAT passes |
| 5. Документы | Governed DMS relations | Metadata, versions, approvals, retention refs | Document/version tests pass |
| 6. Интеграции | Надёжный защищённый обмен | Contracts, secrets, retry, idempotency, monitoring | Security/failure/replay/reconcile pass |
| 7. Тестирование | Evidence-based quality | Test plan, synthetic data, rights/load/rollback tests | Critical defects closed/sign-off |
| 8. Запуск | Контролируемый release | Cutover, communication, backup/export, fallback, rollback | Smoke/reconciliation pass |
| 9. Оптимизация | Измеримое улучшение | Baseline, DQ, incidents and change backlog | KPI change verified |

### Критерии готовности

- подтверждены Cloud/On-Premise, тариф/редакция, версия, регион, права и тестовый контур;
- утверждены scope, master matrix, entity grains, нормативные основания, rights and ADR;
- документ, подпись и официальный реестр не подменяются CRM-полями;
- выполнены synthetic UAT, negative rights, integration failure, load, migration dry-run and reconciliation;
- секреты вынесены из Git, logs/alerts не содержат полного текста, документов и лишних персональных данных;
- утверждены cutover, rollback, DR, degraded mode, monitoring and support RACI;
- владельцы процесса, данных, безопасности и систем приняли критерии результата.

### Официальные источники

Использованы только записи из knowledge/SOURCES.md. Перед реализацией URL, дату обновления, тарифную/версионную применимость и фактический интерфейс проверяют повторно.

- [Официальная пользовательская документация Bitrix24](https://helpdesk.bitrix24.ru/) — CRM, задачи, права, automation and Cloud.
- [Официальная документация REST API](https://apidocs.bitrix24.ru/) — интеграции, авторизация и ограничения.
- [REST API reference](https://apidocs.bitrix24.ru/api-reference/) — методы, параметры and scopes.
- [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/) — официальный инструментарий.
- [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) — создание and configuration Smart Processes.
- [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) — штатная automation.
- [Бизнес-процессы](https://helpdesk.bitrix24.ru/open/23951294/) — выбор типа процесса.
- [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) — официальные способы подключения.
- [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) — datasets and limitations.
- [Маркетплейс Bitrix24](https://www.bitrix24.ru/apps/) — приложения с отдельной оценкой поставщика и безопасности.
- [Документация On-Premise](https://dev.1c-bitrix.ru/docs/portal.php) — коробочная разработка без изменения ядра.

### Ограничения и открытые вопросы

1. Cloud или On-Premise, тариф/редакция, версия, регион, права и доступность функций.
2. Нормативный статус организации, виды обращений, jurisdiction, сроки, архив и требования к защите.
3. Master гражданина, организации, обращения, документа, подписи, поручения, проекта и справочников.
4. Допустимость Contact/Company, Smart Processes, tasks, BI and telephony для конкретного класса данных.
5. Каналы приёма и ответа, identity verification, consent/legal basis and accessibility.
6. Государственные ИС, СЭД/ЭДО, сайты, телефония и BI: APIs, stable IDs, volumes, SLA, replay and support.
7. Права подразделений, контролёров, администраторов, временный доступ, export and recertification.
8. Правила classification, redirect, pause, handoff, closure, reopen, appeal and complaint.
9. BI grain, metric owners, calendars, source timestamps, control totals and DQ thresholds.
10. Test portal, release window, downtime, rollback, degraded mode, DR and support RACI.

Документ не подтверждает соответствие конкретному нормативному акту, классу защищённости, требованиям аттестации или закупки. Такие выводы требуют отдельной юридической и информационно-безопасностной экспертизы на актуальных первичных источниках.
