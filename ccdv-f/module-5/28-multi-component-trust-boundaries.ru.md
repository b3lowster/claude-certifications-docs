# Coordinating several Claude deployments with the trust boundaries holding under review
# Координация нескольких развёртываний Claude с сохранением границ доверия при проверке

The accelerators, deployments, and tradeoffs now come together in a single application. Connecting components multiplies the places where identity, secrets, and untrusted input can cross. The discipline is to identify every boundary before connecting anything.

> Акселераторы, развёртывания и компромиссы теперь объединяются в одном приложении. Соединение компонентов умножает число мест, через которые могут проходить идентификационные данные, секреты и недоверенный ввод. Правило состоит в том, чтобы определить каждую границу до того, как что-либо соединять.

## Map which component does what before you connect them
## Определите, какой компонент что делает, прежде чем их соединять

A multi-component app coordinates more than one Claude capability into a single workflow. An API request might trigger a Claude Code task, which then reaches a customer system through an MCP server. Each component contributes a capability the others do not have. The challenge is that every connection between them creates a place where identity, secrets, and untrusted input can cross. Map which component does what before connecting anything.

> Многокомпонентное приложение объединяет несколько возможностей Claude в один рабочий процесс. Запрос к API может запустить задачу Claude Code, которая затем обращается к системе заказчика через MCP-сервер. Каждый компонент даёт возможность, которой нет у остальных. Сложность в том, что каждое соединение между ними создаёт место, через которое могут проходить идентификационные данные, секреты и недоверенный ввод. Прежде чем что-либо соединять, определите, какой компонент что делает.

## The trust boundary is where data moves
## Граница доверия — там, где перемещаются данные

The trust boundary is the point where data or instructions move from one deployment environment to another. It is exactly where the injection and access controls from the prior module apply. Content fetched by a Claude Code task is untrusted when it reaches the next component. The receiving component should treat it as data, rather than as instructions, following the same principle used throughout the security module. The core discipline here is to identify every seam as a boundary. Don't assume a component is trusted simply because it worked correctly on its own.

> Граница доверия — это точка, в которой данные или инструкции переходят из одной среды развёртывания в другую. Именно здесь применяются средства защиты от внедрения (injection) и контроля доступа из предыдущего модуля. Контент, полученный задачей Claude Code, является недоверенным, когда он попадает в следующий компонент. Принимающий компонент должен рассматривать его как данные, а не как инструкции, следуя тому же принципу, что и во всём модуле по безопасности. Главное правило здесь — считать каждый стык границей. Не считайте компонент доверенным только потому, что он корректно работал сам по себе.

## Least privilege applies to the whole application
## Принцип минимальных привилегий применяется ко всему приложению

Identity and least privilege, which means giving each component only the access its task needs and nothing more, apply to the application as a whole. Each component operates under an identity. The application is only as contained as its most privileged seam, which means a single component scoped too broadly becomes the weak point even when every other component is properly scoped. You scope each component to the least privilege its role in the workflow requires. This is what keeps a steered component from reaching beyond its intended task.

> Идентификация и принцип минимальных привилегий (то есть предоставление каждому компоненту только того доступа, который нужен для его задачи, и не более) применяются к приложению в целом. Каждый компонент работает под определённой учётной записью (identity). Приложение изолировано лишь настолько, насколько изолирован его самый привилегированный стык: один компонент со слишком широкими правами становится слабым местом, даже если права всех остальных компонентов настроены правильно. Ограничивайте права каждого компонента минимумом, который требуется для его роли в рабочем процессе. Именно это не даёт компоненту, которым манипулируют извне, выйти за пределы своей задачи.

## Scoping for a regulated review pulls the module together
## Подготовка к проверке в регулируемой среде объединяет весь модуль

A regulated review requires justifying audit logging, data-residency decisions, and permission controls across the full application. For regulated deployments, Bedrock and Vertex AI are typically the platforms that satisfy regional residency constraints. Confirm ZDR and HIPAA BAA eligibility for each component against the Anthropic Trust Center and platform.claude.com before scoping.

> Проверка в регулируемой среде требует обосновать журналирование для аудита, решения о месте обработки данных и контроль разрешений во всём приложении. Для регулируемых развёртываний платформами, удовлетворяющими ограничениям на региональную обработку данных, как правило, являются Bedrock и Vertex AI. Прежде чем определять рамки проекта, проверьте применимость ZDR (Zero Data Retention, нулевое хранение данных) и HIPAA BAA (соглашение с бизнес-партнёром по HIPAA) для каждого компонента по Anthropic Trust Center и platform.claude.com.

## The multi-component integration map
## Карта интеграции многокомпонентного приложения

| Component | What it contributes | The trust boundary at its seam | The control that enforces it |
|---|---|---|---|
| First-party API | Orchestrates the workflow and holds the entry point. | The request entering the app from outside. | Input validation and the identity the call runs under. |
| Claude Code task | Runs the agentic work and may fetch external content. | Content it fetched, which is untrusted downstream. | Treat fetched content as data at the next seam. |
| MCP server | Reaches a customer system to read or act. | The system access it holds on the app's behalf. | Scope the server to least privilege and log the access. |

| Компонент | Что он даёт | Граница доверия на его стыке | Механизм контроля, который её обеспечивает |
|---|---|---|---|
| Собственный API (first-party) | Оркестрирует рабочий процесс и служит точкой входа. | Запрос, поступающий в приложение извне. | Проверка входных данных и учётная запись (identity), под которой выполняется вызов. |
| Задача Claude Code | Выполняет агентную работу и может получать внешний контент. | Полученный ею контент, который является недоверенным для следующих компонентов. | Рассматривать полученный контент как данные на следующем стыке. |
| MCP-сервер | Обращается к системе заказчика, чтобы читать данные или выполнять действия. | Доступ к системе, которым он обладает от имени приложения. | Ограничить права сервера минимально необходимыми и журналировать доступ. |

| | |
|---|---|
| **Handles well** | Naming every seam as a boundary and scoping each component to least privilege makes a multi-component app deployable under review. |
| **Adds cost or complexity** | Mapping seams, enforcing controls at each, and logging boundary crossings adds design and audit work to every integration. |
| **Use a different approach** | When a seam cannot be secured, do not ship around it: escalate to a human owner. |

| | |
|---|---|
| **Хорошо подходит** | Если считать каждый стык границей и ограничивать права каждого компонента минимально необходимыми, многокомпонентное приложение можно развернуть так, чтобы оно прошло проверку. |
| **Добавляет стоимость или сложность** | Составление карты стыков, контроль на каждом из них и журналирование пересечений границ добавляют работу по проектированию и аудиту к каждой интеграции. |
| **Используйте другой подход** | Если стык невозможно защитить, не обходите его при выпуске: передайте вопрос ответственному человеку. |
