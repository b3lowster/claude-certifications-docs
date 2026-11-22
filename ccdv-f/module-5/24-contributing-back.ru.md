# Moving an asset from private reuse into shared infrastructure a maintainer accepts
# Перевод актива из частного повторного использования в общую инфраструктуру, которую примет мейнтейнер

You have already done most of the work that makes an asset shareable. When you packaged it for your own team to reuse, you pulled out the parameters, wrote down the assumptions, and bundled the eval. The parameters show the asset can be configured rather than rewritten. The documented assumptions tell the maintainer what environment the asset expects. The bundled eval gives them a way to confirm it still works. An asset packaged for internal reuse is already close to what a maintainer needs to accept it.

> Вы уже проделали большую часть работы, которая позволяет делиться активом. Когда вы упаковывали его для повторного использования своей командой, вы вынесли параметры, записали допущения и включили в пакет оценку (eval). Параметры показывают, что актив можно настраивать, а не переписывать. Задокументированные допущения сообщают мейнтейнеру (maintainer), какую среду ожидает актив. Включённая оценка даёт ему способ убедиться, что актив по-прежнему работает. Актив, упакованный для внутреннего повторного использования, уже близок к тому, что нужно мейнтейнеру, чтобы его принять.

The contribution channel is designed to receive that packaged asset. It carries the version, the installation steps, and the components as a single unit, so a team that never spoke to you can install it and get the same working setup.

> Канал для внесения вклада рассчитан именно на такой упакованный актив. Он передаёт версию, шаги установки и компоненты как единое целое, поэтому команда, которая никогда с вами не общалась, может установить его и получить ту же рабочую конфигурацию.

## Match the contribution to the channel built for it
## Подбирайте вклад под предназначенный для него канал

Contributing back means moving an asset from private reuse to shared infrastructure through a documented channel. Each channel is built for a specific kind of contribution. The Claude Cookbook is a GitHub repository of focused reference implementations. It is designed for self-contained single- or multi-pattern implementations demonstrated clearly and working end to end. Open-source MCP servers and tools each live in their own repository with their own contribution conventions. Sending a full multi-component application to the Cookbook is a mismatch. The repository is set up to review one focused pattern rather than an entire application, so a submission that large does not fit what reviewers are looking for and will stall. The first step is matching the contribution to the channel built for it. Putting a full application where a focused example belongs is one of the most common reasons a contribution never gets reviewed.

> Внести вклад (contributing back) — значит перевести актив из частного повторного использования в общую инфраструктуру через задокументированный канал. Каждый канал рассчитан на определённый вид вклада. Claude Cookbook — это репозиторий GitHub с узконаправленными эталонными реализациями. Он предназначен для самодостаточных реализаций одного или нескольких подходов, которые наглядно продемонстрированы и работают от начала до конца. Открытые MCP-серверы и инструменты находятся каждый в своём репозитории со своими правилами внесения вклада. Отправлять в Cookbook полноценное многокомпонентное приложение — несоответствие. Репозиторий настроен на ревью одного узконаправленного подхода, а не целого приложения, поэтому такая объёмная заявка не соответствует тому, что ищут ревьюеры, и застрянет. Первый шаг — подобрать вклад под предназначенный для него канал. Попытка разместить полноценное приложение там, где ожидается узконаправленный пример, — одна из самых частых причин, по которым вклад так и не проходит ревью.

## What makes verifying a contribution possible
## Что делает возможной проверку вклада

A maintainer accepts a contribution they can verify. The bar is set by what they need to check, not by how clever the code is. Four things make that verification possible:

> Мейнтейнер принимает вклад, который может проверить. Планку задаёт то, что ему нужно проверить, а не то, насколько изобретателен код. Проверку делают возможной четыре вещи:

1. **The code does one thing.** A sprawling contribution forces a reviewer to reconstruct your intent before evaluating it.
2. **An example shows it running.** A reviewer should not have to build a harness to see the behavior.
3. **A test proves it works.** A test lets a maintainer verify the result without reproducing the reasoning themselves.
4. **A short statement names the assumptions.** Otherwise, the first failure becomes the maintainer's problem.

