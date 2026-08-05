# Insurance Customer Lifecycle CRM Architecture Case

Архитектурный кейс для синтетической страховой компании, управляющей продажами, клиентскими обращениями, полисами, пролонгациями и сопровождением страховых случаев. Это референсная инженерная модель, а не готовая конфигурация, страховой расчёт или правовая рекомендация.

> **Граница безопасности:** реальные страховщики, страхователи, застрахованные лица, выгодоприобретатели, агенты, договоры, полисы, страховые случаи, медицинские или платёжные сведения, телефоны, email, CRM ID, webhook URL и токены не используются. Все примеры синтетические.

> **Архитектурная граница:** Bitrix24 отвечает за CRM, коммерческие возможности, обращения и координацию клиентского цикла. Core insurance/policy administration system остаётся master полиса и покрытия; rating engine — расчёта и тарифа; claims system — решения и выплаты; ЭДО — подписанных документов; 1С/ERP — финансовых фактов. CRM не принимает underwriting/claims decisions и не подменяет профильные системы.

Обозначения:

- **факт Bitrix24** — подтверждён официальным источником из `knowledge/SOURCES.md`;
- **решение архитектора** — проектная рекомендация, утверждаемая на Discovery;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа/редакции, версии, прав, региона, приложения или внешней системы.

## 1. Business Context

### Модель страхового бизнеса

| Аспект | Референсная модель |
|---|---|
| Клиенты | Физические лица, компании и связанные участники с разными ролями |
| Направления | Имущественные, личные, корпоративные и иные утверждённые продуктовые линии |
| Каналы | Сайт, contact center, агенты, офисы, партнёры и разрешённые коммуникации |
| Продажа | Отдельная opportunity: подбор, расчёт, предложение, договор |
| Полис | Authoritative record в policy administration system |
| Страховой случай | Регистрация/коммуникация в CRM; решение и выплата в claims system |

### Цели CRM

1. Не терять обращения и назначать владельца следующего шага.
2. Создать допустимый единый профиль взаимодействий без ложного объединения.
3. Связать заявку, расчёт, договор, полис, обращение и пролонгацию stable IDs.
4. Разделить права продаж, урегулирования, юристов, финансов и управления.
5. Автоматизировать контроль SLA и сроков без автоматического страхового решения.
6. Получать сверяемую аналитику продаж и пролонгаций.

```text
Channels -> Enquiry -> Product selection -> Rating/Eligibility -> Offer
                                                        |
                                                        v
Client <-> Deal -> Contract -> Policy -> Service/Renewal
                               |
                               +-> Claim intake -> Claims system -> Decision
```

## 2. AS IS Problems

| Проблема | Проявление | Причина | Целевой контроль |
|---|---|---|---|
| Разрозненные клиенты | Каналы создают дубли | Нет identity policy | Stable IDs/manual review |
| Потеря обращений | Нет owner/SLA | Intake не унифицирован | Ack, routing, escalation |
| Пропущенные пролонгации | Срок хранится вручную | Нет authoritative expiry signal | Policy event + eligibility check |
| Нет единой истории | Продажи и урегулирование раздельны | Нет governed relations | Client–policy–case links |
| Слабая аналитика | Разный grain/справочники | Нет metric contracts | Governed datasets |
| Неверные расчёты | CRM копирует тариф | Не назначен master | Rating engine authority |
| Ложный статус полиса | Ручное поле | Нет policy integration | Read-only projection/reconcile |
| Claims в комментариях | Нет lifecycle | CRM подменяет claims system | Structured intake/projection |
| Широкие права | Подразделения видят лишнее | Общие роли | Function/portfolio scopes |
| Дубли интеграций | Retry создаёт записи | Нет idempotency | Source event ID/upsert |

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Лид | Неквалифицированное обращение, если lead model подтверждена | Bitrix24 |
| CRM | Контакт | Физическое лицо | Bitrix24/MDM по ADR |
| CRM | Компания | Юридическое лицо/партнёр | Bitrix24/MDM/ERP |
| CRM | Сделка | Коммерческая возможность | Bitrix24 |
| CRM | Обращение | Вопрос, запрос или жалоба | Bitrix24 |
| Smart Process | Страховая заявка | Координация application lifecycle | Core insurance |
| Smart Process | Страховой продукт | Управляемая проекция каталога | Product/rating system |
| Smart Process | Полис | Read-only operational projection | Policy administration |
| Smart Process | Страховой случай | Intake/status projection | Claims system |
| Smart Process | Пролонгация | Renewal opportunity/control | Bitrix24 + policy source |
| Smart Process | Претензия | Клиентская претензия/SLA | Bitrix24/legal system |

**Факт Bitrix24:** CRM, Smart Processes, роботы, триггеры и бизнес-процессы описаны официальной документацией. Доступность конкретных полей, связей, прав и автоматизации проверяется на целевом портале.

**Решение архитектора:** интеграционные записи содержат `source_system`, `external_id`, `source_updated_at`, `sync_status` и correlation ID. CRM не редактирует свободно premium, coverage, policy validity, claim decision или payment facts.

