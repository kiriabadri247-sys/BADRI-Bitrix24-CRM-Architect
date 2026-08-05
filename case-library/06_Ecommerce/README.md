# E-commerce CRM Architecture Case

Архитектурный кейс для синтетического интернет-магазина. Компания принимает заявки и заказы через сайт, получает оплату, передаёт товары в доставку, обрабатывает обращения и возвраты, развивает повторные продажи. Документ задаёт референсную модель, а не готовую конфигурацию для копирования.

> **Граница безопасности:** реальные компании, покупатели, телефоны, email, адреса, платёжные реквизиты, CRM ID, webhook URL, токены и выгрузки не используются. Все примеры обезличены или синтетические.

> **Архитектурная граница:** Bitrix24 отвечает за профиль клиента, коммуникации, управляемый lifecycle заказа и сервиса. CMS/storefront остаётся master корзины до оформления; ERP/PIM — товара и цены по выбранной политике; ERP/WMS — остатков и исполнения; платёжный провайдер и учётная система — факта оплаты и возврата. Иной выбор допускается только через ADR после Discovery.

Обозначения:

- **факт Bitrix24** — подтверждается официальным источником из `knowledge/SOURCES.md`;
- **решение архитектора** — целевой выбор кейса, который необходимо подтвердить на проекте;
- **требует проверки** — зависит от Cloud/On-Premise, тарифа/редакции, версии, прав, приложения, региона или внешнего провайдера.

## 1. Business Context

### Модель бизнеса и каналы

Референсный сценарий — B2C-интернет-магазин с возможными корпоративными покупателями. Товар выбирается в каталоге, добавляется в корзину, заказ оформляется на сайте, затем проходит оплату, комплектацию, доставку, получение и сервис. Маркетинговые коммуникации выполняются только в разрешённых каналах и при наличии требуемого согласия.

| Аспект | Целевая интерпретация |
|---|---|
| Основной канал | Интернет-магазин/CMS |
| Дополнительные каналы | Телефония, формы, мессенджеры и приложения после проверки |
| Коммерческая единица | Один оформленный заказ с устойчивым внешним ID |
| Клиент | Contact для физлица; Company — для юридического лица |
| Исполнение | ERP/WMS и службы доставки по интеграционным контрактам |
| Сервис | Обращение, претензия или возврат со своим lifecycle |

### Путь клиента

```text
Awareness -> Visit -> Browse -> Cart -> Checkout -> Payment
 -> Fulfilment -> Delivery -> Support -> Repeat purchase
```

### Цели CRM

1. Не терять заказы и обращения независимо от канала.
2. Создать единую историю клиента без неконтролируемых дублей.
3. Обеспечить наблюдаемость оплаты, исполнения, доставки и сервиса.
4. Разделить операционные статусы разных систем и назначить их владельцев.
5. Управлять SLA и отклонениями, не маскируя ошибки автоматическим успехом.
6. Измерять конверсию, средний чек, повторные покупки, возвраты и каналы.

## 2. AS IS Problems

| Проблема AS IS | Проявление | Корневая причина | Целевой контроль |
|---|---|---|---|
| Потеря заявок | Нет owner и первого ответа | Каналы не имеют единого intake | Correlation ID, routing и alert |
| Нет единой истории клиента | Заказы создают новые профили | Нет identity/matching policy | Нормализация и правила дублей |
| Разрыв сайта и CRM | Статусы расходятся | Не определены master и события | Data contract и reconciliation |
| Нет контроля заказов | Ошибка видна после жалобы | Нет end-to-end monitoring | SLA, blockers и incident route |
| Нет аналитики повторных продаж | Покупки не связаны с клиентом | Нестабильный customer key | Устойчивый ID и metric dictionary |
| Остаток неактуален | Клиент покупает недоступный товар | Storefront и WMS не сверяются | Freshness и reservation policy |
| Оплата отмечается вручную | Статус недостоверен | CRM объявлена финансовым master | Provider/finance confirmation |
| Возвраты в переписке | Нельзя оценить срок и сумму | Нет отдельного lifecycle | Return entity и RACI |
| Поддержка не видит заказ | Системы не связаны | Нет cross-system ID | Order relation и разрешённая проекция |
| Маркетинг пишет без правил | Жалобы и репутационный риск | Consent и suppression не управляются | Consent gate и exclusion lists |

## 3. TO BE Architecture

### Целевые сущности

| Контур | Сущность | Назначение | Master-система |
|---|---|---|---|
| CRM | Лид | Неквалифицированное обращение, если lead-based процесс подтверждён | Bitrix24 |
| CRM | Контакт | Профиль физического лица и коммуникации | Bitrix24/CDP/MDM по ADR |
| CRM | Компания | Корпоративный покупатель | Bitrix24/ERP/MDM |
| CRM | Сделка | Коммерческая проекция заказа, если нужна CRM-воронка | Bitrix24 |
| Catalog | Товары | CRM-проекция каталога для продаж и документов | PIM/ERP/CMS по policy |
| Domain | Smart Process «Заказ» | Сквозной lifecycle оформленного заказа | CMS/OMS/ERP master; Bitrix24 orchestration view |
| Service | Smart Process «Возврат» | Запрос и исполнение возврата | Bitrix24/OMS/ERP по ADR |
| Service | Smart Process «Обращение клиента» | Вопрос или запрос поддержки | Bitrix24 |
| Quality | Smart Process «Претензия» | Формальное несоответствие с расследованием | Bitrix24, с фактами из master-систем |

**Факт Bitrix24:** CRM, Smart Processes, роботы, триггеры и бизнес-процессы описаны официальными материалами. Конкретные поля, связи, права, automation и доступность проверяются на целевом портале.

**Решение архитектора:** оформленный заказ имеет устойчивый `external_order_id` и одну систему-владельца. Bitrix24 хранит только необходимую для продаж и сервиса проекцию, не подменяя checkout, эквайринг, склад или бухгалтерский учёт.

```text
Customer -> Site/CMS -> Cart -> Checkout -> Order/OMS
    |                                  |       |
    v                                  v       v
Bitrix24 CRM <-------------------- Order view  Payment provider
    |                                      |
    +-> Support/Claim/Return                v
                                  ERP/WMS -> Delivery
                                       |
                                       v
                                  BI / Reconcile
```

