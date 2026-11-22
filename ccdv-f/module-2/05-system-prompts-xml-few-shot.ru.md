# System prompts, XML, few-shot, and output constraints
# Системные промпты, XML, few-shot и ограничения на вывод

A prompt that works once in interactive use often breaks when it runs in production against untested inputs. The fix for this isn’t just adding more words, it is identifying which structural piece is missing from the prompt and adding that one piece. This section reviews how to read a failed output, then walks through the four techniques that produce the fixes.

> Промпт, который один раз сработал в интерактивном режиме, часто ломается, когда запускается в продакшене на непроверенных входных данных. Решение здесь не в том, чтобы просто добавить больше слов, а в том, чтобы определить, какого структурного элемента не хватает в промпте, и добавить именно этот элемент. В этом разделе рассматривается, как анализировать неудачный вывод, а затем разбираются четыре техники, которые позволяют исправить ситуацию.

When a first-pass response misses, the instinct is often to add more words to the prompt and run it again. However, that instinct can make the problem harder to isolate and rarely fixes it. Rewording changes how you say something but does not add to the structural piece of the prompt that’s missing. For example, if Claude is crossing the boundary between your instructions and your input data, clearer phrasing will not fix it, and if the output format keeps drifting, "please format this correctly" will not fix it either.

> Когда ответ с первой попытки не попадает в цель, инстинктивно хочется добавить в промпт больше слов и запустить его снова. Однако такой подход может затруднить локализацию проблемы и редко её решает. Переформулирование меняет то, как вы что-то говорите, но не добавляет недостающий структурный элемент промпта. Например, если Claude пересекает границу между вашими инструкциями и вашими входными данными, более ясная формулировка этого не исправит, а если формат вывода постоянно «плывёт» (drifting), фраза «пожалуйста, отформатируй это правильно» тоже не поможет.

The failure mode tells you which of the four techniques is absent. Diagnose how your prompt is failing first, then add the specific technique that addresses that failure. The four techniques themselves are defined in full further down this screen.

> Характер сбоя (failure mode) подсказывает, какой из четырёх техник не хватает. Сначала диагностируйте, как именно ваш промпт даёт сбой, а затем добавьте конкретную технику, устраняющую этот сбой. Сами четыре техники подробно описаны ниже на этом экране.

## Diagnosing the failure: which technique is missing
## Диагностика сбоя: какой техники не хватает

| What you observed | What the prompt is missing | Why this technique is the fix |
|---|---|---|
| The result comes back in the wrong shape: a sentence where you expected a label, prose where you expected JSON. | An output constraint. The prompt never specified the form, field names, or stopping point of the response. | An output constraint controls the form of the response independent of its content. Without one, Claude returns plausible text that the downstream parser was not built to accept. |
| The content is off: scope drifts, tone shifts, or Claude answers a wider question than you asked, and it gets worse deeper into the conversation. | A system prompt, or a more specific one. The behavioral contract was too vague to hold across turns. | The system prompt sets the rules that apply to every response regardless of the user turn. When it is underspecified, there is nothing holding role, scope, and format steady as the conversation runs on. |
| The task is right, but the structure is invented: Claude understood what to do and produced output in a shape you never asked for. | Few-shot examples. Claude cannot infer an exact structure from a description alone. | Few-shot examples show the pattern rather than describe it. One correct input-output pair gives Claude the exact shape to match, which a written instruction often fails to pin down. |
| Output is clean on the inputs you tested but breaks on a variant: an edge case, an unusual field, an input you did not anticipate. | A constraint covering the variant. The prompt handles the happy path and has no rule for the case the parser breaks on. | The prompt was validated against a narrow set of inputs. Naming the variant in the constraint, or adding an example that covers it, closes the gap the test inputs never exposed. |

