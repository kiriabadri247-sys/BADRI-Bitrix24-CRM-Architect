# Bitrix24 Permissions Engineering Knowledge Base

Инженерная база знаний для проектирования, тестирования и аудита доступа Bitrix24. Права проектируются до настройки CRM и проверяются на конкретном портале.

> **Статус:** факты основаны только на `knowledge/SOURCES.md`. Названия уровней, наследование, stage-based permissions, guest access, журналы и тарифные возможности считаются **требующими проверки**.

## 1. Архитектура прав Bitrix24

```text
Пользователь → Организационная структура → Роль → Уровень → Действие → Объект
Приложение/интеграция → identity/scope ────────────────────────────────┘
```

| Компонент | Смысл |
|---|---|
| User/department/manager/group | Организационный и проектный контекст |
| Role/level/action | Повторно используемое правило доступа |
| Entity/item owner/responsible/observer | Предмет и контекст элемента |
| Admin/app/integration | Повышенный или машинный субъект с отдельным контролем |

Права — самостоятельная архитектура: data owner, role owner, матрица, исключения, тесты и аудит определяются до запуска.

## 2. Организационная структура

| Элемент | Рекомендация |
|---|---|
| Company/departments/nested units | Отражать управленческую ответственность, не рисовать структуру ради прав |
| Managers/employees | Актуальные назначения и offboarding |
| Functional/admin roles | Отделять функцию от положения в иерархии |
| Matrix/project teams | Дополнительная role/group model без дублирования отделов |
| Contractors/external users | Named, time-bound; возможность внешнего участия проверить |

```text
Org hierarchy + functional roles + temporary project membership → effective access
```

## 3. Модель RBAC

| Роль | Функция | Объекты | Действия | Ограничения | Владелец |
|---|---|---|---|---|---|
| `<ROLE>` | `<FUNCTION>` | `<ENTITIES>` | Read/create/update | No export/delete/admin | `<OWNER>` |

RBAC: права назначаются роли, сотрудники получают роли. Принципы: least privilege, separation of duties, periodic review, documented temporary exceptions; ручные уникальные права — исключение.

## 4. Типовые роли

| Роль | Цель/сущности | Нужно | Недопустимо/тест |
|---|---|---|---|
| Менеджер | Свои продажи/клиенты | Рабочие CRUD без delete/export | Чужая база; test direct URL |
| Старший/РОП | Команда/направление | Read/control team | Global admin |
| Коммерческий директор | Коммерческий контур | Cross-team analytics/control | System settings без причины |
| Бухгалтер | Счета/оплаты/реквизиты | Finance update | Изменение продаж |
| Маркетолог | Sources/campaign analytics | Read/export by policy | PII without purpose |
| Поддержка | Service clients/requests | Assigned work | Sales financial data |
| HR/юрист | Свои процессы/docs | Restricted domain access | Общая CRM без основания |
| Аналитик/аудитор | Reports/read evidence | Read-only | Operational update |
| Интегратор | Contract entities | Minimal API actions | Admin webhook |
| Администратор | Configuration/support | Controlled full access | Shared daily account |

## 5. Уровни доступа

| Архитектурный уровень | Что видит | Риск | Сценарий |
|---|---|---|---|
| Запрещено | Ничего | Блокировка процесса | Чужой домен данных |
| Только свои | Assigned/owned | Потеря общих клиентов | Менеджер |
| Свои+отдел | Team scope | Неверная оргструктура | Руководитель |
| Отдел+подотделы | Иерархический scope, если доступен | Excess visibility | Директор функции |
| Все открытые | Объекты с подтверждённой моделью открытости | Неочевидная видимость | Требует проверки |
| Все | Полный entity scope | Массовая утечка | Ограниченная роль |

