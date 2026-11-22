# Tool Schemas Claude Selects Correctly: Definition, Loop, and Calling Patterns
# Схемы инструментов, которые Claude выбирает правильно: определение, цикл и паттерны вызова

So far, the work has been about shaping what Claude produces: framing the request, giving examples, picking the technique that fits the output you want. With tool-use, you’re not steering language toward a good answer anymore, you’re handing Claude a set of actions and trusting it to pick the right one; that pick is driven almost entirely by what you wrote in the schema.

> До сих пор работа сводилась к тому, чтобы формировать то, что выдаёт Claude: правильно сформулировать запрос, дать примеры, выбрать технику, подходящую под нужный вывод. С использованием инструментов (tool use) вы уже не направляете текст к хорошему ответу — вы передаёте Claude набор действий и доверяете ему выбрать правильное; и этот выбор почти полностью определяется тем, что вы написали в схеме.

## How the tool-use loop works
## Как работает цикл tool use

The most common misconception about tool-use is that Claude runs the tools. Instead, Claude reads your tool definitions, decides which one fits the situation, and tells your application what to call it along with the required inputs. Your application executes the tool, gets the result, and sends it back; then Claude uses that result to continue.

> Самое распространённое заблуждение о tool use — что Claude сам запускает инструменты. На самом деле Claude читает ваши определения инструментов (tool definitions), решает, какой из них подходит к ситуации, и сообщает вашему приложению, что нужно вызвать и с какими входными данными. Ваше приложение выполняет инструмент, получает результат и отправляет его обратно; затем Claude использует этот результат, чтобы продолжить работу.

This back-and-forth shouldn’t be ignored in production: if your application does not handle the return correctly, Claude never gets the data it asked for, and the loop breaks. The boundary between what Claude owns and what your code owns is where most tool-use bugs live. Here is the sequence to ensure proper implementation of tool-use.

> Этот обмен в обе стороны нельзя игнорировать в продакшене: если ваше приложение неправильно обработает возврат результата, Claude так и не получит запрошенные данные, и цикл прервётся. Именно на границе между тем, за что отвечает Claude, и тем, за что отвечает ваш код, живёт большинство ошибок tool use. Ниже приведена последовательность шагов, обеспечивающая правильную реализацию tool use.

### The tool-use loop, step by step
### Цикл tool use по шагам

1. **Define schema.** You define a schema with a name, a description, and an input schema. Claude reads this to decide whether and when to call the tool.
2. **Send message.** Your code sends a message to Claude including the tool definitions and the user's input.
3. **`tool_use` block.** Claude issues a tool-use block containing the tool name, a unique ID, and the input arguments it wants to pass. The API response comes back with `stop_reason: tool_use`.
4. **Execute tool.** Your code executes the tool using those arguments. Note that the assistant turn has already ended (Claude is not holding a connection open or waiting on your server). The model is stateless between calls. To continue, your code makes a fresh API request containing the prior messages plus the tool result.
5. **Return result.** You return the result in a tool-result block that references the original tool-use ID.
6. **Claude continues.** Claude continues using the tool result as context for its next response, either another tool-use block or a final end turn.

> 1. **Определите схему.** Вы задаёте схему с именем, описанием и схемой входных данных (input schema). Claude читает её, чтобы решить, нужно ли вызывать инструмент и когда.
> 2. **Отправьте сообщение.** Ваш код отправляет Claude сообщение, включающее определения инструментов и входные данные пользователя.
> 3. **Блок `tool_use`.** Claude выдаёт блок `tool_use`, содержащий имя инструмента, уникальный ID и входные аргументы, которые он хочет передать. Ответ API возвращается со значением `stop_reason: tool_use`.
> 4. **Выполните инструмент.** Ваш код выполняет инструмент с этими аргументами. Обратите внимание, что ход ассистента к этому моменту уже завершён (Claude не держит соединение открытым и не ждёт ваш сервер). Между вызовами модель не хранит состояние (stateless). Чтобы продолжить, ваш код делает новый запрос к API, содержащий предыдущие сообщения плюс результат инструмента.
> 5. **Верните результат.** Вы возвращаете результат в блоке `tool_result`, который ссылается на исходный ID вызова `tool_use`.
> 6. **Claude продолжает.** Claude продолжает работу, используя результат инструмента как контекст для следующего ответа: это либо ещё один блок `tool_use`, либо финальное завершение хода (end turn).

