# Building a production agent: the loop, wiring paths, orchestration, and human-in-the-loop
# Создание продакшен-агента: цикл, пути связывания, оркестрация и участие человека (human-in-the-loop)

An agent is a multi-step tool-use loop with managed context and a defined goal. You have already built the individual pieces, including tool schemas and context management. This section connects them into a working system and adds the layer that neither topics cover on their own.

> Агент — это многошаговый цикл использования инструментов (tool use) с управляемым контекстом и определённой целью. Отдельные составляющие, включая схемы инструментов и управление контекстом, вы уже построили. В этом разделе они объединяются в работающую систему и добавляется слой, который ни одна из этих тем сама по себе не охватывает.

When components run together across multiple turns, new failure modes appear that isolated testing does not catch. Routing decisions that worked in single-turn tests start to compound. Context fills faster than expected. A step that depends on a previous result gets the wrong input because an earlier tool call was structured incorrectly. The question that should precede every agent build is: does this problem require an agent?

> Когда компоненты работают вместе на протяжении нескольких ходов, появляются новые виды сбоев, которые изолированное тестирование не выявляет. Решения о маршрутизации, работавшие в одноходовых тестах, начинают накапливать ошибки. Контекст заполняется быстрее, чем ожидалось. Шаг, зависящий от предыдущего результата, получает неверные входные данные, потому что более ранний вызов инструмента был неправильно структурирован. Вопрос, который должен предшествовать созданию любого агента: действительно ли эта задача требует агента?

Agents carry coordination overhead, expanded context costs, and more surface area for failure than simpler patterns. Answering that question deliberately is the first design decision.

> Агенты несут накладные расходы на координацию, повышенные затраты на контекст и больше точек возможного отказа, чем более простые паттерны. Обдуманный ответ на этот вопрос — первое проектное решение.

## Workflow or agent: Make this decision before you write the first line
## Workflow или агент: примите это решение до того, как напишете первую строку кода

The most critical mistake in agent development is choosing the wrong pattern at the start. Workflows and agents solve different problems: using an agent when a workflow is sufficient adds behavioral complexity without adding capability. Using a workflow when an agent is needed produces a system that breaks whenever user input deviates from the predetermined path.

> Самая критическая ошибка при разработке агентов — выбрать неверный паттерн в самом начале. Workflow (заранее заданный процесс) и агенты решают разные задачи: использование агента там, где достаточно workflow, добавляет сложности поведения, не добавляя возможностей. Использование workflow там, где нужен агент, даёт систему, которая ломается всякий раз, когда ввод пользователя отклоняется от заранее заданного пути.

| Choose a workflow when… | Choose an agent when… |
|---|---|
| You can enumerate the exact steps in code. | You can specify the goal and the tools but not the exact path. |
| Error cost is real and step-level guardrails matter. | The path through work cannot be enumerated in advance. |
| Observability with standard tooling is required. | Non-determinism is acceptable and the agent's possible actions are constrained by its registered toolset. |
| The inputs are well-constrained to a known set. | User inputs vary unpredictably in content and structure. |
| Every execution of the task follows the same sequence. | The task requires creative sequencing of available tools. |

| Выбирайте workflow, когда… | Выбирайте агента, когда… |
|---|---|
| Точные шаги можно перечислить в коде. | Можно задать цель и инструменты, но не точный путь. |
| Цена ошибки реальна, и важны ограничители (guardrails) на уровне отдельных шагов. | Путь выполнения работы невозможно перечислить заранее. |
| Требуется наблюдаемость (observability) с помощью стандартных инструментов. | Недетерминированность допустима, а возможные действия агента ограничены зарегистрированным набором инструментов. |
| Входные данные чётко ограничены известным набором. | Ввод пользователей непредсказуемо меняется по содержанию и структуре. |
| Каждое выполнение задачи следует одной и той же последовательности. | Задача требует творческого выстраивания последовательности доступных инструментов. |

## The agent is the pattern. The wiring path is an implementation choice.
## Агент — это паттерн. Путь связывания — выбор реализации.

Once you have decided the task needs an agent, you have also decided on a pattern: a loop that calls tools, manages context, and runs until a goal is met. For single-agent systems, that pattern is constant across all three wiring paths. Multi-agent architectures, where a planner, executor, and evaluator run as separate agents handing off through structured artifacts, introduce additional design decisions beyond the loop itself. Those patterns are covered later in this track. That pattern does not change based on how you build it, what changes is how much of the loop you write yourself versus how much you hand to a library or a hosted service.

> Решив, что задаче нужен агент, вы тем самым выбрали и паттерн: цикл, который вызывает инструменты, управляет контекстом и работает, пока цель не достигнута. Для одноагентных систем этот паттерн одинаков для всех трёх путей связывания (wiring paths). Мультиагентные архитектуры, в которых планировщик (planner), исполнитель (executor) и оценщик (evaluator) работают как отдельные агенты и передают работу друг другу через структурированные артефакты, требуют дополнительных проектных решений помимо самого цикла. Эти паттерны рассматриваются далее в этом треке. Сам паттерн не меняется в зависимости от того, как вы его строите; меняется лишь то, какую часть цикла вы пишете сами, а какую отдаёте библиотеке или облачному сервису.

