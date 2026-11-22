# Surviving production failure: tool errors
# Как пережить сбои в продакшене: ошибки инструментов

Your tests now tell you a failure exists and the trace tells you where it happens. The next question is what the system does the moment a failure happens in live traffic.

> Теперь тесты сообщают вам о наличии сбоя, а трассировка (trace) показывает, где он происходит. Следующий вопрос — что делает система в момент, когда сбой происходит на реальном трафике.

Production introduces failures a prototype never sees. The difference between a resilient system and a fragile one is whether you decided in advance how each kind of failure is handled.

> В продакшене возникают сбои, с которыми прототип никогда не сталкивается. Устойчивая система отличается от хрупкой тем, решили ли вы заранее, как обрабатывается каждый вид сбоя.

## Every failure starts with one question: is it retriable or terminal?
## Каждый сбой начинается с одного вопроса: можно ли повторить попытку или ошибка окончательная?

The test is a single question: would waiting and trying the exact same request again plausibly work? If yes, it is retriable. If not, it is terminal. A rate limit clears with time; a malformed request will fail identically until the request itself is fixed.

> Проверка сводится к одному вопросу: может ли сработать тот же самый запрос, если подождать и повторить его? Если да, ошибка повторяемая (retriable). Если нет — окончательная (terminal). Ограничение частоты запросов (rate limit) со временем снимается; некорректный запрос будет завершаться одинаковой ошибкой, пока сам запрос не исправят.

Production traffic produces failures development never shows you: rate-limit responses, timeouts, malformed tool results, and transient network errors. The first decision for any failure is whether a later attempt is likely to succeed. If so, the failure is retriable. If not, retrying only wastes time and budget, making it terminal. A rate-limit response or a temporary server overload is retriable, because the same request will probably go through in a moment. A malformed request or an authentication failure is terminal, because retrying the identical bad request changes nothing. Every subsequent handling decision depends on which bucket a failure lands in. On the Anthropic API, the status code tells you the bucket. A `429` means you hit a rate limit and a `529` means the service is temporarily overloaded, both are retriable. A `400` means a bad request and a `401` means an auth failure, both are terminal. Server errors in the `5xx` range, including a `500` internal error and a `504` timeout, are also retriable, because they are Anthropic-side faults that typically resolve on retry.

> Продакшен-трафик порождает сбои, которых вы никогда не видите при разработке: ответы об ограничении частоты запросов, тайм-ауты, некорректные результаты инструментов и временные сетевые ошибки. Первое решение для любого сбоя — вероятно ли, что следующая попытка будет успешной. Если да, сбой повторяемый. Если нет, повторная попытка лишь тратит время и бюджет, поэтому сбой окончательный. Ответ об ограничении частоты запросов или временная перегрузка сервера — повторяемые сбои, потому что тот же запрос, скорее всего, пройдёт через мгновение. Некорректный запрос или ошибка аутентификации — окончательные, потому что повтор того же плохого запроса ничего не изменит. Каждое последующее решение по обработке зависит от того, в какую категорию попал сбой. В Anthropic API категорию определяет код состояния. `429` означает, что вы упёрлись в ограничение частоты запросов, а `529` — что сервис временно перегружен; обе ошибки повторяемые. `400` означает некорректный запрос, а `401` — ошибку аутентификации; обе окончательные. Серверные ошибки диапазона `5xx`, включая внутреннюю ошибку `500` и тайм-аут `504`, тоже повторяемые, потому что это сбои на стороне Anthropic, которые обычно устраняются при повторе.

```python
RETRIABLE = {429, 529, 500, 502, 503, 504}   # rate limit, overload, transient
TERMINAL  = {400, 401, 403, 404}             # bad request, auth, missing

def is_retriable(status):
    return status in RETRIABLE   # everything else fails fast
```

The reason this one distinction carries so much weight is that it determines whether waiting helps. A retriable error is one where the cause is transient: the service was momentarily over capacity, a connection dropped, or you briefly exceeded a per-minute limit. Time alone resolves it, so a later attempt is likely to succeed. A terminal error is one where the cause is in the request itself: a malformed body, an expired key, a model name that does not exist. Time changes nothing, because each request will produce an identical error. Retrying a terminal error wastes the retry budget and hides the actual problem behind a wall of identical failures. Each unnecessary retry consumes retry budget and increases the latency that a retriable failure elsewhere in the flow might have needed. Correct classification preserves the retry budget for failures that need it.

> Это одно различие так важно потому, что оно определяет, поможет ли ожидание. Повторяемая ошибка — та, причина которой временная: сервис на мгновение был перегружен, соединение оборвалось или вы ненадолго превысили поминутный лимит. Её устраняет само время, поэтому следующая попытка, скорее всего, будет успешной. Окончательная ошибка — та, причина которой в самом запросе: некорректное тело запроса, истёкший ключ, несуществующее имя модели. Время ничего не меняет, потому что каждый запрос приведёт к той же ошибке. Повтор окончательной ошибки тратит бюджет повторов и прячет настоящую проблему за стеной одинаковых сбоев. Каждый лишний повтор расходует бюджет повторов и увеличивает задержку, запас которой мог бы понадобиться повторяемому сбою в другом месте процесса. Правильная классификация сохраняет бюджет повторов для тех сбоев, которым он действительно нужен.

