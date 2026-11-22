# Connecting Claude to enterprise systems and authenticating it securely
# Подключение Claude к корпоративным системам и безопасная аутентификация

Earlier the module covered how to build an MCP server and configure its transport and scope. For a server used only by your team on an internal project, the GitHub personal-access-token example covers the authentication pattern.

> Ранее в модуле было рассмотрено, как создать MCP-сервер и настроить его транспорт (transport) и область действия (scope). Для сервера, которым пользуется только ваша команда во внутреннем проекте, подход к аутентификации показан на примере GitHub с персональным токеном доступа (personal access token).

This section covers what changes when the integration must work in a regulated environment: the identity, secret-handling, and data-residency questions that a prototype usually ignores become requirements that must be answered in the production deployment.

> В этом разделе рассматривается, что меняется, когда интеграция должна работать в регулируемой среде: вопросы идентификации, работы с секретами и места хранения и обработки данных (data residency), которые прототип обычно игнорирует, становятся требованиями, на которые продакшен-развёртывание обязано дать ответ.

## Why enterprise integration is different from a working prototype
## Чем корпоративная интеграция отличается от работающего прототипа

A prototype that connects Claude to an internal service answers one question: does the connection work? A production enterprise integration must answer several additional questions: Who is the model acting as, and is that identity auditable? What data can it access, and where does that data leave the organization? Can an administrator lock the configuration so no individual developer can change the authentication setup? Can the access be logged in a way that satisfies a compliance audit?

> Прототип, подключающий Claude к внутреннему сервису, отвечает на один вопрос: работает ли подключение? Корпоративная интеграция в продакшене должна ответить ещё на несколько: от чьего имени действует модель и можно ли эту идентичность проверить при аудите? К каким данным у неё есть доступ и где эти данные покидают организацию? Может ли администратор заблокировать конфигурацию так, чтобы ни один отдельный разработчик не мог изменить настройку аутентификации? Можно ли журналировать доступ так, чтобы это удовлетворяло аудиту на соответствие требованиям (compliance audit)?

These questions are not new to enterprise software; they have the same identity, access, and compliance requirements that apply to any external system touching regulated data. Treating them as part of the integration design is what separates a demo from something deployment-ready.

> Эти вопросы не новы для корпоративного ПО: к ним применяются те же требования к идентификации, доступу и соответствию нормам, что и к любой внешней системе, работающей с регулируемыми данными. Именно то, что они рассматриваются как часть проектирования интеграции, отличает демо от решения, готового к развёртыванию.

## Authentication patterns by service type
## Подходы к аутентификации в зависимости от типа сервиса

The right authentication mechanism depends on where the service runs and what identity model it supports. Select each tab for the pattern and when to use it.

> Правильный механизм аутентификации зависит от того, где работает сервис и какую модель идентификации он поддерживает. Выберите каждую вкладку, чтобы узнать подход и когда его использовать.

### Remote services with user identity
### Удалённые сервисы с идентификацией пользователя

Use OAuth. The MCP server returns a `401 Unauthorized` to signal that authentication is required. The client initiates a browser-based sign-in flow. After the user approves access, a token is issued and stored. No one copies a secret by hand; the OAuth flow is the expected pattern for cloud services, SaaS tools, and any integration where the user's identity is part of the authorization model. The Linear MCP server from earlier in this module uses this pattern; the GitHub server, by contrast, authenticates with a personal access token passed as a header.

> Используйте OAuth. MCP-сервер возвращает `401 Unauthorized`, сообщая, что требуется аутентификация. Клиент запускает процесс входа через браузер. После того как пользователь одобрит доступ, токен выдаётся и сохраняется. Никто не копирует секрет вручную; OAuth — ожидаемый подход для облачных сервисов, SaaS-инструментов и любой интеграции, где идентичность пользователя является частью модели авторизации. MCP-сервер Linear, рассмотренный ранее в этом модуле, использует этот подход; сервер GitHub, напротив, аутентифицируется с помощью персонального токена доступа, передаваемого в заголовке.

### Remote services with service identity
### Удалённые сервисы с идентификацией сервиса

Use an API key passed through an environment variable. The key identifies the service account. The key must never be committed to a configuration file; it lives in the environment at the point of execution. For a CI pipeline using the Agent SDK, the key is injected as a secret by the pipeline runner, not baked into the code.

