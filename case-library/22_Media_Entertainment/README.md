# Media and Entertainment Customer Engagement CRM Architecture Case

Архитектурный кейс для синтетической media and entertainment-компании, которая продаёт рекламные размещения, производит контент, ведёт медиапроекты и мероприятия, взаимодействует с партнёрами и развивает аудиторию. Документ задаёт референсную CRM-архитектуру, а не готовую конфигурацию, медиаплан, юридическое заключение или замену CMS, DAM/MAM, ad server, ticketing, ERP и бухгалтерской системы.

> **Граница данных:** реальные рекламодатели, партнёры, сотрудники, представители аудитории, договоры, бюджеты, медиаматериалы, статистика кампаний, телефоны, email, CRM ID, webhook URL, токены и платёжные реквизиты не используются. Все примеры синтетические и обезличенные.

> **Архитектурная граница:** Bitrix24 отвечает за CRM, коммерческие возможности, проекты, задачи, согласования и клиентские коммуникации. CMS/DAM/MAM остаются master контента и медиаверсий; ad server или рекламная площадка — показов и кликов; ticketing — билетов и посещений; ERP/1С и платёжный провайдер — финансовых фактов. CRM хранит только необходимые проекции и ссылки.

Обозначения:

- **факт Bitrix24** — подтверждается официальным источником из knowledge/SOURCES.md;
- **решение архитектора** — целевая рекомендация, утверждаемая на Discovery;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа или редакции, версии, региона, приложения и прав;
- **внешний факт** — метрика или статус, подтверждаемый профильной master-системой, а не свободным полем CRM.

## 1. Business Context

### Модель медиа-бизнеса

Компания объединяет B2B-продажи рекламы и спонсорства, производство контента, мероприятия, партнёрские проекты и коммуникации с аудиторией. Один рекламодатель может иметь несколько брендов, агентств, контактов, кампаний, медиапроектов и договоров. Продажа завершается не подписанием сделки, а подтверждённым выполнением обязательств и сверкой результата.

| Поток дохода | Коммерческий объект | Исполнение | Финансовый master |
|---|---|---|---|
| Рекламные размещения | Deal + campaign/placement plan | Ad platform/CMS/project | ERP/1С |
| Спонсорство | Deal + media/event project | Project/event system | ERP/1С |
| Контентное производство | Deal + media project | Tasks/DAM/MAM | ERP/1С |
| Мероприятия | Deal/sponsorship + event | Ticketing/event operations | Ticketing/ERP |
| Подписка или доступ | Opportunity/order projection | Subscription platform | Billing/payment system |
| Лицензирование | Deal + rights/contract reference | Rights/DMS system | ERP/1С |

Цели CRM:

- единая B2B-картина рекламодателя, агентства, партнёра и контактов;
- отсутствие потерь коммерческих запросов;
- управляемая передача от продаж к production;
- контроль scope, сроков, согласований и результатов;
- повторные продажи на основании завершённых обязательств;
- прозрачные unit economics и загрузка команды;
- аудит прав, версий и интеграционных событий.

**Решение архитектора:** до настройки подтверждаются Cloud/On-Premise, тариф, роли, используемые сущности, каналы, объёмы, рекламный инвентарь, системы контента, финансовый контур, допустимый простой и тестовая среда.

## 2. AS IS Problems

| Проблема | Проявление | Причина | Целевой контроль |
|---|---|---|---|
| Нет единой базы | Агентство, бренд и рекламодатель дублируются | Нет account model | Company hierarchy and roles |
| Теряются запросы | Письмо не становится opportunity | Нет intake reconciliation | Source ID, ack, queue |
| Продажи оторваны от production | Продан неподтверждённый scope | Нет delivery gate | Handover package |
| Проекты непрозрачны | Сроки находятся в чатах | Нет owner/milestones | Project lifecycle |
| Контент копируется | Несколько финальных файлов | CRM используется как DAM | Protected master link/version |
| Результаты кампаний спорны | Разные цифры в отчётах | Нет source/grain/attribution | Metric contract |
| Повторные продажи случайны | Нет verified completion signal | Сделка закрыта слишком рано | Outcome and account plan |
| Аналитика слабая | Revenue, margin and workload не сходятся | Смешаны grains и валюты | Reconciled datasets |

AS IS включает lead sources, customer identity, product packages, rate cards, inventory booking, production workflow, content rights, approval chains, projects, events, billing, integrations, reports and exceptions.

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Lead | Неквалифицированный коммерческий запрос, если lead model утверждена | Bitrix24 |
| CRM | Contact | Представитель рекламодателя, агентства или партнёра | Bitrix24/MDM |
| CRM | Company | Рекламодатель, агентство, партнёр, подрядчик | Bitrix24/ERP/MDM |
| CRM | Deal | Отдельная коммерческая возможность | Bitrix24 |
| CRM/Smart Process | Request | Клиентский или партнёрский вопрос вне продажи | Bitrix24 |
| Smart Process | Media Project | Scope, команда, milestones и delivery | Bitrix24/PM system |
| Smart Process | Advertising Campaign | CRM-проекция кампании и результата | Ad platform |
| Smart Process | Event | Коммерческая и организационная карточка мероприятия | Event/ticketing system |
| Smart Process | Partner | Partner lifecycle и роли | Bitrix24/ERP |
| Smart Process | Content Item | Метаданные, права и ссылка на asset | CMS/DAM/MAM |
| Smart Process | Contract | Согласование и ссылка на authoritative version | DMS/ЭДО |

