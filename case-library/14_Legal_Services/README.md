# Legal Services CRM and Matter Management Architecture Case

Архитектурный кейс для синтетической юридической компании, сопровождающей физических лиц и организации. Документ задаёт инженерную модель клиентского цикла, продаж, юридических дел, договоров, задач и аналитики; это не готовая конфигурация и не юридическая консультация.

> **Граница безопасности:** реальные юридические компании, доверители, оппоненты, сотрудники, дела, суды, номера производств, документы, реквизиты, телефоны, email, CRM ID, webhook URL и токены не используются. Все данные и идентификаторы синтетические.

> **Архитектурная граница:** Bitrix24 отвечает за CRM, коммерческую возможность, координацию обращения, задач и допустимую проекцию юридического дела. Утверждённая СЭД/ЭДО остаётся master подписанных документов и версий, 1С/ERP — учёта, счетов и оплат, специализированная legal/court system — процессуальных фактов, если она применяется. CRM не подменяет правовую экспертизу, официальный архив или систему подачи документов.

Обозначения:

- **факт Bitrix24** — подтверждается официальным источником из `knowledge/SOURCES.md`;
- **решение архитектора** — целевая модель кейса, принимаемая на Discovery;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа/редакции, версии, прав, региона, приложения или внешней системы.

## 1. Business Context

### Модель юридического бизнеса

| Аспект | Референсная модель |
|---|---|
| Клиенты | Физические лица, компании, группы компаний; карточки разделены по типу |
| Практики | Консалтинг, договорная, корпоративная, претензионная, судебная и абонентская работа |
| Продажа | Отдельная коммерческая возможность с объёмом, тарифной моделью и конфликт-проверкой |
| Исполнение | Юридическое дело с владельцем, командой, сроками, задачами и результатом |
| Документы | Версионные документы в утверждённой СЭД/хранилище, CRM хранит метаданные/ссылки |
| Финансы | Счета, оплаты и бухгалтерские факты подтверждает 1С/ERP |

### Цели CRM

1. Не терять обращения и историю разрешённых коммуникаций.
2. Разделить продажу услуги и исполнение юридического дела.
3. Назначить владельцев, сроки, контрольные точки и эскалации.
4. Обеспечить matter-centric доступ по принципу наименьших привилегий.
5. Связать договор, документы, задачи и финансы устойчивыми идентификаторами.
6. Получать сверяемую аналитику загрузки, сроков и экономики без раскрытия лишних данных.

```text
Channel -> Enquiry -> Conflict/eligibility check -> Deal -> Contract
                                                    |
                                                    v
Client <-> Legal Matter -> Tasks/Milestones -> Documents -> Outcome
              |                    |              |
              v                    v              v
          Consultation          Workload       DMS/EDO
              |
              +-----------------> Billing <---- 1C/ERP
```

## 2. AS IS Problems

| Проблема | Проявление | Причина | Целевой контроль |
|---|---|---|---|
| Нет единой базы | Дубли клиента в практиках | Нет identity policy | Stable IDs и очередь спорных совпадений |
| Теряется история | Переписка в личных каналах | Нет единого intake | CRM activity policy и owner |
| Нет контроля сроков | Дедлайны в личных календарях | Нет milestone model | Structured deadlines and escalation |
| Хаос документов | Файлы без версии и статуса | CRM используют как папку | DMS master, version metadata |
| Нет загрузки юристов | Задачи не нормированы | Нет capacity data | Planned/actual workload model |
| Смешаны продажа и дело | Одна сделка живёт годами | Нет границы объектов | Deal per opportunity, Matter per engagement |
| Широкий доступ | Практики видят чужие дела | Общие роли | Matter team and practice scope |
| Финансы расходятся | Суммы меняют вручную | Нет finance master | 1С/ERP reconciliation |
| Дедлайны зависят от текста | Срок в комментарии | Нет структурированного поля | Source, timezone, owner, reminders |
| Нет конфликта интересов | Клиент принят до проверки | Процесс не формализован | Gated review without exposing case substance |

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Лид | Неквалифицированное обращение, если lead model подтверждена | Bitrix24 |
| CRM | Контакт | Физическое лицо | Bitrix24/MDM по ADR |
| CRM | Компания | Юридическое лицо/клиент/контрагент | Bitrix24/ERP по ADR |
| CRM | Сделка | Коммерческая возможность по отдельной услуге | Bitrix24 |
| CRM | Обращение | Вопрос, сервисный запрос или жалоба | Bitrix24 |
| Smart Process | Юридическое дело | Matter lifecycle, команда, сроки, результат | Bitrix24/legal system по ADR |
| Smart Process | Договор | Координация согласования и ссылка на версию | СЭД/ЭДО |
| Smart Process | Консультация | Запись, исполнитель, результат | Bitrix24 |
| Smart Process | Судебный процесс | Допустимая operational projection | Legal/court system |
| Smart Process | Задача клиента | Запрос клиенту и контроль ответа | Bitrix24 |
| Smart Process | Претензия | Претензионный lifecycle | Bitrix24/legal system |

**Факт Bitrix24:** CRM, задачи, смарт-процессы, роботы, триггеры и бизнес-процессы описаны официальными материалами. Конкретные связи, права, поля и автоматизация проверяются на целевом портале.