> Используйте API-ключ, передаваемый через переменную окружения. Ключ идентифицирует сервисную учётную запись (service account). Ключ никогда не должен фиксироваться в файле конфигурации; он находится в окружении в момент выполнения. Для CI-пайплайна, использующего Agent SDK, ключ подставляется как секрет средством запуска пайплайна (pipeline runner), а не зашивается в код.

### Local services with file-system access
### Локальные сервисы с доступом к файловой системе

`stdio` transport with no network authentication. The security boundary is the file-system permission model. A denying rule in the settings files is the governance layer.

> Транспорт `stdio` без сетевой аутентификации. Границей безопасности служит модель прав доступа файловой системы. Слоем управления (governance) является запрещающее правило (deny rule) в файлах настроек.

Managing the secret itself is the other half of secure authentication. A credential never travels with the configuration that references it: the config file holds only a variable reference, and the value lives in an environment variable or a managed secret store injected at the point of execution. Store service-account keys in a secret manager rather than in files, and rotate them on a schedule and immediately after any suspected exposure. If a key is leaked, you must rotate it, but remember, you cannot rotate a value baked into committed code. Scope each credential to the narrowest access its task needs, so a compromised key reaches only what that one integration required.

> Управление самим секретом — вторая половина безопасной аутентификации. Учётные данные никогда не распространяются вместе с конфигурацией, которая на них ссылается: файл конфигурации содержит только ссылку на переменную, а само значение находится в переменной окружения или в управляемом хранилище секретов (secret store) и подставляется в момент выполнения. Храните ключи сервисных учётных записей в менеджере секретов (secret manager), а не в файлах, и выполняйте их ротацию по расписанию и сразу после любого подозрения на раскрытие. Если ключ утёк, его необходимо заменить, но помните: значение, зашитое в зафиксированный код, заменить нельзя. Ограничивайте каждые учётные данные минимальным доступом, необходимым для их задачи, чтобы скомпрометированный ключ давал доступ только к тому, что требовалось этой одной интеграции.

## Managing the secret after authentication: storage, rotation, and separation from config
## Управление секретом после аутентификации: хранение, ротация и отделение от конфигурации

Choosing the right authentication pattern establishes the connection, but keeping it is a separate problem. The MCP key leak mentioned earlier was not a bad choice of auth method, it was a credential that lived in the wrong place and could not be cleaned up once it spread. Three practices keep that from happening, and each one addresses a specific way a credential gets exposed.

> Выбор правильного подхода к аутентификации устанавливает подключение, но его сохранение — отдельная задача. Утечка ключа MCP, упомянутая ранее, была вызвана не неудачным выбором метода аутентификации: учётные данные хранились не в том месте, и после распространения их уже нельзя было вычистить. Предотвратить это помогают три практики, и каждая из них устраняет конкретный путь раскрытия учётных данных.

The first practice is separation: a credential never travels with the configuration that references it. The configuration file holds a variable reference, and the value lives somewhere the file does not. This is the rule the leaked-key failure broke. The reason it matters is mechanical: configuration files get committed, shared, and cloned. A value written inline rides along with every one of those copies, and a committed value enters repository history in a way that overwriting does not remove. If you keep the value out of the file, then the file stays safe to share.

> Первая практика — разделение: учётные данные никогда не распространяются вместе с конфигурацией, которая на них ссылается. Файл конфигурации содержит ссылку на переменную, а значение находится там, где файла нет. Именно это правило было нарушено при утечке ключа. Причина его важности чисто механическая: файлы конфигурации фиксируют в репозитории, передают и клонируют. Значение, записанное прямо в файл, попадает в каждую из этих копий, а зафиксированное значение входит в историю репозитория так, что перезапись его не удаляет. Если значение хранится вне файла, файлом можно безопасно делиться.

The second practice is where the value goes once it is out of the file. For a value that lives only on one machine or in one pipeline run, an environment variable injected at the point of execution is enough: the CI runner sets it as a secret, the configuration reads it by name, and nothing is written to disk. For a value that several services or people need, a secret store is better. A secret store is a managed service that holds credentials, returns them to authorized callers at runtime, and records who read what. It centralizes the value so a single rotation updates every consumer at once, and it removes the copies that accumulate when each service keeps its own credential in its own file. Reach for an environment variable when the secret is local and short-lived, and a secret store when the secret is shared or must be audited.