~~~text
Website / Calls / Partners -> Lead or Request -> Contact + Company -> Deal
                                                        |
                                                        v
                              Contract -> Media Project -> Campaign / Event
                                             |              |
                                             v              v
                                       Tasks / Team     Placements / Results
                                             |              |
                                  Content refs <- CMS/DAM   Ad platform metrics
                                             \              /
                                              BI reconciliation
~~~

### Правила целостности

1. Deal описывает коммерческую opportunity, Media Project — исполнение, Campaign — рекламный flight, Placement — единицу размещения.
2. Company-роли advertiser, agency, brand owner, partner and contractor не создают автоматических дублей компании.
3. Content Item хранит метаданные и защищённую ссылку; медиабинарники и версии остаются в CMS/DAM/MAM.
4. Campaign result поступает из рекламной площадки или согласованного аналитического источника.
5. Revenue, payment and cost не редактируются свободно в CRM, если master — ERP/1С/payment system.
6. Integration projection содержит source system, external ID, source timestamp, sync status and correlation ID.
7. Повторное событие не создаёт повторную кампанию, задачу, договор или уведомление.

**Cloud:** нет доступа к ядру, серверному PHP и базе данных; сложная интеграция, обработка медиа и расширенные логи располагаются во внешнем контуре. **On-Premise:** дополнительно проектируются server, PHP, DB, files, cron, backup, updates, load and security; ядро не изменяется.

## 4. Customer Journey

~~~text
Контакт -> Запрос -> Переговоры -> Предложение -> Договор
 -> Проект -> Результат -> Повторное сотрудничество

Exception paths:
No-fit / no inventory / rejected concept / rights issue / cancellation
 -> structured outcome -> retained evidence -> account follow-up rule
~~~

| Этап | Цель | Обязательный результат | Gate |
|---|---|---|---|
| Контакт | Идентифицировать account and role | Contact linked to Company | Dedup and lawful contact basis |
| Запрос | Зафиксировать need, dates and channel | Qualified request | Intake complete |
| Переговоры | Согласовать scope and constraints | Next step and stakeholders | Owner/date |
| Предложение | Представить версию решения | Offer version and validity | Inventory/production check |
| Договор | Зафиксировать approved commitments | Contract reference/status | Authorized approval |
| Проект | Исполнить scope | Milestones, team, deliverables | Handover accepted |
| Результат | Подтвердить delivery and metrics | Acceptance/result version | Source reconciliation |
| Повторное сотрудничество | Создать новую opportunity | Account plan/eligible signal | No duplicate/consent check |

Journey не заканчивается закрытием Deal: delivery, acceptance, campaign metrics, financial fact and lessons learned имеют собственные источники и сроки.

## 5. Media Data Model

| Сущность | Основные данные | Связи | Master |
|---|---|---|---|
| Клиент | Segment, industry, account owner, project history | Contacts, brands, deals, projects | CRM/MDM |
| Контакт | Role, department, influence, communication preference | Companies, deals, approvals | CRM/MDM |
| Проект | Format, scope, dates, budget projection, team, status | Deal, campaign, content, tasks | Bitrix24/PM |
| Кампания | Channels, flight dates, KPI target, result refs | Project, placements, advertiser | Ad platform |
| Размещение | Channel, format, planned slot, status | Campaign, content asset | Ad server/CMS |
| Контент | Type, version, rights status, master link | Project, campaign, approvals | CMS/DAM/MAM |
| Мероприятие | Format, dates, venue projection, capacity | Sponsors, partners, tasks | Event/ticketing |
| Договор | Contract ID, version, status, dates | Company, deal, project | DMS/ЭДО |

~~~text
Advertiser Company 1 ---- N Brand
        |                     |
        +---- N Deal ---- 1..N Media Project
                               |       \
                               |        +---- N Campaign ---- N Placement
                               |                         \
                               +---- N Content Item ------> Asset in DAM/MAM
                               |
                               +---- 0..N Event ---- N Partner Company
~~~

### Идентичность и кардинальность

- advertiser, agency and brand фиксируются раздельными ролями;
- one Deal may initiate several delivery projects only by approved decomposition;
- one campaign has one agreed reporting grain and may contain many placements;
- one content asset can participate in several campaigns through explicit versioned relations;
- project budget, booked revenue, invoiced amount, paid amount and production cost are distinct facts;
- historical company/contact relations are not silently overwritten.

## 6. Sales Pipeline

| Стадия | Цель | Обязательные данные | Действия | Автоматизация |
|---|---|---|---|---|
| Новый запрос | Зарегистрировать opportunity | Source, account, need, owner | Dedup and acknowledge | Task/SLA |
| Квалификация | Проверить fit, budget range, timing | Stakeholders, format, next step | Qualify/disqualify | Reminder/escalation |
| Подготовка концепции | Сформировать solution outline | Brief, channel, constraints | Involve producer/editor | Task set |
| Коммерческое предложение | Зафиксировать offer version | Scope, price, validity, assumptions | Send/record feedback | Follow-up |
| Согласование | Согласовать scope, price and rights | Decision makers, changes, approvals | Version control | Approval route |
| Договор | Подтвердить commitments | DMS ref, version, dates | Handover preparation | Completeness gate |
| Производство | Передать исполнение | Project ID, team, milestones | Delivery control | Project monitoring |
| Завершение | Подтвердить acceptance and outcome | Result, finance refs, lessons | Close/account plan | Repeat-sale eligibility |

~~~text
Новый запрос -> Квалификация -> Концепция -> КП -> Согласование -> Договор
 -> Производство -> Завершение