There are three wiring paths, and they sit on a spectrum of how much infrastructure you own. You can write the loop directly against the Messages API, which gives you full control and full responsibility. You can use the Agent SDK, which runs the same loop inside your own process and hands you tool execution, context management, and the iteration structure already built. Or you can use Claude Managed Agents (currently in public beta), where Anthropic runs the loop and the sandbox and your application streams events in and results back. The sections that follow teach the loop itself, because the loop is what stays constant. The path you choose decides who maintains the parts around it.

> Существует три пути связывания, и они располагаются на шкале того, какой объём инфраструктуры принадлежит вам. Можно написать цикл напрямую поверх Messages API — это даёт полный контроль и полную ответственность. Можно использовать Agent SDK, который запускает тот же цикл внутри вашего собственного процесса и предоставляет уже готовые выполнение инструментов, управление контекстом и структуру итераций. Или можно использовать Claude Managed Agents (сейчас в публичной бете), где Anthropic запускает цикл и песочницу (sandbox), а ваше приложение передаёт события и получает результаты в потоковом режиме. В следующих разделах разбирается сам цикл, потому что именно он остаётся неизменным. Выбранный путь определяет, кто сопровождает всё, что его окружает.

## Wiring paths: who runs the loop, and what you take on
## Пути связывания: кто запускает цикл и что берёте на себя вы

The three paths differ in one variable: how much of the agent's runtime you own. The table is ordered from top to bottom by how much infrastructure you hand off. Choose based on your deployment and compliance constraints, don't be tempted to choose the path that is just fastest to prototype.

> Три пути различаются одной переменной: какой частью среды выполнения агента владеете вы. Варианты упорядочены сверху вниз по тому, какой объём инфраструктуры вы передаёте. Выбирайте исходя из ограничений развёртывания и требований соответствия (compliance), а не поддавайтесь соблазну выбрать путь, который просто быстрее всего позволяет сделать прототип.

### Raw Messages API loop
### Собственный цикл поверх Messages API

- **Who runs the loop:** Your code runs every iteration. You send the request, read the tool-use blocks, execute the tools, and append the results yourself.
- **What you own:** The full loop, tool execution, context management, retries, and exit conditions. Nothing is provided for you.
- **Choose this when:** You need full control over each step, you have constraints a library does not accommodate, or you are teaching yourself how the loop works before adding abstraction.
- **What to check before committing:** The maintenance cost is yours. Every behavior the SDK would give you for free, including context management and parallel tool handling, becomes code you write and test.

> - **Кто запускает цикл:** каждую итерацию выполняет ваш код. Вы сами отправляете запрос, читаете блоки `tool_use`, выполняете инструменты и добавляете результаты.
> - **Чем владеете вы:** всем циклом, выполнением инструментов, управлением контекстом, повторными попытками (retries) и условиями завершения. Ничего не предоставляется готовым.
> - **Выбирайте, когда:** нужен полный контроль над каждым шагом, есть ограничения, которые библиотека не учитывает, или вы хотите сами разобраться, как работает цикл, прежде чем добавлять абстракции.
> - **Что проверить, прежде чем решиться:** стоимость сопровождения ложится на вас. Всё поведение, которое SDK дал бы бесплатно, включая управление контекстом и параллельную обработку инструментов, превращается в код, который вы пишете и тестируете сами.

### Agent SDK
### Agent SDK

- **Who runs the loop:** The SDK runs the loop inside your own process. It iterates and manages context, and your code still executes the tools the agent calls.
- **What you own:** Tool execution and the surrounding application. The SDK provides the loop structure, context management, and tool registration.
- **Choose this when:** You want the loop, context handling, and tool scaffolding that power Claude Code without rebuilding them, and you want the agent running in your own environment in Python or TypeScript.
- **What to check before committing:** Whether filesystem-based features like `CLAUDE.md` and skills load in the Agent SDK is controlled by the `settingSources` configuration. Do not rely on a default: always set `settingSources` explicitly to the sources you intend (for example, `["user", "project", "local"]` to match Claude Code CLI behavior, or `[]` to run fully isolated with only what you pass programmatically). Confirm current default behavior against the Agent SDK reference at build time.