It’s important to note that the loop is not automatic and you need to complete the fourth step. If the miss is systematic, the fix is in the schema definition step.

> Важно помнить, что цикл не выполняется автоматически, и четвёртый шаг должны выполнить вы. Если промах систематический, исправлять нужно на шаге определения схемы.

## Message block structure in a tool-use conversation
## Структура блоков сообщений в диалоге с tool use

A tool-use conversation is built out of structured blocks, not plain text. Each assistant turn and user turn is a list of blocks, and four block types do the work in a tool-use session. A text block carries Claude’s prose response. A `tool_use` block carries a tool call, including the tool name, a unique ID, and the input arguments. A `tool_result` block carries what your code returned after running the tool. A thinking block carries Claude’s internal reasoning, and it only appears when extended thinking is enabled.

> Диалог с tool use строится из структурированных блоков, а не из простого текста. Каждый ход ассистента и ход пользователя — это список блоков, и в сессии с tool use работу выполняют четыре типа блоков. Блок `text` содержит текстовый ответ Claude. Блок `tool_use` содержит вызов инструмента, включая имя инструмента, уникальный ID и входные аргументы. Блок `tool_result` содержит то, что вернул ваш код после выполнения инструмента. Блок `thinking` содержит внутреннее рассуждение Claude и появляется только при включённом extended thinking.

The API enforces a specific pairing between these blocks. Every `tool_use` block in an assistant turn must be answered by a `tool_result` block with a matching ID in the user turn that immediately follows. If the IDs don’t match, if the result is missing, or if the turns are out of order, the request fails validation. This is not something you can fix by adjusting your prompt; it’s structural, and your code has to produce the sequence correctly on every request.

> API требует строгого соответствия между этими блоками. На каждый блок `tool_use` в ходе ассистента должен быть дан ответ блоком `tool_result` с совпадающим ID в ходе пользователя, который следует непосредственно за ним. Если ID не совпадают, результат отсутствует или ходы идут не в том порядке, запрос не проходит валидацию. Это нельзя исправить корректировкой промпта: требование структурное, и ваш код должен формировать правильную последовательность в каждом запросе.

The table below summarizes each block type, what it contains, and the rule that governs how your code must handle it.

> В таблице ниже кратко описаны все типы блоков, их содержимое и правило, определяющее, как ваш код должен их обрабатывать.

| Block type | Role | Contains | Critical rule |
|---|---|---|---|
| `text` block | Assistant/Claude | Claude’s prose output | Claude may return a text block alongside a `tool_use` block in the same turn. When it does, your code must preserve the full content array, including the text block, when appending that turn to conversation history. Dropping the text block corrupts the context Claude relies on for follow-up turns. |
| `tool_use` block | Assistant/Claude | The tool name, a unique ID, and the input arguments Claude wants passed to your function | Every `tool_use` block must be answered by a `tool_result` block in the immediately following user turn. The `tool_result` must carry the same ID. Without that pairing, the API rejects the next request. |
| `tool_result` block | User | Matching `tool_use` ID, the result content, and an optional `is_error` flag set to true when the tool call fails | The `tool_use_id` value must match the original `tool_use` block exactly. Claude uses this ID to connect each result back to the call that produced it, which matters when a single assistant turn issues multiple tool calls and the results arrive in a different order. |
| `thinking` block | Assistant (extended thinking only)/Claude | Claude’s internal reasoning, visible only when extended thinking is enabled | The block must be passed back to the API unchanged in subsequent turns. The signature verifies the reasoning hasn’t been modified, so any edit or summary breaks the signature and the API rejects the message. Redacted thinking blocks follow the same rule: pass them back as received, even though the content is encrypted and not human-readable. |

