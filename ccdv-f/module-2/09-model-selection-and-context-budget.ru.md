# Model selection and keeping multi-turn sessions in budget
# Выбор модели и удержание многоходовых сессий в рамках бюджета

You make one early choice: which model runs the workload. The Claude family covers a range of cost, latency, and capability tradeoffs, so the model you pick sets the price and speed floor that every later decision moves within.

> Одно решение вы принимаете в самом начале: какая модель будет выполнять нагрузку. Семейство Claude охватывает целый спектр компромиссов между стоимостью, задержкой и возможностями, поэтому выбранная модель задаёт базовый уровень цены и скорости, в рамках которого принимаются все последующие решения.

Once the model is set, the next constraint is the context window: the full span of text the model can take in at once, including your prompt, the conversation so far, and every tool result. Every tool result Claude returns gets appended to the context window and stays there for the rest of the session. In a single-turn prompt, that's invisible. In a multi-step agent session running ten or twenty tool calls, the window fills up fast, and once it fills, the agent either compacts (losing detail) or stalls before the task is done.

> Когда модель выбрана, следующее ограничение — контекстное окно: весь объём текста, который модель может принять за один раз, включая ваш промпт, диалог на текущий момент и каждый результат инструмента. Каждый результат инструмента, который возвращает Claude, добавляется в контекстное окно и остаётся там до конца сессии. В одноходовом промпте это незаметно. В многошаговой агентной сессии с десятью или двадцатью вызовами инструментов окно заполняется быстро, и после этого агент либо выполняет сжатие (compaction), теряя детали, либо останавливается, не завершив задачу.

So, the question for any agent workflow is whether you've decided in advance what goes into the context window, what comes back out as a summary, and what never enters at all. That set of choices is context engineering.

> Поэтому для любого агентного процесса ключевой вопрос — решили ли вы заранее, что попадает в контекстное окно, что возвращается из него в виде краткого изложения (summary) и что не попадает туда вовсе. Этот набор решений и называется контекстной инженерией (context engineering).

## Model selection: Start with Sonnet, move deliberately
## Выбор модели: начинайте с Sonnet, меняйте модель обдуманно

The Claude model family currently spans four tiers: Fable, Opus, Sonnet, and Haiku, each optimized for different cost, latency, and capability tradeoffs. Sonnet is the balanced default for most production workloads. Haiku is built for speed and cost efficiency on tasks that fit its capability envelope. Opus handles demanding work above the Sonnet envelope, and Fable is Anthropic's most capable model, built for the most demanding tasks including complex reasoning, advanced coding, research synthesis, and sophisticated agentic workflows where maximum intelligence is the priority. Confirm the current lineup and model identifiers against platform.claude.com/docs at build time.

> Семейство моделей Claude сейчас включает четыре уровня: Fable, Opus, Sonnet и Haiku, каждый из которых оптимизирован под свой компромисс между стоимостью, задержкой и возможностями. Sonnet — сбалансированный вариант по умолчанию для большинства продакшен-нагрузок. Haiku создана для скорости и экономичности на задачах, которые укладываются в границы её возможностей. Opus справляется с требовательными задачами, выходящими за границы возможностей Sonnet, а Fable — самая мощная модель Anthropic, созданная для наиболее сложных задач, включая сложные рассуждения, продвинутое программирование, синтез результатов исследований и сложные агентные процессы, где приоритетом является максимальный интеллект. Проверяйте актуальную линейку и идентификаторы моделей на platform.claude.com/docs на этапе разработки.

The default starting point is Sonnet. Move up to Opus only when an eval set tells you Sonnet isn't meeting your quality bar. Move down to Haiku only when an eval set tells you the quality regression is acceptable at your task, not just to save costs. Your decision to move models should always be a measured decision.

> Отправная точка по умолчанию — Sonnet. Переходите на Opus, только когда набор eval-ов показывает, что Sonnet не дотягивает до вашей планки качества. Переходите на Haiku, только когда набор eval-ов показывает, что снижение качества допустимо для вашей задачи, а не просто ради экономии. Решение о смене модели всегда должно быть взвешенным.

## The context window is not a free resource
## Контекстное окно — не бесплатный ресурс

Think of the context window as the amount of space Claude can hold in working memory. Every message you send, every tool result you return, every document you inject, and every response Claude generates occupies space in that window. If a request is already larger than the context window, the Messages API rejects it with a validation error before generation; if a request fits but generation reaches the ceiling partway, current models return the output generated so far with a `model_context_window_exceeded` stop reason. Neither path silently truncates your oldest content. If you want a session to keep running past the window limit, your application must manage that itself by trimming or summarizing history before the next request goes out.

