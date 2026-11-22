# Packaging a workflow as a plugin: skills, custom commands, and marketplace install
# Упаковка рабочего процесса в плагин: навыки, пользовательские команды и установка из маркетплейса

Previously, we covered the mechanisms that give Claude Code durable context and enforce behavior: `CLAUDE.md` for always-on project memory, rules files for scoped guidance, hooks for deterministic guardrails, and subagents for isolated task delegation.

> Ранее мы рассмотрели механизмы, которые дают Claude Code постоянный контекст и обеспечивают нужное поведение: `CLAUDE.md` — для постоянно действующей памяти проекта, файлы правил (rules files) — для указаний с ограниченной областью действия, хуки (hooks) — для детерминированных защитных ограничений (guardrails) и субагенты (subagents) — для делегирования задач в изолированный контекст.

These mechanisms live in your `.claude` directory and are version-controlled with the project. Now we turn to the next question: how can you package that setup so a teammate can install it simply in one step instead of repeating your manual configuration by hand?

> Эти механизмы находятся в вашем каталоге `.claude` и хранятся в системе контроля версий вместе с проектом. Теперь перейдём к следующему вопросу: как упаковать эту настройку так, чтобы коллега мог установить её за один шаг, а не повторять вручную всю вашу конфигурацию?

## Skills are reusable workflows the agent loads on demand
## Навыки — это повторно используемые рабочие процессы, которые агент загружает по требованию

A skill is a portable Markdown file (`SKILL.md` file) placed in `.claude/skills`. The front matter identifies the skill and describes when it applies, and the body holds the steps. The same skill can run in Claude Code, be invoked through Messages API, or be loaded by the Agent SDK. What changes across the three isn't the file itself; it's where the skill runs, how it gets loaded, and what it's allowed to touch. A developer who has only ever seen skills in Claude Code may assume things that don't hold true on the API, so this section outlines the differences.

> Навык (skill) — это переносимый Markdown-файл (файл `SKILL.md`), размещённый в `.claude/skills`. Frontmatter определяет навык и описывает, когда он применяется, а в основной части содержатся шаги. Один и тот же навык может работать в Claude Code, вызываться через Messages API или загружаться Agent SDK. Между этими тремя вариантами меняется не сам файл, а то, где выполняется навык, как он загружается и к чему ему разрешён доступ. Разработчик, который видел навыки только в Claude Code, может предполагать то, что не выполняется в API, поэтому в этом разделе описаны различия.

## How the skill loads and runs in each
## Как навык загружается и выполняется в каждой среде

Select each tab for how the skill loads, where the steps run, and what you need to know.

> Выберите каждую вкладку, чтобы узнать, как загружается навык, где выполняются шаги и что нужно знать.

### Claude Code

- **How the skill loads:** Discovered from `.claude/skills` on the filesystem. Loads on a description match or when you invoke it by name.
- **Where the steps run:** In your terminal session, against your local files, under the active permission mode and deny rules.
- **What you need to know:** It's filesystem-based and is governed by the settings layer.

> - **Как загружается навык:** обнаруживается в `.claude/skills` в файловой системе. Загружается при совпадении с описанием (description) или когда вы вызываете его по имени.
> - **Где выполняются шаги:** в сессии вашего терминала, над вашими локальными файлами, с учётом активного режима разрешений (permission mode) и запрещающих правил (deny rules).
> - **Что нужно знать:** навык работает на основе файловой системы и управляется слоем настроек (settings).

### Messages API

- **How the skill loads:** Sent along with the request and run inside the code execution container, not your application's environment. Requires code-execution and skills beta headers.
- **Where the steps run:** Inside Anthropic's code execution container, not on your machine. The skill's filesystem and tool access are whatever that container provides.
- **What you need to know:** A skill that assumes local files or local tools won't behave the same way here, because it isn't running where those files are.

> - **Как загружается навык:** передаётся вместе с запросом и выполняется внутри контейнера выполнения кода (code execution container), а не в среде вашего приложения. Требуются бета-заголовки (beta headers) для code-execution и skills.
> - **Где выполняются шаги:** внутри контейнера выполнения кода Anthropic, а не на вашей машине. Доступ навыка к файловой системе и инструментам определяется тем, что предоставляет этот контейнер.
> - **Что нужно знать:** навык, рассчитанный на локальные файлы или локальные инструменты, будет вести себя здесь иначе, потому что он выполняется не там, где находятся эти файлы.

