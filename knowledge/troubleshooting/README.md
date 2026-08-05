# Bitrix24 Troubleshooting Engineering Knowledge Base

Инженерная методология диагностики, локализации и устранения проблем Bitrix24 для Senior Bitrix24 Solution Architect, Technical Support Lead, Integration Engineer и Incident Manager.

> **Статус сведений:** продуктовые факты основаны только на официальных источниках из `knowledge/SOURCES.md`. Наличие журналов, диагностических экранов, лимитов, административных функций и инфраструктурного доступа зависит от Cloud/On-Premise, тарифа или редакции, версии, модулей и прав. Неподтверждённый механизм помечается **«требует проверки»**.

> **Безопасность:** диагностика не должна ухудшать инцидент. Массовые исправления, повторная доставка событий, отключение автоматизаций, изменение прав, очистка кеша и серверные операции выполняются только после фиксации AS IS, оценки влияния, согласования, backup/checkpoint, теста и плана отката. В документацию не включаются секреты, webhook URL, персональные данные и клиентские выгрузки.

## 1. Основы Troubleshooting в Bitrix24

Troubleshooting — управляемый цикл от наблюдаемого симптома к подтверждённой причине и проверенному восстановлению. «Перестало работать» — симптом, а не диагноз.

```text
Observe -> Contain -> Reproduce -> Localize -> Hypothesize -> Test
   -> Restore -> Verify -> Monitor -> RCA -> Prevent recurrence
```

| Принцип | Практика | Антипаттерн |
|---|---|---|
| Evidence first | Время, роль, entity ID, correlation ID, expected/actual | Менять настройки до фиксации |
| One variable | Проверять одну гипотезу | Несколько исправлений одновременно |
| Layer isolation | UI, права, данные, automation, API, provider | Сразу обвинять платформу |
| Reversibility | Минимальное обратимое действие | Массовое исправление без dry-run |
| Verification | Technical + business + data checks | Закрыть после исчезновения сообщения |

## 2. Методика поиска причины проблемы

1. Зафиксировать бизнес-влияние и безопасно локализовать ущерб.
2. Установить точное время, затронутые роли, сущности и среды.
3. Сформулировать expected и actual result.
4. Проверить последнее известное рабочее состояние и изменения после него.
5. Воспроизвести на синтетической сущности или тестовом контуре.
6. Разделить путь по слоям и найти первую точку расхождения.
7. Сформулировать проверяемую гипотезу с опровержимым прогнозом.
8. Выполнить минимальную read-only проверку.
9. Применить обратимое исправление после согласования.
10. Провести regression, наблюдение и RCA.

```text
User action
 -> UI/configuration
 -> permissions
 -> entity/data
 -> robot/BP
 -> REST/webhook
 -> middleware/provider
 -> target system
```

**Рекомендация архитектора:** искать первую неверную точку, а не самый заметный последний симптом.

## 3. Incident Management

| Фаза | Цель | Обязательный результат |
|---|---|---|
| Detect | Подтвердить сигнал | Incident ID и timestamp |
| Triage | Оценить ущерб | Severity, scope, owner |
| Contain | Остановить рост ущерба | Согласованная временная мера |
| Diagnose | Найти причину | Evidence-backed hypothesis |
| Recover | Вернуть сервис | Проверенное восстановление |
| Close | Подтвердить стабильность | Business sign-off |
| Learn | Исключить повтор | RCA и preventive actions |

- [ ] назначены Incident Commander и technical owner;
- [ ] создан единый канал и журнал событий;
- [ ] установлен ритм обновлений;
- [ ] изменения во время инцидента регистрируются;
- [ ] определены criteria восстановления и закрытия;
- [ ] доказательства обезличены.

## 4. Сбор информации перед анализом

| Категория | Что запросить | Безопасная форма |
|---|---|---|
| Environment | Cloud/On-Premise, тариф/редакция, версия | Паспорт без секретов |
| Time | Начало, timezone, последнее успешное событие | ISO timestamp |
| Scope | Пользователи, роли, сущности, доля операций | Агрегировано |
| Reproduction | Preconditions, steps, expected/actual | Синтетический пример |
| Identifiers | Entity/event/request ID | Минимально необходимое |
| Changes | Release, настройки, права, приложение, provider | Change ID |
| Evidence | UI error, safe response, log excerpt | Маскирование ПДн и credentials |

### Чек-лист первичной диагностики

