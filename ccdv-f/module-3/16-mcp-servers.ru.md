# Building and configuring an MCP server: transport, scope, and the GitHub server
# Создание и настройка MCP-сервера: транспорт, область действия и сервер GitHub

Prior sections introduced plugins as the packaging layer that bundles skills, hooks, subagents, and MCP servers into a single installable unit.

> В предыдущих разделах плагины были представлены как слой упаковки, который объединяет навыки (skills), хуки (hooks), субагентов (subagents) и MCP-серверы в единый устанавливаемый пакет.

This section further explains what MCP server bundles are and how to build them. An MCP server is the layer that exposes tools to Claude from outside your codebase. When building an MCP server, one of the first decisions is determining the appropriate transport mechanism and defining the server's scope.

> В этом разделе подробнее объясняется, что входит в MCP-серверы и как их создавать. MCP-сервер — это слой, который предоставляет Claude инструменты извне вашей кодовой базы. При создании MCP-сервера одно из первых решений — выбрать подходящий механизм транспорта (transport) и определить область действия (scope) сервера.

## What is an MCP server and why is it different from wiring a tool directly?
## Что такое MCP-сервер и чем он отличается от прямого подключения инструмента?

When wiring a tool directly into an application, you are responsible for defining the tool's schema and its functionality. Both live in that application's code. If three different applications need access to the same external service, each one maintains its own integration. Model Context Protocol, or MCP, separates tool definitions from individual applications and turns them into a process called a server.

> При прямом подключении инструмента к приложению вы сами отвечаете за определение схемы инструмента и его функциональности. И то и другое находится в коде этого приложения. Если трём разным приложениям нужен доступ к одному и тому же внешнему сервису, каждое поддерживает собственную интеграцию. Model Context Protocol (MCP) отделяет определения инструментов от отдельных приложений и выносит их в процесс, который называется сервером.

An MCP server is a process that exposes tools, resources, and prompts that MCP clients can use. Claude Code has a built-in MCP client. When you connect to an MCP server, Claude Code discovers the tools it provides and can invoke them during a session. With an MCP server, you build the capability once, and every MCP client that connects to it gets access without re-implementing the integration.

> MCP-сервер — это процесс, который предоставляет инструменты (tools), ресурсы (resources) и промпты (prompts), доступные MCP-клиентам. В Claude Code встроен MCP-клиент. Когда вы подключаетесь к MCP-серверу, Claude Code обнаруживает предоставляемые им инструменты и может вызывать их во время сессии. С MCP-сервером вы создаёте возможность один раз, и каждый подключившийся MCP-клиент получает к ней доступ без повторной реализации интеграции.

## MCP servers also expose resources and prompts
## MCP-серверы также предоставляют ресурсы и промпты

An MCP server exposes tools, resources, and prompts. We've already learned about tools: actions the model can call. The other two cover cases where a tool call won't give you what you need.

> MCP-сервер предоставляет инструменты, ресурсы и промпты. Об инструментах мы уже знаем: это действия, которые может вызывать модель. Два других элемента покрывают случаи, когда вызов инструмента не даёт того, что нужно.

A resource is read-only data the server exposes for the client to fetch and place into context directly, rather than the model calling a tool to get it. The client requests a resource by its address, and the server returns the data. Resources come in two forms: a direct resource has a fixed address for data that takes no parameters, such as a list of available documents, and a templated resource puts a parameter in the address, such as a document address that takes a document identifier. Reach for a resource when you want known data to be in context from the start of a turn. You want this when pulling a resource in directly is cheaper and more predictable than using a tool call to go get it. Resource support varies across MCP clients; verify that your client has a mechanism to inject resources into context before relying on this pattern.

