 # Testing and tracing
# Тестирование и трассировка

The eval you just built tells you what good looks like as a number. It does not tell you where a failure happened, nor does it prevent a passing eval from hiding a break somewhere in the workflow.

> Только что созданная оценка (eval) выражает в виде числа, как выглядит хороший результат. Но она не говорит, где произошёл сбой, и не мешает успешно пройденной оценке скрывать поломку где-то в рабочем процессе.

A graded target needs a test and tracing layer underneath it: tests that isolate each failure type, and traces that show which step produced the bad result.

> Под оцениваемой целью нужен слой тестирования и трассировки (tracing): тесты, изолирующие каждый тип сбоя, и трассировки, показывающие, на каком шаге получен плохой результат.

## Various test levels, each catching a failure the others miss
## Разные уровни тестов: каждый ловит сбой, который пропускают другие

A test is only useful if you know which failure it identifies. Four levels divide the work, and most silent production breaks live at one particular level:

> Тест полезен, только если вы знаете, какой сбой он выявляет. Работа делится на четыре уровня, и большинство незаметных поломок в продакшене находится на одном определённом уровне:

- **A unit test** isolates one function, such as a parser or a tool wrapper, and checks it on its own. It tells you that one piece behaves, but nothing about how pieces fit together.
- **A functional test** checks that one Claude call returns the expected shape for a given input: the right fields, the right type, a parseable response. It validates the call rather than the system around it.
- **An integration test** exercises the handoff between two components, for example, where a retrieval result is passed into a model call. This is where most silent failures hide, because each side can pass its own tests while the handoff between them is broken.
- **An end-to-end test** runs the whole flow the way a user would, from input to output. It catches breaks that only appear when everything runs together, at the cost of being the slowest to run and the hardest to localize.

> - **Модульный тест (unit test)** изолирует одну функцию, например парсер или обёртку инструмента, и проверяет её отдельно. Он показывает, что одна часть работает правильно, но ничего не говорит о том, как части сочетаются друг с другом.
> - **Функциональный тест (functional test)** проверяет, что один вызов Claude возвращает ожидаемую структуру для заданного входа: нужные поля, нужный тип, разбираемый ответ. Он проверяет сам вызов, а не систему вокруг него.
> - **Интеграционный тест (integration test)** проверяет передачу данных между двумя компонентами, например место, где результат извлечения (retrieval) передаётся в вызов модели. Именно здесь прячется большинство незаметных сбоев, потому что каждая сторона может проходить свои тесты, а передача между ними при этом сломана.
> - **Сквозной тест (end-to-end test)** прогоняет весь процесс так, как это сделал бы пользователь, от входа до выхода. Он ловит поломки, которые проявляются только при совместной работе всех частей, но запускается дольше всех и хуже всего помогает локализовать проблему.

## Tracing: finding the source of failure
## Трассировка: поиск источника сбоя

Tests tell you that a failure exists, but they do not tell you which step caused it. That is what a trace adds.

> Тесты сообщают, что сбой есть, но не говорят, какой шаг его вызвал. Это и добавляет трассировка (trace).

A trace records each step of a run: the prompt, the tool calls, the intermediate outputs, and the timing. When a case fails, the trace lets you see which step produced the bad result. Without a trace, a failed eval tells you something is wrong but does not tell you where it failed. This is the difference between a five-minute fix and a day spent tracing the workflow by hand. A trace reads like a timeline of the run, and the failing step is usually obvious once you can see the intermediate output.

> Трассировка записывает каждый шаг запуска: промпт, вызовы инструментов, промежуточные результаты и время выполнения. Когда случай не проходит, трассировка позволяет увидеть, на каком шаге получен плохой результат. Без трассировки проваленная оценка сообщает, что что-то не так, но не говорит, где произошёл сбой. В этом разница между исправлением за пять минут и днём, потраченным на ручной разбор рабочего процесса. Трассировка читается как хронология запуска, и сбойный шаг обычно становится очевиден, как только виден промежуточный результат.

```text
[trace run_id=8f21c]  case: "Where is my refund?"
  step 1  retrieve(query)        ok    42ms   -> 3 chunks
  step 2  build_prompt(chunks)   ok     1ms   -> prompt 1,240 tok
  step 3  model.call(prompt)     ok   980ms   -> answer "..."
  step 4  parse(answer)          FAIL   2ms   -> KeyError: amount
          final score: 0   (failure localized to step 4, the parser)
```

The trace turns "the case failed" into "step four: the parser raised a KeyError on a field the model did not return." That is also what makes a change reviewable: you can show the step that moved rather than just the score that dropped.

