# Choosing where a Claude workload runs and versioning what ships
# Выбор места запуска рабочей нагрузки Claude и версионирование того, что выпускается

A packaged asset and a contributed one are both merely code until something runs them. The asset now faces a different question: where it runs and how to lock its version, so an upstream change does not become an untracked change in production. That platform decision is rarely about technical merit alone. In practice, it is usually shaped by where the customer already has cloud infrastructure, identity management, and compliance agreements in place. The first question is usually about which platform the customer already trusts and operates on.

> И упакованный актив, и актив, переданный в общий доступ, остаются просто кодом, пока что-то их не запустит. Теперь перед активом встаёт другой вопрос: где он будет выполняться и как зафиксировать его версию, чтобы изменение у поставщика (upstream) не превратилось в неотслеживаемое изменение в продакшене. Выбор платформы редко определяется только техническими достоинствами. На практике он обычно зависит от того, где у заказчика уже есть облачная инфраструктура, управление идентификацией и соглашения о соответствии требованиям (compliance). Поэтому первый вопрос обычно звучит так: какой платформе заказчик уже доверяет и на какой работает.

## The customer's cloud usually determines the platform
## Платформу обычно определяет облако заказчика

The deployment platform is the environment where the Claude workload runs. The same model can run in several deployment environments, and the customer's existing cloud usually determines which one. The first-party Claude API is Anthropic's own environment and typically receives new features first. Claude Platform on AWS is accessed through the customer's AWS account using Anthropic's own model IDs and lifecycle; inference is Anthropic-operated, outside the AWS boundary. Amazon Bedrock offers two integrations: Claude in Amazon Bedrock uses the Messages API at /anthropic/v1/messages with broad feature parity; confirm any feature-specific requirements against the Bedrock documentation, as a features-not-supported list exists, while Claude on Amazon Bedrock (legacy) uses the InvokeModel/Converse APIs with ARN-versioned identifiers. Google Vertex AI does the same inside Google Cloud. Third-party platforms, such as Microsoft Foundry, embed Claude inside a product the customer already uses. Microsoft Foundry offers Claude in two hosting forms: Hosted on Azure (currently Claude Opus 4.8, Claude Sonnet 5, and Claude Haiku 4.5, with inference running end-to-end on Azure infrastructure, generally available) and Hosted on Anthropic (all other Foundry Claude models, with inference on Anthropic-operated infrastructure). Residency assumptions for regulated customers depend on the hosting form of the specific model. Confirm the hosting form and the current model split with Microsoft at build time.

> Платформа развёртывания — это среда, в которой выполняется рабочая нагрузка Claude. Одна и та же модель может работать в нескольких средах развёртывания, и выбор среды обычно определяется облаком, которое у заказчика уже есть. Собственный (first-party) Claude API — это среда самой Anthropic, и новые возможности, как правило, появляются в ней первыми. Claude Platform on AWS доступна через AWS-аккаунт заказчика и использует собственные идентификаторы моделей Anthropic и её жизненный цикл моделей; инференс выполняется на инфраструктуре Anthropic, за пределами периметра AWS. Amazon Bedrock предлагает две интеграции: Claude in Amazon Bedrock использует Messages API по адресу /anthropic/v1/messages и в целом поддерживает те же возможности (при этом требования к конкретным возможностям нужно сверять с документацией Bedrock, так как там есть список неподдерживаемых функций), а Claude on Amazon Bedrock (устаревшая интеграция) использует API InvokeModel/Converse и идентификаторы с версией в ARN. Google Vertex AI делает то же самое внутри Google Cloud. Сторонние платформы, например Microsoft Foundry, встраивают Claude в продукт, которым заказчик уже пользуется. Microsoft Foundry предлагает Claude в двух вариантах размещения: Hosted on Azure (сейчас это Claude Opus 4.8, Claude Sonnet 5 и Claude Haiku 4.5; инференс полностью выполняется на инфраструктуре Azure, вариант общедоступен) и Hosted on Anthropic (все остальные модели Claude в Foundry; инференс на инфраструктуре Anthropic). Допущения о месте обработки данных для регулируемых заказчиков зависят от варианта размещения конкретной модели. Уточняйте вариант размещения и актуальное распределение моделей у Microsoft на момент разработки.

## Identity and data residency are important for security
## Идентификация и место обработки данных важны для безопасности

Identity and data location are answered by the platform, not your code. Bedrock uses AWS identity and keeps data inside the customer's AWS boundary; Vertex uses Google Cloud identity and boundary. Both offer regional routing when residency is a constraint. Matching the platform to the customer's existing compliance agreement avoids a data-residency review from scratch.

