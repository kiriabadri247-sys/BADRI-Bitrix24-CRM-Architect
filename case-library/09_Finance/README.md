# Finance CRM Architecture Case

Архитектурный кейс для синтетической финансовой организации, которая принимает обращения физических и юридических лиц, подбирает продукты, управляет заявками, согласованиями, договорами и клиентским сопровождением. Документ задаёт референсную CRM-модель, а не готовую конфигурацию и не финансовую core-систему.

> **Граница безопасности:** реальные банки, компании, клиенты, ФИО, телефоны, email, документы, реквизиты, номера счетов, платёжные данные, результаты проверок, CRM ID, webhook URL, токены и выгрузки не используются. Все примеры синтетические.

> **Архитектурная граница:** Bitrix24 отвечает за обращения, клиентский сервис, коммерческий lifecycle, координацию согласований и минимальную проекцию статусов. Core banking/финансовая система остаётся master продукта и исполнения, risk engine — расчёта и результата риск-проверки, ERP/1С — бухгалтерских фактов, ECM/ЭДО — юридически значимых документов по принятой политике.

> **Не юридическая гарантия:** требования к идентификации, проверкам, согласию, локализации, срокам хранения, банковской/коммерческой тайне, записи разговоров, электронным документам и доступу определяются уполномоченными специалистами конкретной организации и юрисдикции. Кейс фиксирует инженерные контроли, но не заменяет правовую, риск-, комплаенс- и ИБ-экспертизу.

Обозначения:

- **факт Bitrix24** — подтверждается официальным источником из `knowledge/SOURCES.md`;
- **решение архитектора** — целевой выбор кейса, подлежащий согласованию;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа/редакции, версии, прав, региона, приложения или внешней системы.

## 1. Business Context

### Модель финансового бизнеса

Референсный сценарий — компания, предлагающая физическим и юридическим лицам финансовые продукты и сопутствующий сервис. Конкретный продукт, регулируемые проверки и правила решения не моделируются: они определяются владельцами продукта, риска, комплаенса и core-систем.

| Аспект | Целевая интерпретация |
|---|---|
| Продукты | Синтетические категории: расчётный, сберегательный, кредитный, инвестиционный или сервисный продукт |
| Каналы продаж | Сайт, формы, телефония, офис, разрешённые сообщения, партнёрский канал |
| Коммерческая единица | Финансовая заявка/сделка на конкретный продукт и версию условий |
| Решение | Уполномоченный процесс/risk engine/core system, не CRM-робот |
| Исполнение | Core banking/финансовая система |
| Сервис | Обращение клиента с отдельным SLA и правами |

### Цели внедрения CRM

1. Не терять обращения и заявки.
2. Создать допустимую единую историю взаимодействий без неконтролируемых дублей.
3. Сделать стадии, согласования, сроки, владельцев и исключения наблюдаемыми.
4. Отделить предложение продукта от подтверждённого решения и исполнения.
5. Защитить документы и чувствительные данные ролевой моделью.
6. Получать сверяемую аналитику продаж, скорости обработки и сервиса.

```text
Channel -> CRM request -> Financial application -> Approval coordination
   -> Contract process -> Core execution -> Service
           |                    |               |
           v                    v               v
       Risk case           ECM/ERP/Core         BI
```

## 2. AS IS Problems

| Проблема AS IS | Проявление | Корневая причина | Целевой контроль |
|---|---|---|---|
| Разрозненные данные клиентов | Несколько профилей и разные статусы | Нет identity/master policy | Нормализация и conservative matching |
| Нет единого процесса продаж | Менеджеры используют разные стадии | Нет exit criteria | Управляемая воронка |
| Нет контроля согласований | Решения остаются в почте/чатах | Нет versioned approval | Approval entity и audit trail |
| Теряются документы | Несколько «финальных» копий | Нет document master/version | ECM reference и metadata |
| Нет аналитики | KPI считаются вручную | Нет grain и reconciliation | Metric dictionary |
| CRM хранит лишние документы | Желание «единого окна» | Не определена классификация | Data minimization |
| Менеджер меняет риск-статус | Нет field ownership | Искажение решения | Read-only projection from master |
| Условия продукта устаревают | Скопированы вручную | Нет version/validity | Product master и revision |
| Сроки согласования не видны | Нет owner/escalation | Заявки зависают | SLA and approval monitoring |
| Сервис не связан с договором | Разные идентификаторы | Нет cross-system keys | Relations и external IDs |

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master |
|---|---|---|---|
| CRM | Лид | Неквалифицированное обращение, если lead-based процесс подтверждён | Bitrix24 |
| CRM | Контакт | Минимальный профиль физического лица | Bitrix24/MDM/core по ADR |
| CRM | Компания | Юридическое лицо/корпоративный клиент | Bitrix24/MDM/core |
| CRM | Сделка | Коммерческая возможность по продукту | Bitrix24 |
| Service | Обращение | Вопрос или запрос сопровождения | Bitrix24 |
| Domain | Smart Process «Финансовая заявка» | Координация заявки и статуса решения | Core/application system master; Bitrix24 projection |
| Catalog | Smart Process «Продукт» | Разрешённая коммерческая проекция продукта | Product/core system |
| Contract | Smart Process «Договор» | Metadata и lifecycle договорного процесса | ECM/ЭДО/core/ERP |
| Governance | Smart Process «Согласование» | Маршрут, версия, участники, решение | Bitrix24/core approval system по ADR |
| Risk | Smart Process «Риск-кейс» | Минимальный статус и coordination context | Risk engine/risk system |
| Service | Smart Process «Обращение клиента» | Управляемый сервисный lifecycle | Bitrix24 |

