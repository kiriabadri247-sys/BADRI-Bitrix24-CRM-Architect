# Bitrix24 Migration Engineering Knowledge Base

Инженерная методология безопасной миграции решений Bitrix24 для Senior Bitrix24 Solution Architect, Migration Lead и Enterprise CRM Consultant.

> **Статус сведений:** продуктовые факты основаны исключительно на официальных источниках из `knowledge/SOURCES.md`. Переносимость объектов, истории, настроек, автоматизаций, файлов и прав зависит от направления миграции, Cloud/On-Premise, тарифа или редакции, версии, модулей, прав и доступных официальных интерфейсов. Неподтверждённая возможность помечается **«требует проверки»**.

> **Безопасность:** клиентские выгрузки, резервные копии, персональные данные, webhook URL и секреты остаются в разрешённых локальных или корпоративных системах и не попадают в Git. Массовые операции требуют dry-run, тестовой выборки, идемпотентности, отчёта по каждой записи и плана восстановления.

> **Рекомендация архитектора:** миграция считается завершённой не после импорта, а после сверки данных, прохождения критических процессов, бизнес-приёмки и стабильного post-migration периода.

## 1. Основы миграции Bitrix24

Миграция — управляемое преобразование архитектуры, данных и эксплуатационной ответственности из исходного состояния в целевое.

```text
Source AS IS -> Scope -> Target design -> Mapping -> Rehearsal
 -> Freeze/delta -> Cutover -> Reconciliation -> UAT -> Hypercare
                                               \-> Rollback/compensation
```

| Принцип | Практика | Риск нарушения |
|---|---|---|
| One master | Владелец каждого типа данных назначен | Конфликт двусторонней записи |
| Traceability | Source ID → target ID → result | Невозможно сверить и откатить |
| Repeatability | Повтор не создаёт дубль | Повреждение данных |
| Least transformation | Преобразования формально описаны | Необъяснимые расхождения |
| Evidence | Баланс, выборка, business checks | Ложная приёмка |

Нельзя использовать название компании как единственный устойчивый ключ и нельзя импортировать поверх рабочей базы без согласования.

## 2. Типы миграций

| Направление | Архитектурный фокус | Основное ограничение |
|---|---|---|
| Cloud → Cloud | Доступные export/import/REST, mapping пользователей и ID | Нет доступа к БД и внутренностям платформы |
| On-Premise → On-Premise | Версии, модули, БД, файлы, инфраструктура | Совместимость и целостность компонентов |
| On-Premise → Cloud | Преобразование Box-возможностей в опубликованные Cloud-механизмы | Кастомный серверный код не переносится автоматически |
| Cloud → On-Premise | Данные и настройки через подтверждённые средства | Полный автоматический перенос не предполагается |
| Между отделами/порталами | Ownership, роли, связи, интеграции | Изменение идентификаторов и контекста |

Применимость каждого направления и официального инструмента **требует проверки** до оценки сроков. Термин «копия портала» не использовать без подтверждённого механизма и scope.

## 3. Стратегия миграции

```text
Analyze -> Prepare -> Design -> Build/transform -> Test migration
 -> Rehearse -> Cutover -> Verify -> Stabilize -> Improve
```

| Этап | Результат | Gate |
|---|---|---|
| Анализ | AS IS, объёмы, качество, зависимости | Scope принят |
| Подготовка | Owners, среды, доступы, backup | Readiness подтверждена |
| Проектирование | Target model, mapping, keys, cutover | Design review пройден |
| Перенос | Управляемые batches и журнал | Баланс каждой пачки |
| Проверка | Technical, data, functional, UAT | Критерии выполнены |
| Запуск | Freeze/delta/cutover | Go/no-go решение |
| Поддержка | Hypercare, monitoring, RCA | Стабильность подтверждена |

Стратегия выбирает big bang, phased, parallel или pilot rollout по зависимостям и допустимому периоду двойной работы. Параллельная запись без conflict resolution запрещена.

## 4. Discovery перед миграцией

### Чек-лист охвата