> - **Кто запускает цикл:** SDK запускает цикл внутри вашего собственного процесса. Он выполняет итерации и управляет контекстом, а инструменты, которые вызывает агент, по-прежнему выполняет ваш код.
> - **Чем владеете вы:** выполнением инструментов и окружающим приложением. SDK предоставляет структуру цикла, управление контекстом и регистрацию инструментов.
> - **Выбирайте, когда:** вам нужны цикл, работа с контекстом и инфраструктура инструментов, на которых построен Claude Code, без их повторной разработки, и вы хотите, чтобы агент работал в вашей собственной среде на Python или TypeScript.
> - **Что проверить, прежде чем решиться:** загружаются ли в Agent SDK функции, основанные на файловой системе, такие как `CLAUDE.md` и skills, определяется конфигурацией `settingSources`. Не полагайтесь на значение по умолчанию: всегда явно задавайте в `settingSources` нужные источники (например, `["user", "project", "local"]`, чтобы повторить поведение Claude Code CLI, или `[]`, чтобы работать полностью изолированно только с тем, что вы передаёте программно). Проверяйте актуальное поведение по умолчанию в справочнике Agent SDK на этапе разработки.

### Claude Managed Agents
### Claude Managed Agents

- **Who runs the loop:** Anthropic runs the loop and the sandbox. Your application sends user events and streams results back over server-sent events.
- **What you own:** The application layer and the agent definition. You define the model, system prompt, tools, MCP servers, and skills once, then reference the agent by ID across sessions.
- **Choose this when:** You need long-running execution measured in minutes or hours, you want a managed sandbox, or you want to avoid building the loop, the sandbox, and the tool-execution layer at all. Also available on Claude Platform on AWS with some feature differences, verify capability parity against your deployment surface before committing.
- **What to check before committing:** Sessions are stateful and stored server-side, which means they are not currently eligible for Zero Data Retention or a HIPAA Business Associate Agreement. (See Anthropic API data retention documentation at platform.claude.com, verify at publish.)

> - **Кто запускает цикл:** Anthropic запускает цикл и песочницу. Ваше приложение отправляет пользовательские события и получает результаты в потоковом режиме через server-sent events.
> - **Чем владеете вы:** прикладным слоем и определением агента. Вы один раз задаёте модель, системный промпт, инструменты, MCP-серверы и skills, а затем ссылаетесь на агента по ID в разных сессиях.
> - **Выбирайте, когда:** нужно длительное выполнение, измеряемое минутами или часами, нужна управляемая песочница или вы хотите вообще не строить цикл, песочницу и слой выполнения инструментов. Также доступно на Claude Platform on AWS с некоторыми отличиями в функциональности — прежде чем решиться, проверьте паритет возможностей на вашей платформе развёртывания.
> - **Что проверить, прежде чем решиться:** сессии хранят состояние (stateful) и сохраняются на стороне сервера, поэтому сейчас на них не распространяется Zero Data Retention или соглашение HIPAA Business Associate Agreement. (См. документацию Anthropic API о хранении данных на platform.claude.com; проверьте на момент публикации.)

Currently in public beta, all endpoints require the `managed-agents-2026-04-01` beta header and behaviors may be refined between releases. Build with a migration plan in place.

> Сейчас функция находится в публичной бете: все эндпоинты требуют beta-заголовка `managed-agents-2026-04-01`, а поведение может уточняться между релизами. Разрабатывайте, имея план миграции.

## Claude Managed Agents: when to use
## Claude Managed Agents: когда использовать

The table above lists Managed Agents as the third path. Let's make that choice concrete because for some workloads it's the right default.

> В таблице выше Managed Agents указаны как третий путь. Давайте конкретизируем этот выбор, потому что для некоторых нагрузок это правильный вариант по умолчанию.

Here's the core difference: with a raw loop or the Agent SDK, your code runs the iteration. You send each request, read the tool-use blocks, run the tools, and append the results. With Managed Agents, Anthropic runs the loop and the sandbox for you. Your application defines the agent once (model, system prompt, tools, MCP servers, skills), refers to it by ID, sends user events, and streams the results back over server-sent events.

> Ключевое различие таково: при собственном цикле или Agent SDK итерации выполняет ваш код. Вы отправляете каждый запрос, читаете блоки `tool_use`, запускаете инструменты и добавляете результаты. С Managed Agents цикл и песочницу запускает за вас Anthropic. Ваше приложение один раз определяет агента (модель, системный промпт, инструменты, MCP-серверы, skills), ссылается на него по ID, отправляет пользовательские события и получает результаты в потоковом режиме через server-sent events.

### What you stop owning, and what you take on instead
### От чего вы избавляетесь и что берёте на себя взамен

| Category | What you stop owning | What you take on instead |
|---|---|---|
| **Execution & infrastructure** | The iteration loop, the execution sandbox, the retries inside the loop, and the tool-execution runtime. Anthropic runs all of it server-side. | An agent definition managed as a versioned API resource, plus an application layer that sends events and consumes the streamed results. |
| **Session duration & state** | Long-running execution management. Sessions can run for minutes or hours without your process holding the loop open. | Server-side session state. Sessions are stateful and stored by Anthropic, and are subject to its data handling policies and constraints (see the constraint note below). |
| **Sandbox lifecycle** | Sandbox provisioning and teardown for tool execution. | A dependency on the managed sandbox's available tools and its execution model, rather than your own environment. |