```text
Website/Calls/Agents -> Lead/Enquiry -> Contact/Company -> Deal
                              |                              |
                              v                              v
                        Application -> Rating Engine -> Offer/Contract
                              |                              |
                              +--------------------------> Policy <- Core Insurance
                                                              |
                                 +----------------------------+----------------+
                                 v                                             v
                           Renewal Deal                                Claim/Complaint
                                                                                |
                                                                                v
                                                                          Claims System

BI <- reconciled datasets <- Bitrix24 + Core/Claims/1C
```

### Правила целостности

- master назначается для клиента, продукта, тарифа, расчёта, договора, полиса, claim decision, выплаты и документа;
- роль лица в конкретном полисе хранится отдельно от общего CRM-профиля;
- повтор события не создаёт дубликат application, policy projection, claim или renewal;
- новая версия/endorsement не перезаписывает историю полиса;
- bulk-операции имеют dry-run, sample, отчёт, reconciliation и rollback;
- автоматизация не делает вывод о риске, покрытии или выплате.

## 4. Customer Journey

| Этап | Цель | Минимальные данные | Контроль |
|---|---|---|---|
| Обращение | Зафиксировать интерес | Канал, topic, минимальный контакт | Consent/dedup/owner/SLA |
| Подбор продукта | Сопоставить потребность и каталог | Product line, customer type | Current product version |
| Расчёт условий | Получить authoritative quote | Rating request ID, inputs reference | Rating engine response/version |
| Заявка | Оформить намерение | Application ID/status, participants | Validation/eligibility gate |
| Согласование | Обработать решение профильной системы/специалиста | Decision reference, open items | Human/system authority |
| Договор | Зафиксировать согласованные условия | Contract ID/version/status | EDO/core confirmation |
| Полис | Получить authoritative policy | Policy ID/version/term/status | Core reconciliation |
| Сопровождение | Обрабатывать запросы | Policy/customer relation, case SLA | Least privilege |
| Продление | Создать новую opportunity | Expiry/eligibility/consent | No automatic promise |

```text
Обращение -> Подбор продукта -> Расчёт условий -> Заявка -> Согласование
 -> Договор -> Полис -> Сопровождение -> Продление

Branches: invalid / duplicate / declined by authoritative process / cancelled /
expired / changed / claim / complaint.
```

## 5. Insurance Data Model

| Сущность | Минимальные поля | Контроль |
|---|---|---|
| Клиент | External ID, type, permitted contacts, segment, preferences | Data minimization/identity review |
| Участник | Client ID, role, effective dates, policy/application ID | Role is contextual |
| Продукт | Product/version ID, category, effective dates, status | Product system master |
| Расчёт | Quote ID/version, product ref, amount/currency, validity | Rating engine master |
| Полис | Policy ID/version, product, term, premium projection, status | Core master/read-only |
| Страховой случай | Claim ID, event reference, status, owner, dates | Claims master/minimal narrative |
| Документ | External document ID, type, version, status, protected link | EDO/DMS master |
| Платёж | External payment ID, policy/invoice ref, amount/currency/status | 1С/ERP master |

```text
Customer N ---- M Policy Participant M ---- 1 Policy ---- 1 Product Version
   |                                                   |
   +---- N Deal/Application ---- N Quote               +---- N Endorsement/Version
   |                                                   +---- N Payment Reference
   +---- N Claim/Support Case --------------------------+
                         |
                         +---- N Document Reference -> EDO/DMS
```

### Кардинальности

| Связь | Кардинальность | Правило |
|---|---:|---|
| Customer → Deal | 1:N | Каждая новая возможность отдельна |
| Customer ↔ Policy | M:N | Роль страхователя/застрахованного/выгодоприобретателя контекстна |
| Application → Quote | 1:N | Версии не перезаписываются |
| Policy → Claim | 1:N | Claim имеет собственный lifecycle |
| Policy → Renewal | 1:N over time | Каждый цикл — отдельная opportunity |
| Claim → Document | 1:N | Protected references, not public copies |

## 6. Sales Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новая заявка | Зарегистрировать спрос | Channel, product interest, owner | Routing/SLA/dedup |
| Квалификация | Проверить тип и потребность | Client type, need, next step | Task/escalation |
| Подбор продукта | Выбрать актуальную версию | Product/version ID | Catalog validation |
| Расчёт | Получить quote | Rating request/quote IDs, validity | Async status/retry control |
| Предложение | Передать условия | Quote version, amount/currency, expiry | Follow-up/version check |
| Согласование | Закрыть необходимые проверки | Decision ref, open items | Approved route |
| Договор | Подтвердить документ | Contract ID/version/status | EDO/core check |
| Полис оформлен | Получить policy fact | Policy ID/version/status/term | Reconciliation |
| Сопровождение | Передать service ownership | Policy relation, owner, open items | Welcome/service task per policy |