> Трассировка превращает «случай не прошёл» в «шаг четыре: парсер выбросил `KeyError` на поле, которое модель не вернула». Именно это делает изменение пригодным для ревью: можно показать шаг, который изменился, а не только упавший балл.

## Routing between the two approaches so you pay for iteration only when you need it
## Маршрутизация между двумя подходами, чтобы платить за итерации только тогда, когда они нужны

You do not have to pick one strategy for everything. A cheap classification step can send single-fact lookups to the fetch-once path and multi-part questions to the search-across-rounds path. This allows you to spend on iteration only when the query needs it. Defaulting everything to iterative search inflates cost and latency on questions a single fetch would have answered, while defaulting everything to a static index gives shallow answers on questions that needed several passes. The router is one small model call that reads the query and picks the path.

> Не обязательно выбирать одну стратегию для всего. Дешёвый шаг классификации может отправлять запросы на поиск одного факта по пути однократного извлечения (fetch-once), а составные вопросы — по пути поиска в несколько раундов (search-across-rounds). Это позволяет тратить ресурсы на итерации, только когда запрос этого требует. Если по умолчанию всё отправлять в итеративный поиск, растут стоимость и задержка на вопросах, на которые хватило бы одного извлечения, а если по умолчанию всё отправлять в статический индекс, ответы на вопросы, требующие нескольких проходов, получаются поверхностными. Маршрутизатор (router) — это один небольшой вызов модели, который читает запрос и выбирает путь.

```python
def route(query):
    kind = classify(query)        # cheap call: "lookup" or "multi_step"
    if kind == "lookup":
        return fetch_once(query)  # static retrieval, one pass
    return agentic_search(query)  # search across rounds
```

That one classification call costs far less than running iterative search on a query a single retrieval would have answered. The router earns its cost whenever your traffic is mixed: some queries are simple lookups and some need several passes. If every query is the same shape, skip the router and hardcode the path that fits.

> Один вызов классификации стоит гораздо меньше, чем итеративный поиск по запросу, на который ответило бы одно извлечение. Маршрутизатор окупается, когда трафик смешанный: часть запросов — простой поиск, а части нужно несколько проходов. Если все запросы одного типа, обойдитесь без маршрутизатора и жёстко задайте подходящий путь.

## The reference you can keep open while you build
## Справочник, который стоит держать под рукой во время разработки

| Level | What it isolates | What it cannot catch |
|---|---|---|
| Unit | One function, such as a parser or tool wrapper, on its own. | Anything about how components fit together. |
| Functional | One Claude call returning the expected shape for an input. | Failures in the system around that single call. |
| Integration | The seam where two components hand off, such as retrieval into the model. | Whole-flow behavior that only emerges end to end. |
| End-to-end | The full flow as a user runs it, input to output. | Where exactly the break is, since it sees only the final result. |
| Retrieval choice | Fetch a fixed set once for single-fact lookups in a stable corpus. | Multi-step questions and changing corpora, which need search across rounds. |

| Уровень | Что изолирует | Что не может поймать |
|---|---|---|
| Модульный (unit) | Одну функцию, например парсер или обёртку инструмента, отдельно от остального. | Ничего о том, как компоненты сочетаются друг с другом. |
| Функциональный (functional) | Один вызов Claude, возвращающий ожидаемую структуру для заданного входа. | Сбои в системе вокруг этого единственного вызова. |
| Интеграционный (integration) | Стык, где два компонента передают данные друг другу, например извлечение (retrieval) → модель. | Поведение всего процесса, которое проявляется только при сквозном прогоне. |
| Сквозной (end-to-end) | Весь процесс так, как его запускает пользователь, от входа до выхода. | Где именно находится поломка, поскольку он видит только итоговый результат. |
| Выбор способа извлечения (retrieval choice) | Однократное получение фиксированного набора для поиска одного факта в стабильном корпусе. | Многошаговые вопросы и меняющиеся корпусы, которым нужен поиск в несколько раундов. |

| | |
|---|---|
| **Handles well** | Localizes a failure to a step and matches each test to the break it can see. |
| **Adds cost or complexity** | Tracing and four test levels are infrastructure you build and maintain. |
| **Use a different approach** | For a single-fact lookup in a stable corpus, fetch-once retrieval beats iterative search. |

| | |
|---|---|
| **Хорошо подходит** | Локализует сбой до конкретного шага и сопоставляет каждый тест с той поломкой, которую он способен увидеть. |
| **Добавляет стоимость или сложность** | Трассировка и четыре уровня тестов — это инфраструктура, которую нужно создать и поддерживать. |
| **Используйте другой подход** | Для поиска одного факта в стабильном корпусе однократное извлечение (fetch-once) лучше итеративного поиска. |
