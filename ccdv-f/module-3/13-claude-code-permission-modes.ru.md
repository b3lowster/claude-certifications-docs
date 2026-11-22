# Claude Code agent loop, permission modes, settings, and where a human gate goes
# Цикл агента в Claude Code, режимы разрешений, настройки и где ставить проверку человеком

Module 2 established how the agent loop works at the API level: the model calls tools, gets results back, and continues until the task is done.

> В Модуле 2 было показано, как работает цикл агента на уровне API: модель вызывает инструменты, получает результаты и продолжает работу, пока задача не будет выполнена.

Claude Code runs that same loop in your terminal but adds an additional layer: a permission system that gates every action the agent wants to take. Before you can configure anything, you need to understand how the loop runs and what the permission modes control.

> Claude Code запускает тот же цикл в вашем терминале, но добавляет ещё один слой: систему разрешений (permissions), которая контролирует каждое действие, которое хочет выполнить агент. Прежде чем что-либо настраивать, нужно понять, как работает цикл и чем управляют режимы разрешений (permission modes).

## How Claude Code works through a task: explore, plan, and code
## Как Claude Code выполняет задачу: исследование, план и код

When you hand Claude Code a task, it does not start writing immediately. It reads files, traces the relevant logic, and builds a picture of the codebase first; this is the exploration phase. Then, once it understands enough to propose a change, it creates a plan. A plan is a structured description of the edits it intends to make. Only after you review and approve the plan does it move into the code phase, where it writes and executes the changes.

> Когда вы передаёте Claude Code задачу, он не начинает писать код сразу. Сначала он читает файлы, прослеживает соответствующую логику и составляет картину кодовой базы — это фаза исследования (explore). Затем, когда он понимает достаточно, чтобы предложить изменение, он составляет план. План — это структурированное описание правок, которые он собирается внести. Только после того как вы просмотрите и одобрите план, он переходит к фазе кода (code), где пишет и выполняет изменения.

This sequence matters for two reasons. First, it produces better output: Claude Code understands the codebase before touching anything, so it makes fewer assumptions and catches more downstream effects. Second, it is where the permission modes plug in: plan mode holds Claude Code in the explore phase, blocking all file edits and shell commands until you release it, making it a useful default for unfamiliar codebases or high-stakes work.

> Эта последовательность важна по двум причинам. Во-первых, она даёт лучший результат: Claude Code разбирается в кодовой базе, прежде чем что-либо трогать, поэтому делает меньше допущений и замечает больше последствий изменений. Во-вторых, именно здесь подключаются режимы разрешений: режим планирования (plan mode) удерживает Claude Code в фазе исследования, блокируя любые правки файлов и команды оболочки (shell), пока вы его не отпустите, поэтому он удобен как вариант по умолчанию для незнакомых кодовых баз или работы с высокими ставками.

## Permission modes: approvals, gates, and constraints
## Режимы разрешений: одобрения, проверки и ограничения

Permission modes control how often Claude Code stops to ask for confirmation. Each mode makes a different tradeoff between speed and oversight. The right choice depends on how well you know the codebase and how reversible the changes are.

> Режимы разрешений определяют, как часто Claude Code останавливается, чтобы запросить подтверждение. Каждый режим — это свой компромисс между скоростью и контролем. Правильный выбор зависит от того, насколько хорошо вы знаете кодовую базу и насколько обратимы изменения.

Select each tab for what that mode auto-approves, what it still gates, and its limitations.

> Выберите каждую вкладку, чтобы увидеть, что этот режим одобряет автоматически, что по-прежнему требует подтверждения и какие у него ограничения.

### `default`

- **What it auto-approves:** Reads only. Prompts before nearly every edit or command.
- **What it still gates:** All file edits and shell commands require confirmation.
- **Limitations:** Safe but slow on trusted work. The baseline for any new project or unfamiliar codebase.

> - **Что одобряется автоматически:** только чтение. Запрашивает подтверждение почти перед каждой правкой или командой.
> - **Что по-прежнему требует подтверждения:** все правки файлов и команды оболочки.
> - **Ограничения:** безопасен, но медленный на доверенной работе. Базовый вариант для любого нового проекта или незнакомой кодовой базы.

### `acceptEdits`