**Факт Bitrix24:** CRM, Smart Processes, роботы, триггеры и бизнес-процессы описаны официальными материалами. Конкретные поля, связи, права, automation и доступность проверяются на целевом портале.

**Решение архитектора:** Bitrix24 получает минимальные внешние ID, категории продукта, версии условий, статусы и владельцев. Полные идентификационные документы, результаты регулируемых проверок, risk features/scores и финансовые секреты не входят в референсную CRM-модель.

```text
Lead -> Contact/Company -> Deal -> Financial Application
                                  |       |          |
                                  v       v          v
                               Product  Approval  Risk Case
                                           |          |
                                           +----+-----+
                                                v
                                         Contract process
                                                |
                              Core/ERP/ECM <----+----> Service/BI
```

### Правила целостности

- один клиент имеет устойчивый `external_customer_id`, но matching не выполняется по одному слабому полю;
- одна заявка имеет `external_application_id` master-системы;
- решение связано с конкретной версией заявки и условий;
- менеджер не изменяет подтверждённый риск- или финансовый результат;
- повторное событие не создаёт вторую заявку, договор, решение или оплату;
- для каждого поля определены master, direction, classification, freshness и conflict rule;
- массовые операции имеют dry-run, sample, audit report, reconciliation и rollback.

## 4. Customer Journey Architecture

| Этап | Цель | Данные | Контроль |
|---|---|---|---|
| Обращение клиента | Зарегистрировать потребность | Канал, минимальный профиль, category, owner | Validation/dedup |
| Квалификация | Определить допустимый маршрут | Client type, product category, next step | Менеджер не обещает решение |
| Подбор продукта | Выбрать применимые варианты | Product IDs/versions, category, constraints | Product master/validity |
| Расчёт условий | Получить подтверждённый расчёт | Calculation ID, amount/range, term, validity | Approved engine/source |
| Согласование | Получить решения полномочных ролей | Application version, route, decisions | Segregation/limits |
| Договор | Согласовать и оформить актуальную версию | Contract ID/version/status | ECM/core master |
| Исполнение | Получить подтверждённый результат | Core operation/product status | Core system fact |
| Сопровождение | Управлять обращениями и изменениями | Case, SLA, owner, result | Service/finance/risk separation |

```text
Обращение клиента -> Квалификация -> Подбор продукта
 -> Расчёт условий -> Согласование -> Договор
 -> Исполнение -> Сопровождение

Exception: отказ / запрос данных / пауза / пересогласование / сервисный инцидент
```

CRM-стадия не равна юридическому или финансовому факту. Положительное действие менеджера не означает одобрение заявки, активацию продукта или исполнение обязательства.

## 5. Finance Data Model

### Основные сущности

| Сущность | Минимальные поля | Ограничение |
|---|---|---|
| Клиент | External ID, client type, segment, communication preferences, relationship owner | Не хранить лишние документы и результаты проверок |
| История взаимодействий | Request/case IDs, channel, category, status, owner, dates | Не копировать sensitive payload в комментарии |
| Заявка | Application ID, product/version, amount/range, term, status, owner, source timestamp | Decision details остаются в master |
| Сделка | Stage, responsible, expected value/date, product relation, document status | Не подменяет application/core contract |
| Продукт | Product ID, category, version, validity, customer category, public conditions | Product master authoritative |
| Согласование | Approval ID, object/version, step, role, decision/status, timestamp | Решение не редактируется после фиксации без нового цикла |
| Риск-кейс | Risk case ID, application, status category, owner, decision reference | Scores/features/check details не копируются |
| Договор | Contract ID, version, status, parties refs, secure link | Original remains in approved system |

```text
Contact/Company 1 ---- N Deal 1 ---- N Financial Application
       |                                  |       |
       +---- N Service Case               |       +---- N Approval
                                          +------------ 1 Product Version
                                          +------------ 0..N Risk Case
                                          +------------ 0..N Contract Version
```

### Идентичность и версии

| Отношение | Кардинальность | Правило |
|---|---:|---|
| Customer → Application | 1:N | Новая самостоятельная заявка не перезаписывает прошлую |
| Application → Product version | N:1 | Условия воспроизводимы по revision |
| Application → Approval | 1:N | Каждый цикл имеет version и reason |
| Application → Risk case | 1:N | Повторная проверка хранит отдельный результат/reference |
| Application → Contract | 1:N versions | Определяется актуальная версия |
| Customer → Service case | 1:N | Сервис не меняет заявку напрямую |

## 6. Sales Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новая заявка | Зарегистрировать request/application | Channel, client ID, product category, owner | Routing и first-action task |
| Квалификация | Подтвердить тип клиента и scope | Client type, request category, next step | SLA и missing-data task |
| Анализ потребности | Зафиксировать коммерческие параметры | Product need, amount/range, term, constraints | Task, no automated eligibility promise |
| Подготовка предложения | Получить versioned proposal | Product/version, calculation ID, validity | Follow-up and expiry control |
| Проверка условий | Выполнить формальные проверки completeness | Application revision, checklist statuses | Route to approved systems/roles |
| Согласование | Получить полномочные решения | Approval route, limits, decisions, timestamps | Timeout/escalation, segregation |
| Договор | Оформить актуальную версию | Contract ID/version, parties refs, status | Document workflow and version guard |
| Исполнение | Получить core-system fact | Core operation ID/status/date | Idempotent status projection |
| Закрытие | Завершить CRM-обязательство | Result source, open cases, owner | Completeness and service handoff |