### Agent SDK

- **How the skill loads:** Loaded by the agent the SDK runs, but whether filesystem settings (`CLAUDE.md`, skills) load is controlled by the `settingSources` configuration. Do not rely on a default: always set it explicitly to the sources you intend, and confirm current default behavior against the Agent SDK reference at build time. You set it through `settingSources` (TypeScript) / `setting_sources` (Python).
- **Where the steps run:** In the process the SDK runs, which is your environment, once you've told it to load filesystem sources.
- **What you need to know:** The common surprise: a skill that worked in Claude Code does nothing under the SDK because `settingSources` was never set, so the skill never loaded.

> - **Как загружается навык:** загружается агентом, которого запускает SDK, но загрузка настроек из файловой системы (`CLAUDE.md`, навыки) управляется конфигурацией `settingSources`. Не полагайтесь на значение по умолчанию: всегда явно указывайте нужные источники и при сборке сверяйте текущее поведение по умолчанию со справочником Agent SDK. Параметр задаётся через `settingSources` (TypeScript) / `setting_sources` (Python).
> - **Где выполняются шаги:** в процессе, который запускает SDK, то есть в вашей среде, после того как вы указали ему загружать источники из файловой системы.
> - **Что нужно знать:** частая неожиданность — навык, который работал в Claude Code, ничего не делает в SDK, потому что `settingSources` так и не был задан и навык просто не загрузился.

### Claude Managed Agents

- **How the skill loads:** Defined once as an API resource that names the model, system prompt, tools, MCP servers, and skills. Anthropic loads the skill server-side when the agent runs, so there is no filesystem discovery step on your side.
- **Where the steps run:** Inside a sandbox Anthropic provisions and runs, not your environment. Your application sends user events and reads streamed results back. The skill has access to whatever that managed sandbox provides, not your local files.
- **What you need to know:** Currently a public beta that requires the `managed-agents-2026-04-01` beta header, and sessions are stored server-side, which means Managed Agents are not currently eligible for Zero Data Retention or HIPAA BAA coverage. Skills are attached when defining the agent resource, not at session time. Update the agent definition to change which skills are available.

> - **Как загружается навык:** определяется один раз как ресурс API, в котором указаны модель, системный промпт, инструменты, MCP-серверы и навыки. Anthropic загружает навык на стороне сервера при запуске агента, поэтому на вашей стороне нет шага обнаружения навыков в файловой системе.
> - **Где выполняются шаги:** внутри песочницы (sandbox), которую создаёт и запускает Anthropic, а не в вашей среде. Ваше приложение отправляет пользовательские события и читает потоковые результаты. Навык имеет доступ к тому, что предоставляет эта управляемая песочница, а не к вашим локальным файлам.
> - **Что нужно знать:** сейчас это публичная бета-версия, которая требует бета-заголовка `managed-agents-2026-04-01`, а сессии хранятся на стороне сервера, поэтому Managed Agents сейчас не подпадают под Zero Data Retention или покрытие HIPAA BAA. Навыки подключаются при определении ресурса агента, а не во время сессии. Чтобы изменить набор доступных навыков, обновите определение агента.

## Three portability rules
## Три правила переносимости

- **Write the description as the matching criterion.** The model loads a skill by comparing your request to its description, so a description that identifies when the skill applies works in every runtime, but a vague one fails to load in all of them.
- **Don't assume a local filesystem or local tools exist inside the skill body.** A skill that shells out to a local command works in Claude Code but breaks on the Messages API, where it runs in a container without a command. Keep the skill's steps confined to what the runtime is guaranteed to provide, or document the dependency.
- **Remember that subagents don't inherit skills.** This was true in Module 2, and is still true here: a subagent starts clean, so a skill the parent relied on has to be listed for the subagent explicitly, in every runtime that supports subagents.

> - **Пишите описание как критерий соответствия.** Модель загружает навык, сравнивая ваш запрос с его описанием, поэтому описание, которое чётко определяет, когда навык применяется, работает в любой среде выполнения (runtime), а расплывчатое не загрузится ни в одной из них.
> - **Не рассчитывайте на наличие локальной файловой системы или локальных инструментов в теле навыка.** Навык, который вызывает локальную команду оболочки, работает в Claude Code, но ломается в Messages API, где он выполняется в контейнере, где этой команды нет. Ограничивайте шаги навыка тем, что среда выполнения гарантированно предоставляет, или документируйте зависимость.
> - **Помните, что субагенты не наследуют навыки.** Это было верно в Модуле 2 и остаётся верным здесь: субагент начинает с чистого листа, поэтому навык, на который опирался родительский агент, нужно явно указать для субагента в каждой среде выполнения, поддерживающей субагентов.

