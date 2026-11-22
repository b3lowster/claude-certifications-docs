# Keeping cost, latency, and reliability in budget across agents
# Как удерживать стоимость, задержку и надёжность в рамках бюджета при работе нескольких агентов

A system that recovers from failure still must be affordable and fast, or it will not survive contact with a real bill.

> Система, которая восстанавливается после сбоев, всё равно должна быть доступной по цене и быстрой, иначе она не переживёт встречи с реальным счётом.

The retry budgets and fallbacks from the last screen keep it reliable. This screen instruments and budgets it, then handles the pattern that multiplies cost fastest: distributing work across several coordinating agents.

> Бюджеты повторов и резервные варианты (fallbacks) из предыдущего раздела обеспечивают её надёжность. В этом разделе мы добавим в неё измерения и бюджет, а затем разберём подход, который быстрее всего умножает стоимость: распределение работы между несколькими координирующими друг друга агентами.

## Cost and latency are invisible in development but decisive in production
## Стоимость и задержка незаметны при разработке, но решающие в продакшене

In development, you run a handful of calls and never see the bill. In production, the same calls run at volume, while cost and latency become the constraint. Observability for a Claude system means instrumenting three metrics per call: token usage (input and output tokens), latency, and error rate. With three metrics for every call, you can see which step is expensive or slow, instead of guessing from a total monthly bill. Instrument every call from the start. Treating observability as a later step means the bill arrives before the explanation. In code, it is a thin wrapper around the call that records the usage the API already returns.

> При разработке вы делаете несколько вызовов и никогда не видите счёт. В продакшене те же вызовы выполняются в больших объёмах, и стоимость и задержка (latency) становятся ограничением. Наблюдаемость (observability) системы на Claude означает измерение трёх метрик для каждого вызова: использования токенов (входных и выходных), задержки и доли ошибок. Имея три метрики для каждого вызова, вы видите, какой шаг дорогой или медленный, а не гадаете по общему месячному счёту. Измеряйте каждый вызов с самого начала. Если отложить наблюдаемость на потом, счёт придёт раньше объяснения. В коде это тонкая обёртка вокруг вызова, которая записывает данные об использовании, которые API и так возвращает.

```python
import time

def instrumented_call(make_call, step_name):
    start = time.perf_counter()
    resp = make_call()          # raises on any API error
    latency_ms = (time.perf_counter() - start) * 1000
    log_metric(step=step_name,
               input_tokens=resp.usage.input_tokens,
               output_tokens=resp.usage.output_tokens,
               latency_ms=latency_ms)
    return resp
```

Once every call logs those three metrics, a cost or latency problem stops being a mystery on the invoice and becomes a row you can sort.

> Как только каждый вызов записывает эти три метрики, проблема со стоимостью или задержкой перестаёт быть загадкой в счёте и становится строкой, которую можно отсортировать.

The value of per-call instrumentation is that it changes the questions you can answer. A cost spike without per-call logging gives you one question: why is the bill high? Per-call logging lets you ask which step, on which request type, is responsible, and retrieve the answer from the data directly. A flow that appears uniformly expensive often turns out to have one step doing ninety percent of the spend, and that step is where every optimization dollar should go. The same is true for latency: the slow step is rarely the one you expected, and the trace plus per-call timing tells you which it is instead of letting you optimize the wrong thing.

> Ценность измерения каждого вызова в том, что оно меняет вопросы, на которые можно ответить. Скачок стоимости без журналирования каждого вызова оставляет вам один вопрос: почему счёт такой большой? Журналирование каждого вызова позволяет спросить, какой шаг и на каком типе запросов за это отвечает, и получить ответ прямо из данных. Процесс, который кажется равномерно дорогим, часто оказывается таким, где один шаг даёт девяносто процентов расходов, и именно туда стоит направлять каждый доллар на оптимизацию. То же верно для задержки: медленный шаг редко оказывается тем, на который вы думали, а трассировка (trace) вместе с замером времени каждого вызова показывает, какой это шаг, и не даёт оптимизировать не то.