> Ресурс (resource) — это данные только для чтения, которые сервер предоставляет, чтобы клиент мог получить их и поместить прямо в контекст, а не модель вызывала инструмент, чтобы их достать. Клиент запрашивает ресурс по его адресу, и сервер возвращает данные. Ресурсы бывают двух видов: прямой ресурс (direct resource) имеет фиксированный адрес для данных без параметров, например списка доступных документов, а шаблонный ресурс (templated resource) содержит параметр в адресе, например адрес документа, принимающий идентификатор документа. Используйте ресурс, когда нужно, чтобы известные данные были в контексте с самого начала хода (turn). Это нужно, когда прямая подгрузка ресурса дешевле и предсказуемее, чем получение тех же данных через вызов инструмента. Поддержка ресурсов различается в разных MCP-клиентах; прежде чем полагаться на этот подход, убедитесь, что ваш клиент умеет добавлять ресурсы в контекст.

A prompt is a pre-written instruction template the server exposes so a client can invoke a vetted prompt by name instead of asking each user to write their own. A user can already ask the model to do most tasks in their own words, so a prompt is useful when specific wording is needed: a task where a carefully built instruction produces materially better results than whatever a user would type, and where you want every client to get the same quality. Packaging the instruction on the server means the prompt is maintained in one place and reused everywhere the server is connected.

> Промпт (prompt) — это заранее написанный шаблон инструкции, который сервер предоставляет, чтобы клиент мог вызвать проверенный промпт по имени, а не просить каждого пользователя писать свой. Большинство задач пользователь и так может поставить модели своими словами, поэтому промпт полезен, когда нужна конкретная формулировка: для задачи, где тщательно составленная инструкция даёт заметно лучший результат, чем то, что напечатал бы пользователь, и где нужно, чтобы каждый клиент получал одинаковое качество. Если инструкция упакована на сервере, промпт поддерживается в одном месте и повторно используется везде, где подключён сервер.

## Transport: how Claude Code talks to the server
## Транспорт: как Claude Code общается с сервером

Transport is the communication channel between the MCP client and the MCP server. The right transport depends on where the server runs. Select each tab for what it is and when to use it.

> Транспорт (transport) — это канал связи между MCP-клиентом и MCP-сервером. Правильный выбор транспорта зависит от того, где работает сервер. Выберите каждую вкладку, чтобы узнать, что это такое и когда его использовать.

### `stdio`

`stdio` runs the server as a local process on the same machine as the client. The client launches the server as a subprocess and communicates through standard input and output. This is the correct choice for a local tool, a personal script, or a development server you run on your own machine. It does not work for a server you want to share across your team or host remotely.

> `stdio` запускает сервер как локальный процесс на той же машине, что и клиент. Клиент запускает сервер как подпроцесс и общается с ним через стандартный ввод и вывод. Это правильный выбор для локального инструмента, личного скрипта или сервера разработки, который вы запускаете на своей машине. Он не подходит для сервера, который нужно сделать общим для команды или разместить удалённо.

### HTTP

HTTP is the recommended form of transport for any server that does not run locally. It connects over a standard HTTP connection and supports servers hosted on a different machine. When you register an HTTP server, you provide the URL and the client connects over the network. Shared team servers and hosted integrations use HTTP.

> HTTP — рекомендуемый вид транспорта для любого сервера, который работает не локально. Он подключается через стандартное HTTP-соединение и поддерживает серверы, размещённые на другой машине. При регистрации HTTP-сервера вы указываете URL, и клиент подключается по сети. Общие серверы команды и размещённые (hosted) интеграции используют HTTP.

### SSE

SSE (Server-Sent Events) is an older means of transport that predates the current HTTP transport. It has been superseded by HTTP transport and is no longer recommended for new servers. If you encounter SSE in existing configuration or documentation, treat it as a legacy option rather than a current recommendation.

> SSE (Server-Sent Events) — более старый вид транспорта, появившийся раньше текущего HTTP-транспорта. Он вытеснен HTTP-транспортом и больше не рекомендуется для новых серверов. Если вы встречаете SSE в существующей конфигурации или документации, считайте его устаревшим (legacy) вариантом, а не актуальной рекомендацией.

## Context cost
## Стоимость в контексте

Each connected MCP server contributes tool definitions that would occupy the context window if loaded upfront. By default, Claude Code defers these definitions rather than loading them upfront, and it uses a search step to discover and load the relevant tools only when a task calls for them. Only the tools called for enter context.