The practical takeaway is that you can author a skill once and reuse it, but you must specifically design for the ability to use it across terminals. A skill that's scoped to a clear description and free of local-environment assumptions ports cleanly across runtimes, but one that assumes a specific local environment does not.

> Практический вывод: навык можно написать один раз и использовать повторно, но его нужно специально проектировать так, чтобы он работал в разных средах. Навык с чётким описанием и без допущений о локальной среде без проблем переносится между средами выполнения, а навык, рассчитанный на конкретную локальную среду, — нет.

| | |
|---|---|
| **Handles well** | A task-specific procedure authored once and reused across the interactive terminal, an API integration, and a headless SDK job. |
| **Adds complexity** | Each runtime loads and sandboxes the skill differently, so you must account for beta headers on the API and `settingSources` on the SDK. |
| **Use a different approach** | For instructions that must apply to every session in a project, `CLAUDE.md` is still the right tool. Skills are for on-demand, portable procedures. |

| | |
|---|---|
| **Хорошо подходит** | Процедура для конкретной задачи, написанная один раз и повторно используемая в интерактивном терминале, в интеграции через API и в фоновом (headless) задании SDK. |
| **Добавляет сложность** | Каждая среда выполнения по-своему загружает навык и изолирует его в песочнице, поэтому нужно учитывать бета-заголовки в API и `settingSources` в SDK. |
| **Используйте другой подход** | Для инструкций, которые должны действовать в каждой сессии проекта, по-прежнему подходит `CLAUDE.md`. Навыки предназначены для переносимых процедур, загружаемых по требованию. |

## Giving a workflow an explicit entry point
## Явная точка входа для рабочего процесса

A custom command is a shortcut for a defined procedure. In current Claude Code, skills are the recommended format for both explicit and automatic invocation: you invoke a skill directly with `/skill-name`, or Claude loads it automatically when relevant. The older `.claude/commands/` directory format still works but is a legacy process. Use skills with `disable-model-invocation: true` in the frontmatter when you want a workflow that only runs when you explicitly call it.

> Пользовательская команда (custom command) — это ярлык для определённой процедуры. В текущей версии Claude Code навыки — рекомендуемый формат как для явного, так и для автоматического вызова: вы вызываете навык напрямую через `/skill-name`, или Claude загружает его автоматически, когда это уместно. Старый формат с каталогом `.claude/commands/` всё ещё работает, но считается устаревшим (legacy). Используйте навыки с `disable-model-invocation: true` во frontmatter, если нужен рабочий процесс, который запускается только при явном вызове.

Plugin commands are namespaced automatically: the plugin's name becomes the prefix, so a `run-tests` command in a plugin named `payments` is invoked as `/payments:run-tests`. This is why two plugins can both ship a `run-tests` command without colliding. Authors should treat the plugin name as part of the interface, since it prefixes every command you ship, and be aware that renaming the plugin renames them all.

> Команды плагинов автоматически получают пространство имён (namespace): имя плагина становится префиксом, поэтому команда `run-tests` в плагине с именем `payments` вызывается как `/payments:run-tests`. Именно поэтому два плагина могут содержать команду `run-tests` без конфликта. Авторам стоит считать имя плагина частью интерфейса, поскольку оно становится префиксом каждой поставляемой команды, и помнить, что переименование плагина переименовывает их все.

## The packaging layer that makes a setup installable
## Слой упаковки, который делает настройку устанавливаемой

A plugin bundles skills, hooks, subagents, and MCP servers into a single installable unit. Plugins can be packaged and distributed through a marketplace, which is a catalog of plugins that someone else has created and shared. The official Anthropic marketplace is available automatically when you start Claude Code, and you can add third-party marketplaces hosted in a GitHub repository with a command like `/plugin marketplace add <owner/repo>`. Teammates can then run one simple install command to get the same setup. The plugin replaces a page of manual setup steps with a versioned, auditable install. The plugin places components as follows:

> Плагин (plugin) объединяет навыки, хуки, субагентов и MCP-серверы в единый устанавливаемый пакет. Плагины можно упаковывать и распространять через маркетплейс (marketplace) — каталог плагинов, которые кто-то другой создал и опубликовал. Официальный маркетплейс Anthropic доступен автоматически при запуске Claude Code, а сторонние маркетплейсы, размещённые в репозитории GitHub, можно добавить командой вида `/plugin marketplace add <owner/repo>`. После этого коллегам достаточно выполнить одну простую команду установки, чтобы получить ту же настройку. Плагин заменяет страницу ручных шагов настройки версионируемой установкой, которую можно проверить (аудировать). Плагин размещает компоненты следующим образом:

- **Skills** go in a skills directory.
- **Hooks, subagents, and settings** go in their respective locations.
- **The plugin manifest** describes the bundle, and the install command wires it into the target installation. Plugins can be downloaded by individuals or at an enterprise-wide level.

> - **Навыки** помещаются в каталог skills.
> - **Хуки, субагенты и настройки** помещаются в соответствующие им места.
> - **Манифест плагина (plugin manifest)** описывает пакет, а команда установки подключает его к целевой установке. Плагины могут загружать как отдельные пользователи, так и вся организация (на уровне предприятия).

Enterprise administrators can deploy plugins organization-wide through managed settings. A managed marketplace allowlist gates which marketplace sources users are permitted to add, so the organization controls where plugins can come from. The allowlist restricts what users can add but does not register marketplaces automatically. If you would like to push a marketplace to all users without requiring them to run the add command themselves, pair the allowlist setting with `extraKnownMarketplaces` in managed settings. The precedence comes from the deployment scope: because managed settings sit above user and project settings in the configuration hierarchy, a plugin deployed at managed scope takes priority and cannot be overridden by users or project files. Review the reference layer for the exact setting names.

> Администраторы предприятия могут развёртывать плагины на всю организацию через управляемые настройки (managed settings). Управляемый список разрешённых маркетплейсов (managed marketplace allowlist) определяет, какие источники маркетплейсов пользователи могут добавлять, поэтому организация контролирует, откуда могут поступать плагины. Список разрешённых ограничивает то, что пользователи могут добавить, но не регистрирует маркетплейсы автоматически. Если нужно распространить маркетплейс на всех пользователей, не заставляя их самих выполнять команду добавления, сочетайте настройку списка разрешённых с `extraKnownMarketplaces` в управляемых настройках. Приоритет определяется областью развёртывания: поскольку управляемые настройки стоят выше пользовательских и проектных в иерархии конфигурации, плагин, развёрнутый на управляемом уровне, имеет приоритет и не может быть переопределён пользователями или файлами проекта. Точные названия настроек смотрите в справочном разделе.

| | |
|---|---|
| **Cost** | Skills add context cost upon activation, but a plugin adds installation and maintenance overhead. The question to ask is whether you want to pay the setup cost once, as you do with a plugin install, or repeatedly, as you do when every developer runs the same manual steps by hand. |
| **Complexity** | A plugin that hard-codes absolute paths in its skills will install correctly for the author and fail for everyone else, because any path or environment assumption baked into a skill or hook command is the thing most likely to break across machines. |
| **Risk** | A plugin carries the components it bundles into every install. It's important to remember that a deny rule or hook the author relied on locally is not included unless it is explicitly listed as part of the bundle. If the skills or hooks are tied to a guardrail that is not included in the bundle, then the protection does not carry over to a teammate's machine. |

| | |
|---|---|
| **Стоимость** | Навыки добавляют стоимость в контексте при активации, а плагин добавляет накладные расходы на установку и сопровождение. Вопрос в том, хотите ли вы заплатить за настройку один раз, как при установке плагина, или многократно, как в случае, когда каждый разработчик вручную выполняет одни и те же шаги. |
| **Сложность** | Плагин, в навыках которого жёстко прописаны абсолютные пути, корректно установится у автора и не заработает у всех остальных, потому что любое допущение о путях или среде, зашитое в навык или команду хука, чаще всего ломается при переносе между машинами. |
| **Риск** | Плагин переносит в каждую установку те компоненты, которые в него упакованы. Важно помнить, что запрещающее правило (deny rule) или хук, на которые автор полагался локально, не включаются, если они явно не указаны как часть пакета. Если навыки или хуки завязаны на защитное ограничение (guardrail), не включённое в пакет, эта защита не переносится на машину коллеги. |