## The levers that affect the budget
## Рычаги, влияющие на бюджет

A cost or latency problem almost always traces to one of a few measurable components. Identifying the lever before tuning it is what keeps optimization from being guesswork. Select each tab for the lever and how it moves cost or latency.

> Проблема со стоимостью или задержкой почти всегда сводится к одному из нескольких измеримых компонентов. Если определить рычаг до того, как его настраивать, оптимизация не превращается в гадание.

### Model selection for the task
### Выбор модели под задачу

Choose a smaller, faster model to cut down on the cost and latency of a more sophisticated one. Reserve the most capable model for the steps that need it, and route simpler work elsewhere.

> Выбирайте меньшую и более быструю модель, чтобы снизить стоимость и задержку по сравнению с более сложной. Оставляйте самую мощную модель для шагов, которым она нужна, а более простую работу направляйте на другие модели.

### Prompt and context size
### Размер промпта и контекста

Every token in the prompt contributes to cost. Trimming context and removing unnecessary tool output reduces the per-call cost directly. This is the context-engineering work from the first module applied to the operational cost.

> Каждый токен в промпте добавляет стоимость. Сокращение контекста и удаление ненужного вывода инструментов напрямую снижают стоимость каждого вызова. Это та же работа по контекстной инженерии (context engineering) из первого модуля, применённая к эксплуатационным расходам.

### Number of tool calls
### Количество вызовов инструментов

Each call adds both cost and latency. A flow that makes more calls than is needed is a common and measurable source of unnecessary spending, one that becomes visible the moment you instrument a call.

> Каждый вызов добавляет и стоимость, и задержку. Процесс, который делает больше вызовов, чем нужно, — частый и измеримый источник лишних расходов, который становится виден, как только вы начинаете измерять вызовы.

### Streamed versus batched output, and prompt caching for repeated context
### Потоковый или пакетный вывод и кэширование промптов для повторяющегося контекста

Streaming changes how latency is perceived by returning the first token to the user as soon as it is ready rather than waiting for the full response. For a user-facing feature, this matters: a response that starts arriving in 300ms feels faster than one that delivers the same content in a single block after two seconds, even if the total generation time is identical. Prompt caching is covered in its own section below.

> Потоковая передача (streaming) меняет восприятие задержки: первый токен возвращается пользователю, как только он готов, без ожидания полного ответа. Для функции, с которой работает пользователь, это важно: ответ, который начинает поступать через 300 мс, ощущается быстрее, чем ответ, выдающий то же содержимое одним блоком через две секунды, даже если общее время генерации одинаковое. Кэшированию промптов посвящён отдельный раздел ниже.

The pattern is to accumulate deltas by index until the stream closes, then reconstruct the tool calls from the completed blocks:

> Подход состоит в том, чтобы накапливать дельты (deltas) по индексу, пока поток не закроется, а затем восстанавливать вызовы инструментов из завершённых блоков:

```python
def stream_with_tools(client, **kwargs):
    tool_blocks = {}          # index -> accumulated block
    text_chunks = []

    with client.messages.stream(**kwargs) as stream:
        for event in stream:
            if event.type == "content_block_start":
                block = event.content_block
                tool_blocks[event.index] = {
                    "type": block.type,
                    "id": getattr(block, "id", None),
                    "name": getattr(block, "name", None),
                    "input_json": ""
                }
            elif event.type == "content_block_delta":
                delta = event.delta
                if delta.type == "input_json_delta":
                    tool_blocks[event.index]["input_json"] += delta.partial_json
                elif delta.type == "text_delta":
                    text_chunks.append(delta.text)
            elif event.type == "message_stop":
                break

    # reconstruct completed tool calls after stream closes
    tool_calls = []
    for block in tool_blocks.values():
        if block["type"] == "tool_use":
            tool_calls.append({
                "id": block["id"],
                "name": block["name"],
                "input": json.loads(block["input_json"])
            })

    return "".join(text_chunks), tool_calls
```