```text
Новая заявка -> Квалификация -> Анализ потребности
 -> Подготовка предложения -> Проверка условий -> Согласование
 -> Договор -> Исполнение -> Закрытие

Branches: запрос данных / отказ / пауза / пересогласование / отмена
```

Каждая стадия имеет exit criteria. Робот не принимает риск-, комплаенс-, кредитное, инвестиционное или иное регулируемое решение; он может только маршрутизировать подтверждённый статус.

## 7. Financial Products Architecture

| Компонент | Содержание | Master | CRM-проекция |
|---|---|---|---|
| Продукт | Product ID, category, lifecycle | Core/product system | Разрешённое название/категория/status |
| Тариф | Tariff ID, version, validity | Pricing/product system | Reference и public/commercial projection |
| Условия | Parameter set и eligibility context | Approved calculation/rules engine | Calculation ID/result summary only |
| Категория клиента | Approved segmentation/classification | MDM/product system | Minimal category for routing |
| Дополнительная услуга | Service ID, compatibility/version | Product catalog | Relation to offer/application |

Текущая карточка продукта не переписывает исторические условия заявки. Каждое предложение связывается с `product_version`, `tariff_version`, `calculation_id` и датой действия. Название продукта не используется как технический ключ.

## 8. Approval Process Architecture

### Маршрут согласования

| Элемент | Требование | Контроль |
|---|---|---|
| Объект | Application/offer/contract + version | Нельзя согласовать «заявку вообще» |
| Роль | Product, risk, legal, finance, management as required | Named role and substitute |
| Лимит | Threshold/authority from approved policy | Не хранить policy logic в комментарии |
| Решение | Approved/rejected/request changes | Immutable event/reference |
| Срок | Due date/calendar | Timeout and escalation |
| Повтор | New version/new cycle | Предыдущее решение сохраняется |

```text
Submit version -> Validate completeness -> Route by approved matrix
 -> Parallel/Sequential approvals -> Consolidate result
      |                 |
      |                 +-> Reject / Request changes
      v
Approved version -> Contract/Execution gate
```

Конкретные лимиты и правила не приводятся. Они принадлежат утверждённой политике/системе правил. Bitrix24 координирует маршрут только после проверки доступности и полномочий.

## 9. Risk Management

| Компонент | CRM-назначение | Master | Ограничение |
|---|---|---|---|
| Risk case | Координационная карточка/status | Risk system | Не хранить расчётные features/scores |
| Проверка | External check ID/category/status | Approved system | Payload/results по allowlist |
| Статус | Requested/In review/Decision issued/Expired | Risk system | Read-only projection |
| Решение | Decision reference/version/timestamp | Authorized risk process | Менеджер не редактирует |
| Exception | Request for clarification/review | Risk owner | Нельзя обходить standard control |

```text
Application version -> Risk request -> Authorized checks
 -> Decision reference -> Approval gate
        |                    |
        +-> Need data        +-> Approved / Rejected / Other policy result
```

CRM не рассчитывает риск и не рекомендует решение. Любая автоматизация использует только подтверждённый статус из master-системы, трассирует source/version и обрабатывает unknown/expired state безопасно.

## 10. Document Architecture

| Документ | Владелец | Master/хранилище | CRM-проекция |
|---|---|---|---|
| Договор | Legal/product owner | ECM/ЭДО/core | ID, type, version, status, secure link |
| Заявление | Application owner | Core/application system/ECM | ID, version, completeness status |
| Расчёт | Product/pricing owner | Calculation engine/core | Calculation ID, validity, result summary |
| Подтверждающий документ | Authorized verification process | Secure DMS/core | Minimal checklist/reference, not copy |
| Внутреннее согласование | Functional approvers | Approval system/Bitrix24 by ADR | Object version, role, decision reference |
| Счёт/акт | Finance | ERP/1С/ECM | External ID, amount/status as required |

Оригиналы, подписи и доказательства действительности остаются в назначенных системах. CRM хранит metadata, связи и разрешённую ссылку. Прикрепление файла к сделке не является достаточным контролем версии, полномочий или хранения.

## 11. Rights Model

| Роль | Клиенты | Сделки | Документы | Риски | Аналитика |
|---|---|---|---|---|---|
| Менеджер | Свои/разрешённые | Свои/командные по scope | Коммерческие metadata | Только минимальный статус | Личная/командная |
| Аналитик | Минимизированные данные/агрегаты | Разрешённый analytical scope | Без оригиналов, если не требуется | Агрегированный/разрешённый status | Аналитическая |
| Риск-менеджер | Минимум для decision process | Связанные applications | Verification references by need | Назначенные risk cases | Риск-процесс по scope |
| Юрист | Стороны/контекст по необходимости | Связанные сделки | Договорные версии и решения | Только при необходимости | Юридическая операционная |
| Бухгалтер | Реквизиты/финансовый контекст | Исполнение/сверка | Счета, акты, finance metadata | Без risk details | Финансовая |
| Руководитель | Подразделение | Контроль и решения по полномочиям | Только необходимые документы | Status/aggregates | Управленческая |
| Администратор | Технически необходимый scope | Настройка, не бизнес-решения | Технический доступ по процедуре | Не принимает risk decision | Техническая диагностика |

Точная реализация CRM-ролей, прав Smart Processes, отделов, групп, наследования и экспорта требует проверки на целевом портале. Администратор не используется как повседневная бизнес-роль.

Контроли: deny-by-default для документов/risk data, least privilege, segregation of duties, service-account minimum scope, just-in-time emergency access if approved, offboarding, access review и тест UI/direct URL/search/REST/export.

## 12. Automation Architecture

