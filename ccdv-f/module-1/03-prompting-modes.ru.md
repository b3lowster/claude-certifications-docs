# Prompting modes: zero-shot, one-shot, multi-shot
# Режимы промптинга: zero-shot, one-shot, multi-shot

## The three modes
## Три режима

Separate from how you word a prompt is how many worked examples you give the model inside it. Zero-shot gives the instruction and no examples: you describe the task and ask for the result. One-shot adds one example of the input paired with the desired output. Multi-shot, also called few-shot, includes several such examples. The examples are not training data; they sit in the prompt and show the model the exact shape of the answer you want, which a description alone often fails to pin down.

> Помимо того, как сформулирован промпт, важно и то, сколько разобранных примеров вы даёте модели внутри него. Zero-shot — это инструкция без примеров: вы описываете задачу и просите результат. One-shot добавляет один пример входных данных в паре с желаемым выводом. Multi-shot, также называемый few-shot, включает несколько таких примеров. Примеры — это не обучающие данные: они находятся в промпте и показывают модели точную форму ответа, который вам нужен, а одного лишь описания для этого часто недостаточно.

## The cost and quality trade-off
## Компромисс между стоимостью и качеством

Each example you add costs tokens on every call and consumes context budget, so the choice trades quality against cost. Reach for zero-shot when the task is simple and the output shape is obvious. Move to one-shot or multi-shot when the output has a specific structure, casing, or edge case that a description keeps missing. Often one or two correct examples usually fix the issue faster than another paragraph of instructions. The general discipline, which Module 2 reinforces, is to add the smallest amount of prompt that produces a reliable result.

> Каждый добавленный пример стоит токенов при каждом вызове и расходует бюджет контекста, поэтому выбор — это компромисс между качеством и стоимостью. Используйте zero-shot, когда задача простая, а форма вывода очевидна. Переходите к one-shot или multi-shot, когда у вывода есть особая структура, регистр букв (casing) или пограничный случай (edge case), которые описание раз за разом не позволяет передать. Как правило, один-два правильных примера решают проблему быстрее, чем ещё один абзац инструкций. Общий принцип, который закрепляется в Модуле 2: добавлять минимальный объём промпта, дающий надёжный результат.

## Mode choice interacts with model choice
## Выбор режима связан с выбором модели

Prompting mode and model choice are related levers. A more capable model often succeeds zero-shot on a task where a smaller model needs a few examples to match the structure, so adding examples can let a cheaper model do the job. The two decisions are worth making together: try the simplest model and the fewest examples that meet your eval, and add capability or examples only where the eval says you need them.

> Режим промптинга и выбор модели — взаимосвязанные рычаги. Более мощная модель часто справляется в режиме zero-shot с задачей, в которой модели поменьше нужно несколько примеров, чтобы соблюсти структуру, поэтому добавление примеров может позволить выполнить работу более дешёвой модели. Эти два решения стоит принимать вместе: попробуйте самую простую модель и минимальное число примеров, при которых проходит ваш eval, и добавляйте мощность модели или примеры только там, где eval показывает, что они нужны.