- **What it auto-approves:** Reads, file edits, and common filesystem commands (`mkdir`, `touch`, `rm`, `rmdir`, `mv`, `cp`, and `sed`) inside the working directory. Auto-approval is scoped to paths inside the working directory, and protected paths still prompt.
- **What it still gates:** All other shell commands; writes outside the working directory; writes to protected paths.
- **Limitations:** Trusted local work where shell execution still needs a human eye. Not appropriate if the agent must run scripts.

> - **Что одобряется автоматически:** чтение, правки файлов и распространённые команды файловой системы (`mkdir`, `touch`, `rm`, `rmdir`, `mv`, `cp` и `sed`) внутри рабочей директории. Автоматическое одобрение ограничено путями внутри рабочей директории, а для защищённых путей подтверждение по-прежнему запрашивается.
> - **Что по-прежнему требует подтверждения:** все остальные команды оболочки; запись за пределами рабочей директории; запись в защищённые пути.
> - **Ограничения:** для доверенной локальной работы, где выполнение команд оболочки всё ещё требует присмотра человека. Не подходит, если агент должен запускать скрипты.

### `plan`

- **What it auto-approves:** Reads only. Research and proposes; makes no edits.
- **What it still gates:** All file edits and shell commands until you approve a plan.
- **Limitations:** Exploration and planning on sensitive or unfamiliar codebases. Not appropriate for tasks that must write output.

> - **Что одобряется автоматически:** только чтение. Исследует и предлагает; никаких правок не вносит.
> - **Что по-прежнему требует подтверждения:** все правки файлов и команды оболочки — до тех пор, пока вы не одобрите план.
> - **Ограничения:** для исследования и планирования в чувствительных или незнакомых кодовых базах. Не подходит для задач, которые должны записывать результат.

### `auto`

- **What it auto-approves:** Everything, but a separate classifier reviews each action first and blocks anything that escalates beyond your request, targets unrecognized infrastructure, or appears driven by hostile or inappropriate content.
- **What it still gates:** Production deploys and migrations, mass deletes, credential exfiltration, and force-push to main are blocked by default.
- **Limitations:** Reduces prompts but does not guarantee safety; this is a research preview, not a substitute for reviewing sensitive operations. Availability depends on plan, model version, and admin settings. Always verify current requirements before build.

> - **Что одобряется автоматически:** всё, но отдельный классификатор предварительно проверяет каждое действие и блокирует всё, что выходит за рамки вашего запроса, затрагивает неизвестную инфраструктуру или, по всей видимости, продиктовано враждебным или неприемлемым содержимым.
> - **Что по-прежнему требует подтверждения:** развёртывания и миграции в продакшене, массовые удаления, вывод учётных данных (credential exfiltration) и force-push в main блокируются по умолчанию.
> - **Ограничения:** уменьшает число запросов подтверждения, но не гарантирует безопасность; это исследовательская предварительная версия (research preview), а не замена проверки чувствительных операций. Доступность зависит от тарифа, версии модели и настроек администратора. Всегда проверяйте актуальные требования перед разработкой.

### `dontAsk`

- **What it auto-approves:** Only tools you pre-approved in an allow rule, plus read-only commands. Auto-DENIES everything else.
- **What it still gates:** Every tool call not on the allow list is denied. There is no queue for confirmation.
- **Limitations:** Built for locked-down CI and scripts. It restricts well, but it is not a way to reduce friction on local interactive work.

> - **Что одобряется автоматически:** только инструменты, заранее одобренные в разрешающем правиле (allow rule), плюс команды только для чтения. Всё остальное автоматически ОТКЛОНЯЕТСЯ.
> - **Что по-прежнему требует подтверждения:** каждый вызов инструмента, которого нет в списке разрешённых, отклоняется. Очереди на подтверждение нет.
> - **Ограничения:** создан для закрытых CI и скриптов. Хорошо ограничивает, но не является способом снизить трение при локальной интерактивной работе.

### `bypassPermissions`

- **What it auto-approves:** All tool calls. No confirmation prompts and no safety checks.
- **What it still gates:** Nothing in normal operation. The standard permission checks are bypassed; only catastrophic delete commands such as `rm -rf /` and `rm -rf ~` still trigger a last-resort prompt.
- **Limitations:** Only inside an isolated container or VM where the environment is disposable. Never on a developer workstation against a live codebase.