| Что вы наблюдаете | Чего не хватает в промпте | Почему эта техника решает проблему |
|---|---|---|
| Результат возвращается не в той форме: предложение там, где вы ожидали метку (label), связный текст там, где ожидали JSON. | Ограничения на вывод (output constraint). В промпте не были заданы форма ответа, имена полей или точка, где ответ должен заканчиваться. | Ограничение на вывод управляет формой ответа независимо от его содержания. Без него Claude возвращает правдоподобный текст, на приём которого последующий парсер (downstream parser) не рассчитан. |
| Содержание не то: рамки задачи (scope) расплываются, меняется тон или Claude отвечает на более широкий вопрос, чем вы задали, и чем дальше в диалог, тем хуже. | Системного промпта или более конкретного системного промпта. Поведенческий контракт был слишком размытым, чтобы сохраняться от хода к ходу. | Системный промпт задаёт правила, которые применяются к каждому ответу независимо от хода пользователя. Если он недостаточно конкретен, ничто не удерживает роль, рамки задачи и формат стабильными по мере продолжения диалога. |
| Задача понята правильно, но структура придумана: Claude понял, что нужно сделать, и выдал вывод в форме, которую вы никогда не запрашивали. | Примеров few-shot. Claude не может вывести точную структуру только из описания. | Примеры few-shot показывают шаблон, а не описывают его. Одна правильная пара «вход — выход» даёт Claude точную форму для воспроизведения, чего письменная инструкция часто не может однозначно задать. |
| На протестированных входных данных вывод чистый, но ломается на каком-то варианте: пограничном случае (edge case), необычном поле, входных данных, которые вы не предусмотрели. | Ограничения, покрывающего этот вариант. Промпт обрабатывает «счастливый путь» (happy path), но не содержит правила для случая, на котором ломается парсер. | Промпт проверялся на узком наборе входных данных. Явное упоминание варианта в ограничении или добавление примера, который его покрывает, закрывает пробел, который тестовые входные данные так и не выявили. |

## Diagnosing a classification prompt that returns the wrong output shape
## Диагностика промпта классификации, возвращающего вывод не в той форме

The rule is simple: name the failure, add the one technique that matches it, and re-run it. If it still fails, diagnose again. When a prompt keeps getting longer with every pass, that’s the sign you’re skipping the diagnosis step and just adding words.

> Правило простое: назовите сбой, добавьте одну соответствующую ему технику и запустите промпт снова. Если сбой сохраняется, проведите диагностику ещё раз. Если промпт становится длиннее с каждой итерацией, это признак того, что вы пропускаете этап диагностики и просто добавляете слова.

The pattern below is the first row of the table in action: a prompt that produces the right content in a shape the downstream code cannot accept. The classifier understands the task and returns the correct category, but the form of that answer varies from run to run, so the router that consumes it fails. The missing piece is an output constraint, and the fix pulls in two of the other techniques to lock the label set and show the format. The walkthrough moves from the bare prompt that causes the problem to the constrained version that resolves it.

> Пример ниже — это первая строка таблицы в действии: промпт, который выдаёт правильное содержание, но в форме, которую последующий код (downstream code) не может принять. Классификатор понимает задачу и возвращает правильную категорию, но форма ответа меняется от запуска к запуску, поэтому роутер (router), который этот ответ потребляет, даёт сбой. Недостающий элемент — ограничение на вывод (output constraint), а для исправления привлекаются ещё две техники: чтобы зафиксировать набор меток (label set) и показать формат. Разбор ведёт от «голого» промпта, вызывающего проблему, к версии с ограничениями, которая её устраняет.

## Worked example: a classification prompt before and after
## Разобранный пример: промпт классификации до и после

A developer needs Claude to classify support tickets into three categories: billing, technical, and escalation. The first prompt is a bare instruction with no constraint on the output:

> Разработчику нужно, чтобы Claude классифицировал обращения в поддержку (support tickets) по трём категориям: billing (оплата), technical (технические вопросы) и escalation (эскалация). Первый промпт — это «голая» инструкция без каких-либо ограничений на вывод:

**❌ Before / До: prompt without an output constraint / промпт без ограничения на вывод**

```text
System: "You are a support classifier. Classify the ticket."

User: <ticket>I was charged twice for the same month.</ticket>
```

> *Перевод:* System: «Ты классификатор обращений в поддержку. Классифицируй обращение.»
> User: `<ticket>`С меня дважды списали оплату за один и тот же месяц.`</ticket>`

Claude returns "Billing" on some runs, "billing" on others, and occasionally a full sentence like "This looks like a billing issue." The downstream router expects one of a fixed set of labels and breaks on the inconsistency.

> В одних запусках Claude возвращает «Billing», в других — «billing», а иногда целое предложение вроде «This looks like a billing issue.» («Похоже, это вопрос оплаты.»). Последующий роутер ожидает одну метку из фиксированного набора и ломается на этой непоследовательности.