- [ ] пользователи, статусы, отделы, руководители и внешние участники;
- [ ] организационная структура, группы и рабочие области;
- [ ] CRM-сущности, направления, стадии и семантика;
- [ ] стандартные и пользовательские поля;
- [ ] роботы, триггеры, БП, шаблоны и задачи;
- [ ] Smart Processes, `entityTypeId`, категории, стадии и связи;
- [ ] роли, права CRM, файлов, BI и интеграций;
- [ ] REST, webhooks, OAuth, приложения и 1С;
- [ ] телефония, номера, маршруты, записи и provider contracts;
- [ ] BI datasets, формулы, refresh и публикации;
- [ ] документы, шаблоны, файлы и сроки хранения;
- [ ] объёмы, рост, архив, качество и legal constraints.

### Паспорт источника

| Поле | Содержание |
|---|---|
| Owner | Бизнес-владелец и технический владелец |
| Volume | Count, size, date range, growth |
| Quality | Completeness, validity, uniqueness |
| Identity | Source key и external ID |
| Dependencies | Fields, relations, automation, reports |
| Sensitivity | ПДн, коммерческая тайна, retention |
| Extractability | Подтверждённый интерфейс и ограничения |

## 5. Матрица объектов миграции

«Можно перенести» здесь означает статус проектной проверки, а не универсальную гарантию.

| Объект | Можно перенести | Ограничения | Проверка |
|---|---|---|---|
| Пользователи | Требует проверки | Identity, приглашения, статус, лицензии | Target role test |
| Отделы/структура | Требует проверки | Иерархия и руководители | Org chart reconciliation |
| Лиды | Через подтверждённый механизм | Поля, статусы, история | Count/sample/relations |
| Сделки | Через подтверждённый механизм | Pipeline, stage, currency | Balance/sample |
| Контакты/компании | Через подтверждённый механизм | Дубли, реквизиты, связи | Identity tests |
| Товары | Требует проверки | Каталог, цены, валюты, единицы | Financial sample |
| Активности | Требует проверки | Types, participants, dates | Timeline sample |
| История | Требует проверки отдельно | Не считать доступной по умолчанию | Official mechanism + sample |
| Пользовательские поля | Конфигурация + значения отдельно | Codes/types/enums | Metadata diff |
| Роботы/БП | Требует проверки | IDs, actors, fields, activities | All paths regression |
| Smart Processes | Требует проверки | Type/category/stage/relations | Metadata and item sample |
| Права | Перепроектировать и проверить | Users/departments differ | Permission matrix |
| Файлы | Требует проверки | Size, links, owners, access | Hash/sample/access |
| BI | Перепроектировать/проверить | Grain, datasets, IDs, permissions | Control totals |
| Интеграции | Перенастроить | Domains, IDs, secrets, scopes | Contract/E2E tests |
| Телефония | Переподключить по плану | Provider, numbers, routing | Synthetic calls |

## 6. CRM Migration

| Сущность | Подготовка | Порядок/связи | Проверка |
|---|---|---|---|
| Лиды | Статусы, источники, дубли | До/без конвертированных связей по дизайну | Count/status sample |
| Контакты | Нормализация phone/email | До связанных сделок | Identity/duplicate report |
| Компании | Реквизиты и устойчивый ключ | До связанных сделок | Key/relations sample |
| Сделки | Pipeline, stage, currency, responsible | После клиентов/справочников | Amount/stage/relations |
| Товары | Catalog, units, price/currency | До product rows | Financial reconciliation |
| Поля | Code/type/multiple/enums | Metadata before values | Schema diff |
| Активности | Type/date/participants/link | После owners/entities | Timeline sample |
| История | Отдельное решение | Только подтверждённый механизм | Completeness evidence |

```text
Reference data -> Users -> Companies/Contacts -> Leads/Deals
 -> Product rows/Activities -> Relations -> Automation enablement
```

**Рекомендация архитектора:** новые target IDs сохраняются в migration map; source ID не записывается в случайное бизнес-поле без формальной модели.

## 7. Automation Migration

Автоматизация мигрируется как спецификация поведения, а не как непрозрачный визуальный объект.

| Объект | Инвентаризация | Критическая проверка |
|---|---|---|
| Робот | Entity, stage, condition, action, actor | Первый/повторный вход |
| Триггер | Event, filter, transition | Дубли/порядок событий |
| БП | Start, variables, constants, activities | Все ветви, waits, errors |
| Шаблон | Dependencies and version | IDs/fields/users resolved |
| Задача | Creator, assignee, deadline, relation | Права и повторный запуск |

```text
Automation inventory -> Dependency substitution -> Disabled deployment
 -> Synthetic tests -> Controlled enablement -> Monitor
```