```text
Новая заявка -> Квалификация -> Подбор продукта -> Расчёт -> Предложение
 -> Согласование -> Договор -> Полис оформлен -> Сопровождение
```

Стадия отражает подтверждённое состояние. Повторный вход не повторяет сообщение или задачу. Причины закрытия структурированы, но не подменяют authoritative underwriting/eligibility result.

## 7. Insurance Product Architecture

| Объект | Содержание | Master/контроль |
|---|---|---|
| Продукт | Stable product ID and name | Product system |
| Категория | Governed taxonomy | Data owner |
| Версия | Effective dates/status | Immutable version ID |
| Условия | Reference to approved terms | DMS/product system |
| Тариф | Rating factors/rules | Rating engine; not CRM formula |
| Дополнительная услуга | Product relation/eligibility | Versioned catalog |

**Рекомендация архитектора:** CRM хранит минимальную проекцию для подбора и аналитики. Расчёт, применимость продукта и итоговые условия возвращает профильная система; свободная копия тарифной логики в роботах запрещена.

## 8. Policy Management

| Операция | CRM | Core insurance | Контроль |
|---|---|---|---|
| Создание | Request/status projection | Authoritative issuance | Idempotent policy ID |
| Изменение | Endorsement request/communication | Authoritative version | Effective dates/history |
| Продление | New Deal/renewal object | Renewed policy/version | Eligibility/current status |
| Завершение | Status projection | Authoritative expiry/cancellation | Source reason/date |
| Контроль сроков | Tasks/reminders | Policy term source | Timezone/freshness/suppression |

```text
Application -> Quote -> Contract -> Policy v1
                                  -> Endorsement -> Policy v2
                                  -> Expiry -> Renewal Opportunity -> New Policy
```

Полис не перезаписывается при изменении; CRM хранит version/effective dates. Нельзя считать полис действующим, оплаченным или покрывающим событие только по локальному CRM-полю.

## 9. Claims Management

| Этап | CRM/Smart Process | Claims system | Контроль |
|---|---|---|---|
| Intake | Register minimal notification | Create/accept claim | Claim external ID |
| Документы | Request/status/secure refs | Evidence repository/master | Completeness, access, version |
| Рассмотрение | Status/communication projection | Assessment/decision | No CRM decision |
| Сроки | Customer SLA tasks | Operational/regulated deadlines master | Source/calendar/escalation |
| Решение | Read-only status/reference | Authoritative decision | Source timestamp |
| Выплата/закрытие | Minimal projection | Finance/claims fact | Reconciliation |

```text
Claim Notice -> Validate Policy Reference -> Claims Intake -> Document Request
 -> Assessment in Claims System -> Decision -> Payment/Closure Projection
```

Чувствительные сведения и вложения минимизируются. Конкретные требования к составу данных, срокам и доступу определяются компанией и применимым регулированием, а не этим кейсом.

## 10. Customer Support

| Тип | Назначение | SLA/маршрут | Запрет |
|---|---|---|---|
| Обращение | Информация/изменение | Service queue | Не менять policy fact вручную |
| Консультация | Разъяснение доступной информации | Qualified specialist | Не давать неподтверждённое обещание |
| Претензия | Formal complaint lifecycle | Restricted owner/escalation | Не скрывать в комментариях |
| Claim question | Коммуникация по claim | Claims relation | Не принимать решение в CRM |
| Технический запрос | Portal/document access | Support route | Не раскрывать данные без verification |

SLA имеет календарь, timezone, pause/reopen rules, threshold, owner и evidence. Транзакционные и маркетинговые сообщения разделены.

## 11. Rights Model

| Роль | Клиенты | Полисы | Страховые случаи | Документы | Аналитика |
|---|---|---|---|---|---|
| Агент | Assigned/minimum | Assigned sale projection | No access by default | Sales docs minimum | Personal portfolio |
| Менеджер | Assigned/team | Sales/service projection | Intake status minimum | Need-to-know | Team |
| Специалист по урегулированию | Claim participants minimum | Required policy context | Assigned claims | Claim docs by role | Claims operations |
| Юрист | Approved cases | Required context | Legal/complaint scope | Authorized documents | Restricted aggregates |
| Бухгалтер | Billing identity minimum | Invoice/policy refs | Payment ref only | Financial docs | Finance |
| Руководитель | Governed portfolio | Portfolio | Team/portfolio | As role requires | Governed dashboard |
| Администратор | Technical configuration | No automatic business entitlement | No automatic claim access | No automatic content access | Config/dataset as needed |

Принципы: deny-by-default, least privilege, separation of duties, portfolio/product/department scopes, time-bound temporary access, recertification, export controls и negative tests.

## 12. Automation Architecture

| Автоматизация | Trigger | Действие | Защита |
|---|---|---|---|
| Intake SLA | First valid request | Assign/task/timer | Idempotency marker |
| Quote status | Authoritative callback/poll | Update projection/task | Version/order check |
| Renewal reminder | Eligible policy nearing expiry | Task/permitted notification | Current status, consent, suppression |
| Claim SLA | Threshold reached | Escalation | One event per threshold |
| Document request | Approved missing-item state | Client/internal task | Current completeness/version |
| Approval | Defined commercial exception | Structured route | Decision owner/audit |
| Error handling | Integration failure | Queue/alert/retry | Bounded backoff/correlation |

