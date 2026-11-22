# Durable project context with CLAUDE.md, rules files, hooks, and subagents
# Постоянный контекст проекта: CLAUDE.md, файлы правил, хуки и субагенты

Previously we saw how Claude Code gates actions through permission modes and settings files. That configuration layer controls what the agent is allowed to do.

> Ранее мы рассмотрели, как Claude Code контролирует действия с помощью режимов разрешений (permission modes) и файлов настроек (settings files). Этот слой конфигурации определяет, что агенту разрешено делать.

This cluster builds on top of it: now you'll learn how to configure what the agent knows and how it behaves, so the rules and project context you define in a session are still in effect at the start of the next one.

> Этот блок строится поверх него: теперь вы узнаете, как настроить то, что агент знает и как он себя ведёт, чтобы правила и контекст проекта, заданные в одной сессии, продолжали действовать с самого начала следующей.

## CLAUDE.md: the project file that loads into every session
## CLAUDE.md: файл проекта, который загружается в каждую сессию

Every time Claude Code starts in a project directory, it looks for a file named `CLAUDE.md` at the root and reads it. The contents are appended to your prompt before any message from you arrives. This means every convention, constraint, and command you put in `CLAUDE.md` is present from the first prompt of every session, without you having to re-state it.

> Каждый раз, когда Claude Code запускается в каталоге проекта, он ищет в корне файл с именем `CLAUDE.md` и читает его. Его содержимое добавляется к вашему промпту ещё до того, как придёт какое-либо ваше сообщение. Это значит, что каждое соглашение, ограничение и команда, которые вы поместили в `CLAUDE.md`, присутствуют с первого промпта каждой сессии, и вам не нужно повторять их заново.

The `/init` command scans your codebase and generates a starter `CLAUDE.md`. The generated file is a great baseline but should be validated before using. Refine it to hold the rules that control the outcome of your prompts: your testing commands, your framework conventions, the paths the agent should not touch, and the style decisions that differ from defaults.

> Команда `/init` сканирует вашу кодовую базу и создаёт стартовый `CLAUDE.md`. Сгенерированный файл — отличная основа, но перед использованием его нужно проверить. Доработайте его так, чтобы в нём были правила, определяющие результат ваших промптов: команды для запуска тестов, соглашения вашего фреймворка, пути, которые агент не должен трогать, и стилистические решения, отличающиеся от стандартных.

Size is the main failure mode. A `CLAUDE.md` that keeps growing with every new instruction can dilute the rules that matter most. A larger file consumes more of the context window, which makes any single instruction a smaller fraction of what loads, and that reduces the chance that the agent follows the one rule that catches a real mistake. Hold `CLAUDE.md` to the constraints that change behavior and move everything else into Skills that load on demand.

> Главная причина сбоев — размер. `CLAUDE.md`, который разрастается с каждой новой инструкцией, может размыть самые важные правила. Чем больше файл, тем большую часть контекстного окна (context window) он занимает, поэтому каждая отдельная инструкция составляет всё меньшую долю загружаемого, и это снижает шанс, что агент выполнит то единственное правило, которое ловит реальную ошибку. Оставляйте в `CLAUDE.md` только ограничения, которые меняют поведение, а всё остальное переносите в Skills, загружаемые по требованию.

## Rules instruction files: scoping guidance to where it applies
## Файлы правил: инструкции только там, где они нужны

In the previous section, we established that `CLAUDE.md` loads into every session and should hold the instructions that apply across the whole project. The next question is what to do with guidance that matters only in one part of the codebase. That is where rules instruction files come in: they let you apply instructions only where they are relevant, instead of loading them into every session.

> В предыдущем разделе мы выяснили, что `CLAUDE.md` загружается в каждую сессию и должен содержать инструкции, относящиеся ко всему проекту. Следующий вопрос — что делать с указаниями, которые важны только в одной части кодовой базы. Здесь на помощь приходят файлы правил (rules instruction files): они позволяют применять инструкции только там, где они уместны, вместо того чтобы загружать их в каждую сессию.

`CLAUDE.md` is always on, and rules files add a narrower layer on top of that baseline. They live in the project's `.claude/rules/` directory and can be scoped to specific paths using a `paths` glob in their YAML frontmatter. A rule scoped this way loads into context only when Claude Code works with files matching the pattern, this allows a rule to be applied to one part of the codebase without cluttering the rest of the context.

