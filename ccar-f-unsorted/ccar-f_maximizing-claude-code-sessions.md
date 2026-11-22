# CCAR-F Study Notes — "Maximizing the value of your Claude Code sessions"

**Source:** https://claude.com/blog/maximizing-the-value-of-your-claude-code-sessions
**Author:** Lydia Hallie · **Published:** August 14, 2026 · **Category:** Claude Code

---

## 1. Core idea

With agentic tools, the *same* completed task can cost different amounts depending on how the session is run. Token efficiency does **not** mean using fewer tokens overall — it means making sure the tokens you spend go toward the thing you actually asked for.

Session cost model = **how many tokens end up in the context × how many turns they stay there × how many contexts run at the same time.**

---

## 2. What decides the PRICE of a token

You pay for inference time on the GPU. Three factors:

| Factor | Effect |
|---|---|
| **Model** | Bigger model = more work per input and output token; it multiplies everything else. Big model for genuinely hard/ambiguous work, small model for routine work. |
| **Input vs output** | Output is priced at roughly **5× input**. |
| **Cached or not** | Cache read = **0.1× input price**. Cache write = up to **2× input price** (one time per token). |

### Two phases of a request
- **Prefill** — the model reads the request and context: system prompt, `CLAUDE.md`, your message, plus everything added to the conversation since (files read, command output). → **input tokens**
- **Decode** — the model writes: thinking, tool calls, visible text. Generated one token at a time (a 200-token answer = 200 sequential model runs), so decode occupies the GPU far longer per token. → **output tokens**

A large share of output tokens are **thinking tokens**; the amount of thinking per turn is controlled by the **effort level**.

---

## 3. Prompt caching (high-probability exam topic)

If a request begins with exactly the same tokens as a request the server just saw, the server reuses the stored state and only prefills what comes after.

**Fixed prefix order of every request:**
`tool definitions → system prompt → conversation (with CLAUDE.md at the front of the conversation)`

- The cache must match **from the very start of the request forward**.
- If anything near the front changes, everything behind it is prefilled again at full price.
- **Ideal case:** a tool result appended to the **end** of the conversation — nothing sits behind it.
- Claude Code manages the cache automatically; there is nothing to enable, but you can break it.

### Per-turn bill
**cache reads on the history + full input price on whatever is new + output price on the response.**
A typical turn is lopsided: tens of thousands of tokens in, a few hundred out.
This applies on a subscription too — prices are hidden, but the same requests draw down your limits.

### What busts the cache

| Action | Why |
|---|---|
| `/model` | Every model has its own cache → whole conversation re-prefilled. Includes **opusplan**, which switches model every time you enter/exit plan mode. |
| `/effort` | Effort level is part of the cache key → same effect. (This is why both `/model` and `/effort` ask for confirmation mid-conversation.) |
| **Fast mode** | Part of the cache key; the re-prefill happens **at fast-mode prices**. Turn it on at the start. Turning it **off** is free, cache-wise. |
| `/compact` | The conversation is replaced by a shorter one, so nothing in it matches (the system prompt in front of it survives). |
| **Time** | Cache expires after **1 hour on a subscription**, **5 minutes on an API key** (`ENABLE_PROMPT_CACHING_1H=1` makes it an hour). Every turn resets the clock. Resuming an old session almost always re-prefills — the cache is gone and the system prompt is rebuilt at launch. |

**Cheap moments to switch model/effort:** start of a session, or right after `/clear`.
**Expensive moment:** the middle of a long conversation.

**`/rewind` vs `/compact`:** if the last few turns went the wrong way, `/rewind` to just before them — it only cuts turns off the end, everything before stays cached, so it costs nothing. `/compact` rewrites the whole conversation and always costs something. Writing the summary is much cheaper **while the old conversation is still cached** — i.e. compact *before* a long break, not after.

---

## 4. What decides HOW MANY tokens a session sends