```text
Event -> Validate source/version/rights -> Idempotency check -> Rule
 -> Bounded action -> Record result -> Exception queue/escalation
```

Автоматизация не рассчитывает тариф, не подтверждает покрытие, не принимает claim/underwriting decision и не отмечает выплату без authoritative event.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| Core insurance | Applications, policies, versions, statuses | Official API/middleware | IDs, ordering, reconciliation, DLQ |
| Rating service | Quote request/result/version | Documented API/middleware | Input schema, validity, no local recalculation |
| Claims system | Claim ID/status/decision/payment refs | API/middleware | Restricted scope, version, audit |
| 1С | Invoice/payment projections | Supported integration/middleware | Amount/currency/control totals |
| Сайт | Enquiries/application intake | CRM form/official app/webhook/REST | Validation, consent, dedup, anti-abuse |
| Телефония | Call events/CRM linkage | Bitrix24 telephony/SIP/REST app | Rights, missed-call queue, correlation |
| BI | Governed datasets/aggregates | BI Constructor or approved ETL | Grain, masking, freshness |
| ЭДО | Contract/document ID/version/status | Official connector/API/middleware | Signature status/version reconciliation |

```text
Source -> AuthN/AuthZ -> Schema validation -> Correlation/idempotency
 -> Master ownership -> CRM write or quarantine -> Reconciliation/monitoring
```

Секреты хранятся вне Git; retry ограничен; неизвестные справочники карантинируются; двустороннее редактирование требует conflict policy. REST methods/scopes/limits проверяются по актуальной официальной документации.

## 14. Marketing Automation

| Сценарий | Условие | Действие | Контроль |
|---|---|---|---|
| Сегментация | Approved minimal attributes | Build governed audience | No sensitive inference |
| Предложение | Product eligibility from master | Create campaign/task | Consent/suppression |
| Продление | Current policy/eligibility signal | Renewal Deal/reminder | No guarantee of terms |
| Кросс-продажа | Approved relation/rule | New opportunity | Frequency cap/duplicate check |
| Service message | Active policy/case purpose | Transactional communication | Separate from marketing |

Маркетинговая автоматизация не выводит чувствительный риск-профиль и не обещает цену/покрытие. Перед отправкой повторно проверяются eligibility, consent, suppression, channel и current status.

## 15. Analytics

| Отчёт | Grain | Показатели | Контроль качества |
|---|---|---|---|
| Продажи продуктов | Authoritative issued policy | Count/premium by product/channel | Core reconciliation |
| Конверсия | One enquiry/application/deal | Stage conversion/cycle time | Funnel rules |
| Количество полисов | Policy/version status | Active/new/expired | Snapshot/effective dates |
| Пролонгации | Eligible expiring policy | Renewal rate/lapse | Eligibility denominator |
| Эффективность агентов | Agent + period + portfolio | Conversion/service discipline | Assignment changes/context |
| Прибыльность | Product/customer/period | Reconciled revenue/cost/margin | Finance/actuarial owner |

```text
CRM + Core + Claims + 1C/ERP -> Conformed IDs -> Quality/Reconciliation
 -> Governed Metric Layer -> Role-based Dashboard
```

Метрика имеет owner, formula, grain, cohort/window, timezone, currency, exclusions, source, freshness SLA и control total. CRM-данные не заменяют actuarial/financial model.

## 16. Security Architecture

| Контроль | Реализация | Проверка |
|---|---|---|
| Минимизация | Only necessary attributes/projections | Field/file inventory |
| Разделение подразделений | Role/portfolio/product scopes | Cross-unit negative tests |
| Claims access | Explicit assignment/need-to-know | Unauthorized claim test |
| Документы | Protected EDO/DMS links | Revoked/expired access test |
| Аудит | CRM history + masked integration logs | Trace by correlation ID |
| Export | Approval/minimum fields/protected target | Export permission test |
| Secrets | Environment/secret store/rotation | Secret scan/revoke drill |
| Retention | Approved retention/delete/hold policy | Periodic review |

**Cloud:** нет доступа к ядру, server PHP и DB; используются штатные права/история и внешние логи. **On-Premise:** дополнительно защищаются сервер, PHP, БД, файлы, cron, backup и журналы; ядро не изменяется.