Closed-lost -> Structured reason -> Account follow-up rule
Change request -> Impact assessment -> New version / new Deal / Project change
~~~

Производство в sales pipeline является коммерческой контрольной стадией, но детальный delivery lifecycle живёт в Media Project. Стадии не заменяют задачи и checklist.

## 7. Advertising Sales Architecture

| Объект | Назначение | Обязательные атрибуты | Контроль |
|---|---|---|---|
| Рекламодатель | Account receiving campaign value | Stable ID, segment, owner | Dedup/rights |
| Агентство | Intermediary and role | Relationship, validity, contacts | No advertiser substitution |
| Пакет | Commercial offer configuration | Version, contents, validity | Rate card source |
| Кампания | Agreed objective and flight | Dates, KPI target, channels | Source and attribution |
| Размещение | Unit of planned delivery | Inventory ID, format, slot | Availability/source status |
| Договор | Commitments and version | DMS ID/status/effective dates | Authoritative approval |
| Результат | Delivery and performance facts | Grain, source, timestamp | Reconciliation |

~~~text
Advertiser / Agency -> Opportunity -> Package / Offer Version -> Contract
 -> Campaign -> Placement Plan -> Delivery Source -> Result -> Acceptance
~~~

Bitrix24 не объявляется ad server или inventory reservation engine. Availability, delivered impressions, clicks, views and other platform metrics подтверждаются соответствующим источником. Любая attribution model версионируется.

## 8. Content Production Management

| Этап | Результат | Ответственный | Gate |
|---|---|---|---|
| Brief | Approved input and constraints | Sales/producer | Completeness |
| Concept | Versioned creative concept | Producer/editor | Customer/internal approval |
| Planning | Team, milestones, resources | Project manager | Capacity and dependencies |
| Production | Working assets and tasks | Creative team | Review checkpoints |
| Editorial/legal review | Approved version and rights status | Editor/authorized reviewer | No publication before approval |
| Publication/delivery | Authoritative asset/version published | CMS/DAM owner | Source confirmation |
| Acceptance | Client/internal result accepted | Project owner | Evidence and exceptions |

~~~text
Brief -> Concept vN -> Plan -> Production -> Review -> Approved Asset
                                              |              |
                                              +-> Rework     +-> CMS/DAM publish
~~~

Каждая задача имеет owner, due date, input, expected deliverable and acceptance criteria. Content Item не содержит неуправляемые копии master asset. Авторские и лицензионные права — отдельный обязательный gate, определяемый юристом и DMS.

## 9. Event Management

| Область | CRM/Project projection | Master and control |
|---|---|---|
| Мероприятие | Type, sponsor, dates, owner, status | Event platform |
| Участники | Минимальная разрешённая регистрационная проекция | Ticketing/registration |
| Партнёры | Company roles, commitments, contacts | CRM/DMS |
| Задачи | Production, promotion, logistics, follow-up | Bitrix24/PM |
| Бюджет | Approved summary and finance refs | ERP/1С |
| Билеты/доступ | Status/count projection | Ticketing |
| Результат | Attendance, sponsor delivery, feedback | Source systems |

~~~text
Event Opportunity -> Event Card -> Plan / Partners / Tasks
                         |              |
                         v              v
                    Ticketing       Sponsor Deliverables
                         \              /
                          Result / Finance Reconciliation
~~~

Capacity, ticket validity, check-in and payment facts не редактируются вручную в CRM при наличии профильной master-системы.

## 10. Partner Management

| Partner type | Lifecycle | Key controls |
|---|---|---|
| Content partner | Prospect, approved, active, review | Rights, scope, content versions |
| Distribution partner | Agreement, channels, reporting | Territory/channel and metrics |
| Production contractor | Qualification, assignment, acceptance | Capacity, access, deliverables |
| Venue/technical partner | Booking, readiness, delivery | Dates, resources, fallback |
| Sponsorship partner | Opportunity, contract, activation, result | Commitments and evidence |

Партнёр хранится как Company с ролями и связанными проектами; отдельный Smart Process применяется, если нужен самостоятельный lifecycle, qualification or periodic review. Коммуникации и договоры не переносятся в одну многострочную заметку.

## 11. Rights Model

| Роль | Клиенты | Проекты | Контент | Документы | Аналитика |
|---|---|---|---|---|---|
| Менеджер продаж | Own/team accounts | Commercial summary | Approved previews | Sales refs | Own/team sales |
| Продюсер | Need-to-know | Assigned projects | Production assets by role | Project refs | Delivery/load |
| Редактор | Minimum client context | Assigned | Review/edit by scope | Rights/approval refs | Editorial quality |
| Маркетолог | Approved segments | Campaign scope | Approved assets | Campaign refs | Channel/campaign |
| Менеджер проекта | Project accounts | Full assigned delivery | Project assets | Project contract refs | Plan/fact |
| Директор | Governed portfolio | Portfolio | Approved overview | Approved scope | Executive |
| Администратор | Нет automatic business entitlement | Configuration | Нет automatic asset access | Нет automatic content | Config/datasets as needed |

Принципы: deny-by-default, least privilege, team/project scopes, segregation of sales/editorial/finance duties, named accounts, temporary access expiry, recertification, download/export controls and negative tests.

Права в CMS/DAM, event platform and DMS проектируются отдельно. Доступ к CRM-карточке не должен автоматически выдавать доступ к master content or document.

## 12. Automation Architecture

