# Bitrix24 Security Engineering Knowledge Base

Инженерная база знаний для безопасной архитектуры, аудита и сопровождения Bitrix24. Это не юридическое заключение и не гарантия защищённости: применимые требования определяют профильные специалисты с учётом юрисдикции, договора, тарифа, редакции и инфраструктуры.

> **Статус:** факты основаны только на `knowledge/SOURCES.md`. 2FA, SSO, guest access, session/device controls, журналы, backup, IP restrictions и OAuth-детали считаются **требующими проверки на конкретном портале**, если точный контракт не подтверждён источником.

> **Запрет:** реальные логины, пароли, токены, webhook URL, телефоны, записи, персональные данные и клиентские выгрузки в Git не помещаются.

## 1. Модель безопасности Bitrix24

```text
Пользователь → Роль → Права → Сущность → Данные
Устройство ────────┘       Приложение/интеграция ────────┘
```

| Субъект/зона | Ответственность |
|---|---|
| Bitrix24 | Безопасность управляемой платформы в пределах официальной модели Cloud |
| Владелец бизнеса | Классификация данных, risk acceptance, владельцы и бюджет |
| Администратор | Пользователи, роли, настройки, приложения и изменения |
| Интегратор/разработчик | Least privilege, secure code, secrets, tests, документация |
| Внешний провайдер | Свой сервис, канал, SLA и security contract |
| Сотрудник/подрядчик | Индивидуальная учётная запись, устройство и соблюдение политики |

Объекты контроля: пользователи, группы, подразделения, роли, CRM/Smart Process/задачи/диск/телефония/BI, приложения, REST, внешние сервисы и пользовательские устройства.

## 2. Cloud и On-Premise

| Зона | Cloud | On-Premise | Ответственный | Комментарий |
|---|---|---|---|---|
| Инфраструктура | Управляется поставщиком | Сервер/ОС/БД/web/PHP/FS у владельца | Vendor / клиент | Границы договора проверить |
| Пользователи/данные | Клиент | Клиент | Клиент | Shared responsibility |
| Обновления | Vendor-managed | Owner-managed | Vendor / клиент | Box: stage+backup+tests |
| Network/perimeter | Ограниченный контроль клиента | Полный контроль владельца | Клиент | Firewall/remote access |
| Backup/monitoring | Доступные штатные средства | Полная политика владельца | Клиент | Restore test обязателен |
| Custom code | Apps/REST | Supported modules/code | Developer+owner | Не изменять ядро |

```text
Cloud: client IAM/data/apps ↔ managed service
Box: client IAM/data/code ↔ client server/OS/DB/network
```

## 3. Пользователи и учётные записи

| Этап | Контроль |
|---|---|
| Создание/приглашение | Owner, role, department, expiry, approved channel |
| Изменение роли | Заявка, approver, least privilege, audit |
| Увольнение | Block, transfer, revoke, verify |
| Подрядчик/временный | Named account, expiry, narrow scope |
| Guest/external | Только если функция подтверждена и risk accepted |
| Service account | Отдельный owner, purpose, minimal permissions, rotation |

```text
Joiner → approved access → periodic review → mover → leaver
```

Общая учётная запись запрещает attribution, безопасный offboarding и персональную ответственность; пароль неизбежно распространяется.

## 4. Аутентификация

| Механизм | Контроль |
|---|---|
| Login/password | Unique, strong policy, secure recovery, no sharing |
| 2FA | Использовать при подтверждённой доступности; recovery-контроль |
| SSO/external auth | Проверить тариф/редакцию и официальный контракт |
| Sessions/devices | Проверять доступные controls; lost-device runbook |
| Recovery | Identity verification и защита канала восстановления |

```text
Identity proof → authentication → session → authorization → audit
```