> - **Что одобряется автоматически:** все вызовы инструментов. Никаких запросов подтверждения и никаких проверок безопасности.
> - **Что по-прежнему требует подтверждения:** в обычной работе — ничего. Стандартные проверки разрешений обходятся; только катастрофические команды удаления, такие как `rm -rf /` и `rm -rf ~`, всё ещё вызывают последний запрос подтверждения.
> - **Ограничения:** только внутри изолированного контейнера или виртуальной машины, где среда одноразовая. Никогда — на рабочей машине разработчика с живой кодовой базой.

## Where does the configuration live and who it applies to
## Где хранится конфигурация и на кого она распространяется

Settings can be placed at several levels, and each level determines the scope of the rules it contains.

- **User level (`~/.claude/settings.json`):** Applies to every project on the machine. This is the right place for preferences that should follow you everywhere, such as a preferred default mode for exploration work.
- **Project level (`.claude/settings.json`, committed to the repo):** Applies to everyone who clones the repository. This is the right place for team-wide conventions, allow rules for the tools your project uses, and deny rules for paths that should not be touched.
- **Local project level (`.claude/settings.local.json`):** Personal overrides for one project, automatically git-ignored. This is the right place for your own preferences that should not be committed to the whole team.
- **Enterprise level (`managed-settings.json`, set by administrators):** Cannot be overridden by users or project files. The right place for organization-wide security controls such as denying edits to environment files or blocking specific shell commands across all projects.

> Настройки можно задавать на нескольких уровнях, и каждый уровень определяет область действия содержащихся в нём правил.
>
> - **Уровень пользователя (`~/.claude/settings.json`):** применяется ко всем проектам на машине. Подходящее место для предпочтений, которые должны следовать за вами везде, например предпочитаемого режима по умолчанию для исследовательской работы.
> - **Уровень проекта (`.claude/settings.json`, закоммичен в репозиторий):** применяется ко всем, кто клонирует репозиторий. Подходящее место для общекомандных соглашений, разрешающих правил для инструментов, которые использует проект, и запрещающих правил для путей, которые нельзя трогать.
> - **Локальный уровень проекта (`.claude/settings.local.json`):** личные переопределения для одного проекта, автоматически добавляемые в git-ignore. Подходящее место для ваших собственных предпочтений, которые не должны попасть в коммит для всей команды.
> - **Уровень предприятия (`managed-settings.json`, задаётся администраторами):** не может быть переопределён пользователями или файлами проекта. Подходящее место для общеорганизационных мер безопасности, например запрета на правку файлов окружения или блокировки определённых команд оболочки во всех проектах.

Allow and deny rules layer on top of the selected mode. A deny rule always wins over an allow rule, regardless of the mode in effect. The most durable governance control is an enterprise-level deny rule: it cannot be removed by any individual developer and applies even when a bypass mode is set.

> Разрешающие (allow) и запрещающие (deny) правила накладываются поверх выбранного режима. Запрещающее правило всегда имеет приоритет над разрешающим, независимо от действующего режима. Самый надёжный инструмент управления — запрещающее правило на уровне предприятия: его не может удалить ни один отдельный разработчик, и оно действует даже при включённом режиме обхода (bypass).

## Where a human still has to look: placing the review gate by worst-case cost
## Где человеку всё ещё нужно проверять: размещение точки проверки по цене наихудшего исхода

Permission modes and deny rules decide what the agent can do without asking. They do not decide where you, the human, still need to look before an action lands. That decision rests on one question, the same one that separates a safe mode from a risky one: what is the worst outcome if this action runs without a person checking it? The lower the cost of being wrong, the more you can let through. The higher the cost, and the harder it is to undo, the more a step needs a human gate before it executes.

> Режимы разрешений и запрещающие правила определяют, что агент может делать, не спрашивая. Они не определяют, где вам, человеку, всё ещё нужно посмотреть, прежде чем действие будет выполнено. Это решение опирается на один вопрос — тот же, что отличает безопасный режим от рискованного: каков наихудший исход, если это действие выполнится без проверки человеком? Чем ниже цена ошибки, тем больше можно пропускать. Чем выше цена и чем сложнее отменить действие, тем больше шагу нужна проверка человеком перед выполнением.