**Решение архитектора:** чувствительное содержание минимизируется; предмет дела, документы и участники доступны только команде matter. Интеграционные записи содержат `source_system`, `external_id`, `source_updated_at`, `sync_status` и correlation ID.

```text
Website/Mail/Calls -> Intake -> Contact/Company -> Deal -> Contract
                         |             |             |
                         v             v             v
                    Conflict Gate   Legal Matter <-> DMS/EDO
                                         |
                   +---------------------+--------------------+
                   v                     v                    v
               Consultation          Tasks/Deadlines      Claim/Court projection
                   |                     |                    |
                   +---------------------+--------------------+
                                         v
                                  Billing projection <- 1C/ERP
                                         |
                                         v
                                   Governed BI
```

### Границы и целостность

- один master назначается для клиента, договора, версии документа, счёта, оплаты и процессуального статуса;
- повторное событие не создаёт дубликат клиента, сделки, matter, договора или задачи;
- конфликт-проверка предшествует раскрытию подробностей и началу работы;
- закрытая сделка не заменяет действующее юридическое дело;
- сроки имеют источник, календарь, timezone, owner, правило изменения и evidence;
- bulk-операции используют dry-run, sample, журнал, сверку и rollback.

## 4. Client Journey

| Этап | Цель | Минимальные данные | Gate/контроль |
|---|---|---|---|
| Обращение | Зарегистрировать запрос | Канал, минимальный контакт, тема верхнего уровня | Dedup, owner, SLA |
| Первичная консультация | Понять допустимый scope | Участники, категория, срочность | Conflict/eligibility gate |
| Оценка ситуации | Определить объём и риски | Assumptions, deliverables, deadlines | Responsible lawyer review |
| Предложение | Зафиксировать коммерческие условия | Scope/version/validity/price model | Approval trail |
| Договор | Получить утверждённое основание | Contract external ID, status, version | DMS/EDO confirmation |
| Ведение дела | Выполнить согласованный scope | Matter team, milestones, tasks, evidence refs | Rights and deadline control |
| Завершение | Зафиксировать результат и обязательства | Outcome code, client acceptance, open items | Closure checklist |
| Повторное обращение | Создать новый request/opportunity | New scope and relation to client | No overwrite of closed matter |

```text
Обращение -> Первичная консультация -> Оценка ситуации -> Предложение
 -> Договор -> Ведение дела -> Завершение -> Повторное обращение

Branches: conflict / out of scope / insufficient data / declined / paused /
terminated / appeal or follow-up as a separate governed lifecycle.
```

## 5. Legal Data Model

| Сущность | Минимальные поля | Контроль |
|---|---|---|
| Клиент | External ID, type, permitted contacts, responsible practice, communication preferences | No excessive sensitive narrative |
| Компания | External company ID, name, identifiers if justified, group relation | Name is not sole key |
| Дело | Matter ID, category, status, owner, team, start/end, milestones, confidentiality class | Matter-centric rights |
| Документ | Document external ID, type, version, status, owner, DMS link/checksum if approved | No uncontrolled duplicate file |
| Договор | Contract ID, client, scope reference, version, approval/signing status | DMS/EDO master |
| Консультация | Appointment ID, matter/client, lawyer, plan/fact, result code | Minimal notes |
| Задача | Matter ID, result, assignee, deadline, status, dependency | No deadline only in title |
| Счёт | Invoice external ID, contract/matter, amount, currency, status | 1С/ERP master |

```text
Client 1 ---- N Deal 1 ---- 0..1 Contract
   |             |                    |
   |             +--------------------+
   |                                  v
   +---- N Legal Matter 1 ---- N Task/Milestone
              |    |                    |
              |    +---- N Consultation |
              |                         v
              +---- N Document Ref -> DMS/EDO
              +---- N Invoice Ref  -> 1C/ERP
              +---- 0..N Claim/Court Projection
```

### Кардинальности

| Связь | Кардинальность | Правило |
|---|---:|---|
| Client → Matter | 1:N | Новый scope обычно отдельный matter |
| Matter ↔ Participant | M:N | Роль и сторона обязательны; доступ не наследуется автоматически |
| Deal → Matter | 1:0..N | Создание только после gate/договора по утверждённой модели |
| Matter → Document | 1:N | Метаданные/ссылки, версия в DMS |
| Matter → Task | 1:N | Каждая задача имеет проверяемый результат |
| Matter → Invoice | 1:N | Финансовый статус приходит из 1С/ERP |

## 6. Sales Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новый запрос | Зафиксировать спрос | Канал, тема, owner, received time | Routing/SLA/dedup |
| Квалификация | Проверить тип клиента и потребность | Client type, service category, next step | Task and escalation |
| Консультация | Собрать минимум для оценки | Participants, question, appointment result | Conflict gate task |
| Оценка дела | Определить scope/resources | Assumptions, deliverables, effort band, risks | Practice review |
| Предложение | Представить условия | Offer version, validity, fee model | Approval/follow-up |
| Согласование | Закрыть разногласия | Open items, decision owner, next date | Escalation |
| Договор | Подтвердить документ | Contract ID/version/status | DMS/EDO status check |
| Работа по делу | Передать исполнению | Matter ID/team/milestones | Create bounded onboarding tasks |
| Закрытие | Зафиксировать коммерческий итог | Outcome/loss reason, handoff status | Completeness check |

```text
Новый запрос -> Квалификация -> Консультация -> Оценка дела
 -> Предложение -> Согласование -> Договор -> Работа по делу -> Закрытие
```