## 17. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Заявка с сайта | Valid synthetic payload | Одна запись, owner/SLA |
| 2 | Повтор webhook | Same event ID | No duplicate |
| 3 | Невалидный контакт | Missing required value | Controlled error/quarantine |
| 4 | Подбор продукта | Current product version | Valid projection selected |
| 5 | Расчёт | Rating response/version | Quote linked once; no CRM recalculation |
| 6 | Просроченный quote | Validity expired | Offer blocked/recalculated by master |
| 7 | Policy issuance | Authoritative event | One policy projection/version |
| 8 | Endorsement | New policy version | History preserved |
| 9 | Renewal eligible | Current policy near expiry | New Deal/task once |
| 10 | Renewal ineligible | Cancelled/changed source state | No misleading message |
| 11 | Claim notice | Existing policy reference | Claim intake linked without decision |
| 12 | Claim duplicate | Same claim event ID | Duplicate absent |
| 13 | Cross-unit rights | Unauthorized user | Client/policy/claim unavailable |
| 14 | Payment update | 1С source event | Read-only projection reconciled |
| 15 | Integration timeout | Write accepted then timeout | Retry idempotent |
| 16 | Consent withdrawal | Suppression event | Marketing stops |
| 17 | Document version | New EDO version | Metadata/history updated |
| 18 | BI reconciliation | Synthetic totals | Difference within threshold/incident |
| 19 | Cloud/On-Premise variant | Capability matrix | Unsupported assumption detected |
| 20 | Rollback | Failed release step | Verified previous state restored |

Все тесты используют synthetic data; реальные личности, полисы, claims, документы и платежи запрещены. Проверяются positive, negative, boundary, concurrency, retry, rights, audit and recovery.

## 18. Deployment

| Этап | Результат | Gate |
|---|---|---|
| Discovery | AS IS, systems, tariff, volumes, data classes | Business/security approval |
| Архитектура | TO BE, master matrix, rights, ADR, NFR | Architecture review |
| Настройка | CRM/Smart Processes/automation in test scope | Configuration review |
| Миграция | Inventory/map/cleanse/dry-run/report | Counts/sample reconciliation |
| Интеграции | Contracts/secrets/retry/monitoring | Failure/replay tests |
| Обучение | Role-based synthetic exercises | UAT sign-off |
| Запуск | Window/runbook/support/rollback | Go/no-go |
| Stabilization | Incidents and baseline | Exit criteria |

```text
Inventory -> Dependency review -> Backup/export -> Test configuration
 -> Synthetic tests -> UAT -> Migration dry-run -> Go/No-Go
 -> Release -> Smoke/reconciliation -> Hypercare or Rollback
```

Рабочие поля, стадии, права, роботы и интеграции не удаляются/переименовываются до dependency analysis, backup/export, pilot and approved rollback.

## 19. Monitoring

| Сигнал | Правило | Действие | Владелец |
|---|---|---|---|
| Заявка без обработки | Intake SLA | Assign/escalate | Sales operations |
| Пропущенная пролонгация | Eligibility window/SLA | Review queue | Renewal owner |
| Claim без владельца | Unassigned/threshold | Assign/escalate | Claims operations |
| Stale policy projection | Freshness SLA | Mark stale/stop unsafe action | Core owner |
| Integration error | Error/retry/DLQ | Incident/replay after fix | Integration support |
| Payment mismatch | CRM vs 1С totals | Finance reconciliation | Finance owner |
| Rights anomaly | Unexpected access/denial | Security incident | Security owner |
| BI stale/mismatch | Freshness/control total | Data incident | BI owner |

Alert содержит system, object reference, correlation ID, time, severity, owner и runbook; без секретов и чувствительного содержания.