| Механизм | Сценарий | Защита | Ошибка/эскалация |
|---|---|---|---|
| Робот/задача | Первый контакт | Один marker на request | Fallback owner |
| Робот | Проверка completeness | Approved field checklist/version | Return for completion |
| Уведомление | Срок согласования | Minimal context/link | Delivery failure recorded |
| Задача | Действие согласующего | Named role, deadline, outcome | Substitute/escalation |
| Бизнес-процесс | Approval coordination | Object version/authority matrix | Reject/timeout/new cycle |
| Робот | Проекция core status | Trusted source/event ID | Unknown state → exception |
| Робот/задача | Service follow-up | Eligibility and channel policy | No financial promise |

```text
Event -> Authenticate -> Validate source/version -> Check authority
 -> Execute coordination -> Record immutable result/reference
        |                                  |
        +-> reject safely                  +-> bounded retry/escalation
```

Каждая automation имеет owner, trigger, preconditions, classification, idempotency key, timeout, retry budget, error route, audit trail и test case. Автоматизация не создаёт одобрение, если нет подтверждённого решения уполномоченного контура.

## 13. Integration Architecture

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| Core/банковская система | Customer/application/product/contract/execution status по allowlist | REST/OAuth/application/event/batch contract if supported | Auth, idempotency, version, privacy filter, reconcile |
| Risk/decision system | Risk case ID/status/decision reference | Controlled API/event | Trusted source, expiry, sequence, no feature payload |
| ERP/1С | Counterparty, invoice, act, payment/accounting facts | Verified connector/application or contract integration | External IDs, mapping, queue, totals |
| Сайт | Request and permitted contact/context | CRM form/REST after verification | Validation, dedup, spam control |
| Телефония | Call event and CRM relation | Official Bitrix24 telephony/verified provider | Rights, unmatched calls, recording policy |
| BI/DWH | Approved minimal datasets | BI Constructor/controlled export | Access, minimization, freshness, reconcile |
| ЭДО/ECM | Document ID/version/status/signing workflow | API/application/secure link | Authenticity, version, access, retry |

```text
Source -> Authenticate -> Minimize -> Validate -> Map -> Idempotency -> Target
  |                                                                  |
  +-> secured audit                                      target result
                 \                                       /
                  +------ retry / reconciliation --------+
```

Контракт фиксирует source of truth, purpose, approved scope, direction, keys, schema/version, classification, retention, currency/timezone, event ordering, timeout, retry budget, rate limits, security, monitoring, recovery и owner.

Наличие готовой интеграции с конкретной core-системой, risk engine, ЭДО или конфигурацией 1С не предполагается. Приложение проверяется в официальном Маркетплейсе, внешний API — в документации поставщика отдельным scope.

## 14. Analytics

| Отчёт | Метрика | Grain | Ограничение/сверка |
|---|---|---|---|
| Продажи продуктов | Applications/contracts/executed products | Product/application/cohort | Core execution fact |
| Конверсия | Request → application → decision → contract → execution | Cohort/application | Одни eligibility правила |
| Скорость обработки | Stage/approval/application cycle time | Application/version | Event history and calendars |
| Эффективность менеджеров | SLA, qualified pipeline, completed outcomes | Manager-period | Не считать решение risk как заслугу менеджера |
| Прибыльность | Revenue/cost/margin | Product/customer cohort | Finance/core, not CRM estimate |
| Качество сервиса | Response, resolution, repeat contacts, permitted feedback | Case-period | Не смешивать с risk/product result |
| Согласования | Queue, cycle, rejection/rework | Approval/version | Versioned decisions |
| Интеграции | Error/lag/reconciliation delta | Flow-period | Technical monitoring |

```text
CRM + Core + Risk + ERP/1C + ECM
 -> Purpose-limited contracts -> Quality/security gate
 -> Dataset/DWH -> Metric dictionary -> Dashboard -> Decision
                                      ^                |
                                      +-- reconcile ---+
```

BI Constructor, datasets и поля зависят от портала. Metric dictionary фиксирует formula, grain, cohort, filters, timezone, currency, classification, owner, freshness и reconciliation source. Аналитический доступ предоставляется к минимизированным/агрегированным данным по утверждённой модели.

## 15. Security Architecture

### Инженерные принципы

| Принцип | Реализация | Проверка |
|---|---|---|
| Data minimization | Field allowlist для каждого процесса и потока | Data inventory/owner review |
| Segregation of duties | Продажа, risk decision, legal и finance разделены | Approval/role tests |
| Least privilege | Минимальные entity/field/export/API права | Access matrix testing |
| Document protection | Оригиналы в secure master, CRM — metadata/reference | Direct-link/access review |
| Secret management | Tokens/webhooks вне CRM notes, Git и логов | Secret scan/rotation procedure |
| Auditability | События, версии и решения трассируемы | Audit sampling/RCA |
| Retention | Сроки определяет утверждённая политика | Scheduled review/process |
| Safe analytics | Минимизация, агрегирование, controlled export | Dataset review |

```text
Collect minimum -> Classify -> Authorize -> Process by purpose
 -> Audit safely -> Retain by policy -> Restrict/Delete by approved process
```

Кейс не утверждает соответствие законодательству, банковскому, инвестиционному, платёжному или иному отраслевому регулированию. До внедрения обязательны решения Legal, Compliance, Risk, Information Security, Data Owners и Internal Control, включая выбор Cloud/On-Premise.