Стадии отражают состояние, действия оформляются задачами. Повторный вход не должен повторять письмо, задачу или создание matter. Проигрыш имеет структурированную причину без раскрытия лишнего содержания.

## 7. Case Management Architecture

| Компонент | Содержание | Контроль |
|---|---|---|
| Matter | Отдельное поручение/дело | Unique matter ID, client, practice, owner |
| Статус | Intake/active/waiting/review/closed or approved variant | Entry/exit criteria |
| Milestone | Внешняя или внутренняя контрольная точка | Source, deadline, timezone, evidence |
| Team | Partner, lead lawyer, assistants | Explicit membership and review |
| Work item | Task with result and estimate | Dependency, assignee, plan/fact |
| Outcome | Structured closure result | Approval and open-obligation check |

```text
Matter Open -> Planning -> Active Work -> Internal Review -> Client/External Wait
       ^                                              |
       +---------------- controlled return -----------+
                                                      v
                                          Completion -> Archive/Follow-up
```

**Рекомендация архитектора:** судебный процесс выделяется только при самостоятельном lifecycle. Официальные даты и статусы не выводятся из текста автоматически без подтверждённого источника и ответственного.

## 8. Contract Management

| Шаг | CRM/Smart Process | СЭД/ЭДО | Контроль |
|---|---|---|---|
| Проект | Metadata, owner, scope link | Authoritative file/version | Template/version ID |
| Согласование | Route/status/tasks | Document review/signature workflow | Segregation of duties |
| Подписание | Status projection | Signature evidence/master | No manual signed flag without source |
| Изменение | New version/addendum relation | Authoritative version | Effective dates |
| Завершение | Obligations/status reference | Retention/archive | Open obligations check |

Договор не хранится одним перезаписываемым файлом. Номер, версия, стороны, effective date и статус подписания поступают из утверждённого источника; CRM не формирует юридический факт подписи самостоятельно.

## 9. Document Architecture

| Область | Разрешённая модель | Запрещённая практика |
|---|---|---|
| Документы клиента | Metadata/link to protected repository | Upload to public/shared folder |
| Шаблоны | Universal anonymized templates | Real party details in Git |
| Версии | Immutable version ID/status | Overwrite without history |
| Доступ | Matter team/need-to-know | Practice-wide access by default |
| Передача | Approved protected channel | Email/link without access/expiry control |
| Retention | Approved policy and legal hold process | Ad hoc deletion or endless storage |

```text
CRM Document Reference -> Access Check -> DMS Document/Version
        |                      |                 |
        v                      v                 v
Matter/Contract ID        Audit Event       Retention/Legal Hold
```

## 10. Lawyer Work Management

| Элемент | Правило | Метрика |
|---|---|---|
| Задача | Один результат, owner, deadline, matter ID | On-time completion |
| Загрузка | Planned capacity minus approved unavailable time | Planned/actual by period |
| Дедлайн | Source, timezone, reminder, escalation | At-risk/overdue count |
| Проверка качества | Reviewer and acceptance criteria | Rework/review time |
| Зависимость | Predecessor and blocking reason | Blocked duration |
| Делегирование | Explicit reassignment with handoff | Unowned work |

**Ограничение:** трудозатраты и загрузка достоверны только при единых правилах учёта. Метрики не используются как автоматическое заключение о качестве юриста без контекста, проверки данных и утверждённой методики.

## 11. Rights Model

| Роль | Клиенты | Дела | Документы | Финансы | Аналитика |
|---|---|---|---|---|---|
| Юрист | Assigned/authorized | Matter team | Authorized matter docs | Limited billing context | Personal/team minimum |
| Помощник юриста | Minimum assigned | Assigned tasks/matters | Need-to-know working docs | None by default | Operational minimum |
| Руководитель практики | Practice scope subject to ethical wall | Practice/approved matters | Review scope | Practice summary | Practice dashboard |
| Бухгалтер | Billing identity minimum | Matter reference only | Financial docs only | Create/read per process | Finance aggregates |
| Партнёр | Approved portfolio | Sponsor/oversight scope | As matter role permits | Approved commercial view | Governed portfolio |
| Администратор | Technical configuration | No automatic content entitlement | No automatic document entitlement | Configuration only | Dataset/config as required |

Принципы: deny-by-default, least privilege, explicit matter team, ethical walls, separation of duties, time-bound temporary access, periodic recertification и negative tests. Технический администратор не получает безусловное право читать содержание дел.

**Требует проверки:** фактическая гранулярность CRM/Smart Process rights, наследование, scopes, аудит и ограничения тарифа/редакции проверяются на целевом портале.

## 12. Automation Architecture

| Автоматизация | Trigger | Действие | Защита |
|---|---|---|---|
| Intake SLA | First valid request | Owner/task/timer | Idempotency marker |
| Conflict review | Minimum participants captured | Restricted review task | No broad disclosure |
| Deadline reminder | Approved milestone approaches | Lawyer/reviewer notification | Current version/status/timezone |
| Escalation | Threshold breached | Notify lead/practice owner | One event per threshold |
| Approval | Offer/contract requires review | Structured route | Version lock/audit |
| Client reminder | Awaiting approved client input | Permitted notification/task | Consent, purpose, suppression |
| Closure | All gates satisfied | Checklist/archive follow-up | Open task/document/billing checks |