A `tool_use` block is not safe to act on until the stream closes and the full `input_json` has been accumulated. Acting on a partial block produces malformed tool inputs. The same retriable-versus-terminal failure handling from the failure screen applies here: a stream that breaks mid-response is a transient failure and the whole request should be retried, not the partial output passed downstream.

> На блок `tool_use` нельзя безопасно опираться, пока поток не закрылся и не накоплен полный `input_json`. Действия на основе частичного блока приводят к некорректным входным данным инструмента. Здесь применяется та же обработка сбоев по принципу «повторяемый или окончательный» из раздела о сбоях: поток, оборвавшийся посреди ответа, — временный сбой, и весь запрос нужно повторить, а не передавать частичный результат дальше.

## Prompt caching: reusing the work already done on a stable prefix
## Кэширование промптов: повторное использование уже выполненной работы над неизменным префиксом

Before the model generates anything, it processes your input: it breaks the prompt into tokens and builds the internal representations it needs to attend over them. On an ordinary request, that processing work is discarded once the response comes back. When your next request repeats the same content, the same processing runs again from scratch. The lever that removes that repeated work is prompt caching.

> Прежде чем что-либо сгенерировать, модель обрабатывает ваши входные данные: разбивает промпт на токены и строит внутренние представления, необходимые для работы механизма внимания (attention) над ними. При обычном запросе эта работа отбрасывается, как только приходит ответ. Когда следующий запрос повторяет то же содержимое, та же обработка выполняется заново. Рычаг, устраняющий эту повторную работу, — кэширование промптов (prompt caching).

Prompt caching stores the processing work for a stretch of content so a later request can read it back rather than recompute it. The first request writes the work to a cache, and follow-up requests that send the same content up to a marked point read from that cache instead of reprocessing. Cache writes are billed at a premium over base input tokens, 1.25x for the 5-minute TTL, 2x for the 1-hour, while cache reads cost a fraction of standard input (0.1x), so the economics only work when reads outnumber writes. That is also why caching fits stable, frequently reused prefixes: the more requests that hit the same cached content, the lower the blended cost and latency across the batch.

> Кэширование промптов сохраняет результат обработки фрагмента содержимого, чтобы последующий запрос мог прочитать его, а не вычислять заново. Первый запрос записывает результат в кэш, а последующие запросы, отправляющие то же содержимое до отмеченной точки, читают из кэша вместо повторной обработки. Запись в кэш тарифицируется с надбавкой к базовой цене входных токенов — 1,25x при TTL 5 минут и 2x при TTL 1 час, — а чтение из кэша стоит лишь долю стандартной цены входных токенов (0,1x), поэтому экономия возникает, только когда чтений больше, чем записей. Именно поэтому кэширование подходит для неизменных, часто повторяемых префиксов: чем больше запросов попадает в одно и то же закэшированное содержимое, тем ниже средние стоимость и задержка по всему набору запросов.

Caching can be set up automatically or with explicit breakpoints. In automatic mode, you add a single cache flag at the top level of your request and the system manages breakpoints as the conversation grows, this is the recommended starting point for most use cases. With explicit breakpoints, you place a `cache_control` marker on a specific content block, and the model caches all the work up to and including that point. Either way, content after the last breakpoint is processed normally. The components most worth caching are the ones that stay the same between requests: a long system prompt and a large tool schema are the usual candidates, since they rarely change while the user message changes every turn.

> Кэширование можно настроить автоматически или с помощью явных точек кэширования (breakpoints). В автоматическом режиме вы добавляете один флаг кэширования на верхнем уровне запроса, и система сама управляет точками кэширования по мере роста диалога; это рекомендуемый вариант для начала в большинстве случаев. При явных точках кэширования вы ставите маркер `cache_control` на конкретный блок содержимого, и модель кэширует всю работу до этой точки включительно. В обоих случаях содержимое после последней точки кэширования обрабатывается обычным образом. Больше всего стоит кэшировать компоненты, которые не меняются между запросами: обычные кандидаты — длинный системный промпт и большая схема инструментов, поскольку они меняются редко, а сообщение пользователя меняется каждый ход.