> Вторая практика — то, где хранится значение после того, как оно убрано из файла. Для значения, которое существует только на одной машине или в одном запуске пайплайна, достаточно переменной окружения, подставляемой в момент выполнения: CI-раннер задаёт её как секрет, конфигурация читает её по имени, и ничего не записывается на диск. Для значения, которое нужно нескольким сервисам или людям, лучше подходит хранилище секретов. Хранилище секретов — это управляемый сервис, который хранит учётные данные, выдаёт их авторизованным вызывающим сторонам во время выполнения и фиксирует, кто что прочитал. Оно централизует значение, так что одна ротация сразу обновляет его для всех потребителей, и устраняет копии, которые накапливаются, когда каждый сервис хранит свои учётные данные в собственном файле. Используйте переменную окружения, когда секрет локальный и короткоживущий, и хранилище секретов, когда секрет общий или требует аудита.

The third practice is rotation: replacing a credential with a new one on a schedule and immediately after any suspected exposure. Rotation is the only appropriate response to a leaked key, because a key that has been exposed cannot be made secret again. You must issue a new one. This is why the inline-credential pattern is so costly: a value baked into committed code cannot be rotated cleanly, since the old value stays in history and every consumer hardcoded to it breaks upon change. A credential read from a secret store or an environment variable rotates without touching the code that uses it, because the code references the value by name and the name does not change when the value behind it does.

> Третья практика — ротация: замена учётных данных новыми по расписанию и сразу после любого подозрения на раскрытие. Ротация — единственная правильная реакция на утёкший ключ, потому что раскрытый ключ уже нельзя снова сделать секретным. Нужно выпустить новый. Именно поэтому подход с учётными данными, записанными прямо в файл, так дорого обходится: значение, зашитое в зафиксированный код, нельзя чисто заменить, поскольку старое значение остаётся в истории, а каждый потребитель, у которого оно жёстко прописано, ломается при изменении. Учётные данные, которые читаются из хранилища секретов или переменной окружения, заменяются без изменения использующего их кода, потому что код обращается к значению по имени, а имя не меняется, когда меняется стоящее за ним значение.

Two habits can make rotation cheaper: scope each credential to the narrowest access its task needs, so a key that leaks reaches only what that integration required. Keep a record of which services use each credential, so a rotation does not surface its consumers.

> Две привычки делают ротацию дешевле: ограничивайте каждые учётные данные минимальным доступом, необходимым для их задачи, чтобы утёкший ключ давал доступ только к тому, что требовалось этой интеграции. Ведите учёт того, какие сервисы используют каждые учётные данные, чтобы ротация не выявляла потребителей неожиданно (по факту поломки).

The leaked-key failure earlier in the module identified the mistake: a credential written inline to a committed file. To prevent this from happening, apply these three practices: separation keeps the value out of the file, a secret store or environment variable gives the value a home that the file does not share, and rotation can help with recovery only when the first two are held.

> Случай с утечкой ключа, рассмотренный ранее в модуле, показал ошибку: учётные данные, записанные прямо в зафиксированный файл. Чтобы этого не произошло, применяйте три практики: разделение не допускает значение в файл, хранилище секретов или переменная окружения дают значению место, не связанное с файлом, а ротация помогает восстановиться, только если соблюдены первые две.

## What regulated industries add on top of working authentication
## Что добавляют регулируемые отрасли поверх работающей аутентификации

A financial services or healthcare customer asks more questions than "does authentication work?" They ask where data is processed, how access is logged, and whether an administrator can lock the configuration so a developer cannot change the auth setup during an audit window.

> Клиент из сферы финансовых услуг или здравоохранения задаёт больше вопросов, чем «работает ли аутентификация?». Он спрашивает, где обрабатываются данные, как журналируется доступ и может ли администратор заблокировать конфигурацию, чтобы разработчик не мог изменить настройку аутентификации в период аудита.

