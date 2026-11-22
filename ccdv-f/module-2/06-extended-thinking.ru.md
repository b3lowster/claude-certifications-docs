# Extended Thinking: Turning reasoning on, calibrating effort, and reading it back correctly
# Extended Thinking: включение рассуждения, калибровка effort и корректное чтение результата

The prompting techniques shape what Claude produces. Extended thinking shapes how much work Claude does before it answers. Turn it on, and the model writes out its step-by-step reasoning first, then gives you the final answer. Your job is to decide when that extra work is worth the cost and to handle the reasoning it sends back.

> Техники промптинга определяют, что именно выдаёт Claude. Extended thinking (расширенное мышление) определяет, сколько работы Claude выполняет, прежде чем ответить. Если его включить, модель сначала излагает своё пошаговое рассуждение, а затем даёт итоговый ответ. Ваша задача — решить, когда эта дополнительная работа оправдывает затраты, и правильно обработать возвращаемое рассуждение.

## What extended thinking does
## Что делает extended thinking

When you turn on extended thinking, the model "thinks out loud" before it responds. You’ll see this reasoning come back as its own thinking block in the API response, positioned just ahead of the block that holds the actual answer. On the newest models, the thinking block's content is omitted by default; you must request a readable summary through the display setting to see it.

> Когда вы включаете extended thinking, модель «думает вслух», прежде чем ответить. Это рассуждение возвращается в ответе API отдельным блоком thinking, который располагается непосредственно перед блоком с самим ответом. На новейших моделях содержимое блока thinking по умолчанию не включается в ответ; чтобы его увидеть, нужно запросить читаемое краткое изложение (summary) через настройку `display`.

On current models reasoning is adaptive: you enable it with the thinking parameter where it is not already on by default, and the model decides how much reasoning each request needs. You tune depth with the effort setting rather than a fixed token budget. The older budget_tokens control is deprecated and, on the newest model generations, returns a 400 error.

> На текущих моделях рассуждение адаптивное: вы включаете его параметром `thinking` там, где оно не включено по умолчанию, а модель сама решает, сколько рассуждения требуется для каждого запроса. Глубину вы настраиваете параметром `effort`, а не фиксированным бюджетом токенов. Старый параметр `budget_tokens` устарел (deprecated) и на новейших поколениях моделей возвращает ошибку 400.

That reasoning isn’t free; thinking tokens cost the same as output tokens, so running a simple task at high effort means paying for accuracy you don’t need. The choice here mirrors the one you have already made: match the tool to the task. Don’t reach for extended thinking by default, apply it strategically where needed.

> Это рассуждение не бесплатно: токены thinking стоят столько же, сколько выходные токены, поэтому выполнение простой задачи с высоким `effort` означает оплату точности, которая вам не нужна. Этот выбор повторяет тот, что вы уже делали: подбирайте инструмент под задачу. Не включайте extended thinking по умолчанию — применяйте его целенаправленно там, где это действительно нужно.

## When to use extended thinking
## Когда использовать extended thinking

| Task shape | Extended thinking call | Reason |
|---|---|---|
| Multi-step reasoning where the model has to hold several constraints at once: a math derivation, a multi-hop logic problem, planning a sequence of dependent actions. | Enable it, with the effort level matched to the depth of the problem. | The reasoning pass is where the model works through dependencies it would otherwise skip. |
| Mechanical or lookup tasks: classification, format conversion, extracting a field, short factual answers. | Leave it off. | Extended thinking will not improve the answer, and you will be paying more tokens for something you didn't need. A bare prompt with an output constraint is the right tool. |
| Agentic loops where the model plans across several tool calls. | Enable it and budget for the planning step rather than per call. | Reasoning before a plan reduces wrong-tool selection downstream. Note the carry-back rule below, which applies in every tool-use loop. |

| Тип задачи | Решение по extended thinking | Обоснование |
|---|---|---|
| Многошаговое рассуждение, при котором модели нужно одновременно удерживать несколько ограничений: математический вывод, логическая задача в несколько шагов (multi-hop), планирование последовательности зависимых действий. | Включить, подобрав уровень `effort` под сложность задачи. | Именно на этапе рассуждения модель прорабатывает зависимости, которые иначе пропустила бы. |
| Механические задачи или задачи поиска справочной информации (lookup): классификация, преобразование формата, извлечение поля, короткие фактические ответы. | Оставить выключенным. | Extended thinking не улучшит ответ, а вы заплатите за дополнительные токены, которые вам не были нужны. Правильный инструмент здесь — простой промпт с ограничением на вывод (output constraint). |
| Агентные циклы (agentic loops), в которых модель планирует работу на несколько вызовов инструментов. | Включить и закладывать бюджет на этап планирования, а не на каждый вызов. | Рассуждение перед составлением плана снижает вероятность выбора неверного инструмента на последующих шагах. Обратите внимание на правило обратной передачи (carry-back rule) ниже, которое действует в любом цикле с использованием инструментов (tool-use loop). |

## The carry-back rule: thinking blocks must return to the API unchanged
## Правило обратной передачи: блоки thinking должны возвращаться в API без изменений

When extended thinking is on and your conversation uses tools, there’s one rule you can’t skip: every thinking block you get back has to go back to the API exactly as it arrived on the next turn. Each block comes with a signature that confirms the reasoning wasn’t tampered with. If you edit it, summarize it, or drop it, the signature stops matching and the API rejects the request.

> Когда extended thinking включено и в диалоге используются инструменты, есть одно правило, которое нельзя пропустить: каждый полученный блок thinking на следующем ходе должен быть отправлен обратно в API ровно в том виде, в каком он пришёл. Каждый блок сопровождается подписью (signature), которая подтверждает, что рассуждение не было изменено. Если вы отредактируете блок, сократите его до краткого изложения или удалите, подпись перестанет совпадать, и API отклонит запрос.

Redacted thinking blocks work the same way. Their contents are encrypted and not meant to be read by humans, but they still have to be returned untouched.

> Скрытые блоки thinking (redacted thinking blocks) работают точно так же. Их содержимое зашифровано и не предназначено для чтения людьми, но их всё равно необходимо возвращать нетронутыми.

This is a structural requirement, not a prompting choice you get to make. The most common slip-up is stripping out the thinking block to save context, which ends up breaking your next request. If the real worry is how much context piles up from accumulated reasoning, the fix is the context-engineering work we’ll cover in this module.

> Это структурное требование, а не решение на уровне промптинга, которое вы можете принять по своему усмотрению. Самая распространённая ошибка — удалять блок thinking ради экономии контекста, что в итоге ломает следующий запрос. Если на самом деле вас беспокоит, сколько контекста накапливается из-за рассуждений, решение — в работе по контекстной инженерии (context engineering), которую мы рассмотрим в этом модуле.