Three properties decide whether caching helps with a given workload:

> Поможет ли кэширование при конкретной нагрузке, определяют три свойства:

1. **The cached content must be identical.** The cache is matched on an exact prefix, so any change before the breakpoint, even adding a single word like "please," invalidates the cache and forces a full reprocess. This is why caching fits stable content and works against anything that must reflect live state, because content that changes every request never produces a cache hit.
2. **The same content must recur and recur soon.** The default cache lifetime is five minutes, refreshed on each hit. A one-hour lifetime is available at additional cost. The saving only lands when the same prefix is sent again within that window. A prefix reused several times a minute pays off, while one reused once an hour does not under the default TTL, because the cache has expired before the next request arrives.
3. **The cached prefix must be long enough to clear the minimum.** There is a minimum length threshold for caching, and it varies by model. Shorter prompts see no benefit regardless of how stable they are. The longer and more stable the prefix, the more processing work the cache reuses, which is why caching is most effective on high-volume systems carrying a long, fixed system prompt.

> 1. **Закэшированное содержимое должно быть идентичным.** Кэш сопоставляется по точному префиксу, поэтому любое изменение до точки кэширования, даже добавление одного слова вроде «пожалуйста», делает кэш недействительным и вызывает полную повторную обработку. Поэтому кэширование подходит для неизменного содержимого и не работает для всего, что должно отражать текущее состояние: содержимое, меняющееся при каждом запросе, никогда не попадает в кэш.
> 2. **То же содержимое должно повторяться, и повторяться скоро.** По умолчанию время жизни кэша — пять минут, и оно обновляется при каждом попадании. Время жизни в один час доступно за дополнительную плату. Экономия возникает, только если тот же префикс отправляется снова в пределах этого окна. Префикс, повторяемый несколько раз в минуту, окупается, а повторяемый раз в час — нет при TTL по умолчанию, потому что кэш истекает до прихода следующего запроса.
> 3. **Закэшированный префикс должен быть достаточно длинным, чтобы превысить минимум.** Для кэширования есть минимальный порог длины, и он зависит от модели. Более короткие промпты не получают выгоды, насколько бы неизменными они ни были. Чем длиннее и стабильнее префикс, тем больше работы по обработке кэш использует повторно, поэтому кэширование наиболее эффективно в высоконагруженных системах с длинным фиксированным системным промптом.

There is one tradeoff to weigh against the saving. Caching assumes the cached content is still correct on the later request. If the prefix needs to reflect data that can change, the cache holds a version that may be stale for as long as it lives. That is a consistency window your use case must be able to tolerate. For a fixed system prompt and a stable tool schema there is nothing to go stale, which is why those are safe and high-value places to cache.

> Экономию нужно сопоставить с одним компромиссом. Кэширование предполагает, что закэшированное содержимое по-прежнему верно при последующем запросе. Если префикс должен отражать данные, которые могут меняться, кэш хранит версию, которая может быть устаревшей всё время своей жизни. Это окно несогласованности, которое ваш сценарий должен быть способен допустить. У фиксированного системного промпта и стабильной схемы инструментов устаревать нечему, поэтому их безопасно и выгодно кэшировать.

## The Batches API: trading latency for a lower bill
## Batches API: обмен задержки на меньший счёт

Some work does not need an answer immediately. An overnight classification run, a backfill over a large dataset, or a scheduled report can all wait. For that kind of work, the Message Batches API processes requests asynchronously, and in exchange it costs less per request than the same calls made one at a time. The cost reduction is significant enough that it is the deciding lever for any non-urgent, high-volume task. The current discount is version-pinned, so confirm it against the reference layer at build time.

> Некоторой работе не нужен немедленный ответ. Ночная классификация, дозаполнение (backfill) большого набора данных или плановый отчёт — всё это может подождать. Для такой работы Message Batches API обрабатывает запросы асинхронно и взамен стоит дешевле в расчёте на запрос, чем те же вызовы по одному. Снижение стоимости достаточно значительно, чтобы стать решающим рычагом для любой несрочной задачи с большим объёмом. Текущий размер скидки привязан к версии, поэтому сверяйте его со справочным разделом во время сборки.