> Каждый подключённый MCP-сервер добавляет определения инструментов, которые занимали бы место в контекстном окне (context window), если бы загружались заранее. По умолчанию Claude Code откладывает загрузку этих определений и с помощью шага поиска находит и загружает нужные инструменты, только когда они требуются задаче. В контекст попадают только востребованные инструменты.

An opt-in mode loads tool definitions upfront when they fit within roughly 10 percent of the context window, deferring only when that limit is exceeded. Either way, connecting only the servers you need keeps each request lean, because every connected server adds to the pool of definitions the model has to account for.

> Включаемый по желанию (opt-in) режим загружает определения инструментов заранее, если они укладываются примерно в 10 процентов контекстного окна, и откладывает загрузку, только когда этот предел превышен. В любом случае подключение только нужных серверов сохраняет каждый запрос компактным, потому что каждый подключённый сервер пополняет набор определений, которые модели приходится учитывать.

## Prompt caching: paying once for reusable requests
## Кэширование промптов: платить один раз за повторно используемые запросы

The context-cost problem you just saw with MCP servers has both a cost and window dimension. Every request reprocesses its input from scratch, including the parts that were identical on the last request, meaning that you pay for reprocessing each time. Prompt caching can stop you from paying twice for the same stable content.

> У проблемы стоимости контекста, которую вы только что увидели на примере MCP-серверов, есть два измерения: денежная стоимость и размер окна. Каждый запрос обрабатывает свои входные данные заново, включая части, которые были идентичны в прошлом запросе, то есть вы каждый раз платите за повторную обработку. Кэширование промптов (prompt caching) позволяет не платить дважды за одно и то же неизменное содержимое.

Caching stores the processing work done on a stable prefix of your request so a follow-up request can reuse it instead of reprocessing the same tokens. The first request writes the prefix to the cache, and follow-up requests send identical content up to the same point in the cache at a fraction of the cost. The content must match exactly: a single changed character before the cache point invalidates that cache and forces a fresh write. That is why the strongest candidates for caches are the parts of a request that rarely change, such as a long system prompt, a large set of tool definitions, or a reference document you ask several questions about.

> Кэширование сохраняет результат обработки неизменного префикса запроса, чтобы следующий запрос мог использовать его повторно, а не обрабатывать те же токены заново. Первый запрос записывает префикс в кэш, а последующие запросы отправляют идентичное содержимое до той же точки кэша за малую долю стоимости. Содержимое должно совпадать в точности: один изменённый символ до точки кэширования делает этот кэш недействительным и вызывает новую запись. Поэтому лучшие кандидаты для кэширования — редко меняющиеся части запроса: длинный системный промпт, большой набор определений инструментов или справочный документ, по которому вы задаёте несколько вопросов.

You turn on caching by marking a cache breakpoint; there is no global setting that turns caching on. In the Messages API you add a `cache_control` field of type `ephemeral` to the last block you want cached; this caches everything up to and including that block. You can place up to four breakpoints. The request is processed in a fixed order of tools, system prompt, and messages, so a breakpoint after the tools caches the tool definitions while keeping the messages dynamic.

> Кэширование включается установкой точки кэширования (cache breakpoint); глобальной настройки, включающей кэширование, нет. В Messages API вы добавляете поле `cache_control` с типом `ephemeral` к последнему блоку, который нужно закэшировать; при этом кэшируется всё до этого блока включительно. Можно поставить до четырёх точек кэширования. Запрос обрабатывается в фиксированном порядке: инструменты, системный промпт, сообщения, поэтому точка кэширования после инструментов кэширует определения инструментов, а сообщения остаются динамическими.

The cache has a time limit. The default cache lifetime is five minutes from the last read. An opt-in one-hour lifetime is available by setting a `ttl` of `1h` on the breakpoint. The five-minute default suits a back-and-forth model where requests arrive every few minutes, since each read resets the clock. The one-hour option suits a workload with longer gaps between requests, such as an agent that pauses between steps, where the five-minute window would expire before the next request. If the window expires before the next request, you are left paying the write cost again for no read benefit. Please note that caching only applies above a minimum token threshold (1,024 tokens for most current models) so short prompts will not be cached even if a breakpoint is set.