### Правила целостности

- один оформленный заказ — один устойчивый `external_order_id`;
- корзина и заказ не считаются одной сущностью;
- повтор webhook/event не создаёт второй заказ, платёж, возврат или сообщение;
- финансовый успех подтверждается провайдером/учётной системой, не текстом менеджера;
- остаток и резерв подтверждаются WMS/ERP/OMS согласно master policy;
- для каждого поля определены owner, direction, freshness и conflict rule;
- массовые операции используют dry-run, sample, журнал, reconciliation и rollback.

## 4. Customer Journey Architecture

| Этап | Событие | Данные | Контроль |
|---|---|---|---|
| Посещение сайта | Anonymous/session visit | Consent-aware analytics ID | Не создавать CRM-клиента без основания |
| Заявка | Форма/звонок/чат | Контакт, вопрос, канал | Validation, dedup, owner |
| Корзина | Товары выбраны, checkout не завершён | Cart ID, items, timestamp | CMS master; consent для коммуникации |
| Заказ | Checkout подтверждён | Order ID, customer, lines, totals | Idempotent create |
| Оплата | Provider status | Payment ID, amount, currency, state | Signature/auth validation и reconciliation |
| Доставка | Shipment accepted | Shipment ID, carrier, status, dates | Normalized status mapping |
| Получение | Delivery confirmed | Fact date, result | Source and dispute handling |
| Повторная покупка | Новый независимый заказ | Customer ID, order ID | История связана без перезаписи |

```text
Посещение сайта -> Заявка -> Корзина -> Заказ
 -> Оплата -> Доставка -> Получение -> Повторная покупка

От любого post-order этапа -> Support / Return / Claim
```

Переход между этапами основан на подтверждённом событии. Аналитический путь не должен автоматически становиться операционной стадией CRM: просмотр и корзина могут оставаться в web analytics/CMS, пока не возникло допустимое бизнес-событие.

## 5. E-commerce Data Model

### Основные сущности и поля

| Сущность | Ключевые поля | Контроль качества |
|---|---|---|
| Клиент | customer ID, segment, consent, first/last purchase dates, aggregate history | Предпочтения выводятся по утверждённым правилам; не хранить лишние данные |
| Заказ | external ID, lines, quantity, amount, currency, status, delivery, payment summary | Totals и статусы имеют источник/время |
| Order line | product ID, variant, quantity, unit price, discount, tax context | Снимок условий заказа не заменяет текущую цену каталога |
| Товар | product ID, SKU, category, attributes, active state | Master и версия mapping определены |
| Оплата | payment ID, order ID, amount, currency, status | Минимальная проекция без платёжных секретов |
| Доставка | shipment ID, carrier, address reference, plan/fact, status | Адрес защищён; status mapping документирован |
| Возврат | return ID, order/line, quantity, reason, status, amount | Нельзя вернуть больше допустимого количества |
| Обращение | case ID, customer/order, category, priority, SLA, owner | Канал и next action обязательны |
| Претензия | claim ID, order/item, severity, evidence reference, decision | Жалоба и подтверждённый дефект разделены |

```text
Contact/Company 1 ---- N Order 1 ---- N Order Line N ---- 1 Product
                         |  |  |
                         |  |  +---- N Payment event
                         |  +------- N Shipment
                         +---------- N Return / Case / Claim
```

### Кардинальность и идентичность

| Отношение | Кардинальность | Правило |
|---|---:|---|
| Customer → Order | 1:N | Повторная покупка — новый заказ |
| Order → Payment | 1:N | Возможны попытки/части; успешный факт сверяется |
| Order → Shipment | 1:N | Частичная доставка не теряется |
| Order line → Return line | 1:N | Возвращённое количество контролируется |
| Customer → Case | 1:N | Одно обращение — один управляемый вопрос |
| Order → Claim | 1:N | Претензия связана с конкретным контекстом |

## 6. Sales Pipeline

| Стадия | Цель | Обязательные данные | Автоматизация |
|---|---|---|---|
| Новый заказ | Принять оформленный заказ | Order ID, customer, lines, totals, source time | Idempotent create и owner/fallback |
| Проверка данных | Проверить полноту и риск | Контакты, доставка, товары, totals | Validation и controlled exception task |
| Подтверждение заказа | Зафиксировать готовность к исполнению | Confirmation source/time, reservation context | Уведомление и переход только по результату |
| Оплата | Получить допустимый финансовый статус | Payment ID/state, amount, currency | Provider event validation, reconcile |
| Передача в доставку | Подтвердить отгрузочный handoff | Shipment ID, carrier, quantity, plan date | Task/notification и status mapping |
| Доставлено | Получить факт доставки | Provider status, fact time, exceptions | Запустить контроль получения/сервиса |
| Закрыто | Подтвердить завершение обязательства | Final status, returns/claims context | Follow-up по consent; completeness check |

```text
Новый заказ -> Проверка данных -> Подтверждение заказа -> Оплата
 -> Передача в доставку -> Доставлено -> Закрыто

Exception branches: Cancelled / Payment failed / Delivery problem / Return / Claim
```

Стадия отражает бизнес-результат, а не нажатие кнопки. Предоплата, постоплата, частичная оплата и наложенный платёж проектируются как отдельные сценарии после Discovery, а не смешиваются одним флагом.

## 7. Product Architecture

| Компонент | Содержание | Master | Правило CRM-проекции |
|---|---|---|---|
| Каталог | Product/SKU/variant | PIM/ERP/CMS по ADR | Устойчивый ID, только необходимые поля |
| Категории | Иерархия merchandising | PIM/CMS | Не использовать название как ключ |
| Характеристики | Цвет, размер и иные атрибуты | PIM | Нормализованные значения/версии |
| Цена | Price type, amount, currency, validity | ERP/PIM/pricing engine | Timestamp и применённая цена заказа |
| Остатки | Available/reserved by location | ERP/WMS/OMS | Проекция с warehouse context и freshness |

**Рекомендация архитектора:** CRM-каталог используется для коммерческого контекста, если его возможности подтверждены и согласованы. Он не становится автоматически PIM или WMS. Для заказа сохраняется снимок SKU, цены и скидки на момент оформления; текущая карточка товара не должна переписывать историю заказа.