| Тип блока | Роль | Содержимое | Ключевое правило |
|---|---|---|---|
| Блок `text` | Assistant/Claude | Текстовый вывод Claude | Claude может вернуть блок `text` вместе с блоком `tool_use` в одном и том же ходе. В этом случае ваш код, добавляя этот ход в историю диалога, должен сохранить весь массив content целиком, включая блок `text`. Удаление блока `text` искажает контекст, на который Claude опирается в последующих ходах. |
| Блок `tool_use` | Assistant/Claude | Имя инструмента, уникальный ID и входные аргументы, которые Claude хочет передать в вашу функцию | На каждый блок `tool_use` должен быть дан ответ блоком `tool_result` в непосредственно следующем ходе пользователя. `tool_result` должен содержать тот же ID. Без такой пары API отклонит следующий запрос. |
| Блок `tool_result` | User | Соответствующий ID из `tool_use`, содержимое результата и необязательный флаг `is_error`, который устанавливается в true при сбое вызова инструмента | Значение `tool_use_id` должно в точности совпадать с исходным блоком `tool_use`. По этому ID Claude связывает каждый результат с вызовом, который его породил; это важно, когда один ход ассистента содержит несколько вызовов инструментов, а результаты приходят в другом порядке. |
| Блок `thinking` | Assistant (только при extended thinking)/Claude | Внутреннее рассуждение Claude, видимое только при включённом extended thinking | Блок должен передаваться обратно в API без изменений в последующих ходах. Подпись (signature) подтверждает, что рассуждение не было изменено, поэтому любое редактирование или сокращение до краткого изложения нарушает подпись, и API отклоняет сообщение. Скрытые блоки thinking (redacted thinking blocks) подчиняются тому же правилу: передавайте их обратно в том виде, в каком получили, даже если их содержимое зашифровано и нечитаемо для человека. |

The critical invariant is that every `tool_use` block from an assistant turn must have a corresponding `tool_result` block in the immediately following user turn. Missing `tool_result` blocks, or `tool_result` blocks that appear in a later turn rather than the immediately following user turn, cause an API validation error.

> Критически важный инвариант: каждый блок `tool_use` из хода ассистента должен иметь соответствующий блок `tool_result` в непосредственно следующем ходе пользователя. Отсутствующие блоки `tool_result` или блоки `tool_result`, которые появляются в более позднем ходе, а не в непосредственно следующем ходе пользователя, вызывают ошибку валидации API.

## Schema anatomy: What Claude reads to make a tool selection decision
## Анатомия схемы: что читает Claude, принимая решение о выборе инструмента

A tool schema has three parts, including name, description, and input_schema. The description determines whether Claude selects the tool correctly or not.

1. **Name:** A short identifier that should be specific. For example, `get_account_balance` is more useful to Claude than `get_data`.
2. **Description:** A critical part that Claude reads to decide whether a tool is required or not. You should always write the description in two parts, including when to and when not to use the tool:
   - A description that says "use this to find information" will cause wrong selections because Claude cannot distinguish it from any other tool that retrieves something.
   - A description that says "use this to retrieve the current balance for a specific account ID and do not use this for transaction history" gives Claude an exclusion condition to work with and is appropriately descriptive.
3. **`input_schema`:** Defines the parameters (the inputs your tool function accepts) using JSON Schema.
   - You should mark parameters as required when Claude requires them to call the tool correctly.
   - You can mark parameters as optional when the tool can operate without them. Overlapping parameter types between tools is the most common source of wrong-tool calls.

> Схема инструмента состоит из трёх частей: `name`, `description` и `input_schema`. Именно описание определяет, правильно ли Claude выберет инструмент.
>
> 1. **Имя (`name`):** короткий идентификатор, который должен быть конкретным. Например, `get_account_balance` полезнее для Claude, чем `get_data`.
> 2. **Описание (`description`):** критически важная часть, которую Claude читает, чтобы решить, нужен ли инструмент. Описание всегда следует писать из двух частей: когда использовать инструмент и когда не использовать:
>    - Описание вида «используй это, чтобы найти информацию» приведёт к неверному выбору, потому что Claude не сможет отличить этот инструмент от любого другого, который что-то извлекает.
>    - Описание вида «используй это, чтобы получить текущий баланс для конкретного ID счёта, и не используй это для истории транзакций» даёт Claude условие исключения, на которое можно опереться, и является достаточно описательным.
> 3. **`input_schema`:** определяет параметры (входные данные, которые принимает функция инструмента) с помощью JSON Schema.
>    - Помечайте параметры как обязательные (required), если они нужны Claude для правильного вызова инструмента.
>    - Параметры можно помечать как необязательные (optional), если инструмент может работать без них. Пересекающиеся типы параметров у разных инструментов — самая частая причина вызова не того инструмента.