```text
Event -> Validate rights/source/version -> Check idempotency -> Evaluate rule
 -> Create bounded action -> Record result -> Exception queue/escalation
```

Для каждого робота, триггера и бизнес-процесса фиксируются entity, event, conditions, fields, performer, deadline, repeat behavior, error handling, escalation и test case. Автоматизация не принимает юридическое решение.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| ЭДО/СЭД | Document ID/version/status/signing reference | Official connector/app or documented API/middleware | Idempotency, checksum/version, reconciliation |
| 1С | Clients, invoice/payment projections | Supported integration or governed middleware | Stable IDs, amounts/currency, totals |
| Телефония | Call events and CRM linkage | Bitrix24 telephony/SIP/REST app as applicable | Rights, missed-call queue, correlation |
| Сайт | Enquiries/forms | CRM form/official app/webhook/REST | Validation, consent, dedup, anti-abuse |
| Почта | Authorized correspondence/activity | Built-in connection or approved integration | Mailbox ownership, access, retention |
| BI | Governed datasets/aggregates | BI Constructor or approved ETL | Grain, masking, freshness, reconciliation |

```text
Source Event -> AuthN/AuthZ -> Schema validation -> Correlation/idempotency
 -> Master ownership -> Bitrix24 write or quarantine -> Reconciliation/alert
```

- секреты и webhook URL хранятся вне Git и маскируются в логах;
- retry ограничен и не повторяет необратимую операцию без idempotency key;
- неизвестные значения справочников попадают в exception queue;
- REST-методы, scopes и лимиты подтверждаются официальной документацией;
- Cloud не предоставляет server/PHP/DB access; On-Premise расширяется без изменения ядра.

## 14. Billing Architecture

| Объект | CRM | 1С/ERP | Контроль |
|---|---|---|---|
| Услуга | Commercial catalog/reference | Accounting nomenclature master as applicable | Stable service ID |
| Тарифная модель | Offer/contract metadata | Billing rules/master if automated | Version/effective date |
| Счёт | Request/status projection | Authoritative invoice | External invoice ID |
| Оплата | Read-only projection | Authoritative payment | Amount/currency/date reconciliation |
| Расход/себестоимость | Aggregate/approved BI only | Finance/time system | Access and allocation rules |
| Прибыльность | Governed metric | Reconciled sources | Formula/period/owner |

Bitrix24 не объявляется бухгалтерской master-system. Ручная отметка оплаты не является финансовым подтверждением. Вознаграждение, налоги и признание дохода определяются утверждённой финансовой методологией вне этого кейса.

## 15. Analytics

| Отчёт | Grain | Показатели | Контроль качества |
|---|---|---|---|
| Количество дел | One matter | Opened/active/closed by category/practice | Status dictionary |
| Прибыльность клиентов | Client + period | Reconciled revenue/cost/margin | Finance owner/formula |
| Загрузка юристов | Lawyer + period + work item | Capacity, planned, actual, overdue | Time/capacity completeness |
| Скорость обработки | Request/matter milestone | First response, cycle time, wait time | Calendar/pause policy |
| Эффективность каналов | Qualified request/opportunity | Volume, conversion, cost if available | Attribution window |
| Качество исполнения | Matter outcomes/rework | Timeliness, review/rework indicators | No context-free ranking |

```text
CRM + Tasks + DMS metadata + 1C/ERP
              -> Conformed IDs -> Quality/Reconciliation
              -> Governed Metric Layer -> Role-based Dashboard
```

Каждая метрика имеет owner, formula, grain, time window, timezone, currency, exclusions, freshness SLA и control total. Доступность BI Конструктора и datasets зависит от тарифа и версии.

## 16. Security Architecture

| Контроль | Реализация | Проверка |
|---|---|---|
| Конфиденциальность | Data classification and minimization | Field/file inventory |
| Matter access | Explicit team and ethical wall | Cross-matter negative tests |
| Practice separation | Role/scope plus exceptions | Cross-practice tests |
| Документы | Protected DMS link and access check | Revoked/expired access test |
| Аудит | CRM history + masked integration logs | Trace by correlation ID |
| Экспорт | Approval, minimum columns, protected target | Export role/sample review |
| Secrets | Environment/secret store and rotation | Secret scan/revocation drill |
| Retention/legal hold | Approved policy and controlled exceptions | Periodic evidence review |

**Cloud:** доступны только штатные права, история и интеграционные журналы; нет доступа к ядру, серверному PHP и БД. **On-Premise:** дополнительно защищаются сервер, PHP, БД, файлы, cron, backup и журналы; кастомизация ядра запрещена.