## 8. Order Management

### Lifecycle

| Операция | Preconditions | Результат | Контроль |
|---|---|---|---|
| Создание | Valid checkout event | Один заказ с external ID | Idempotency и totals |
| Изменение | Допустимый статус и версия | Новая версия/событие заказа | Optimistic/version check |
| Отмена | Полномочия и причина | Cancelled state, связанные действия | Оплата, резерв, доставка согласованы |
| Возврат | Получение/право/quantity rule | Return lifecycle | Не считать request фактом возврата денег |
| Повторный заказ | Новый checkout | Новый order ID | Связать с клиентом, не клонировать статусы |

```text
Create -> Validate -> Confirm -> Pay -> Fulfil -> Ship -> Deliver -> Close
   |          |         |        |       |
   +-> Cancel +-> Hold  +-> Fail +-> Partial
                                  |
                                  +-> Return / Claim
```

Изменение состава после оплаты требует согласованной модели: отмена/корректировка, перерасчёт, возврат и повторное подтверждение не должны выполняться несвязанными ручными правками.

## 9. Customer Support

| Тип | Когда использовать | Ключевые данные | Exit criteria |
|---|---|---|---|
| Вопрос | Нужна информация без формального несоответствия | Topic, order, channel, owner | Ответ дан и подтверждён следующий шаг |
| Обращение | Требуется управляемое действие | Category, priority, SLA, owner | Запрос выполнен/обоснованно закрыт |
| Претензия | Заявлено нарушение/качество | Severity, order/item, evidence, decision | Решение сообщено, последствия обработаны |
| Возврат | Запрошено обратное движение товара/денег | Lines, quantity, reason, status | Товар и финансовый результат сверены |

```text
Inbound -> Identify customer/order -> Classify -> Assign -> Resolve
   -> Confirm result -> Close -> Quality review
            |
            +-> Escalate / Claim / Return
```

SLA включает старт, допустимые паузы, календарь, приоритет, целевое время реакции/решения и escalation route. Реализация SLA **требует проверки** на портале; при отсутствии нужного штатного механизма применяется документированный контрольный процесс.

## 10. Marketing Automation

| Практика | Допустимое использование | Защита |
|---|---|---|
| Сегментация | По согласованным признакам и качественным данным | Data minimization, owner, freshness |
| Повторные продажи | После завершённого заказа и при допустимой коммуникации | Consent/suppression и frequency cap |
| Уведомления | Транзакционные события заказа | Отделить от маркетинговой рассылки |
| Коммуникации | Разрешённые каналы и шаблоны | Unsubscribe, audit, no secrets |
| Кампании | Измеримый audience, offer, period | Holdout/attribution по утверждённой методике |

Корзина не считается разрешением на маркетинговую коммуникацию. Триггер abandoned cart используется только после правовой, продуктовой и технической проверки. Персонализация не должна раскрывать содержимое заказа в небезопасном канале.

## 11. Delivery Architecture

| Элемент | Master | Данные в Bitrix24 | Контроль |
|---|---|---|---|
| Служба доставки | OMS/ERP | Provider code и service level | Справочник и активность |
| Отправление | WMS/OMS/carrier | Shipment ID, order, quantity, status | Idempotent update |
| Статус | Carrier/aggregator | Нормализованный status + source status | Mapping version |
| Срок | OMS/carrier | Planned/fact dates | Timezone и exception |
| Уведомление | CRM/notification service | Template/event result | Consent/channel policy и delivery status |

```text
Order ready -> Shipment create -> Carrier accepted -> In transit
 -> Delivered
        |
        +-> Delay / Failed attempt / Lost / Return to sender
```

CRM не выводит факт доставки только из плановой даты. Неизвестный статус провайдера сохраняется как exception и не преобразуется в успешный.

## 12. Payments Architecture

| Событие | Требуемые данные | Источник истины | Контроль |
|---|---|---|---|
| Payment initiated | Order/payment ID, amount, currency | Provider/checkout | Correlation ID |
| Authorized/paid | Provider state and timestamp | Provider + finance reconciliation | Authenticity and amount check |
| Failed | Code/category, retry eligibility | Provider | Не логировать чувствительный payload |
| Cancelled | Initiator/reason | Provider/OMS | Связанные order actions |
| Refund requested | Return/decision ID, amount | OMS/ERP | Approval and available amount |
| Refunded | Refund ID, amount, fact date | Provider/finance | Reconciliation and idempotency |

```text
Order -> Payment attempt -> Provider decision
             |                 |-> Paid -> Reconcile
             |                 |-> Failed -> Controlled retry/new attempt
             +-----------------|-> Cancelled

Return decision -> Refund request -> Provider result -> Finance reconcile
```

Bitrix24 не хранит платёжные секреты и полные карточные данные. Подлинность callback/webhook, конкретные подписи и API проверяются по документации платёжного провайдера в отдельном scope; они не выдумываются в данном кейсе.

## 13. Rights Model

| Роль | Клиенты | Заказы | Товары | Обращения | Аналитика |
|---|---|---|---|---|---|
| Оператор | Назначенные/очередь | Обработка разрешённых заказов | Чтение доступного каталога | Назначенные обращения | Личная/операционная |
| Менеджер | Свои/командные | Исключения и продажи | Чтение, без master-правок | Эскалации | Командная по scope |
| Маркетолог | Сегменты без лишних данных | Агрегированный контекст | Коммерческие признаки | Без приватной переписки, если не требуется | Каналы/кампании |
| Склад | Минимальный контекст получателя | Fulfilment/отгрузка | Остатки в master-системе | Только логистические исключения | Операционная |
| Руководитель | Подразделение | Контроль и решения по полномочиям | Разрешённый обзор | Эскалации | Управленческая |
| Администратор | Технически необходимый доступ | Настройка, не бизнес-решения | Настройка интеграции | Настройка | Техническая диагностика |

Точная реализация CRM-ролей, прав Smart Processes, групп, отделов, наследования и экспорта требует проверки на целевом портале. Администратор не должен использоваться как повседневная бизнес-роль.

Контроли: least privilege, service accounts с минимальным scope, MFA/политики портала по возможности, offboarding, access review и тест UI/URL/search/REST/export.

## 14. Integrations