## 16. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Новое обращение | Validation, dedup, owner | Один request и первый шаг |
| 2 | Физлицо/юрлицо | Entity routing and required context | Correct Contact/Company model |
| 3 | Повторная заявка | Existing customer/new application | Профиль не дублирован |
| 4 | Версия продукта | Product/tariff validity | Proposal linked to exact version |
| 5 | Расчёт условий | Calculation ID/version | CRM не пересчитывает master result |
| 6 | Completeness check | Required field/document references | Неполная заявка не ушла на решение |
| 7 | Approval within limit | Authority/version | Решение принято допустимой ролью |
| 8 | Approval above limit | Escalation route | Недостаточная роль не подтверждает |
| 9 | Изменение заявки | New version/new approval cycle | Старое решение сохранено |
| 10 | Risk status event | Source/idempotency/expiry | Read-only projection updated once |
| 11 | Unknown/expired risk state | Safe mapping | Исполнение не запущено |
| 12 | Договор | Version/rights/current status | Актуальная версия определена |
| 13 | Core execution | External ID/repeat event | Факт отражён без дубля |
| 14 | Обращение клиента | Routing/SLA/relation | Case owner и deadline корректны |
| 15 | Проверка документов | UI/direct URL/REST/export | Только разрешённая роль имеет доступ |
| 16 | Ошибка интеграции | Timeout/retry/dead event | Нет потери/дубля, alert трассируем |
| 17 | События не по порядку | Sequence/version | Статус не откатился |
| 18 | Миграция | Dry-run/allowlist/sample/totals | Только разрешённые поля перенесены |
| 19 | BI dataset | Classification/access/grain | Sensitive fields исключены по design |
| 20 | Rollback | Trigger/runbook/reconcile | Безопасное состояние восстановлено |

Тестовые данные только синтетические. UAT подписывают Sales, Product, Risk, Compliance, Legal, Finance, Security, Data и Integration owners. Негативные, повторные, конкурентные, role, recovery и segregation-of-duties scenarios обязательны.

## 17. Deployment

| Этап | Действия | Gate |
|---|---|---|
| Discovery | AS IS, portal, systems, classification, owners, constraints | Inventory/scope signed |
| Архитектура | Data model, master matrix, rights, ADR, risks | TO BE and control owners approved |
| Настройка | Test portal/funnel, fields, stages, roles | Peer/security review and smoke |
| Миграция | Minimize, map, dry-run, sample, reconcile | Data owner acceptance |
| Интеграции | Contract, security, failure/recovery tests | End-to-end successful |
| Обучение | Role-based and prohibited-data guidance | UAT/readiness sign-off |
| Запуск | Freeze, backup/export, cutover, smoke | Go/no-go |
| Hypercare | Monitoring, incident route, daily reconcile | No critical defects, support accepted |

```text
Discovery -> Classification/control review -> Test configuration
 -> Contract tests -> Sample migration -> UAT -> Go/No-Go
 -> Cutover -> Smoke -> Reconcile -> Hypercare
                  |
                  +-> Rollback by approved trigger
```

Cloud и On-Premise оцениваются отдельно Legal/Compliance/IB. Для коробочной версии дополнительно проверяются версия продукта/модулей, PHP, БД, файлы, cron, backup/restore, инфраструктура и безопасность. Ядро продукта не изменяется.

## 18. Monitoring

| Контроль | Сигнал | Owner | Реакция |
|---|---|---|---|
| Зависшие заявки | Нет owner/движения дольше policy | Sales/application owner | Assign, blocker review |
| Просроченные согласования | Approval deadline breached | Approval owner | Substitute/escalate |
| Расхождение CRM/core | Application/contract/status delta | Data/integration owner | Reconcile and source correction |
| Ошибки интеграций | Error/retry/dead event | Integration owner | Recover/reconcile/incident |
| Обращение без ответа | SLA threshold | Service lead | Reassign/escalate |
| Несанкционированная правка | Protected status changed | Security/data owner | Contain/audit |
| Документ доступен лишней роли | Access review finding | Security/document owner | Restrict/incident process |
| Финансовое расхождение | CRM projection ≠ ERP/core | Finance owner | Reconciliation |
| Аномальный экспорт/API | Security signal | Security owner | Contain, revoke, investigate |

SLI: success/error rate, latency, backlog age, retry count, last successful sync, stale age и reconciliation delta. Логи не содержат secrets, documents, scores, full identifiers или финансовые реквизиты.

