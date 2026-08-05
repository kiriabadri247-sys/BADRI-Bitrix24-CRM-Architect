# EdTech CRM Architecture Case

Синтетический архитектурный кейс образовательной компании, продающей онлайн-обучение через входящие заявки, консультации, продажи курсов, рассрочки, оплаты, обучение и повторные продажи.

> **Статус:** это референсная архитектура, а не готовая настройка. Наличие лидов, Smart Processes, роботов, бизнес-процессов, BI, телефонии, мессенджеров и приложений зависит от Cloud/On-Premise, тарифа или редакции, версии, прав, региона и провайдера. Конкретная реализация требует Discovery целевого портала.

> **Данные и секреты:** компании, люди, суммы, домены, номера, ID и credentials синтетические или обозначены placeholders. Реальные персональные и платёжные данные в Git запрещены.

> **Архитектурная граница:** CRM управляет продажей и клиентским взаимодействием. Финансовый факт подтверждает платёжная или учётная мастер-система. LMS/образовательная система остаётся master обучения, если она используется.

## 1. Business Context

| Аспект | Референсное описание |
|---|---|
| Бизнес | Продажа онлайн-курсов B2C и ограниченный B2B-сегмент |
| Каналы | Сайт, реклама, мессенджеры, входящие звонки |
| Цели CRM | Не терять заявки, управлять продажами, оплатами и повторными предложениями |
| Результат | Проверяемый путь от обращения до оплаты и передачи в обучение |
| Ограничения | ПДн, платёжные данные, сезонность, внешние providers, тариф Bitrix24 |

### Цели

- единый реестр клиента и коммуникаций;
- формальная квалификация заявок;
- прозрачная воронка и контроль SLA;
- связанная модель курса, оплаты и обучения;
- достоверная аналитика источников и менеджеров;
- контролируемые повторные продажи.

### Stakeholders

| Роль | Интерес | Ответственность |
|---|---|---|
| Собственник | Revenue, growth, risks | Sponsor/приёмка |
| РОП | Funnel, SLA, manager performance | Process owner |
| Маркетолог | Sources, CPL/CAC/ROMI при наличии данных | Attribution owner |
| Бухгалтер | Оплаты, задолженность, возвраты | Financial data owner |
| Куратор | Передача и сопровождение обучения | Learning operations |
| IT/Integrator | Contracts, reliability, support | Technical owner |

## 2. AS IS анализ

```text
Ads/Site/Calls/Messages -> spreadsheets/chats -> managers
       -> manual payment check -> manual learning handoff
```

| Проблема | Проявление | Влияние |
|---|---|---|
| Заявки теряются | Нет единой точки регистрации | Потеря продаж |
| Нет истории клиента | Каналы разрознены | Повторные вопросы и дубли |
| Нет контроля менеджеров | Нет SLA/activities | Поздняя реакция |
| Нет аналитики | Источники и статусы не унифицированы | Неверный бюджет |
| Нет контроля оплат | Статусы ведутся вручную | Ошибочная передача в обучение |
| Нет повторных продаж | Клиент не сегментирован | Потеря LTV |
| Ручные процессы | Копирование между системами | Ошибки и задержки |

До проектирования требуется проверить фактические объёмы, роли, поля, источники, процессы, integrations, потери данных и допустимый downtime.

## 3. TO BE архитектура

```text
Site/Ads/Messengers/Calls
          |
          v
Lead qualification -> Contact/Company -> Deal sale
                                         |      |
                                   Course ref  Payment records/status
                                         |      |
                                         +-> Enrollment/Learning handoff
                                                -> Repeat sale

External payment/accounting master <-> Middleware <-> Bitrix24
LMS master <-----------------------------^              |
BI <----------------------------------------------------+
```

| Слой | Решение |
|---|---|
| CRM | Leads for raw demand; deals for opportunities; contacts/companies for clients |
| Product | CRM catalog or controlled reference; Smart Process only for independent lifecycle |
| Payment | Separate payment records/status linked to deal; external financial master |
| Learning | Smart Process enrollment or external LMS master, depending scope |
| Automation | Robots/tasks for local actions; BP for approvals/exceptions |
| Integration | Backend/middleware, REST/webhooks, queues and reconciliation |
| Analytics | CRM reports/BI with formal metric contracts |