**Key rule: nothing is sent just once.** Every file Claude reads and every command output is re-sent on every subsequent turn for the rest of the session. It is cached (cheap), but cheap ≠ free, and it occupies context the model must think around every turn.

### What is in the context before you type
Tool definitions, system prompt, `CLAUDE.md`, and anything else loaded at startup.

- Run **`/context`** in a fresh session to see what is loaded.
- Keep `CLAUDE.md` to **specific instructions**; move workflow-specific ones into **skills** (loaded only when used).
- Turn off MCP servers you don't need this session with **`/mcp`**.

### Files
Cost depends on how much Claude has to figure out on its own:

| Prompt | Cost |
|---|---|
| "the tests are failing" | greps + several files opened to find the relevant one; all results stay in context long after they are useful |
| "fix the failing test in `utils.test.ts`" | no searching, **one Read call** |
| "fix the failing test in `@utils.test.ts`" | **no Read call at all** |

**@-mention** attaches the file to your message before anything is sent, so it is in the very first request. The file occupies the same context space either way → mention it **once per conversation**; @-mentioning it again on a later turn generally attaches a **second copy**.

### Command output
Command output is appended to the conversation exactly like a file and stays for the same number of turns.

- **Very large output is fine:** past **30,000 characters** Claude Code writes the output to a file and puts only a short preview + the path in the conversation (configurable via **`BASH_MAX_OUTPUT_LENGTH`**).
- **The danger is everything under that limit** — e.g. a test runner printing 400 passing tests one per line lands under the threshold, and those 400 lines then ride along on every remaining turn.
- Mitigations: quiet flags, `tail`, a subagent, or a hook (in the docs) that rewrites noisy commands before they run.
- **Tip:** put the 2–3 commands you run daily into `CLAUDE.md` **with quiet flags**, exactly as you'd type them (e.g. `npx vitest run <file> --reporter=dot`).

### How many turns the context stays
One long session costs more than the same work split into several short ones — turn 40 also re-reads the 39 turns before it.

- **`/clear`** when starting something new (don't carry one task's context into the next).
- **`/compact`** when the earlier part of the *same* task is finished; tell it what to keep, or add a "Compact instructions" section to `CLAUDE.md`.
- **`/rename`** before `/clear` if you may want the session back later.
- **`/autocompact 200k`** restores the old auto-compact safety net on a 1M-context model (requires Claude Code **v2.1.221+**).
- **`/loop`** fires as a full turn inside the session where it was set up, carrying that whole conversation each time; more than an hour since the last turn also means a cache miss. **Run loops from a fresh session in another terminal.**

### Subagents
A subagent gets **its own context window**: own system prompt, the tools, and your `CLAUDE.md` — **but not your conversation**. It runs its own turns; only its **answer** returns to the main session, everything else is discarded.

- **Downside:** it may have to re-read things the main session already had, and it pays for its own turns. For a small job it is pure overhead.
- **Pays off** when a job produces a lot of output you don't need to keep (e.g. going through a log). You can request it explicitly: "go through this log in a subagent".
- The main session only receives what the subagent chose to report.
- **Tip:** for a noisy job you delegate repeatedly, create a subagent definition with `model: haiku` (or sonnet); otherwise it runs on whatever the main session runs on.

---

## 5. TL;DR checklist (memorize)

1. **`/clear` between tasks** — stops irrelevant prior context being re-sent.
2. **Set model and effort before you start** — changing either mid-conversation busts the cache.
3. **@-mention files instead of naming them** — saves a Read call (or a search).
4. **Add quiet flags to noisy commands, or run them in a subagent** — output stays in the conversation for the rest of the session.
5. **Run `/context` once in a fresh session** — see what's loaded (`CLAUDE.md`, MCP tool definitions) and cut what's unnecessary.
6. **`/compact` before stepping away** — the cache expires after an hour, and summarizing is much cheaper while still cached.

### Quick reference — commands, env vars, numbers