Read this against the table above, this situation matches what is described in the first row: the output comes back in a shape the parser cannot accept, so the missing piece is an output constraint. Adding that constraint pulls in two more techniques, because locking the label set and showing the format are jobs those techniques do better than a written instruction can. Few-shot examples show Claude the exact label and casing to return, and XML tags keep those examples separate from the instruction so Claude does not read them as part of the task:

> Если сопоставить это с таблицей выше, ситуация соответствует описанию в первой строке: вывод возвращается в форме, которую парсер не может принять, значит, недостающий элемент — ограничение на вывод. Добавление этого ограничения влечёт за собой ещё две техники, потому что зафиксировать набор меток и показать формат эти техники умеют лучше, чем письменная инструкция. Примеры few-shot показывают Claude точную метку и регистр, которые нужно вернуть, а XML-теги отделяют эти примеры от инструкции, чтобы Claude не воспринимал их как часть задачи:

**✅ After / После: output constraint + XML tags + few-shot / ограничение на вывод + XML-теги + few-shot**

```text
System: "You are a support classifier. Classify each ticket into exactly one of: BILLING, TECHNICAL, ESCALATION. Return only the label. No other text."

<sample_input>My account shows two charges for April.</sample_input>
<ideal_output>BILLING</ideal_output>

<sample_input>The API keeps returning a 429 error.</sample_input>
<ideal_output>TECHNICAL</ideal_output>

User: <ticket>I was charged twice for the same month.</ticket>
```

> *Перевод:* System: «Ты классификатор обращений в поддержку. Отнеси каждое обращение ровно к одной из категорий: BILLING, TECHNICAL, ESCALATION. Верни только метку. Никакого другого текста.»
> `<sample_input>`В моём аккаунте два списания за апрель.`</sample_input>` → `<ideal_output>`BILLING`</ideal_output>`
> `<sample_input>`API постоянно возвращает ошибку 429.`</sample_input>` → `<ideal_output>`TECHNICAL`</ideal_output>`
> User: `<ticket>`С меня дважды списали оплату за один и тот же месяц.`</ticket>`

Three techniques are doing distinct work here. The system prompt sets the output contract: exactly one label from a fixed set, nothing else. The XML tags mark where each example ends and the next begins, so Claude does not read the examples as part of the instruction. The few-shot pairs show the exact casing and format rather than describing it. Together they produce a result consistent enough to route programmatically.

> Здесь три техники выполняют разную работу. Системный промпт задаёт контракт вывода: ровно одна метка из фиксированного набора и ничего больше. XML-теги обозначают, где заканчивается один пример и начинается следующий, чтобы Claude не воспринимал примеры как часть инструкции. Пары few-shot показывают точный регистр и формат, а не описывают их. Вместе они дают результат, достаточно стабильный для программной маршрутизации.

The table below shows how we can stack all the four techniques together, where the prompt should be simplified, and where we should diagnose before adding more before too many iterations.

> В таблице ниже показано, как можно сочетать все четыре техники, где промпт следует упростить и где нужно провести диагностику, прежде чем добавлять что-то ещё, не доводя дело до слишком большого числа итераций.

| Approach | When it applies |
|---|---|
| **Stack all four techniques** | Stacking all four techniques against a clearly defined output contract. Tasks with well-specified formats and edge cases that can be covered by examples. |
| **Simplify the prompt** | Adding all four techniques to a simple task that only needs one. A "summarize this paragraph" prompt does not need few-shot examples and an output schema. |
| **Diagnose before adding more** | Prompts that are growing longer with each iteration rather than more precise. If you have re-prompted five times and the output is still wrong, diagnose the failure type before adding more text. |

| Подход | Когда применяется |
|---|---|
| **Сочетайте все четыре техники** | Все четыре техники применяются вместе при чётко определённом контракте вывода. Задачи с хорошо специфицированными форматами и пограничными случаями, которые можно покрыть примерами. |
| **Упростите промпт** | Все четыре техники добавлены к простой задаче, которой достаточно одной. Промпту «кратко перескажи этот абзац» не нужны ни примеры few-shot, ни схема вывода (output schema). |
| **Диагностируйте, прежде чем добавлять** | Промпты, которые с каждой итерацией становятся длиннее, а не точнее. Если вы переписали промпт пять раз, а вывод всё ещё неверный, определите тип сбоя, прежде чем добавлять новый текст. |