The trade is latency for cost. You submit a batch and results come back within an asynchronous completion window rather than immediately. A batch is the wrong tool for anything a user is waiting on and the right tool for anything driven by a schedule. The decision mirrors streaming in reverse: streaming optimizes how fast a single response feels for a user in the loop, while batching optimizes the bill for work where no user is waiting. The two levers never compete for the same request, because a request is either user-facing, or it is not.

> Здесь задержка обменивается на стоимость. Вы отправляете пакет, и результаты приходят в пределах асинхронного окна выполнения, а не сразу. Пакет — неподходящий инструмент для всего, чего ждёт пользователь, и подходящий — для всего, что выполняется по расписанию. Это решение зеркально противоположно потоковой передаче: потоковая передача оптимизирует то, насколько быстрым ощущается отдельный ответ для пользователя в процессе, а пакетная обработка оптимизирует счёт за работу, которую никто не ждёт. Эти два рычага никогда не конкурируют за один и тот же запрос, потому что запрос либо обращён к пользователю, либо нет.

Batching and prompt caching compound when a non-urgent job reuses the same context across many requests. The batch discount lowers the cost of each request and caching lowers the cost of the repeated prefix inside each one, so a scheduled job carrying a long fixed system prompt benefits from both. That combination is exactly what the cost-and-orchestration checkpoint later in this module asks you to recognize.

> Пакетная обработка и кэширование промптов усиливают друг друга, когда несрочное задание повторно использует один и тот же контекст во многих запросах. Скидка за пакет снижает стоимость каждого запроса, а кэширование снижает стоимость повторяющегося префикса внутри каждого из них, поэтому плановое задание с длинным фиксированным системным промптом выигрывает от обоих. Именно такое сочетание вам предложат распознать на контрольной точке по стоимости и оркестрации далее в этом модуле.

## Multi-agent orchestration as a deliberate tradeoff
## Многоагентная оркестрация как осознанный компромисс

In an orchestrator-worker pattern, a lead agent decomposes a task into subtasks and delegates them to several subagents that work in parallel, each with its own context window. Once assignments are complete, they compile their results. In code, the structure consists of planning, a parallel fan-out, and synthesis.

> В подходе «оркестратор — исполнители» (orchestrator-worker) ведущий агент (lead agent) разбивает задачу на подзадачи и делегирует их нескольким субагентам, которые работают параллельно, каждый в собственном контекстном окне. Когда задания выполнены, результаты собираются вместе. В коде эта структура состоит из планирования, параллельного распределения (fan-out) и синтеза.

```python
async def orchestrate(task):
    plan = await lead.plan(task)              # lead agent decomposes
    results = await gather(*[                 # subagents run in parallel
        worker.run(subtask) for subtask in plan.subtasks
    ])                                        # each spends its own tokens
    return await lead.synthesize(results)     # lead compiles the answer
```

This genuinely helps with large tasks that can be split into independent parts. For example, research across many separate sources, since the subagents can explore at the same time instead of one after another.

> Это действительно помогает с большими задачами, которые можно разделить на независимые части. Например, с исследованием по множеству отдельных источников, поскольку субагенты могут изучать их одновременно, а не один за другим.

The way to hold this is as a hiring decision. Five researchers finish a broad survey faster than one, but you pay five salaries. You only hire a team when the work genuinely splits into parts people can do without waiting on each other.

> Это стоит воспринимать как решение о найме. Пять исследователей закончат широкий обзор быстрее, чем один, но вы платите пять зарплат. Команду нанимают, только когда работа действительно делится на части, которые люди могут выполнять, не дожидаясь друг друга.