> Представляйте контекстное окно как объём рабочей памяти, который может удерживать Claude. Каждое отправленное вами сообщение, каждый возвращённый результат инструмента, каждый добавленный документ и каждый сгенерированный Claude ответ занимают место в этом окне. Если запрос уже превышает контекстное окно, Messages API отклоняет его с ошибкой валидации до начала генерации; если запрос помещается, но в процессе генерации упирается в потолок, текущие модели возвращают уже сгенерированный вывод с причиной остановки `model_context_window_exceeded`. Ни в одном из этих случаев самое старое содержимое не обрезается незаметно. Если вы хотите, чтобы сессия продолжала работать после достижения лимита окна, ваше приложение должно само управлять этим, обрезая или суммаризируя историю перед отправкой следующего запроса.

In development, the window rarely fills because test inputs are small and sessions are short. In production, tool outputs are often three to five times longer than test fixtures, sessions run for more turns, and the window fills at turn eight rather than turn fifty, which means they fill earlier than development. The cost of not planning for this is a production outage.

> При разработке окно заполняется редко, потому что тестовые входные данные небольшие, а сессии короткие. В продакшене вывод инструментов часто в три-пять раз длиннее тестовых данных (test fixtures), сессии длятся больше ходов, и окно заполняется на восьмом ходе, а не на пятидесятом, то есть раньше, чем при разработке. Цена отсутствия планирования — сбой в продакшене.

## Four strategies for staying in budget
## Четыре стратегии, позволяющие оставаться в рамках бюджета

The previous section made the case for moving state out of the live context window. The reason behind that is the budget. Every token in the window costs money on input and adds latency to the response, and a long session compounds both. The four strategies below are concrete ways to manage that budget, each suited to a different shape of conversation.

> В предыдущем разделе обосновывалась необходимость выносить состояние за пределы активного контекстного окна. Причина этого — бюджет. Каждый токен в окне стоит денег на входе и добавляет задержку к ответу, а в длинной сессии оба эффекта накапливаются. Четыре стратегии ниже — конкретные способы управлять этим бюджетом, и каждая подходит для своего типа диалога.

| Strategy | What it does | When to apply | What continuity you lose |
|---|---|---|---|
| **Pruning** | Lets you jump back to an earlier message and continue from there, removing the conversation that came after. | After Claude has gone down an unproductive path or accumulated debugging back-and-forth that won't help the next task. | The work done after the rewind point is gone. If Claude learned something useful in that stretch, it has to relearn it. |
| **Compaction** (`/compact` in Claude Code; server-side compaction in the API, a beta strategy the platform performs for you, with manual summarization as the client-side alternative) | Summarizes the conversation history into a condensed version that preserves the key information Claude has learned. The summary costs fewer tokens than the original turns. | When the session is approaching the context ceiling but you want to keep working on the same feature with the knowledge Claude has built up. | Details can be lost in the summarization. Anything not captured in the summary will not be available to Claude going forward. |
| **Clearing** (`/clear` in Claude Code; new session in API) | Starts a new conversation with empty context. Nothing from the previous session carries forward. | When the next task is completely different from the current one, and previous context would only introduce bias or confusion. | All session context is gone. Anything Claude needs to remember across sessions has to be put somewhere persistent, like a `CLAUDE.md` file. |
| **Subagent Handoffs** | Spawns a subagent in its own isolated context window with only the task description and system prompt it needs. The subagent does the work and returns a summary. | When a subtask is self-contained enough to delegate, especially exploration work where the journey clutters the main context but the answer is short. | Visibility into how the subagent reached its conclusion. The intermediate steps are discarded with the subagent's context. |