## The four techniques
## Четыре техники

### System prompts
### Системные промпты

System prompts carry the behavioral contract for the whole session. Write them once and treat them as your persistent instruction layer. They define Claude’s role, the output format, and any rules that must not change between conversations.

> Системные промпты (system prompts) несут поведенческий контракт для всей сессии. Напишите их один раз и относитесь к ним как к постоянному слою инструкций. Они определяют роль Claude, формат вывода и любые правила, которые не должны меняться от диалога к диалогу.

### XML Tags
### XML-теги

XML tags are used when the prompt mixes inputs with instructions. A prompt that asks Claude to debug code using provided documentation is a good example; without tags, the code and the documentation look the same to Claude.

> XML-теги используются, когда в промпте смешаны входные данные и инструкции. Хороший пример — промпт, который просит Claude отладить код с помощью предоставленной документации: без тегов код и документация выглядят для Claude одинаково.

Wrap them with descriptive tag names like `<my_code>` and `<docs>` and the boundary becomes unambiguous. You do not need to use official XML tag names; descriptive names that match your content work best.

> Оберните их в теги с говорящими названиями, например `<my_code>` и `<docs>`, и граница станет однозначной. Использовать какие-то «официальные» имена XML-тегов не нужно: лучше всего работают описательные имена, соответствующие вашему содержимому.

### Few-shot Examples
### Примеры few-shot

Few-shot examples are considered useful because they show rather than just tell. Instead of trying to describe the exact format you want, you provide one correct input-output pair and let Claude infer the pattern. To use this, wrap examples using consistent XML structure, for instance `<sample_input>` and `<ideal_output>`, so the boundary between example and prompt is clear. You can use some examples from your highest-scoring evaluation outputs rather than writing them from scratch.

> Примеры few-shot считаются полезными, потому что они показывают, а не просто рассказывают. Вместо того чтобы пытаться описать точный нужный формат, вы даёте одну правильную пару «вход — выход» и позволяете Claude самому вывести шаблон. Для этого оборачивайте примеры в единообразную XML-структуру, например `<sample_input>` и `<ideal_output>`, чтобы граница между примером и промптом была чёткой. Вместо того чтобы писать примеры с нуля, можно взять некоторые из выводов, получивших наивысшие оценки в ваших eval-ах.

### Output constraints
### Ограничения на вывод

Output constraints are the last line of defense before Claude’s response reaches your parser. You should specify exactly what you need, including field names, types, length limits, whether to include preamble, and what to do when data is absent. Use structured output features in cases when the format must be machine-readable.

> Ограничения на вывод (output constraints) — последняя линия обороны, прежде чем ответ Claude попадёт в ваш парсер. Точно укажите, что вам нужно: имена полей, типы, ограничения по длине, нужна ли вводная часть (preamble) и что делать, если данные отсутствуют. Если формат должен быть машиночитаемым, используйте функции структурированного вывода (structured output).

## The iteration loop: Diagnosing before re-prompting
## Цикл итераций: диагностика перед повторным промптингом

When a first-pass response misses the mark, the instinct is to add more words to the prompt and try again. That instinct almost always makes the problem harder to diagnose and rarely fixes it.

> Когда ответ с первой попытки не попадает в цель, инстинктивно хочется добавить в промпт больше слов и попробовать снова. Такой подход почти всегда затрудняет диагностику проблемы и редко её решает.

Instead, diagnose the problem first, and then re-prompt based on your findings. The failure type tells you which technique is missing:

- **Wrong format:** This is caused due to a missing output constraint. The prompt never specified what shape the result should take.
- **Wrong content or scope drift:** This is caused due to an underspecified system prompt; the behavioral contract was too vague to hold across conversations.
- **Correct task but hallucinated structure:** This happens when few-shot examples are needed. Claude cannot infer the exact structure from a description alone.
- **Good output on simple inputs but breaks on edge cases:** The prompt handles the happy path but has no constraint covering the variant the parser breaks on.