## Decision table: Schema design choices
## Таблица решений: выбор при проектировании схемы

The schema is what Claude reads to decide which tool to call, what arguments to pass in, and whether it has enough information to respond. A schema that’s vague, under-described, or missing required fields will produce tool calls that look syntactically correct but pick the wrong tool, pass malformed inputs, or loop unnecessarily. The five decisions below determine whether your implementation behaves predictably under real conditions. The table notes where sequential and parallel tool-calling diverge.

> Схема — это то, что Claude читает, чтобы решить, какой инструмент вызвать, какие аргументы передать и достаточно ли у него информации для ответа. Расплывчатая, недостаточно описанная схема или схема без обязательных полей приведёт к вызовам инструментов, которые синтаксически выглядят корректно, но выбирают не тот инструмент, передают некорректные входные данные или без необходимости зацикливаются. Пять решений ниже определяют, будет ли ваша реализация вести себя предсказуемо в реальных условиях. В таблице отмечено, где последовательный и параллельный вызов инструментов (sequential и parallel tool calling) различаются.

| Decision | How to handle it | Why it matters |
|---|---|---|
| **Subtask dependency** | When one tool’s output feeds the next, the calls have to run in sequence because the second call cannot be built until the first result comes back. When the subtasks are independent of each other, you can structure the tool set so Claude issues multiple `tool_use` blocks in a single turn and your code runs them concurrently. | This is the one decision that changes how you design the schema. Current Claude models default to parallel calls when calls are independent. Where a real dependency exists, model it as separate turns so the first result is available before the next call is built. Use `disable_parallel_tool_use` to force one tool call per turn if needed. |
| **Required fields** | Mark a field as required only when the call doesn’t make sense without it. Place these in the `required` array of the input schema. | Marking everything required forces Claude to fabricate values for fields it has no basis to fill in. The `required` array is how you tell Claude which inputs are non-negotiable. |
| **Optional fields** | Use optional fields for parameters with sensible defaults or where absence carries meaning. Leave them out of the `required` array and give them defaults in the function signature. | Optional fields let Claude omit information it doesn’t have, instead of guessing. If a field is optional but marked required, every call must invent a value, which can cause bad inputs. |
| **Description length** | Write three to four sentences per tool covering what it does, when Claude should reach for it, and what it returns. Include examples of valid inputs where format matters. | If the description is too short, Claude guesses because there isn’t enough signal to distinguish your tool from others. If the description is too long, the trigger conditions get buried under detail Claude doesn’t reference at decision time. |
| **Overlapping parameter types** | When two tools accept the same parameter shape, add disambiguating language to each description that names the domain or trigger the tool is meant for. | Claude routes on name plus description, with parameter types as a secondary signal. When signatures are identical, routing collapses to description alone, and similar-sounding descriptions become indistinguishable. |