| Стратегия | Что она делает | Когда применять | Какую непрерывность вы теряете |
|---|---|---|---|
| **Обрезка (Pruning)** | Позволяет вернуться к более раннему сообщению и продолжить с него, удалив ту часть диалога, которая шла после. | Когда Claude пошёл по непродуктивному пути или накопил переписку по отладке, которая не поможет в следующей задаче. | Работа, выполненная после точки возврата, теряется. Если Claude узнал на этом отрезке что-то полезное, ему придётся узнать это заново. |
| **Сжатие (Compaction)** (`/compact` в Claude Code; серверное сжатие в API — бета-стратегия, которую платформа выполняет за вас, а альтернатива на стороне клиента — ручная суммаризация) | Сводит историю диалога к сжатой версии, сохраняющей ключевую информацию, которую узнал Claude. Краткое изложение занимает меньше токенов, чем исходные ходы. | Когда сессия приближается к потолку контекста, но вы хотите продолжать работу над той же функцией, сохранив знания, накопленные Claude. | При суммаризации детали могут теряться. Всё, что не попало в краткое изложение, в дальнейшем будет недоступно Claude. |
| **Очистка (Clearing)** (`/clear` в Claude Code; новая сессия в API) | Начинает новый диалог с пустым контекстом. Ничего из предыдущей сессии не переносится. | Когда следующая задача полностью отличается от текущей, и предыдущий контекст лишь внёс бы предвзятость или путаницу. | Весь контекст сессии теряется. Всё, что Claude должен помнить между сессиями, нужно хранить в постоянном месте, например в файле `CLAUDE.md`. |
| **Передача субагентам (Subagent Handoffs)** | Запускает субагента в собственном изолированном контекстном окне только с тем описанием задачи и системным промптом, которые ему нужны. Субагент выполняет работу и возвращает краткое изложение. | Когда подзадача достаточно самостоятельна, чтобы её делегировать, особенно исследовательская работа, где сам процесс засоряет основной контекст, а ответ получается коротким. | Видимость того, как субагент пришёл к своему выводу. Промежуточные шаги отбрасываются вместе с контекстом субагента. |

## Two more levers: prompt caching and token counting
## Ещё два рычага: кэширование промптов и подсчёт токенов

The four strategies above manage what enters the context window. Two API features reduce what you pay for what's already there.

> Четыре стратегии выше управляют тем, что попадает в контекстное окно. Две функции API снижают плату за то, что в нём уже находится.

Prompt caching stores the processing work done on a stable prefix of your request so follow-up requests can reuse it instead of reprocessing the same tokens. The first request writes the prefix to cache; subsequent requests that send identical content up to that point pay a fraction of the original cost. The strongest candidates are parts of the request that rarely change across turns: a long system prompt, a large tool definition set, or a reference document you query repeatedly. You enable caching by marking a cache breakpoint with a `cache_control` field of type `ephemeral` on the last block you want cached. You can place up to four breakpoints. For multi-turn sessions with a stable system prompt and tool schemas, caching those prefixes once and reusing them across turns is the highest-leverage cost reduction available.

> Кэширование промптов (prompt caching) сохраняет результат обработки стабильного префикса вашего запроса, чтобы последующие запросы могли использовать его повторно, а не обрабатывать те же токены заново. Первый запрос записывает префикс в кэш; последующие запросы, отправляющие идентичное содержимое до этой точки, платят лишь часть исходной стоимости. Лучшие кандидаты — части запроса, которые редко меняются от хода к ходу: длинный системный промпт, большой набор определений инструментов или справочный документ, к которому вы обращаетесь многократно. Кэширование включается установкой точки кэширования (cache breakpoint) с помощью поля `cache_control` с типом `ephemeral` на последнем блоке, который нужно закэшировать. Можно установить до четырёх таких точек. Для многоходовых сессий со стабильным системным промптом и схемами инструментов однократное кэширование этих префиксов и их повторное использование от хода к ходу — самый эффективный из доступных способов снизить стоимость.

Token counting lets you measure context pressure before a request goes out rather than after it fails. The `count_tokens` endpoint takes the same request body as a messages call and returns the token count without running inference. Use it during development to verify your context budget assumptions hold against real tool outputs, not just test fixtures, and in production to gate requests that would exceed the window before they error.

> Подсчёт токенов (token counting) позволяет измерить нагрузку на контекст до отправки запроса, а не после его сбоя. Эндпоинт `count_tokens` принимает то же тело запроса, что и вызов messages, и возвращает число токенов без запуска инференса. Используйте его при разработке, чтобы проверить, что ваши предположения о бюджете контекста выполняются на реальном выводе инструментов, а не только на тестовых данных, а в продакшене — чтобы заранее блокировать запросы, которые превысили бы окно, до того как они завершатся ошибкой.

## The three places a RAG path can break
## Три места, где может сломаться путь RAG

The path has three places where it can go wrong: the chunking, the embedding match, and the assembly into the prompt.

- **Chunking** decides what a unit of retrievable context is. Split too small and a single chunk lacks the surrounding context to be useful. Split too large and one chunk dilutes the match with unrelated text. Sentence-based or section-based chunking with a little overlap is a reasonable default. The overlap matters because facts that cross a boundary would otherwise be split apart and become difficult to retrieve.
- **The embedding match** decides which chunks are returned. It uses a similarity search, so it retrieves content that is semantically close. This is not always what contains the exact term you need. A query for a specific identifier can miss the relevant chunk if a more semantically similar result outranks it. This is why a lexical match is sometimes run alongside the semantic one.
- **The assembly step** is where retrieved chunks must reach the model in the structure the prompt expects, otherwise the model answers from memory instead of from the retrieved text.