> `CLAUDE.md` действует всегда, а файлы правил добавляют поверх этой основы более узкий слой. Они находятся в каталоге проекта `.claude/rules/` и могут быть ограничены конкретными путями с помощью glob-шаблона `paths` в их YAML frontmatter. Правило с такой областью действия загружается в контекст, только когда Claude Code работает с файлами, соответствующими шаблону. Так правило можно применить к одной части кодовой базы, не засоряя остальной контекст.

Note that the scoping comes from the frontmatter, not from file placement. Rules files can be organized into subdirectories of `.claude/rules/` (e.g., `.claude/rules/database/`), but that structure is organizational only; a rules file without a `paths` field loads unconditionally at launch, with the same priority as `CLAUDE.md`, no matter where it sits inside `.claude/rules/`.

> Обратите внимание: область действия задаётся во frontmatter, а не расположением файла. Файлы правил можно раскладывать по подкаталогам `.claude/rules/` (например, `.claude/rules/database/`), но эта структура служит только для организации; файл правил без поля `paths` загружается безусловно при запуске, с тем же приоритетом, что и `CLAUDE.md`, где бы он ни находился внутри `.claude/rules/`.

In practice, put broad project memory and universal constraints in `CLAUDE.md`, and put narrow, path-specific guidance in rules files scoped with `paths`. A constraint like "never modify the database schema" lives in `CLAUDE.md` because it applies everywhere. A constraint like "all SQL in the database module must include an explicit transaction boundary" lives in `.claude/rules/database.md` with frontmatter such as:

> На практике общую память проекта и универсальные ограничения помещайте в `CLAUDE.md`, а узкие указания для конкретных путей — в файлы правил с полем `paths`. Ограничение вроде «никогда не изменяй схему базы данных» живёт в `CLAUDE.md`, потому что действует везде. Ограничение вроде «весь SQL в модуле базы данных должен содержать явную границу транзакции» живёт в `.claude/rules/database.md` с frontmatter, например:

```yaml
---
paths:
  - "src/db/**/*.sql"
---
```

so it enters context only when Claude is working with those files.

> чтобы оно попадало в контекст, только когда Claude работает с этими файлами.

## Hooks: running your own scripts at fixed points in the lifecycle
## Хуки: запуск собственных скриптов в фиксированных точках жизненного цикла

A Hook allows you to intercept and control tool calls before or after they execute. When you write a specific rule in `CLAUDE.md` telling the agent to run Prettier after every file edited, the agent will follow it most of the time. Alternatively, a hook makes it happen every single time without exceptions because the hook fires independently of what the model decides to do.

> Хук (hook) позволяет перехватывать и контролировать вызовы инструментов до или после их выполнения. Если вы напишете в `CLAUDE.md` конкретное правило, требующее от агента запускать Prettier после каждого отредактированного файла, агент будет соблюдать его в большинстве случаев. Хук же гарантирует, что это произойдёт каждый раз без исключений, потому что он срабатывает независимо от того, что решит модель.

Hooks are defined in settings files and configured using the `/hooks` command. Each hook is bound to a lifecycle event, an optional matcher that scopes it to specific tool types, and a command that runs when the event fires. The core events for most guardrail and automation use cases are:

> Хуки задаются в файлах настроек и настраиваются командой `/hooks`. Каждый хук привязан к событию жизненного цикла (lifecycle event), необязательному фильтру (matcher), ограничивающему его определёнными типами инструментов, и команде, которая выполняется при срабатывании события. Основные события для большинства сценариев защитных ограничений (guardrails) и автоматизации:

- **`PreToolUse`:** Runs before a tool call executes. Because it runs first, a `PreToolUse` hook can examine the tool call and exit with code 2 to block it, writing the reason to stderr as feedback the agent sees. This is how you enforce access controls at the configuration layer rather than hoping the agent respects a `CLAUDE.md` instruction.
- **`PostToolUse`:** Runs after a tool call completes. Since the call has already happened, this event cannot block it, which makes it the right place for automated side effects: running a code formatter after an edit, triggering tests after a file change, or logging the operation for an audit trail.
- **`UserPromptSubmit`:** Runs when you submit a prompt, before the model processes it. Use it when you need to inject context or validate the request before any work starts.
- **`Stop`:** Runs when the model finishes responding. Use it for follow-up actions that belong at the end of a turn, such as notifications, cleanup tasks, or committing the audit log.
- **`Notification`:** Runs when Claude Code sends a notification, which occurs when Claude needs permission to use a tool or after Claude Code has been idle for 60 seconds. Use it to route those signals to an external channel or logging system.
- **`SessionStart`:** Runs when a session starts or resumes. Use it to initialize state, validate environment variables, or confirm required services are reachable before the agent begins work.
- **`SessionEnd`:** Runs when a session ends. Use it for teardown tasks, final audit writes, or notifications that the session has closed.