| Автоматизация | Trigger | Действие | Защита |
|---|---|---|---|
| Intake SLA | Новый валидный request | Owner/task/ack | Source key/idempotency |
| Concept task set | Qualified opportunity | Tasks for producer/editor | One set/version |
| Offer follow-up | Current offer nearing expiry | Manager task | Version/state check |
| Handover | Contract and readiness gates passed | Create/link project and task package | Unique Deal/project key |
| Milestone control | Due threshold | Reminder/escalation | One event/threshold |
| Approval | Specific content/offer version ready | Start route | Version lock/authority |
| Repeat sale | Accepted result and eligibility | Account task/new opportunity | Consent/suppression/no duplicate |
| Integration failure | Error/timeout | Queue/alert/bounded retry | Correlation/DLQ |

~~~text
Event -> Validate source / version / rights -> Idempotency -> Business rule
 -> Bounded action -> Audit result -> Exception queue / escalation
~~~

Робот не подтверждает публикацию, права на контент, delivered advertising metrics, подписание договора, оплату или acceptance без authoritative/human gate. Для массовых операций обязательны dry-run, test sample, batch report and rollback.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| Сайт | Leads, requests, event registrations by scope | CRM form/approved app/webhook/REST | Validation, consent, dedup |
| CRM рекламной площадки | Accounts, opportunities, placement refs where available | Documented API/middleware | Mapping, scopes, reconciliation |
| Web/ad analytics | Campaign IDs and governed metrics | API/ETL/middleware | Grain, attribution, freshness |
| Телефония | Call events and CRM relation | Bitrix24 telephony/SIP/REST app | Rights, unmatched calls, correlation |
| Платежи | Payment/refund status projection | Provider API/event | Authenticity, idempotency, amount reconcile |
| BI/DWH | Governed datasets and aggregates | BI Constructor or approved ETL | Access, grain, DQ, freshness |
| CMS/DAM/MAM | Content IDs, versions, rights/status refs | API/middleware/approved connector | Version, access, checksum/status |
| ERP/1С | Contracts, invoices, payments, costs | Supported integration/middleware | Stable IDs, currency, totals |

~~~text
Source -> AuthN/AuthZ -> Minimize -> Validate schema/version
 -> Map master IDs -> Idempotency -> Bitrix24 projection
 -> Result log -> Bounded retry / quarantine -> Reconciliation
~~~

Наличие готового коннектора к конкретной ad platform, CMS, DAM, payment or ticketing system не предполагается. REST methods, scopes, limits and applications проверяются по официальной документации и на целевом портале. Секреты хранятся вне Git и payload logs.

## 14. Marketing Automation

| Сценарий | Аудитория | Условие допуска | Контроль |
|---|---|---|---|
| B2B nurture | Qualified advertiser/partner contacts | Approved business rule and channel | Frequency/suppression |
| Repeat offer | Completed project cohort | Verified completion and relevance | No active duplicate |
| Event invitation | Approved segment | Communication basis and capacity | Opt-out/capacity check |
| Audience campaign | Governed audience segment | Consent/legal basis and approved purpose | No sensitive inference |
| Service message | Active project/event participant | Transactional purpose | Separate from marketing |
| Loyalty | Verified engagement history | Defined program eligibility | Benefit/source reconciliation |

~~~text
Approved Data -> Segment Definition -> Eligibility / Consent / Suppression
 -> Campaign Version -> Send -> Delivery/Response -> Measurement -> Retention
~~~

Анонимная аудитория, зарегистрированный участник, B2B contact and advertising account — разные identities. Нельзя создавать Contact на каждого anonymous visitor. Транзакционные и маркетинговые коммуникации разделяются.

## 15. Analytics

| Отчёт | Grain | Показатели | Контроль качества |
|---|---|---|---|
| Продажи рекламы | One authoritative Deal/order | Revenue projection, conversion, cycle | ERP/CRM reconcile |
| Эффективность проектов | Project/milestone | On-time, acceptance, rework | Scope/version context |
| Прибыльность | Project/campaign/period | Revenue, direct cost, margin | Finance-owned formula |
| Загрузка команды | Assignment/person/period | Planned, active, overdue | Capacity/calendar |
| Эффективность каналов | Campaign/channel/period | Governed reach, response, conversion | Source/attribution |
| Повторное сотрудничество | Account + accepted projects | Repeat rate, interval, cohort | Stable identity |
| Контент | Asset/version/channel | Production time, reuse, outcome | DAM/CMS version |
| Мероприятия | Event/session/registration | Registrations, attendance, sponsor delivery | Ticketing reconciliation |

~~~text
CRM + ERP/1C + CMS/DAM + Ad Platforms + Ticketing
 -> Conformed Account / Project / Campaign / Asset IDs
 -> DQ and Reconciliation -> Metric Layer -> Role-based Dashboards
~~~

Для метрики фиксируются owner, formula, grain, period, timezone, currency, attribution window/model, exclusions, source, freshness, access and control total. Impressions, people, leads, deals, projects and payments нельзя суммировать как единый grain.

## 16. Security Architecture

| Контроль | Реализация | Проверка |
|---|---|---|
| Commercial confidentiality | Team/project scopes | Cross-team negative test |
| Content assets | Protected master links and role access | Unauthorized download |
| Rights metadata | Versioned restrictions and expiry | Expired-use test |
| Audience data | Minimization/pseudonymization | Field and dataset inventory |
| Documents | DMS links, version and approval | Revoked access |
| Integrations | Least-privilege credentials and rotation | Scope/revoke drill |
| Logs | Masked payload and correlation ID | Secret/PII scan |
| Export | Approval/minimum/protected destination | Export test |
| Audit | CRM history plus integration/security logs | End-to-end trace |