> У этого пути есть три места, где что-то может пойти не так: разбиение на фрагменты (chunking), сопоставление эмбеддингов (embedding match) и сборка в промпт.
>
> - **Разбиение на фрагменты (chunking)** определяет, что является единицей извлекаемого контекста. Если разбить слишком мелко, отдельному фрагменту не хватит окружающего контекста, чтобы быть полезным. Если разбить слишком крупно, один фрагмент размывает совпадение посторонним текстом. Разумный вариант по умолчанию — разбиение по предложениям или разделам с небольшим перекрытием (overlap). Перекрытие важно, потому что иначе факты, попадающие на границу, оказались бы разделены, и их стало бы трудно извлечь.
> - **Сопоставление эмбеддингов (embedding match)** определяет, какие фрагменты будут возвращены. Оно использует поиск по сходству (similarity search), поэтому извлекает семантически близкое содержимое. Это не всегда то, что содержит нужный вам точный термин. Запрос по конкретному идентификатору может пропустить релевантный фрагмент, если более семантически похожий результат окажется выше в ранжировании. Поэтому иногда наряду с семантическим поиском выполняется и лексический.
> - **Этап сборки (assembly)** — это место, где извлечённые фрагменты должны попасть к модели в той структуре, которую ожидает промпт; иначе модель отвечает по памяти, а не на основе извлечённого текста.

The fetch-once path gives you a system you can reason about: you can inspect which chunks were retrieved for a query and test that retrieval directly. The cost is the infrastructure: the index that must be built, stored, kept in sync as the corpus changes, and secured wherever it lives. The search-across-rounds path removes that infrastructure and the staleness that comes with it, since the model reads the current files at query time, at the cost of spending more tokens and time per query and giving you a less inspectable process. For a stable reference corpus queried with simple lookups, the index is worth owning. For a changing corpus or multi-step questions, the iterative search is usually the simpler system despite costing more per query.

> Путь с однократным извлечением (fetch-once) даёт систему, поведение которой можно анализировать: вы можете посмотреть, какие фрагменты были извлечены для запроса, и напрямую протестировать это извлечение. Цена — инфраструктура: индекс, который нужно построить, хранить, синхронизировать при изменении корпуса и защищать, где бы он ни находился. Путь с поиском в несколько раундов (search-across-rounds) убирает эту инфраструктуру и связанную с ней проблему устаревания данных, поскольку модель читает актуальные файлы в момент запроса, но ценой большего расхода токенов и времени на каждый запрос и менее прозрачного процесса. Для стабильного справочного корпуса, к которому обращаются с простыми запросами, индекс стоит того, чтобы им владеть. Для меняющегося корпуса или многошаговых вопросов итеративный поиск обычно оказывается более простой системой, несмотря на более высокую стоимость одного запроса.

The reported performance gain for single-agent agentic search over a retrieval index is a version-pinned figure. Confirm it against the reference layer at build time rather than relying on the number in this module.

> Заявленный выигрыш в производительности агентного поиска одним агентом (single-agent agentic search) по сравнению с поисковым индексом — это показатель, привязанный к конкретной версии. Проверяйте его по справочным материалам на этапе разработки, а не полагайтесь на цифру из этого модуля.

Now, let's understand a bit about two of the most common strategies: compaction and subagent handoffs.

> Теперь разберём подробнее две самые распространённые стратегии: сжатие (compaction) и передачу субагентам (subagent handoffs).

## Applying compaction: What gets preserved depends on how you write the summarizer
## Применение сжатия: что сохранится, зависит от того, как написан суммаризатор

When you use `/compact` in Claude Code, the tool decides what to include in the summary. In the API, the documented primary strategy is server-side compaction (beta): the platform summarizes the conversation for you when it is configured on the request. When you instead implement manual compaction in an API session, you write the summarizer prompt yourself. That prompt determines what the agent will know in subsequent turns.

> Когда вы используете `/compact` в Claude Code, инструмент сам решает, что включить в краткое изложение. В API основная задокументированная стратегия — серверное сжатие (server-side compaction, бета): платформа сама суммаризирует диалог, если это настроено в запросе. Если же вы реализуете ручное сжатие в сессии API, промпт суммаризатора вы пишете сами. Именно этот промпт определяет, что агент будет знать в последующих ходах.