**Cloud:** external code runs outside the platform via published interfaces. **On-Premise:** supported modules/extensions may be evaluated; core changes are prohibited.

## 4. CRM Architecture

### Лиды

Лид создаётся для необработанного входящего обращения, когда компания действительно использует квалификацию до коммерческой opportunity.

| Поле | Назначение | Правило |
|---|---|---|
| Product interest | Интерес к направлению/курсу | Controlled reference |
| Source | Канал/источник | Не подменяет UTM model |
| Interest/need | Квалификационный контекст | Минимум свободного текста |
| Responsible | Owner обработки | Назначение по формальному правилу |
| Status/result | Qualification outcome | Причина отказа обязательна по процессу |

Источники: сайт, реклама, мессенджеры и звонки — только через подтверждённые коннекторы или интеграции. Duplicate policy применяется до создания/конвертации.

### Сделки

```text
Новая заявка -> Связались -> Квалификация -> Презентация
 -> Согласование -> Оплата -> Обучение -> Успешно
                                      \-> Неуспешно с причиной
```

| Стадия | Контрольная точка | Действие/automation | Право/ограничение |
|---|---|---|---|
| Новая заявка | Deal создана и связана с клиентом | Назначить owner/SLA task | Менеджер видит свой scope |
| Связались | Контакт состоялся | Activity result | Не переводить автоматически без факта |
| Квалификация | Need/budget/timing определены | Required fields | Причина отказа формальна |
| Презентация | Предложение сделано | Follow-up task | Template/version controlled |
| Согласование | Условия согласованы | Discount approval if needed | Исключения по ролям |
| Оплата | Ожидается/поступает оплата | Payment integration/tasks | Финансовый факт external master |
| Обучение | Продажа подтверждена, handoff начат | Enrollment/task/integration | Не означает завершение обучения |
| Успешно | Acceptance criteria выполнены | Repeat-sale eligibility | Закрывает opportunity |

Если бизнес сразу создаёт подтверждённую opportunity, допустим Deal-First без лидов; решение фиксируется ADR.

## 5. Product Architecture

| Вариант | Когда применять | Плюсы | Ограничения |
|---|---|---|---|
| Товары CRM | Курсы продаются как номенклатура/позиции сделки | Суммы и product rows | Каталог/цены зависят от версии и интеграции |
| Пользовательские поля | Малый стабильный справочник | Быстро и просто | Плохо для сложных тарифов/версий |
| Smart Process | Курс/программа имеет собственный lifecycle и связи | Структура, стадии, права | Избыточно для статичного каталога |

### Решение архитектора

Для каталога продаваемых курсов использовать товары CRM, если подтверждённая модель каталога удовлетворяет ценам, валютам и тарифам. Направление хранить справочником. Smart Process использовать не для каждой товарной позиции, а для enrollment/program lifecycle, если нужен отдельный управляемый объект. Источник номенклатуры и цены назначается master-системой.

## 6. Payment Architecture

```text
Deal 1 -> Payment N
Payment: planned amount, due date, external transaction ID,
         status, paid amount/date, refund state, reconciliation status
```

| Сценарий | Модель | Контроль |
|---|---|---|
| Полная оплата | Один confirmed payment закрывает agreed amount | External verification |
| Рассрочка | Payment schedule N records | Due date, paid/debt, installment ID |
| Частичная оплата | Aggregate confirmed payments | Remaining debt calculated from verified data |
| Задолженность | Due amount > confirmed paid by cutoff | Collection task/escalation |
| Возврат | Separate refund status/transaction | Не стирать original payment history |

Платёжная/учётная система — master финансового статуса. Callback считается сигналом и сверяется; повторная доставка идемпотентна. Хранить Payment в Smart Process можно только при подтверждённых функциях и необходимости lifecycle; иначе использовать внешнюю финансовую систему с контролируемым отображением в CRM.

## 7. Automation Architecture

### Роботы и задачи

| Trigger | Action | Guard | Error handling |
|---|---|---|---|
| New lead/deal | Назначить менеджера | Queue/region/product rule | Fallback owner |
| New assignment | Уведомление | Dedup | Monitoring/noise review |
| SLA threshold | Reminder/task/escalation | Current stage still valid | Overdue report |
| Stage entry | Создать задачу | Operation/stage key | No duplicate on re-entry |
| Verified payment | Controlled transition | Master status confirmed | Manual review on mismatch |