Точные уровни и смысл проверяются в [Поддержке24](https://helpdesk.bitrix24.ru/) и портале.

## 6. Действия над сущностями

| Action | Контроль |
|---|---|
| Read/create/update | Раздельные permissions и field/stage context |
| Delete | Запрещено по умолчанию; approval/recovery |
| Export/import | Отдельный data-exfiltration/change risk |
| Stage/responsible change | Business transition and ownership |
| Automation/settings | Elevated configuration access |
| Analytics | Aggregate/detail/PII scopes |

## 7. Права CRM

| Объект | Типовая роль | Риск/зависимость | Тест |
|---|---|---|---|
| Leads/deals | Sales roles | Direction/stage/owner | List, kanban, URL, export |
| Contacts/companies | Shared/private customer model | Duplicates under isolation | Search and relations |
| Quotes/invoices/products | Sales/finance | Prices and financial data | Read/update separation |
| Activities/docs/comments/history | Context roles | Hidden sensitive content | Related object access |
| User fields | Field consumers | PII/automation dependencies | UI+REST+export |

## 8. Направления сделок

```text
Direction A role matrix ≠ Direction B role matrix
Transfer A→B → re-evaluate owner, visibility, automation and audit
```

Отдельные роли оправданы разными отделами/конфиденциальностью/циклами. Продукт сам по себе не всегда требует направления.

## 9. Права по стадиям

| Sensitive action | Архитектурное правило |
|---|---|
| Amount/responsible/update | Ограничить на approval/closed stages, если механизм доступен |
| Delete/export | Усиленный контроль |
| Transition | Allowed state machine and role |
| Closed stages | Read/limited correction with audit |

Конкретная stage permission **требует проверки**; не обещать её в ТЗ без источника.

## 10. Контакты и компании

| Модель | Плюс | Риск |
|---|---|---|
| Shared base | Единый клиент/повторы | Excess visibility |
| Private ownership | Конфиденциальность | Дубли и конфликт интересов |
| Department service | Team continuity | Wrong org mapping |

Несколько ответственных и ownership-механизмы проверяются; единый customer key защищает от дублей.

## 11. Смарт-процессы

| Сценарий | Access model |
|---|---|
| Договоры | Legal/finance/sales by stage |
| Заявки/service | Requester/assignee/manager |
| Оборудование | Custodian/service roles |
| HR | Restricted HR/manager/subject policy |
| Закупки | Requester/approver/procurement/finance |

```text
Smart Process role → category/stage/item/field → related CRM access
```

Relations, REST, export/delete и automation проверяются обеими сторонами.

## 12. Задачи и проекты

| Subject | Access concern |
|---|---|
| Creator/responsible/accomplice/observer | Different work roles; exact permissions verify |
| Workgroup/project | Open/private and membership model |
| Files/comments/results | May expose task content |
| External participant | Only if officially available and approved |

## 13. Диск и файлы

| Object | Control |
|---|---|
| Personal/shared/group folders | Owner and role access |
| View/edit/delete | Separate operations |
| Shared/external links | Availability, expiry and audience verify |
| CRM docs/templates | Source entity + file access |
| Confidential docs | No public/common links |

Наследование папок **требует проверки** на портале.

## 14. Телефония

| Area | Role control |
|---|---|
| Settings/numbers/routing/PBX | Telephony admin only |
| Call card/statistics | Operator/manager scope |
| Listen/export recordings | Separate sensitive permission |

Источник контекста: [официальная телефония](https://helpdesk.bitrix24.ru/open/6450911/).

## 15. Открытые линии и коммуникации

| Area | Control |
|---|---|
| Queue/distribution | Eligible agents and manager |
| Dialog history/messages | Need-to-know and PII |
| CRM card | Entity permissions remain relevant |
| Supervisor access | Approved quality/control role |

## 16. BI и аналитика

| Access | Risk/control |
|---|---|
| Reports/datasets/SQL/external BI | Broad read; least data scope |
| Export/Power BI | External copy and licensing architecture |
| Dashboard/public link | Audience and detail leakage |
| Analyst | Read-only, PII minimization |

Источник datasets: [BI Конструктор](https://helpdesk.bitrix24.ru/open/24355590/).

## 17. Бизнес-процессы и роботы

```text
Trigger/initiator → execution context → read/change/task/message/REST → audit
```

Кто запускает и от чьего имени выполняется действие **требует проверки**. Elevated automation, files, mass actions и REST проходят отдельный risk review.

## 18. REST, Webhooks и приложения

| Mechanism | Permission source | Control |
|---|---|---|
| Incoming webhook | Creator/user+scope context | Minimum, separate owner, revoke on leaver |
| OAuth/app | Granted scope + user/app context | Tenant isolation, rotation/revoke |
| Service identity | Named technical account | Owner, no shared admin |

Источник: [REST API](https://apidocs.bitrix24.ru/) и [API reference](https://apidocs.bitrix24.ru/api-reference/).

## 19. Администраторы

| Type | Rule |
|---|---|
| Full admin | Minimal named set, no sharing |
| Temporary/integrator/developer | Request, purpose, expiry, revoke |
| Emergency | Controlled break-glass, verification and review |
| Change | Approval, backup, test, changelog |

Чек-лист: [ ] request; [ ] owner; [ ] expiry; [ ] individual account; [ ] actions logged where available; [ ] revoke; [ ] verify.

## 20. Подрядчики

Separate named account, minimum rights, explicit expiry, test portal first, controlled production window, organizational NDA without legal guarantees, no shared credentials, revoke and confirmed deletion of local copies.

```text
Contract need → temporary access → supervised work → revoke → verify/delete copies
```

## 21. Временные права

```text
Запрос → обоснование → согласование → назначение → срок → контроль → отзыв → проверка → документирование
```

## 22. Конфликт ролей

| Несовместимые функции | Риск | Разделение |
|---|---|---|
| Продажа / скидка | Self-approval | Manager / approver |
| Создание / оплата | Fraud/error | Sales / finance |
| Изменение / аудит | Audit bias | Operator / auditor |
| Admin / control | Unchecked privilege | Admin / security owner |
| Development / release | Supply-chain | Developer / releaser |
| Data owner / analyst | Metric manipulation | Owner / read analyst |
| Contractor / permanent admin | Persistent access | Temporary scoped role |

## 23. Матрица доступа

| Сущность/действие | Менеджер | Руководитель | Бухгалтер | Аналитик | Администратор |
|---|---|---|---|---|---|
| CRM read/create/update | Свои/да/да | Отдел/да/да | Finance read | Read scope | Support |
| Delete/export | Нет/нет | Limited/approved | Нет/limited | Нет/limited | Approved |
| Settings/automation | Нет | Process-limited | Нет | Нет | Да |
| Analytics | Own | Team | Finance | Approved datasets | Admin/support |

Шаблон не является готовой настройкой и адаптируется под портал.

## 24. Методика проектирования

```text
Inventory → data owners → org → roles → objects/actions/levels
→ exceptions → matrix → configure → test → accept → document → audit
```

## 25. Тестирование прав

| № | Сценарий | Тип |
|---:|---|---|
| 1 | Роль видит разрешённый список/канбан/card | Positive |
| 2 | Создаёт/изменяет разрешённый объект | Positive |
| 3 | Руководитель видит нужный team scope | Positive |
| 4 | Разрешённая automation работает | Positive |
| 5 | Разрешённый report/file доступен | Positive |
| 6 | Direct URL чужого объекта запрещён | Negative |
| 7 | Search не раскрывает объект | Negative |
| 8 | Export/delete запрещены | Negative |
| 9 | REST с лишней операцией запрещён | Negative |
| 10 | Related file/BI detail не обходит права | Negative |
| 11 | Change responsible/stage ограничено | Negative |
| 12 | Non-admin cannot configure automation | Negative |

Права нельзя принимать только под администратором.

## 26. Негативные сценарии

| Проверка | Ожидание |
|---|---|
| Manager sees чужие deals/all export | Denied |
| Accountant changes sales | Denied |
| Analyst edits data | Denied |
| Contractor remains admin/leaver access | Revoked |
| Direct link bypass | Denied |
| App excessive scope | Not granted |
| Webhook after owner leaver | Rotated/revoked per runbook |
| External/common file link | Restricted/disabled |

## 27. Offboarding

- [ ] Block account; transfer CRM/tasks/responsibles.
- [ ] Remove admin/roles/groups/telephony/disk/BI.
- [ ] Revoke webhooks/OAuth/external services.
- [ ] Rotate shared legacy secrets.
- [ ] Verify sessions/access and record completion.

## 28. Периодический аудит

Рекомендация архитектора: critical permissions — чаще, full role review — периодически (например, monthly/quarterly), но частота определяется риском, а не универсальным правилом.

| Scope | Evidence |
|---|---|
| Admins/inactive/leavers/contractors | Current accounts |
| Roles/deviations | Approved matrix |
| Apps/webhooks | Owner, scope, last use |
| Links/exports | Registers and exceptions |

## 29. Cloud vs On-Premise

| Объект | Cloud | On-Premise | Box additions | Риск |
|---|---|---|---|---|
| Users/org/CRM/Smart/tasks/files | Product/tariff | Version/modules | Same plus owned stack | Misconfiguration |
| Apps/REST | Published boundary | Module/version | Custom modules | Excess rights |
| Server/DB/FS/code | No client access | Owner access | OS/DB/files/custom code | Privilege expansion |
| Logs/infrastructure | Available controls | Full-stack responsibility | More evidence/control | Blind spots |

Источник Box: [официальная документация](https://dev.1c-bitrix.ru/docs/portal.php).

## 30. Типовые ошибки

| № | Ошибка | Риск |
|---:|---|---|
| 1 | Права после внедрения | Rework/leak |
| 2 | Все admins | Excess privilege |
| 3 | Общие аккаунты | No attribution |
| 4 | Role per person | No RBAC |
| 5 | Нет role owner | Drift |
| 6 | Нет matrix | Uncontrolled access |
| 7 | Нет negative tests | Hidden bypass |
| 8 | Нет offboarding | Former access |
| 9 | Contractor remains | Persistent access |
| 10 | Webhook owned by leaver | Integration risk |
| 11 | Manager sees all CRM | PII exposure |
| 12 | Mass export allowed | Exfiltration |
| 13 | Excess isolation | Duplicates |
| 14 | Manager cannot see reports | Process block |
| 15 | CRM rights ignore org | Wrong scope |
| 16 | Smart/deal rights mismatch | Relation leak |
| 17 | Automation elevated | Mass impact |
| 18 | Test only admin | False acceptance |
| 19 | Read/update combined | Excess change |
| 20 | Delete granted by default | Data loss |
| 21 | Export treated as read | Exfiltration |
| 22 | Stage transfer not tested | Leakage |
| 23 | Contacts fragmented | Duplicates |
| 24 | Company name as key | False ownership |
| 25 | Shared client no owner | Conflict |
| 26 | Files inherit assumed rights | Exposure |
| 27 | Public links ignored | Exposure |
| 28 | Recordings broad access | Sensitive leak |
| 29 | BI details broader than CRM | PII leak |
| 30 | Analyst can edit | Integrity risk |
| 31 | OAuth scopes excessive | App compromise |
| 32 | Admin webhook | Full API risk |
| 33 | One service account shared | No isolation |
| 34 | No token rotation | Persistent access |
| 35 | No revoke test | False closure |
| 36 | Temporary access no expiry | Permanent privilege |
| 37 | Exception undocumented | Role drift |
| 38 | SoD absent | Fraud/error |
| 39 | Developer releases own code | Supply-chain risk |
| 40 | Auditor changes data | Audit invalidity |
| 41 | HR data in common Smart Process | Sensitive leak |
| 42 | Contractor uses employee account | No attribution |
| 43 | Guest assumed available/safe | Invalid architecture |
| 44 | Open/private project untested | Content leak |
| 45 | Task files ignored | Data leak |
| 46 | Comments reveal secrets | Leak |
| 47 | Direct URL untested | Access bypass risk |
| 48 | Search untested | Metadata leak |
| 49 | Mobile untested | Different exposure |
| 50 | REST untested | API bypass risk |
| 51 | Import rights ignored | Mass change |
| 52 | Settings access broad | Configuration damage |
| 53 | App inventory absent | Unknown access |
| 54 | Owner change not handled | Orphan integration |
| 55 | Org reorg no review | Wrong scope |
| 56 | Role copy without review | Privilege creep |
| 57 | Former group membership | Residual access |
| 58 | Shared secrets unchanged | Former access |
| 59 | No periodic review | Drift |
| 60 | No evidence/changelog | Unverifiable controls |
| 61 | Production data in tests | PII leak |
| 62 | Admin used daily | Higher compromise impact |
| 63 | Emergency access uncontrolled | Hidden privilege |
| 64 | Open objects misunderstood | Data exposure |
| 65 | Isolation guarantee assumed | False security |

## 31. Чек-лист Permissions Architect

- [ ] Org, roles, owners, entities, actions and levels documented.
- [ ] CRM directions/stages/clients and Smart Processes mapped.
- [ ] Tasks/disk/telephony/BI/BP/REST/apps tested.
- [ ] Admins/contractors/temporary rights controlled.
- [ ] Positive/negative tests, offboarding, audit and docs ready.

## 32. Чек-лист приёмки прав

- [ ] Matrix approved; roles created; users assigned.
- [ ] Test accounts prepared.
- [ ] Positive and negative scenarios passed.
- [ ] Direct URLs, search, export, REST, automation and files checked.
- [ ] Documentation delivered and owner assigned.

## 33. Официальные источники

| Источник | Использование | Проверено |
|---|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | Current roles/settings context | 2026-08-05 |
| [REST API](https://apidocs.bitrix24.ru/) | Webhooks/OAuth/apps | 2026-08-05 |
| [API reference](https://apidocs.bitrix24.ru/api-reference/) | Scope/method contracts | 2026-08-05 |
| [Smart Processes](https://helpdesk.bitrix24.ru/open/18913884/) | Smart permissions context | 2026-08-05 |
| [Telephony](https://helpdesk.bitrix24.ru/open/6450911/) | Telephony context | 2026-08-05 |
| [BI datasets](https://helpdesk.bitrix24.ru/open/24355590/) | Analytics context | 2026-08-05 |
| [On-Premise docs](https://dev.1c-bitrix.ru/docs/portal.php) | Box access boundaries | 2026-08-05 |

```text
Official source → portal/tariff verification → matrix → role tests → acceptance
```

Сторонние уровни прав и юридические утверждения не использованы.
