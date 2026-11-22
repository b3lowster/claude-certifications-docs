# Comparing platforms on latency, compliance, and cost so the choice survives review
# Сравнение платформ по задержке, соответствию требованиям и стоимости, чтобы выбор выдержал проверку

In the last two screens you chose a platform and pinned its version. That choice was right for the customer's cloud, but "right for their cloud" is not yet an argument a procurement and security team will sign off on.

> В двух предыдущих разделах вы выбрали платформу и зафиксировали её версию. Этот выбор был правильным для облака заказчика, но «подходит для их облака» — ещё не тот аргумент, под которым подпишутся отдел закупок и команда безопасности.

## Measure latency from the customer's region
## Измеряйте задержку из региона заказчика

Latency depends on where the platform runs relative to the customer and on how access to new features is routed. A platform running in the customer's own cloud region can reduce round-trip time compared to a first-party endpoint located farther away. The trade-off is timing of access: the first-party API typically receives new capabilities before they reach other platforms. The number is only accurate when you measure it from the customer's actual region against their actual payload. A measurement from your laptop hides the round-trip penalty that appears once the workload runs where the customer is. Within Bedrock specifically, the choice between global and regional endpoints is also the primary residency control and can affect cost. You should measure from the customer's actual region against both options before committing.

> Задержка зависит от того, где работает платформа относительно заказчика, и от того, как маршрутизируется доступ к новым возможностям. Платформа, работающая в собственном облачном регионе заказчика, может сократить время прохождения запроса туда и обратно (round-trip) по сравнению с более удалённым эндпоинтом собственного API Anthropic (first-party). Компромисс здесь во времени доступа: собственный API, как правило, получает новые возможности раньше других платформ. Значение задержки точно только тогда, когда вы измеряете его из реального региона заказчика на его реальной полезной нагрузке. Измерение с вашего ноутбука скрывает дополнительную задержку на прохождение запроса, которая проявляется, когда рабочая нагрузка выполняется там, где находится заказчик. В Bedrock выбор между глобальными и региональными эндпоинтами также является основным механизмом управления местом обработки данных и может влиять на стоимость. Прежде чем принимать решение, проведите измерения из реального региона заказчика для обоих вариантов.

## Compliance often determines the platform
## Соответствие требованиям часто определяет платформу

Compliance is often the dimension that ends the debate. A customer who already holds a certification on one cloud is unlikely to re-certify on another. Data residency is a rule that a customer's data must be processed in a specific country or region. Available compliance certifications and who can audit access differ by platform, and a regulated financial or healthcare customer treats these as pass-or-fail rather than as tradeoffs to balance. The first-party Claude API may not offer EU data residency; confirm current regional coverage at platform.claude.com, since EU-only residency typically requires Bedrock or Vertex AI; on third-party platforms such as Microsoft Foundry, hosting is per-model: Azure-hosted Foundry models run inference end-to-end on Azure infrastructure, while Anthropic-hosted Foundry models do not satisfy EU regional residency requirements. Residency must be confirmed per model and deployment with Microsoft. Raise the compliance constraint during scoping, or it surfaces at contract review after the work is done.

> Соответствие требованиям (compliance) часто становится тем критерием, который завершает спор. Заказчик, у которого уже есть сертификация в одном облаке, вряд ли станет проходить её заново в другом. Место обработки данных (data residency) — это правило, согласно которому данные заказчика должны обрабатываться в определённой стране или регионе. Доступные сертификации и то, кто может проводить аудит доступа, различаются в зависимости от платформы, и регулируемый заказчик из финансовой сферы или здравоохранения рассматривает их как критерии «прошёл / не прошёл», а не как компромиссы, которые можно взвешивать. Собственный Claude API может не предоставлять обработку данных в ЕС; уточняйте актуальное региональное покрытие на platform.claude.com, поскольку обработка данных только в ЕС, как правило, требует Bedrock или Vertex AI. На сторонних платформах, таких как Microsoft Foundry, размещение определяется для каждой модели отдельно: модели Foundry с размещением в Azure выполняют инференс полностью на инфраструктуре Azure, а модели Foundry с размещением у Anthropic не удовлетворяют требованиям о региональной обработке данных в ЕС. Место обработки данных нужно подтверждать у Microsoft для каждой модели и каждого развёртывания. Поднимайте вопрос об ограничениях по соответствию требованиям на этапе определения рамок проекта (scoping), иначе он всплывёт при проверке контракта, когда работа уже будет сделана.