## 17. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Обращение с сайта | Валидные synthetic fields | Одна запись, owner и SLA |
| 2 | Повтор webhook | Тот же event ID | Дубликат не создан |
| 3 | Невалидный контакт | Пустое обязательное значение | Controlled error/quarantine |
| 4 | Конфликт-проверка | Synthetic overlapping participant | Restricted review, no broad disclosure |
| 5 | Квалификация | Next step and category | Переход разрешён, одна задача |
| 6 | Создание предложения | Version/validity present | Approval route started once |
| 7 | Договор не подписан | Нет source confirmation | Matter start gate blocks/flags per policy |
| 8 | Создание matter | Approved deal/contract | Unique matter linked to client |
| 9 | Deadline | Timezone/calendar boundary | Correct reminder/escalation |
| 10 | Повтор входа в статус | Re-entry | Duplicate task/message absent |
| 11 | Версия документа | New DMS version | Metadata updated, history preserved |
| 12 | Права другого matter | Unauthorized lawyer | Content and files unavailable |
| 13 | Ethical wall | User in another practice | Restricted matter unavailable |
| 14 | Счёт | 1С authoritative event | One projection with correct amount/currency |
| 15 | Ошибка интеграции | Timeout after accepted write | Retry does not duplicate |
| 16 | Закрытие дела | Open critical task/document | Closure blocked/escalated |
| 17 | Повторное обращение | Existing client, new scope | New request/matter; history preserved |
| 18 | Export | User without approval | Export denied or minimized |
| 19 | BI reconciliation | Synthetic control totals | Difference within threshold or incident |
| 20 | Rollback | Failed release step | Verified prior state restored |

Тестовые данные синтетические; реальные клиенты, стороны, факты, документы и контакты запрещены. Проверяются happy path, negative, boundary, concurrency, retry, rights, audit, restore и Cloud/On-Premise differences.

## 18. Deployment

| Этап | Результат | Gate |
|---|---|---|
| Discovery | AS IS, scope, systems, volumes, confidentiality classes | Business/security approval |
| Архитектура | TO BE, master matrix, rights, ADR, NFR | Architecture review |
| Настройка | CRM/Smart Processes/automation in test scope | Configuration review |
| Перенос данных | Inventory/map/cleanse/dry-run/report | Count/sample reconciliation |
| Интеграции | Contracts, secrets, retry, monitoring | Failure/replay tests |
| Обучение | Role-based training without real data | UAT sign-off |
| Запуск | Runbook, window, support and rollback | Go/no-go |
| Stabilization | Incident and metric baseline | Exit criteria |

```text
Inventory -> Dependency review -> Backup/export -> Test configuration
 -> Synthetic tests -> UAT -> Migration dry-run -> Go/No-Go
 -> Release -> Smoke/reconciliation -> Hypercare or Rollback
```

Действующие поля, стадии, права, роботы и интеграции не удаляются/переименовываются до dependency analysis, backup/export, пилота и согласованного rollback.

## 19. Monitoring

| Сигнал | Правило | Действие | Владелец |
|---|---|---|---|
| Пропущенное обращение | Intake/SLA threshold | Alert and assign | Client intake owner |
| Просроченное дело | Milestone breached | Escalate to matter lead | Practice owner |
| Документ без статуса | DMS freshness exceeded | Exception/reconcile | Document owner |
| Задача без владельца | Unassigned/open | Assign/escalate | Matter lead |
| Integration error | Error/retry/DLQ threshold | Incident/replay after fix | Integration support |
| Rights anomaly | Unexpected access/denial | Security incident/review | Security owner |
| Billing mismatch | CRM vs 1С control total | Finance reconciliation | Finance owner |
| BI freshness | Dataset SLA exceeded | Mark stale and investigate | BI owner |

Alert содержит system, synthetic/object reference, correlation ID, timestamp, severity, owner и runbook; секреты и содержание дела в уведомление не включаются.