## 20. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Дубли клиентов | Разные каналы | Разорванная история | Stable IDs/dedup |
| 2 | Ложное объединение | Weak identifier | Чужой полис в профиле | Conservative matching/review |
| 3 | Потеря заявки | Нет ack | Упущенная продажа | Intake reconciliation |
| 4 | Дубли Deal | Retry | Искажённая воронка | Event idempotency |
| 5 | Неверный продукт | Stale catalog | Mis-sale risk | Version/effective dates |
| 6 | Тариф в CRM | Logic copied | Неверный premium | Rating engine master |
| 7 | Quote без версии | Overwrite | Неверные условия | Immutable quote version |
| 8 | Истёкший quote отправлен | No validity check | Dispute | Pre-send validation |
| 9 | Policy status manual | No core integration | False coverage impression | Read-only source status |
| 10 | Policy version overwritten | One record | Lost history | Version/effective dates |
| 11 | Participant roles mixed | Flat client link | Wrong access/communication | Contextual roles |
| 12 | Renewal promised automatically | No eligibility check | Misleading offer | Master check/human gate |
| 13 | Renewal duplicates | Multiple jobs | Spam/double work | Unique cycle key |
| 14 | Claim decision in CRM | Scope creep | Wrong decision | Claims authority |
| 15 | Payment marked manually | Convenience | Financial error | 1С/claims source |
| 16 | Claim docs broadly visible | Shared rights | Sensitive leak | Need-to-know |
| 17 | Claim in comments | No entity | No SLA/audit | Structured lifecycle |
| 18 | SLA without calendar | Simplification | False breach | Calendar/pause rules |
| 19 | Complaint lost | Mixed with support | Reputation risk | Separate type/route |
| 20 | Marketing without consent | No suppression | Compliance risk | Eligibility/consent check |
| 21 | Transactional message treated as marketing | Purpose mixed | Missed service/violation | Purpose separation |
| 22 | Sensitive segmentation | Uncontrolled inference | Discrimination/privacy risk | Approved minimal attributes |
| 23 | Cross-unit access | Common role | Data leak | Portfolio/function scopes |
| 24 | Admin as super-reader | Role confusion | Uncontrolled access | Separate privileges |
| 25 | Temporary access remains | No expiry | Residual access | Time-bound access |
| 26 | Export unrestricted | Default permission | Mass leakage | Approval/minimum fields |
| 27 | Secret in Git | Example value | Compromise | Secret store/rotation |
| 28 | Raw logs | Debug shortcut | PII leak | Masking |
| 29 | Broad token | One integration user | Large blast radius | Least privilege |
| 30 | Retry without idempotency | Timeout ambiguity | Duplicate policy/claim | Source event ID |
| 31 | Infinite retry | No DLQ | Load/storm | Bounded backoff |
| 32 | Event order ignored | Async source | Stale status wins | Version/order guard |
| 33 | Schema drift | Source update | Corrupt data | Contract validation |
| 34 | No correlation ID | Fragmented logs | Slow RCA | End-to-end trace |
| 35 | Two masters | Bidirectional edit | Ping-pong/conflict | Field ownership |
| 36 | Batch without limits | Premature optimization | Partial failure | Official limit check/report |
| 37 | Migration without dry-run | Urgency | Mass corruption | Sample/report/rollback |
| 38 | Real policies in tests | Production copy | Data breach | Synthetic data |
| 39 | No reconciliation | HTTP success trusted | Silent loss | Counts/control totals |
| 40 | Profitability from CRM only | Missing costs | False KPI | Finance/actuarial methodology |
| 41 | Wrong BI grain | Versions counted as policies | Inflated metrics | Metric contract |
| 42 | Renewal denominator wrong | Ineligible policies included | False rate | Eligibility definition |
| 43 | Agent ranking without context | Portfolio differs | Harmful decisions | Contextual governed metrics |
| 44 | Tariff not checked | Assumption | Feature unavailable | Capability assessment |
| 45 | Cloud treated as server | Expected DB/PHP | Impossible design | Cloud boundary |
| 46 | Core modified On-Premise | Quick fix | Update/security failure | Extensions only |
| 47 | Release without rights tests | Happy path only | Data exposure | Negative test suite |
| 48 | No rollback | Success assumed | Long outage | Tested rollback |
| 49 | Alert contains claim data | Full payload | Sensitive disclosure | Minimal alert |
| 50 | No support owner | Missing RACI | Long incident | Runbook/on-call |
| 51 | No baseline | AS IS unmeasured | Effect unprovable | Baseline/KPI |
| 52 | No change register | Oral changes | Poor support | Versioned register |
| 53 | Stale product projection | No freshness | Wrong recommendation | Freshness SLA |
| 54 | EDO version mismatch | Status only | Wrong contract | Document/version IDs |
| 55 | Archive confused with backup | No restore design | Failed recovery | Separate policies |

## 21. ADR

| ADR | Решение | Альтернатива | Обоснование | Последствие |
|---|---|---|---|---|
| ADR-INS-001 | Contact/Company for customer identity | Universal client Smart Process | Use native CRM semantics | Contextual participant relation needed |
| ADR-INS-002 | Policy is read-only core projection | CRM as policy master | Authority/versioning belong to core | Integration/freshness required |
| ADR-INS-003 | Claim is intake/status projection | Full adjustment in CRM | Decision/evidence belong to claims system | Restricted link and reconciliation |
| ADR-INS-004 | Documents are protected EDO/DMS refs | Authoritative files in CRM | Version/signature/retention controls | Access integration required |
| ADR-INS-005 | Middleware/contracts govern integrations | Free point-to-point webhooks | Central idempotency/monitoring | Platform owner required |
| ADR-INS-006 | Rights segmented by function/portfolio | Broad department access | Least privilege | Explicit assignment/negative tests |
| ADR-INS-007 | Renewal is a new Deal/cycle | Reopen old sale | Preserves history and conversion | Unique renewal key needed |
| ADR-INS-008 | BI uses reconciled conformed datasets | Raw CRM reports only | Core/claims/finance facts differ | Metric/DQ ownership required |

ADR получает owner, status `proposed/accepted/superseded`, date, context, alternatives, consequences, portal/version и evidence.