| Решение | Как с этим работать | Почему это важно |
|---|---|---|
| **Зависимость подзадач** | Когда вывод одного инструмента используется как вход для следующего, вызовы должны выполняться последовательно, потому что второй вызов нельзя сформировать, пока не вернётся первый результат. Когда подзадачи независимы друг от друга, набор инструментов можно построить так, чтобы Claude выдавал несколько блоков `tool_use` за один ход, а ваш код выполнял их параллельно. | Это единственное решение, которое меняет сам подход к проектированию схемы. Текущие модели Claude по умолчанию делают параллельные вызовы, если вызовы независимы. Там, где реальная зависимость есть, моделируйте её отдельными ходами, чтобы первый результат был доступен до формирования следующего вызова. При необходимости используйте `disable_parallel_tool_use`, чтобы принудительно ограничить вызовы одним инструментом за ход. |
| **Обязательные поля** | Помечайте поле как обязательное, только если без него вызов не имеет смысла. Такие поля указываются в массиве `required` схемы входных данных. | Если пометить обязательными все поля, Claude будет вынужден выдумывать значения для полей, которые ему не на чем основать. Массив `required` — это способ сообщить Claude, какие входные данные обязательны без исключений. |
| **Необязательные поля** | Используйте необязательные поля для параметров с разумными значениями по умолчанию или там, где само отсутствие значения несёт смысл. Не включайте их в массив `required` и задайте им значения по умолчанию в сигнатуре функции. | Необязательные поля позволяют Claude опустить информацию, которой у него нет, вместо того чтобы угадывать. Если поле по сути необязательное, но помечено как обязательное, в каждом вызове придётся придумывать значение, что может приводить к некорректным входным данным. |
| **Длина описания** | Пишите по три-четыре предложения на инструмент: что он делает, когда Claude следует к нему обращаться и что он возвращает. Если формат важен, приводите примеры допустимых входных данных. | Если описание слишком короткое, Claude приходится угадывать, потому что сигнала недостаточно, чтобы отличить ваш инструмент от других. Если описание слишком длинное, условия срабатывания теряются среди деталей, на которые Claude не опирается в момент принятия решения. |
| **Пересекающиеся типы параметров** | Когда два инструмента принимают параметры одинаковой формы, добавьте в описание каждого из них уточняющие формулировки, называющие предметную область или условие срабатывания, для которых предназначен инструмент. | Claude выбирает инструмент по имени и описанию, а типы параметров служат второстепенным сигналом. Когда сигнатуры идентичны, выбор сводится только к описанию, и похожие по смыслу описания становятся неразличимыми. |

## Worked example: A schema that causes wrong-tool selection and the fix
## Разобранный пример: схема, из-за которой выбирается не тот инструмент, и её исправление

This is an illustrative example based on common patterns observed in tool-use implementations. Tool names, descriptions, and test results are constructed to demonstrate the selection-disambiguation principle, not drawn from a specific production system.

> Это иллюстративный пример, основанный на типичных паттернах, встречающихся в реализациях tool use. Имена инструментов, описания и результаты тестов составлены для демонстрации принципа устранения неоднозначности при выборе (selection disambiguation), а не взяты из конкретной продакшен-системы.

A developer registers two tools, including `search_knowledge_base` and `get_cached_result`. The tool names are distinct, but Claude’s tool selection weighs descriptions heavily; when descriptions overlap, name alone is not sufficient to disambiguate. Both have descriptions that start with "use this to find information." Without exclusion conditions, Claude frequently selected the wrong tool on ambiguous inputs during development testing.

> Разработчик регистрирует два инструмента: `search_knowledge_base` и `get_cached_result`. Имена инструментов различаются, но при выборе инструмента Claude придаёт описаниям большой вес; когда описания пересекаются, одного имени недостаточно, чтобы устранить неоднозначность. Описания обоих инструментов начинаются со слов «use this to find information» («используй это, чтобы найти информацию»). Без условий исключения (exclusion conditions) Claude во время тестирования при разработке часто выбирал не тот инструмент на неоднозначных входных данных.

The problem is that both descriptions look identical to Claude at the point where the selection decision is made. The fix is adding an additional sentence per description:

> Проблема в том, что в момент принятия решения о выборе оба описания выглядят для Claude одинаково. Исправление — добавить в каждое описание ещё одно предложение:

**✅ `search_knowledge_base`**

```text
"Use this to search the knowledge base when the user asks a question that requires looking up current information. Do not use this if the result of a prior search in this session already covers the question."
```

> *Перевод:* «Используй это для поиска в базе знаний, когда пользователь задаёт вопрос, требующий поиска актуальной информации. Не используй это, если результат предыдущего поиска в этой сессии уже отвечает на вопрос.»

**✅ `get_cached_result`**

```text
"Use this to retrieve a result that was already fetched during this session. Only use this if search_knowledge_base was called earlier in this conversation for the same query."
```