## 20. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Ложное объединение клиентов | Слабые ключи | Раскрытие чужого дела | Stable IDs/manual review |
| 2 | Дубли клиентов | Разные каналы | Разорванная история | Normalization/dedup queue |
| 3 | Потеря обращения | Нет intake ack | Упущенный запрос | SLA/reconciliation |
| 4 | Дело принято до conflict check | Нет gate | Конфликт/репутационный риск | Restricted mandatory gate |
| 5 | Conflict check раскрывает детали | Широкая рассылка | Нарушение конфиденциальности | Minimal participants/restricted reviewers |
| 6 | Сделка заменяет дело | Неверная модель | Потеря исполнения | Deal/Matter separation |
| 7 | Одно matter на все услуги | Упрощение | Смешение сроков/прав | Matter per engagement/scope |
| 8 | Широкий доступ практики | Общая роль | Утечка | Explicit matter team |
| 9 | Admin читает всё | Role confusion | Неконтролируемый доступ | Separate technical/content rights |
| 10 | Временный доступ не отозван | Нет expiry | Остаточные права | Time-bound access/recertification |
| 11 | Документ в общей папке | Удобство | Раскрытие | Protected DMS |
| 12 | Версия перезаписана | Нет versioning | Использован неверный документ | Immutable version ID |
| 13 | Неверный signed status | Ручная отметка | Ошибочное действие | EDO/DMS confirmation |
| 14 | Срок в комментарии | Нет поля | Пропуск дедлайна | Structured milestone |
| 15 | Неверная timezone | Несколько регионов | Просрочка | Explicit timezone/calendar |
| 16 | Изменён источник срока | Нет audit | Неясная ответственность | Source/evidence/version history |
| 17 | Напоминание повторяется | Re-entry | Шум/ошибка | Idempotency marker |
| 18 | Автоматизация решает юридический вопрос | Scope creep | Неверное решение | Human approval boundary |
| 19 | Закрытие с открытыми задачами | Нет gate | Невыполненные обязательства | Closure checklist |
| 20 | Удаление при legal hold | Нет retention process | Потеря evidence | Approved hold/retention policy |
| 21 | Бессрочное хранение | Нет owner | Избыточный риск | Retention schedule |
| 22 | Реальные дела в тестах | Копия production | Data breach | Synthetic data only |
| 23 | Секрет в Git | Ошибка интегратора | Компрометация | Secret store/scan/rotation |
| 24 | Raw payload in logs | Debug shortcut | PII/confidentiality leak | Masked logs |
| 25 | Широкий REST token | Excess scopes | Большой blast radius | Least privilege |
| 26 | Retry without idempotency | Timeout ambiguity | Дубли | Source event ID |
| 27 | Infinite retry | Нет DLQ | Нагрузка/шторм | Bounded backoff |
| 28 | Event out of order | Async delivery | Старый статус победил | Source version/order control |
| 29 | Schema drift | Source changed | Corrupt data | Contract versioning |
| 30 | Нет correlation ID | Разные журналы | Долгая диагностика | End-to-end trace |
| 31 | 1С и CRM редактируют оплату | Нет master | Финансовый конфликт | Finance master matrix |
| 32 | Суммы без валюты | Упрощение | Неверная аналитика | Currency/date required |
| 33 | Прибыльность без методики | Неполные затраты | Ложный KPI | Finance-owned formula |
| 34 | Загрузка по числу задач | Разный размер работ | Неверные решения | Estimate/capacity model |
| 35 | KPI ранжирует без контекста | Автоматизация оценки | Демотивация/ошибка | Governed contextual metrics |
| 36 | Экспорт raw client data | Широкие права | Утечка | Minimum export/approval |
| 37 | Почта подключена общим ящиком | Нет ownership | Чужая переписка | Mailbox/access policy |
| 38 | Дубли задач | Robot re-run | Ложная загрузка | Idempotent task key |
| 39 | Нет replacement handoff | Смена юриста | Unowned deadlines | Controlled reassignment |
| 40 | Просрочка скрыта статусом waiting | Нет pause rules | Недостоверный SLA | Defined pause/reason |
| 41 | Тариф не проверен | Design by assumption | Функция недоступна | Capability assessment |
| 42 | Cloud treated as server | Ожидается DB/PHP | Нереализуемость | Cloud boundary |
| 43 | Изменено ядро On-Premise | Quick fix | Update/security failure | Extension mechanisms |
| 44 | Миграция без dry-run | Срочность | Массовая ошибка | Sample/report/rollback |
| 45 | Название компании как key | Нестабильность | Дубли/ошибочные связи | External IDs |
| 46 | Нет reconciliation | Успешный HTTP считается успехом | Тихая потеря | Counts/control totals |
| 47 | Release без rights tests | Happy path only | Раскрытие | Negative access suite |
| 48 | Нет rollback | Успех предполагается | Длительный простой | Tested rollback |
| 49 | Alert содержит дело | Полный payload | Confidentiality leak | Minimal alert payload |
| 50 | Нет support owner | Размытая RACI | Долгий incident | Runbook/on-call |
| 51 | Нет baseline | AS IS не измерен | Эффект не доказан | Baseline/acceptance criteria |
| 52 | Нет change register | Конфигурация меняется устно | Неподдерживаемость | Versioned register |
| 53 | Интеграция ЭДО не проверяет версию | Status only | Неверный документ | Document/version ID |
| 54 | Участники дела видны всем | Поля без ограничений | Раскрытие сторон | Minimize/scope fields |
| 55 | Архив принимается за backup | Термины смешаны | Невозможное восстановление | Separate retention/backup plans |

## 21. ADR

| ADR | Решение | Альтернатива | Обоснование | Последствие |
|---|---|---|---|---|
| ADR-LEGAL-001 | Contact/Company — базовая модель клиента | Один Smart Process «Клиент» | Использовать штатные CRM-сущности | Нужна identity/group policy |
| ADR-LEGAL-002 | Юридическое дело отделено от Deal | Вести исполнение в сделке | Разные lifecycle, права и метрики | Нужна надёжная связь Deal–Matter |
| ADR-LEGAL-003 | DMS/EDO master файла и версии | Хранить authoritative files in CRM | Version/signature/retention belong to document system | CRM stores protected refs |
| ADR-LEGAL-004 | Contract is versioned projection linked to DMS | Contract fields only in Deal | Separate approval/effective dates | Integration/reconciliation required |
| ADR-LEGAL-005 | Rights are matter-centric with ethical walls | Practice-wide access | Least privilege/confidentiality | Explicit team management needed |
| ADR-LEGAL-006 | Tasks represent executable work, milestones represent deadlines | One generic activity list | Clear ownership and escalation | Registers/dictionaries required |
| ADR-LEGAL-007 | Billing facts come from 1С/ERP | Manual CRM payment status | Financial authority/reconciliation | Read-only financial projection |
| ADR-LEGAL-008 | BI uses governed aggregate datasets | Raw CRM export per analyst | Confidentiality and consistent metrics | Metric/data-quality ownership |

ADR получает статус `proposed/accepted/superseded`, owner, date, context, options, consequences, portal/version и evidence of validation.