> - **`PreToolUse`:** выполняется до вызова инструмента. Поскольку он срабатывает первым, хук `PreToolUse` может проверить вызов инструмента и завершиться с кодом 2, чтобы заблокировать его, записав причину в stderr как обратную связь, которую увидит агент. Так контроль доступа обеспечивается на уровне конфигурации, а не надеждой на то, что агент соблюдёт инструкцию из `CLAUDE.md`.
> - **`PostToolUse`:** выполняется после завершения вызова инструмента. Поскольку вызов уже произошёл, это событие не может его заблокировать, поэтому оно подходит для автоматических побочных действий: запуска форматировщика кода после правки, запуска тестов после изменения файла или записи операции в журнал аудита (audit trail).
> - **`UserPromptSubmit`:** выполняется, когда вы отправляете промпт, до того как модель его обработает. Используйте его, когда нужно добавить контекст или проверить запрос до начала работы.
> - **`Stop`:** выполняется, когда модель заканчивает ответ. Используйте его для завершающих действий в конце хода (turn): уведомлений, очистки или фиксации журнала аудита.
> - **`Notification`:** выполняется, когда Claude Code отправляет уведомление — это происходит, когда Claude нужно разрешение на использование инструмента или после 60 секунд бездействия Claude Code. Используйте его, чтобы направлять эти сигналы во внешний канал или систему логирования.
> - **`SessionStart`:** выполняется при запуске или возобновлении сессии. Используйте его, чтобы инициализировать состояние, проверить переменные окружения или убедиться, что нужные сервисы доступны, до того как агент начнёт работу.
> - **`SessionEnd`:** выполняется при завершении сессии. Используйте его для завершающей очистки, финальных записей аудита или уведомлений о закрытии сессии.

A hook that blocks edits to a production configuration path using a `PreToolUse` event enforces that constraint at every tool call during every session, regardless of permission mode. That is the difference between a guardrail and a convention.

> Хук на событие `PreToolUse`, блокирующий правки по пути с продакшен-конфигурацией, обеспечивает это ограничение при каждом вызове инструмента в каждой сессии, независимо от режима разрешений. В этом и состоит разница между защитным ограничением (guardrail) и соглашением (convention).

## Subagents: delegating work to an isolated context
## Субагенты: передача работы в изолированный контекст

A subagent is a specialized assistant that Claude Code can delegate tasks to, and each assistant runs a task in its own separate context and returns only its output. It does not inherit your main conversation history, the files you have accumulated in context, or your current session state. When you send a task to a subagent, it starts from a clean slate, does the work, and hands back the result.

> Субагент (subagent) — это специализированный помощник, которому Claude Code может делегировать задачи; каждый такой помощник выполняет задачу в собственном отдельном контексте и возвращает только результат. Он не наследует историю вашего основного разговора, файлы, накопленные в контексте, или текущее состояние сессии. Когда вы передаёте задачу субагенту, он начинает с чистого листа, выполняет работу и возвращает результат.

The built-in subagents differ in what they load at startup; this difference determines how your project rules apply. Always check the current list in the Claude Code docs, because the set has grown over time, but know that the specific split that affects your project rules holds across versions. The Explore and Plan built-in subagents skip `CLAUDE.md` and git status to keep research fast and cheap. They are optimized for speed, so project-level rules and repository state defined in `CLAUDE.md` are not in their context when they run. The general-purpose subagent loads both. If you delegate a task to Explore or Plan and a rule from your `CLAUDE.md` applies, it's because that context was not loaded. For tasks where your project constraints must be respected, use the general-purpose subagent or a custom subagent that explicitly loads the rules it needs.

> Встроенные субагенты различаются тем, что они загружают при запуске, и от этого зависит, как применяются правила вашего проекта. Всегда сверяйтесь с актуальным списком в документации Claude Code, потому что со временем он расширялся, но имейте в виду, что разделение, влияющее на правила проекта, сохраняется во всех версиях. Встроенные субагенты Explore и Plan пропускают `CLAUDE.md` и git status, чтобы исследование было быстрым и дешёвым. Они оптимизированы для скорости, поэтому правила уровня проекта и состояние репозитория, заданные в `CLAUDE.md`, во время их работы в контексте отсутствуют. Субагент general-purpose загружает и то, и другое. Если вы делегируете задачу Explore или Plan и правило из вашего `CLAUDE.md` не соблюдается, это потому, что этот контекст не был загружен. Для задач, где ограничения проекта обязательно должны соблюдаться, используйте субагент general-purpose или собственный (custom) субагент, который явно загружает нужные ему правила.