**❌ Summarizer prompt / Промпт суммаризатора:**

```text
"summarize the conversation so far"
```

> *Перевод:* «кратко изложи диалог на текущий момент»

Produces a general summary that may drop task-critical state, which files were modified, what decision was made at a branch point, and what error was encountered and resolved.

> Даёт общее краткое изложение, в котором может потеряться критически важное для задачи состояние: какие файлы были изменены, какое решение было принято в точке ветвления и какая ошибка возникла и была устранена.

**✅ Summarizer prompt / Промпт суммаризатора:**

```text
"summarize the conversation, preserving all file paths modified, all decisions made, and any errors encountered and their resolutions"
```

> *Перевод:* «кратко изложи диалог, сохранив все пути изменённых файлов, все принятые решения, а также все возникшие ошибки и способы их устранения»

Produces a summary the agent can use.

> Даёт краткое изложение, которым агент действительно может пользоваться.

This is not an edge case; task-critical state loss from an under-specified summarizer is one of the most common sources of multi-session agent failures.

> Это не редкий пограничный случай: потеря критически важного для задачи состояния из-за недостаточно конкретного суммаризатора — один из самых частых источников сбоев агентов, работающих в нескольких сессиях.

## Subagent handoffs: Managing long-horizon tasks
## Передача субагентам: управление долгосрочными задачами

When a task is too large for a single context window, increasing the window is not a solution. The solution is to decompose the task and pass only the relevant context to each subagent. A subagent receives a scoped task and the minimum context it needs, the results of prior steps that are directly relevant, the tools it needs to complete its task, and clear exit conditions. The parent agent collects the results. This pattern keeps per-turn cost low and makes long-horizon tasks tractable.

> Когда задача слишком велика для одного контекстного окна, увеличение окна — не решение. Решение — декомпозировать задачу и передавать каждому субагенту только релевантный контекст. Субагент получает задачу с чётко очерченными рамками и минимально необходимый контекст: непосредственно относящиеся к делу результаты предыдущих шагов, инструменты, нужные для выполнения задачи, и чёткие условия завершения. Родительский агент собирает результаты. Этот паттерн сохраняет низкую стоимость каждого хода и делает долгосрочные задачи выполнимыми.

Like compaction and pruning, subagent handoffs add implementation overhead, so apply them only where context cost is a real constraint: a simple single-turn prompt or short workflow doesn't need this.

> Как и сжатие с обрезкой, передача субагентам добавляет затрат на реализацию, поэтому применяйте её только там, где стоимость контекста — реальное ограничение: простому одноходовому промпту или короткому процессу это не нужно.

| | |
|---|---|
| **Handles well** | Multi-step agent sessions that exceed the token budget and need decomposition. Best designed at the architecture stage rather than patched in as a production fix. |
| **Use a different approach** | Pipelines that never approach the window limit. Measure actual token usage against your model's context limit before adding management overhead. |

| | |
|---|---|
| **Хорошо подходит** | Многошаговые агентные сессии, которые превышают бюджет токенов и требуют декомпозиции. Лучше закладывать это на этапе проектирования архитектуры, а не добавлять как заплатку в продакшене. |
| **Используйте другой подход** | Пайплайны, которые никогда не приближаются к лимиту окна. Прежде чем добавлять накладные расходы на управление контекстом, измерьте фактическое использование токенов относительно лимита контекста вашей модели. |

The strategies covered so far assume you know your context budget is under pressure and you are choosing a tool to manage it. The critical point here is not to know the pressure exists until the session breaks. A workload can pass every test in development and then fail in production for one reason: the tool output got bigger, the sessions got longer, and the context window that held twenty turns cleanly now fills at turn eight. The next section walks through exactly how that happens, using a worked postmortem of an agent that ran fine on test fixtures and then hit its ceiling once real documents started flowing through it.

> Рассмотренные до сих пор стратегии предполагают, что вы знаете о нагрузке на бюджет контекста и выбираете инструмент для управления ею. Критическая опасность в том, чтобы не знать о существовании этой нагрузки, пока сессия не сломается. Нагрузка может пройти все тесты при разработке, а затем дать сбой в продакшене по одной причине: вывод инструментов стал больше, сессии стали длиннее, и контекстное окно, которое спокойно вмещало двадцать ходов, теперь заполняется на восьмом. В следующем разделе подробно разбирается, как это происходит, на примере разбора инцидента (postmortem) с агентом, который отлично работал на тестовых данных, а затем упёрся в потолок, как только через него пошли реальные документы.