| Категория | От чего вы избавляетесь | Что берёте на себя взамен |
|---|---|---|
| **Выполнение и инфраструктура** | Цикл итераций, песочница для выполнения, повторные попытки внутри цикла и среда выполнения инструментов. Всё это Anthropic запускает на стороне сервера. | Определение агента, управляемое как версионируемый ресурс API, плюс прикладной слой, который отправляет события и обрабатывает результаты, приходящие в потоковом режиме. |
| **Длительность и состояние сессии** | Управление длительным выполнением. Сессии могут работать минутами или часами, и вашему процессу не нужно держать цикл открытым. | Состояние сессии на стороне сервера. Сессии хранят состояние (stateful), хранятся у Anthropic и подчиняются её политикам и ограничениям в отношении обработки данных (см. примечание об ограничениях ниже). |
| **Жизненный цикл песочницы** | Создание и удаление песочницы для выполнения инструментов. | Зависимость от инструментов, доступных в управляемой песочнице, и от её модели выполнения, а не от вашей собственной среды. |

### Choose Managed Agents when
### Выбирайте Managed Agents, когда

- **The task runs long.** Execution measured in minutes or hours is awkward to hold open in your own process, and the managed loop is built for exactly that.
- **You want a managed sandbox.** If you'd otherwise be building and securing an execution environment for tool calls, using Managed Agents takes a large piece of infrastructure off your plate.
- **You'd rather not build the loop, the sandbox, and the tool-execution layer at all,** and you're willing to define the agent as an API resource instead.

> - **Задача выполняется долго.** Выполнение, измеряемое минутами или часами, неудобно держать открытым в собственном процессе, а управляемый цикл создан именно для этого.
> - **Вам нужна управляемая песочница.** Если иначе вам пришлось бы строить и защищать среду выполнения для вызовов инструментов, Managed Agents снимает с вас большую часть инфраструктуры.
> - **Вы предпочитаете вообще не строить цикл, песочницу и слой выполнения инструментов** и готовы вместо этого определить агента как ресурс API.

### The constraint that decides it for regulated work
### Ограничение, которое решает всё для регулируемых сфер

Managed Agent sessions are stateful and stored server-side. That storage is the reason these sessions aren't currently eligible for Zero Data Retention or a HIPAA Business Associate Agreement. So, if your workload carries PHI or falls under a ZDR requirement, this path is ruled out no matter how well it fits operationally, and you route to the Agent SDK or a raw loop on a covered configuration instead. The governing constraint picks the path before convenience gets a say.

> Сессии Managed Agents хранят состояние (stateful) и сохраняются на стороне сервера. Именно из-за такого хранения на эти сессии сейчас не распространяется Zero Data Retention (ZDR) или соглашение HIPAA Business Associate Agreement. Поэтому если ваша нагрузка работает с защищённой медицинской информацией (PHI) или подпадает под требование ZDR, этот путь исключён, как бы хорошо он ни подходил с операционной точки зрения, и вместо него вы используете Agent SDK или собственный цикл на конфигурации, на которую эти требования распространяются. Определяющее ограничение выбирает путь раньше, чем удобство получает право голоса.

> 💡 *A common progression is to prototype on the Agent SDK locally, then move to Managed Agents for production. The core agent definition carries over conceptually. What changes is the format: the Agent SDK uses code-level and filesystem configuration, while Managed Agents defines the agent as a versioned API resource. Expect a re-expression step, not a direct export.*

> 💡 *Типичный путь — сделать прототип на Agent SDK локально, а затем перейти на Managed Agents для продакшена. Основное определение агента концептуально переносится. Меняется формат: Agent SDK использует конфигурацию на уровне кода и файловой системы, а Managed Agents определяет агента как версионируемый ресурс API. Рассчитывайте на этап переформулирования, а не на прямой экспорт.*

| | |
|---|---|
| **Handles well** | Long-running agents, and workloads where you'd rather not build or secure a sandbox and loop yourself. |
| **Adds cost or complexity** | Server-side stateful sessions, an agent-as-resource definition format, and a beta surface that can change between releases. |
| **Use a different approach** | For PHI or ZDR workloads, or when you need full in-process control, stay on the Agent SDK or a raw loop on a covered configuration. |

| | |
|---|---|
| **Хорошо подходит** | Долго работающие агенты и нагрузки, для которых вы не хотите сами строить и защищать песочницу и цикл. |
| **Добавляет затрат или сложности** | Сессии с состоянием на стороне сервера, формат определения «агент как ресурс» и бета-функциональность, которая может меняться между релизами. |
| **Используйте другой подход** | Для нагрузок с PHI или требованиями ZDR, а также когда нужен полный контроль внутри собственного процесса, оставайтесь на Agent SDK или собственном цикле на конфигурации, на которую распространяются эти требования. |

## Wiring the loop: the four steps that hold across every path
## Связывание цикла: четыре шага, общие для всех путей

The four steps below define a working agent loop no matter which path you build on. When you write the loop against the Messages API, you implement all four yourself. When you use the Agent SDK, it provides the structure for registering tools, setting the system prompt, and iterating the loop, and your code still handles tool execution. The steps are the same; what differs is how much you write versus inherit.