- [ ] проблема воспроизводима;
- [ ] точное время и timezone известны;
- [ ] Cloud/On-Premise и версия подтверждены;
- [ ] затронутые и незатронутые роли сравнены;
- [ ] сущность, стадия, pipeline и ID известны;
- [ ] expected result связан с требованием;
- [ ] последние изменения перечислены;
- [ ] проверена область: единичная, групповая или системная;
- [ ] временная мера не создаёт риск данных;
- [ ] логи очищены от секретов и ПДн.

## 5. Классификация проблем

| Класс | Типичный симптом | Первый слой проверки | Владелец |
|---|---|---|---|
| CRM | Неверная стадия/поле/дубль | Entity, configuration, history | CRM architect |
| Права | Объект не виден или виден лишнему | Role matrix, ownership | Security/CRM admin |
| Автоматизация | Действие не запустилось/повторилось | Trigger, condition, actor | Automation owner |
| REST | Ошибка/пустой результат | Auth, method, params, response | Integration engineer |
| Webhooks | Событие потеряно/повторено | Delivery, queue, idempotency | Integration engineer |
| Интеграции | Расхождение систем | Contract, master, mapping | Integration owner |
| Телефония | Звонок/маршрут/запись | Provider, route, rights | Telephony owner |
| BI | Неверная метрика | Grain, formula, filters, refresh | BI owner |
| Производительность | Задержка/timeout | Baseline, scope, queue | Performance owner |
| Файлы | Нет доступа/загрузки | Rights, size/type, storage | Portal admin |
| Задачи | Не тот исполнитель/статус | Roles, project, automation | Process owner |
| Smart Processes | Неверный тип/стадия/связь | `entityTypeId`, category, rights | Solution architect |

Один инцидент может иметь несколько симптомов, но RCA должен различать primary cause, contributing factors и последствия.

## 6. Диагностика CRM

| Проблема | Возможная причина | Проверка | Решение | Риск |
|---|---|---|---|---|
| Сделка не переходит | Обязательное поле, права, automation | Роль, стадия, поля, history | Исправить подтверждённую зависимость | Обход бизнес-контроля |
| Лид не конвертируется | Данные, дубли, права | Синтетический лид, duplicate rules | Устранить конкретную причину | Создание дублей |
| Контакт/компания не найдены | Нормализация или видимость | Поиск целевой ролью, формат данных | Исправить данные/доступ | Ложное объединение |
| Поле пустеет | Writer или mapping | History, robots, REST writers | Устранить конфликт мастеров | Потеря данных |
| Активность не связана | Неверная связь/контекст | Entity relation и actor | Восстановить точную связь | Ошибочная история |

### Сделки, лиды, контакты и компании

- подтвердить назначение сущности и источник создания;
- сравнить проблемный и рабочий объект;
- проверить ответственного, направление, стадию и права;
- проверить ключи дублей и нормализацию телефона/email;
- проверить связи клиент—сделка—активность;
- найти всех writers: UI, robot, BP, REST, import.

### Стадии, поля, активности и история

```text
Current value <- UI
              <- Robot/BP
              <- REST/integration
              <- Import
              <- Migration
```

История изменений используется как доказательство только в пределах реально доступной на портале детализации. Полнота истории и журналов **требует проверки**.

## 7. Диагностика роботов

| Проблема | Возможная причина | Проверка | Решение | Риск |
|---|---|---|---|---|
| Не запустился | Не тот вход на стадию/условие | Timeline, condition, test entity | Исправить trigger/condition | Запуск на старых объектах |
| Запустился дважды | Повторный переход/запись | Event sequence | Guard/idempotency | Двойная задача/сообщение |
| Неверный исполнитель | Context/role mapping | Actor и права | Исправить назначение | Раскрытие данных |
| Цикл стадий | Взаимные действия | Transition graph | Разорвать цикл | Массовый шторм |
| Тихая ошибка | Нет контролируемой ветви | History и dependent action | Добавить observability | Ложный успех |

Диагностировать на отдельной синтетической сущности. Не отключать все роботы рабочей воронки ради проверки.

## 8. Диагностика бизнес-процессов

```text
Start -> Input -> Condition -> Activity -> Wait/Approval -> Result
  |        |          |           |             |
context  missing    wrong path   actor/error   timeout/delegate
```

- определить template/version и конкретный instance;
- проверить способ запуска и входные параметры;
- проверить переменные, константы, условия и типы данных;
- установить последнюю успешно завершённую activity;
- проверить права исполнителя и доступ к документу;
- проверить ожидание, срок, делегирование и повторный запуск;
- сравнить с рабочим экземпляром той же версии.

Точные названия активностей и доступность журналов зависят от конфигурации портала.

## 9. Диагностика Smart Processes