> У кэша есть ограничение по времени. По умолчанию время жизни кэша — пять минут с последнего чтения. Можно включить время жизни в один час, задав `ttl` со значением `1h` для точки кэширования. Пятиминутный вариант по умолчанию подходит для режима активного диалога, где запросы приходят каждые несколько минут, поскольку каждое чтение сбрасывает таймер. Часовой вариант подходит для нагрузки с более длинными паузами между запросами, например для агента, который делает паузы между шагами, — там пятиминутное окно истекло бы до следующего запроса. Если окно истекает до следующего запроса, вы снова платите за запись, не получив выгоды от чтения. Обратите внимание: кэширование применяется только выше минимального порога токенов (1 024 токена для большинства текущих моделей), поэтому короткие промпты не кэшируются, даже если точка кэширования задана.

## Retrieval-augmented generation: how Claude pulls in only the knowledge a request needs
## Генерация с дополнением извлечёнными данными: как Claude подгружает только те знания, которые нужны запросу

The context-cost problem you just saw with MCP servers is the same one a large body of reference material creates. A model reads everything in its context window for every request, so the more documents you load up front, the more context is used, and the less room is left for the work. Retrieval-augmented generation, usually shortened to RAG, is the pattern that resolves this. Instead of loading every document into context, the system stores the material outside the context window, finds the parts most relevant to the current request, and supplies only those parts to the model at request time. The model then generates its answer from that retrieved slice rather than from the whole library.

> Проблема стоимости контекста, которую вы только что увидели на примере MCP-серверов, — та же, что возникает при большом объёме справочных материалов. Модель читает всё содержимое контекстного окна при каждом запросе, поэтому чем больше документов вы загружаете заранее, тем больше контекста расходуется и тем меньше места остаётся для работы. Генерация с дополнением извлечёнными данными (retrieval-augmented generation, обычно сокращённо RAG) — подход, который решает эту проблему. Вместо загрузки всех документов в контекст система хранит материал вне контекстного окна, находит части, наиболее релевантные текущему запросу, и передаёт модели только их в момент запроса. Затем модель формирует ответ на основе этого извлечённого фрагмента, а не всей библиотеки.

RAG comes in two forms:

> RAG бывает двух видов:

- **Classical RAG** does the hard work upfront. Before anyone asks a question, the source material gets split into chunks, and each chunk is converted into a set of numbers (called an embedding) that captures its meaning mathematically. Those numbers are stored in a database. When a user asks a question, the system converts the question into the same type of numbers, then finds whichever chunks have the most similar numbers. Think of it like a librarian who, before the library opens, has already read every book and written a precise summary card for every chapter, so when you arrive with a question, they can pull the right cards instantly.
- **Agentic search** skips the upfront indexing entirely. There's no pre-built database. Instead, the model figures out what it needs the moment you ask, then goes and fetches it: searching live sources, reading documents on demand, pulling in results as the task unfolds. Think of it like a researcher who, when you ask a question, goes and finds the answer themselves rather than consulting pre-prepared cards.

> - **Классический RAG (classical RAG)** выполняет основную работу заранее. Ещё до того, как кто-либо задаст вопрос, исходный материал разбивается на фрагменты (chunks), и каждый фрагмент преобразуется в набор чисел (так называемое векторное представление, embedding), который математически отражает его смысл. Эти числа хранятся в базе данных. Когда пользователь задаёт вопрос, система преобразует вопрос в числа того же типа и находит фрагменты с наиболее похожими числами. Представьте библиотекаря, который ещё до открытия библиотеки прочитал все книги и написал точную карточку-аннотацию для каждой главы, поэтому, когда вы приходите с вопросом, он мгновенно достаёт нужные карточки.
> - **Агентный поиск (agentic search)** полностью пропускает предварительное индексирование. Заранее построенной базы данных нет. Вместо этого модель в момент вашего вопроса определяет, что ей нужно, и идёт за этим: ищет в актуальных источниках, читает документы по требованию, подтягивает результаты по ходу выполнения задачи. Представьте исследователя, который на ваш вопрос сам идёт и находит ответ, а не сверяется с заранее подготовленными карточками.