| Item | Meaning |
|---|---|
| `/clear` | new task, clean context |
| `/compact` | shorten conversation (busts cache; do it while still cached) |
| `/rewind` | cut off last turns; free, cache-preserving |
| `/context` | show what is loaded in the context |
| `/model`, `/effort` | set at the start; both remember last choice |
| `/mcp` | turn off unneeded MCP servers |
| `/rename`, `/loop`, `/autocompact 200k` | session naming, looping, auto-compact threshold (v2.1.221+) |
| `MAX_THINKING_TOKENS=0` | disables thinking for one session (except on Fable 5); one step below `/effort` low |
| `BASH_MAX_OUTPUT_LENGTH` | threshold (default 30,000 chars) after which output goes to a file |
| `ENABLE_PROMPT_CACHING_1H=1` | 1-hour cache on an API key (default 5 min) |
| **5×** | output price vs input price |
| **0.1×** | cache read vs input price |
| **up to 2×** | cache write vs input price |
| **1 h / 5 min** | cache lifetime: subscription / API key |

---
---

# Конспект для экзамена CCAR-F — «Максимальная отдача от сессий Claude Code»

**Источник:** https://claude.com/blog/maximizing-the-value-of-your-claude-code-sessions
**Автор:** Lydia Hallie · **Дата:** 14 августа 2026 · **Категория:** Claude Code

---

## 1. Главная идея

С агентными инструментами одна и та же выполненная задача может стоить по-разному — в зависимости от того, как ведётся сессия. Эффективность по токенам — это **не** «тратить меньше токенов», а «следить, чтобы потраченные токены шли именно на то, что вы просили».

Модель стоимости сессии = **сколько токенов попало в контекст × сколько ходов они там остаются × сколько контекстов работает одновременно.**

---

## 2. Что определяет ЦЕНУ токена

Вы платите за время инференса на GPU. Три фактора:

| Фактор | Эффект |
|---|---|
| **Модель** | Более крупная модель выполняет больше работы и на входных, и на выходных токенах; она умножает всё остальное. Большая модель — для действительно сложных/неоднозначных задач, маленькая — для рутины. |
| **Вход vs выход** | Выходные токены стоят примерно **в 5 раз дороже** входных. |
| **Кэш** | Чтение из кэша = **0,1× цены входа**. Запись в кэш = до **2× цены входа** (однократно на токен). |

### Две фазы запроса
- **Prefill (предзаполнение)** — модель читает запрос и контекст: системный промпт, `CLAUDE.md`, ваше сообщение и всё, что добавилось в диалог с тех пор (прочитанные файлы, вывод команд). → **входные токены**
- **Decode (генерация)** — модель пишет: размышления, вызовы инструментов, видимый текст. Генерация идёт по одному токену за раз (ответ в 200 токенов = 200 последовательных прогонов модели), поэтому decode занимает GPU намного дольше на токен. → **выходные токены**

Большая часть выходных токенов — это **токены размышления (thinking)**; их объём за ход регулируется **уровнем усилий (effort)**.

---

## 3. Кэширование промпта (важнейшая тема)

Если запрос начинается ровно с тех же токенов, что и недавно виденный сервером запрос, сервер переиспользует сохранённое состояние и предзаполняет только то, что идёт дальше.

**Фиксированный порядок префикса любого запроса:**
`определения инструментов → системный промпт → диалог (в начале диалога — CLAUDE.md)`

- Кэш должен совпадать **с самого начала запроса**.
- Если меняется что-то ближе к началу — всё, что стоит после, предзаполняется заново по полной цене.
- **Идеальный случай:** результат инструмента, добавленный в **конец** диалога — за ним ничего нет.
- Claude Code управляет кэшем сам, включать ничего не нужно, но его можно сломать.

### Счёт за один ход
**чтение истории из кэша + полная цена входа за всё новое + цена выхода за ответ.**
Типичный ход асимметричен: десятки тысяч токенов внутрь, несколько сотен наружу.
На подписке действует то же самое — цены не видны, но те же запросы расходуют ваши лимиты.