> Вопросы идентификации и местонахождения данных решает платформа, а не ваш код. Bedrock использует идентификацию AWS и держит данные внутри периметра AWS заказчика; Vertex использует идентификацию и периметр Google Cloud. Обе платформы поддерживают региональную маршрутизацию, если место обработки данных является ограничением. Если подобрать платформу под уже существующее у заказчика соглашение о соответствии требованиям, не придётся проводить проверку места обработки данных с нуля.

## Pin the version so an upstream model change is not a silent production change
## Фиксируйте версию, чтобы обновление модели у поставщика не стало незаметным изменением в продакшене

Versioning is what keeps a model or prompt change from becoming a silent change in production. Every Claude model ID points to a specific model snapshot. Aliases such as Opus and Sonnet are convenient, but they evolve over time and may resolve to different versions across deployment platforms. A pinned full model ID resolves to a fixed snapshot. Pin the specific model version rather than the alias, so an upstream model update is a deliberate choice rather than a silent production change. Then version the prompt and the asset alongside the code. Finally, keep the prior version available so the regression can be rolled back. An unpinned deployment makes every upstream model update an untracked change to your output.

> Именно версионирование не даёт изменению модели или промпта превратиться в незаметное изменение в продакшене. Каждый идентификатор модели Claude указывает на конкретный снимок (snapshot) модели. Псевдонимы (алиасы), такие как Opus и Sonnet, удобны, но со временем меняются и на разных платформах развёртывания могут указывать на разные версии. Полный зафиксированный идентификатор модели всегда указывает на один и тот же снимок. Фиксируйте конкретную версию модели, а не псевдоним, чтобы обновление модели у поставщика было осознанным решением, а не незаметным изменением в продакшене. Затем версионируйте промпт и актив вместе с кодом. Наконец, держите предыдущую версию доступной, чтобы при регрессии можно было откатиться. При развёртывании без фиксации версии каждое обновление модели у поставщика становится неотслеживаемым изменением ваших результатов.

```python
# Pre-4.6 example: a convenience alias can resolve to a new
# version without you knowing
model = "claude-haiku-4-5"

# Pre-4.6 pinned snapshot: the version is fixed until you change this line
model = "claude-haiku-4-5-20251001"
```

For Claude 4.6 and later, the model ID alone pins to a specific snapshot; for earlier models, the ID plus a date suffix is required. Verify the current convention at platform.claude.com at build time.

> Для Claude 4.6 и более поздних моделей идентификатор модели сам по себе фиксирует конкретный снимок; для более ранних моделей нужен идентификатор с суффиксом даты. Проверяйте актуальное соглашение об именовании на platform.claude.com на момент разработки.

## Promote a version through the eval
## Продвигайте версию через оценку (eval)

Gate promotion on the eval suite. Send a new version to a portion of traffic, compare against the pinned baseline, and promote or roll back on the result. This is where the eval stops being a one-time test and becomes the deployment gate.

> Сделайте набор оценок (eval suite) условием продвижения версии. Направьте часть трафика на новую версию, сравните её с зафиксированной базовой версией и по результату либо продвигайте её, либо откатывайтесь. Именно здесь оценка перестаёт быть разовым тестом и становится контрольной точкой развёртывания.

## Deployment platforms compared
## Сравнение платформ развёртывания

| Platform | Identity and data model | When to choose it | How versioning is pinned |
|---|---|---|---|
| First-party Claude API | Anthropic identity and terms. | The customer has no binding cloud or residency constraint and wants the newest capabilities. | Pin the full model ID and keep the prior snapshot. |
| Claude Platform on AWS | Anthropic identity and terms, accessed through the customer's AWS account; inference is Anthropic-operated outside the AWS boundary. Model lifecycle follows Anthropic's deprecation schedule. | The customer is on AWS but wants Anthropic model IDs, lifecycle, and feature parity with the first-party API. | Pin using the same model ID format as the Claude API (for example, claude-opus-4-8). Lifecycle follows Anthropic's schedule. (Confirm at publish time.) |
| Claude in Amazon Bedrock | Messages API at /anthropic/v1/messages, broad feature parity with the first-party API; confirm feature-specific requirements against the Bedrock documentation. Data stays inside the customer's configured AWS boundary. | The customer is on AWS, wants broad feature parity with the first-party API (confirm feature-specific requirements), and holds a compliance posture there. | Pin the full model ID using the anthropic. prefix format. Partner retirement dates differ from Anthropic's schedule. Confirm at publish time. |
| Claude on Amazon Bedrock (legacy) | AWS identity and billing, InvokeModel/Converse APIs with ARN-versioned model identifiers. | The customer is on an existing Bedrock integration using InvokeModel or Converse and has not migrated to the Messages API. | Pin via ARN-versioned model identifiers per Bedrock's versioning controls. |
| Google Vertex AI | Google Cloud identity, Identity and Access Management (IAM), and billing, with regional or global endpoints for residency. | The customer is on Google Cloud and holds a compliance posture there. | Pin the full model ID before rollout using Vertex's model ID format. Partner retirement dates differ from Anthropic's schedule. |
| Third-party platform | The wrapping product's identity and billing model. Note: Claude in Microsoft Foundry offers two hosting forms: Hosted on Azure (currently Opus 4.8, Sonnet 5, and Haiku 4.5; inference end-to-end on Azure) and Hosted on Anthropic (all other Foundry Claude models). Confirm residency and compliance terms with Microsoft before selecting this path for a regulated customer. | The customer already runs the platform that embeds Claude. | Pin per the platform's versioning controls. |