### Бизнес-процессы

- согласование скидки: threshold, approver, substitute, timeout, reject/rework;
- контроль документов: checklist, version, owner, missing-document escalation;
- проблемные оплаты: mismatch, unknown outcome, overdue, manual decision.

Не автоматизировать неописанный процесс и не создавать циклы перемещения стадий.

## 8. Sales Department Architecture

| Роль | Основной доступ | Ограничения |
|---|---|---|
| Менеджер | Свои лиды/сделки/activities | Нет admin/export/financial edit |
| Старший менеджер | Команда/эскалации | Без platform administration |
| РОП | Отдел, pipeline, analytics | Critical settings по change process |
| Маркетолог | Sources/marketing analytics | Ограниченный CRM content |
| Бухгалтер | Payment view/confirmation workflow | Не меняет sales stages без правила |
| Куратор | Enrollment/learning handoff | Минимум финансовых данных |

Права проверяются отдельными test users через UI, direct URL, REST, export и BI. Точная модель уровней зависит от портала.

## 9. Integration Architecture

| Интеграция | Поток | Master/contract | Reliability |
|---|---|---|---|
| Сайт | Form→backend→CRM | Site submission, CRM accepted record | Validate, dedup, source, consent |
| Телефония | Provider/connector→activity→CRM link | Provider call ID | Synthetic calls, route fallback |
| Мессенджеры | Supported connector/app→communication | Channel/app-dependent | Identity, history, rights |
| Платёжная система | Payment event→verify→CRM status | Provider/accounting master | Idempotency/reconciliation |
| BI | CRM/datasets→dashboard | Metric contract | Freshness, access, control totals |