**Cloud:** сложные integration and media-processing components remain external; нет доступа к platform DB/core. **On-Premise:** дополнительно защищаются OS, web server, PHP, DB, modules, files, cron, backup and logs; core is not modified.

Авторские, лицензионные, рекламные и персональные ограничения определяются уполномоченными владельцами вне CRM. CRM хранит утверждённый статус и ссылку, но не формирует юридическое заключение.

## 17. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Новый запрос с сайта | Валидный synthetic payload | One Lead/Deal or Request, owner and ack |
| 2 | Duplicate source event | Same external ID | No duplicate |
| 3 | Advertiser through agency | Two company roles | Correct relations, no false merge |
| 4 | Lead conversion path | Qualified synthetic request | Contact/Company/Deal linked |
| 5 | Concept preparation | Producer/editor assigned | One task set with deadlines |
| 6 | Offer version change | v2 approved after v1 | Only current version used |
| 7 | Inventory unavailable | External rejection | Controlled exception/no false promise |
| 8 | Contract handover | Readiness gates complete | One Media Project linked |
| 9 | Content approval | Rejected asset version | Publication blocked/rework preserved |
| 10 | Campaign result | Authoritative metrics arrive | Correct campaign/grain/source |
| 11 | Payment mismatch | Provider vs ERP totals | Reconciliation incident |
| 12 | Event registration | Synthetic participant | Correct event relation/minimum data |
| 13 | Cross-project rights | Unassigned editor | Access denied |
| 14 | Expired partner access | Temporary role expired | Access revoked |
| 15 | Integration timeout | Accepted then timeout | Idempotent bounded retry |
| 16 | Out-of-order metric event | Older source timestamp | New state not overwritten |
| 17 | Repeat sale | Accepted project signal | New opportunity/task, old Deal intact |
| 18 | BI grain | Deals/projects/campaigns/assets | Separate correct metrics |
| 19 | Migration dry-run | Synthetic sample | Report without production write |
| 20 | Rollback | Failed release | Previous state restored |

Тесты используют synthetic advertisers, contacts, campaigns, assets, events and payments. Обязательны positive, negative, repeated, rights, versioning, failure, recovery and reconciliation paths.

## 18. Deployment

| Этап | Результат | Gate |
|---|---|---|
| Discovery | AS IS, systems, products, rights, volumes | Business/data/security approval |
| Архитектура | TO BE, master matrix, grains, ADR, NFR | Architecture review |
| Настройка | Isolated CRM/Smart Processes/automation | Configuration review |
| Миграция | Inventory, mapping, clean, dry-run | Reconciliation/sign-off |
| Интеграции | Contracts, secrets, retry, monitoring | Failure/security/replay tests |
| Обучение | Role-based synthetic scenarios | UAT |
| Запуск | Cutover, communication, fallback, rollback | Go/no-go |
| Stabilization | Hypercare, incidents and baseline | Exit criteria |

~~~text
Inventory -> Dependency / Rights Review -> Test Configuration
 -> Synthetic UAT -> Migration Dry-run -> Go/No-Go
 -> Release -> Smoke / Reconciliation -> Hypercare or Rollback
~~~

Поля, стадии, роли, роботы и интеграции действующего портала не удаляются и не переименовываются до dependency analysis, backup/export, pilot and approved rollback.

## 19. Monitoring

| Сигнал | Правило | Действие | Владелец |
|---|---|---|---|
| Потерянная заявка | Source count differs/no ack | Incident/replay | Sales operations |
| Сделка без следующего шага | Threshold by stage | Task/escalate | Sales manager |
| Просроченный milestone | Plan date breached | Escalate/replan | Project owner |
| Задача без владельца | Empty/inactive assignee | Reassign/escalate | Process owner |
| Integration error | Error/retry/DLQ | Investigate/replay | Integration support |
| Stale campaign metrics | Freshness exceeded | Mark stale/block report | Analytics owner |
| Content rights expiry | Approved threshold | Stop-use review/task | Rights owner |
| Payment mismatch | Control totals differ | Finance incident | Finance owner |
| Access anomaly | Unexpected view/export | Security incident | Security owner |

Alert содержит system, object reference, correlation ID, time, severity, owner and runbook; без asset binary, полного договора, секрета, платёжных реквизитов и лишних персональных данных.