| Система | Данные | Метод интеграции | Контроль ошибок |
|---|---|---|---|
| Сайт/CMS | Customer, cart/checkout, order | CRM form/REST/OAuth/application после проверки | Schema, dedup, correlation ID |
| PIM | Product/SKU/category/attributes | API/controlled batch | Version, mapping, inactive items |
| Платёжная система | Payment/refund state | Provider API/webhook + Bitrix24 integration layer | Authenticity, idempotency, reconcile |
| Доставка | Shipment/status/plan/fact | Carrier API/aggregator/application | Mapping, unknown states, delay alert |
| Склад/WMS | Stock/reservation/fulfilment | API/event/batch exchange | Unit/location/timestamp/totals |
| 1С/ERP | Orders, products, finance, inventory by scope | Officially verified connector/application or contract integration | External IDs, queue, retry, reconciliation |
| BI/DWH | Allowed datasets | BI Constructor/controlled export | Freshness, access, metric reconcile |
| Телефония | Call event and CRM relation | Official Bitrix24 telephony/verified provider | Unmatched calls, rights, recording policy |

```text
Source -> Authenticate -> Validate -> Map -> Idempotency -> Target
  |                                                     |
  +-> source event log                         target result
               \                                 /
                +---- retry / reconciliation ---+
```

Контракт включает source of truth, direction, keys, schema version, mapping, units, currency, timezone, create/update/cancel semantics, ordering, timeout, retry budget, rate-limit handling, personal-data policy, monitoring, recovery и owner.

Наличие готового коннектора конкретной CMS, платёжной системы, доставки, WMS или конфигурации 1С не предполагается. Приложение проверяется в официальном Маркетплейсе, а внешний контракт — в документации провайдера отдельным этапом.

## 15. Analytics

| Отчёт | Метрика | Grain | Контроль качества |
|---|---|---|---|
| Продажи | Orders, paid/delivered revenue | Order/period | Finance/order status reconciliation |
| Средний чек | Revenue / eligible orders | Order/cohort | Currency and cancellation rules |
| Конверсия | Visit/cart/checkout/order/paid | Session/customer/order cohort | Согласованные denominator и attribution |
| Повторные покупки | Repeat customers/orders | Customer/cohort | Stable customer identity |
| Возвраты | Return count/quantity/amount/rate | Return line/order | Request ≠ completed refund |
| Эффективность каналов | Qualified/paid orders and cost-aware return | Source/campaign | UTM quality and attribution policy |
| Доставка | On-time, delay, failed delivery | Shipment | Carrier mapping and timezone |
| Поддержка | Response/resolution/SLA/CSAT if collected lawfully | Case | Pause/calendar rules |

```text
CMS + CRM + ERP/WMS + Payment + Delivery
       -> Data contracts -> Quality gate -> Dataset/DWH
       -> Metric dictionary -> Dashboard -> Decision
                              ^                |
                              +-- reconcile ---+
```

BI Constructor, datasets и доступные поля зависят от портала. Каждая метрика имеет formula, grain, cohort, filters, timezone, currency, attribution, owner, freshness и reconciliation source.

## 16. Security Architecture

### 1. Role Based Access Control

Роли operator, sales manager, support, marketing, warehouse, finance, e-commerce head и administrator получают доступ по каналу и операции. Платёжные, возвратные, экспортные и административные полномочия отделяются от обработки заказа; least privilege применяется к пользователям и интеграциям.

| Роль | Рабочие объекты | Минимальный доступ |
|---|---|---|
| Operator / sales | Клиенты и заказы | Обработка разрешённой очереди без payment settings |
| Support | Обращения, возвраты, претензии | Только связанный клиентский контекст |
| Marketing | Сегменты и кампании | Минимизированные данные без финансовых реквизитов |
| Warehouse | Исполнение и отгрузка | Заказ и доставка без лишних CRM-данных |
| Finance | Оплата и возврат | Подтверждение authoritative status без CRM administration |

### 2. Department Isolation

Sales, support, marketing, warehouse и finance разделяются по ролям и operational scope. Каналы, магазины или регионы изолируются без создания дублей Customer 360. Междепартаментная передача заказа, возврата или претензии открывает только необходимые связанные элементы.

### 3. CRM Permissions

| Объект | Правило доступа |
|---|---|
| Лиды | Intake/marketing qualification, если lead model принят |
| Контакты | По клиентскому и support scope |
| Компании | Только B2B account roles, если применимо |
| Сделки | По заказу/воронке, ответственному и каналу |
| Smart Processes | По роли в заказе, возврате, обращении или претензии |

Особо тестируются экспорт, массовая коммуникация, изменение статусов оплаты/возврата и API-доступ. CRM не становится master платёжных credentials.

### 4. Document Security

Заказы, чеки/подтверждения, возвратные документы, обращения и вложения доступны по need-to-know. Платёжные credentials и полные чувствительные payload не сохраняются в CRM или документации. Файлы размещаются в утверждённом хранилище с контролем просмотра, изменения, скачивания и retention.

### 5. Audit

Отслеживаются изменения заказа, суммы, скидки, оплаты, доставки, возврата, ответственного и consent/communication status в пределах подтверждённых функций. CRM history и masked logs сайта, OMS, payment и delivery systems связываются correlation ID. Назначенные owners проводят reconciliation и RCA.

### 6. Cloud Limitations

- права, доступные журналы и ограничения массовых операций зависят от тарифа и конфигурации;
- возможности CRM, Smart Processes, marketing и integrations проверяются на конкретном Cloud-портале;
- доступа к server, PHP, core и DB нет;
- внешние payment/OMS/security controls не объявляются возможностями Bitrix24.

### 7. On-Premise Considerations

Администратор дополнительно отвечает за OS, web server, PHP, DB, modules, files, cron/agents, network, TLS, logs, updates и backup. Восстановление и обновления тестируются, service accounts и административные роли ограничиваются, ядро не изменяется.

---

## 17. Testing