> *Перевод:* «Используй это, чтобы получить результат, который уже был получен в ходе этой сессии. Используй это, только если `search_knowledge_base` уже вызывался ранее в этом диалоге для того же запроса.»

The exclusion conditions give Claude a decision rule rather than two identical-looking options. These conditions rely on complete conversation history being passed in each request. If prior turns are truncated or dropped, Claude cannot evaluate them and the exclusion logic silently fails.

> Условия исключения дают Claude правило принятия решения вместо двух одинаково выглядящих вариантов. Эти условия рассчитаны на то, что в каждом запросе передаётся полная история диалога. Если предыдущие ходы обрезаны или удалены, Claude не может их оценить, и логика исключения незаметно перестаёт работать.

Every additional tool you register increases the surface area Claude has to reason over, so this discipline only pays off when the underlying tools are distinct. The table below shows where exclusion-condition disambiguation helps and where a different approach is warranted.

> Каждый дополнительный зарегистрированный инструмент увеличивает пространство, в котором Claude приходится рассуждать, поэтому такой подход окупается, только когда сами инструменты действительно различны. В таблице ниже показано, где помогает устранение неоднозначности с помощью условий исключения, а где нужен другой подход.

| | |
|---|---|
| **Handles well** | Routing Claude to the right tool reliably when descriptions are specific and exclusion conditions are stated. |
| **Poor fit** | Two tools that do similar things and need ever-longer descriptions to keep apart: at that point, merge them into one tool with a type parameter instead. |

| | |
|---|---|
| **Хорошо подходит** | Надёжная маршрутизация Claude к нужному инструменту, когда описания конкретны и условия исключения сформулированы явно. |
| **Плохо подходит** | Два инструмента, которые делают похожие вещи и требуют всё более длинных описаний, чтобы их различать: в этом случае лучше объединить их в один инструмент с параметром типа (type parameter). |

## When someone else has already written your tools: MCP as an alternative to manual schema authoring
## Когда инструменты уже написал кто-то другой: MCP как альтернатива ручному написанию схем

Everything in the previous sections assumes you are writing the tool schemas yourself: name, description, input_schema, and the function that executes when Claude issues a tool_use block. For many integrations, you do not need to do that. The Model Context Protocol, MCP, is a standardized communication layer that moves tool definitions and execution out of your application code and into dedicated servers. When an MCP server exists for the service you want to reach, you can connect directly to the MCP server rather than building the integration yourself.

> Всё, о чём говорилось в предыдущих разделах, предполагает, что вы пишете схемы инструментов сами: `name`, `description`, `input_schema` и функцию, которая выполняется, когда Claude выдаёт блок `tool_use`. Для многих интеграций этого делать не нужно. Model Context Protocol (MCP) — это стандартизированный коммуникационный слой, который выносит определения инструментов и их выполнение из кода вашего приложения на отдельные серверы. Если для сервиса, к которому вы хотите обращаться, существует MCP-сервер, можно подключиться к нему напрямую, а не строить интеграцию самостоятельно.

Take a GitHub integration as a concrete case. GitHub exposes repositories, pull requests, issues, projects, and more. To build a complete integration using the tool schema approach from this module, you would need to write a schema and an execution function for every piece of that functionality and maintain it as GitHub’s API evolves. An MCP server for GitHub has already done that. So, your application connects to the server, receives the full list of available tools, and Claude selects among them using the same description-based routing you have already been working with. The underlying mechanism is identical, but what changes is who wrote it and who owns the tool definitions.

> Возьмём в качестве конкретного примера интеграцию с GitHub. GitHub предоставляет репозитории, pull requests, issues, проекты и многое другое. Чтобы построить полную интеграцию на основе подхода со схемами инструментов из этого модуля, пришлось бы написать схему и функцию выполнения для каждой части этой функциональности и поддерживать их по мере развития API GitHub. MCP-сервер для GitHub уже сделал это. Поэтому ваше приложение подключается к серверу, получает полный список доступных инструментов, и Claude выбирает среди них, используя ту же маршрутизацию на основе описаний, с которой вы уже работали. Базовый механизм тот же самый, меняется лишь то, кто написал определения инструментов и кто ими владеет.