You may have already encountered agentic search without knowing its name. In Claude Code, when you're connected to many external tools (MCP servers), Claude doesn't load every tool definition up front; that would be too much to hold at once. Instead, it discovers and loads only the tools it needs for the current task. Claude.ai Projects works the same way for uploaded documents: when a project's knowledge base grows larger than can fit in the active window, it surfaces only the document sections most relevant to each question rather than loading everything.

> Возможно, вы уже сталкивались с агентным поиском, не зная его названия. В Claude Code, когда подключено много внешних инструментов (MCP-серверов), Claude не загружает заранее все определения инструментов — удерживать столько сразу было бы слишком много. Вместо этого он находит и загружает только инструменты, нужные для текущей задачи. Проекты (Projects) в Claude.ai работают так же с загруженными документами: когда база знаний проекта становится больше, чем помещается в активное окно, подтягиваются только разделы документов, наиболее релевантные каждому вопросу, а не загружается всё.

Both approaches do the same fundamental thing; they both find a relevant slice of material and generate from it. The difference is timing: classical RAG finds the slice by matching against an index built in advance; agentic search finds it by searching at the moment of need.

> Оба подхода по сути делают одно и то же: находят релевантный фрагмент материала и формируют ответ на его основе. Разница во времени: классический RAG находит фрагмент сопоставлением с заранее построенным индексом, а агентный поиск — поиском в момент, когда это нужно.

Two properties of retrieval are worth understanding before you reach for it:

> Прежде чем использовать извлечение (retrieval), стоит понимать два его свойства:

- **It scales.** As your source material grows, the cost of each request stays flat, because the model only ever receives the slice relevant to that question, not the entire library. A knowledge base can grow to thousands of documents and a single question still pulls back roughly the same amount of text. That's what lets retrieval work at scale: the source can keep growing without the request growing with it.
- **It's only as good as what it finds.** The model reasons over the slice it receives. If the retrieval step misses the document you needed, the model never sees it. This means that how you organize your material is important: files with vague names ("notes_final_v3.pdf") are harder to surface than files with descriptive ones ("Q3 refund policy, updated August 2024"). Grouping related files together helps too. Good retrieval starts with a well-organized source.

> - **Оно масштабируется.** По мере роста исходного материала стоимость каждого запроса остаётся неизменной, потому что модель всегда получает только фрагмент, релевантный данному вопросу, а не всю библиотеку. База знаний может разрастись до тысяч документов, а один вопрос по-прежнему извлекает примерно тот же объём текста. Именно это позволяет извлечению работать в больших масштабах: источник может расти дальше, а запрос вместе с ним не растёт.
> - **Оно хорошо ровно настолько, насколько хорошо то, что найдено.** Модель рассуждает на основе полученного фрагмента. Если на шаге извлечения нужный документ будет пропущен, модель его так и не увидит. Поэтому важно, как организован ваш материал: файлы с расплывчатыми названиями («notes_final_v3.pdf») найти труднее, чем файлы с описательными («Q3 refund policy, updated August 2024» — «Политика возвратов за 3-й квартал, обновлена в августе 2024»). Группировка связанных файлов тоже помогает. Хорошее извлечение начинается с хорошо организованного источника.

## Configuration scope: who loads the server
## Область действия конфигурации: кто загружает сервер

The scope determines which users and projects load the server. Each scope corresponds to a different configuration location.

> Область действия (scope) определяет, какие пользователи и проекты загружают сервер. Каждой области соответствует своё место хранения конфигурации.