| № | Сценарий | Проверка | Ожидаемый результат |
|---:|---|---|---|
| 1 | Заказ с сайта | Schema, dedup, lines, totals | Один заказ с external ID |
| 2 | Повтор события заказа | Idempotency | Дубликат не создан |
| 3 | Неизвестный товар | Mapping exception | Заказ не искажён, создан alert |
| 4 | Изменение заказа | Version и допустимый status | История и totals согласованы |
| 5 | Успешная оплата | Authenticity, amount, currency | Статус подтверждён и сверяем |
| 6 | Повтор payment callback | Event ID | Финансовый эффект не дублируется |
| 7 | Ошибка оплаты | Error route и retry policy | Заказ не помечен оплаченным |
| 8 | Частичная оплата | Amount aggregation | Остаток рассчитан по policy |
| 9 | Отмена заказа | Резерв, платёж, доставка | Связанные действия согласованы |
| 10 | Передача в доставку | Shipment ID/quantity | Один shipment и корректный статус |
| 11 | Неизвестный статус доставки | Mapping fallback | Нет ложного «Доставлено» |
| 12 | Задержка доставки | Threshold/escalation | Owner и клиентский сценарий запущены |
| 13 | Частичная доставка | Несколько shipment | Количества не потеряны |
| 14 | Возврат товара | Order line/quantity/reason | Return lifecycle создан корректно |
| 15 | Возврат денег | Refund ID/amount/repeat | Факт подтверждён без дубля |
| 16 | Повторная покупка | Existing customer/new order | История связана, контакт не дублирован |
| 17 | Обращение поддержки | Routing/SLA/order relation | Owner и deadline корректны |
| 18 | Проверка прав | UI/URL/REST/export | Роль видит только разрешённый scope |
| 19 | Ошибка интеграции | Timeout/retry/dead event | Нет потери/дубля, alert трассируем |
| 20 | Миграция клиентов | Dry-run/sample/totals | Mapping и дедупликация подтверждены |
| 21 | BI reconciliation | Order/payment/return sample | Расхождение объяснено или исправлено |
| 22 | Rollback релиза | Trigger/runbook | Предыдущее безопасное состояние восстановлено |

Тестовые данные только синтетические. UAT подписывают E-commerce, Sales, Support, Warehouse, Finance, Marketing и Integration owners. Обязательны positive, negative, repeat, concurrency, role и recovery cases.

## 18. Deployment

| Этап | Действия | Gate |
|---|---|---|
| Аудит | AS IS, portal capabilities, channels, systems, volumes | Inventory и scope подписаны |
| Архитектура | Data model, master matrix, lifecycle, RACI, ADR | TO BE утверждён |
| Настройка | Test portal/funnel, fields, stages, rights | Peer review и smoke |
| Интеграции | Contract/failure/recovery tests | End-to-end и reconcile успешны |
| Перенос данных | Mapping, clean, dry-run, sample, totals | Acceptance report подписан |
| Обучение | Role-based scenarios | UAT/readiness sign-off |
| Запуск | Freeze, backup/export, cutover, smoke | Go/no-go decision |
| Hypercare | Monitoring, incident route, daily reconcile | Нет critical defects, support принял систему |

```text
Audit -> Test configuration -> Integration test -> Sample migration
 -> UAT -> Go/No-Go -> Cutover -> Smoke -> Reconcile -> Hypercare
                              |
                              +-> Rollback by trigger
```

Cloud и On-Premise планируются отдельно. Для коробочной версии дополнительно проверяются версия продукта/модулей, PHP, БД, файлы, cron, backup/restore, инфраструктура и безопасность. Ядро продукта не изменяется.

## 19. Monitoring

| Контроль | Сигнал | Owner | Реакция |
|---|---|---|---|
| Необработанные заказы | Нет owner/перехода дольше policy | Operations lead | Назначить и провести RCA |
| Ошибки оплаты | Failed/unknown/reconcile delta | Payments owner | Проверить provider, retry/incident |
| Задержки доставки | Planned date breached/unknown status | Logistics owner | Carrier escalation и клиентский сценарий |
| Обращения без ответа | SLA threshold | Support lead | Reassign/escalate |
| Ошибки интеграций | Error/retry/dead event | Integration owner | Recover/reconcile |
| Расхождение остатков | Storefront ≠ WMS | Inventory owner | Safe availability и source correction |
| Дубли заказов/клиентов | Duplicate key/match alert | Data owner | Contain и исправить правило |
| Возврат без финансового результата | Return completed, refund unknown | Finance/returns owner | Reconcile и incident |
| Аномальный доступ/экспорт | Security signal | Security owner | Contain, audit, revoke |

SLI: success/error rate, latency, backlog age, retry count, last successful sync, reconciliation delta и data freshness. Логи не содержат секретов, полные контактные/адресные данные или платёжный payload.