Применимость телефонии сверяется по [официальной инструкции](https://helpdesk.bitrix24.ru/open/6450911/); мессенджеры и платежи зависят от конкретного официального приложения/провайдера.

## 10. REST/Webhook Architecture

```text
Event/webhook -> Receiver -> Queue -> Worker -> REST read/write
                                 \-> retry/DLQ -> reconciliation
```

| Нужда | Механизм | Данные | Ограничение |
|---|---|---|---|
| Получить актуальную карточку | REST read | Минимальный `select` | Права/контракт |
| Создать/update entity | REST write | External/operation ID | Idempotency/unknown outcome |
| Сигнал об изменении | Подтверждённое событие/webhook | Safe entity/event ID | Event availability/order проверить |
| Массовая обработка | Queue + controlled batch | Per-item report | Partial errors |

Точные методы и события проверяются в [REST API reference](https://apidocs.bitrix24.ru/api-reference/). Webhook URL является секретом; подпись, порядок и retry policy не предполагаются.

## 11. Data Model

```text
Клиент
  -> Контакт/Компания
      -> Сделка (продажа курса)
          -> Курс/товар
          -> Payment 1..N
          -> Enrollment/обучение
              -> Повторная сделка
```

| Объект | Identity | Master | Связи |
|---|---|---|---|
| Client | Verified contact/company key | CRM or agreed external master | Deals/enrollments |
| Course | Product/external catalog ID | CRM catalog or ERP/LMS | Deal product rows |
| Payment | External transaction/schedule ID | Payment/accounting system | Deal |
| Enrollment | Enrollment external ID | LMS or Smart Process | Client/course/deal |

Для каждого пользовательского поля фиксируются code, type, multiplicity, required stage, owner, rights, usage и test.

## 12. Analytics

| Отчёт | Метрика/срез | Предусловие |
|---|---|---|
| Лиды | Count, sources, qualification | Стабильные statuses/sources |
| Продажи | Won deals/revenue | Confirmed amount semantics |
| Конверсия | Stage/lead→deal/win | Formal denominator |
| Менеджеры | Response, activities, conversion | Correct ownership/period |
| Источники | Leads/deals/revenue by channel | UTM/source governance |
| Деньги | Planned/confirmed/debt | Financial master reconciliation |
| Возвраты | Count/amount/reason | Separate refund events |
| Повторные продажи | Repeat deals/LTV cohort | Stable customer identity |

BI Конструктор и datasets зависят от тарифа/прав; отправная точка — [официальная документация datasets](https://helpdesk.bitrix24.ru/open/24355590/).

## 13. Permissions

| Роль | CRM | Сделки | Оплаты | Аналитика | Настройки |
|---|---|---|---|---|---|
| Менеджер | Свой scope | Свои | Read limited | Personal/team limited | Нет |
| Старший | Команда | Команда | Read limited | Team | Нет/минимум |
| РОП | Отдел | Отдел | Summary/read | Sales dashboards | Controlled CRM settings |
| Маркетолог | Limited client data | Aggregate/read as approved | Нет | Sources/marketing | Нет |
| Бухгалтер | Client identifiers by need | Read | Confirm/manage workflow | Financial | Нет CRM admin |
| Куратор | Enrollment clients | Read linked | Minimal status | Learning | Нет |

Матрица — рекомендация. Точные уровни, inheritance и stage permissions проверяются на целевом портале по [Поддержке24](https://helpdesk.bitrix24.ru/).

## 14. Security Architecture

### 1. Role Based Access Control

Роли проектируются по функциям: sales manager, admissions, curator, finance, support, head of department и administrator. Пользователь получает только доступ, необходимый для своей операции; административные, экспортные и интеграционные полномочия отделяются от рабочих ролей.

| Роль | Рабочий контур | Минимальный доступ |
|---|---|---|
| Sales / admissions | Лиды, контакты, компании, сделки поступления | Создание и изменение своего/разрешённого scope |
| Curator / support | Обучение и обращения | Связанные клиенты и сервисные объекты без финансовых настроек |
| Finance | Оплаты и документы | Проверка финансовых статусов без управления CRM-конфигурацией |
| Руководитель | Подразделение и аналитика | Контроль разрешённого отдела |
| Administrator | Техническая конфигурация | Отдельная контролируемая роль, не используемая для повседневной работы |

**Решение архитектора:** least privilege подтверждается позитивными и негативными тестами под тестовыми пользователями, а не только просмотром матрицы.

### 2. Department Isolation

Отделы продаж, admissions, обучения, поддержки и финансов разделяются организационной структурой и CRM-ролями. Междепартаментный доступ предоставляется только для согласованной передачи клиента, оплаты, обучения или обращения; исключения имеют владельца и срок пересмотра.

### 3. CRM Permissions

| CRM-объект | Правило доступа |
|---|---|
| Лиды | Только роли квалификации и их руководители, если лиды используются |
| Контакты | По клиентскому scope и подтверждённой служебной необходимости |
| Компании | По B2B-направлению и ownership account |
| Сделки | По воронке, ответственному и подразделению |
| Smart Processes | По роли в lifecycle курса, обучения, оплаты или сервиса |

Создание, просмотр, изменение, удаление, экспорт и изменение настроек проверяются раздельно. Точная гранулярность прав и доступность ограничений зависят от целевого портала.

### 4. Document Security

Договоры, счета, подтверждения оплаты, учебные документы и вложения доступны только участникам соответствующего процесса. Конфиденциальные файлы хранятся в утверждённом контуре; CRM содержит минимально необходимую копию или защищённую ссылку. Просмотр, изменение, выгрузка, версия и срок хранения регулируются владельцем документа.

### 5. Audit

Контролируются изменения ответственного, стадии, суммы, статуса оплаты, прав и критичных полей. Используются доступная история CRM и обезличенные интеграционные/операционные журналы; конкретная полнота аудита проверяется на портале. За регулярный review отвечает назначенная роль, инциденты получают correlation ID и RCA без секретов и персональных данных в общих логах.

### 6. Cloud Limitations

- функции безопасности и детализация прав зависят от тарифа, региона и настроек портала;
- нет доступа к ядру, серверному PHP-коду и базе данных;
- доступные журналы, история, экспортные ограничения и интеграционные scopes проверяются на конкретном портале;
- заказчик отвечает за пользователей, роли, конфигурацию, приложения, интеграции и клиентские устройства в пределах доступных средств.

### 7. On-Premise Considerations

Администратор дополнительно отвечает за OS, web server, PHP, DB, modules, files, cron/agents, network, TLS, обновления, журналы и hardening. Резервное копирование включает данные, файлы и конфигурацию, а восстановление регулярно тестируется. Инфраструктурный, административный и прикладной доступ разделяются; ядро Bitrix24 не изменяется.

---

## 15. Testing Plan

| Сценарий | Positive | Negative/repeat | Expected evidence |
|---|---|---|---|
| Новая заявка | Lead/deal создан и assigned | Duplicate/spam/invalid field | Source, owner, one entity |
| Продажа курса | Deal проходит допустимые stages | Forbidden transition | Fields, activity, history |
| Оплата | Verified payment updates status | Duplicate callback/timeout | One payment effect, reconciliation |
| Рассрочка | Schedule and debt correct | Missed/late installment | Correct due/paid/debt |
| Возврат | Refund linked, original retained | Duplicate refund | Financial consistency |
| Повторная продажа | New deal same client | Existing opportunity | No duplicate customer/deal |
| Ошибка интеграции | Queue/retry/manual review | Permanent error | Alert, no data loss |

### Чек-лист качества

- [ ] synthetic data only;
- [ ] target roles tested, not admin only;
- [ ] positive, negative, duplicate, timeout and recovery covered;
- [ ] financial totals reconciled;
- [ ] BI formulas/control sample checked;
- [ ] UAT signed by Sales, Finance and Learning owners;
- [ ] rollback rehearsal completed.

## 16. Deployment Plan

```text
Audit -> Architecture -> Configure test/pilot -> Migrate sample
 -> Train -> UAT -> Production cutover -> Smoke -> Hypercare
```

| Этап | Результат | Gate |
|---|---|---|
| Подготовка | AS IS, scope, backup/export, owners | Readiness approval |
| Настройка | Fields, stages, rights, automation disabled/tested | Peer review |
| Перенос | Dry-run, mapping, sample, reconciliation | Data acceptance |
| Обучение | Role-based guides and exercises | Attendance/knowledge check |
| Запуск | Window, cutover, smoke, communications | Go/no-go |
| Поддержка | Monitoring, incidents, known issues | Hypercare exit |

Cloud использует опубликованные настройки/REST; On-Premise дополнительно требует server/PHP/DB/modules/backups and update compatibility.

## 17. Monitoring

| Контроль | Метрика | Alert/action |
|---|---|---|
| Заявки | Expected vs created by source | Missing flow investigation |
| Сделки | Stage aging/no activity | Manager/ROP escalation |
| Оплаты | Unknown/mismatch/overdue | Finance reconciliation |
| Интеграции | Errors, lag, backlog, unknown | Runbook/retry/manual review |
| Automation | Missing/duplicate/error/duration | Automation owner |
| Data quality | Missing required/duplicates | Steward queue |

Monitoring связывается с Incident→Troubleshooting→RCA. Пороги согласуются по baseline и бизнес-риску, универсальные значения не задаются.

## 18. Risks

| № | Риск | Причина | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Потеря заявки | Ошибка source/integration | Потеря продажи | Synthetic journey + reconciliation |
| 2 | Дубли клиентов | Нормализация/keys отсутствуют | Разрыв истории | Identity policy |
| 3 | Дубли сделок | Повтор webhook/retry | Искажённая воронка | Idempotency key |
| 4 | Неверный owner | Ошибка распределения | Нарушение SLA | Queue rules/fallback |
| 5 | Лид завис | Нет SLA/action | Потеря контакта | Aging alerts |
| 6 | Giant pipeline | Смешаны продажа и обучение | Сложная аналитика | Separate lifecycle objects |
| 7 | Course catalog drift | Нет master | Неверный продукт/цена | Catalog ownership |
| 8 | Ложный статус оплаты | Callback принят за факт | Ранний доступ к обучению | Verify with financial master |
| 9 | Двойная оплата | Retry без idempotency | Финансовый инцидент | Transaction ID/reconcile |
| 10 | Неверная задолженность | Partial payments не modeled | Ошибочная коммуникация | Payment schedule model |
| 11 | Потерян возврат | Original payment overwritten | Неверная отчётность | Separate refund history |
| 12 | Enrollment создан дважды | Repeat event | Дубли в LMS | External enrollment ID |
| 13 | Продажа без клиента | Broken relation | Нет истории | Referential check |
| 14 | Данные LMS устарели | Нет master/reconciliation | Ошибка обучения | Defined ownership/sync |
| 15 | Робот выполняется повторно | Re-entry | Дубли задач/сообщений | Operation guard |
| 16 | Цикл стадий | Взаимные automation | Storm | Transition graph tests |
| 17 | Скидка без согласования | Нет threshold/right | Потеря margin | Approval workflow |
| 18 | Уведомительный шум | Слишком много robots | Игнорирование alerts | Noise budget/dedup |
| 19 | Избыточные права | Роль спроектирована широко | Утечка ПДн/финансов | Least privilege tests |
| 20 | Маркетолог видит лишние данные | Analytics access смешан с CRM | Privacy risk | Aggregate/limited rights |
| 21 | Уволенный сохраняет доступ | Offboarding gap | Security incident | Revoke checklist |
| 22 | Webhook утёк | URL в code/log | Unauthorized REST | Secret storage/rotation |
| 23 | OAuth scope избыточен | App convenience | Excess access | Scope review |
| 24 | Пропуск события | Event-only design | Drift | Hybrid reconciliation |
| 25 | Очередь растёт | Provider/downstream slow | Задержка процессов | Depth/age alerts |
| 26 | REST limit/timeout | Неверный profile | Exchange failure | Official contract + batching |
| 27 | Телефония недоступна | Provider/route | Потеря звонков | Joint fallback/runbook |
| 28 | Неверная связь звонка | Caller ambiguity | Чужая карточка | Normalize/manual review |
| 29 | BI недостоверен | Плохие поля/formula | Неверные решения | Data quality gate |
| 30 | UTM потеряны | Source mapping | Неверный marketing ROI | UTM governance |
| 31 | Нет rollback | Success-only deployment | Длительный простой | Tested rollback |
| 32 | Реальные данные в tests | Нет synthetic policy | Privacy breach | Synthetic fixtures |
| 33 | Cloud функция недоступна | Тариф не проверен | Срыв проекта | Early portal verification |
| 34 | Box custom code ломает update | Изменено ядро | Технический долг | Supported module/no core changes |
| 35 | Нет owner поддержки | Проект завершён без handover | Долгие incidents | Support RACI/runbooks |

## 19. Architecture Decision Records

### ADR-EDU-001: Использовать лиды или Deal-First

| Поле | Решение |
|---|---|
| Context | Высокий неоднородный входящий поток требует квалификации |
| Decision | Использовать лиды для raw inquiries; пересмотреть при зрелом qualified-only flow |
| Alternative | Deal-First |
| Consequences | Дополнительный lifecycle, но отдельная аналитика qualification |
| Review trigger | Большинство обращений заранее квалифицировано |

### ADR-EDU-002: Product catalog против Smart Process

| Поле | Решение |
|---|---|
| Context | Курсы продаются как позиции, но enrollment имеет lifecycle |
| Decision | Course — CRM product/reference; enrollment — Smart Process или LMS object |
| Alternative | Всё хранить полями сделки |
| Consequences | Чёткое разделение product и learning lifecycle |
| Review trigger | Catalog/Smart Process capabilities не соответствуют требованиям |

### ADR-EDU-003: Где хранить оплату

| Поле | Решение |
|---|---|
| Context | Нужны full/partial/installment/refund statuses |
| Decision | Financial system master; CRM stores linked operational representation |
| Alternative | Одно поле `Paid` в сделке |
| Consequences | Надёжная сверка, но интеграция сложнее |
| Review trigger | Меняется учётная/платёжная система |

### ADR-EDU-004: Где хранить курс и обучение

| Поле | Решение |
|---|---|
| Context | Продажа и фактическое обучение — разные процессы |
| Decision | Course catalog отдельно; enrollment in LMS or confirmed Smart Process |
| Alternative | Стадия «Обучение» как единственный реестр |
| Consequences | Корректный ownership, дополнительные связи |
| Review trigger | Появляется/меняется LMS |

## 20. Типовые ошибки EdTech внедрения

| № | Ошибка | Почему возникает | Последствие | Решение |
|---:|---|---|---|---|
| 1 | Все обращения сразу сделки | Нет qualification design | Мусорная воронка | Lead/entry criteria |
| 2 | Все заявки только лиды | Не определён выход | Вечная база лидов | Formal conversion |
| 3 | Студент хранится как сделка | Сущности смешаны | Дубли истории | Contact/company identity |
| 4 | Курс только text field | Быстро | Нестабильная аналитика | Catalog/reference |
| 5 | Smart Process для каждого справочника | Универсальность | Overengineering | Entity selection |
| 6 | Продажа и обучение в одной гигантской воронке | Хочется один экран | Giant lifecycle | Deal + enrollment |
| 7 | Воронка для менеджера | Права не спроектированы | Дубли automation | Roles/ownership |
| 8 | Воронка для рекламного канала | Аналитика через config | Fragmentation | Source/UTM fields |
| 9 | Нет formal stages | Используются действия | Неверная аналитика | State semantics |
| 10 | Стадии пропускаются свободно | Нет guards | Потеря контроля | Transition matrix |
| 11 | Нет причины отказа | Менеджерам неудобно | Нет learning analytics | Required reason |
| 12 | Все поля обязательны сразу | Максимальный контроль | Мусорные значения | Stage-based requirements |
| 13 | Дубли стандартных полей | Не изучена CRM | Conflicts | Standard-first |
| 14 | Телефон — уникальный ключ | Простота | Wrong merge | Verified identity policy |
| 15 | Название компании — key | B2B shortcut | False duplicates | Stable identifiers |
| 16 | Повторная продажа в старой сделке | Экономия | Сломана история | New deal |
| 17 | Источник меняется менеджером | Нет governance | Неверный attribution | Source ownership |
| 18 | UTM не сохраняются | Mapping gap | Неверный ROMI | UTM contract |
| 19 | Цена редактируется свободно | Нет master | Margin errors | Catalog/approval |
| 20 | Скидка без approval | Быстрые продажи | Потеря margin | Threshold workflow |
| 21 | Одно поле `Оплачено` | Простая модель | Нет partial/refund/debt | Payment records |
| 22 | Callback равен оплате | Доверие событию | False access | Financial verification |
| 23 | Возврат стирает оплату | Нет event model | Неверная история | Separate refund |
| 24 | Рассрочка хранится текстом | Нет schedule | Нет задолженности | Installment model |
| 25 | Payment ID отсутствует | UI-only | Дубли | External transaction ID |
| 26 | LMS и CRM оба master | Двусторонняя запись | Conflicts | Ownership per data |
| 27 | Enrollment без external ID | Нет mapping | Дубли обучения | Cross-system ID |
| 28 | Доступ к курсу до verify | Stage automation | Revenue risk | Verified payment guard |
| 29 | Робот на каждое действие | No architecture | Cycles/noise | Automation matrix |
| 30 | Робот повторяет задачу при re-entry | Нет idempotency | Duplicate tasks | Operation/stage key |
| 31 | БП согласования giant | Всё в одном | Неподдерживаемость | Modular process |
| 32 | Timeout означает approve | Удобство | Uncontrolled discount | Escalation |
| 33 | Нет substitute approver | Отпуск | Blocked sale | Delegation rule |
| 34 | Уведомление заменяет задачу | Быстро | Нет результата | Task-Based |
| 35 | SLA универсально задан | Копия чужой нормы | Нереалистичный KPI | Business-defined SLO |
| 36 | Manager rights слишком широки | Удобство | Data leak | Least privilege |
| 37 | Маркетолог получает полный CRM export | Нет BI design | Privacy risk | Aggregated analytics |
| 38 | Бухгалтер меняет sales stages | Roles mixed | Broken funnel | Separate responsibilities |
| 39 | Куратор видит финансовые details | Нет field scope | Excess access | Minimal view |
| 40 | Тестируют администратором | Convenience | Hidden rights defect | Target users |
| 41 | Сайт пишет секрет из браузера | Wrong integration | Credential leak | Backend adapter |
| 42 | Webhook в code/log | Copy/paste | Compromise | Secret store/rotation |
| 43 | Нет retry | Happy path | Потеря transient event | Bounded retry |
| 44 | Retry любого сбоя | Нет taxonomy | Storm | Error classification |
| 45 | Нет idempotency | One delivery assumption | Duplicates | Operation key |
| 46 | Нет reconciliation | Event-only | Silent drift | Periodic compare |
| 47 | Очередь без monitoring | Backend hidden | Backlog | Depth/age alerts |
| 48 | Телефония без provider fallback | CRM-centric | Lost calls | Joint runbook |
| 49 | Caller ID связывается случайно | Duplicate contacts | Wrong client card | Manual conflict queue |
| 50 | BI строится до data quality | Желание dashboard | False KPI | Data quality gate |
| 51 | Conversion без denominator | Термин не определён | Спор об отчёте | Metric contract |
| 52 | Revenue равен deal amount | Нет payment truth | Overstated sales | Confirmed financial metric |
| 53 | Refunds исключены | Positive-only analytics | Wrong net revenue | Refund metric |
| 54 | Повторные продажи без identity | Нет client key | Wrong LTV | Stable customer ID |
| 55 | Нет test portal/pilot | Сроки | Production failures | Safe environment |
| 56 | Реальные ученики в tests | Convenience | Privacy breach | Synthetic data |
| 57 | Нет rollback | Optimism | Long outage | Tested reverse plan |
| 58 | Нет monitoring | Project ends at launch | Late incidents | Operational dashboards |
| 59 | Нет support handover | Developer leaves | No recovery | Runbook/RACI |
| 60 | Cloud и Box считаются одинаковыми | Generic design | Impossible implementation | Explicit applicability |

## 21. Implementation Roadmap

| Этап | Результат | Критерий готовности |
|---|---|---|
| 1. Аудит | AS IS, data quality, dependencies, baseline | Scope approved |
| 2. Архитектура | TO BE, patterns, ADR, rights, integration map | Architecture sign-off |
| 3. Настройка CRM | Entities, fields, stages, cards, roles | Configuration tests |
| 4. Автоматизация | Robots/BP/tasks with guards | Repeat/error tests |
| 5. Интеграции | Site, payments, telephony, BI contracts | E2E/reconciliation |
| 6. Тестирование | Regression, performance by risk, UAT | Acceptance signed |
| 7. Запуск | Migration/cutover, training, smoke, hypercare | Stable operation |
| 8. Оптимизация | Metrics, findings, backlog, ADR reviews | Improvement cycle active |

Развёртывать постепенно: Foundation→sales pilot→payments→learning handoff→analytics→optimization.

## 22. Коммерческая оценка

| Параметр | Референсная оценка |
|---|---|
| Сложность | 8/10 при payments, telephony, LMS и BI; уточняется Discovery |
| Сроки | Оцениваются после объёмов, тарифа, integrations и data quality; несколько этапов |
| Команда | Solution Architect, BA, Bitrix24 specialist, Integration Developer, QA, Project/Implementation Lead; BI/DevOps по scope |
| Главные риски | Data quality, financial truth, integration contracts, rights, adoption |

### Что входит

- Discovery и AS IS;
- TO BE и ADR;
- CRM/rights/automation design;
- integration specifications;
- test/deployment/monitoring plans;
- role-based training and hypercare scope.

### Что не входит без отдельного scope

- лицензии Bitrix24 и внешних приложений;
- услуги телефонии, платежного провайдера и LMS;
- неподтверждённая миграция полной истории;
- юридическая экспертиза ПДн, платежей и записей;
- кастомная разработка On-Premise-модулей;
- очистка production-дублей и data remediation;
- постоянная техническая поддержка после hypercare.

### Открытые вопросы для оценки

- Cloud или On-Premise, тариф/редакция и регион;
- объём лидов, сделок, пользователей и файлов;
- текущие источники, LMS, учётная и платёжная системы;
- модель продукта, цен, рассрочек и возвратов;
- матрица ролей и требования к ПДн;
- допустимый downtime, migration depth и support SLA.

## Официальные источники

Использованы только источники, зарегистрированные в `knowledge/SOURCES.md`:

| Источник | Применение |
|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | CRM, права, automation и Cloud-функции |
| [Документация REST API](https://apidocs.bitrix24.ru/) | REST, webhooks, OAuth и интеграционные границы |
| [REST API reference](https://apidocs.bitrix24.ru/api-reference/) | Контракты методов и полей |
| [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) | Product/enrollment options |
| [Телефония](https://helpdesk.bitrix24.ru/open/6450911/) | Варианты подключения и provider boundary |
| [Наборы данных BI](https://helpdesk.bitrix24.ru/open/24355590/) | Analytics datasets |
| [Документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php) | On-Premise applicability |

Если источник не подтверждает функцию, метод, событие, импорт, лимит или приложение, положение считается **требующим проверки** и не включается в обязательства перед заказчиком.

# Related Documents

- [Case Library](../README.md)
- [Architecture Patterns](../PATTERNS.md)
- [Discovery Template](../DISCOVERY_TEMPLATE.md)
- [Architecture Template](../ARCHITECTURE_TEMPLATE.md)