- **Local scope** stores the server configuration in `~/.claude.json` under the current project's path. It applies only to the project you are currently working on and is not shared with teammates. This is the right scope for a server tied to a specific project context that you are not ready to commit to the repository, or for tooling that only makes sense in one project.
- **User scope** stores the server configuration in your personal Claude settings and makes it available across all your projects. It is still personal: teammates do not see it, and it is not written into the repository. This is the right scope for a personal utility you use in every project, such as a local database tool or a script you rely on regardless of which codebase you are working on.
- **Project scope** writes the server configuration to a `.mcp.json` file at the root of the repository. When that file is committed to version control, everyone who clones the repository gets the same server automatically. This is the right scope for a server the whole team can access, because the configuration travels with the code. One thing to keep in mind: a project-scoped server runs from each teammate's machine. For a `stdio` server, the committed configuration stores the launch command, and every clone spawns its own local subprocess, so each teammate needs the runtime (such as Node for an `npx`-launched server) installed locally.
- **Enterprise scope** deploys through a centrally managed configuration controlled by an administrator. Administrators can push servers to all users in the organization without individual configuration steps. This is the right scope for shared internal services, security tooling, or any server that must be present across the organization and cannot be left to individual developers to configure.

> - **Локальная область (local scope)** хранит конфигурацию сервера в `~/.claude.json` под путём текущего проекта. Она действует только для проекта, с которым вы сейчас работаете, и не передаётся коллегам. Это подходящая область для сервера, привязанного к контексту конкретного проекта, который вы пока не готовы фиксировать в репозитории, или для инструментов, имеющих смысл только в одном проекте.
> - **Пользовательская область (user scope)** хранит конфигурацию сервера в ваших личных настройках Claude и делает его доступным во всех ваших проектах. Она по-прежнему личная: коллеги её не видят, и она не записывается в репозиторий. Это подходящая область для личной утилиты, которой вы пользуетесь в каждом проекте, например локального инструмента для работы с базой данных или скрипта, на который вы полагаетесь независимо от кодовой базы.
> - **Проектная область (project scope)** записывает конфигурацию сервера в файл `.mcp.json` в корне репозитория. Когда этот файл зафиксирован в системе контроля версий, каждый, кто клонирует репозиторий, автоматически получает тот же сервер. Это подходящая область для сервера, к которому должна иметь доступ вся команда, потому что конфигурация распространяется вместе с кодом. Важно помнить: сервер с проектной областью запускается на машине каждого коллеги. Для сервера `stdio` зафиксированная конфигурация хранит команду запуска, и каждый клон запускает собственный локальный подпроцесс, поэтому у каждого коллеги локально должна быть установлена среда выполнения (например, Node для сервера, запускаемого через `npx`).
> - **Корпоративная область (enterprise scope)** развёртывается через централизованно управляемую конфигурацию, которую контролирует администратор. Администраторы могут распространять серверы на всех пользователей организации без индивидуальной настройки. Это подходящая область для общих внутренних сервисов, инструментов безопасности или любого сервера, который должен присутствовать во всей организации и настройку которого нельзя оставлять отдельным разработчикам.

## Permission rules that target a single MCP tool, not the whole server
## Правила разрешений для отдельного MCP-инструмента, а не всего сервера

Connecting a server exposes its full tool list, but you rarely want the agent to reach every one of those tools without checking. The permission layer from the permission-modes section extends to MCP tools, and the rules can name an individual tool rather than the whole server.

> Подключение сервера открывает весь его список инструментов, но редко нужно, чтобы агент мог без проверки использовать каждый из них. Слой разрешений из раздела о режимах разрешений распространяется и на MCP-инструменты, а правила могут указывать отдельный инструмент, а не весь сервер.

An MCP tool is identified in a permission rule by its server and tool name: `mcp__server__tool`. An allow rule on `mcp__github__create_issue` lets that one tool run without a prompt while every other tool on the GitHub server still prompts. A deny rule on a write-capable tool blocks it while read-only tools on the same server stay available. This is how you connect a broad server but keep the agent inside a narrow slice of what it can do. A deny on one tool overrides an allow on the server.

> В правиле разрешений MCP-инструмент обозначается именем сервера и именем инструмента: `mcp__server__tool`. Разрешающее правило (allow) для `mcp__github__create_issue` позволяет этому одному инструменту работать без запроса подтверждения, а все остальные инструменты сервера GitHub по-прежнему запрашивают его. Запрещающее правило (deny) для инструмента с правом записи блокирует его, а инструменты того же сервера, работающие только на чтение, остаются доступными. Так можно подключить сервер с широкими возможностями, но удерживать агента в узкой части того, что он может делать. Запрет для одного инструмента имеет приоритет над разрешением для сервера.