### Что сбрасывает кэш

| Действие | Почему |
|---|---|
| `/model` | У каждой модели свой кэш → весь диалог предзаполняется заново. Сюда же относится **opusplan**, который меняет модель при каждом входе/выходе из режима плана. |
| `/effort` | Уровень усилий входит в ключ кэша → тот же эффект. Поэтому `/model` и `/effort` требуют подтверждения при смене посреди диалога. |
| **Fast mode** | Тоже часть ключа кэша, причём повторное предзаполнение считается **по ценам fast mode**. Включать — в начале сессии. Выключение для кэша бесплатно. |
| `/compact` | Диалог заменяется более коротким, совпадений не остаётся (системный промпт перед ним сохраняется). |
| **Время** | Кэш живёт **1 час на подписке** и **5 минут на API-ключе** (`ENABLE_PROMPT_CACHING_1H=1` делает час). Каждый ход обнуляет таймер. Возобновление старой сессии почти всегда означает полное предзаполнение: кэш уже истёк, а системный промпт пересобирается при запуске. |

**Дешёвые моменты для смены модели/усилий:** начало сессии или сразу после `/clear`.
**Дорогой момент:** середина длинного диалога.

**`/rewind` против `/compact`:** если последние ходы ушли не туда — сделайте `/rewind` к моменту перед ними: он отрезает только последние ходы, всё до них остаётся в кэше, и это бесплатно. `/compact` переписывает весь диалог и всегда чего-то стоит. Сам пересказ дешевле, **пока старый диалог ещё в кэше** — то есть компактить нужно **до** длинного перерыва, а не после.

---

## 4. Что определяет, СКОЛЬКО токенов отправляет сессия

**Ключевое правило: ничто не отправляется один раз.** Каждый прочитанный файл и каждый вывод команды пересылается заново на каждом следующем ходу до конца сессии. Это кэшируется (дёшево), но дёшево ≠ бесплатно, и оно занимает контекст, вокруг которого модели приходится думать на каждом ходу.

### Что есть в контексте до того, как вы что-то напечатали
Определения инструментов, системный промпт, `CLAUDE.md` и всё, что загружается при старте.

- Запустите **`/context`** в свежей сессии, чтобы увидеть загруженное.
- Держите в `CLAUDE.md` только **конкретные инструкции**; специфичные для рабочих процессов выносите в **skills** (они загружаются только при использовании).
- Ненужные в этой сессии MCP-серверы отключайте через **`/mcp`**.

### Файлы
Стоимость зависит от того, сколько Claude приходится выяснять самому:

| Запрос | Стоимость |
|---|---|
| «тесты падают» | grep-и + несколько открытых файлов, чтобы найти нужный; все результаты остаются в контексте надолго после того, как перестали быть полезными |
| «почини падающий тест в `utils.test.ts`» | без поиска, **один вызов Read** |
| «почини падающий тест в `@utils.test.ts`» | **без вызова Read вообще** |

**@-упоминание** прикрепляет файл к сообщению до отправки, поэтому он попадает в самый первый запрос. Места в контексте файл занимает одинаково → упоминайте его **один раз за диалог**; повторное @-упоминание на следующем ходу обычно прикрепляет **вторую копию**.

### Вывод команд
Вывод команды добавляется в диалог точно так же, как прочитанный файл, и остаётся там столько же ходов.

- **Очень большой вывод — не проблема:** после **30 000 символов** Claude Code записывает вывод в файл, а в диалог кладёт только короткий превью и путь (настраивается через **`BASH_MAX_OUTPUT_LENGTH`**).
- **Опасно всё, что меньше этого порога** — например, тест-раннер, печатающий 400 успешных тестов по строке на каждый: он укладывается в лимит, и эти 400 строк едут с вами на каждом оставшемся ходу.
- Решения: тихие флаги, `tail`, субагент или хук (есть в документации), переписывающий шумные команды перед запуском.
- **Совет:** запишите 2–3 ежедневные команды в `CLAUDE.md` **вместе с тихими флагами**, ровно в том виде, как вы их набираете (например, `npx vitest run <file> --reporter=dot`).