## 19. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | CRM используется как core banking | Неверный scope | Неполный/неуправляемый финансовый контур | System context/core master |
| 2 | CRM рассчитывает риск | Automation без governance | Неверное решение | Risk engine/authorized process |
| 3 | Менеджер меняет risk result | Широкие права | Искажение решения | Read-only projection |
| 4 | Полный score/features хранятся в CRM | Желание единой карточки | Sensitive overcollection | Minimal status/reference |
| 5 | Дубли клиентов | Нет identity policy | Разорванная история | Conservative matching |
| 6 | Ошибочное объединение клиентов | Слабый ключ | Чужие заявки/документы | Human-controlled merge |
| 7 | Телефон/email — единственный ключ | Shared/changed data | Wrong identity | Approved multi-attribute resolution |
| 8 | Повтор event создаёт заявку | Нет idempotency | Duplicate processing | External application ID |
| 9 | Заявка перезаписана после решения | Нет versioning | Decision detached from inputs | Immutable revision/new cycle |
| 10 | Продукт устарел | Нет version/validity | Incorrect offer | Product master/version |
| 11 | Тариф без validity | Manual copy | Financial dispute | Tariff revision/date |
| 12 | Calculation не связан с заявкой | Несколько версий | Wrong conditions | Calculation ID/revision |
| 13 | Менеджер обещает одобрение | Sales pressure | Customer/reputational risk | Scripts and decision boundary |
| 14 | Approval без object version | Process design gap | Wrong version approved | Versioned approval |
| 15 | Approver превышает лимит | Нет authority matrix | Unauthorized decision | Limit/role check |
| 16 | Автор сам согласует | Нет segregation | Conflict/fraud risk | Separation of duties |
| 17 | Timeout считается approval | Optimistic fallback | Unauthorized execution | Exception/escalation |
| 18 | Unknown risk state = approved | Unsafe mapping | Invalid execution | Safe default/hold |
| 19 | Expired decision reused | Нет validity | Stale approval | Expiry/recheck |
| 20 | Events out of order | Async integration | Status rollback | Sequence/version |
| 21 | Contract wrong version | Multiple copies | Legal dispute | Current version master |
| 22 | Identity document attached to deal | Convenience | Excess access | Secure DMS/reference |
| 23 | Full bank/payment details in CRM | Overcollection | Security breach | Token/reference/minimization |
| 24 | Secret in webhook URL published | Misclassification | Compromise | Secret store/rotation |
| 25 | Core/risk payload in logs | Debug full payload | Sensitive leak | Allowlist/masking |
| 26 | Error alert includes document | No sanitization | Leak in notifications | Sanitized errors |
| 27 | Call recording without review | Default feature | Legal/privacy risk | Legal/IB decision |
| 28 | Marketing without eligibility | Missing consent/suppression | Complaint/legal risk | Approved communication policy |
| 29 | Risk manager sees unrelated clients | Broad role | Excess access | Need-to-know |
| 30 | Analyst receives row-level documents | Convenience | Data leak | Aggregated/minimized dataset |
| 31 | Administrator works as manager | Role confusion | Uncontrolled changes | Separate admin account |
| 32 | Service account is administrator | Fast setup | Large blast radius | Least scope |
| 33 | Terminated employee keeps access | No offboarding | Breach | Revoke/reassign/rotate |
| 34 | Retry infinite | No budget | Load/backlog | Bounded backoff |
| 35 | Retry non-idempotent | No event key | Duplicate contract/status | Idempotency |
| 36 | REST limit/timeout | Peak processing | Lag | Queue/backoff/verified batch |
| 37 | Free bidirectional editing | No field owner | CRM/core conflict | Master matrix |
| 38 | Full core export imported | No minimization | Sensitive overcollection | Field allowlist |
| 39 | Migration without dry-run | Rush | Mass corruption | Sample/report/rollback |
| 40 | No reconciliation | HTTP success treated as result | Silent loss | Totals/sample/delta |
| 41 | Backup contains unnecessary data | No classification | Exposure/retention risk | Classified backup policy |
| 42 | Backup not restored in test | Formal control only | Rollback fails | Restore drill |
| 43 | Tariff lacks required controls | Capability check missed | Architecture infeasible | Portal verification |
| 44 | Marketplace app not assessed | Trust listing | Vendor/security risk | Assessment/exit plan |
| 45 | On-Premise outdated | Old modules/PHP | Errors/vulnerabilities | Supported update plan |
| 46 | Conversion uses CRM stage as execution fact | Wrong source | Overstated sales | Core fact reconciliation |
| 47 | Profitability uses CRM estimate | No finance source | Wrong margin | ERP/core costs |
| 48 | Manager KPI rewards risk approval | Wrong attribution | Distorted behavior | Balanced outcome/quality metrics |
| 49 | Approval speed ignores rework/version | Wrong grain | Misleading KPI | Approval-cycle model |
| 50 | Dashboard lacks freshness | Stale data | Wrong decision | Last refresh/SLA |
| 51 | No integration owner | Project ended | Errors unhandled | Support RACI |
| 52 | No security incident route | Process undefined | Delayed containment | Approved incident plan |
| 53 | No concurrency test | Sequential QA only | Duplicate application/approval | Parallel scenarios |
| 54 | Staff not trained on prohibited data | Configuration-only rollout | Sensitive notes/docs | Role guidance |
| 55 | No hypercare | Immediate handoff | Incidents accumulate | Support window |

## 20. ADR

### ADR-FIN-001: Модель клиента

- **Контекст:** CRM нуждается в профиле, но identity/core data имеют повышенную чувствительность.
- **Решение:** Contact/Company хранит минимальную CRM-проекцию и external customer ID; core/MDM остаётся master.
- **Альтернативы:** полный core-profile в CRM; отдельный профиль на каждую заявку.
- **Последствия:** conservative matching, minimization и strict rights.
- **Статус:** Proposed; требует Legal/Compliance/IB/Data approval.

### ADR-FIN-002: Финансовая заявка

- **Контекст:** заявка имеет версии, проверки, решение и исполнение вне CRM.
- **Решение:** Smart Process — orchestration projection; application/core system — master.
- **Альтернативы:** сделка как единственная сущность; CRM как application system.
- **Последствия:** external ID, version, status mapping, idempotency.
- **Статус:** Proposed.

### ADR-FIN-003: Продукты и тарифы

- **Контекст:** условия меняются и должны быть воспроизводимы.
- **Решение:** product/core system — master; CRM хранит product/tariff revision и calculation reference.
- **Альтернативы:** ручные поля; CRM catalog as sole master.
- **Последствия:** validity, snapshot and mapping governance.
- **Статус:** Proposed.

### ADR-FIN-004: Документы

- **Контекст:** документы различаются по значимости, версии и доступу.
- **Решение:** originals in ECM/ЭДО/core; CRM stores metadata/reference by allowlist.
- **Альтернативы:** all files in deal.
- **Последствия:** classification, version, retention, direct-link testing.
- **Статус:** Proposed.

### ADR-FIN-005: Согласования