## 20. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Потеря коммерческого запроса | Нет ack | Упущенная продажа | Intake reconciliation |
| 2 | Duplicate Lead/Deal | Retry/multiple channels | Double work | Source key/dedup |
| 3 | False company merge | Weak name match | Wrong disclosure | Stable ID/manual review |
| 4 | Agency equals advertiser | Flat account model | Wrong ownership | Explicit roles |
| 5 | Brand stored as duplicate company | No hierarchy | Fragmented history | Approved brand model |
| 6 | Lead used for every request | Universal pattern | Service mixed with sales | Classification |
| 7 | One Deal for recurring business | Convenience | No cycle analytics | Deal per opportunity |
| 8 | Deal and project merged | One object easier | Sales/delivery conflict | Separate lifecycles |
| 9 | Campaign and project merged | Grain ignored | Wrong reporting | Explicit relation |
| 10 | Placement omitted | Aggregate-only model | Delivery dispute | Placement grain |
| 11 | Inventory promised from CRM | No source check | Missed commitment | Inventory gate |
| 12 | Offer version uncontrolled | Files in chat | Wrong scope/price | Versioned offer |
| 13 | Handover incomplete | Sales closes early | Production rework | Acceptance package |
| 14 | Change request overwrites scope | No baseline | Margin/schedule loss | Change control |
| 15 | Content copied into CRM | CRM as DAM | Version/storage risk | Master link |
| 16 | Wrong asset version published | No version gate | Brand/reputation damage | Approval lock |
| 17 | Rights status manual/stale | No legal source | Unauthorized use | Rights master/expiry |
| 18 | Contract status guessed | No DMS sync | Invalid delivery | DMS authority |
| 19 | Metrics entered manually | Reporting pressure | False result | Platform source |
| 20 | Attribution model undefined | Dashboard-first | Disputed ROI | Versioned contract |
| 21 | Payment equals Deal close | Sales semantics | Financial error | ERP/provider authority |
| 22 | Cost absent | Revenue-only CRM | False margin | Finance integration |
| 23 | Event capacity stale | Ticketing disconnected | Oversell | Source freshness |
| 24 | Anonymous visitor becomes Contact | Tracking overreach | Privacy/data bloat | Aggregate audience |
| 25 | Consent inferred | No approved basis | Unauthorized communication | Eligibility rule |
| 26 | Service and marketing mixed | One automation | Wrong messages | Separate purposes |
| 27 | Broad content access | Shared project | Asset leak | Role-based links |
| 28 | Admin as business super-reader | Role confusion | Uncontrolled access | Separate privileges |
| 29 | Export unrestricted | Defaults | Commercial data leak | Approval/minimum |
| 30 | Temporary partner access persists | Manual grant | Residual access | Expiry/recertification |
| 31 | Secret in Git | Real example | Compromise | Secret store/rotation |
| 32 | Broad integration token | One account | Large blast radius | Least privilege |
| 33 | Raw payload logged | Debugging | PII/commercial leak | Masking |
| 34 | Infinite retry | No DLQ | Load/outage | Bounded backoff |
| 35 | Retry duplicates project | No idempotency | Double production | Business key |
| 36 | Out-of-order status | Async ignored | Old version wins | Version/order guard |
| 37 | Schema drift | Source change | Corrupt mapping | Contract validation |
| 38 | No correlation ID | Fragmented logs | Slow RCA | End-to-end trace |
| 39 | Free bidirectional edit | No master | Ping-pong | Field ownership |
| 40 | Connector assumed | Unverified claim | Blocked release | Official verification |
| 41 | Migration without inventory | Unknown sources | Missing relations | Source report |
| 42 | Import without dry-run | Time pressure | Mass corruption | Sample/report/rollback |
| 43 | Real media client data in UAT | Prod copy | Confidentiality breach | Synthetic data |
| 44 | No reconciliation | HTTP success trusted | Silent loss | Counts/control totals |
| 45 | BI mixes grains | Simplistic model | False KPI | Metric contract |
| 46 | Team ranking ignores project complexity | Context omitted | Harmful decisions | Capacity/case mix |
| 47 | Peak campaign load untested | Average design | Queue collapse | Load/degraded mode |
| 48 | Tariff not checked | Assumption | Feature unavailable | Portal assessment |
| 49 | Cloud assumes DB/media processing | On-Premise habit | Impossible design | External services |
| 50 | On-Premise core modified | Quick fix | Upgrade/security failure | Extensions only |
| 51 | Release lacks rights tests | Happy path | Hidden exposure | Negative suite |
| 52 | Rollback untested | Paper plan | Long outage | Restore drill |
| 53 | Alert has no owner | Notification only | Unresolved incident | RACI/runbook |
| 54 | No change register | Oral changes | Configuration drift | Versioned register |
| 55 | DR ignores CMS/DAM/ad platforms | CRM-only recovery | Broken service | End-to-end DR |

## 21. ADR

| ADR | Решение | Альтернатива | Обоснование | Последствие |
|---|---|---|---|---|
| ADR-MED-001 | Contact/Company plus explicit advertiser, agency and brand roles | One flat customer record | Supports B2B relationship history | Role governance required |
| ADR-MED-002 | Deal separated from Media Project | One pipeline for sales and delivery | Different owners, states and KPI | Handover relation needed |
| ADR-MED-003 | Campaign and Placement are separate grains | Campaign-only aggregate | Enables delivery reconciliation | More external IDs |
| ADR-MED-004 | Content Item is metadata/link, CMS/DAM is master | Store media files in CRM | Preserves versions, rights and scale | Master availability needed |
| ADR-MED-005 | Contract/rights statuses come from DMS/authorized source | CRM checkbox as legal fact | Prevents false approval | Integration/manual verified gate |
| ADR-MED-006 | Integrations use governed middleware/contracts | Point-to-point webhooks | Idempotency, monitoring and security | Platform owner required |
| ADR-MED-007 | Rights limited by team/project/purpose | Company-wide read | Protects commercial/content data | Negative tests required |
| ADR-MED-008 | BI uses reconciled conformed grains | Raw CRM counters | Reliable revenue, delivery and attribution | Metric/DQ ownership |

Каждый ADR получает owner, status proposed/accepted/superseded, date, context, alternatives, consequences, portal/version and evidence.