Custom subagents also do not automatically see your skills. If you define a custom subagent in `.claude/agents` and it needs a specific skill, you must explicitly list that skill in the agent's front matter. Built-in agents do not have preloaded skills. If a built-in agent needs skill-backed behavior, the correct path is to create a custom subagent with those skills listed in its configuration.

> Собственные субагенты также не видят ваши навыки (skills) автоматически. Если вы определяете собственный субагент в `.claude/agents` и ему нужен определённый навык, этот навык необходимо явно указать во frontmatter агента. У встроенных агентов нет предзагруженных навыков. Если встроенному агенту нужно поведение на основе навыков, правильный путь — создать собственный субагент с этими навыками, перечисленными в его конфигурации.

## Choosing the mechanism
## Выбор механизма

The map below names each mechanism, what it loads, when it runs, its context cost, and what belongs in it. Use it to decide which mechanism carries a specific piece of project knowledge, since each one makes a different tradeoff between how much context it costs and how reliably it applies.

> Карта ниже описывает каждый механизм: что он загружает, когда срабатывает, сколько контекста стоит и что в нём должно храниться. Используйте её, чтобы решить, какой механизм должен нести конкретный фрагмент знаний о проекте, поскольку каждый из них по-своему балансирует между стоимостью в контексте и надёжностью применения.

### `CLAUDE.md`

- **What it loads:** Full file contents prepended to context at session start.
- **When it runs:** Every session, unconditionally.
- **Context cost:** Persistent per session. Dilutes with size.
- **Belongs here:** Universal project constraints, commands, and framework decisions.

> - **Что загружает:** полное содержимое файла, добавляемое в начало контекста при старте сессии.
> - **Когда срабатывает:** в каждой сессии, безусловно.
> - **Стоимость в контексте:** постоянная в течение сессии. С ростом размера правила размываются.
> - **Что здесь хранить:** универсальные ограничения проекта, команды и решения по фреймворку.

### Rules file
### Файл правил

- **When it runs:** When Claude reads a file matching the rule's `paths` patterns. Unscoped rules load at session start.
- **Context cost:** Path-scoped: adds to context only when triggered. Unscoped: same persistent cost as `CLAUDE.md`.
- **Belongs here:** Path-specific guidance that would be noise everywhere else.

> - **Когда срабатывает:** когда Claude читает файл, соответствующий шаблонам `paths` правила. Правила без области действия загружаются при старте сессии.
> - **Стоимость в контексте:** с областью действия по путям — добавляется в контекст только при срабатывании. Без области действия — такая же постоянная стоимость, как у `CLAUDE.md`.
> - **Что здесь хранить:** указания для конкретных путей, которые в любом другом месте были бы шумом.

### Hook
### Хук

- **What it loads:** Runs your script at the lifecycle event. No content added to context.
- **When it runs:** At the configured event (`PreToolUse`, `PostToolUse`, etc.).
- **Context cost:** Minimal: only the script output if routed back to Claude.
- **Belongs here:** Enforced guardrails, automated side effects, audit logging.

> - **Что загружает:** запускает ваш скрипт при событии жизненного цикла. В контекст ничего не добавляется.
> - **Когда срабатывает:** при настроенном событии (`PreToolUse`, `PostToolUse` и т. д.).
> - **Стоимость в контексте:** минимальная — только вывод скрипта, если он возвращается Claude.
> - **Что здесь хранить:** обязательные защитные ограничения, автоматические побочные действия, журнал аудита.

### Subagent
### Субагент

- **What it loads:** Task context only. Isolated from the main session.
- **When it runs:** When dispatched by the main session for a delegated task.
- **Context cost:** Returns a summary, not the full task history.
- **Belongs here:** Exploration, investigation, and tasks whose output would otherwise bloat the main context. Also useful for tasks that can be broken down and parallelized.

> - **Что загружает:** только контекст задачи. Изолирован от основной сессии.
> - **Когда срабатывает:** когда основная сессия отправляет ему делегированную задачу.
> - **Стоимость в контексте:** возвращает сводку, а не полную историю выполнения задачи.
> - **Что здесь хранить:** исследование, анализ и задачи, вывод которых иначе раздул бы основной контекст. Также полезен для задач, которые можно разбить на части и выполнять параллельно.