Не включать автоматизации до загрузки зависимостей и не запускать их на исторических объектах без явного решения.

## 8. Rights Migration

Права обычно требуют перепроектирования под целевую организационную структуру.

- [ ] source users сопоставлены с target identity;
- [ ] inactive/terminated users исключены по процедуре;
- [ ] отделы, руководители, группы и роли имеют владельцев;
- [ ] CRM directions/stages и Smart Processes покрыты;
- [ ] create/read/update/delete/export проверены;
- [ ] файлы, задачи, BI, телефония и приложения проверены отдельно;
- [ ] positive и negative tests выполнены целевыми пользователями;
- [ ] временные migration permissions имеют expiry.

```text
Source responsibility -> Target role -> Object scope -> Actions
 -> Positive test + Negative test -> Approval
```

## 9. Integration Migration

| Интеграция | Что меняется | Обязательная проверка |
|---|---|---|
| REST | Domain, IDs, fields, methods, rights | Official contract, E2E, idempotency |
| Webhooks | URL, owner, scope, events | Rotation/revoke, duplicate handling |
| OAuth | App context, redirect/config, users/scopes | Official flow and lifecycle |
| Applications | Availability, vendor, settings, rights | Target compatibility and support |
| 1С | Configuration, module/app, mapping, master | Rehearsal and reconciliation |
| Telephony | Provider, numbers, routes, CRM link | Joint cutover and fallback |
| Sites/forms | Endpoint, fields, consent, source | Synthetic submission |