- **Контекст:** решение зависит от версии, роли и лимита.
- **Решение:** versioned approval entity/process with immutable decision reference and segregation.
- **Альтернативы:** chat/email approval; one checkbox.
- **Последствия:** authority matrix, timeout, new cycle on changes.
- **Статус:** Proposed.

### ADR-FIN-006: Модель прав

- **Контекст:** Sales, risk, legal, finance and admin need different scopes.
- **Решение:** deny-by-default, role-based least privilege, UI/URL/REST/export tests.
- **Альтернативы:** common role; administrator for operations.
- **Последствия:** test accounts, access review, offboarding.
- **Статус:** Proposed.

### ADR-FIN-007: Аналитическая архитектура

- **Контекст:** CRM lacks execution, risk and cost facts.
- **Решение:** purpose-limited datasets and metric dictionary, facts reconciled with core/ERP.
- **Альтернативы:** only CRM reports; full raw export.
- **Последствия:** classification, grain, freshness and data-quality gates.
- **Статус:** Proposed.

### ADR-FIN-008: Лиды или Application-First

- **Контекст:** не каждое обращение требует лида, а оформленная заявка уже имеет самостоятельный lifecycle.
- **Решение:** lead only for confirmed unqualified intake; application/deal flow for valid application event.
- **Альтернативы:** lead per application; Deal-First for all requests.
- **Последствия:** choice fixed before channel and KPI configuration.
- **Статус:** Open until Discovery.

## 21. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | CRM объявлена core banking | Неверный scope | Неполный финансовый контур | Core master/system context |
| 2 | CRM-робот одобряет продукт | Automation without governance | Unauthorized decision | Authorized engine/process |
| 3 | Risk score stored in Deal | Desire for single card | Sensitive overcollection | Status/reference only |
| 4 | Full check results copied to comments | Convenience | Data exposure | Risk system only |
| 5 | Manager edits risk status | Broad rights | Decision distortion | Read-only projection |
| 6 | One customer created per application | No identity policy | Duplicates | Stable external ID |
| 7 | Phone is sole key | Shared/changed phone | Wrong merge | Conservative matching |
| 8 | Email is sole key | Shared address | Wrong identity | Approved resolution |
| 9 | Company name is sole key | Non-unique/mutable | Wrong merge | MDM/external ID |
| 10 | Duplicates auto-merged without review | Cleanup pressure | Mixed customers | Human-controlled merge |
| 11 | Lead created for every core event | No event boundary | CRM noise | Business-event model |
| 12 | One deal for all customer products | Customer/process mixed | No product analytics | Deal/application unit |
| 13 | Application is only a task | Task replaces entity | No version/status | Smart Process projection |
| 14 | Application update overwrites inputs | No revision | Decision not reproducible | Immutable version |
| 15 | Repeated event creates application | No idempotency | Duplicate processing | External/event ID |
| 16 | Product name is technical key | No product ID | Wrong mapping | Stable product code |
| 17 | Current tariff overwrites history | No snapshot | Wrong contract/report | Tariff revision |
| 18 | Tariff has no validity | Manual reference | Stale offer | Validity dates |
| 19 | Calculation stored as free text | No engine reference | Cannot reproduce | Calculation ID |
| 20 | Amount has no currency | Incomplete model | Wrong analytics | Amount + currency |
| 21 | Manager calculates conditions manually | No master | Financial error | Approved engine/source |
| 22 | Proposal not linked to calculation version | Artifacts separated | Wrong conditions | Version relation |
| 23 | Sales stage means approval | Semantics mixed | False promise | Separate status/gate |
| 24 | Approval is one checkbox | Oversimplification | No role/version/audit | Approval entity |
| 25 | Object version not recorded | Generic approval | Wrong version approved | Version ID |
| 26 | Approver exceeds authority | No limit matrix | Unauthorized decision | Authority check |
| 27 | Author approves own request | No segregation | Conflict/fraud risk | Separation of duties |
| 28 | Timeout interpreted as approval | Unsafe default | Unauthorized execution | Exception/escalation |
| 29 | Rejected request edited and reused | No new cycle | Audit broken | New version/approval |
| 30 | Unknown risk state means success | Optimistic mapping | Invalid execution | Hold/exception state |
| 31 | Expired decision reused | No validity | Stale decision | Expiry/recheck |
| 32 | Events applied out of order | Async integration | Status rollback | Sequence/version |
| 33 | Contract stored as unversioned file | Convenience | Wrong document | ECM/version metadata |
| 34 | Identity document attached to Deal | Single-window desire | Excess access | Secure DMS/reference |
| 35 | Document copied across deals | No master | Divergent versions | One master/reference |
| 36 | Signing status entered manually | No ЭДО/core sync | False legal status | Master event |
| 37 | Full account/requisite data in CRM | Overcollection | Security risk | Field allowlist |
| 38 | Payment credentials stored in notes | Unsafe practice | Critical compromise | Never store; secure provider |
| 39 | Payment/core fact set by manager | No integration | False execution | Core/ERP confirmation |
| 40 | Repeat callback duplicates status/action | No event key | Duplicate effect | Idempotency |
| 41 | Webhook URL in README | Secret misclassification | Compromise | Secret store/env |
| 42 | Core payload logged fully | Debug convenience | Sensitive leak | Allowlist/masking |
| 43 | Error alert includes document | No sanitization | Leak via message | Sanitized error |
| 44 | No timeout | Happy path | Hung workers | Explicit timeout |
| 45 | Infinite retry | No budget | Load/backlog | Bounded backoff |
| 46 | Retry non-idempotent | No key | Duplicates | Event ID |
| 47 | Error silently ignored | No dead letter | Missing applications | Failed-event register |
| 48 | Bidirectional free editing | No field owner | Core/CRM conflict | Master matrix |
| 49 | Batch used without checking method | Speed focus | Partial failure | Verify limits/semantics |
| 50 | Manager sees all customers | Broad role | Excess access | Need-to-know |
| 51 | Analyst receives row-level documents | Convenience | Data exposure | Aggregate/minimize |
| 52 | Risk manager sees sales notes unrelated to decision | Copied role | Excess access | Separate views/scopes |
| 53 | Accountant sees risk results | No segregation | Excess access | Finance-only scope |
| 54 | Administrator works as manager | Role confusion | Uncontrolled changes | Separate accounts |
| 55 | Integration account is administrator | Fast setup | Large blast radius | Least scope |
| 56 | Direct URLs/REST/export not tested | UI-only QA | Access bypass | Multi-channel tests |
| 57 | Terminated employee keeps access | No offboarding | Breach | Revoke/reassign/rotate |
| 58 | Call recording enabled without review | Default setting | Legal/privacy risk | Legal/IB decision |
| 59 | Marketing ignores suppression | Lists not synchronized | Unwanted contact | Communication governance |
| 60 | Full core export imported | No minimization | Sensitive overcollection | Field allowlist |
| 61 | Real clients used in test | No synthetic dataset | Data breach | Synthetic-only QA |
| 62 | Full migration without dry-run | Rush | Mass corruption | Sample first |
| 63 | No source-target report | Opaque migration | No rollback/reconcile | Mapping log |
| 64 | No totals/sample reconciliation | HTTP success assumed | Silent omissions | Acceptance report |
| 65 | Backup includes unnecessary documents | No classification | Exposure | Classified backup |
| 66 | Backup never restored | Formal check | Rollback fails | Restore drill |
| 67 | Conversion uses stage as core fact | Wrong source | Overstated sales | Core reconciliation |
| 68 | Profitability uses expected value | No actual costs | Wrong margin | ERP/core finance data |
| 69 | Manager KPI rewards approvals | Wrong attribution | Perverse incentives | Balanced metrics |
| 70 | Approval speed ignores rework/version | Wrong grain | Misleading KPI | Cycle/version model |
| 71 | Service SLA mixes approval delays | Domains mixed | Wrong performance | Separate clocks |
| 72 | Dashboard lacks freshness | Stale data | Wrong decision | Last refresh/SLA |
| 73 | BI export contains raw identifiers | No dataset review | Excess access | Minimize/aggregate |
| 74 | Marketplace app accepted without assessment | Trust listing | Vendor/security risk | Review/exit plan |
| 75 | Tariff capability not checked | Discovery missed | Infeasible rights/workflow | Portal verification |
| 76 | On-Premise version unsupported | Upgrade ignored | Errors/vulnerabilities | Supported plan |
| 77 | Launch without control-owner sign-off | Configuration focus | Unaccepted risk | Formal gates |
| 78 | Training only for administrators | No role enablement | Sensitive data in notes | Role guidance/UAT |
| 79 | No integration/support owner | Project ends | Incidents unhandled | Support RACI |
| 80 | No hypercare | Immediate handoff | Issues accumulate | Support window |