A few statuses sit on the line and are worth calling out. A timeout is usually retriable because the work may simply have taken longer than the client was willing to wait. Repeated timeouts on expensive requests is a signal to fix the request itself, not to retry it. A `500` from the service is retriable, because it is a server-side fault that often clears. A `403` is terminal, because it is a permissions problem that a retry cannot fix. When you are unsure, the safe default is to treat an error as terminal and raise it. A failure incorrectly classified as terminal fails loudly and gets fixed. A failure incorrectly classified as retriable hammers a service and hides the real problem behind a wall of retries.

> Несколько кодов состояния находятся на границе, и о них стоит сказать отдельно. Тайм-аут обычно повторяемый, потому что работа могла просто занять больше времени, чем клиент был готов ждать. Повторяющиеся тайм-ауты на дорогих запросах — сигнал исправить сам запрос, а не повторять его. Ошибка `500` от сервиса повторяемая, потому что это сбой на стороне сервера, который часто проходит сам. `403` — окончательная ошибка, потому что это проблема прав доступа, которую повтор не исправит. Если вы не уверены, безопасный вариант по умолчанию — считать ошибку окончательной и выбрасывать её. Сбой, ошибочно отнесённый к окончательным, проявляется громко и исправляется. Сбой, ошибочно отнесённый к повторяемым, забрасывает сервис запросами и прячет настоящую проблему за стеной повторов.

## The SDK already retries some failures, so know what it covers before you write your own
## SDK уже повторяет некоторые запросы, поэтому узнайте, что он покрывает, прежде чем писать свои повторы

Before you build a retry loop by hand, check what the SDK does for you. The Anthropic client libraries automatically retry transient failures with progressive retry delays, up to a configurable number of attempts. The point of knowing this is to avoid adding your own retries on top of the ones the SDK is already running. Two retry loops wrapped around the same call multiply attempts against a rate limit rather than capping them. Decide where the retry lives: either let the SDK handle transient cases and reserve your own code for application-specific fallbacks, or turn the SDK retries down and own the full path yourself. Running both layers retrying the same failure without either knowing about the other is the pattern to avoid.

> Прежде чем вручную писать цикл повторов, проверьте, что SDK делает за вас. Клиентские библиотеки Anthropic автоматически повторяют запросы при временных сбоях с нарастающими задержками, вплоть до настраиваемого числа попыток. Знать это нужно, чтобы не добавлять собственные повторы поверх тех, что уже выполняет SDK. Два цикла повторов вокруг одного и того же вызова умножают число попыток при ограничении частоты запросов, а не ограничивают его. Решите, где находится логика повторов: либо пусть SDK обрабатывает временные сбои, а ваш код отвечает за резервные варианты (fallbacks), специфичные для приложения, либо уменьшите повторы SDK и полностью возьмите этот путь на себя. Избегать нужно ситуации, когда оба слоя повторяют один и тот же сбой, не зная друг о друге.

The API also returns rate-limit headers on each response that tell you how much of your limit remains and when it resets. The most useful is `retry-after`, which a `429` or `529` response includes to tell you how long to wait before trying again. Honoring that value is more precise than guessing with backoff alone, because the service is telling you exactly when capacity returns. The corrected retry code later in this module reads `retry-after` first and falls back to exponential backoff only when the header is absent. Treat the header as the authoritative wait time when it is present, and treat your own backoff as the fallback when it is not. The specific header names and limit values are version-pinned, so confirm them against the reference layer at build time.

> API также возвращает в каждом ответе заголовки об ограничениях частоты запросов, которые сообщают, сколько осталось от вашего лимита и когда он сбросится. Самый полезный — `retry-after`: его содержит ответ `429` или `529`, чтобы сообщить, сколько ждать перед следующей попыткой. Следовать этому значению точнее, чем угадывать с помощью одной только отсрочки (backoff), потому что сервис прямо говорит, когда снова появится ресурс. Исправленный код повторов далее в этом модуле сначала читает `retry-after` и переходит на экспоненциальную отсрочку (exponential backoff), только если заголовка нет. Считайте заголовок авторитетным временем ожидания, когда он есть, а собственную отсрочку — резервным вариантом, когда его нет. Конкретные названия заголовков и значения лимитов привязаны к версии, поэтому сверяйте их со справочным разделом во время сборки.

## Tool errors must come back to Claude explicitly rather than dropped
## Ошибки инструментов нужно явно возвращать Claude, а не отбрасывать

When your code runs a tool and that tool fails, the result should be returned to Claude with `is_error` explicitly set to `true`. It should not return as a silent empty result. With the error returned, the model can react: try a different approach, ask for clarification, or stop. A tool that drops its own error and returns nothing produces a confident yet wrong answer downstream. This is because the model treats the empty result as valid data and continues reasoning on top of it. A visible failure is far easier to catch than a confident but incorrect answer built on missing data.