The enterprise managed configuration from earlier sections answers the last question: an administrator-deployed server configuration that cannot be overridden by individual users means the auth setup is consistent across the organization and does not depend on each developer's settings file being correct.

> На последний вопрос отвечает корпоративная управляемая конфигурация (enterprise managed configuration) из предыдущих разделов: конфигурация сервера, развёрнутая администратором и не переопределяемая отдельными пользователями, означает, что настройка аутентификации единообразна во всей организации и не зависит от правильности файла настроек каждого разработчика.

Audit hooks answer the logging question: a `PostToolUse` hook that logs every tool call and its parameters to an audit store provides the record a compliance review needs. The hook fires deterministically for every call, regardless of what the model decides, and the log is not something the model can skip.

> На вопрос о журналировании отвечают хуки аудита: хук `PostToolUse`, записывающий каждый вызов инструмента и его параметры в хранилище аудита, создаёт запись, необходимую для проверки на соответствие требованиям. Хук срабатывает детерминированно при каждом вызове, независимо от решений модели, и модель не может пропустить запись в журнал.

Data residency answers the processing question: a server configured with an HTTP endpoint in a specific region, combined with a platform deployment that pins processing to that region, gives a compliance reviewer a checkable answer to where data goes. This is why the infrastructure requirement and the platform choice from earlier in the module matter at audit time, not just at build time.

> На вопрос об обработке отвечает размещение данных (data residency): сервер, настроенный с HTTP-эндпоинтом в определённом регионе, в сочетании с развёртыванием платформы, привязывающим обработку к этому региону, даёт проверяющему проверяемый ответ на вопрос, куда уходят данные. Вот почему требования к инфраструктуре и выбор платформы, рассмотренные ранее в модуле, важны не только при сборке, но и во время аудита.

## Code modernization: applying the full module to legacy change
## Модернизация кода: применение всего модуля к изменению унаследованного кода

Code modernization is a useful test case for everything this module covers, because it concentrates on the risks each tool was designed to manage. Large-scale changes to an unfamiliar legacy codebase carry high blast radius, unpredictable dependencies, and limited reversibility. The tools from this module address each of those risks directly when you apply them before the work starts. The explore, plan, and code loop is the core workflow for this type of work. Plan mode holds the agent in the read-only explore phase while you build confidence in its changes. You can review the proposed edits, identify anything that touches paths you did not expect, and push back before a single file is modified. Hooks enforce guardrails that prevent edits to specific paths during the most sensitive phases. `CLAUDE.md` carries the conventions for the new target patterns, so the agent applies them consistently across the full scope of changes rather than drifting back to the legacy patterns it reads in the surrounding code.

> Модернизация кода — хороший проверочный пример для всего, что рассматривается в этом модуле, потому что в ней сосредоточены те риски, для управления которыми создан каждый инструмент. Масштабные изменения в незнакомой унаследованной (legacy) кодовой базе имеют большой радиус поражения (blast radius), непредсказуемые зависимости и ограниченную обратимость. Инструменты из этого модуля напрямую устраняют каждый из этих рисков, если применить их до начала работы. Основной рабочий процесс для такой работы — цикл «исследование, план, код» (explore, plan, code). Режим планирования (plan mode) удерживает агента в фазе исследования только для чтения, пока вы не обретёте уверенность в его изменениях. Вы можете просмотреть предлагаемые правки, выявить всё, что затрагивает неожиданные пути, и возразить ещё до изменения хотя бы одного файла. Хуки обеспечивают защитные ограничения (guardrails), запрещающие правки по определённым путям на самых чувствительных этапах. `CLAUDE.md` содержит соглашения для новых целевых шаблонов, поэтому агент последовательно применяет их во всём объёме изменений, а не возвращается к унаследованным шаблонам, которые видит в окружающем коде.

A responsible scoping approach for high-risk work addresses three questions before the session starts.

> Ответственный подход к определению рамок работы с высоким риском отвечает на три вопроса до начала сессии.

- **What is the blast radius if something goes wrong:** which systems depend on the code being changed, and what breaks downstream if an edit is wrong?
- **How are changes audited:** is there a `PostToolUse` hook logging every tool call, and does that log satisfy whoever needs to review what the agent touched?
- **Who approves each phase before the next one begins?** Plan mode enforces the boundary between exploration and execution, but the approval decision itself is yours to define and document before work begins.