| Проблема | Возможная причина | Проверка | Решение | Риск |
|---|---|---|---|---|
| Item не найден | Неверный `entityTypeId`/ID/права | Metadata и target role | Исправить идентификатор/роль | Работа с другим типом |
| Неверная стадия | Category mismatch | Category-stage mapping | Исправить mapping | Нарушение lifecycle |
| Связь отсутствует | Relation не создана/не видна | Обе стороны связи | Создать точную связь | Дубликат relation |
| Поле недоступно | Field config/rights | Metadata и card | Исправить подтверждённую настройку | Потеря значения |

Официальный контекст: [смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) и [REST API reference](https://apidocs.bitrix24.ru/api-reference/). Не предполагать категории, стадии, связи или operations без проверки типа.

## 10. Диагностика REST API

```text
DNS/TLS -> HTTP -> Authorization -> Scope/permissions -> Method
 -> Parameters -> API response -> Business state -> Side effects
```

| Слой | Проверка | Evidence |
|---|---|---|
| Авторизация | Credential действителен и не раскрыт | Redacted credential ID |
| Права | Пользователь/приложение видит объект | Positive/negative request |
| Метод | Имя существует в reference | Ссылка на contract |
| Параметры | Required/type/format/select/filter | Safe request schema |
| Ответ | HTTP и API payload | Redacted response |
| Ошибка | Permanent или transient | Classification |
| Лимиты | Актуальный официальный контракт | Timestamp/source |

Практический порядок:

1. Повторить read-only запрос минимального scope.
2. Отделить transport error от API error.
3. Убрать необязательные параметры.
4. Проверить тем же actor объект, который точно доступен.
5. Сверить метод и поля с [официальным reference](https://apidocs.bitrix24.ru/api-reference/).
6. После timeout не повторять write вслепую: выполнить reconciliation.

## 11. Диагностика Webhooks

| Проблема | Возможная причина | Проверка | Решение | Риск |
|---|---|---|---|---|
| Входящий вызов отклонён | Отзыв/права/URL | Safe auth check | Ротация по процедуре | Утечка URL |
| Событие не дошло | Subscription/delivery/endpoint | Timeline по обеим сторонам | Исправить подтверждённый слой | Потеря события |
| Событие повторено | Delivery/retry | Event key и queue | Идемпотентный consumer | Дубли |
| Порядок нарушен | Асинхронная доставка | Version/timestamps | Reconciliation | Откат нового значения |
| Callback медленный | Тяжёлая логика | Receiver latency | Validate/enqueue/ack | Timeout storm |

Не предполагать подпись, retry policy или порядок доставки без официального контракта. Полный входящий webhook URL запрещено помещать в тикет или лог.

## 12. Диагностика интеграций

```text
Source record -> Mapping -> Transport/queue -> Adapter -> Bitrix24
      |             |            |             |
   master ID     version      retry/DLQ     response/state
```

### 1С

Сначала определить конфигурацию 1С, используемое приложение/модуль, направление обмена, мастер-систему, mapping и последний успешный цикл. Универсального контракта «интеграции с 1С» нет; механизм **требует проверки** для проекта.

### Внешние системы, CRM-коннекторы и приложения

- проверить release/version обеих сторон;
- сверить external ID и master ownership;
- проверить queue/backlog, retry и dead-letter;
- сопоставить request, response и итоговое состояние;
- проверить scopes приложения и отзыв доступа;
- проверить изменение mapping и справочников;
- выполнить reconciliation, не слепой replay.

## 13. Диагностика телефонии

Официальная отправная точка: [варианты подключения телефонии](https://helpdesk.bitrix24.ru/open/6450911/). Провайдерские возможности диагностируются отдельно от Bitrix24.

| Проблема | Возможная причина | Проверка | Решение | Риск |
|---|---|---|---|---|
| Нет подключения | Provider/SIP/config/network | Граница provider—Bitrix24 | Совместное устранение | Простой линии |
| Неверный маршрут | Рабочее время/очередь/fallback | Route matrix | Исправить одно правило | Потеря звонков |
| Нет записи | Настройка/право/storage/legal | Test call и role | Подтверждённая настройка | Правовой риск |
| Нет CRM-события | Link/search/integration | Synthetic caller, timeline | Исправить mapping | Дубликат клиента |
| Пользователь не видит звонок | Права/роль | Target user test | Исправить least privilege | Раскрытие данных |

Не использовать реальные номера, SIP-логины и домены в документации.

## 14. Диагностика прав доступа

Проверка выполняется отдельным тестовым пользователем целевой роли, не администратором.

- [ ] организация, отдел и руководитель зафиксированы;
- [ ] все назначенные роли перечислены;
- [ ] ownership объекта подтверждён;
- [ ] направление, стадия и Smart Process проверены отдельно;
- [ ] create/read/update/delete/export протестированы;
- [ ] UI, direct URL, REST, BI и файлы проверены;
- [ ] конфликт ролей и наследование проверены на портале;
- [ ] automation actor не обходит модель доступа.

Названия уровней и точная логика наследования **требуют проверки** по [Поддержке24](https://helpdesk.bitrix24.ru/) и на целевом портале.

## 15. Диагностика производительности

```text
Symptom -> Scope -> Baseline -> Component timing -> Queue/resource
 -> Hypothesis -> Safe measurement -> Improvement -> Compare
```

| Сигнал | Возможная причина | Проверка | Риск проверки |
|---|---|---|---|
| Медленная карточка | Поля/UI/automation/data | Сравнение объектов и ролей | Изменение production |
| Долгий REST | Query/select/limit/platform | Timing по слоям | Дополнительная нагрузка |
| Растёт очередь | Downstream/retry storm | Arrival vs processing rate | Replay storm |
| Медленный БП | Wait/activity/backlog | Instance timeline | Массовый перезапуск |
| Медленный BI | Dataset/query/refresh | Grain и контрольный запрос | Тяжёлый запрос |

Не делать вывод по одному измерению или среднему значению; фиксировать sample size, percentile, время и нагрузочный профиль.

## 16. Логи и источники данных

| Источник | Что доказывает | Чего не доказывает |
|---|---|---|
| CRM history/timeline | Доступные события сущности | Полную внутреннюю трассу платформы |
| BP/automation history | Доступные шаги процесса | Причину вне зафиксированных данных |
| REST client logs | Request metadata, timing, response class | Состояние Bitrix24 без read-back |
| Queue metrics | Backlog, retries, failures | Корректность бизнес-результата |
| Provider logs | Границу внешнего сервиса | Внутреннее состояние Bitrix24 |
| Server logs (Box) | Инфраструктурные события | Бизнес-причину без корреляции |

Журналировать безопасные metadata: timestamp, correlation ID, method alias, status class, duration, retry count. Не журналировать токены, полный webhook URL, тела с ПДн и записи разговоров.

## 17. Root Cause Analysis

RCA выполняется после стабилизации, не задерживая восстановление.

```text
Incident -> Evidence -> Causal chain
          -> Root cause + contributing factors
          -> Corrective + preventive actions
          -> Owner/date -> Effectiveness review
```

Root cause должна объяснять наблюдения и быть подтверждена тестом или доказательством. «Ошибка пользователя», «Bitrix24 сломался» и «человеческий фактор» без системной причины не являются достаточным RCA.

## 18. Методика 5 Why

| Шаг | Вопрос | Требование к ответу |
|---|---|---|
| Why 1 | Почему возник симптом? | Наблюдаемый механизм |
| Why 2 | Почему механизм дал сбой? | Подтверждённая зависимость |
| Why 3 | Почему контроль не предотвратил? | Gap процесса/архитектуры |
| Why 4 | Почему gap существовал? | Управленческая/техническая причина |
| Why 5 | Почему система допускает повтор? | Корневая контролируемая причина |

Не требуется искусственно ровно пять уровней. Остановиться нужно на причине, для которой можно назначить эффективное preventive action.

## 19. Incident Timeline

| Время (timezone) | Событие | Источник | Решение/действие | Результат |
|---|---|---|---|---|
| `T0` | Первый подтверждённый симптом | Safe evidence | Incident opened | Scope unknown |
| `T1` | Последнее успешное событие | History/log | Boundary set | Search narrowed |
| `T2` | Containment | Change ID | Growth stopped | Monitor |
| `T3` | Recovery | Test evidence | Service restored | Observe |

Различать время события, время обнаружения и время записи. Все timestamps приводить к одному timezone, сохраняя исходное значение при необходимости.

## 20. Приоритизация проблем

### Матрица Severity/Priority

| Severity | Воздействие | Пример | Ожидаемое управление |
|---|---|---|---|
| Critical | Потеря/раскрытие данных, остановка ключевого процесса | Чужие CRM-данные доступны роли | Немедленная координация и containment |
| High | Критический путь не работает, обхода нет | Массово не создаются сделки | Срочное восстановление |
| Medium | Ограниченный ущерб, обход есть | Один отчёт неверен | Плановое исправление с контролем |
| Low | Косметика/удобство | Текстовая ошибка | Backlog |

Priority определяет порядок с учётом Severity, охвата, времени, договорных обязательств, workaround и риска исправления. Severity и Priority не равны автоматически.

## 21. Быстрые проверки за 5 минут

Цель — подтвердить инцидент, оценить ущерб и исключить очевидную локальную причину без изменений production.

- [ ] записать точное время, timezone и пользователя;
- [ ] проверить масштаб: один объект, роль, отдел или все;
- [ ] сравнить с рабочим пользователем/объектом;
- [ ] проверить последнее известное успешное событие;
- [ ] проверить недавний release/change;
- [ ] зафиксировать безопасный текст ошибки;
- [ ] проверить доступность внешнего provider/endpoint по согласованному health signal;
- [ ] назначить owner и Severity;
- [ ] запретить несогласованные параллельные исправления;
- [ ] сообщить первый статус.

## 22. Диагностика за 30 минут

```text
0–5 min triage -> 5–10 scope/timeline -> 10–20 layer isolation
 -> 20–25 hypothesis test -> 25–30 decision/update
```

- [ ] построить путь данных от источника до результата;
- [ ] определить первую точку расхождения;
- [ ] проверить configuration diff и permissions;
- [ ] сопоставить CRM history, integration timing и provider status;
- [ ] выполнить один безопасный reproduction;
- [ ] классифицировать transient/permanent/data/configuration;
- [ ] предложить containment и rollback с рисками;
- [ ] обновить timeline и следующую контрольную точку.

Если причина не найдена, результат 30 минут — не догадка, а суженный scope, проверенные гипотезы и качественная эскалация.

## 23. Глубокий технический аудит

| Область | Артефакт |
|---|---|
| Architecture | Context/dataflow diagrams |
| Configuration | Fields, stages, roles, automation register |
| Data | Profiling, duplicates, master ownership |
| Integration | Contracts, mapping, idempotency, queues |
| Performance | Baseline, percentiles, capacity, backlog |
| Security | Threats, permissions, secrets, audit |
| Operations | Monitoring, backup, rollback, incident history |

- [ ] выбрать период и репрезентативную выборку;
- [ ] сравнить intended architecture с фактической;
- [ ] проверить recurrent incidents и скрытые workarounds;
- [ ] ранжировать findings по риску;
- [ ] отделить quick wins от архитектурных изменений;
- [ ] подготовить безопасный roadmap и критерии проверки.

## 24. Cloud: возможности и ограничения диагностики

| Можно проверить | Ограничение |
|---|---|
| Доступные UI-настройки, история, права, REST responses | Нет доступа к ядру, серверному PHP и БД |
| Внешние integration logs и metrics | Внутренние журналы платформы доступны только в опубликованном объёме |
| Роботы/БП в доступных интерфейсах | Детализация зависит от тарифа, функции и прав |
| Синтетические проверки и reconciliation | Нагрузочные эксперименты должны быть безопасными |
| Официальная документация и support route | Внутреннюю архитектуру Cloud нельзя предполагать |

```text
Client-controlled: configuration + users + data + apps + integration
Vendor-controlled: managed platform internals
Diagnosis boundary: published UI/API/support evidence
```

## 25. On-Premise: серверная диагностика

On-Premise расширяет ответственность владельца. Любое серверное действие выполняется специалистом с нужными правами, backup и rollback. Официальная точка входа — [документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php).

| Слой | Проверка | Ограничение/риск |
|---|---|---|
| Сервер | CPU, RAM, disk, network, time | Измерять, не менять вслепую |
| PHP | Версия, конфигурация, ошибки, workers | Совместимость с версией продукта |
| База данных | Connections, slow operations, locks, capacity | Не менять схему/индексы без официального анализа |
| Модули | Версии, зависимости, обновления | Проверять на staging |
| Файлы | Права, место, целостность, storage | Не выполнять массовую очистку |
| Кеш | Подтверждённая причина и процедура | Очистка может увеличить нагрузку |
| cron/агенты | Запуск, задержки, конкуренция | Не запускать задачи повторно вслепую |

```text
Request -> web server -> PHP -> application/modules -> DB/files/cache
                                  -> agents/cron -> integrations
```

## 26. Практические сценарии диагностики

### Сценарий A: поле сделки возвращается к старому значению

1. Зафиксировать объект, время и оба значения.
2. Получить хронологию writers: UI, robot, BP, REST, import.
3. Сопоставить correlation/time с внешней системой.
4. Подтвердить мастер-систему.
5. Остановить только ошибочный writer после согласования.
6. Исправить тестовую сущность, затем выполнить точную компенсацию.

### Сценарий B: webhook создаёт дубли

1. Сопоставить delivery key и созданные entity IDs.
2. Проверить timeout и retry по обеим сторонам.
3. Подтвердить отсутствие idempotency key/read-before-create.
4. Ввести идемпотентный guard на тестовом потоке.
5. Не объединять/удалять дубли без отдельного согласования.

### Сценарий C: пользователь не видит сделку

1. Проверить целевым пользователем, не администратором.
2. Зафиксировать роли, отдел, ownership, pipeline и stage.
3. Сравнить доступ через list, direct URL, search и REST.
4. Устранить конкретный конфликт роли, не расширяя доступ всем.

### Сценарий D: BI и CRM показывают разные суммы

1. Согласовать формулу, grain, период, timezone и валюту.
2. Учесть refresh latency, закрытые/отменённые стадии и дубли.
3. Сверить малую контрольную выборку с мастер-источником.
4. Исправить contract метрики или качество данных.

## 27. Типовые ошибки Troubleshooting

| № | Ошибка | Последствие / корректирующее действие |
|---:|---|---|
| 1 | Симптом назван причиной | Сформировать проверяемую гипотезу |
| 2 | Нет expected result | Привязать к требованию |
| 3 | Нет точного времени | Зафиксировать timestamp/timezone |
| 4 | Не определён масштаб | Сравнить затронутые группы |
| 5 | Нет последнего успешного события | Найти temporal boundary |
| 6 | Несколько изменений одновременно | Проверять одну переменную |
| 7 | Настройки меняют до evidence | Зафиксировать AS IS |
| 8 | Диагностика ухудшает инцидент | Начинать с read-only |
| 9 | Production используется для эксперимента | Применить test/pilot |
| 10 | Нет Incident Commander | Назначить координацию |
| 11 | Нет технического владельца | Назначить subsystem owner |
| 12 | Параллельные команды не синхронизированы | Вести единый timeline |
| 13 | Severity определяется эмоциями | Использовать impact matrix |
| 14 | Priority приравнена Severity | Учитывать urgency/workaround |
| 15 | Статус не обновляется | Задать cadence |
| 16 | В тикет помещены ПДн | Обезличить evidence |
| 17 | В лог попал token | Отозвать и исправить logging |
| 18 | Полный webhook URL опубликован | Немедленно ротировать |
| 19 | Скриншот без контекста | Добавить role/time/entity |
| 20 | Лог без correlation ID | Ввести безопасную корреляцию |
| 21 | Доверяют одному источнику | Триангулировать evidence |
| 22 | UI-сообщение считается root cause | Проверить нижние слои |
| 23 | Admin test заменяет role test | Использовать target user |
| 24 | Прямой URL не проверяется | Тестировать object access |
| 25 | Экспорт исключён из проверки прав | Добавить export test |
| 26 | BI-права считаются равными CRM | Проверить отдельно |
| 27 | Конфликт ролей игнорируется | Инвентаризировать все роли |
| 28 | Наследование прав придумано | Проверить портал/официальный источник |
| 29 | Поле исправляют без поиска writers | Построить writer inventory |
| 30 | Стадию меняют без анализа automation | Проверить transition graph |
| 31 | Дубли объединяют в ходе диагностики | Отдельно согласовать data fix |
| 32 | Телефон считают уникальным ID | Нормализовать и разрешать конфликт |
| 33 | Название компании — единственный ключ | Использовать устойчивые IDs |
| 34 | История CRM считается полной | Учитывать доступную детализацию |
| 35 | Активность не проверяется на связь | Проверить relation обеих сторон |
| 36 | Робот отключают глобально | Изолировать тест/условие |
| 37 | Повторный вход на стадию не анализируется | Проверить event sequence |
| 38 | Цикл роботов лечат retry | Разорвать causal loop |
| 39 | Actor automation не проверен | Сверить права/контекст |
| 40 | БП проверен без instance ID | Найти конкретный запуск |
| 41 | Версия шаблона БП неизвестна | Зафиксировать template baseline |
| 42 | Ожидание БП принято за зависание | Проверить state/deadline |
| 43 | БП перезапускают массово | Сначала тест и idempotency |
| 44 | Делегирование не учитывается | Проверить actor/history |
| 45 | Smart Process ищут по неверному типу | Проверить `entityTypeId` |
| 46 | Category и stage смешаны | Сверить mapping |
| 47 | Relation проверена с одной стороны | Проверить обе сущности |
| 48 | REST method придуман по аналогии | Сверить reference |
| 49 | HTTP 200 принят за успех | Проверить API payload/state |
| 50 | HTTP error и API error смешаны | Разделить transport/application |
| 51 | Required params не сверены | Использовать contract |
| 52 | Read-only поле пытаются записать | Проверить metadata/reference |
| 53 | Неверный scope трактуется как сбой API | Проверить authorization |
| 54 | Пустой список принят за отсутствие данных | Проверить права/filter |
| 55 | Пагинация забыта | Обойти все страницы |
| 56 | `select` чрезмерен | Запросить минимум |
| 57 | Лимит назван без актуального источника | Проверить official docs |
| 58 | Timeout сразу повторяет write | Сначала reconciliation |
| 59 | Retry всех ошибок | Классифицировать transient/permanent |
| 60 | Retry без backoff/предела | Ограничить стратегию |
| 61 | Batch считается атомарным | Проверять каждую команду |
| 62 | Partial result потерян | Сохранить per-item outcome |
| 63 | Webhook duplicate считается аномалией | Делать consumer идемпотентным |
| 64 | Порядок событий считается гарантированным | Проверить version/state |
| 65 | Callback выполняет тяжёлую логику | Быстро validate/enqueue |
| 66 | Нет dead-letter обработки | Создать controlled review |
| 67 | Replay выполняется без dry-run | Оценить эффект |
| 68 | Queue backlog игнорируется | Сравнить arrival/processing |
| 69 | External ID отсутствует | Ввести устойчивое сопоставление |
| 70 | Master system не назначена | Установить ownership |
| 71 | Mapping разных версий смешан | Версионировать contract |
| 72 | Ошибка 1С обобщена без конфигурации | Зафиксировать версию/механизм |
| 73 | Provider обвинён без boundary evidence | Сопоставить обе стороны |
| 74 | Marketplace app считается частью ядра | Проверить vendor/app contract |
| 75 | Scope приложения не проверен | Audit permissions |
| 76 | Отозванный доступ не тестируется | Выполнить revoke check |
| 77 | Телефония диагностируется только в CRM | Разделить provider/route/CRM |
| 78 | Реальный номер используется в тесте | Использовать согласованный synthetic route |
| 79 | Caller ID считается точным ключом | Учитывать формат/дубли |
| 80 | Запись разговора считается гарантированной | Проверить настройку/права/условия |
| 81 | Правовой аспект записи игнорируется | Привлечь владельца compliance |
| 82 | BI сравнивается без формулы | Зафиксировать metric contract |
| 83 | Grain dataset неизвестен | Определить единицу строки |
| 84 | Refresh latency игнорируется | Проверить окно обновления |
| 85 | Timezone BI и CRM различаются | Нормализовать время |
| 86 | Валюты суммируются без преобразования | Проверить currency rules |
| 87 | Среднее время скрывает хвост | Использовать percentiles |
| 88 | Одно измерение принято за baseline | Собрать выборку |
| 89 | Нагрузочный тест идёт в production | Получить отдельный контур/согласование |
| 90 | Нет stop condition | Задать безопасный предел |
| 91 | Retry storm принят за platform slowdown | Анализировать причинную связь |
| 92 | Очередь очищается удалением | Сохранить/reconcile данные |
| 93 | Cloud анализируется как собственный сервер | Соблюдать managed boundary |
| 94 | Внутренности Cloud предполагаются | Использовать published evidence/support |
| 95 | On-Premise анализируется только в UI | Проверить инфраструктуру |
| 96 | PHP меняют без compatibility test | Проверить supported version |
| 97 | БД оптимизируют вслепую | Анализировать query/locks официально |
| 98 | Индекс добавляют без оценки обновлений | Проверить поддержку/стенд |
| 99 | Кеш очищают как универсальное лечение | Подтвердить причину/риск |
| 100 | cron запускают повторно вслепую | Проверить state/idempotency |
| 101 | Агенты дублируются | Проверить concurrency |
| 102 | Disk full не включён в диагностику | Проверить capacity |
| 103 | Файловые права меняют массово | Использовать точный scope |
| 104 | Время сервера не проверено | Сверить clock/timezone |
| 105 | Модули обновляют во время инцидента без плана | Отделить recovery от upgrade |
| 106 | Нет backup перед изменением Box | Создать/проверить restore |
| 107 | Backup считается рабочим без restore test | Проверить восстановление |
| 108 | Containment выдают за permanent fix | Планировать RCA/action |
| 109 | Workaround не имеет срока | Назначить owner/date |
| 110 | RCA ищет виновного | Анализировать систему |
| 111 | 5 Why подгоняется под ответ | Использовать evidence |
| 112 | Contributing factor назван root cause | Построить causal chain |
| 113 | Corrective action не проверяется | Назначить effectiveness review |
| 114 | Timeline смешивает event/detection time | Разделить timestamps |
| 115 | Инцидент закрыт без business check | Получить подтверждение |
| 116 | Нет observation window | Контролировать recurrence |
| 117 | Temporary access не отозван | Выполнить post-incident cleanup |
| 118 | Test data оставлены в CRM | Удалить точечно по ID |
| 119 | Документация не обновлена | Зафиксировать known error/runbook |
| 120 | Уроки инцидента не попали в regression | Добавить preventive test |

## 28. Чек-лист передачи проблемы разработчику

- [ ] incident/defect ID и Severity указаны;
- [ ] environment, версия, тариф/редакция и timezone указаны;
- [ ] expected и actual result сформулированы;
- [ ] воспроизводимые шаги и preconditions приложены;
- [ ] synthetic test data или точные обезличенные IDs указаны;
- [ ] первое и последнее известное время события известно;
- [ ] affected/unaffected scope описан;
- [ ] последние изменения перечислены;
- [ ] safe request/response schema и correlation ID приложены;
- [ ] проверенные гипотезы и результаты перечислены;
- [ ] workaround и business impact описаны;
- [ ] секреты, webhook URL и ПДн удалены;
- [ ] критерий исправления и regression scope определены.

## 29. Чек-лист закрытия инцидента

- [ ] сервис восстановлен по техническим критериям;
- [ ] critical business journey подтверждён владельцем;
- [ ] данные и очереди reconciled;
- [ ] negative permissions/security check пройден;
- [ ] временные изменения документированы или удалены;
- [ ] временные права и credentials отозваны/ротированы;
- [ ] monitoring стабилен в observation window;
- [ ] timeline и deployment/change records завершены;
- [ ] RCA назначен или завершён согласно Severity;
- [ ] corrective/preventive actions имеют owner и срок;
- [ ] regression test добавлен;
- [ ] пользователи и support уведомлены;
- [ ] client data не попали в Git/отчёт.

## 30. Шаблон технического отчёта

```markdown
# Technical Incident Report: <INC-ID>

## Executive summary
- Status:
- Severity / Priority:
- Business impact:
- Start / detect / restore / close (timezone):

## Environment
- Cloud or On-Premise:
- Tariff/edition/version:
- Affected components and roles:

## Expected vs actual
- Expected:
- Actual:
- Reproduction:

## Evidence
- Safe IDs and correlation IDs:
- Redacted observations:
- Sources checked:

## Diagnosis
- Timeline:
- Hypotheses tested:
- Confirmed cause or current boundary:

## Actions
- Containment:
- Recovery:
- Verification and reconciliation:

## Risks and limitations
- Residual risk:
- What could not be verified:

## Follow-up
- Action / owner / deadline / verification:
```

## 31. Шаблон RCA документа

```markdown
# Root Cause Analysis: <INC-ID>

## Impact and scope
## Detection and response
## Normal architecture and failure path
## Evidence-backed timeline
## Root cause
## Contributing factors
## 5 Why analysis
## What worked / what failed
## Corrective actions
## Preventive actions
## Owners and deadlines
## Effectiveness criteria and review date
## Residual risks and accepted limitations
```

## 32. Официальные источники

Дата последней проверки фиксируется в `knowledge/SOURCES.md`. Перед диагностикой конкретного портала актуальность источника и применимость функции проверяются повторно.

| Источник | Тип / приоритет | Для диагностики |
|---|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | Официальная пользовательская документация / высокий | Интерфейсы, настройки и продуктовые ограничения |
| [Документация REST API](https://apidocs.bitrix24.ru/) | Официальная документация разработчика / критический | REST, события и интеграционные границы |
| [REST API reference](https://apidocs.bitrix24.ru/api-reference/) | Официальный справочник / критический | Методы, параметры, поля и ошибки |
| [Локальные интеграции](https://apidocs.bitrix24.ru/local-integrations/) | Официальная документация / критический | Webhook и локальные интеграционные сценарии |
| [Смарт-процессы](https://helpdesk.bitrix24.ru/open/18913884/) | Официальная статья / высокий | Smart Processes и их настройки |
| [Варианты подключения телефонии](https://helpdesk.bitrix24.ru/open/6450911/) | Официальная статья / высокий | Границы телефонии и внешних провайдеров |
| [Наборы данных BI](https://helpdesk.bitrix24.ru/open/24355590/) | Официальная статья / высокий | BI datasets и аналитический контекст |
| [Документация коробочного Bitrix24](https://dev.1c-bitrix.ru/docs/portal.php) | Официальная документация 1С-Битрикс / критический | On-Premise, модули и инфраструктура |

Сторонние блоги и форумы не используются как источник продуктовых фактов. Если механизм, журнал, параметр, лимит или поведение не подтверждены официальным источником, они отмечаются **«требует проверки»**, а эскалация выполняется через официальные каналы поддержки с обезличенными доказательствами.
