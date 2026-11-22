# Packaging a working build so the next engagement starts from an asset
# Упаковка рабочей сборки, чтобы следующий проект начинался с готового актива

You finished the prior modules with a build that runs: an agent loop, a configured MCP server, an eval that proves that the prompt works. The most time-consuming and expensive thing on a team is the engineering time that gets spent rebuilding the same thing for the next customer.

> Вы завершили предыдущие модули с работающей сборкой: циклом агента, настроенным MCP-сервером и оценкой (eval), которая доказывает, что промпт работает. Самое затратное по времени и деньгам в работе команды — инженерное время, которое уходит на повторное создание того же самого для следующего клиента.

## What an accelerator does: keep the reusable parts and separate out the rest
## Что делает акселератор: сохраняет повторно используемые части и отделяет остальное

An accelerator is a solution packaged so future engagements start from a working foundation rather than from a blank repository. In blueprint terms, this is packaging for reuse: separating engagement-specific code from the reusable core and parameterizing the rest. Take a working build, separate the parts that are customer-specific, and expose them as parameters with documented defaults. The asset then configures rather than gets entirely rewritten. Packaging for reuse while the build is fresh is cheaper than reconstructing the intent months later, when the person who knew why a value was hardcoded has moved on.

> Акселератор (accelerator) — это решение, упакованное так, чтобы будущие проекты начинались с рабочей основы, а не с пустого репозитория. В терминах типового плана (blueprint) это упаковка для повторного использования: отделение кода, специфичного для конкретного проекта, от повторно используемого ядра и параметризация остального. Возьмите рабочую сборку, выделите части, специфичные для клиента, и вынесите их в параметры с задокументированными значениями по умолчанию. Тогда актив настраивается, а не переписывается целиком. Упаковать решение для повторного использования, пока сборка свежая, дешевле, чем восстанавливать замысел через несколько месяцев, когда человек, знавший, почему значение было жёстко прописано, уже ушёл на другой проект.

## Most reusable work falls into different asset types, and each one packages differently
## Большая часть повторно используемой работы относится к разным типам активов, и каждый упаковывается по-своему

Most reusable work falls into one of three categories used throughout this module: a template, a configurable server, or a portable eval. Each type holds a different kind of work and needs to be packaged in its own way. Reaching for the wrong type can make an asset look reusable while still making it difficult to apply.

> Большая часть повторно используемой работы относится к одной из трёх категорий, которые используются на протяжении всего модуля: шаблон (template), настраиваемый сервер (configurable server) или переносимая оценка (portable eval). Каждый тип содержит свой вид работы и требует собственного способа упаковки. Если выбрать не тот тип, актив может выглядеть пригодным для повторного использования, но применять его всё равно будет сложно.

| Asset type | What it bundles | What correct packaging requires |
|---|---|---|
| Agent Template | The system prompt, the tool schemas, and the loop structure from a working agent. | Pull the domain-specific values into configuration with documented defaults, so a new team sets the values rather than editing the loop. |
| MCP Server Package | The tools the server exposes, with their inputs and the scope the installing team controls. | Document each tool input and let the installing team set the scope, so the server installs into a new environment without code edits. |
| Eval Suite | The graded test set and the judge rubric that prove the asset works. | Ship the dataset and rubric together so a new team can run them in their own context and confirm the asset still works there. The same eval suite also acts as the gate at deployment. When you promote a new model version to production, run it against a pinned baseline score before the version goes live. |

| Тип актива | Что в него входит | Что требуется для правильной упаковки |
|---|---|---|
| Шаблон агента (Agent Template) | Системный промпт, схемы инструментов и структура цикла работающего агента. | Вынести значения, специфичные для предметной области, в конфигурацию с задокументированными значениями по умолчанию, чтобы новая команда задавала значения, а не правила цикл. |
| Пакет MCP-сервера (MCP Server Package) | Инструменты, которые предоставляет сервер, с их входными параметрами и областью действия, которую контролирует устанавливающая команда. | Задокументировать входные параметры каждого инструмента и дать устанавливающей команде задать область действия, чтобы сервер устанавливался в новую среду без правки кода. |
| Набор оценок (Eval Suite) | Оцениваемый тестовый набор и рубрика судьи, доказывающие, что актив работает. | Поставлять набор данных и рубрику вместе, чтобы новая команда могла запустить их в своём контексте и убедиться, что актив там по-прежнему работает. Тот же набор оценок служит и контрольной точкой при развёртывании. При переводе новой версии модели в продакшен прогоните его и сравните с зафиксированным базовым баллом до того, как версия начнёт работать. |

Shipping an agent as a set of loose scripts instead of a template is the most common version of the wrong approach. The scripts run, so they look reusable, but every customer-specific value is buried in a different file, and the next team copies and diverges them instead of configuring one asset.