> Вместо этого сначала диагностируйте проблему, а затем переписывайте промпт с учётом выводов. Тип сбоя подсказывает, какой техники не хватает:
>
> - **Неверный формат:** причина — отсутствие ограничения на вывод (output constraint). В промпте не было указано, какую форму должен иметь результат.
> - **Неверное содержание или расползание рамок задачи (scope drift):** причина — недостаточно конкретный системный промпт; поведенческий контракт был слишком размытым, чтобы сохраняться от диалога к диалогу.
> - **Задача выполнена верно, но структура выдумана (hallucinated structure):** так бывает, когда нужны примеры few-shot. Claude не может вывести точную структуру только из описания.
> - **Хороший вывод на простых входных данных, но сбои на пограничных случаях (edge cases):** промпт обрабатывает «счастливый путь» (happy path), но не содержит ограничения, покрывающего вариант, на котором ломается парсер.

The fix is structural, not a matter of phrasing. For example, if Claude is ignoring a boundary between your instructions and your content, clearer wording will not fix it, and if the output format keeps drifting, saying "please format this correctly" will not fix it either. In each case, identify which of the four techniques is absent and add it.

> Исправление носит структурный характер, а не сводится к формулировкам. Например, если Claude игнорирует границу между вашими инструкциями и вашим содержимым, более ясная формулировка этого не исправит, а если формат вывода постоянно «плывёт», фраза «пожалуйста, отформатируй это правильно» тоже не поможет. В каждом случае определите, какой из четырёх техник не хватает, и добавьте её.

## Moving output control from the prompt into the API with structured outputs
## Перенос контроля над выводом из промпта в API с помощью structured outputs

Everything up to this point shapes the output by writing instructions into the prompt and hoping Claude follows them. That works most of the time, but the prompt is a request, so a model can still return a stray sentence, a wrong field name, or malformed JSON that breaks the parser downstream.

> Всё, что рассматривалось до сих пор, формирует вывод за счёт инструкций в промпте в надежде, что Claude будет им следовать. В большинстве случаев это работает, но промпт — это лишь просьба, поэтому модель всё равно может вернуть лишнее предложение, неверное имя поля или некорректный JSON, который сломает последующий парсер.

The Claude API has a separate mechanism that removes that gap for production code. It is called structured outputs, and instead of asking for a shape in words, you hand the API a JSON schema, and the model is constrained at generation time to produce output that matches it. This technique is constrained decoding: as Claude generates each token, the API only allows tokens that keep the output valid against your schema, so a response that violates the schema cannot be produced in the first place.

> В Claude API есть отдельный механизм, который устраняет этот пробел для продакшен-кода. Он называется structured outputs (структурированный вывод): вместо того чтобы просить нужную форму словами, вы передаёте API JSON-схему, и модель во время генерации ограничивается так, чтобы выдавать соответствующий ей вывод. Эта техника называется ограниченным декодированием (constrained decoding): по мере того как Claude генерирует каждый токен, API допускает только те токены, при которых вывод остаётся валидным относительно вашей схемы, поэтому ответ, нарушающий схему, в принципе не может быть сгенерирован.

Structured outputs cover two situations that show up in real pipelines. Each one constrains a different part of what the model returns, and you can use them on their own or together in the same request.

- **JSON outputs constrain the final response.** You set the `output_config.format` parameter with type `json_schema` and your schema, and Claude returns valid JSON in the response text that matches that schema every time. Reach for this when the model itself is producing the structured payload your code consumes, like extracting fields from a support ticket or formatting an API response, because it removes the parse-and-retry code you would otherwise write around every call.
- **Strict tool use constrains the inputs Claude passes to your tools.** You set `strict` to `true` on a tool definition, and the arguments Claude sends to that tool are validated against the input schema before your code runs. Reach for this in agentic loops where a malformed tool argument would crash the function or trigger a wrong action; this helps guarantee the call your code receives already conforms to the contract you defined.

> Structured outputs охватывают две ситуации, которые встречаются в реальных пайплайнах. Каждая из них ограничивает свою часть того, что возвращает модель, и их можно использовать по отдельности или вместе в одном запросе.
>
> - **JSON outputs ограничивают итоговый ответ.** Вы задаёте параметр `output_config.format` с типом `json_schema` и своей схемой, и Claude каждый раз возвращает в тексте ответа валидный JSON, соответствующий этой схеме. Используйте это, когда модель сама формирует структурированные данные (payload), которые потребляет ваш код, например при извлечении полей из обращения в поддержку или форматировании ответа API, — это избавляет от кода «распарсить и повторить» (parse-and-retry), который иначе пришлось бы писать вокруг каждого вызова.
> - **Strict tool use ограничивает входные данные, которые Claude передаёт вашим инструментам.** Вы устанавливаете `strict` в `true` в определении инструмента (tool definition), и аргументы, которые Claude отправляет этому инструменту, проверяются по входной схеме до запуска вашего кода. Используйте это в агентных циклах (agentic loops), где некорректный аргумент инструмента привёл бы к падению функции или запуску неверного действия; это помогает гарантировать, что вызов, получаемый вашим кодом, уже соответствует заданному вами контракту.

