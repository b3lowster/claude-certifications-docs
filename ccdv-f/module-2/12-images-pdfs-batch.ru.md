# Images, PDFs, and high-volume processing
# Изображения, PDF и обработка больших объёмов

Up to now you've been managing what Claude remembers between turns. Multimodal ingestion shifts the question to what you're sending in: every image and PDF consumes context budget before Claude reads a single character of your prompt, which changes how you structure requests and what you can fit in one. The second half of this topic deals with the opposite end of the same problem. When you have thousands of inputs to process, sending one request at a time and waiting for each response stops making sense, and the Batch API is how you handle that volume without blocking your application.

> До сих пор вы управляли тем, что Claude помнит между ходами. Мультимодальный ввод смещает вопрос к тому, что вы отправляете: каждое изображение и каждый PDF расходуют бюджет контекста ещё до того, как Claude прочитает хотя бы один символ вашего промпта, а это меняет то, как вы структурируете запросы и что помещается в один запрос. Вторая половина этой темы посвящена противоположному концу той же проблемы. Когда нужно обработать тысячи входных данных, отправлять по одному запросу и ждать каждого ответа становится бессмысленно, и Batch API — это способ справиться с таким объёмом, не блокируя приложение.

## Image token cost: Calculate before you commit
## Стоимость изображения в токенах: посчитайте, прежде чем принимать решение

Images are not free in terms of context budget. Claude views images in patches: each 28×28-pixel block of the image is one visual token, so an image costs ⌈width / 28⌉ × ⌈height / 28⌉ visual tokens. A 1,000 × 1,000 pixel image is ⌈1000/28⌉ × ⌈1000/28⌉ = 36 × 36 patches, about 1,296 visual tokens. At that rate, ten high-resolution screenshots consume as much context as a detailed system prompt. Each model also has a maximum native image resolution, expressed as a long-edge limit and a visual-token limit, and these limits differ by model tier. The newest models accept substantially larger images than the standard tier. Images larger than either limit are downscaled before processing, so the formula runs on the scaled dimensions. Confirm the current per-tier limits against the Vision page (Resolution and token cost) at build time; the limits have changed between model generations and will again.

> С точки зрения бюджета контекста изображения не бесплатны. Claude воспринимает изображения фрагментами (patches): каждый блок размером 28×28 пикселей — это один визуальный токен, поэтому изображение стоит ⌈ширина / 28⌉ × ⌈высота / 28⌉ визуальных токенов. Изображение 1000 × 1000 пикселей — это ⌈1000/28⌉ × ⌈1000/28⌉ = 36 × 36 фрагментов, около 1296 визуальных токенов. При таком расходе десять скриншотов высокого разрешения занимают столько же контекста, сколько подробный системный промпт. У каждой модели также есть максимальное собственное разрешение изображений, выраженное ограничением по длинной стороне и ограничением на число визуальных токенов, и эти ограничения различаются в зависимости от уровня модели. Новейшие модели принимают значительно более крупные изображения, чем стандартный уровень. Изображения, превышающие любое из ограничений, перед обработкой уменьшаются, так что формула применяется к уменьшенным размерам. Проверяйте актуальные ограничения для каждого уровня на странице Vision (раздел Resolution and token cost) на этапе разработки: ограничения менялись между поколениями моделей и изменятся снова.

The calculation matters at design time. If you are building a pipeline that processes images, measure the token cost of a typical production image against your model's context limit before you write the ingestion code. The fix for an over-budget pipeline is often a ten-minute image resize step. If you discover this after deployment, it takes even longer.

> Этот расчёт важен на этапе проектирования. Если вы строите пайплайн для обработки изображений, измерьте стоимость в токенах типичного продакшен-изображения относительно лимита контекста вашей модели, прежде чем писать код загрузки. Исправление для пайплайна, превышающего бюджет, часто сводится к десятиминутному шагу изменения размера изображений. Если же вы обнаружите это после развёртывания, времени уйдёт гораздо больше.

## Different ways to send an image: When each is right
## Разные способы отправки изображения: когда уместен каждый из них

### Inline base64
### Встроенный base64

- **How it works:** Encode the image bytes as a base64 string and include the data directly in the message block.
- **Overhead:** The full encoded payload travels with every request, which inflates request size and counts against latency on large images.
- **When to use:** Best for one-off images where adding an upload step would add complexity without a payoff. The same image sent repeatedly multiplies the cost, so reach for a different method if reuse is likely.

> - **Как это работает:** байты изображения кодируются в строку base64, и данные включаются прямо в блок сообщения.
> - **Накладные расходы:** полные закодированные данные передаются с каждым запросом, что увеличивает размер запроса и сказывается на задержке при больших изображениях.
> - **Когда использовать:** лучше всего для разовых изображений, когда дополнительный шаг загрузки добавил бы сложности без выгоды. Повторная отправка одного и того же изображения умножает стоимость, поэтому, если вероятно повторное использование, выбирайте другой способ.