> Когда ваш код запускает инструмент и этот инструмент завершается сбоем, результат нужно вернуть Claude с явно установленным `is_error` в значении `true`. Он не должен возвращаться как молчаливый пустой результат. Получив ошибку, модель может отреагировать: попробовать другой подход, попросить уточнение или остановиться. Инструмент, который проглатывает собственную ошибку и ничего не возвращает, приводит к уверенному, но неверному ответу на следующих шагах. Причина в том, что модель воспринимает пустой результат как корректные данные и продолжает рассуждать на его основе. Видимый сбой гораздо легче поймать, чем уверенный, но неверный ответ, построенный на отсутствующих данных.

```python
def run_tool(tool_use):
    try:
        result = execute(tool_use)
        return {"type": "tool_result", "tool_use_id": tool_use.id,
                "content": result}
    except Exception as e:
        # surface the error so Claude can react, do NOT return empty
        return {"type": "tool_result", "tool_use_id": tool_use.id,
                "is_error": True, "content": f"Tool failed: {e}"}

def run_tool(tool_use):
# A refusal is a 200 at the HTTP layer, the retriable classifier will not catch it
if response.stop_reason == "refusal":
    raise ValueError("Model refused the request. Review input before retrying.")
```

With `is_error` set, the model knows the tool failed and can react. Without it, the model treats the empty result as valid data and continues on a false premise.

> Когда `is_error` установлен, модель знает, что инструмент завершился сбоем, и может отреагировать. Без него модель воспринимает пустой результат как корректные данные и продолжает работу, исходя из ложной предпосылки.

## The error-handling decision table you can keep open while you build
## Таблица решений по обработке ошибок, которую стоит держать под рукой во время разработки

| Error type | Retriable or fail-fast | Backoff strategy | Fallback behavior |
|---|---|---|---|
| Rate limit (`429`) | Retriable | Exponential backoff with jitter, honor `retry-after`, capped attempts. | After the cap, raise a clean error or route to a cached or simpler result. |
| Overloaded (`529`) | Retriable | Backoff; a `529` reflects Anthropic-side load, so it is not a rate-limit signal. | Fail over to a fallback path or return a graceful error if it persists. |
| Bad request (`400`) | Fail fast | No retry. The identical request will fail again. | Fix or reject the input and surface the error to the caller. |
| Tool result error | Depends on the tool | Retry only if the underlying cause is transient. | Return the error flag to Claude so the model can react, never silence it. |
| Refusal (`200`, `stop_reason: "refusal"`) | Fail fast | No retry. The model made a content decision, not a transient error. | Raise the refusal to the caller. Log it. Do not silently retry or treat it as valid output. |

| Тип ошибки | Повторять или сразу завершать (fail-fast) | Стратегия отсрочки (backoff) | Резервное поведение (fallback) |
|---|---|---|---|
| Ограничение частоты запросов (`429`) | Повторять | Экспоненциальная отсрочка со случайным разбросом (jitter), соблюдение `retry-after`, ограниченное число попыток. | После исчерпания лимита попыток выбросить понятную ошибку или перейти к кэшированному или более простому результату. |
| Перегрузка (`529`) | Повторять | Отсрочка; `529` отражает нагрузку на стороне Anthropic, поэтому это не сигнал об ограничении частоты запросов. | Переключиться на резервный путь или, если ошибка не проходит, вернуть корректно оформленную ошибку. |
| Некорректный запрос (`400`) | Сразу завершать | Без повторов. Тот же запрос снова завершится ошибкой. | Исправить или отклонить входные данные и передать ошибку вызывающей стороне. |
| Ошибка результата инструмента | Зависит от инструмента | Повторять, только если исходная причина временная. | Вернуть Claude флаг ошибки, чтобы модель могла отреагировать; никогда не замалчивать её. |
| Отказ (`200`, `stop_reason: "refusal"`) | Сразу завершать | Без повторов. Модель приняла решение по содержанию, это не временная ошибка. | Передать отказ вызывающей стороне. Записать в журнал. Не повторять молча и не считать корректным результатом. |

| | |
|---|---|
| **Handles well** | Keeps one bad response from cascading into an outage by handling each failure type by name. |
| **Adds cost or complexity** | Every failure path is code you write, test, and maintain on top of the happy path. |
| **Use a different approach** | Do not retry a terminal error. Retrying a `400` does nothing but waste the retry budget. |

| | |
|---|---|
| **Хорошо подходит** | Не даёт одному неудачному ответу перерасти в каскадный отказ, поскольку каждый тип сбоя обрабатывается явно, по названию. |
| **Добавляет стоимость или сложность** | Каждый путь обработки сбоя — это код, который нужно написать, протестировать и поддерживать в дополнение к основному сценарию (happy path). |
| **Используйте другой подход** | Не повторяйте окончательную ошибку. Повтор `400` лишь впустую тратит бюджет повторов. |