- **Register tools:** Each tool follows the same schema structure. The SDK registers them against the agent, so Claude knows what is available.
- **Set the system prompt:** Scope it to the agent's task. A broad system prompt produces broader, less reliable tool routing. A system prompt that names the specific task and the tools available for it produces more consistent behavior.
- **Handle the tool-use loop:** Whether you iterate the loop yourself or the SDK iterates it for you, your code handles execution. Every tool call Claude issues must be executed by your code and returned in a tool-result block.
- **Define exit conditions:** The agent loop runs until it receives a stop condition. Without explicit exit conditions, the agent will continue requesting tool calls beyond what the task requires. You should define when done means done.

> Четыре шага ниже определяют работающий цикл агента независимо от того, на каком пути вы строите. Когда вы пишете цикл поверх Messages API, все четыре шага вы реализуете сами. Когда вы используете Agent SDK, он предоставляет структуру для регистрации инструментов, задания системного промпта и выполнения итераций цикла, а выполнение инструментов по-прежнему остаётся за вашим кодом. Шаги одни и те же; различается лишь то, сколько вы пишете сами, а сколько получаете готовым.
>
> - **Зарегистрируйте инструменты:** каждый инструмент следует одной и той же структуре схемы. SDK регистрирует их для агента, чтобы Claude знал, что ему доступно.
> - **Задайте системный промпт:** ограничьте его задачей агента. Широкий системный промпт приводит к более размытой и менее надёжной маршрутизации инструментов. Системный промпт, который называет конкретную задачу и доступные для неё инструменты, даёт более стабильное поведение.
> - **Обрабатывайте цикл tool use:** выполняете ли вы итерации цикла сами или это делает за вас SDK, выполнением занимается ваш код. Каждый вызов инструмента, который выдаёт Claude, должен быть выполнен вашим кодом и возвращён в блоке `tool_result`.
> - **Определите условия завершения:** цикл агента работает, пока не получит условие остановки. Без явных условий завершения агент будет продолжать запрашивать вызовы инструментов сверх того, что требует задача. Вы должны определить, когда работа действительно считается выполненной.

### Loop wiring checklist: verify these regardless of path
### Чек-лист связывания цикла: проверьте это независимо от выбранного пути

| # | Item | What to verify |
|---|---|---|
| 1 | **Tools registered** | Every tool the agent may need is in the registration list. No unregistered tools are referenced in the system prompt. |
| 2 | **System prompt scoped** | The system prompt names the task and the available tools. It does not describe tools the agent does not have. It does not omit tools the agent does have that require scoping guidance. |
| 3 | **Tool-use loop implemented** | Your code handles every tool-use block Claude issues and returns a tool-result block for each one before the next assistant turn. All tool-use blocks from a single assistant turn must be resolved together. |
| 4 | **HITL insertion point defined** | At least one point in the loop has a human-in-the-loop check. See the section below for where to insert it. |
| 5 | **Exit conditions defined** | The loop has a clear stopping criterion that does not depend on Claude volunteering to stop. |

| # | Пункт | Что проверить |
|---|---|---|
| 1 | **Инструменты зарегистрированы** | Каждый инструмент, который может понадобиться агенту, есть в списке регистрации. Системный промпт не ссылается на незарегистрированные инструменты. |
| 2 | **Системный промпт ограничен задачей** | Системный промпт называет задачу и доступные инструменты. Он не описывает инструменты, которых у агента нет, и не пропускает имеющиеся у агента инструменты, для которых нужны указания по рамкам использования. |
| 3 | **Цикл tool use реализован** | Ваш код обрабатывает каждый блок `tool_use`, который выдаёт Claude, и возвращает блок `tool_result` для каждого из них до следующего хода ассистента. Все блоки `tool_use` из одного хода ассистента должны быть обработаны вместе. |
| 4 | **Определена точка участия человека (HITL)** | Как минимум в одной точке цикла есть проверка с участием человека (human-in-the-loop). Где её разместить, см. в разделе ниже. |
| 5 | **Определены условия завершения** | У цикла есть чёткий критерий остановки, который не зависит от того, решит ли Claude остановиться сам. |

## Human-in-the-loop (HITL): Insertion points and when each applies
## Участие человека (HITL): точки встраивания и когда применяется каждая из них

A human-in-the-loop checkpoint pauses agent execution and routes to a human review step before proceeding. The question that determines where to insert one is: what is the worst possible outcome if this step runs without a human check?

> Контрольная точка с участием человека (human-in-the-loop) приостанавливает выполнение агента и передаёт работу на проверку человеку, прежде чем продолжить. Где её разместить, определяет вопрос: каков наихудший возможный исход, если этот шаг выполнится без проверки человеком?