Точные REST-методы проверяются в [официальном справочнике](https://apidocs.bitrix24.ru/api-reference/), локальные сценарии — в [официальной документации](https://apidocs.bitrix24.ru/local-integrations/). Секреты не копируются в migration mapping или Git.

## 10. Data Migration

```text
Extract -> Profile -> Clean -> Deduplicate -> Map -> Validate
 -> Dry-run -> Load batches -> Reconcile -> Correct/accept
```

### Очистка

Нормализовать телефоны, email, идентификаторы, реквизиты, даты, валюты, единицы, пробелы и справочники. Исправление значения должно иметь правило и журнал, а не ручную догадку.

### Дедубликация

Определить match keys, survivorship, merge authority и исключения. Автоматическое объединение production-дублей — отдельная опасная операция и не является обязательной частью переноса.

### Mapping

| Source | Target | Transform | Required | Default | Owner | Test |
|---|---|---|---|---|---|---|
| `<source_field>` | `<target_field>` | Формальное правило | yes/no | Безопасное значение | Data owner | Positive/negative |

### Валидация и качество

| Dimension | Проверка |
|---|---|
| Completeness | Обязательные значения и source balance |
| Validity | Тип, формат, диапазон, enum |
| Uniqueness | Дубли по согласованным keys |
| Consistency | Связанные поля и справочники |
| Accuracy | Выборка против мастер-источника |
| Timeliness | Дельта и cutoff timestamp |

## 11. Migration Planning

### План проекта

| Workstream | Результат | Gate |
|---|---|---|
| Discovery | Scope/data/dependencies | Sign-off |
| Design | Target/mapping/cutover | Architecture approval |
| Build | Tools/configuration/test data | Code/config review |
| Rehearsal | Timing/errors/reconciliation | Readiness |
| Cutover | Controlled migration | Go/no-go |
| Hypercare | Stability and acceptance | Closure |

### RACI

| Активность | Business Owner | Migration Lead | Data Owner | QA | IT/Operations |
|---|---|---|---|---|---|
| Scope | A | R | C | C | C |
| Mapping | C | R | A/R | C | C |
| Rehearsal | I | A/R | C | R | R |
| Go/no-go | A | R | C | C | C |
| Rollback | A | R | C | C | R |
| Acceptance | A/R | C | C | R | I |

### Календарь и риски

Календарь включает discovery, freeze, тестовые выгрузки, rehearsal, исправления, cutover window, delta, UAT, hypercare и rollback deadline. Для каждой внешней системы фиксируется совместное окно.

## 12. Backup и rollback стратегия

| Контекст | Backup/checkpoint | Rollback |
|---|---|---|
| Cloud | Доступная официальная выгрузка/фиксация AS IS | Точная компенсация и возврат configuration |
| On-Premise | Согласованные DB/files/config + restore proof | Совместимый restore и прикладная сверка |
| Integration | State, queue, mapping version | Previous version, controlled replay |

```text
Trigger -> Stop writers -> Preserve evidence -> Rollback/restore
 -> Reconcile delta -> Smoke -> Business verification -> Communicate
```

- определить RPO/RTO, trigger, authority и rollback deadline;
- отделить обратимые configuration changes от перенесённых данных;
- учесть новые target-записи после cutover;
- не считать export полной резервной копией портала;
- проверить восстановление до migration window.

## 13. Testing

| Уровень | Цель | Критерий |
|---|---|---|
| Smoke | Основные сущности и доступ | Critical path работает |
| Functional | Поля, связи, стадии, права, automation | Requirements met |
| Integration | REST/webhooks/1С/telephony/sites | Contracts and reconciliation pass |
| UAT | Бизнес-процессы целиком | Owner sign-off |

### Чек-лист тестирования

- [ ] test data синтетические;
- [ ] source/target counts сбалансированы;
- [ ] поля и связи сверены на стратифицированной выборке;
- [ ] повторный запуск не создаёт дубли;
- [ ] partial failure и resume проверены;
- [ ] роли имеют positive/negative cases;
- [ ] automation включается контролируемо;
- [ ] BI formulas и totals сверены;
- [ ] rollback rehearsal выполнен.

## 14. Cutover план

```text
T-7 approve -> T-1 readiness/freeze -> T0 final extract/checkpoint
 -> Load reference/users -> CRM data -> relations -> delta
 -> Enable integrations/automation -> Smoke -> UAT -> Observe
```

| Шаг | Owner | Evidence | Stop condition |
|---|---|---|---|
| Freeze | Business/Migration Lead | Timestamp and scope | Несогласованные writers |
| Final extract | Data owner | Source count/hash | Неполный extract |
| Load | Migration team | Batch report | Error/data threshold |
| Reconcile | QA/Data owner | Balance/sample | Mismatch beyond tolerance |
| Switch | Integration owner | E2E result | Duplicate/lost event |
| Accept | Business owner | UAT record | Critical journey fails |

Порядок зависит от dependency graph. Решение go/no-go принимает заранее назначенный владелец.

## 15. Post Migration контроль

- [ ] source and target balances подтверждены;
- [ ] unmapped/skipped/failed records имеют disposition;
- [ ] связи, суммы, даты, валюты и активности сверены;
- [ ] automation errors и duplicate effects контролируются;
- [ ] integration queues, lag и reconciliation стабильны;
- [ ] permissions проверены целевыми ролями;
- [ ] BI freshness и control totals подтверждены;
- [ ] пользователи и support получили инструкции;
- [ ] временные доступы и credentials отозваны;
- [ ] hypercare owner, период и exit criteria назначены.

```text
Technical reconciliation + Functional smoke + Business UAT
 + Monitoring window = Migration acceptance
```

## 16. Cloud ограничения

- нет доступа к ядру, серверному PHP и БД;
- нельзя предполагать полный системный export/import;
- доступность полей, объектов, истории, automation, BI и REST зависит от тарифа и прав;
- официальные ограничения и методы проверяются перед design freeze;
- массовые операции должны учитывать ошибки, timeout, повтор, pagination и batch outcomes;
- инфраструктурный backup/restore Cloud не управляется клиентом;
- для внутренних проблем используется официальная поддержка.

| Область | Допустимый подход | Нельзя предполагать |
|---|---|---|
| Data | Published export/import/REST | Прямой DB transfer |
| Configuration | Реестр + доступные official means | Полный автоматический clone |
| History | Проверка отдельного механизма | Полную переносимость |
| Apps | Target availability and scopes | Идентичность окружений |

## 17. On-Premise особенности

Официальная точка входа: [документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php).

| Компонент | Что проверить | Риск |
|---|---|---|
| База данных | Version, charset/collation при применимости, size, integrity, restore | Несовместимость/потеря |
| Файлы | Scope, permissions, ownership, size, integrity | Потерянные документы |
| Модули | Versions, dependencies, custom modules | Ошибка обновления |
| Сервер | OS/web/PHP/resources/network/time | Нестабильность |
| Product version | Source/target compatibility | Невозможный переход |

```text
Infrastructure compatibility -> Full backup -> Isolated rehearsal
 -> Upgrade/move by supported procedure -> Module tests -> Data/app checks
```

Не изменять ядро и не переносить БД/файлы независимо без подтверждённой согласованной точки.

## 18. Типовые ошибки миграции

| № | Ошибка | Причина | Последствие | Как предотвратить |
|---:|---|---|---|---|
| 1 | Нет Migration Lead | Не назначен owner | Несогласованные решения | Назначить accountable role |
| 2 | Нет business owner | Миграция считается IT-задачей | Нет приёмки | Формализовать ownership |
| 3 | Scope не определён | Discovery пропущен | Бесконечное расширение | Scope/out of scope sign-off |
| 4 | Все объекты считаются переносимыми | Нет проверки средств | Невыполнимый план | Object matrix |
| 5 | Cloud и Box смешаны | Не учтены границы | Ошибка архитектуры | Разделить направления |
| 6 | Тариф/редакция неизвестны | Нет environment passport | Недоступная функция | Проверить портал |
| 7 | Версии Box не сверены | Спешка | Несовместимость | Compatibility review |
| 8 | Нет AS IS | Настройки не инвентаризированы | Потеря поведения | Baseline |
| 9 | Нет target design | Копируется плохой процесс | Технический долг | Проектировать TO BE |
| 10 | Миграция равна импорту CSV | Игнорируются зависимости | Потеря связей | Полная архитектура |
| 11 | Нет data owner | Некому принять правила | Споры о качестве | Назначить owner |
| 12 | Нет мастер-системы | Двустороннее редактирование | Конфликты | Ownership matrix |
| 13 | Название компании — ключ | Нет устойчивого ID | Ложные совпадения | External/stable key |
| 14 | Телефон без нормализации | Разные форматы | Дубли | Normalization rule |
| 15 | Email без нормализации | Пробелы/регистр | Дубли | Formal cleanup |
| 16 | ИНН/реквизиты не валидируются | Грязный источник | Ошибочные компании | Validation |
| 17 | Даты без timezone | Разные среды | Смещение событий | Normalize timezone |
| 18 | Валюты смешаны | Нет currency mapping | Неверные суммы | Financial rules |
| 19 | Enum значения не mapped | Разные справочники | Пустые/неверные поля | Dictionary map |
| 20 | Mandatory fields не учтены | Target отличается | Import failures | Schema diff |
| 21 | Multiple fields упрощены | Неверный transform | Потеря значений | Preserve cardinality |
| 22 | Files считаются обычными строками | Нет file strategy | Потеря документов | File matrix |
| 23 | История обещана без проверки | Маркетинговое допущение | Невыполненное требование | Official verification |
| 24 | Activities перенесены до owners | Неверный порядок | Потеря участников | Dependency order |
| 25 | Relations грузятся до entities | Нет target IDs | Висячие связи | Two-pass load |
| 26 | Source ID теряется | Нет migration map | Нет сверки | Persist secure ID map |
| 27 | Target ID пишется вручную | Нет automation/control | Ошибки mapping | Generated report |
| 28 | Нет dry-run | Сразу production | Массовая ошибка | Preview/report |
| 29 | Нет тестовой выборки | Проверяется только весь объём | Высокий риск | Stratified sample |
| 30 | Нет rehearsal | Сроки предполагаются | Срыв окна | Full rehearsal |
| 31 | Rehearsal на другом mapping | Изменения не заморожены | Ложная готовность | Version artifacts |
| 32 | Нет performance estimate | Объём не измерен | Не хватает окна | Measure throughput |
| 33 | Неподтверждённый REST limit | Догадка | Неверный план | Official source/baseline |
| 34 | Pagination забыта | Тест на малом объёме | Неполный extract | Full traversal test |
| 35 | Batch считается атомарным | Ошибочная модель | Partial loss | Per-command outcomes |
| 36 | Timeout равен failure | Не проверено состояние | Дубли при retry | Reconciliation |
| 37 | Retry всех ошибок | Нет классификации | Шторм/дубли | Bounded policy |
| 38 | Нет idempotency | Повтор не учтён | Дубли | Operation key |
| 39 | Нет checkpoint/resume | Скрипт одноразовый | Полный перезапуск | Durable state |
| 40 | Логи содержат ПДн | Нет masking | Утечка | Safe metadata |
| 41 | Логи содержат webhook | Secret hygiene отсутствует | Компрометация | Redact/rotate |
| 42 | Client export попал в Git | Нарушены правила | Утечка | Local protected storage |
| 43 | Backup не создан | Оптимизм | Нет восстановления | Backup gate |
| 44 | Export принят за полный backup | Непонимание границ | Неполный rollback | Define scope |
| 45 | Backup не восстановлен тестово | Проверено только создание | Не работает rollback | Restore rehearsal |
| 46 | RPO/RTO неизвестны | Нет business input | Спор при сбое | Agree targets |
| 47 | Rollback написан после переноса | Нет планирования | Долгий простой | Design together |
| 48 | Нет rollback trigger | Решение субъективно | Поздний возврат | Measurable threshold |
| 49 | Нет rollback deadline | Дельта растёт | Потеря данных | Define last safe point |
| 50 | Откат удаляет по широкому фильтру | Нет ID map | Чужие данные удалены | Exact IDs only |
| 51 | Новые target records не учтены | Нет delta strategy | Потеря после restore | Reconcile writes |
| 52 | Freeze не определён | Users continue work | Moving target | Formal freeze |
| 53 | Freeze слишком длинный | Плохой план | Business outage | Delta migration |
| 54 | Delta extraction отсутствует | Только full dump | Потеря изменений | Cutoff/watermark |
| 55 | Clock/timezone не синхронизированы | Нет time model | Пропущенная delta | Normalize clocks |
| 56 | Параллельная запись без правил | Оба портала active | Конфликты | One writer/conflict policy |
| 57 | Cutover без go/no-go owner | Нет authority | Хаос | Assign decision maker |
| 58 | Нет stop conditions | Цель — закончить любой ценой | Ущерб растёт | Gates |
| 59 | Окно не согласовано | IT решает само | Пользовательский ущерб | Business approval |
| 60 | Providers не участвуют | Внешние зависимости забыты | Интеграции не включены | Joint runbook |
| 61 | Пользователи не уведомлены | Нет communication plan | Работа не там | Notifications |
| 62 | Права мигрируют 1:1 | Org structure изменилась | Избыточный доступ | Redesign roles |
| 63 | Тестирует администратор | Удобство | Скрытая ошибка прав | Target users |
| 64 | Negative access tests отсутствуют | Проверяют только доступ | Утечка | Permission matrix |
| 65 | Inactive users переносятся | Нет offboarding review | Orphan access | Filter/approval |
| 66 | Temporary rights бессрочны | Нет expiry | Security debt | Time-bound access |
| 67 | Роботы включены во время bulk load | Порядок не продуман | Массовые side effects | Disabled load/controlled enable |
| 68 | БП запускаются на истории | Trigger не изолирован | Дубли задач/сообщений | Guard conditions |
| 69 | IDs в automation не заменены | Hardcode | Неверные actors/fields | Dependency substitution |
| 70 | Не проверен повторный вход стадии | Regression узкая | Двойной робот | Re-entry test |
| 71 | Smart Process type не mapped | Неверный `entityTypeId` | Запись не туда | Metadata map |
| 72 | Category/stage смешаны | Разная модель | Неверный lifecycle | Mapping matrix |
| 73 | Webhook копируется как текст | Нет lifecycle | Secret leak/wrong portal | Recreate/rotate |
| 74 | OAuth context не проверен | Предполагается переносимость | Auth failure | Official flow test |
| 75 | App недоступно на target | Нет inventory | Потеря функции | Marketplace/vendor review |
| 76 | Scopes приложений не reviewed | Копирование прав | Избыточный доступ | Least privilege |
| 77 | 1С считается стандартной одной интеграцией | Не зафиксирована конфигурация | Неверный план | Mechanism passport |
| 78 | External IDs заменены без bridge | Новый target | Дубли/рассинхронизация | Crosswalk |
| 79 | Очередь очищена перед cutover | Нет retention | Потеря событий | Drain/preserve/reconcile |
| 80 | Replay выполняется вслепую | Спешка | Дубли | Dry-run/idempotency |
| 81 | Телефония переключена без fallback | Нет joint plan | Потеря звонков | Provider rollback |
| 82 | Реальные номера в тесте без согласования | Нет synthetic route | Влияние на клиентов | Safe test numbers |
| 83 | Сайты продолжают писать в source | Endpoint не переключён | Разрыв данных | Cutover inventory |
| 84 | BI переносится без grain/formulas | Фокус на визуале | Неверные KPI | Metric contract |
| 85 | BI refresh не учтён | Нет расписания | Ложное расхождение | Wait/verify window |
| 86 | Files переносятся без прав | Только bytes | Утечка/нет доступа | Access tests |
| 87 | File hashes не сверяются | Нет integrity check | Повреждение | Hash/sample |
| 88 | Documents links не проверены | Relations забыты | Broken references | Link validation |
| 89 | Только row count считается сверкой | Простая метрика | Скрытые искажения | Fields/relations/totals |
| 90 | Выборка только первых строк | Удобство | Не видны edge cases | Stratified sample |
| 91 | Failed rows теряются | Нет balance | Неполная миграция | Full disposition |
| 92 | Skipped rows без причины | Нет taxonomy | Нельзя принять | Reason codes |
| 93 | Ошибки исправляют вручную без журнала | Давление срока | Невоспроизводимость | Controlled correction |
| 94 | UAT выполняется после закрытия source | Поздно | Нет безопасного возврата | UAT before final closure |
| 95 | Нет business sign-off | Technical-only acceptance | Спор о результате | Formal acceptance |
| 96 | Нет hypercare | Проект распущен | Ошибки без owner | Support window |
| 97 | Monitoring включён после cutover | Не подготовлен | Позднее обнаружение | Preconfigure alerts |
| 98 | Source выключен до reconciliation | Желание быстрее закрыть | Нельзя сверить | Retention/read-only period |
| 99 | Временные credentials не отозваны | Нет closure checklist | Security risk | Revoke/rotate |
| 100 | Lessons не документируются | Проект закрыт | Повтор ошибок | Post-migration review |
| 101 | On-Premise ядро изменяется для переноса | Short-cut | Update incompatibility | Supported extensions |
| 102 | DB и files переносятся из разных точек | Несогласованный backup | Неконсистентность | Coordinated checkpoint |
| 103 | PHP/модули не проверены | Фокус на данных | Runtime failure | Compatibility test |
| 104 | Capacity target не рассчитана | Нет volume forecast | Деградация | Capacity planning |
| 105 | Архив переносится без business need | «Перенести всё» | Стоимость/риск | Retention decision |

## 19. Практические сценарии

| № | Ситуация | Анализ | Решение | Проверка |
|---:|---|---|---|---|
| 1 | Cloud → Cloud CRM | Objects, fields, pipelines, IDs, limits | Phased API/import with map | Counts, samples, relations |
| 2 | Box → Box upgrade/move | Versions, DB/files/modules/server | Supported staged procedure | Restore, module/app smoke |
| 3 | Box → Cloud | Custom code and unsupported objects | Redesign to Cloud mechanisms | Functional/UAT parity |
| 4 | Cloud → Box | Data/config portability | Confirm official interfaces; rebuild gaps | Data + infrastructure tests |
| 5 | Объединение отделов | Duplicate customers and roles | New ownership/dedup model | Permission and identity tests |
| 6 | Разделение портала | Shared clients/processes | Formal partition and master rules | No cross-scope leakage |
| 7 | Миграция сделок | Pipelines/stages/currencies | Load clients then deals/relations | Amount/stage balance |
| 8 | Миграция Smart Process | Types/categories/stages/relations | Metadata first, items second | `entityTypeId` and lifecycle tests |
| 9 | Миграция automation | Hardcoded IDs/actors/dependencies | Rebuild disabled, then pilot enable | All paths/re-entry |
| 10 | Миграция прав | Different users/departments | Target RBAC redesign | Positive/negative matrix |
| 11 | Миграция 1С | Configuration, master, mapping, queue | Joint rehearsal and cutover | Reconciliation/duplicates |
| 12 | Переключение webhooks | Owners, scopes, double delivery | New credentials + guarded overlap | Revoke old, no duplicate effect |
| 13 | Телефония | Provider/numbers/routes/CRM link | Joint window and fallback | Synthetic inbound/outbound |
| 14 | BI reports | Datasets/grain/formulas/rights | Rebuild verified metric contracts | Control totals/freshness |
| 15 | Файлы и документы | Volume, rights, links, retention | Phased transfer with integrity map | Hash/sample/access/link tests |
| 16 | Большой объём с коротким окном | Throughput and delta | Preload + watermark delta | Full balance/cutoff evidence |

## 20. Чек-листы

### Аудит до миграции

- [ ] направление, Cloud/On-Premise, тариф/редакция и версии подтверждены;
- [ ] scope, owners, volumes, quality и retention зафиксированы;
- [ ] object/dependency matrices готовы;
- [ ] master systems и keys согласованы;
- [ ] переносимость каждого объекта проверена;
- [ ] legal/security constraints подтверждены.

### Подготовка

- [ ] target architecture и mapping approved;
- [ ] инструменты имеют dry-run, timeout, retry и idempotency;
- [ ] test environment и synthetic data готовы;
- [ ] backup/restore и rollback rehearsed;
- [ ] full rehearsal завершён;
- [ ] cutover, communications и monitoring готовы.

### День переноса

- [ ] freeze/cutoff объявлены и подтверждены;
- [ ] checkpoint/backup завершён;
- [ ] source count и final extract сверены;
- [ ] batches журналируются по каждой записи;
- [ ] stop conditions и deadline контролируются;
- [ ] delta и external systems переключены по plan;
- [ ] smoke и initial reconciliation пройдены.

### Проверка после переноса

- [ ] counts, fields, relations, sums и dates сверены;
- [ ] failed/skipped/unknown имеют disposition;
- [ ] rights и critical journeys tested;
- [ ] automation, REST, webhooks, 1С и телефония checked;
- [ ] BI totals/freshness verified;
- [ ] UAT signed и hypercare active.

### Возврат при ошибке

- [ ] rollback trigger достигнут и authority подтвердил;
- [ ] writers/integrations безопасно остановлены;
- [ ] evidence, queues и new target IDs сохранены;
- [ ] обратные шаги/restore выполнены;
- [ ] delta reconciled без потери;
- [ ] smoke/business check пройдены;
- [ ] пользователи уведомлены и incident/RCA opened.

## 21. Матрица рисков

| Риск | Вероятность | Влияние | Меры |
|---|---|---|---|
| Потеря данных | Оценить по проекту | Critical | Backup, map, reconciliation, rollback |
| Дубли | Оценить | High | Stable keys, idempotency, dry-run |
| Потеря связей | Оценить | High | Dependency order, two-pass relations |
| Неверные права | Оценить | Critical | RBAC redesign, negative tests |
| Неверные суммы/валюты | Оценить | Critical/High | Financial mapping/control totals |
| Срыв окна | Оценить | High | Rehearsal, throughput, checkpoints |
| Интеграционный разрыв | Оценить | High | Joint cutover, queues, reconciliation |
| Автоматизация создаёт side effects | Оценить | High | Disabled load, guarded enablement |
| Непереносимая функция | Оценить | High | Early official verification/redesign |
| Security leak | Оценить | Critical | Protected storage, masking, least privilege |
| Box incompatibility | Оценить | Critical | Version/module/staging tests |
| Недостоверный BI | Оценить | High | Metric contracts, freshness, totals |

Вероятность и влияние задаются проектом, а не универсальной шкалой. Каждый риск имеет owner, trigger, mitigation, contingency и residual acceptance.

## 22. Официальные источники

Дата последней проверки хранится в `knowledge/SOURCES.md`; перед миграцией актуальность и применимость перепроверяются.

| Источник | Тип / приоритет | Использование |
|---|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | Официальная пользовательская документация / высокий | Объекты, настройки, импорт/экспорт и ограничения продукта |
| [Документация REST API](https://apidocs.bitrix24.ru/) | Официальная документация разработчика / критический | Архитектура API, webhooks и OAuth |
| [REST API reference](https://apidocs.bitrix24.ru/api-reference/) | Официальный справочник / критический | Методы, поля, параметры и ошибки |
| [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/) | Официальная документация / критический | Webhook и локальные сценарии |
| [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) | Официальная статья / высокий | Smart Processes и целевая модель |
| [Варианты подключения телефонии](https://helpdesk.bitrix24.ru/open/6450911/) | Официальная статья / высокий | Телефонный cutover и provider context |
| [Наборы данных BI](https://helpdesk.bitrix24.ru/open/24355590/) | Официальная статья / высокий | BI datasets и аналитическая сверка |
| [Документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php) | Официальная документация 1С-Битрикс / критический | On-Premise, версии, модули и серверный контекст |

Сторонние блоги и форумы не используются как источник продуктовых возможностей. Если источник не подтверждает перенос объекта, истории, конфигурации, приложения или направления миграции, возможность считается **требующей проверки**, а не обещается заказчику.