The API MCP connector is another useful control. If you are reaching the server through the API MCP connector, an `mcp_toolset` object lets you set an `enabled` flag per tool. This `enabled` flag lets you register a server but expose only the specific tools you want the model to see. A permission rule decides whether an exposed tool may run; the `enabled` flag decides whether the model sees the tool at all. The first is a governance control, the second is a context-cost and scope control. These controls are often used together. Always verify the exact rule syntax and the connector beta header against the documentation before publishing.

> Ещё один полезный механизм контроля — MCP-коннектор API (API MCP connector). Если вы обращаетесь к серверу через MCP-коннектор API, объект `mcp_toolset` позволяет задать флаг `enabled` для каждого инструмента. Флаг `enabled` позволяет зарегистрировать сервер, но открыть только те инструменты, которые должна видеть модель. Правило разрешений определяет, может ли открытый инструмент выполняться; флаг `enabled` определяет, видит ли модель инструмент вообще. Первое — механизм управления (governance), второе — механизм контроля стоимости контекста и области действия. Их часто используют вместе. Перед публикацией всегда сверяйте точный синтаксис правил и бета-заголовок коннектора с документацией.

## The GitHub MCP server: transport, scope, and authentication in a concrete example
## MCP-сервер GitHub: транспорт, область действия и аутентификация на конкретном примере

The GitHub MCP server is a remote server maintained by GitHub that exposes tools for repository management including reviewing pull requests, opening issues, searching code, and more. By walking through the connection process, you can see how transport, scope, and authentication work together in a server maintained by someone else.

> MCP-сервер GitHub — это удалённый сервер, поддерживаемый GitHub, который предоставляет инструменты для управления репозиториями: просмотр pull request'ов, создание issues, поиск по коду и многое другое. Пройдя процесс подключения, можно увидеть, как транспорт, область действия и аутентификация работают вместе на сервере, который поддерживает кто-то другой.

The GitHub server uses HTTP transport because it is hosted remotely by GitHub. You register it by providing the server URL, and the client connects over the network. For scope, choose project scope when your whole team needs access to the same repository tooling, and local scope when only you need access to the server.

> Сервер GitHub использует HTTP-транспорт, потому что он размещён удалённо у GitHub. Вы регистрируете его, указав URL сервера, и клиент подключается по сети. В качестве области действия выбирайте проектную, если всей команде нужен доступ к одним и тем же инструментам для работы с репозиторием, и локальную, если доступ к серверу нужен только вам.

Authentication for the GitHub MCP server uses a Personal Access Token. You generate the token in GitHub, then pass it as a Bearer token in the request header of your MCP configuration. The token must be supplied through an environment variable and referenced in the configuration file. It must not be committed inline to `.mcp.json`, because a token written directly into a committed file enters repository history and cannot be removed by overwriting the file in a later commit.

> Аутентификация на MCP-сервере GitHub использует персональный токен доступа (Personal Access Token). Вы создаёте токен в GitHub, а затем передаёте его как Bearer-токен в заголовке запроса в вашей MCP-конфигурации. Токен должен передаваться через переменную окружения, на которую ссылается файл конфигурации. Его нельзя фиксировать прямо в `.mcp.json`, потому что токен, записанный непосредственно в зафиксированный файл, попадает в историю репозитория и не удаляется перезаписью файла в последующем коммите.

OAuth is a different authentication mechanism, used by servers where the service authenticates individual users through a browser-based sign-in flow. Linear is an example of a server that uses this pattern. When you connect to a Linear MCP server for the first time, the client redirects to Linear's sign-in page. After you approve access, a token is issued and stored automatically. No credential is copied or managed by hand. OAuth is the right pattern for any integration where the service's authorization model is tied to user identity.

> OAuth — другой механизм аутентификации, который используют серверы, где сервис аутентифицирует отдельных пользователей через вход в браузере. Пример сервера с таким подходом — Linear. При первом подключении к MCP-серверу Linear клиент перенаправляет вас на страницу входа Linear. После того как вы одобрите доступ, токен выдаётся и сохраняется автоматически. Никакие учётные данные не копируются и не обрабатываются вручную. OAuth — правильный подход для любой интеграции, где модель авторизации сервиса привязана к личности пользователя.