### URL reference
### Ссылка по URL

- **How it works:** Pass a publicly reachable URL in the source block, and Claude fetches the image at request time.
- **Overhead:** No payload travels with the request, but you take on the dependency that the URL must be stable, public, and reachable at the moment Claude tries to fetch it.
- **When to use:** Best when the image is already hosted at a stable public URL you control. Skip it for anything behind auth, anything signed with a short expiry, or anything you can't guarantee will be reachable when the request runs.

> - **Как это работает:** в блоке source передаётся общедоступный URL, и Claude загружает изображение в момент обработки запроса.
> - **Накладные расходы:** данные с запросом не передаются, но появляется зависимость: URL должен быть стабильным, публичным и доступным в тот момент, когда Claude пытается его загрузить.
> - **Когда использовать:** лучше всего, когда изображение уже размещено по стабильному публичному URL, который вы контролируете. Не используйте этот способ для ресурсов за аутентификацией, ссылок с подписью и коротким сроком действия или всего, доступность чего вы не можете гарантировать в момент выполнения запроса.

### Files API
### Files API

- **How it works:** Upload the file once through a separate API call, receive a `file_id`, and reference that ID in any future message.
- **Overhead:** The upload is a one-time cost; every later request carries the ID instead of the bytes, so payload overhead drops to near-zero from that point on. Currently in beta and not available on Bedrock or Vertex AI; verify availability for your deployment platform.
- **When to use:** Best when the same image or PDF appears across multiple requests, or when the asset is large enough that re-sending it would dominate request size. Also, the cleanest choice when you want asset management to live separately from inference calls, and the right choice for images that appear across multiple conversation turns, since the `file_id` carries no payload weight as history grows.

> - **Как это работает:** файл один раз загружается отдельным вызовом API, вы получаете `file_id` и ссылаетесь на этот ID в любых последующих сообщениях.
> - **Накладные расходы:** загрузка — разовые затраты; каждый последующий запрос передаёт ID вместо байтов, поэтому с этого момента накладные расходы на передачу данных практически нулевые. Сейчас функция в бете и недоступна в Bedrock и Vertex AI; проверьте доступность для вашей платформы развёртывания.
> - **Когда использовать:** лучше всего, когда одно и то же изображение или PDF встречается в нескольких запросах или когда ресурс настолько велик, что его повторная отправка составляла бы основную часть размера запроса. Это также самый чистый вариант, когда вы хотите, чтобы управление ресурсами было отделено от вызовов инференса, и правильный выбор для изображений, которые встречаются в нескольких ходах диалога, поскольку `file_id` не добавляет веса данных по мере роста истории.

## Sending PDFs: The document block
## Отправка PDF: блок document

For PDFs, the block type is `document` rather than `image`. The source structure follows the same pattern as images, which means it can be base64, a URL, or a Files API `file_id`. There is no required name field on a document block. The block accepts an optional `title` field for a readable document name, and an optional `context` field for additional metadata, but neither is required to send a PDF. All other mechanics, including token cost considerations and Files API reuse, apply in the same way.

> Для PDF тип блока — `document`, а не `image`. Структура source такая же, как у изображений: это может быть base64, URL или `file_id` из Files API. Обязательного поля с именем у блока document нет. Блок принимает необязательное поле `title` для читаемого названия документа и необязательное поле `context` для дополнительных метаданных, но ни одно из них не требуется для отправки PDF. Вся остальная механика, включая соображения о стоимости в токенах и повторное использование через Files API, применяется точно так же.

```json
{
  "type": "document",
  "source": {
    "type": "base64",
    "media_type": "application/pdf",
    "data": "<base64-encoded-pdf-bytes>"
  },
  "title": "contract_review.pdf"
}
```

## Applying prompting techniques to multimodal inputs
## Применение техник промптинга к мультимодальным входным данным

The same prompting techniques from the first section apply to image and PDF analysis. A bare "describe this image" prompt produces shallow output for the same reason a bare text prompt does as Claude has no target structure to aim for.

> Те же техники промптинга из первого раздела применимы и к анализу изображений и PDF. «Голый» промпт «опиши это изображение» даёт поверхностный результат по той же причине, что и «голый» текстовый промпт: у Claude нет целевой структуры, к которой нужно стремиться.

The difference is that images carry ambiguity that text cannot, which includes overlapping objects, depth and spatial relationships, and partial occlusion. A prompt for visual analysis should name how Claude should handle each type of ambiguity. "If objects overlap, describe each separately and note the overlap" is a concrete constraint that a text-only prompt would never need.

> Разница в том, что изображения несут неоднозначность, которой не бывает в тексте: перекрывающиеся объекты, глубина и пространственные отношения, частичное загораживание. Промпт для визуального анализа должен указывать, как Claude следует обрабатывать каждый тип неоднозначности. «Если объекты перекрываются, опиши каждый отдельно и отметь перекрытие» — конкретное ограничение, которое никогда не понадобилось бы чисто текстовому промпту.