> Самый распространённый вариант неправильного подхода — поставлять агента как набор разрозненных скриптов вместо шаблона. Скрипты работают и поэтому выглядят пригодными для повторного использования, но каждое значение, специфичное для клиента, спрятано в отдельном файле, и следующая команда копирует их и вносит расходящиеся изменения, вместо того чтобы настраивать один актив.

## Document both the code and the assumptions
## Документируйте и код, и допущения

Code describes behavior. Documentation covers what a future builder cannot reliably infer from reading the source: the assumptions the asset makes about its environment, the inputs it expects, the failure modes it already handles, and the eval that defines whether it still works. Without this, the next team treats the asset as a black box and rebuilds it.

> Код описывает поведение. Документация охватывает то, что будущий разработчик не может надёжно понять из исходного кода: допущения актива о своей среде, ожидаемые входные данные, уже обрабатываемые виды сбоев и оценку (eval), которая определяет, работает ли он по-прежнему. Без этого следующая команда воспринимает актив как чёрный ящик и создаёт его заново.

## Bundle the audit log as part of the package
## Включайте журнал аудита в состав пакета

A regulated customer's reviewer asks what data the asset touches, what identity it acts under, and what log it leaves. An accelerator without these passes a demo and stalls at the first security review. Treat the audit log as part of the package.

> Проверяющий со стороны клиента из регулируемой отрасли спрашивает, с какими данными работает актив, от имени какой идентичности он действует и какой журнал оставляет. Акселератор без ответов на эти вопросы проходит демонстрацию, но застревает на первой же проверке безопасности. Считайте журнал аудита частью пакета.

## The packaging checklist
## Контрольный список упаковки

Keep this checklist next to the build while you package it. Each column is a decision you make once per asset.

> Держите этот контрольный список рядом со сборкой, пока упаковываете её. Каждый столбец — это решение, которое принимается один раз для каждого актива.

| Asset type | What to parameterize | What to document | What to bundle for audit |
|---|---|---|---|
| Agent template | Every value that changes per customer: prompts, paths, scopes, credentials by reference, and thresholds. | Environment assumptions, expected inputs, handled failure modes, and the eval that defines working. | The data touched, the identity acted under, and the log of what the asset did. |
| MCP Server | Scopes, credentials by reference, and per-customer paths. | Expected inputs per tool, scope boundaries, and handled failure modes. | The data touched, the identity acted under, and the log of what the asset did. |
| Eval Suite | Thresholds and dataset paths that change per customer or environment. | The rubric logic, what the scores mean, and the baseline the asset is pinned to. | The data touched, the identity acted under, and the log of what the asset did. |

| Тип актива | Что параметризовать | Что документировать | Что включить для аудита |
|---|---|---|---|
| Шаблон агента | Каждое значение, которое меняется от клиента к клиенту: промпты, пути, области действия, учётные данные (по ссылке) и пороговые значения. | Допущения о среде, ожидаемые входные данные, обрабатываемые виды сбоев и оценку (eval), которая определяет, что актив работает. | Данные, с которыми работал актив, идентичность, от имени которой он действовал, и журнал того, что он сделал. |
| MCP-сервер | Области действия, учётные данные (по ссылке) и пути, специфичные для клиента. | Ожидаемые входные данные для каждого инструмента, границы областей действия и обрабатываемые виды сбоев. | Данные, с которыми работал актив, идентичность, от имени которой он действовал, и журнал того, что он сделал. |
| Набор оценок | Пороговые значения и пути к наборам данных, которые меняются в зависимости от клиента или среды. | Логику рубрики, значение баллов и базовый уровень, к которому привязан актив. | Данные, с которыми работал актив, идентичность, от имени которой он действовал, и журнал того, что он сделал. |

| | |
|---|---|
| **Handles well** | Parameterizing while the build is fresh turns one delivery into an asset the next engagement configures in hours. |
| **Adds cost or complexity** | Separating generalizable from customer-specific parts and documenting assumptions adds real time to the first build. |
| **Use a different approach** | For a one-off a customer will never reuse, packaging overhead is not worth it: ship the build and move on. |

| | |
|---|---|
| **Хорошо подходит** | Параметризация, пока сборка свежая, превращает одну поставку в актив, который следующий проект настраивает за несколько часов. |
| **Добавляет стоимость или сложность** | Разделение обобщаемых и специфичных для клиента частей и документирование допущений заметно увеличивают время на первую сборку. |
| **Используйте другой подход** | Для разового решения, которое клиент никогда не будет использовать повторно, затраты на упаковку не оправданы: поставьте сборку и двигайтесь дальше. |