### How MCP fits into the tool-use loop
### Как MCP встраивается в цикл tool use

The loop you built earlier in this module does not change when you introduce MCP. Claude still issues a `tool_use` block, your application still executes the tool and returns a `tool_result`, and the message block pairing rules still apply. The difference is in the setup step. Instead of registering schemas you wrote, your MCP client sends a `ListToolsRequest` to the MCP server, receives the full tool list back, and passes those definitions to Claude. From Claude’s perspective, those tools are indistinguishable from ones you authored manually.

> Цикл, который вы построили ранее в этом модуле, не меняется при появлении MCP. Claude по-прежнему выдаёт блок `tool_use`, ваше приложение по-прежнему выполняет инструмент и возвращает `tool_result`, и правила парности блоков сообщений по-прежнему действуют. Разница — на этапе настройки. Вместо регистрации написанных вами схем ваш MCP-клиент отправляет MCP-серверу `ListToolsRequest`, получает в ответ полный список инструментов и передаёт эти определения Claude. С точки зрения Claude эти инструменты ничем не отличаются от написанных вручную.

One practical implication worth noting: MCP servers add tool definitions to the context window even when the tools are not being used in the current turn. If you connect several servers at once, the tool definitions themselves consume budget before the first message arrives. The schema design discipline from earlier in this module applies here too. Register only the servers you are actively using, and check context cost against your window limit if you are connecting multiple servers in the same session.

> Стоит отметить одно практическое следствие: MCP-серверы добавляют определения инструментов в контекстное окно, даже если эти инструменты не используются в текущем ходе. Если подключить сразу несколько серверов, сами определения инструментов расходуют бюджет ещё до прихода первого сообщения. Здесь тоже применимы принципы проектирования схем, рассмотренные ранее в этом модуле. Регистрируйте только те серверы, которые действительно используете, и сверяйте стоимость контекста с лимитом окна, если подключаете несколько серверов в одной сессии.

If you are using the API MCP Connector, you control loading cost through an `mcp_toolset` object in the tools array. The `mcp_toolset` carries a `default_config` block that applies to every tool on the server, and you can override individual tools through `configs` keyed by tool name. Two settings matter for context cost:

- The `defer_loading` boolean, set inside `default_config` or a per-tool entry in `configs`, delays loading a tool definition until the model needs it, which reduces upfront context cost when you connect a server with a large tool list.
- The `enabled` boolean turns individual tools on or off, so you can register a server but expose only the tools you want the model to see. The MCP Connector requires the `mcp-client-2025-11-20` beta header to be set on the request.

Without that header, the `mcp_toolset` configuration will not apply as described here.

> Если вы используете API MCP Connector, стоимость загрузки контролируется через объект `mcp_toolset` в массиве `tools`. `mcp_toolset` содержит блок `default_config`, который применяется ко всем инструментам сервера, а отдельные инструменты можно переопределить через `configs` с ключами по имени инструмента. На стоимость контекста влияют две настройки:
>
> - Булев параметр `defer_loading`, задаваемый внутри `default_config` или в записи для конкретного инструмента в `configs`, откладывает загрузку определения инструмента до тех пор, пока он не понадобится модели, что снижает начальную стоимость контекста при подключении сервера с большим списком инструментов.
> - Булев параметр `enabled` включает или выключает отдельные инструменты, так что можно зарегистрировать сервер, но открыть модели только те инструменты, которые вы хотите ей показать. Для MCP Connector в запросе должен быть установлен beta-заголовок `mcp-client-2025-11-20`.
>
> Без этого заголовка конфигурация `mcp_toolset` не будет применяться так, как описано здесь.

The other piece worth knowing at this stage is how the client actually talks to the server. MCP runs over one of two transports, and which one you use depends on where the server lives. Local servers use stdio and your application spawns the server as a subprocess and communicates over standard input and output. Remote servers use Streamable HTTP and your application connects over the network via HTTP, using POST for client-to-server messages and an optional GET-based SSE stream for server-initiated messages. An older SSE-only transport exists but is deprecated, and new integrations should use Streamable HTTP. One constraint worth flagging if you are using Anthropic’s MCP connector in the API: only HTTP-exposed servers are supported through the connector, and stdio servers require managing the MCP client connection yourself via the SDK. Once the connection is established and tool definitions are received, your application code treats both transports identically.