Anthropic's own research system uses this pattern and has reported findings that define the tradeoff. On an Anthropic internal research eval, a multi-agent setup with Claude Opus 4 as lead and Claude Sonnet 4 subagents showed a substantial improvement over a single-agent Claude Opus 4 baseline on internal evals. The cost is roughly fifteen times the tokens of a normal chat interaction, because every subagent spends its own tokens against its own context.

> Собственная исследовательская система Anthropic использует этот подход, и опубликованные результаты определяют этот компромисс. Во внутренней исследовательской оценке Anthropic многоагентная конфигурация с Claude Opus 4 в роли ведущего агента и субагентами Claude Sonnet 4 показала существенное улучшение по сравнению с базовым вариантом из одного агента Claude Opus 4. Цена — примерно в пятнадцать раз больше токенов, чем при обычном диалоге в чате, потому что каждый субагент тратит собственные токены на собственный контекст.

The pattern is also less effective for tightly coupled tasks such as coding, where each step depends on previous parts and cannot be explored in parallel. Anthropic's analysis found that token usage accounts for most of the performance variance. The architecture works primarily because it buys more parallel computation.

> Этот подход также менее эффективен для тесно связанных задач, например программирования, где каждый шаг зависит от предыдущих частей и не может исследоваться параллельно. Анализ Anthropic показал, что большая часть разброса в результатах объясняется расходом токенов. Архитектура работает прежде всего потому, что покупает больше параллельных вычислений.

Use it only when the task genuinely requires parallel exploration. A single agent with good context handles most work at a fraction of the cost. The multiplier also compounds when something misbehaves. A runaway subagent or an oversized tool result can push well past the fifteen times baseline before the request completes.

> Используйте его, только когда задача действительно требует параллельного исследования. Один агент с хорошим контекстом справляется с большей частью работы за малую долю стоимости. Множитель к тому же растёт, когда что-то идёт не так. Вышедший из-под контроля субагент или слишком большой результат инструмента могут далеко превысить пятнадцатикратный уровень ещё до завершения запроса.

A rough cost estimation makes the tradeoff concrete. Suppose a single agent answers a research question in about ten thousand tokens. The orchestrator-worker version spins up a lead and four subagents, each reading its own slice of sources in its own context. The lead then synthesizes their returns. Anthropic reports that five contexts plus the synthesis pass use fifteen times the number of tokens. So, the same question costs on the order of a hundred and fifty thousand tokens.

> Грубая оценка стоимости делает компромисс конкретным. Допустим, один агент отвечает на исследовательский вопрос примерно за десять тысяч токенов. Версия «оркестратор — исполнители» запускает ведущего агента и четыре субагента, каждый из которых читает свою часть источников в собственном контексте. Затем ведущий агент синтезирует их результаты. По данным Anthropic, пять контекстов плюс проход синтеза используют в пятнадцать раз больше токенов. Значит, тот же вопрос обходится примерно в сто пятьдесят тысяч токенов.

If the question was a single lookup dressed up as research, you paid the multiplier for nothing, because four of the five contexts were doing work the task never needed. The number is neither inherently large nor small. Its value depends entirely on whether the task requires the additional agents.

> Если вопрос был простым поиском одного факта, замаскированным под исследование, вы заплатили множитель впустую, потому что четыре из пяти контекстов выполняли работу, которая задаче вообще не была нужна. Само по себе это число ни большое, ни маленькое. Его ценность целиком зависит от того, нужны ли задаче дополнительные агенты.

There is a control dimension the cost estimation does not capture. Spreading work across agents multiplies the places a failure can occur, so each subagent needs the same retriable-versus-terminal handling, the same backoff, and the same fallback discipline from the failure screen, applied independently. A single subagent that hits a rate limit and has no backoff can stall the whole compilation step while the lead waits for a return that never comes. The orchestration pattern does not replace the failure-handling work, it multiplies it, which is another reason to use it only when the parallel exploration is worth that added surface area. A model choice detail also helps here: consider using a more capable model as the lead agent and cheaper models for the subagents, so you are not paying top-tier rates across every parallel context. This reduces the cost multiplier while preserving the coordination quality where it matters.

