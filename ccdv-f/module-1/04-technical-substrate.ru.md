# The technical substrate: SDKs, REST, streaming, async
# Техническая основа: SDK, REST, стриминг, асинхронность

## How a developer reaches Claude: SDK versus raw REST
## Как разработчик обращается к Claude: SDK или «чистый» REST

At its core, Claude is reached over an HTTP REST API: your code sends a request to an endpoint with your API key and a JSON body, and reads a JSON response back. You can call that endpoint directly with any HTTP client. More commonly you use an official SDK, available for Python and TypeScript among others, which is a thin convenience layer over the same REST API. It handles authentication, request construction, retries, and response parsing so you write less boilerplate. The SDK and raw REST reach the same API and the same model. The SDK saves you from assembling requests by hand. Module 2 builds against the SDK and the Messages API, which sits on this same foundation.

> В основе доступ к Claude осуществляется через HTTP REST API: ваш код отправляет запрос на эндпоинт (endpoint) с вашим API-ключом и телом в формате JSON и получает обратно JSON-ответ. Вызывать этот эндпоинт можно напрямую с помощью любого HTTP-клиента. Чаще, однако, используют официальный SDK, доступный, в частности, для Python и TypeScript, — это тонкий вспомогательный слой поверх того же REST API. Он берёт на себя аутентификацию, формирование запросов, повторные попытки (retries) и разбор ответов, так что вам приходится писать меньше шаблонного кода (boilerplate). SDK и «чистый» REST обращаются к одному и тому же API и одной и той же модели. SDK просто избавляет вас от ручной сборки запросов. В Модуле 2 работа строится на SDK и Messages API, который опирается на ту же основу.

## Synchronous, streaming, and real-time responses
## Синхронные, потоковые ответы и ответы в реальном времени

A synchronous request is the simplest pattern: you send the request and wait for the complete response to come back in one piece, then act on it. That is fine for short responses and backend jobs where no one is waiting. When a response is long or a user is watching, streaming sends the response in pieces as the model generates it. Output appears immediately rather than after a blank-screen wait, and your code reassembles the pieces into the final message. Claude exposes streaming over the same HTTP connection using server-sent events. Module 2 teaches how to consume a stream safely and recover when it is interrupted.

> Синхронный запрос — самый простой паттерн: вы отправляете запрос, ждёте, пока полный ответ вернётся целиком, и затем действуете на его основе. Это подходит для коротких ответов и фоновых задач на бэкенде, где никто не ждёт результата. Когда ответ длинный или пользователь наблюдает за процессом, стриминг (streaming) отправляет ответ частями по мере того, как модель его генерирует. Вывод появляется сразу, а не после ожидания перед пустым экраном, и ваш код собирает части в итоговое сообщение. Claude предоставляет стриминг через то же HTTP-соединение с помощью server-sent events (SSE). В Модуле 2 разбирается, как безопасно получать поток и восстанавливаться при его прерывании.

## Asynchronous patterns for high-volume work
## Асинхронные паттерны для больших объёмов работы

Two patterns address high-volume work, and they solve different problems.

> Для работы с большими объёмами существуют два паттерна, и они решают разные задачи.

The Python SDK exposes an async client (AsyncAnthropic) that uses non-blocking async/await to make API calls without tying up your application thread. In the TypeScript SDK the standard Anthropic client is Promise-based, so you await calls directly. There is no separate async client class. Either way the request still returns in real time, but your application can handle other work while it waits. This is the right pattern when you need concurrency without blocking.

> Python SDK предоставляет асинхронный клиент (`AsyncAnthropic`), который использует неблокирующий `async`/`await` для вызовов API, не занимая поток вашего приложения. В TypeScript SDK стандартный клиент `Anthropic` основан на промисах (Promise), поэтому вызовы просто ожидаются через `await`. Отдельного класса асинхронного клиента там нет. В обоих случаях ответ на запрос по-прежнему приходит в реальном времени, но ваше приложение может выполнять другую работу, пока ждёт. Это правильный паттерн, когда нужна конкурентность (concurrency) без блокировок.

The Message Batches API is a separate pattern for bulk offline workloads. You submit a large set of requests in one call, receive an identifier, and poll for completion. Batch jobs can take up to 24 hours to complete and run at a lower per-token cost in exchange for that latency. This suits offline pipelines, evaluation runs, and bulk jobs where no user is waiting on each result and cost matters more than turnaround time.

> Message Batches API — отдельный паттерн для массовых офлайн-нагрузок. Вы отправляете большой набор запросов одним вызовом, получаете идентификатор и периодически опрашиваете (poll) статус завершения. Выполнение пакетных заданий (batch jobs) может занимать до 24 часов, и в обмен на эту задержку они стоят дешевле в пересчёте на токен. Это подходит для офлайн-пайплайнов, прогонов оценки (evaluation runs) и массовых заданий, где никто из пользователей не ждёт каждого результата, а стоимость важнее скорости выполнения.
