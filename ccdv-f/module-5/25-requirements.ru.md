# From business requirements to functional and infrastructure requirements
# От бизнес-требований к функциональным и инфраструктурным требованиям

The deployment-platform decisions that follow all assume the requirements already exist: the residency rule, the latency target, the identity model. This screen is where those requirements come from: turning a business problem into the functional and infrastructure requirements a deployment decision can be defended against.

> Все последующие решения о платформе развёртывания предполагают, что требования уже существуют: правило о месте обработки данных (residency), целевая задержка (latency), модель идентификации. В этом разделе рассказывается, откуда берутся эти требования: как превратить бизнес-задачу в функциональные и инфраструктурные требования, на которые можно опереться при обосновании решения о развёртывании.

## Capturing functional requirements from a business problem
## Выявление функциональных требований из бизнес-задачи

A functional requirement names what the system must do, stated with enough detail to check. A business problem (e.g. "help support agents answer faster") is not yet a requirement; the functional requirements derive from it (e.g. "classify each ticket into one of four queues; draft a reply citing the relevant policy; never auto-send without human approval"). The discipline is to write each as a checkable statement of behavior. A vague goal cannot be designed against or verified, while a specific one becomes a line in an eval and a criterion at review.

> Функциональное требование определяет, что должна делать система, и сформулировано достаточно подробно, чтобы его можно было проверить. Бизнес-задача (например, «помочь сотрудникам поддержки отвечать быстрее») — это ещё не требование; функциональные требования выводятся из неё (например, «относить каждое обращение к одной из четырёх очередей; составлять черновик ответа со ссылкой на соответствующую политику; никогда не отправлять ответ автоматически без одобрения человека»). Суть в том, чтобы записывать каждое требование как проверяемое описание поведения. По расплывчатой цели нельзя ни проектировать, ни проверять, а конкретная становится строкой в оценке (eval) и критерием при проверке.

## Deriving infrastructure requirements
## Выведение инфраструктурных требований

Infrastructure requirements are the non-functional constraints the deployment must satisfy. Most of them are not stated in the business problem; instead, you derive them by asking the questions the business problem implies. Latency: how fast does a response need to be, measured where the user is? Scale: how many requests, and at what peak? Residency: where must the data be processed, and under which regulation? Identity: who acts, under what credentials, and what must be auditable? Latency, scale, residency, and identity are the infrastructure requirements that most often decide the deployment platform, and they are easiest to capture at the start, before a platform is chosen for other reasons.

> Инфраструктурные требования — это нефункциональные ограничения, которым должно удовлетворять развёртывание. Большинство из них в бизнес-задаче не сформулированы; вы выводите их, задавая вопросы, которые подразумевает бизнес-задача. Задержка: насколько быстрым должен быть ответ, если измерять там, где находится пользователь? Масштаб: сколько запросов и каков пик? Место обработки данных: где должны обрабатываться данные и по какому регламенту? Идентификация: кто действует, с какими учётными данными и что должно подлежать аудиту? Задержка, масштаб, место обработки данных и идентификация — инфраструктурные требования, которые чаще всего определяют платформу развёртывания, и их проще всего зафиксировать в самом начале, до того как платформа будет выбрана по другим причинам.

## Documenting requirements so a decision can be defended
## Документирование требований для обоснования решения

Requirements are written down because the deployment decision will be reviewed by people who did not gather them. A short requirements record covering the functional behaviors, the infrastructure constraints, and the regulation each constraint comes from lets you defend a platform choice as following from the requirements rather than from familiarity. This record is the input the next screen's deployment decision reads from.

> Требования записывают, потому что решение о развёртывании будут проверять люди, которые их не собирали. Краткая запись требований, охватывающая функциональное поведение, инфраструктурные ограничения и регламент, из которого вытекает каждое ограничение, позволяет обосновать выбор платформы как следствие требований, а не привычки. Эта запись — входные данные для решения о развёртывании в следующем разделе.

| | |
|---|---|
| **Handles well** | Turning a business problem into checkable functional and infrastructure requirements before any platform is chosen. |
| **Adds cost or complexity** | Eliciting infrastructure constraints up front takes a scoping conversation the team is tempted to skip. |
| **Use a different approach** | For a throwaway prototype with no review and no regulated data, lightweight notes are enough. |

| | |
|---|---|
| **Хорошо подходит** | Превращение бизнес-задачи в проверяемые функциональные и инфраструктурные требования до выбора какой-либо платформы. |
| **Добавляет стоимость или сложность** | Чтобы заранее выявить инфраструктурные ограничения, нужен разговор об определении рамок проекта, который команде хочется пропустить. |
| **Используйте другой подход** | Для одноразового прототипа без проверки и без регулируемых данных достаточно простых заметок. |