## 22. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Нет Discovery | Сразу настраивают поля | Неверный scope | AS IS/stakeholders |
| 2 | Не определён deployment type | Шаблонный дизайн | Недоступные механизмы | Cloud/On-Premise assessment |
| 3 | Не проверен тариф | Предположение | Blocked release | Verify target portal |
| 4 | CRM названа legal DMS | Scope confusion | Потеря версий/контролей | Architecture boundary |
| 5 | Все обращения — лиды | Lead used universally | Сервис смешан с продажами | Request classification |
| 6 | Лиды отключены без ADR | Мода | Unqualified demand mixed | Lead model decision |
| 7 | Физлица и компании смешаны | Одно поле клиента | Ошибочные связи | Contact/Company model |
| 8 | Компания создаётся на каждую практику | Организационный shortcut | Дубли | One client identity + relations |
| 9 | Название — integration key | Нет external ID | Wrong match | Stable source IDs |
| 10 | Телефон — sole dedup key | Удобство | False merge | Conservative matching |
| 11 | Conflict check after contract | Нет gate | Engagement risk | Early restricted check |
| 12 | Conflict notes доступны всем | Общая задача | Disclosure | Restricted review object |
| 13 | Сделка используется как matter | Один объект проще | Нет execution lifecycle | Separate Smart Process |
| 14 | Один matter на все поручения | Экономия | Mixed rights/deadlines | Matter per scope |
| 15 | Matter на каждую задачу | Granularity error | Entity explosion | Task for work item |
| 16 | Судебный процесс всегда отдельный | Copy template | Unneeded complexity | Create only if lifecycle warrants |
| 17 | Стадии — список действий | Process misunderstanding | Ambiguous state | State-based pipeline |
| 18 | Слишком много стадий | Every task is stage | Low usability | Tasks/checklists |
| 19 | Mandatory fields too early | Maximum control | Fake data | Stage-based requiredness |
| 20 | Loss reason in comment | Нет dictionary | No analytics | Governed codes |
| 21 | Next step absent | Stage-focused process | Stalled requests | Activity policy |
| 22 | Contract status changed manually | No EDO integration | False signed state | Source confirmation |
| 23 | Contract file overwritten | No version model | Wrong terms | Immutable version refs |
| 24 | Addendum replaces contract | Simplification | Lost history | Related version/effective dates |
| 25 | Documents stored in shared drive | Convenience | Unauthorized access | Protected DMS |
| 26 | Files copied into comments | Fast sharing | No version/access control | DMS links |
| 27 | Template contains client data | Reused real file | Data leak | Anonymized templates |
| 28 | Deadline only in task title | No structured field | Missed reminder | Date/source/timezone fields |
| 29 | Deadline has no owner | Team responsibility | Nobody acts | Single accountable owner |
| 30 | Timezone omitted | Regional work | Wrong deadline | Explicit timezone |
| 31 | Calendar pauses undefined | SLA shortcut | False breach | Calendar/pause dictionary |
| 32 | Re-entry duplicates reminder | Robot stateless | Noise/errors | Idempotency marker |
| 33 | Automation makes legal conclusion | Overautomation | Wrong decision | Human approval |
| 34 | Robot loops stages | Mutual actions | Automation storm | Loop guard |
| 35 | Error catch is empty | Hide failures | Silent loss | Explicit handling/logging |
| 36 | Task has no result criteria | Activity counting | Unverifiable work | Acceptance criteria |
| 37 | Workload = task count | Different effort ignored | Bad planning | Estimate/capacity |
| 38 | Actual effort never captured | Process burden | Unreliable economics | Minimal governed time data |
| 39 | Delegation loses context | Reassign only | Deadline risk | Handoff checklist |
| 40 | Quality review is optional | No gate | Defect reaches client | Reviewer/acceptance rule |
| 41 | Lawyer sees all matters | Broad role | Confidentiality incident | Matter team rights |
| 42 | Assistant inherits all practice rights | Hierarchy assumption | Excess access | Explicit minimum role |
| 43 | Accountant reads matter content | Shared entity access | Excess disclosure | Finance-only projection |
| 44 | Admin equals super-reader | Technical role confusion | Uncontrolled access | Separate privileges |
| 45 | Temporary access never expires | Manual granting | Residual rights | Expiry/recertification |
| 46 | Rights tested only positively | Happy path | Hidden cross-access | Negative tests |
| 47 | Export unrestricted | Default permission | Mass leak | Approval/minimum columns |
| 48 | Secrets in README | Example becomes real | Credential compromise | Placeholders/secret store |
| 49 | Webhook in source | Convenience | Unauthorized calls | Environment/rotation |
| 50 | Integration user overprivileged | One token for all | Large blast radius | Least privilege |
| 51 | Raw payload logged | Debugging | Confidential data leak | Masked structured logs |
| 52 | Retry creates duplicate matter | No idempotency | Duplicate work | Source event key |
| 53 | Infinite retry | No stop policy | Load/outage | Bounded backoff/DLQ |
| 54 | Out-of-order event overwrites status | Async ignored | Stale data | Version/order guard |
| 55 | Schema changes unnoticed | No contract tests | Corrupt fields | Schema validation |
| 56 | No correlation ID | Fragmented logs | Slow RCA | End-to-end trace |
| 57 | Two systems edit payment | No master | Financial mismatch | 1С/ERP authority |
| 58 | Manual paid checkbox | Convenience | False payment | Read-only projection |
| 59 | Currency/date omitted | Local assumption | Wrong report | Required finance dimensions |
| 60 | Profitability from Deal amount | Incomplete cost/revenue | Misleading KPI | Reconciled finance model |
| 61 | BI grain undefined | Rows mixed | Double counting | Metric contract |
| 62 | Lawyer ranking by speed only | Context ignored | Harmful incentives | Balanced reviewed metrics |
| 63 | Data quality unmeasured | Dashboard-first | False confidence | DQ controls |
| 64 | Migration without inventory | Unknown sources | Missing data | Source/volume report |
| 65 | Import without dry-run | Deadline pressure | Mass corruption | Sample/dry-run |
| 66 | Real cases used in UAT | Easy dataset | Confidentiality breach | Synthetic data |
| 67 | No count/sample reconciliation | HTTP success trusted | Silent loss | Control report |
| 68 | Old fields removed immediately | Cleanup | Broken dependencies | Deprecation/dependency scan |
| 69 | Release without smoke | No window | Defect in production | Release checklist |
| 70 | Rollback untested | Plan on paper | Long outage | Restore drill |
| 71 | Monitoring only technical | API up considered success | Requests still lost | Business controls |
| 72 | Alert contains document title/content | Full payload | Disclosure | Minimal identifiers |
| 73 | Alert has no owner | Notification-only | Unresolved incident | RACI/runbook |
| 74 | Cloud assumed to expose DB | On-Premise habits | Impossible design | Cloud boundary |
| 75 | On-Premise core modified | Quick fix | Update/security failure | Extensions/modules |
| 76 | No change register | Oral configuration | Hard support | Versioned documentation |
| 77 | No role-based training | Generic demo | Workarounds | Role scenarios/UAT |
| 78 | No hypercare | Immediate handoff | Incident backlog | Stabilization period |
| 79 | Archive treated as backup | Terms confused | Failed recovery | Separate backup/retention |
| 80 | Done means configured | No acceptance evidence | Disputed outcome | Measurable tested criteria |