## 20. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Дубли заказов | Повтор события | Двойное исполнение | Idempotency key |
| 2 | Дубли клиентов | Нет identity policy | Разорванная история | Normalization/matching |
| 3 | Корзина ошибочно стала заказом | События не разделены | Ложная выручка | Checkout confirmation gate |
| 4 | Заказ потерян | Нет retry/fallback | Потеря продажи | Queue, retry, alert |
| 5 | Товар не сопоставлен | Разные SKU | Неверный состав | Product mapping registry |
| 6 | Название товара — ключ | Нет stable ID | Merge вариантов | Product/SKU ID |
| 7 | Цена устарела | Нет validity/freshness | Убыток/жалоба | Pricing master/timestamp |
| 8 | История заказа переписана текущей ценой | Нет snapshot | Неверные документы | Order-line snapshot |
| 9 | Остаток устарел | Lag интеграции | Oversell | Freshness/reservation policy |
| 10 | Остаток без склада | Упрощённый mapping | Ложная доступность | Warehouse context |
| 11 | Оплата отмечена вручную | CRM объявлена master | Ложный paid state | Provider/finance confirmation |
| 12 | Повтор callback дублирует оплату | Нет event ID | Двойной эффект | Idempotency |
| 13 | Callback не проверен | Нет authenticity control | Подмена статуса | Provider-specific validation |
| 14 | Частичная оплата считается полной | Один boolean | Ошибка исполнения | Amount/state model |
| 15 | Возврат денег больше оплаты | Нет limit check | Финансовая потеря | Available refundable amount |
| 16 | Return request равен refund fact | Смешаны процессы | Неверная отчётность | Separate lifecycle/events |
| 17 | Отмена не снимает резерв | Контуры не связаны | Недоступный остаток | Saga/reconciliation rule |
| 18 | Отмена после отгрузки проходит как обычная | Нет status guard | Потеря товара | Allowed transition matrix |
| 19 | Неизвестный delivery status считается успехом | Плохой mapping | Заказ закрыт ошибочно | Exception state |
| 20 | Задержка не эскалируется | Нет threshold | Жалоба клиента | Delivery SLA monitor |
| 21 | Частичная доставка потеряна | Один shipment field | Неверный статус | 1:N shipments |
| 22 | Адрес доступен маркетологу | Широкая роль | Privacy breach | Need-to-know |
| 23 | Платёжные данные в CRM | Избыточный payload | Security/compliance risk | Data minimization/token reference |
| 24 | Секрет в webhook URL опубликован | Неверная классификация | Компрометация | Secret store/rotation |
| 25 | ПДн в логах | Payload logging | Утечка | Masking/retention |
| 26 | Маркетинг без consent | Нет consent gate | Жалобы/правовой риск | Consent/suppression |
| 27 | Отписка применяется с задержкой | Асинхронный lag | Нежелательная коммуникация | Priority suppression sync |
| 28 | Корзина запускает спам | Нет frequency cap | Репутационный ущерб | Eligibility/frequency policy |
| 29 | Транзакционное сообщение смешано с рекламой | Один шаблон | Consent conflict | Separate message classes |
| 30 | SLA поддержки неверен | Не учтены паузы/календарь | Ложная просрочка | SLA definition |
| 31 | Обращение без заказа | Identity failure | Долгая диагностика | Search/link workflow |
| 32 | Претензия закрыта без решения | KPI pressure | Повторный конфликт | Exit criteria/QA |
| 33 | Возврат без проверки количества | Нет line-level model | Избыточный возврат | Quantity validation |
| 34 | События приходят не по порядку | Async integration | Откат статуса | Version/sequence control |
| 35 | Retry бесконечный | Нет budget | Нагрузка/backlog | Bounded backoff/dead letter |
| 36 | REST limits/timeout | Peak traffic | Lag/potеря | Queue/backoff/verified batch |
| 37 | Свободная двусторонняя правка | Нет field ownership | Расхождение | Master matrix |
| 38 | Массовый импорт без dry-run | Спешка | Порча данных | Sample/report/rollback |
| 39 | Нет reconciliation | HTTP success принят за результат | Тихие пропуски | Totals/sample/delta |
| 40 | Тариф не поддерживает модель | Capability check пропущен | Нереализуемый scope | Portal verification |
| 41 | Marketplace app изменилось | Vendor dependency | Сбой процесса | Assessment/exit plan |
| 42 | On-Premise устарел | Старые модули/PHP | Ошибки/уязвимости | Supported update plan |
| 43 | Backup не восстанавливается | Нет restore test | Rollback невозможен | Restore drill |
| 44 | Роль тестирует администратор | Нет role QA | Доступы ломаются после запуска | Role-based UAT |
| 45 | Service account — администратор | Простая настройка | Большой blast radius | Least scope |
| 46 | Уволенный сотрудник остаётся owner | Нет offboarding | Бесхозные заказы | Reassignment/revoke |
| 47 | KPI без grain | Смешаны order/line/event | Двойной счёт | Metric dictionary |
| 48 | Средний чек включает отмены | Нет eligibility rule | Искажённый KPI | Defined denominator |
| 49 | Repeat rate строится по email без identity policy | Нестабильный ключ | Ложные повторы | Customer ID/resolution |
| 50 | Attribution меняется ретроспективно | Нет policy | Спорный ROMI | Fixed attribution/version |
| 51 | Dashboard без freshness | Старые данные | Ошибочное решение | Last refresh/SLA |
| 52 | Нет owner интеграции | Проект завершён | Инциденты не разбираются | Support RACI |
| 53 | Нет safe degradation | Provider outage | Массовая порча статусов | Stale/unknown state |
| 54 | Нет concurrency test | Тест только последовательный | Oversell/duplicate refund | Parallel scenarios |
| 55 | Нет hypercare | Система передана сразу | Инциденты накапливаются | Support window |

## 21. ADR

### ADR-ECOM-001: Модель заказа

- **Контекст:** заказ создаётся на сайте и исполняется несколькими системами.
- **Решение:** CMS/OMS/ERP назначается master заказа; Smart Process и/или сделка — контролируемая CRM-проекция по согласованному lifecycle.
- **Альтернативы:** только сделка; Bitrix24 как единственный OMS.
- **Последствия:** external ID, state mapping и reconciliation обязательны.
- **Статус:** Proposed; master выбирается на Discovery.

### ADR-ECOM-002: Хранение товаров

- **Контекст:** товар имеет SKU, варианты, атрибуты, цены и остатки.
- **Решение:** PIM/ERP/CMS — master; CRM-каталог содержит необходимую коммерческую проекцию.
- **Альтернативы:** CRM как master; поля заказа без product relation.
- **Последствия:** mapping, snapshot order line и freshness.
- **Статус:** Proposed.

### ADR-ECOM-003: Модель возврата

- **Контекст:** запрос, приём товара и возврат денег — разные факты.
- **Решение:** отдельный Return lifecycle со строками, количеством и связью с refund events.
- **Альтернативы:** стадия заказа; комментарий обращения.
- **Последствия:** нужны quantity limits, RACI и reconciliation.
- **Статус:** Proposed.

### ADR-ECOM-004: Интеграция сайта

- **Контекст:** сайт создаёт высокочастотные события корзины и заказа.
- **Решение:** в CRM передаются только согласованные business events через versioned contract и idempotency.
- **Альтернативы:** отправлять каждое web-событие; периодический полный CSV.
- **Последствия:** queue, retry, monitoring и privacy filters.
- **Статус:** Proposed.

### ADR-ECOM-005: Архитектура оплат

- **Контекст:** платёжный статус критичен и приходит асинхронно.
- **Решение:** провайдер/finance — master; Bitrix24 получает минимальный сверяемый статус и external IDs.
- **Альтернативы:** ручной флаг; хранение полного provider payload.
- **Последствия:** authenticity, idempotency и reconciliation.
- **Статус:** Proposed.

### ADR-ECOM-006: Архитектура доставки