The reason this belongs in the production code and not just in the prompt is because of reliability under inputs you did not test. A prompt-level instruction to return only JSON holds on the cases you tried and then slips on an edge case you did not, which is the exact failure the earlier classification example walked through. A schema constraint does not slip, because the API enforces it on every token rather than trusting the model to remember the instruction. That moves output correctness from something you verify after the fact to something the API rules out before it happens.

> Это место в продакшен-коде, а не только в промпте, из-за надёжности на входных данных, которые вы не тестировали. Инструкция в промпте «возвращай только JSON» соблюдается на проверенных вами случаях, а затем даёт сбой на пограничном случае, который вы не проверили, — ровно тот сбой, который разбирался в примере с классификацией выше. Ограничение схемой не даёт сбоев, потому что API применяет его к каждому токену, а не полагается на то, что модель запомнит инструкцию. Тем самым корректность вывода из того, что вы проверяете постфактум, превращается в то, что API исключает заранее.

Constraining generation has costs, and a developer choosing this in production needs to weigh them rather than enabling it everywhere by default. Below are some of those costs you must consider:

- **The first request on a new schema is slower.** The API compiles your schema into a grammar before it can constrain output, and that compilation adds latency on the first call. Compiled grammars are cached for 24 hours from last use, so steady traffic on a stable schema pays the cost once, but a workload that changes schemas constantly pays it repeatedly.
- **Your input token count rises.** When structured outputs are on, the API adds a system prompt describing the expected format, and that injected prompt is billed like any other input token. The increase is small per call, but it is worth knowing when you are estimating cost at volume.
- **A guaranteed schema is not a guaranteed success.** Two cases still return output that does not match: a refusal, where the model declines for safety reasons and the response carries `stop_reason` `refusal`, and a truncation, where the response hits the `max_tokens` limit and stops mid-structure with `stop_reason` `max_tokens`. Your code still checks `stop_reason` rather than assuming every response parses.
- **It does not combine with message prefilling.** JSON outputs and prefilling the assistant message are incompatible, so a pattern that starts the response for Claude and a pattern that constrains the whole response to a schema cannot run on the same request. Pick the one that fits the task.

> Ограничение генерации имеет свою цену, и разработчику, выбирающему этот подход в продакшене, нужно взвесить издержки, а не включать его везде по умолчанию. Ниже перечислены издержки, которые необходимо учитывать:
>
> - **Первый запрос с новой схемой выполняется медленнее.** Прежде чем ограничивать вывод, API компилирует вашу схему в грамматику, и эта компиляция добавляет задержку при первом вызове. Скомпилированные грамматики кэшируются на 24 часа с момента последнего использования, поэтому при стабильном трафике на неизменной схеме эта цена платится один раз, а нагрузка, в которой схемы постоянно меняются, платит её снова и снова.
> - **Растёт число входных токенов.** Когда structured outputs включены, API добавляет системный промпт с описанием ожидаемого формата, и этот добавленный промпт тарифицируется как любые другие входные токены. Прирост на один вызов невелик, но о нём стоит знать при оценке стоимости на больших объёмах.
> - **Гарантированная схема — не гарантированный успех.** В двух случаях вывод всё равно не соответствует схеме: при отказе (refusal), когда модель отказывается отвечать по соображениям безопасности и ответ содержит `stop_reason` `refusal`, и при обрезке (truncation), когда ответ упирается в лимит `max_tokens` и обрывается посреди структуры с `stop_reason` `max_tokens`. Поэтому ваш код по-прежнему проверяет `stop_reason`, а не исходит из того, что любой ответ успешно распарсится.
> - **Не сочетается с предзаполнением сообщения (message prefilling).** JSON outputs и предзаполнение сообщения ассистента несовместимы, поэтому паттерн, при котором вы начинаете ответ за Claude, и паттерн, ограничивающий весь ответ схемой, нельзя использовать в одном запросе. Выберите тот, что подходит под задачу.