GitHub MCP uses a service credential you generate and store; Linear MCP initiates a sign-in flow that handles the credential for you. Both are remote HTTP servers, and both follow the same transport and scope logic. The authentication step is what differs.

> GitHub MCP использует учётные данные сервиса, которые вы создаёте и храните сами; Linear MCP запускает процесс входа, который сам обрабатывает учётные данные за вас. Оба — удалённые HTTP-серверы, и оба следуют одной и той же логике транспорта и области действия. Различается только шаг аутентификации.

## The MCP setup reference
## Справочник по настройке MCP

The table below captures transport and scope decisions for each deployment context.

> В таблице ниже собраны решения о транспорте и области действия для каждого контекста развёртывания.

| Context | Transport | Scope | Config location | Secrets handling |
|---|---|---|---|---|
| Personal local tool (runs on your machine only) | `stdio` | Local | `~/.claude.json` (per-project entry) | Environment variables only. Never in config file. |
| Shared team server (all teammates connect to same service) | HTTP | Project (`.mcp.json`) | `.mcp.json` committed to repo root | OAuth or env variables. API keys must never be committed to `.mcp.json`. |
| Personal experiment (not ready to share) | `stdio` or HTTP | Local | Personal Claude settings | Environment variables only. |
| Organization-wide deployment (admin-managed) | HTTP | Enterprise | Managed settings (admin-controlled) | Secrets managed by administrator. Config locked to prevent override. |

| Контекст | Транспорт | Область действия | Где хранится конфигурация | Работа с секретами |
|---|---|---|---|---|
| Личный локальный инструмент (работает только на вашей машине) | `stdio` | Локальная (Local) | `~/.claude.json` (запись для конкретного проекта) | Только переменные окружения. Никогда не в файле конфигурации. |
| Общий сервер команды (все коллеги подключаются к одному сервису) | HTTP | Проектная (Project, `.mcp.json`) | `.mcp.json`, зафиксированный в корне репозитория | OAuth или переменные окружения. API-ключи никогда не должны фиксироваться в `.mcp.json`. |
| Личный эксперимент (пока не готов к совместному использованию) | `stdio` или HTTP | Локальная (Local) | Личные настройки Claude | Только переменные окружения. |
| Развёртывание на всю организацию (управляется администратором) | HTTP | Корпоративная (Enterprise) | Управляемые настройки (managed settings, под контролем администратора) | Секретами управляет администратор. Конфигурация заблокирована от переопределения. |

| | |
|---|---|
| **Cost** | Each connected MCP server adds its tool definitions to the context window. The more servers connected, the larger every request. Load only the servers a given task needs. |
| **Complexity** | Transport and scope are independent decisions, but they interact: a `stdio` server cannot be project-scoped for sharing because it runs only on one machine. Match transport to where the server runs before choosing scope. |
| **Risk** | Committing an API key inside `.mcp.json` to version control is the most common mistake in this section. The key travels into repository history where rotating later is not sufficient to remove the exposure. Secrets go in environment variables. The configuration file holds only the server address. |

| | |
|---|---|
| **Стоимость** | Каждый подключённый MCP-сервер добавляет определения своих инструментов в контекстное окно. Чем больше подключено серверов, тем больше каждый запрос. Загружайте только те серверы, которые нужны конкретной задаче. |
| **Сложность** | Транспорт и область действия — независимые решения, но они влияют друг на друга: сервер `stdio` нельзя сделать общим через проектную область, потому что он работает только на одной машине. Прежде чем выбирать область действия, подберите транспорт под то, где работает сервер. |
| **Риск** | Фиксация API-ключа внутри `.mcp.json` в системе контроля версий — самая частая ошибка в этой теме. Ключ попадает в историю репозитория, и последующей ротации ключа недостаточно, чтобы устранить его раскрытие. Секреты хранятся в переменных окружения. Файл конфигурации содержит только адрес сервера. |