- **Контекст:** один заказ может иметь несколько отправлений и provider statuses.
- **Решение:** shipment — отдельная связанная сущность/проекция с нормализованным mapping.
- **Альтернативы:** одно поле статуса в заказе.
- **Последствия:** 1:N модель, unknown state и delay monitoring.
- **Статус:** Proposed.

### ADR-ECOM-007: Аналитическая архитектура

- **Контекст:** данные находятся в CMS, CRM, ERP/WMS, оплатах и доставке.
- **Решение:** datasets строятся по контрактам с metric dictionary и reconciliation.
- **Альтернативы:** только CRM-отчёты; ручные таблицы.
- **Последствия:** data quality, stable identity и freshness monitoring.
- **Статус:** Proposed.

### ADR-ECOM-008: Лиды или Order-First

- **Контекст:** оформленный заказ уже является квалифицированным бизнес-событием.
- **Решение:** заказ создаётся напрямую в order/deal flow; лид используется только для отдельного неквалифицированного обращения, если это подтверждено процессом.
- **Альтернативы:** лид на каждый заказ; Deal-First для всех событий.
- **Последствия:** routing и аналитика каналов настраиваются согласно выбору.
- **Статус:** Open до Discovery.

## 22. Типовые ошибки внедрения

| № | Ошибка | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Корзина считается заказом | События не разделены | Ложные продажи | Checkout gate |
| 2 | Каждый заказ проходит через лид | Шаблон без анализа | Лишний lifecycle | Order-first where justified |
| 3 | Один клиент — одна сделка навсегда | Клиент и заказ смешаны | Нельзя считать покупки | Order/deal per transaction |
| 4 | Новый заказ создаёт новый контакт | Нет matching | Дубли клиента | Identity policy |
| 5 | Email — единственный ключ | Shared/changed email | Ошибочное объединение | Multi-attribute resolution |
| 6 | Телефон не нормализован | Каналы дают разные форматы | Дубли | Normalization |
| 7 | Анонимного посетителя сразу создают в CRM | Analytics смешана с operations | Мусор и privacy risk | Business event threshold |
| 8 | Order ID генерируют независимо в двух системах | Нет master | Коллизии | Master-generated external ID |
| 9 | Повтор webhook создаёт заказ | Нет idempotency | Двойное исполнение | Event/order key |
| 10 | Название товара — ключ | Нет SKU/ID | Неверный mapping | Product/SKU ID |
| 11 | Варианты товара объединены | Модель без variant | Ошибка комплектации | Variant ID |
| 12 | Категория хранится свободным текстом | Нет справочника | Нельзя анализировать | Controlled hierarchy |
| 13 | Текущая цена переписывает заказ | Нет snapshot | Неверная история | Order-line snapshot |
| 14 | Цена без валюты | Неполная модель | Ошибочная сумма | Amount + currency |
| 15 | Скидка без источника/правила | Ручная правка | Потеря маржи | Discount metadata/approval |
| 16 | Налоговый контекст игнорируется | Упрощение | Расхождение totals | Finance contract |
| 17 | Остаток CRM объявлен master | Нет WMS boundary | Oversell | WMS/ERP master |
| 18 | Остаток без timestamp | Нет freshness | Продажа недоступного | Freshness field/SLA |
| 19 | Остаток без склада | Одна цифра | Неверная доступность | Location context |
| 20 | Резерв и остаток смешаны | Нет semantics | Двойная продажа | Separate states |
| 21 | Изменение заказа перезаписывает строки | Нет version/event model | Потеря аудита | Versioned change |
| 22 | Отмена разрешена из любого статуса | Нет transition matrix | Потеря товара/денег | Guard conditions |
| 23 | Отмена не связана с резервом | Системы независимы | Товар заблокирован | Compensating action/reconcile |
| 24 | Отмена не связана с оплатой | Нет orchestration | Деньги не возвращены | Refund workflow |
| 25 | Статус вводится менеджером вручную | Нет master sync | Ложный state | Source-controlled field |
| 26 | Все payment attempts хранятся одним boolean | Простая модель | Частичные/повторные оплаты потеряны | 1:N events |
| 27 | Успех оплаты принимается без проверки | Доверие callback | Подмена | Provider authentication |
| 28 | Повтор callback дублирует оплату | Нет event ID | Финансовая ошибка | Idempotency |
| 29 | Ошибка оплаты логирует payload | Debug целиком | Утечка | Masking/minimal logs |
| 30 | Карточные данные сохраняются в CRM | Неверный scope | Critical security risk | Token/reference only |
| 31 | Частичная оплата считается полной | Один статус | Преждевременная доставка | Amount reconciliation |
| 32 | Refund request считается refund fact | Lifecycle смешан | Ложная отчётность | Separate states |
| 33 | Refund превышает доступную сумму | Нет aggregate check | Потеря денег | Refundable balance |
| 34 | Повтор refund event дублирует результат | Нет idempotency | Финансовое расхождение | Refund ID |
| 35 | Один заказ имеет одно поле доставки | Не учтены части | Потеря shipment history | 1:N shipments |
| 36 | Carrier status копируется без mapping | Разные словари | Хаотичный lifecycle | Versioned mapping |
| 37 | Неизвестный статус = доставлено | Оптимистичный fallback | Ложное закрытие | Unknown/exception state |
| 38 | Плановая дата считается фактом | Нет source semantics | Неверный SLA | Plan/fact separation |
| 39 | Задержка не создаёт exception | Нет monitoring | Клиент жалуется первым | Threshold/alert |
| 40 | Адрес доставки доступен всем | Широкие права | Privacy breach | Need-to-know |
| 41 | Возврат хранится комментарием | Нет entity | Нет SLA/аналитики | Return lifecycle |
| 42 | Возврат не связан со строкой заказа | Только order-level | Неверное количество | Line relation |
| 43 | Возвращённое количество не контролируется | Нет aggregate | Избыточный возврат | Quantity validation |
| 44 | Причина возврата — только текст | Нет classifier | Аналитика бесполезна | Controlled reason + note |
| 45 | Претензия и вопрос одинаковы | Нет classification | Неверный priority | Separate type/lifecycle |
| 46 | Обращение закрывается отправкой ответа | Activity = result | Проблема не решена | Exit criteria |
| 47 | SLA не учитывает календарь | Простая разница дат | Ложная просрочка | Calendar policy |
| 48 | Пауза SLA не контролируется | Нет reason/owner | Искусственно хороший KPI | Allowed pauses/audit |
| 49 | Эскалация уведомляет всех | Нет matrix | Alert fatigue | Tiered escalation |
| 50 | Ответ поддержки раскрывает заказ | Шаблон без minimization | Утечка | Secure channel/minimal content |
| 51 | Корзина запускает рекламу без consent | Growth pressure | Жалобы | Consent gate |
| 52 | Отписка не блокирует кампанию | Списки не синхронизированы | Нежелательная рассылка | Priority suppression |
| 53 | Транзакционные и маркетинговые сообщения смешаны | Один pipeline | Ошибка consent | Message classification |
| 54 | Нет frequency cap | Много триггеров | Спам | Contact policy |
| 55 | Сегмент не имеет owner/freshness | Создали один раз | Ошибочная аудитория | Governance |
| 56 | Робот циклически меняет стадии | Взаимные triggers | Нагрузка/спам | Loop guard |
| 57 | Робот повторно создаёт задачу | Нет marker | Дубли работы | Idempotent automation |
| 58 | Ошибка automation скрыта | Нет error route | Заказ завис | Alert/escalation |
| 59 | Webhook URL хранится в README | Секрет считают ссылкой | Компрометация | Env/secret store |
| 60 | Service account — администратор | Быстрая интеграция | Большой blast radius | Least scope |
| 61 | Нет timeout | Happy path | Зависшие workers | Explicit timeout |
| 62 | Retry бесконечный | Нет budget | Нагрузка/backlog | Bounded backoff |
| 63 | Retry неидемпотентен | Нет event key | Дубли | Idempotency |
| 64 | События применяются не по порядку | Async поток | Откат статуса | Sequence/version |
| 65 | Неизвестное поле silently игнорируется | Нет schema validation | Потеря данных | Contract validation |
| 66 | Batch используется без проверки методов | Желание ускорить | Частичный сбой | Verify semantics/limits |
| 67 | Полный импорт без dry-run | Спешка | Массовая порча | Sample first |
| 68 | Название клиента — ключ импорта | Нет ID | Ошибочные merges | Stable key |
| 69 | После импорта нет totals | Успешный HTTP принят за результат | Пропуски | Reconciliation report |
| 70 | Нет rollback map | Не планировали отмену | Нельзя восстановить | Source-target log |
| 71 | Права проверяет только администратор | Нет role QA | Production access defects | Role-based tests |
| 72 | Маркетолог видит адреса/переписку | Роль слишком широка | Privacy incident | Field/entity restriction |
| 73 | Склад видит полную клиентскую историю | Удобство | Избыточный доступ | Minimal fulfilment view |
| 74 | Уволенный сотрудник остаётся owner | Нет offboarding | Бесхозные записи | Reassign/revoke |
| 75 | Средний чек включает отмены/возвраты без правила | Нет metric dictionary | Неверный KPI | Eligibility formula |
| 76 | Конверсия смешивает разные cohorts | Нет grain | Несравнимые числа | Cohort definition |
| 77 | Повторная покупка определяется только email | Identity weak | Ложный repeat rate | Customer resolution |
| 78 | Dashboard не показывает freshness | Данные запаздывают | Ошибочное решение | Last refresh/SLA |
| 79 | Backup не проверен восстановлением | Формальная проверка | Rollback не работает | Restore drill |
| 80 | Запуск без hypercare/support RACI | Проект закрыт сразу | Инциденты без owner | Support window |