## The Message Batches API: High-volume asynchronous processing
## Message Batches API: асинхронная обработка больших объёмов

When you need to run the same prompt pattern against hundreds or thousands of inputs, the synchronous API is the wrong model. Each synchronous call blocks until complete. At scale, that means your application is either burning threads or running thousands of concurrent connections against rate limits.

> Когда нужно прогнать один и тот же шаблон промпта по сотням или тысячам входных данных, синхронный API — неподходящая модель. Каждый синхронный вызов блокируется до завершения. В масштабе это означает, что ваше приложение либо расходует потоки, либо держит тысячи одновременных соединений, упираясь в лимиты запросов (rate limits).

The Message Batches API accepts up to 100,000 or 256 MB requests (whichever comes first) in a single batch call. You submit the batch, receive a `batch_id`, and poll for completion. When the batch finishes, you download the results. The per-token cost for batch requests is lower than for synchronous ones.

> Message Batches API принимает до 100 000 запросов или 256 МБ (что наступит раньше) в одном пакетном вызове. Вы отправляете пакет, получаете `batch_id` и периодически опрашиваете статус завершения. Когда пакет обработан, вы скачиваете результаты. Стоимость за токен для пакетных запросов ниже, чем для синхронных.

The tradeoff is latency: batch processing is non-deterministic and can take up to 24 hours, often much faster. The pattern suits offline pipelines, evaluation runs, and data processing jobs, not real-time user interactions.

> Компромисс — задержка: время пакетной обработки непредсказуемо и может составлять до 24 часов, хотя часто всё происходит гораздо быстрее. Этот паттерн подходит для офлайн-пайплайнов, прогонов оценки и задач обработки данных, но не для взаимодействия с пользователями в реальном времени.

| Use case | Right API pattern | Why |
|---|---|---|
| A user uploads a photo and expects an immediate classification | **Synchronous API** | Real-time response is required. Batch latency is unacceptable for interactive use. |
| A nightly pipeline classifies 5,000 customer records | **Message Batches API** | Latency is not a constraint. Batch cost reduction and asynchronous processing are both valuable. |
| An evaluation run tests a new prompt against 2,000 examples | **Message Batches API** | Offline task with no real-time requirement. Batch is the correct pattern. |
| A chatbot generates a reply to a user's message | **Synchronous API** | User is waiting; batch would introduce unacceptable delay. |

| Сценарий | Подходящий паттерн API | Почему |
|---|---|---|
| Пользователь загружает фото и ожидает немедленной классификации | **Синхронный API** | Нужен ответ в реальном времени. Задержка пакетной обработки неприемлема для интерактивного использования. |
| Ночной пайплайн классифицирует 5000 записей о клиентах | **Message Batches API** | Задержка не является ограничением. Ценны и снижение стоимости за счёт пакетной обработки, и асинхронность. |
| Прогон оценки (eval) тестирует новый промпт на 2000 примеров | **Message Batches API** | Офлайн-задача без требований к работе в реальном времени. Пакетная обработка — правильный паттерн. |
| Чат-бот генерирует ответ на сообщение пользователя | **Синхронный API** | Пользователь ждёт; пакетная обработка внесла бы неприемлемую задержку. |

## When multimodal and batch fit together, and when they don't
## Когда мультимодальность и пакетная обработка сочетаются, а когда нет

The combination works for offline workloads that reuse the same assets and need structured output across thousands of inputs. A nightly pipeline classifying images against a fixed taxonomy is the textbook case: Files API removes redundant uploads, Batches API absorbs the latency, structured-output techniques keep results machine-readable.

> Такое сочетание подходит для офлайн-нагрузок, которые многократно используют одни и те же ресурсы и требуют структурированного вывода для тысяч входных данных. Хрестоматийный пример — ночной пайплайн, классифицирующий изображения по фиксированной таксономии: Files API избавляет от повторных загрузок, Batches API поглощает задержку, а техники структурированного вывода сохраняют результаты машиночитаемыми.

Two failure modes break the fit.

- **The first is misreading latency:** reaching for batch in any user-facing flow with an image produces a system that passes tests and fails in production, because the user is waiting and the batch isn't.
- **The second is underestimating context cost:** images and PDFs consume budget before Claude processes any text, so pipelines loading multiple large images per request blow past token limits at scale. Measure token cost on production-scale inputs before you build.

> Два вида сбоев нарушают это сочетание.
>
> - **Первый — неверная оценка задержки:** использование пакетной обработки в любом пользовательском сценарии с изображением даёт систему, которая проходит тесты, но сбоит в продакшене, потому что пользователь ждёт, а пакет — нет.
> - **Второй — недооценка стоимости контекста:** изображения и PDF расходуют бюджет ещё до того, как Claude обработает какой-либо текст, поэтому пайплайны, загружающие по несколько больших изображений в каждый запрос, в масштабе превышают лимиты токенов. Измерьте стоимость в токенах на входных данных продакшен-масштаба, прежде чем начинать разработку.