## 22. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Нет Discovery | Сразу настройка | Wrong scope | AS IS/stakeholders |
| 2 | Не определён Cloud/On-Premise | Template design | Unsupported solution | Capability assessment |
| 3 | Тариф не проверен | Assumption | Blocked release | Verify portal |
| 4 | CRM объявлена core insurance | Scope confusion | Wrong policy facts | Master boundary |
| 5 | Все обращения — лиды | Universal lead | Service mixed with sales | Request classification |
| 6 | Лиды отключены без ADR | Fashion | Unqualified demand mixed | Lead decision |
| 7 | Contact/Company mixed | Flat customer field | Wrong relations | Native entity model |
| 8 | Компания дублируется по продуктам | Organizational shortcut | Fragmented history | One identity/relations |
| 9 | Name is integration key | No external ID | Wrong match | Stable IDs |
| 10 | Phone is sole dedup key | Convenience | False merge | Conservative matching |
| 11 | Participant role stored on Contact | Context ignored | Wrong policy role | Contextual relation |
| 12 | One Deal for all policies | Entity saving | No lifecycle analytics | Deal per opportunity |
| 13 | Policy stored only in Deal fields | No entity/version | Lost history | Policy projection |
| 14 | Product and version mixed | No effective dates | Wrong terms | Versioned catalog |
| 15 | Rating formula copied to robot | Fast prototype | Wrong premium | Rating engine API |
| 16 | Quote overwritten | One field | Audit loss | Immutable quote versions |
| 17 | Quote validity absent | Simplification | Expired offer | Valid-until check |
| 18 | Stages describe tasks | Process mistake | Ambiguous state | State-based stages |
| 19 | Too many stages | Every action is stage | Low adoption | Tasks/checklists |
| 20 | Fields mandatory too early | Maximum control | Fake values | Stage-based requiredness |
| 21 | Loss reason in comment | No dictionary | No analytics | Governed codes |
| 22 | No next step | Stage-only discipline | Stalled Deal | Activity policy |
| 23 | Robot duplicates task | Re-entry ignored | Fake workload | Idempotency marker |
| 24 | Robot loops stages | Mutual transitions | Automation storm | Loop guard |
| 25 | Approval outside system | Chat/email | No audit trail | Structured route |
| 26 | Policy status manual | No core sync | False state | Source projection |
| 27 | Endorsement overwrites policy | Flat model | Lost version history | Version/effective dates |
| 28 | Cancellation date absent | Status only | Wrong coverage view | Source reason/date |
| 29 | Renewal reopens old Deal | KPI shortcut | Lost sales history | New renewal Deal |
| 30 | Renewal based on local date | Stale CRM | Wrong customer contact | Core eligibility event |
| 31 | Reminder after cancellation | No pre-send check | Bad experience | Current-state validation |
| 32 | Claim in Deal comments | Quick workaround | No SLA/rights | Separate intake entity |
| 33 | CRM decides claim | Overautomation | Wrong outcome | Claims system/human authority |
| 34 | Claim docs in shared folder | Convenience | Sensitive leak | Protected DMS |
| 35 | Claim and complaint mixed | One queue | Wrong SLA | Explicit classification |
| 36 | Payment manually checked | Convenience | False financial fact | 1С/claims event |
| 37 | Premium/payment mixed | Terms confusion | Bad report | Separate measures |
| 38 | Currency absent | Local assumption | Wrong totals | Required currency/date |
| 39 | Service SLA has no calendar | Simplification | False breach | Calendar/pause rules |
| 40 | Reopened case resets SLA silently | No policy | Misleading KPI | Reopen rule |
| 41 | Agent sees all clients | Broad role | Data leak | Portfolio scope |
| 42 | Claims team sees marketing data | Shared record | Excess access | Field/function minimum |
| 43 | Accountant sees claim narrative | Common rights | Excess disclosure | Finance projection only |
| 44 | Admin equals business reader | Role confusion | Uncontrolled access | Separate privileges |
| 45 | Temporary rights never expire | Manual access | Residual exposure | Expiry/recertification |
| 46 | Rights tested only positively | Happy path | Hidden access | Negative tests |
| 47 | Export unrestricted | Default setting | Mass leak | Approval/minimum columns |
| 48 | Secret in README | Real example | Compromise | Placeholder/secret store |
| 49 | Webhook committed | Convenience | Unauthorized calls | Environment/rotation |
| 50 | Token overprivileged | One credential | Large blast radius | Least privilege |
| 51 | Raw payload logged | Debug | PII exposure | Masked logs |
| 52 | Retry creates duplicates | No idempotency | Duplicate policy/claim | Source event ID |
| 53 | Infinite retry | No DLQ | Load/outage | Bounded retry |
| 54 | Out-of-order ignored | Async not designed | Stale state | Version/order guard |
| 55 | Schema not validated | Trust source | Corrupt fields | Contract validation |
| 56 | No correlation ID | Separate logs | Slow RCA | Trace ID |
| 57 | Free bidirectional edits | No master | Ping-pong | Field ownership |
| 58 | Batch used without limits | Premature optimization | Partial errors | Official checks/report |
| 59 | Migration without inventory | Unknown sources | Missing objects | Source/volume report |
| 60 | Import without dry-run | Urgency | Mass corruption | Sample/dry-run |
| 61 | Real policies in UAT | Production copy | Privacy breach | Synthetic data |
| 62 | No count/sample reconciliation | HTTP success trusted | Silent loss | Control report |
| 63 | BI counts policy versions as policies | Wrong grain | Inflated count | Metric contract |
| 64 | Renewal rate includes ineligible | Bad denominator | False KPI | Eligibility definition |
| 65 | Profitability from Deal amount | Missing finance/claims | False margin | Reconciled model |
| 66 | Agent ranked without portfolio context | Simplistic KPI | Harmful incentives | Governed metrics |
| 67 | No data-quality dashboard | Dashboard-first | False confidence | DQ controls |
| 68 | Old field deleted immediately | Cleanup | Broken dependencies | Deprecation scan |
| 69 | Release without smoke | Time pressure | Production defect | Release checklist |
| 70 | Rollback untested | Paper plan | Long outage | Restore drill |
| 71 | Monitoring only HTTP uptime | Technical focus | Requests lost | Business signals |
| 72 | Alert contains claim content | Full payload | Disclosure | Minimal alert |
| 73 | Alert has no owner | Notification only | Unresolved incident | RACI/runbook |
| 74 | Cloud assumes DB access | On-Premise habit | Impossible design | Cloud boundary |
| 75 | On-Premise core modified | Quick fix | Upgrade/security failure | Extension mechanisms |
| 76 | No change register | Oral configuration | Poor support | Versioned docs |
| 77 | Generic training only | No role scenarios | Workarounds | Role-based UAT |
| 78 | No hypercare | Immediate closure | Incident backlog | Stabilization |
| 79 | Archive treated as backup | Terms confused | Failed restore | Separate policies |
| 80 | Done means configured | No evidence | Disputed result | Tested acceptance criteria |