| Платформа | Модель идентификации и данных | Когда выбирать | Как фиксируется версия |
|---|---|---|---|
| Собственный Claude API (first-party) | Идентификация и условия Anthropic. | У заказчика нет обязательных ограничений по облаку или месту обработки данных, и ему нужны новейшие возможности. | Зафиксируйте полный идентификатор модели и сохраните предыдущий снимок. |
| Claude Platform on AWS | Идентификация и условия Anthropic, доступ через AWS-аккаунт заказчика; инференс выполняется на инфраструктуре Anthropic за пределами периметра AWS. Жизненный цикл моделей следует графику вывода из эксплуатации Anthropic. | Заказчик работает в AWS, но хочет идентификаторы моделей Anthropic, её жизненный цикл и паритет возможностей с собственным API. | Фиксируйте версию в том же формате идентификатора, что и в Claude API (например, claude-opus-4-8). Жизненный цикл следует графику Anthropic. (Уточните на момент публикации.) |
| Claude in Amazon Bedrock | Messages API по адресу /anthropic/v1/messages, в целом паритет возможностей с собственным API; требования к конкретным возможностям сверяйте с документацией Bedrock. Данные остаются внутри настроенного периметра AWS заказчика. | Заказчик работает в AWS, хочет в целом паритет возможностей с собственным API (уточните требования к конкретным возможностям) и имеет там подтверждённое соответствие требованиям. | Зафиксируйте полный идентификатор модели в формате с префиксом anthropic. Даты вывода из эксплуатации у партнёра отличаются от графика Anthropic. Уточните на момент публикации. |
| Claude on Amazon Bedrock (устаревшая интеграция) | Идентификация и биллинг AWS, API InvokeModel/Converse с идентификаторами моделей, версия которых задаётся в ARN. | У заказчика уже есть интеграция с Bedrock через InvokeModel или Converse, и он не перешёл на Messages API. | Фиксируйте через идентификаторы моделей с версией в ARN согласно механизмам версионирования Bedrock. |
| Google Vertex AI | Идентификация Google Cloud, Identity and Access Management (IAM) и биллинг, с региональными или глобальными эндпоинтами для требований к месту обработки данных. | Заказчик работает в Google Cloud и имеет там подтверждённое соответствие требованиям. | Зафиксируйте полный идентификатор модели до выката в формате идентификаторов Vertex. Даты вывода из эксплуатации у партнёра отличаются от графика Anthropic. |
| Сторонняя платформа | Модель идентификации и биллинга продукта-обёртки. Примечание: Claude в Microsoft Foundry доступен в двух вариантах размещения: Hosted on Azure (сейчас Opus 4.8, Sonnet 5 и Haiku 4.5; инференс полностью на Azure) и Hosted on Anthropic (все остальные модели Claude в Foundry). Прежде чем выбирать этот путь для регулируемого заказчика, уточните у Microsoft условия по месту обработки данных и соответствию требованиям. | Заказчик уже использует платформу, в которую встроен Claude. | Фиксируйте версию средствами версионирования самой платформы. |

| | |
|---|---|
| **Handles well** | Matching the platform to the customer cloud and pinning the version keeps a migration reviewable and a rollback possible. |
| **Adds cost or complexity** | Pinning, retaining prior versions, and gating promotion on the eval add release-process overhead to every deployment. |
| **Use a different approach** | For a throwaway prototype that never touches production, a moving alias is fine: pinning is for what ships. |

| | |
|---|---|
| **Хорошо подходит** | Если подобрать платформу под облако заказчика и зафиксировать версию, миграцию можно проверить, а откат остаётся возможным. |
| **Добавляет стоимость или сложность** | Фиксация версии, хранение предыдущих версий и продвижение только после прохождения оценки (eval) добавляют накладные расходы на процесс выпуска к каждому развёртыванию. |
| **Используйте другой подход** | Для одноразового прототипа, который никогда не попадает в продакшен, подвижный псевдоним вполне подходит: фиксация версии нужна для того, что выпускается. |