## What drives total cost beyond the per-token rate
## Что определяет общую стоимость помимо цены за токен

Per-token rates are broadly aligned across platforms; total cost moves on egress, platform fees, and integration effort. A lower token price can cost more in total once data transfer and integration are factored in. Instrument cost per call for each platform. Confirm the current pricing pages at scoping.

> Цены за токен на разных платформах в целом сопоставимы; общая стоимость меняется за счёт исходящего трафика (egress), платформенных сборов и трудозатрат на интеграцию. Более низкая цена за токен может в итоге обойтись дороже, если учесть передачу данных и интеграцию. Настройте измерение стоимости каждого вызова для каждой платформы. Сверяйтесь с актуальными страницами цен на этапе определения рамок проекта.

## The cross-platform comparison reference
## Справочная таблица для сравнения платформ

| Dimension | How it differs by platform | How to measure it | Where each platform wins |
|---|---|---|---|
| Latency | A platform in the customer's region shortens the round trip, while the first-party API may reach new features first. | From the customer's actual region against their actual payload. | An in-region cloud platform wins on round-trip latency, while the first-party API is advantaged on earliest feature access. |
| Compliance | Data residency, certifications, and audit controls are determined by the deployment platform. | Against the customer's existing certification and residency requirements during scoping. | The cloud platform the customer has already certified wins, because it needs no re-certification. |
| Cost | Token price, data egress, platform fees, and integration effort all vary. | Total cost per call per platform, including egress and integration, rather than token price alone. | The platform with the lowest total cost for the actual workload wins, which is not always the cheapest token. |

| Критерий | Чем различаются платформы | Как измерять | Где выигрывает каждая платформа |
|---|---|---|---|
| Задержка | Платформа в регионе заказчика сокращает время прохождения запроса туда и обратно, тогда как собственный API (first-party) может раньше получать новые возможности. | Из реального региона заказчика на его реальной полезной нагрузке. | Облачная платформа в регионе заказчика выигрывает по задержке прохождения запроса, а собственный API имеет преимущество в самом раннем доступе к новым возможностям. |
| Соответствие требованиям | Место обработки данных, сертификации и средства аудита определяются платформой развёртывания. | Сопоставляя с существующей сертификацией заказчика и его требованиями к месту обработки данных на этапе определения рамок проекта. | Выигрывает облачная платформа, для которой у заказчика уже есть сертификация, поскольку повторная сертификация не нужна. |
| Стоимость | Различаются цена за токен, исходящий трафик (egress), платформенные сборы и трудозатраты на интеграцию. | Общая стоимость одного вызова на каждой платформе, включая исходящий трафик и интеграцию, а не только цена за токен. | Выигрывает платформа с наименьшей общей стоимостью для реальной рабочей нагрузки, и это не всегда платформа с самым дешёвым токеном. |

| | |
|---|---|
| **Handles well** | Measuring all three dimensions per platform turns a placement into one a procurement team will sign off on. |
| **Adds cost or complexity** | Instrumenting latency, compliance, and cost across platforms requires real measurement work before any code ships. |
| **Use a different approach** | When the customer's compliance requirement is already pass-or-fail, skip the full comparison. That constraint determines the placement on its own. |

| | |
|---|---|
| **Хорошо подходит** | Если измерить все три критерия для каждой платформы, выбор размещения становится таким, под которым подпишется отдел закупок. |
| **Добавляет стоимость или сложность** | Чтобы измерить задержку, соответствие требованиям и стоимость на разных платформах, нужна реальная работа по измерениям ещё до выпуска какого-либо кода. |
| **Используйте другой подход** | Если требование заказчика по соответствию уже работает по принципу «прошёл / не прошёл», полное сравнение можно пропустить. Это ограничение само определяет размещение. |