| Insertion point | What triggers the check | Risk level it addresses |
|---|---|---|
| **Before a destructive tool call** | The agent is about to execute a write, delete, or send operation. | **High:** irreversible actions where a wrong call cannot be undone |
| **After a planning step** | The agent has generated a plan and is about to begin executing it. | **Medium:** incorrect plans that would produce the wrong outcome even if all steps execute correctly |
| **On unexpected output** | The tool result contains an error flag, an empty result, or a value outside expected bounds. | **Variable:** catches failure modes that retry logic alone will not resolve |

| Точка встраивания | Что запускает проверку | На какой уровень риска рассчитана |
|---|---|---|
| **Перед деструктивным вызовом инструмента** | Агент собирается выполнить операцию записи, удаления или отправки. | **Высокий:** необратимые действия, при которых ошибочный вызов невозможно отменить |
| **После этапа планирования** | Агент составил план и собирается приступить к его выполнению. | **Средний:** неверные планы, которые привели бы к неправильному результату, даже если все шаги выполнены корректно |
| **При неожиданном выводе** | Результат инструмента содержит флаг ошибки, пустой результат или значение за пределами ожидаемых границ. | **Переменный:** выявляет сбои, которые одна лишь логика повторных попыток (retry) не устранит |

## Tool orchestration: Over-tooling and under-tooling
## Оркестрация инструментов: избыток и нехватка инструментов

The agent's routing behavior is shaped by two things, including how tools are described and how many tools are registered. Too many tools with overlapping descriptions produce erratic routing. Too few tools force the agent to either hallucinate a path or return an incomplete result.

> Поведение агента при выборе инструментов определяется двумя вещами: тем, как описаны инструменты, и тем, сколько их зарегистрировано. Слишком много инструментов с пересекающимися описаниями приводят к непредсказуемой маршрутизации. Слишком мало инструментов вынуждают агента либо выдумывать путь (галлюцинировать), либо возвращать неполный результат.

Over-tooling is the more common problem in production agents. Teams register every tool they might need "just in case" and discover that Claude's selection quality degrades as the tool surface grows. Start with the minimum set required for the task and add tools only when a specific gap in capability is confirmed.

> Избыток инструментов (over-tooling) — более распространённая проблема продакшен-агентов. Команды регистрируют все инструменты, которые могут понадобиться, «на всякий случай», а потом обнаруживают, что качество выбора инструментов у Claude падает по мере роста их набора. Начинайте с минимального набора, необходимого для задачи, и добавляйте инструменты только тогда, когда подтверждён конкретный пробел в возможностях.

| When agents are the right call | What you take on when you use an Agent | When to choose a workflow instead |
|---|---|---|
| Goal-directed tasks where the exact path cannot be enumerated in advance. Handling variable inputs that would require dozens of conditional branches in a workflow. | Agents add behavioral complexity: the path through the task emerges from the model's reasoning over accumulated context rather than from explicit branching logic in your code. Observability requires transcript-level tooling rather than standard operational logging. | When you can enumerate the steps in code, use a workflow. Agents are the last step in progression. Start with the simplest pattern that solves the problem, a single API call, then a workflow, then an agent. And move up only when the simpler pattern cannot handle the variability the task requires. |

| Когда агент — правильный выбор | Что вы берёте на себя, используя агента | Когда вместо этого выбрать workflow |
|---|---|---|
| Целенаправленные задачи, точный путь выполнения которых невозможно перечислить заранее. Обработка изменчивых входных данных, которая в workflow потребовала бы десятков условных ветвлений. | Агенты добавляют сложности поведения: путь выполнения задачи складывается из рассуждений модели над накопленным контекстом, а не из явной логики ветвления в вашем коде. Для наблюдаемости нужны инструменты на уровне полных записей диалога (transcripts), а не стандартное операционное логирование. | Когда шаги можно перечислить в коде, используйте workflow. Агенты — последняя ступень в этой последовательности. Начинайте с самого простого паттерна, решающего задачу: один вызов API, затем workflow, затем агент. Переходите на следующую ступень, только когда более простой паттерн не справляется с той вариативностью, которую требует задача. |

## Regulated data constraints set your delivery route and credentials before you write the wiring
## Ограничения для регулируемых данных определяют маршрут доставки и учётные данные ещё до того, как вы напишете связывание

If your data needs to be handled with specific constraints (e.g., attorney-client privilege, HIPAA, GDPR, FedRAMP, or an internal data-residency policy), that constraint decides which endpoint your code calls, which credentials it carries, and where its logs land before you make a single design choice about prompts, tools, or memory.

> Если ваши данные должны обрабатываться с определёнными ограничениями (например, адвокатская тайна (attorney-client privilege), HIPAA, GDPR, FedRAMP или внутренняя политика о местонахождении данных (data residency)), именно это ограничение определяет, к какому эндпоинту обращается ваш код, какие учётные данные он использует и куда попадают его логи, — ещё до того, как вы примете хоть одно проектное решение о промптах, инструментах или памяти.

As a developer you usually do not pick the surface, but you do write the code that targets a specific endpoint, attaches credentials, configures the region, and emits logs. Get the governing constraint named at the start, because the wrong client configuration is much more expensive to undo after the agent is wired than to set correctly the first time. The five constraints below cover the cases you are most likely to hit in production.