## 22. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Нет Discovery | Настройка сразу | Wrong scope | AS IS/stakeholders |
| 2 | Cloud/On-Premise не определён | Template design | Unsupported solution | Capability matrix |
| 3 | Тариф не проверен | Assumption | Missing function | Verify target portal |
| 4 | Нет владельцев процессов | IT-only project | No decisions | RACI |
| 5 | CRM объявлена CMS/DAM/ad server | Scope confusion | Bad architecture | System boundaries |
| 6 | Все входы — Leads | Universal lead | Service/partners mixed | Intake classification |
| 7 | Лиды отключены без ADR | Fashion | Unqualified demand mixed | Lead decision |
| 8 | Contact and Company mixed | Flat customer | Broken B2B history | Native entities |
| 9 | Agency stored as advertiser | No role model | Wrong contract/analytics | Explicit roles |
| 10 | Brand is duplicate Company | Weak hierarchy | Fragmented account | Approved brand entity |
| 11 | Name is integration key | No external ID | Wrong match | Stable IDs |
| 12 | One Deal for all annual work | Entity saving | No opportunity history | Deal per sale |
| 13 | Deal equals Media Project | One pipeline | Sales/delivery conflict | Separate objects |
| 14 | Campaign equals Project | Grain ignored | False result | Relation and grains |
| 15 | Placement omitted | Aggregate model | Cannot reconcile delivery | Placement entity/ref |
| 16 | Event stored as Deal only | Sales-first model | Operations invisible | Event lifecycle |
| 17 | Partner copied for each role | No role attributes | Duplicates | Company role model |
| 18 | Stages describe tasks | Every action as stage | Ambiguity | State-based stages |
| 19 | Too many stages | Process micromanagement | Low adoption | Tasks/checklists |
| 20 | Mandatory fields too early | Maximum control | Fake values | Stage requiredness |
| 21 | Brief lives in comments | No structure/version | Production rework | Versioned brief |
| 22 | No next step | Stage-only control | Stalled Deal | Activity/task policy |
| 23 | Concept has no version | Chat approvals | Wrong delivery | Version ID |
| 24 | Offer overwritten | One file | Scope dispute | Immutable versions |
| 25 | Inventory not checked | Sales optimism | Impossible commitment | Source gate |
| 26 | Handover has no acceptance | Sales closes early | Lost requirements | Handover checklist |
| 27 | Project created twice | Retry/re-entry | Double work | Unique key |
| 28 | Change request edits baseline | No change control | Margin loss | Impact/version approval |
| 29 | Robot duplicates tasks | Re-entry ignored | Fake workload | Idempotency marker |
| 30 | Robots loop stages | Mutual rules | Automation storm | Loop guard |
| 31 | Approval starts without version | Generic workflow | Wrong asset approved | Version lock |
| 32 | Robot approves rights | Overautomation | Unauthorized use | Human/source gate |
| 33 | Content files stored in CRM copies | Convenience | Version/storage leak | DAM/MAM link |
| 34 | Asset filename is identity | Weak key | Wrong version | Stable asset ID |
| 35 | Rights expiry not tracked | No owner | Unauthorized reuse | Effective dates/task |
| 36 | Publication marked manually | No CMS event | False status | Source confirmation |
| 37 | Metrics pasted into comments | No data contract | No audit/BI | Structured source facts |
| 38 | Impressions treated as people | Grain confusion | Inflated reach | Metric dictionary |
| 39 | Attribution window absent | Dashboard default | Disputed ROI | Versioned attribution |
| 40 | Revenue equals Deal amount | CRM-only view | Finance mismatch | ERP reconciliation |
| 41 | Payment status manual | Convenience | Financial error | Provider/ERP master |
| 42 | Cost not integrated | Revenue focus | False margin | Finance-owned costs |
| 43 | Event registrations stored as free text | Quick setup | No capacity/identity | Ticketing relation |
| 44 | Check-in edited in CRM | No source sync | False attendance | Ticketing master |
| 45 | Anonymous audience creates Contacts | Tracking habit | Data bloat/privacy | Aggregate audience |
| 46 | Marketing consent assumed | No governance | Unauthorized campaign | Eligibility evidence |
| 47 | Transactional and marketing mixed | One template | Wrong communication | Separate purposes |
| 48 | No frequency cap | Campaign pressure | Audience fatigue | Suppression policy |
| 49 | Operator sees all accounts | Broad role | Commercial leak | Team scope |
| 50 | Editor sees finance | Shared record | Excess access | Functional minimum |
| 51 | Marketer sees raw contracts | Common rights | Excess disclosure | Campaign projection |
| 52 | Admin equals business reader | Role confusion | Uncontrolled access | Separate privileges |
| 53 | Temporary partner access never expires | Manual grant | Residual risk | Expiry/recertification |
| 54 | Rights only positive-tested | Happy path | Hidden cross-access | Negative matrix |
| 55 | Export unrestricted | Defaults | Mass leak | Approval/minimum |
| 56 | Token in README | Real example | Compromise | Placeholder/secret store |
| 57 | Webhook committed | Quick integration | Unauthorized calls | Environment/rotation |
| 58 | One broad token | Simplicity | Large blast radius | Purpose scopes |
| 59 | Raw payload logged | Debugging | PII/commercial leak | Masking |
| 60 | Infinite retry | No DLQ | Load/outage | Bounded retry |
| 61 | Retry duplicates campaign/project | No idempotency | Double execution | Source event key |
| 62 | Out-of-order ignored | Async not designed | Old status wins | Version/order guard |
| 63 | Schema not validated | Trust source | Corrupt data | Contract validation |
| 64 | No correlation ID | Fragmented logs | Slow RCA | Trace ID |
| 65 | Free bidirectional edits | No master | Ping-pong | Field ownership |
| 66 | Connector invented | Guessing | Blocked delivery | Official verification |
| 67 | REST method invented | No reference check | Broken integration | API docs |
| 68 | Batch ignores limits | Premature optimization | Partial failure | Official check/report |
| 69 | Migration has no inventory | Unknown sources | Missing objects | Source/volume report |
| 70 | Import without dry-run | Time pressure | Mass corruption | Sample/dry-run |
| 71 | Real client/content data in UAT | Prod copy | Confidentiality breach | Synthetic data |
| 72 | No relation reconciliation | HTTP success trusted | Silent orphan records | Counts/sample report |
| 73 | BI mixes Deal/Project/Campaign/Placement | Wrong grain | False KPI | Metric contract |
| 74 | Average hides delays | Simplistic dashboard | Long tail invisible | Percentiles/cohorts |
| 75 | Team ranking ignores complexity | KPI pressure | Harmful management | Context/capacity |
| 76 | Peak launch not load-tested | Average design | Queue collapse | Load/degraded-mode test |
| 77 | Old fields deleted immediately | Cleanup | Broken dependencies | Deprecation scan |
| 78 | Release without smoke/rollback | Deadline pressure | Long incident | Checklist/drill |
| 79 | Cloud assumes server/media access | On-Premise habit | Impossible design | External boundary |
| 80 | On-Premise core modified | Quick fix | Upgrade/security failure | Extensions only |