## 23. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Confirmed AS IS/scope | Stakeholders, systems, tariffs, volumes, data classes | Facts/open questions approved |
| 2. Архитектура | Approved TO BE | Entity model, master matrix, rights, ADR, NFR | Architecture/security review |
| 3. CRM настройка | Governed intake/sales | Fields, stages, roles, automation register | Synthetic paths pass |
| 4. Продукты | Versioned product projection | IDs, versions, mapping, rating boundary | Freshness/version tests pass |
| 5. Полисы | Policy/renewal/claim projections | Lifecycle, relations, rights, SLA | Core-to-CRM UAT passes |
| 6. Интеграции | Reliable exchange | Contracts, secrets, retry, idempotency, monitoring | Failure/replay/reconcile pass |
| 7. Тестирование | Evidence-based quality | Test plan, synthetic data, rights suite, UAT | Critical defects closed/sign-off |
| 8. Запуск | Controlled release | Cutover, backup/export, rollback, support | Smoke/reconciliation pass |
| 9. Оптимизация | Measured improvement | Baseline, DQ/incident/change backlog | KPI change verified |

### Критерии готовности

- подтверждены Cloud/On-Premise, тариф/редакция, версия, регион и права;
- утверждены master-system matrix, entity boundaries, rights and ADR;
- проверены права всех ролей позитивными и негативными тестами;
- выполнены synthetic tests, UAT, migration dry-run and reconciliation;
- секреты вынесены из Git, logs/alerts не содержат чувствительных данных;
- утверждены release, rollback, monitoring, incident, retention and support procedures;
- бизнес-владельцы приняли измеримые критерии результата.

### Официальные источники

Использованы только записи из `knowledge/SOURCES.md`; перед реализацией URL, дата обновления и применимость проверяются повторно.

- [Официальная пользовательская документация Bitrix24](https://helpdesk.bitrix24.ru/) — CRM, задачи, права, автоматизация и Cloud.
- [Официальная документация REST API](https://apidocs.bitrix24.ru/) — интеграции, авторизация и ограничения.
- [REST API reference](https://apidocs.bitrix24.ru/api-reference/) — методы, параметры и scopes.
- [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/) — официальный инструментарий.
- [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) — создание и настройка Smart Processes.
- [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) — штатная автоматизация.
- [Бизнес-процессы](https://helpdesk.bitrix24.ru/open/23951294/) — выбор типа процесса.
- [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) — официальные способы подключения.
- [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) — официальные datasets и ограничения.
- [Маркетплейс Bitrix24](https://www.bitrix24.ru/apps/) — приложения с отдельной оценкой поставщика и безопасности.
- [Документация On-Premise](https://dev.1c-bitrix.ru/docs/portal.php) — коробочная разработка без изменения ядра.

### Ограничения и открытые вопросы

1. Cloud или On-Premise, тариф/редакция, версия, регион и права.
2. Product lines, channels, lead model, portfolios and responsible owners.
3. Master клиента, продукта, quote, policy, claim, payment и документов.
4. Core/rating/claims/1С/ЭДО APIs, stable IDs, volumes, SLA and replay capabilities.
5. Допустимые CRM projections и состав чувствительных данных.
6. Consent/suppression, retention/delete/hold, audit and export policies.
7. SLA calendars, timezone, pause/reopen and escalation rules.
8. Renewal eligibility, window, ownership and duplicate-prevention rules.
9. BI grain, formulas, owners, actuarial/finance reconciliation and DQ thresholds.
10. Test portal/pipeline, release window, downtime, rollback and support RACI.