> Как разработчик вы обычно не выбираете платформу, но именно вы пишете код, который обращается к конкретному эндпоинту, подставляет учётные данные, настраивает регион и пишет логи. Назовите определяющее ограничение в самом начале, потому что неверную конфигурацию клиента гораздо дороже исправлять после того, как агент уже связан, чем правильно задать с первого раза. Пять ограничений ниже охватывают случаи, с которыми вы, скорее всего, столкнётесь в продакшене.

| Constraint | What it tends to rule out in code | What usually survives a code review |
|---|---|---|
| **Attorney-client privilege** | Calls from a consumer-grade Claude.ai surface that the firm cannot audit end-to-end. Code paths that send privileged document content to any endpoint the firm has not approved for privileged material, regardless of how the prompt or system message is structured. | Direct API or SDK calls from inside the firm's own application, authenticated via SSO, routed through a firm-approved LLM gateway with full request and response logging. Note that Anthropic's native Compliance Conversation content (prompts, responses, and tool call payloads) is not captured by Anthropic by default on direct API traffic, so the organization must implement conversation logging in the application layer and route it to an approved log destination. Tool calls and tool results stay inside the audited path. Confirm the final logging design with your Anthropic account team. |
| **HIPAA (PHI handling)** | Code that sends Protected Health Information to any endpoint or delivery route not covered by a Business Associate Agreement for the specific configuration in use. This includes any logging or retention path your code writes to that has not been scoped under the same BAA. | Direct API or SDK calls on a BAA-covered configuration. BAA coverage for Anthropic first-party API access is arranged with Anthropic, which provisions a dedicated HIPAA-enabled organization that enforces feature restrictions on its own end. Confirm the covered configuration with your Anthropic account team. An alternative is a cloud-mediated route via AWS Bedrock or GCP Vertex on the partner's existing HIPAA-eligible cloud account. **Note:** *the BAA does not cover Console, Workbench, beta features, or consumer plans. Not all API features are covered under the BAA, verify the current feature eligibility list in Anthropic's Implementation Guide before configuring.* |
| **GDPR and data residency** | Delivery routes where the region of model execution cannot be pinned in code, or where the request can be served from a region outside the approved geographic boundary. Defaulting to a global endpoint without specifying region is the common pattern that breaks here. | A cloud-mediated route such as Bedrock or Vertex, with the region pinned in the client configuration to a covered jurisdiction. The direct Anthropic API is a separate case; it does not currently provide EU data residency, so partners with EU data residency requirements should route through Bedrock or Vertex rather than calling the API directly. |
| **FedRAMP and government** | Any code path that calls an endpoint not on an authorized cloud environment at the required impact level. This includes development and test paths that hit the commercial endpoint while production hits the authorized one, because credentials and code patterns leak between them. | Three authorized routes exist as of publish time. Claude for Government (C4G) carries a direct FedRAMP High authorization held through Palantir Federal Cloud Service – Supporting Services (PFCS-SS). Claude via Amazon Bedrock GovCloud is approved for FedRAMP High and DoD IL4/5 workloads. Claude via Vertex AI Assured Workloads is also FedRAMP authorized. Claude Enterprise on AWS Marketplace is not FedRAMP authorized, so teams requiring FedRAMP compliance must use one of the three routes above. Verify current authorization status at trust.anthropic.com before configuring. |
| **Internal data-residency policy** | Calls from any SDK client configured against a cloud vendor outside the partner's approved list, regardless of whether the underlying technical capability would support the workload. Procurement-level constraints rule the code path out before engineering preferences enter the conversation. | The delivery route on the partner's approved cloud vendor. In code terms, that is whichever SDK client and endpoint configuration their CIO has already cleared. Build against that one rather than switching mid-project because another route looks easier. |