## 22. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Подтверждены AS IS, данные, systems and constraints | Inventory, classification, stakeholders, capability check | Masters and control owners defined |
| 2. Архитектура | Согласован TO BE | Data model, master matrix, rights, ADR, risk register | Legal/Compliance/Risk/IB/Data approvals obtained as required |
| 3. CRM настройка | Работает минимальный sales/service contour | Field allowlist, stages, roles, dedup, tests | Smoke/rights tests successful |
| 4. Финансовые процессы | Работают application/product/approval/contract projections | Registries, versions, gates, exception rules | End-to-end synthetic journey passed |
| 5. Интеграции | Работают purpose-limited core/risk/ERP flows | Contracts, filters, retries, monitoring, reconcile | Failure/recovery tests successful |
| 6. Тестирование | Подтверждена готовность | Test report, security/control findings, UAT | No critical/high defects |
| 7. Запуск | Выполнен управляемый cutover | Runbook, backup, rollback, communication | Smoke/reconcile successful |
| 8. Оптимизация | Стабилизированы SLA, access and KPI | Baseline, access review, backlog, lessons learned | Support model accepted |

### Официальные источники

Используются только источники из `knowledge/SOURCES.md`; дата последней проверки реестра — 2026-08-05. Эти источники подтверждают возможности Bitrix24, но не заменяют юридические, финансовые и отраслевые требования, которые находятся вне реестра и потому не цитируются в кейсе.

| Источник | Применение |
|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | Пользовательские возможности CRM и Cloud |
| [Документация REST API](https://apidocs.bitrix24.ru/) | Интеграционная архитектура и ограничения |
| [REST API reference](https://apidocs.bitrix24.ru/api-reference/) | Проверка контрактов методов и полей |
| [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/) | Проверка вариантов интеграционного доступа |
| [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) | Проектирование application, product, contract, approval, risk и service views |
| [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) | Проверка automation |
| [Как выбрать тип бизнес-процесса](https://helpdesk.bitrix24.ru/open/23951294/) | Выбор approval/orchestration механизма |
| [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) | Проверка вариантов телефонии |
| [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) | Проверка datasets и аналитики |
| [Маркетплейс Bitrix24](https://www.bitrix24.ru/apps/) | Проверка приложений и коннекторов |
| [Документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php) | On-Premise возможности и ограничения |

Если функция, поле, связь, событие, REST-метод, приложение или ограничение не подтверждены официальным источником и целевым порталом, они помечаются **«требует проверки»** и не включаются в обязательства перед заказчиком.