> - **Каков радиус поражения, если что-то пойдёт не так:** какие системы зависят от изменяемого кода и что сломается дальше по цепочке, если правка окажется ошибочной?
> - **Как проводится аудит изменений:** есть ли хук `PostToolUse`, журналирующий каждый вызов инструмента, и устраивает ли этот журнал тех, кому нужно проверить, что затронул агент?
> - **Кто одобряет каждый этап перед началом следующего?** Режим планирования обеспечивает границу между исследованием и выполнением, но само решение об одобрении вы должны определить и задокументировать до начала работы.

These questions are not specific to modernization work. They apply to any high-risk agentic task. Code modernization surfaces them clearly because the scope is large, the codebase is unfamiliar, and the cost of getting it wrong is high.

> Эти вопросы касаются не только модернизации. Они относятся к любой агентной задаче с высоким риском. Модернизация кода просто наглядно их выявляет, потому что объём большой, кодовая база незнакома, а цена ошибки высока.

## The authentication and integration checklist
## Контрольный список по аутентификации и интеграции

The table below names the key decisions for each service type.

> В таблице ниже перечислены ключевые решения для каждого типа сервиса.

| Service type | Auth method | Where secrets live | What gets logged | Who can lock the config |
|---|---|---|---|---|
| Remote with user identity (SaaS, cloud) | OAuth | Token issued by OAuth provider and stored by client. | `PostToolUse` hook to audit log. | Administrator via enterprise managed settings. |
| Remote with service identity (internal API) | API key in environment variable | Environment only. Never in committed config. | `PostToolUse` hook to audit log. | Administrator via enterprise managed settings. |
| Local (file system, local DB) | File-system permissions | No credential needed. Deny rules enforce path access. | `PostToolUse` hook to audit log. | Deny rules in enterprise managed settings. |

| Тип сервиса | Метод аутентификации | Где хранятся секреты | Что журналируется | Кто может заблокировать конфигурацию |
|---|---|---|---|---|
| Удалённый с идентификацией пользователя (SaaS, облако) | OAuth | Токен выдаётся OAuth-провайдером и хранится клиентом. | Хук `PostToolUse` записывает в журнал аудита. | Администратор через корпоративные управляемые настройки (enterprise managed settings). |
| Удалённый с идентификацией сервиса (внутренний API) | API-ключ в переменной окружения | Только в окружении. Никогда в зафиксированной конфигурации. | Хук `PostToolUse` записывает в журнал аудита. | Администратор через корпоративные управляемые настройки. |
| Локальный (файловая система, локальная БД) | Права доступа файловой системы | Учётные данные не нужны. Доступ к путям контролируют запрещающие правила (deny rules). | Хук `PostToolUse` записывает в журнал аудита. | Запрещающие правила в корпоративных управляемых настройках. |

| | |
|---|---|
| **Cost** | OAuth flows add a one-time setup step per user per service. API key management requires a secret rotation process, and audit logging through `PostToolUse` hooks adds a small overhead to every tool call. |
| **Complexity** | Regulated environments add requirements that don't appear in a prototype. Identifying them during scoping is the discipline that keeps integrations on schedule. |
| **Risk** | The risk concentrates when a prototype moves toward production. A system that uses hardcoded credentials, has no audit log, and cannot be centrally locked will not pass a regulated customer's security review. The fixes are not hard, but they require attention before the review. |

| | |
|---|---|
| **Стоимость** | OAuth добавляет однократный шаг настройки для каждого пользователя в каждом сервисе. Управление API-ключами требует процесса ротации секретов, а журналирование аудита через хуки `PostToolUse` добавляет небольшие накладные расходы к каждому вызову инструмента. |
| **Сложность** | Регулируемые среды добавляют требования, которых нет в прототипе. Выявлять их на этапе определения рамок проекта — та дисциплина, которая позволяет укладываться в сроки интеграции. |
| **Риск** | Риск концентрируется при переходе прототипа в продакшен. Система с жёстко прописанными учётными данными, без журнала аудита и без возможности централизованной блокировки не пройдёт проверку безопасности у клиента из регулируемой отрасли. Исправления несложны, но ими нужно заняться до проверки. |