> Есть измерение управляемости, которое оценка стоимости не отражает. Распределение работы между агентами умножает число мест, где может произойти сбой, поэтому каждому субагенту нужна та же обработка по принципу «повторяемый или окончательный», та же отсрочка (backoff) и та же дисциплина резервных вариантов из раздела о сбоях, применённые к каждому независимо. Один субагент, упёршийся в ограничение частоты запросов без отсрочки, может застопорить весь этап сборки результатов, пока ведущий агент ждёт ответа, который так и не придёт. Оркестрация не заменяет работу по обработке сбоев, а умножает её, и это ещё одна причина использовать её, только когда параллельное исследование стоит этой дополнительной площади для сбоев. Здесь помогает и выбор моделей: рассмотрите вариант с более мощной моделью в роли ведущего агента и более дешёвыми моделями для субагентов, чтобы не платить по высшему тарифу за каждый параллельный контекст. Это снижает множитель стоимости, сохраняя качество координации там, где оно важно.

## Reliability has a floor you tune cost within
## У надёжности есть нижняя граница, в пределах которой настраивается стоимость

Cost is only half of the budget. The other half is reliability, and it establishes a baseline below which the cost should not go.

> Стоимость — лишь половина бюджета. Другая половина — надёжность, и она задаёт базовый уровень, ниже которого снижать стоимость нельзя.

The cheapest configuration is rarely the most reliable. Start by defining the base first, such as a retry budget and a latency ceiling, and then tune cost above it rather than below. Cutting costs beneath the reliability floor replaces a visible expense with silent failures. In production, this is often a worse trade because a slightly higher bill is easier to defend than a system that doesn't work.

> Самая дешёвая конфигурация редко бывает самой надёжной. Начните с определения базы, например бюджета повторов и потолка задержки, а затем настраивайте стоимость выше неё, а не ниже. Снижение затрат ниже уровня надёжности заменяет видимые расходы незаметными сбоями. В продакшене это часто худший обмен, потому что немного больший счёт легче обосновать, чем неработающую систему.

A concrete version of the reliability floor makes the discipline clear. Suppose you decide a user-facing request must be completed within four seconds and may retry a failed dependency up to three times. Those constraints define the floor. Now, every cost optimization must satisfy these requirements. Switching to a smaller, cheaper model is fine if it still fits within the latency ceiling and does not increase the error rate enough to burn the retry budget. Reducing the retry count to two to save costs on a slow dependency is not acceptable if it pushes the failure rate beyond what the floor allows. In this case, you would be exchanging a lower cost for more failed requests.

> Конкретный пример нижней границы надёжности проясняет этот принцип. Допустим, вы решили, что запрос пользователя должен выполняться за четыре секунды и может повторять обращение к сбойной зависимости до трёх раз. Эти ограничения задают нижнюю границу. Теперь каждая оптимизация стоимости должна удовлетворять этим требованиям. Переход на меньшую и более дешёвую модель допустим, если она по-прежнему укладывается в потолок задержки и не повышает долю ошибок настолько, чтобы исчерпать бюджет повторов. Сокращение числа повторов до двух ради экономии на медленной зависимости недопустимо, если это поднимает долю сбоев выше того, что разрешает нижняя граница. В этом случае вы обменивали бы меньшую стоимость на большее число неудачных запросов.

The floor is what keeps optimization honest: it forces every cost-saving change to demonstrate that it did not quietly trade reliability away. It also provides a clear boundary below which you do not cut, regardless of how attractive the savings may appear.

> Именно нижняя граница сохраняет оптимизацию честной: она заставляет каждое изменение, экономящее средства, доказать, что оно незаметно не пожертвовало надёжностью. Она также задаёт чёткую границу, ниже которой вы не сокращаете, какой бы привлекательной ни казалась экономия.