## 23. Implementation Roadmap

| Этап | Результат | Минимальные артефакты | Критерий готовности |
|---:|---|---|---|
| 1. Discovery | Подтверждены AS IS, объёмы и система-владельцы | Inventory, journeys, integrations, capability check | Owners и open questions назначены |
| 2. Архитектура | Согласован TO BE | Data model, master matrix, lifecycle, ADR, risks | Владельцы подписали решения |
| 3. CRM настройка | Работает клиентский и сервисный контур | Fields, stages, roles, dedup, tests | Smoke/rights tests успешны |
| 4. Каталог | Настроена контролируемая product projection | Mapping, IDs, prices/stock freshness rules | Sample reconciliation успешна |
| 5. Заказы | Работает order/payment/delivery/return lifecycle | State mappings, automation, exception routes | Сквозные synthetic cases пройдены |
| 6. Интеграции | Работают безопасные потоки | Contracts, retries, monitoring, reconciliation | Failure/recovery tests успешны |
| 7. Тестирование | Подтверждена бизнес-готовность | Test report, defects, UAT sign-off | Нет открытых critical/high defects |
| 8. Запуск | Выполнен управляемый cutover | Runbook, backup, rollback, communications | Smoke и reconcile успешны |
| 9. Оптимизация | Стабилизированы SLA и KPI | Baseline, backlog, lessons learned | Support model принят owners |

### Официальные источники

Используются только источники из `knowledge/SOURCES.md`; дата последней проверки реестра — 2026-08-05.

| Источник | Применение |
|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | Пользовательские возможности CRM и Cloud |
| [Документация REST API](https://apidocs.bitrix24.ru/) | Интеграционная архитектура и ограничения |
| [REST API reference](https://apidocs.bitrix24.ru/api-reference/) | Проверка контрактов методов и полей |
| [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/index.html) | Проверка вариантов интеграционного доступа |
| [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) | Проектирование заказа, возврата, обращения и претензии |
| [Роботы и триггеры](https://helpdesk.bitrix24.ru/open/24633726/) | Проверка automation |
| [Как выбрать тип бизнес-процесса](https://helpdesk.bitrix24.ru/open/23951294/) | Выбор approval/orchestration механизма |
| [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) | Проверка вариантов телефонии |
| [BI Конструктор: наборы данных](https://helpdesk.bitrix24.ru/open/24355590/) | Проверка datasets и аналитики |
| [Маркетплейс Bitrix24](https://www.bitrix24.ru/apps/) | Проверка приложений и коннекторов |
| [Документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php) | On-Premise возможности и ограничения |

Если функция, поле, связь, событие, REST-метод, приложение или ограничение не подтверждены официальным источником и целевым порталом, они помечаются **«требует проверки»** и не включаются в обязательства перед заказчиком.