> 1. **Код делает одну вещь.** Разросшийся вклад заставляет ревьюера восстанавливать ваш замысел, прежде чем его оценить.
> 2. **Пример показывает его в работе.** Ревьюеру не должно требоваться создавать тестовую обвязку (harness), чтобы увидеть поведение.
> 3. **Тест доказывает, что он работает.** Тест позволяет мейнтейнеру проверить результат, не воспроизводя рассуждения самостоятельно.
> 4. **Краткое описание называет допущения.** Иначе первый же сбой станет проблемой мейнтейнера.

## Rights and attribution come before technical review
## Права и указание авторства — до технического ревью

Licensing and attribution decide whether a contribution can be accepted at all, which is why they come before the technical review. Code carried in from a customer engagement may have constraints on where it can go. Confirming you have the right to contribute it, and attributing anything you built on, is a gate the contribution must pass first. Skipping this is what turns a contribution into a problem the legal team must unwind later.

> Лицензирование и указание авторства (attribution) определяют, можно ли вообще принять вклад, поэтому они идут до технического ревью. На код, перенесённый из клиентского проекта, могут распространяться ограничения на то, где его можно использовать. Подтверждение того, что у вас есть право внести этот код, и указание авторства всего, на чём вы строили, — это контрольная точка, которую вклад должен пройти первым. Если её пропустить, вклад превращается в проблему, которую потом придётся распутывать юристам.

The example worked here is the customer service agent case. A reusable conversation-handling pattern, built during an engagement, gets stripped of customer specifics and prepared as a general example for the Cookbook. The contribution-back motion is shared across all three roles in this curriculum. Your job as the Developer is technical readiness: the focused code, the example, the test, the assumptions, and the rights check. The engagement context comes from the broader team.

> Разбираемый здесь пример — случай с агентом клиентской поддержки. Повторно используемый подход к ведению диалога, созданный в рамках проекта, очищается от специфики клиента и готовится как общий пример для Cookbook. Процесс внесения вклада общий для всех трёх ролей этой учебной программы. Ваша задача как разработчика (Developer) — техническая готовность: узконаправленный код, пример, тест, допущения и проверка прав. Контекст проекта обеспечивает остальная команда.

## The contribution-readiness reference
## Справочник по готовности вклада

| | |
|---|---|
| **Channel** | Cookbook for a focused example, or the tool or server's own repository for a tool or fix. |
| **What a maintainer checks** | That the code does one thing and that they can read it in full. |
| **Licensing and attribution** | Confirm that you have the right to contribute code from an engagement, with prior work attributed. |
| **The example and test bar to clear** | A runnable example plus a test that proves the behavior, not just a description of it. |

| | |
|---|---|
| **Канал** | Cookbook — для узконаправленного примера, собственный репозиторий инструмента или сервера — для инструмента или исправления. |
| **Что проверяет мейнтейнер** | Что код делает одну вещь и что его можно прочитать целиком. |
| **Лицензирование и указание авторства** | Подтвердите, что у вас есть право вносить код из клиентского проекта, с указанием авторства использованных ранее работ. |
| **Планка для примера и теста** | Запускаемый пример плюс тест, который доказывает поведение, а не просто его описывает. |

| | |
|---|---|
| **Handles well** | A packaged asset needs only the example, test, and rights check to become shared infrastructure others build on. |
| **Adds cost or complexity** | Clearing the maintainer bar and the licensing gate is real work on top of making the code run for you. |
| **Use a different approach** | When code carries an engagement licensing constraint you cannot clear, do not contribute it: escalate to the owner instead. |

| | |
|---|---|
| **Хорошо подходит** | Упакованному активу нужны только пример, тест и проверка прав, чтобы стать общей инфраструктурой, на которой строят другие. |
| **Добавляет стоимость или сложность** | Преодолеть планку мейнтейнера и пройти лицензионную проверку — реальная работа сверх того, чтобы код просто работал у вас. |
| **Используйте другой подход** | Если на код распространяется лицензионное ограничение клиентского проекта, которое вы не можете снять, не вносите его: передайте вопрос владельцу. |