The order matters, because cost and reliability create opposing pressures, and cost is usually louder. A high bill shows up on a dashboard every day and generates constant pressure to reduce spending. A reliability problem shows up as occasional failures that are easy to dismiss as noise until they accumulate into an incident. If you optimize cost first and reliability second, the louder pressure wins, and you discover the reliability floor only after crossing it. Setting the floor first reverses that: reliability becomes the fixed constraint, and cost becomes the thing you optimize underneath it. The eval set from the earlier section is what makes the floor enforceable: a pinned baseline score defines the minimum acceptable reliability in a checkable form, so any cost-saving change that drops the score below the baseline fails the gate before it ships.

> Порядок важен, потому что стоимость и надёжность создают противоположное давление, и стоимость обычно «громче». Большой счёт каждый день виден на дашборде и постоянно подталкивает к сокращению расходов. Проблема с надёжностью проявляется редкими сбоями, которые легко списать на шум, пока они не накопятся в инцидент. Если сначала оптимизировать стоимость, а потом надёжность, побеждает более громкое давление, и вы обнаруживаете нижнюю границу надёжности, только пересёкши её. Если сначала задать нижнюю границу, всё наоборот: надёжность становится фиксированным ограничением, а стоимость — тем, что вы оптимизируете в его рамках. Обеспечить соблюдение нижней границы позволяет оценочный набор (eval) из предыдущего раздела: зафиксированный базовый балл задаёт минимально допустимую надёжность в проверяемом виде, поэтому любое изменение ради экономии, которое опускает балл ниже базового, не проходит проверку ещё до выпуска.

## The observability and orchestration reference you can keep open while you build
## Справочник по наблюдаемости и оркестрации, который стоит держать под рукой во время разработки

| Metric | Where to instrument it | Single-agent versus orchestrator-worker |
|---|---|---|
| Token cost | Per call, aggregated per request and per flow. | A single agent incurs a token cost once per step. An orchestrator-worker multiplies token consumption by the number of subagents, roughly a 15x token multiplier in Anthropic's reported case. That multiplier applies to both input and output tokens, since each subagent receives its own context and generates its own output. |
| Latency | Per call, with traces identifying the slowest step in the workflow. | Parallel subagents can reduce wall-clock time on independent work but add coordination latency to plan and compile. |
| Error rate | Per call and per dependency. | More agents mean potential failure points, each subagent requires the same retry and fallback handling as a single agent. |

| Метрика | Где её измерять | Один агент или «оркестратор — исполнители» |
|---|---|---|
| Стоимость в токенах | Для каждого вызова, с агрегацией по запросу и по процессу. | Один агент несёт расход токенов один раз на шаг. Схема «оркестратор — исполнители» умножает расход токенов на число субагентов — по данным Anthropic, примерно в 15 раз. Этот множитель относится и к входным, и к выходным токенам, поскольку каждый субагент получает собственный контекст и генерирует собственный вывод. |
| Задержка | Для каждого вызова, с трассировкой, показывающей самый медленный шаг процесса. | Параллельные субагенты могут сократить реальное время (wall-clock time) на независимой работе, но добавляют задержку на координацию — планирование и сборку результатов. |
| Доля ошибок | Для каждого вызова и для каждой зависимости. | Больше агентов — больше потенциальных точек отказа; каждому субагенту нужна та же обработка повторов и резервных вариантов, что и одиночному агенту. |

| | |
|---|---|
| **Handles well** | Makes spend and latency visible per call, so a cost problem traces to a named lever. |
| **Adds cost or complexity** | Parallel subagents multiply token cost, roughly by 15x in the reported case, before improving any answer. |
| **Use a different approach** | For tightly coupled work, such as coding, a single agent with good context beats fan-out. |

| | |
|---|---|
| **Хорошо подходит** | Делает расходы и задержку видимыми для каждого вызова, поэтому проблему со стоимостью можно свести к конкретному рычагу. |
| **Добавляет стоимость или сложность** | Параллельные субагенты умножают стоимость в токенах — в описанном случае примерно в 15 раз — ещё до того, как улучшат какой-либо ответ. |
| **Используйте другой подход** | Для тесно связанной работы, например программирования, один агент с хорошим контекстом лучше параллельного распределения (fan-out). |