### Сколько ходов контекст остаётся
Одна длинная сессия стоит дороже, чем та же работа, разбитая на несколько коротких: ход №40 заодно перечитывает 39 предыдущих.

- **`/clear`** — когда начинаете новое (не тащите контекст одной задачи в другую).
- **`/compact`** — когда ранняя часть **той же** задачи завершена; скажите, что сохранить, или добавьте раздел «Compact instructions» в `CLAUDE.md`.
- **`/rename`** перед `/clear`, если сессия может понадобиться позже.
- **`/autocompact 200k`** возвращает привычный порог автокомпакта на модели с контекстом 1M (нужен Claude Code **v2.1.221+**).
- **`/loop`** выполняется как полноценный ход внутри той сессии, где был настроен, и каждый раз тащит с собой весь её диалог; если с последнего хода прошло больше часа — вдобавок промах кэша. **Запускайте циклы из свежей сессии в другом терминале.**

### Субагенты
Субагент получает **собственное контекстное окно**: свой системный промпт, инструменты и ваш `CLAUDE.md` — **но не ваш диалог**. Он делает свои ходы, а в основную сессию возвращается только его **ответ**; всё остальное выбрасывается.

- **Минус:** ему иногда приходится перечитывать то, что у основной сессии уже было, и он платит за собственные ходы. Для мелкой задачи это чистые накладные расходы.
- **Оправдан**, когда задача порождает много вывода, который не нужно хранить (например, разбор лога). Можно попросить напрямую: «разбери этот лог в субагенте».
- Основная сессия получает только то, что субагент решил сообщить.
- **Совет:** для шумной задачи, которую вы делегируете постоянно, сделайте отдельное описание субагента с `model: haiku` (или sonnet); иначе он работает на той же модели, что и основная сессия.

---

## 5. Итоговый чек-лист (выучить)

1. **`/clear` между задачами** — прежний нерелевантный контекст перестаёт отправляться заново.
2. **Задать модель и уровень усилий до старта** — смена посреди диалога ломает кэш.
3. **@-упоминать файлы вместо названия** — экономит вызов Read (или поиск).
4. **Тихие флаги для шумных команд либо запуск в субагенте** — вывод остаётся в диалоге до конца сессии.
5. **Один раз выполнить `/context` в свежей сессии** — увидеть загруженное (`CLAUDE.md`, определения MCP-инструментов) и убрать лишнее.
6. **`/compact` перед перерывом** — кэш истекает через час, а пересказ намного дешевле, пока он ещё в кэше.

### Быстрая справка — команды, переменные, числа

| Элемент | Значение |
|---|---|
| `/clear` | новая задача, чистый контекст |
| `/compact` | сжать диалог (ломает кэш; делать, пока он ещё закэширован) |
| `/rewind` | отрезать последние ходы; бесплатно, кэш сохраняется |
| `/context` | показать, что загружено в контекст |
| `/model`, `/effort` | задавать в начале; оба помнят прошлый выбор |
| `/mcp` | отключить ненужные MCP-серверы |
| `/rename`, `/loop`, `/autocompact 200k` | имя сессии, циклы, порог автокомпакта (v2.1.221+) |
| `MAX_THINKING_TOKENS=0` | отключает размышления для одной сессии (кроме Fable 5); на ступень ниже `/effort low` |
| `BASH_MAX_OUTPUT_LENGTH` | порог (по умолчанию 30 000 символов), после которого вывод уходит в файл |
| `ENABLE_PROMPT_CACHING_1H=1` | часовой кэш на API-ключе (по умолчанию 5 минут) |
| **5×** | цена выхода относительно входа |
| **0,1×** | цена чтения из кэша относительно входа |
| **до 2×** | цена записи в кэш относительно входа |
| **1 ч / 5 мин** | время жизни кэша: подписка / API-ключ |