> Ещё одна вещь, которую стоит знать на этом этапе, — как клиент на самом деле общается с сервером. MCP работает поверх одного из двух транспортов, и выбор зависит от того, где находится сервер. Локальные серверы используют `stdio`: ваше приложение запускает сервер как подпроцесс и обменивается с ним данными через стандартный ввод и вывод. Удалённые серверы используют Streamable HTTP: ваше приложение подключается по сети через HTTP, используя POST для сообщений от клиента к серверу и необязательный SSE-поток на основе GET для сообщений, инициированных сервером. Существует и более старый транспорт только на SSE, но он устарел (deprecated), и новые интеграции должны использовать Streamable HTTP. Одно ограничение стоит отметить, если вы используете MCP Connector от Anthropic в API: через коннектор поддерживаются только серверы, доступные по HTTP, а для серверов `stdio` подключение MCP-клиента нужно организовывать самостоятельно через SDK. После того как соединение установлено и определения инструментов получены, код вашего приложения работает с обоими транспортами одинаково.

**Use an existing MCP server when:** A well-maintained MCP server already exists for the service you need (check that it covers the specific operations you require and is actively maintained against the service’s current API. Writing and owning those schemas yourself adds implementation overhead for no additional capability. Note that the Claude API MCP Connector only supports remote servers. Local stdio servers require Claude Desktop or Claude Code as the client; they cannot be connected directly through the API.

> **Используйте готовый MCP-сервер, когда:** для нужного вам сервиса уже существует хорошо поддерживаемый MCP-сервер (проверьте, что он покрывает нужные вам операции и активно поддерживается в соответствии с текущим API сервиса). Самостоятельное написание и сопровождение этих схем добавляет затрат на реализацию, не давая никаких дополнительных возможностей. Обратите внимание, что Claude API MCP Connector поддерживает только удалённые серверы. Для локальных серверов `stdio` в качестве клиента нужны Claude Desktop или Claude Code; напрямую через API их подключить нельзя.

**Write the schemas yourself when:** No MCP server covers your use case, or when you need precise control over tool scope and description quality that a general-purpose server does not provide. Before defaulting to manual schemas for scope control, note that the API MCP Connector supports allowlisting and denylisting specific tools per server via `MCPToolset` configuration. Manual authoring may still be warranted for description quality, but not always for scope.

> **Пишите схемы сами, когда:** ни один MCP-сервер не покрывает ваш сценарий или когда вам нужен точный контроль над набором инструментов (scope) и качеством описаний, которого сервер общего назначения не даёт. Прежде чем по умолчанию переходить на ручные схемы ради контроля набора инструментов, учтите, что API MCP Connector поддерживает разрешающие и запрещающие списки (allowlisting и denylisting) конкретных инструментов для каждого сервера через конфигурацию `MCPToolset`. Ручное написание может быть оправдано ради качества описаний, но не всегда ради контроля набора инструментов.

**Combine both:** Connect to an MCP server for breadth then apply the description-tuning discipline from earlier in this module to the specific tools you are actively routing to. MCP and manual schema authoring are not mutually exclusive as the server gives you coverage, and your descriptions give you precision where it matters. Apply tool allowlisting via `MCPToolset` to limit the surface area Claude reasons over before layering in description tuning. Narrowing the tool set and sharpening the descriptions are two separate levers, and you should use both.

> **Сочетайте оба подхода:** подключитесь к MCP-серверу ради широты охвата, а затем примените рассмотренные ранее в этом модуле принципы настройки описаний к тем инструментам, на которые вы активно направляете запросы. MCP и ручное написание схем не исключают друг друга: сервер даёт вам охват, а ваши описания — точность там, где она важна. Прежде чем браться за настройку описаний, ограничьте с помощью allowlisting через `MCPToolset` пространство, в котором рассуждает Claude. Сужение набора инструментов и уточнение описаний — два отдельных рычага, и стоит использовать оба.