Не описывать 2FA, trusted devices или SSO как гарантированные без проверки [Поддержки24](https://helpdesk.bitrix24.ru/).

## 5. Авторизация и права доступа

| Роль | Просмотр | Создание | Изменение | Удаление | Экспорт | Настройки |
|---|---|---|---|---|---|---|
| Менеджер | Свои/назначенные | По процессу | Рабочие поля | Нет | Нет | Нет |
| Руководитель | Отдел | По функции | Контроль | Ограничено | По регламенту | Нет |
| Бухгалтер | Финансовый контур | По функции | Финансовые поля | Нет | Ограничено | Нет |
| Поддержка | Свои обращения | По процессу | Сервисные поля | Нет | Нет | Нет |
| Интегратор | Только контракт | Только контракт | Только контракт | Нет | API-minimum | Нет |
| Администратор | По сопровождению | Да | Да | С согласованием | С основанием | Да |
| Аудитор | Read-only scope | Нет | Нет | Нет | Контролируемо | Нет |

```text
Effective access = role ∩ entity ∩ operation ∩ stage/direction ∩ context
```

Отдельно тестируются CRM, Smart Processes, задачи, диск, телефония, аналитика, приложения и admin settings под реальными тестовыми ролями.

## 6. Административные права

| Риск | Контроль |
|---|---|
| Постоянный full admin | Named admins, минимум, review |
| Временное повышение | Approval, expiry, purpose, post-check |
| Подрядчик | Separate account, supervised window, revoke |
| Изменение | Ticket, peer review, backup, test, changelog |
| Передача пароля | Запрещена; индивидуальная учётная запись |

```text
Request → approve → time-bound elevate → perform/log → revoke → verify
```

Чек-лист: [ ] цель; [ ] срок; [ ] owner; [ ] backup; [ ] журнал; [ ] revoke; [ ] session/secret rotation; [ ] итоговая проверка.

## 7. Безопасность CRM-данных

| Актив | Риск | Контроль |
|---|---|---|
| PII/реквизиты/платежи | Массовый доступ/экспорт | RBAC, minimization, export approval |
| Документы/files/comments | Скрытая передача данных | Rights, link policy, content guidance |
| Calls/messages/recordings | Чувствительное содержание | Separate access, retention, audit |
| History | Раскрытие действий | Restricted read, retention |
| Delete | Потеря и аналитические разрывы | Approval, archive, backup, rollback |

```text
Classify → minimize → authorize → use → archive → delete
```

Реальные данные в тестах запрещены; общая ссылка и комментарий не являются безопасным каналом передачи.

## 8. REST API Security

| Механизм | Доступ | Область | Риски | Когда использовать |
|---|---|---|---|---|
| Incoming webhook | Secret URL | User+scope | URL leak, excessive rights | Narrow one-portal server task |
| OAuth app | Access/refresh lifecycle | Granted scopes | Token/client secret leak | Managed app/multi-tenant |
| Local app | Portal-specific contract | App permissions | Misconfiguration | One portal customization |

```text
Secret store → authenticated REST request → permission check → entity data
```

Минимальные scope, test/prod separation, rotation, revoke и logs без секретов обязательны. Источник: [REST API](https://apidocs.bitrix24.ru/).

## 9. Webhook Security

| Правило | Реализация |
|---|---|
| URL — секрет | Secret manager, runtime-only |
| Отдельная задача | Separate webhook/owner/environment |
| Передача | Не код, README, чат, screenshot или log |
| Утечка | Revoke, replace, inspect actions |
| Завершение | Delete/revoke and verify denial |

```text
Leak suspected → stop consumer → revoke → rotate → investigate → restore
```

Тестовые webhooks не переиспользуются в production. Подробности: `knowledge/webhooks/README.md`.

## 10. OAuth Security

| Элемент | Контроль |
|---|---|
| Authorization flow | Использовать только подтверждённый официальный flow |
| Access/refresh token | Encryption, tenant isolation, expiry/refresh handling |
| Client secret | Server-side secret storage |
| Callback/state | Проверять официальную применимость и anti-forgery contract |
| Revoke | Uninstall/revoke runbook и удаление tokens |

```text
Authorize → callback validation → token store → refresh/rotate → revoke
```

Authorization code, `state` и сроки токенов требуют проверки по [API reference](https://apidocs.bitrix24.ru/api-reference/), не задаются по памяти.

## 11. Секреты и конфигурация

| Место | Правило |
|---|---|
| `.env` | Только локально, исключён из Git |
| Secret storage/CI | RBAC, audit, environment separation |
| Config | Без credential; безопасные placeholders |
| Backup/log/screenshot/docs | Secrets redacted/excluded |
| Tests/export | Synthetic values only |

```text
Secret source → runtime injection → memory → redacted telemetry
Git/chat/README/test/export ─X→ secret
```

## 12. Безопасность интеграций

| Связь | Данные/Auth/Channel | Риск и ответственность |
|---|---|---|
| Bitrix24↔1С | Clients/products/payments; service identity; TLS/VPN as applicable | Master conflicts; both owners |
| ↔Site | Forms/orders; backend auth; HTTPS | PII/spam/browser secret |
| ↔Telephony | Numbers/records; connector credential | Recording access/provider boundary |
| ↔Messenger | Dialogs/identities; app contract | Provider limits and PII |
| ↔Payment | Amount/status; verified provider auth | Replay/fraud/unknown result |
| ↔API/Middleware | Mapped objects; OAuth/key | Central compromise |
| ↔BI | Datasets/export | Broad read/PII publication |

```text
System A → authenticated encrypted channel → validation/mapping → System B
                     └→ safe audit / revoke / test
```

## 13. Безопасность 1С-интеграции

| Контроль | Требование |
|---|---|
| Service identity | Minimum rights, owner, rotation |
| Data | Clients, requisites, products, prices, payments, docs by ownership |
| Network | Approved protected channel and restricted endpoints |
| Journal | Operation IDs without full sensitive payload |
| Contractor | Time-bound named access |
| Environments | Separate test/prod bases and credentials |

```text
1С master data ↔ controlled exchange ↔ Bitrix24 consumer/owner fields
```

Архитектура зависит от конфигурации 1С, версии и приложения; универсального безопасного шаблона нет.

## 14. Безопасность телефонии

| Актив | Контроль |
|---|---|
| Numbers/routes | Owner, approved routing, change control |
| SIP credentials | Secret manager, no Git/chat |
| Calls/records | RBAC, legal review, retention, controlled export |
| PBX/connectors | Provider contract, least access, monitoring |
| Contractors | Time-bound access and revoke |

```text
Caller → provider/PBX → connector → Bitrix24 → CRM/recording rights
```

Источник вариантов подключения: [официальная телефония](https://helpdesk.bitrix24.ru/open/6450911/).

## 15. Безопасность файлов и документов

| Область | Контроль |
|---|---|
| Disk/folders | Role-based access and periodic review |
| Shared/external links | Expiry/password if officially available; otherwise restrict |
| CRM docs/templates | Data minimization and version owner |
| Download/export/backups | Approved storage and encryption |
| Leavers/delete | Revoke links, transfer owner, retention policy |

```text
Create → classify → share minimally → version → archive/delete
```

Конкретные функции ссылок требуют проверки на портале.

## 16. Безопасность бизнес-процессов и роботов

| Риск | Контроль |
|---|---|
| Запуск/исполнитель | Проверить контекст прав и инициатора |
| Change/delete | Approval, constraints, dry-run for bulk |
| Messages/files | Не раскрывать данные получателю без прав |
| REST | Secrets outside template, timeout, idempotency |
| Loops/pauses | Limits, SLA, monitoring |
| Elevated automation | Отдельный risk acceptance и минимальный scope |

```text
Trigger → permission/context check → action → audit/error escalation
```

## 17. Безопасность Smart Processes

| Область | Контроль |
|---|---|
| Roles/stages | Tested role matrix and transitions |
| Elements/relations | Both-side access review |
| User fields | PII classification and field rights |
| Automation/REST | Minimal rights, no loops, operation ID |
| Export/delete | Approval, archive, audit |

```text
Role → Smart Process item → stage/field/relation permissions
```

## 18. Аналитика и BI

| Контроль | Требование |
|---|---|
| Reports/datasets | Least read, row/detail access |
| External BI/Power BI/SQL | Separate architecture and licensing verification |
| PII | Aggregate/anonymize where possible |
| Dashboards/links | Audience and publication controls |
| Analyst | Named role, export and local-copy policy |

```text
Operational data → governed dataset → aggregate → restricted dashboard
```

BI-возможности зависят от тарифа: [официальные datasets](https://helpdesk.bitrix24.ru/open/24355590/).

## 19. Логи и аудит

| Событие | Что фиксировать безопасно |
|---|---|
| Data/process/integration change | Actor/operation/correlation ID, time, result class |
| Rights/app change | Who, approved request, before/after category |
| Export | Если механизм доступен — actor, scope, time |
| Errors | Category, attempt, safe metadata |
| Retention/access | Owner, period, read rights, tamper controls |

```text
Action → structured audit event → restricted storage → alert/review
```

Наличие конкретного системного журнала **требует проверки**; внешний журнал не должен содержать секреты и PII payload.

## 20. Резервное копирование

| Контур | Cloud | On-Premise |
|---|---|---|
| Возможность клиента | Доступные экспорты/копии критичных справочников | DB, files, config, code, scheduled jobs |
| Ответственность | Определить доступный RPO/RTO и защищённое хранение | Полный backup, encryption, offsite и restore test |
| Ограничение | Не считать экспорт полной резервной копией | Backup без restore test не подтверждён |

```text
Backup → integrity check → isolated restore → functional verification
```

## 21. Удаление и жизненный цикл данных

| Этап | Контроль |
|---|---|
| Create/use/change | Purpose, owner, access, history |
| Archive/export | Retention, protected storage, register |
| Delete/restore | Approval, dependencies, recoverability |
| Contract end/leaver | Revoke, transfer, local-copy destruction |
| Test data | Expiry and verified deletion |

```text
Create → use → update → archive → delete → verify destruction
```

## 22. Разделение сред

| Среда | Данные/credential | Правило |
|---|---|---|
| Development | Synthetic/local secrets | No production access |
| Testing | Synthetic scenarios | Separate app/webhook/domain |
| Staging | Production-like synthetic | Controlled acceptance |
| Production | Real governed data | No experiments |

```text
Code promotion: dev → test → stage → prod
Secrets/data: isolated per environment
```

## 23. Безопасная разработка

| Риск | Контроль вокруг Bitrix24 |
|---|---|
| Input/XSS/CSRF/SSRF | Validate, encode, official framework protections, allowlist destinations |
| SQL injection (Box/external) | Parameterized access; no direct unsafe SQL |
| Dependencies | Pin/review/update, license and provenance check |
| Errors/retries | No secret leakage; timeout and bounded retry |
| Code changes | Peer review, tests, scanning and changelog |

```text
Threat model → secure design → review/test → deploy → monitor/patch
```

Не утверждается наличие уязвимости Bitrix24: описаны риски собственного кода и компонентов.

## 24. Управление изменениями

| Шаг | Артефакт |
|---|---|
| Request/approval | Scope, owner, risk |
| Plan/backup/test | Steps, test data, rollback |
| Window/deploy | Communication and executor |
| Verify | Expected/actual and security checks |
| Close | Changelog, docs, residual risk |

```text
Request → assess → approve → backup/test → deploy → verify/rollback → document
```

## 25. Инциденты безопасности

| Фаза | Действие |
|---|---|
| 1–3 | Обнаружить, ограничить, отозвать токены/webhooks |
| 4–6 | Блокировать пользователей, сменить пароли, остановить интеграцию |
| 7–8 | Сохранить журналы, оценить масштаб |
| 9–11 | Восстановить, документировать, предотвратить повторение |

```text
Detect → contain → preserve evidence → eradicate → recover → lessons learned
```

Эскалация к профильной ИБ и юридической функции обязательна по внутреннему регламенту; документ их не заменяет.

## 26. Offboarding сотрудника

| Блок | Проверка |
|---|---|
| Account | Block, sessions/recovery controls as available |
| Work | Transfer tasks, CRM ownership and documents |
| Rights | Groups, admin, apps, files, BI, telephony, mail |
| Secrets | Revoke personal tokens; rotate shared legacy secrets |
| Audit | Check exports, record completion and owner |

```text
HR event → inventory access → transfer → revoke → verify → record
```

## 27. Offboarding подрядчика

| Система | Действие |
|---|---|
| GitHub/Bitrix24 | Remove account/team/admin access |
| Webhook/OAuth/API | Revoke and rotate |
| Server/1С/telephony/BI | Disable named access and sessions |
| Docs/local copies | Transfer, request deletion confirmation |
| Recovery access | Remove backup accounts/keys |

```text
Contract end → access inventory → revoke all → rotate → verify/delete copies
```

## 28. Cloud vs On-Premise

| Контроль | Cloud | On-Premise | Ответственность клиента | Риск |
|---|---|---|---|---|
| Infrastructure/server/DB/network | Vendor | Client | High in Box | Misconfiguration |
| Updates/files/backups | Vendor boundary | Client | Define RPO/RTO | Unpatched/no restore |
| Apps/webhooks/API | Client config | Client config+modules | Least rights | Token/app compromise |
| Logs/monitoring | Available controls | Full stack | Review/operate | Blind spots |
| Developer access/code | Apps/integrations | Server/custom modules | Time-bound/review | Supply-chain/core edits |
| Scaling/config | Service limits | Owner capacity | Plan/test | Availability |

```text
Cloud shared responsibility ≠ no client responsibility
Box ownership = platform + infrastructure + custom code responsibility
```

Источник On-Premise: [официальная документация](https://dev.1c-bitrix.ru/docs/portal.php).

## 29. Модель угроз

| Угроза | Актив/канал | Последствие | Защита | Контроль |
|---|---|---|---|---|
| Account compromise | User/session | Data access | Strong auth/2FA if available | Login review |
| Webhook/OAuth leak | URL/token | API access | Secret store/rotation | Revoke test |
| Excess rights | Role | Broad exposure | Least privilege | Access review |
| Mass export/delete | CRM/BI | Leak/loss | Approval/RBAC/backup | Audit alert |
| Malicious app/integration | App/API | Exfiltration/change | Vendor review/scopes | App inventory |
| Log/test leak | Logs/non-prod | PII/secrets | Redaction/synthetic data | Scanning |
| Former employee/shared password | Account | Persistent access | Offboarding/no sharing | Periodic review |
| Infected device | Endpoint | Session theft | Endpoint policy | Incident process |
| Data tampering/replay | Integration/event | Wrong state/duplicates | Validation/idempotency | Reconciliation |

```text
Threat → asset → trust boundary → preventive control → detective control → response
```

## 30. Типовые ошибки

| № | Ошибка | Риск |
|---:|---|---|
| 1 | Общий admin account | Нет attribution |
| 2 | Нет процесса увольнения | Persistent access |
| 3 | Чрезмерные права | Data exposure |
| 4 | Постоянный доступ подрядчика | Supply-chain risk |
| 5 | Webhook в коде | API compromise |
| 6 | Токен в README | Leak |
| 7 | Токен в screenshot | Leak |
| 8 | Реальные данные в тестах | PII leak |
| 9 | Среды не разделены | Cross-environment impact |
| 10 | Нет отзыва доступа | Persistent credential |
| 11 | Нет backup | Data loss |
| 12 | Restore не тестируется | False assurance |
| 13 | Нет changelog | Untraceable changes |
| 14 | Экспорт без контроля | Exfiltration |
| 15 | Общие ссылки на файлы | Unauthorized access |
| 16 | Нет rotation | Long-lived compromise |
| 17 | Пароли в таблице | Broad leak |
| 18 | Bulk automation без dry-run | Mass damage |
| 19 | Нет rollback | Extended outage |
| 20 | Нет integration owner | No response |
| 21 | Нет incident plan | Slow containment |
| 22 | Admin используется ежедневно | Elevated risk |
| 23 | Права по фамилиям | Fragile access model |
| 24 | Service account без owner | Orphan access |
| 25 | Один credential для всех систем | No isolation |
| 26 | Test webhook в prod | Environment breach |
| 27 | Secret в логах | Leak |
| 28 | Полный payload в логах | PII leak |
| 29 | Нет TLS | Interception |
| 30 | Подпись webhook придумана | False protection |
| 31 | IP allowlist — единственная защита | Fragility |
| 32 | OAuth tokens не шифруются | Tenant compromise |
| 33 | Refresh failures скрыты | Silent outage |
| 34 | App scopes не проверены | Excess access |
| 35 | Marketplace app без аудита | Vendor risk |
| 36 | Подрядчик передаёт пароль | No accountability |
| 37 | Бывший сотрудник владеет webhook | Availability/security risk |
| 38 | CRM export хранится локально бессрочно | Shadow data |
| 39 | Записи звонков доступны всем | Sensitive leak |
| 40 | Нет retention | Excess data |
| 41 | BI dashboard публичен | Broad disclosure |
| 42 | Drill-through раскрывает PII | Authorization bypass in analytics |
| 43 | Автоматизация имеет лишние права | Privilege escalation impact |
| 44 | Цикл роботов | Mass changes |
| 45 | Direct DB write в коробке | Integrity/update risk |
| 46 | Изменение ядра | Unsupported updates |
| 47 | Server unpatched | Infrastructure risk |
| 48 | Нет monitoring | Late detection |
| 49 | Logs доступны всем | Sensitive disclosure |
| 50 | Logs можно незаметно изменить | Audit failure |
| 51 | Нет correlation ID | Investigation failure |
| 52 | Нет backup encryption | Backup leak |
| 53 | Backup рядом с production | Common-mode loss |
| 54 | Нет offsite/isolated copy (Box) | Disaster loss |
| 55 | Удаление без approval | Data loss |
| 56 | Тесты на production | Business impact |
| 57 | Input не валидируется | Custom-code attack |
| 58 | Ошибки раскрывают secrets | Information leak |
| 59 | Dependency не проверяется | Supply-chain risk |
| 60 | Нет security owner | Risks unowned |
| 61 | Нет access review | Permission creep |
| 62 | Нет asset/app inventory | Unknown exposure |
| 63 | Нет local-copy destruction | Residual data |
| 64 | Юридические требования предполагаются | Compliance error |
| 65 | Безопасность считается только задачей Bitrix24 | Shared-responsibility gap |

```text
Identity + excessive privilege + leaked secret + no monitoring = high impact
```

## 31. Чек-лист Security Architect

- [ ] Users, roles, admins, contractors and service identities inventoried.
- [ ] CRM/Smart Process/tasks/files/telephony/BI rights tested.
- [ ] OAuth, webhooks, REST, apps and integrations have owners/scopes/revoke.
- [ ] Secrets, environments, logs and test data comply with policy.
- [ ] Backup/restore, monitoring, incident and offboarding runbooks tested.
- [ ] Secure development, change control and documentation enforced.

```text
Inventory → assess → remediate → verify → monitor → reassess
```

## 32. Чек-лист аудита портала

| Проверка | Evidence |
|---|---|
| Org/users/leavers/admins | Current lists and approvals |
| CRM entity roles | Role tests, not admin screenshots only |
| Apps/webhooks/integrations | Owner, scope, last use, revoke plan |
| Open links/exports/local copies | Registers and storage controls |
| Logs/backups | Availability, access, retention, restore evidence |
| Documentation | Architecture, contacts, changelog, incidents |

```text
Scope → read-only evidence → findings/risk → remediation → retest
```

## 33. Официальные источники

| Источник | Использование | Проверено в реестре |
|---|---|---|
| [Поддержка24](https://helpdesk.bitrix24.ru/) | Пользователи, права и актуальные функции | 2026-08-05 |
| [Bitrix24 REST API](https://apidocs.bitrix24.ru/) | REST/OAuth/webhooks/apps | 2026-08-05 |
| [API reference](https://apidocs.bitrix24.ru/api-reference/) | Scope, methods, events and errors | 2026-08-05 |
| [Local integrations](https://apidocs.bitrix24.ru/local-integrations/) | Official integration models | 2026-08-06 |
| [Marketplace](https://www.bitrix24.ru/apps/) | App inventory/source | 2026-08-05 |
| [Telephony](https://helpdesk.bitrix24.ru/open/6450911/) | Connection security context | 2026-08-05 |
| [BI datasets](https://helpdesk.bitrix24.ru/open/24355590/) | Analytics access context | 2026-08-05 |
| [On-Premise docs](https://dev.1c-bitrix.ru/docs/portal.php) | Box infrastructure | 2026-08-05 |

```text
Official source → portal/version verification → security control → evidence
```

Сторонние блоги, форумы, сертификаты и гарантии не использованы.