That same worst-case question places the gate whether the agent is writing code or running unattended in an automated step such as a bot that comments on or blocks a pull request. Three placements follow from it:

- **Let low-stakes, reversible actions through without a gate.** A formatting fix or an edit confined to the working directory carries little cost if it is wrong, so requiring a human to approve each one buys oversight you do not need and slows the work. This is the case `acceptEdits` is built for.
- **Gate any action that is hard to undo or reaches a sensitive path:** a write outside the working directory, a destructive shell command, or an edit to a security-relevant or protected file. The cost of a wrong call there is high, so the agent should pause and surface the action for a person before it runs. A deny rule enforces this deterministically, and `default` or `plan` mode keeps the prompt in place while you decide.
- **Never let the agent be the only gate on a change to code your team has marked sensitive.** There the agent's work is an input to a human decision, not a replacement for one, so a person must review the change before it merges no matter how confident the agent or its own review sounds.

> Тот же вопрос о наихудшем исходе определяет место проверки независимо от того, пишет ли агент код или работает без присмотра в автоматизированном шаге, например как бот, который комментирует или блокирует pull request. Из него следуют три варианта размещения:
>
> - **Пропускайте малорискованные обратимые действия без проверки.** Исправление форматирования или правка в пределах рабочей директории обходятся недорого в случае ошибки, поэтому требование одобрять каждую из них человеком даёт ненужный контроль и замедляет работу. Именно для этого случая создан `acceptEdits`.
> - **Ставьте проверку на любое действие, которое сложно отменить или которое затрагивает чувствительный путь:** запись за пределами рабочей директории, деструктивную команду оболочки или правку файла, важного для безопасности или защищённого. Цена ошибки здесь высока, поэтому агент должен остановиться и показать действие человеку до его выполнения. Запрещающее правило обеспечивает это детерминированно, а режим `default` или `plan` сохраняет запрос подтверждения, пока вы принимаете решение.
> - **Никогда не позволяйте агенту быть единственной проверкой изменений в коде, который ваша команда пометила как чувствительный.** Здесь работа агента — это входные данные для решения человека, а не его замена, поэтому человек должен проверить изменение до слияния, как бы уверенно ни звучал агент или его собственное ревью.

The placement of the gate and the choice of permission mode are the same decision viewed from two sides. The mode sets the default for a whole session, and the gate is where you override that default for the one action whose cost is too high to leave to the default; both come from asking what breaks if this runs unchecked.

> Размещение точки проверки и выбор режима разрешений — это одно и то же решение, рассмотренное с двух сторон. Режим задаёт поведение по умолчанию для всей сессии, а точка проверки — это место, где вы переопределяете это поведение для того единственного действия, цена которого слишком высока, чтобы оставлять его на умолчание; и то и другое следует из вопроса, что сломается, если это выполнится без проверки.

| | |
|---|---|
| **Cost** | Running in default mode on trusted work adds prompt latency to every tool call, and this accumulates on a long refactor. |
| **Complexity** | Multiple levels of settings and files with an override hierarchy require consistent care. A deny rule at the enterprise level that contradicts an allow rule at the project level needs to be understood by everyone maintaining the project config. |
| **Risk** | The risk lies in using the wrong mode for the context. For example, a bypass mode set out of impatience on a non-isolated machine removes every safety prompt between the agent and your live files, and, unlike the other modes, also removes the protected-path guard. |

| | |
|---|---|
| **Стоимость** | Работа в режиме `default` над доверенными задачами добавляет задержку на запрос подтверждения к каждому вызову инструмента, и при длительном рефакторинге она накапливается. |
| **Сложность** | Несколько уровней настроек и файлов с иерархией переопределений требуют постоянного внимания. Запрещающее правило на уровне предприятия, противоречащее разрешающему правилу на уровне проекта, должно быть понятно всем, кто сопровождает конфигурацию проекта. |
| **Риск** | Риск — в использовании неподходящего режима для конкретного контекста. Например, режим обхода, включённый из нетерпения на неизолированной машине, убирает все защитные запросы подтверждения между агентом и вашими рабочими файлами и, в отличие от других режимов, снимает также защиту защищённых путей. |