## 23. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Подтверждённый AS IS/scope | Stakeholders, products, channels, systems, tariff, volumes | Facts/open questions approved |
| 2. Архитектура | Утверждённый TO BE | Entity/grain model, master matrix, rights, ADR, NFR | Architecture/security review |
| 3. CRM настройка | Governed account and intake model | Fields, stages, roles, automation register | Synthetic paths pass |
| 4. Продажи | Controlled opportunity lifecycle | Brief, offer, contract and handover gates | Sales UAT passes |
| 5. Проекты | Delivery, content and event coordination | Project/campaign/content relations, tasks, approvals | End-to-end UAT passes |
| 6. Интеграции | Reliable secure exchange | Contracts, secrets, retry, idempotency, monitoring | Failure/replay/reconcile pass |
| 7. Тестирование | Evidence-based quality | Test plan, synthetic data, rights/load/rollback tests | Critical defects closed/sign-off |
| 8. Запуск | Controlled release | Cutover, backup/export, communication, fallback, rollback | Smoke/reconciliation pass |
| 9. Оптимизация | Measured improvement | Baseline, DQ, incidents and change backlog | KPI change verified |

### Критерии готовности

- подтверждены Cloud/On-Premise, тариф/редакция, версия, регион, права и тестовый контур;
- утверждены advertiser/agency/brand model, Lead decision, entity grains, masters, rights and ADR;
- CMS/DAM/MAM, ad platforms, ticketing, DMS, ERP/1С and payment sources не подменяются CRM-полями;
- выполнены synthetic UAT, negative rights, version, failure, load, migration dry-run and reconciliation tests;
- секреты вынесены из Git; logs/alerts не содержат assets, contracts, payment data or unnecessary PII;
- утверждены cutover, rollback, DR, degraded mode, monitoring and support RACI;
- business, editorial, finance, legal/rights, data and security owners приняли критерии результата.

### Официальные источники

Использованы только записи из knowledge/SOURCES.md. Перед реализацией URL, дату обновления, тарифную и версионную применимость проверяют повторно.

- [Официальная пользовательская документация Bitrix24](https://helpdesk.bitrix24.ru/) — CRM, задачи, права, automation and Cloud.
- [Официальная документация REST API](https://apidocs.bitrix24.ru/) — интеграции, авторизация и ограничения.
- [REST API reference](https://apidocs.bitrix24.ru/api-reference/) — методы, параметры and scopes.
- [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/index.html) — официальный инструментарий.
- [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) — создание and configuration Smart Processes.
- [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) — штатная automation.
- [Бизнес-процессы](https://helpdesk.bitrix24.ru/open/23951294/) — выбор типа процесса.
- [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) — официальные способы подключения.
- [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) — datasets and limitations.
- [Маркетплейс Bitrix24](https://www.bitrix24.ru/apps/) — приложения с отдельной оценкой поставщика и безопасности.
- [Документация On-Premise](https://dev.1c-bitrix.ru/docs/portal.php) — коробочная разработка без изменения ядра.

### Ограничения и открытые вопросы

1. Cloud или On-Premise, тариф/редакция, версия, регион и фактические права.
2. Revenue streams, sales models, lead usage, advertiser/agency/brand relations and account ownership.
3. Master клиента, контента, rights, campaign, placement, event, contract, inventory, revenue, cost and payment.
4. CMS/DAM/MAM, ad platform, ticketing, ERP/1С, payment and analytics APIs, volumes, limits and support.
5. Content rights, approval, expiry, territories, channels and authoritative status source.
6. Audience identity, consent/legal basis, segmentation, suppression, retention and cross-channel rules.
7. Offer, inventory, handover, change request, acceptance, cancellation and repeat-sale policies.
8. Team/project rights, contractor access, downloads, exports and recertification.
9. BI grains, attribution model, currency, metric owners, freshness, control totals and DQ thresholds.
10. Test portal, release window, downtime, rollback, degraded mode, DR and support RACI.

Документ не подтверждает юридическую допустимость конкретного контента, рекламы, лицензии, коммуникации или обработки аудиторных данных. Такие решения требуют отдельной правовой, договорной и информационно-безопасностной проверки.