| Ограничение | Что оно обычно исключает в коде | Что обычно проходит код-ревью |
|---|---|---|
| **Адвокатская тайна (attorney-client privilege)** | Вызовы из потребительской платформы Claude.ai, которую фирма не может проверить от начала до конца. Пути в коде, отправляющие содержимое документов, составляющих адвокатскую тайну, на любой эндпоинт, не одобренный фирмой для таких материалов, независимо от того, как структурирован промпт или системное сообщение. | Прямые вызовы API или SDK изнутри собственного приложения фирмы с аутентификацией через SSO, проходящие через одобренный фирмой LLM-шлюз (gateway) с полным логированием запросов и ответов. Учтите, что собственное содержимое диалогов Compliance (промпты, ответы и данные вызовов инструментов) по умолчанию не сохраняется Anthropic при прямом трафике через API, поэтому организация должна реализовать логирование диалогов на уровне приложения и направлять его в одобренное хранилище логов. Вызовы инструментов и их результаты остаются внутри проверяемого пути. Согласуйте итоговую схему логирования с вашей командой по работе с клиентами в Anthropic. |
| **HIPAA (обработка PHI)** | Код, отправляющий защищённую медицинскую информацию (PHI) на любой эндпоинт или маршрут доставки, не охваченный соглашением Business Associate Agreement (BAA) для конкретной используемой конфигурации. Сюда относятся и любые пути логирования или хранения, в которые пишет ваш код и которые не включены в то же BAA. | Прямые вызовы API или SDK на конфигурации, охваченной BAA. Покрытие BAA для прямого доступа к API Anthropic оформляется с Anthropic, которая выделяет отдельную организацию с поддержкой HIPAA, где ограничения функциональности применяются на её стороне. Согласуйте охваченную конфигурацию с вашей командой по работе с клиентами в Anthropic. Альтернатива — маршрут через облако, AWS Bedrock или GCP Vertex, в существующем облачном аккаунте партнёра, соответствующем требованиям HIPAA. **Примечание:** *BAA не распространяется на Console, Workbench, бета-функции и потребительские тарифы. Не все функции API охвачены BAA — перед настройкой проверьте актуальный список допустимых функций в Implementation Guide от Anthropic.* |
| **GDPR и местонахождение данных (data residency)** | Маршруты доставки, где регион выполнения модели нельзя зафиксировать в коде или где запрос может обслуживаться из региона за пределами одобренной географической границы. Типичный паттерн, который здесь ломается, — использование глобального эндпоинта по умолчанию без указания региона. | Маршрут через облако, например Bedrock или Vertex, с регионом, зафиксированным в конфигурации клиента на допустимой юрисдикции. Прямой API Anthropic — отдельный случай: сейчас он не обеспечивает хранение данных в ЕС, поэтому партнёрам с требованиями к хранению данных в ЕС следует использовать Bedrock или Vertex, а не вызывать API напрямую. |
| **FedRAMP и государственный сектор** | Любой путь в коде, обращающийся к эндпоинту, который не находится в авторизованной облачной среде требуемого уровня воздействия (impact level). Сюда относятся и пути разработки и тестирования, которые обращаются к коммерческому эндпоинту, тогда как продакшен обращается к авторизованному, потому что учётные данные и паттерны кода между ними «протекают». | На момент публикации существует три авторизованных маршрута. Claude for Government (C4G) имеет прямую авторизацию FedRAMP High, полученную через Palantir Federal Cloud Service – Supporting Services (PFCS-SS). Claude через Amazon Bedrock GovCloud одобрен для нагрузок FedRAMP High и DoD IL4/5. Claude через Vertex AI Assured Workloads также имеет авторизацию FedRAMP. Claude Enterprise на AWS Marketplace не имеет авторизации FedRAMP, поэтому командам, которым нужно соответствие FedRAMP, необходимо использовать один из трёх маршрутов выше. Перед настройкой проверьте актуальный статус авторизации на trust.anthropic.com. |
| **Внутренняя политика о местонахождении данных** | Вызовы из любого SDK-клиента, настроенного на облачного поставщика вне одобренного партнёром списка, независимо от того, поддерживали бы технические возможности этого поставщика данную нагрузку. Ограничения на уровне закупок исключают такой путь в коде ещё до того, как в обсуждение вступят инженерные предпочтения. | Маршрут доставки через одобренного партнёром облачного поставщика. С точки зрения кода это тот SDK-клиент и та конфигурация эндпоинта, которые уже одобрил CIO партнёра. Стройте решение на них, а не переключайтесь в середине проекта из-за того, что другой маршрут кажется проще. |

This table covers the constraints that directly determine endpoint selection and credential configuration. SOC 2 is not in scope here. It governs how your systems are built and operated, not which endpoint your code calls, and is covered in Module 4 alongside other security posture and audit requirements.

> Эта таблица охватывает ограничения, которые напрямую определяют выбор эндпоинта и настройку учётных данных. SOC 2 здесь не рассматривается. Он регулирует то, как строятся и эксплуатируются ваши системы, а не то, к какому эндпоинту обращается ваш код, и рассматривается в Модуле 4 вместе с другими требованиями к уровню безопасности (security posture) и аудиту.

### Forward pointer
### Забегая вперёд

Module 4 (Production Engineering, Evals & Security) goes deep on secure-by-design patterns for IAM and privacy, defenses against prompt injection from untrusted inputs, runtime guardrails, and agent hardening. The role of this section is narrower: surface the constraint at the point in the build where it actually rules options out, which is when you pick the endpoint, the SDK client configuration, and the credentials your agent carries into production.

> В Модуле 4 (Production Engineering, Evals & Security) подробно разбираются паттерны безопасности по умолчанию (secure-by-design) для IAM и защиты персональных данных, защита от prompt injection из недоверенных входных данных, ограничители во время выполнения (runtime guardrails) и усиление защиты агентов (agent hardening). Задача этого раздела уже: обозначить ограничение в той точке разработки, где оно действительно исключает варианты, то есть когда вы выбираете эндпоинт, конфигурацию SDK-клиента и учётные данные, с которыми ваш агент выйдет в продакшен.
