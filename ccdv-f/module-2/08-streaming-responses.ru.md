# Streaming responses and handling partial output without corrupting state
# Потоковые ответы и обработка частичного вывода без повреждения состояния

Every request so far has waited for the whole response to arrive before doing anything with it. That's fine, until the response is long, or a user is sitting there staring at a blank screen. Streaming sends the response in pieces, sending them along as the model generates them. That makes things feel faster, but it also gives your code a new job: now you are tasked with assembling the final content yourself based on the series of outputs, and you need to be prepared if the series stops early.

> До сих пор каждый запрос ждал, пока придёт весь ответ целиком, и только потом что-то с ним делал. Это нормально — до тех пор, пока ответ не становится длинным или пользователь не сидит, глядя на пустой экран. Стриминг (streaming) отправляет ответ частями, передавая их по мере того, как модель их генерирует. Благодаря этому всё ощущается быстрее, но у вашего кода появляется новая задача: теперь вы сами должны собирать итоговое содержимое из последовательности фрагментов и быть готовы к тому, что эта последовательность оборвётся раньше времени.

## What streaming changes about the response
## Что стриминг меняет в ответе

In a non-streamed request, the API hands you one complete message with every content block, fully formed. In a streamed request, the API instead sends a series of events that describe the message as it's being built. Your code listens to that series and reassembles the blocks. The message you end up with is identical to what a non-streamed call would have given you, but the difference is that you have to assemble the pieces, and you decide what to do if the events stop before the message is finished.

> В запросе без стриминга API возвращает одно готовое сообщение, в котором все блоки содержимого полностью сформированы. В потоковом запросе API вместо этого отправляет последовательность событий (events), описывающих сообщение по мере его построения. Ваш код слушает эту последовательность и заново собирает блоки. Итоговое сообщение идентично тому, что вернул бы вызов без стриминга, но разница в том, что собирать части приходится вам, и вы же решаете, что делать, если события прекратятся до завершения сообщения.

It helps to know what's not happening: the model isn't holding some live object open for you. Each event is its own small message describing a single change, a block started, some text or input got added to it, a block finished, the whole message finished. Your handler takes each event and applies it to the partial state it's been building up.

> Полезно понимать, чего здесь не происходит: модель не держит для вас открытым некий «живой» объект. Каждое событие — это отдельное небольшое сообщение, описывающее одно изменение: начался блок, в него добавился текст или входные данные, блок завершился, завершилось всё сообщение. Ваш обработчик (handler) принимает каждое событие и применяет его к частичному состоянию, которое он постепенно накапливает.

## The event sequence, and what your handler does with each
## Последовательность событий и что ваш обработчик делает с каждым из них

| Event | What it signals | What your handler does |
|---|---|---|
| `message_start` | A new message is beginning. Carries the message shell with empty content and initial usage. | Set up an empty content array to collect blocks in. |
| `content_block_start` | A new content block is opening, with its type (`text`, `tool_use`, or `thinking`) and index. | Make a slot at that index for the named block type. A `tool_use` block opens with its name and id, but no input yet. |
| `content_block_delta` | An incremental piece of one block: a text fragment, a fragment of JSON input for a tool call, or a thinking fragment. | Append the fragment to the block at that index. Tool-call inputs arrive as a partial JSON string spread across several deltas, you can't parse them until the block closes. |
| `content_block_stop` | The block at this index is complete. | Finalize the block. For a `tool_use` block, this is the first moment the accumulated JSON input is complete enough to parse. |
| `message_delta` | Top-level changes to the message: the `stop_reason` and final usage counts. | Record the `stop_reason`. It tells you whether the model finished or stopped for some other reason. |
| `message_stop` | The stream is complete. | The assembled content array is now the finished message. From here, treat it exactly like a non-streamed response. |

| Событие | Что оно означает | Что делает ваш обработчик |
|---|---|---|
| `message_start` | Начинается новое сообщение. Содержит «оболочку» сообщения с пустым содержимым и начальными данными об использовании токенов (usage). | Создать пустой массив content, в который будут собираться блоки. |
| `content_block_start` | Открывается новый блок содержимого; указываются его тип (`text`, `tool_use` или `thinking`) и индекс. | Создать по этому индексу ячейку для блока указанного типа. Блок `tool_use` открывается с именем и id, но пока без входных данных. |
| `content_block_delta` | Очередной фрагмент одного блока: фрагмент текста, фрагмент входного JSON для вызова инструмента или фрагмент рассуждения (thinking). | Добавить фрагмент к блоку с этим индексом. Входные данные вызова инструмента приходят как частичная JSON-строка, распределённая по нескольким дельтам, и распарсить их нельзя, пока блок не закроется. |
| `content_block_stop` | Блок с этим индексом завершён. | Финализировать блок. Для блока `tool_use` это первый момент, когда накопленный входной JSON достаточно полон, чтобы его распарсить. |
| `message_delta` | Изменения сообщения на верхнем уровне: `stop_reason` и итоговые счётчики использования токенов. | Сохранить `stop_reason`. Он показывает, завершила ли модель работу штатно или остановилась по другой причине. |
| `message_stop` | Поток завершён. | Собранный массив content теперь является готовым сообщением. Дальше обращайтесь с ним точно так же, как с ответом без стриминга. |