## 23. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Подтверждённый AS IS/scope | Stakeholders, practices, portals, tariff, volumes, confidentiality | Facts/open questions approved |
| 2. Архитектура | Утверждённый TO BE | Entity model, master matrix, rights, NFR, ADR | Architecture/security review passed |
| 3. CRM настройка | Governed intake/sales | Fields, stages, roles, automation register | Synthetic positive/negative tests pass |
| 4. Дела | Matter lifecycle | Statuses, team, milestones, tasks, closure | End-to-end matter UAT passes |
| 5. Документы | Governed DMS references | Version/status/access/retention mappings | Version and access tests pass |
| 6. Интеграции | Reliable exchange | Contracts, secrets, retry, idempotency, monitoring | Failure/replay/reconciliation pass |
| 7. Тестирование | Evidence-based quality | Test plan, synthetic data, rights suite, UAT | Critical defects closed/sign-off |
| 8. Запуск | Controlled production release | Cutover, backup/export, rollback, support roster | Smoke/reconciliation pass |
| 9. Оптимизация | Measured improvement | Baseline, DQ/incident/change backlog | KPI change verified and approved |

### Критерии готовности целевого проекта

- подтверждены Cloud/On-Premise, тариф/редакция, версия, регион и права;
- утверждены master-system matrix, Matter/Deal boundary, ethical walls and ADR;
- проверены права каждой роли позитивными и негативными тестами;
- выполнены synthetic tests, UAT, migration dry-run and reconciliation;
- секреты вынесены из Git, logs/alerts не содержат конфиденциального содержания;
- утверждены release, rollback, monitoring, incident, retention and support procedures;
- бизнес-владельцы приняли измеримые критерии результата.

### Официальные источники

Использованы только записи из `knowledge/SOURCES.md`; перед реализацией URL, дата обновления и применимость проверяются повторно.

- [Официальная пользовательская документация Bitrix24](https://helpdesk.bitrix24.ru/) — CRM, задачи, права, автоматизация и Cloud-возможности.
- [Официальная документация REST API](https://apidocs.bitrix24.ru/) — интеграции, авторизация и ограничения.
- [REST API reference](https://apidocs.bitrix24.ru/api-reference/) — точные методы, параметры и scopes.
- [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/) — официальный интеграционный инструментарий.
- [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) — создание и настройка Smart Processes.
- [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) — штатная автоматизация.
- [Бизнес-процессы](https://helpdesk.bitrix24.ru/open/23951294/) — выбор типа процесса.
- [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) — официальные варианты подключения.
- [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) — официальные datasets и ограничения.
- [Маркетплейс Bitrix24](https://www.bitrix24.ru/apps/) — опубликованные приложения с отдельной оценкой поставщика и безопасности.
- [Документация On-Premise](https://dev.1c-bitrix.ru/docs/portal.php) — коробочная разработка без изменения ядра.

### Ограничения и открытые вопросы

1. Cloud или On-Premise, тариф/редакция, версия, регион и доступные права.
2. Практики, типы поручений, лидовая модель и правила открытия matter.
3. Master клиентских данных, документов/версий, счетов/оплат и процессуальных статусов.
4. Требуется ли отдельная legal/court system и какие данные допустимо проектировать в CRM.
5. Политика conflict check, ethical walls, временного доступа и recertification.
6. Классификация конфиденциальности, retention, legal hold, archive/delete и audit requirements.
7. Сервисные календари, timezone, источники сроков, pause/escalation rules.
8. СЭД/ЭДО/1С API, owners, stable IDs, SLA, объёмы и rollback/replay capabilities.
9. Методика загрузки, прибыльности и KPI, owners метрик и quality thresholds.
10. Test portal/pipeline, release window, допустимый простой, rollback и support RACI.