## The rule that keeps your state from getting corrupted: don't act on a partial block
## Правило, которое защищает состояние от повреждения: не действуйте на основе частичного блока

The `tool_use` block is the one to watch. Its input shows up as a partial JSON string spread across many `content_block_delta` events, and that string isn't valid JSON until `content_block_stop` closes the block. If your code tries to parse the input or run the tool before the block closes, it either chokes on malformed JSON or runs with half the arguments missing. So, the rule is simple: collect the deltas, and act only after `content_block_stop` for that block.

> Особого внимания требует блок `tool_use`. Его входные данные приходят как частичная JSON-строка, распределённая по множеству событий `content_block_delta`, и эта строка не является валидным JSON, пока `content_block_stop` не закроет блок. Если ваш код попытается распарсить входные данные или запустить инструмент до закрытия блока, он либо споткнётся о некорректный JSON, либо выполнится без половины аргументов. Поэтому правило простое: собирайте дельты и действуйте только после `content_block_stop` для этого блока.

The same discipline applies when you add a streamed assistant turn to your conversation history. Add it only after `message_stop`, with every block fully assembled. A turn built from a stream that got cut off partway is incomplete, and the `tool_use` pairing rules will reject your next request if a half-built `tool_use` block ends up in the history.

> Та же дисциплина действует, когда вы добавляете полученный через стриминг ход ассистента в историю диалога. Добавляйте его только после `message_stop`, когда все блоки полностью собраны. Ход, собранный из потока, который оборвался на середине, неполон, и правила парности `tool_use` приведут к отклонению вашего следующего запроса, если в историю попадёт недостроенный блок `tool_use`.

## When the stream stops early
## Когда поток обрывается раньше времени

Streams sometimes fail in the middle. A dropped network connection, a timeout, or a client disconnect can end the event series before `message_stop` arrives. The failure that really bites is treating whatever you've collected so far as if it were complete. A partial text block shown to a user is just a cosmetic glitch and a partial `tool_use` block written into history is a structural problem that corrupts the next turn.

- **Track completion on purpose.** A turn is usable only once `message_stop` has arrived. Until then, treat what you've accumulated as provisional.
- **On an interrupted stream, throw away the partial assistant turn** instead of saving it to history, then retry the request. Committing a half-built turn is exactly what breaks the following request.
- **Check the `stop_reason` from `message_delta` before you continue a loop.** A `stop_reason` of `tool_use` means your assembled tool calls are ready to run; any other value means you're on a different path, not the tool path.

> Иногда потоки обрываются на середине. Разрыв сетевого соединения, тайм-аут или отключение клиента могут завершить последовательность событий до прихода `message_stop`. По-настоящему болезненная ошибка — считать всё, что вы успели собрать, завершённым. Частичный текстовый блок, показанный пользователю, — всего лишь косметический сбой, а частичный блок `tool_use`, записанный в историю, — это структурная проблема, которая повреждает следующий ход.
>
> - **Отслеживайте завершение намеренно.** Ход можно использовать только после прихода `message_stop`. До этого момента считайте накопленные данные предварительными.
> - **При прерванном потоке отбрасывайте частичный ход ассистента**, а не сохраняйте его в историю, и повторите запрос. Именно сохранение недостроенного хода ломает следующий запрос.
> - **Проверяйте `stop_reason` из `message_delta`, прежде чем продолжать цикл.** Значение `stop_reason`, равное `tool_use`, означает, что собранные вызовы инструментов готовы к выполнению; любое другое значение означает, что вы идёте по другому пути, а не по пути вызова инструментов.

| | |
|---|---|
| **Handles well** | Long responses and user-facing interfaces where showing output as it generates removes the blank-screen wait. |
| **Adds cost or complexity** | You assemble blocks yourself, you must not act on partial blocks, and you must handle mid-stream interruption explicitly. |
| **Use a different approach** | For short responses or backend jobs where no one is waiting on the output, a non-streamed call is simpler and removes the partial-state risk entirely. |

| | |
|---|---|
| **Хорошо подходит** | Длинные ответы и пользовательские интерфейсы, где показ вывода по мере генерации избавляет от ожидания перед пустым экраном. |
| **Добавляет затрат или сложности** | Вы сами собираете блоки, не должны действовать на основе частичных блоков и обязаны явно обрабатывать прерывание посреди потока. |
| **Используйте другой подход** | Для коротких ответов или фоновых задач на бэкенде, где никто не ждёт вывода, вызов без стриминга проще и полностью исключает риск частичного состояния. |
