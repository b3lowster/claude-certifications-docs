# CCAR-F Remediation Guide / Руководство по подготовке к пересдаче

**Candidate:** Volodymyr Tsukanov · **Attempt:** 4 Sep 2026 · **Score:** 435 / 720 · **Grade:** Fail

---

## 0. Score analysis / Разбор результата

You need +285 points. The good news: your failures are **clustered**, not spread evenly. Five clusters account for almost every 0%.

| # | Cluster | Objectives at 0–33% | Priority |
|---|---------|--------------------|----------|
| 1 | Claude Code configuration mechanisms | 4 objectives at 0% | **Critical** |
| 2 | Structured output & extraction schemas | 4 at 0%, 1 at 25% | **Critical** |
| 3 | Context supply & codebase exploration | 2 at 0%, 2 at 50% | High |
| 4 | Handoff, escalation, human routing | 2 at 0%, 1 at 33% | High |
| 5 | API mechanics (sync vs batch, stateless) | 1 at 0%, 1 at 50% | Medium |
| 6 | Hooks (PreToolUse/PostToolUse patterns) | 1 at 0% (but 100% on the PostToolUse implementation item) | Medium |

**The single unifying idea you missed:** the exam repeatedly tests *context vs. enforcement*. Anything written in natural language (CLAUDE.md, rules, prompts, tool descriptions) is **guidance** — probabilistic. Anything expressed as configuration or code (hooks, `permissions.deny`, JSON schemas, `tool_choice`) is **enforcement** — deterministic. Almost every 0% objective is a question of the form "should this be guidance or enforcement, and at what scope?"

**Вам нужно +285 баллов.** Хорошая новость: провалы сгруппированы, а не размазаны. Пять кластеров дают почти все нули. Главная идея, которую вы упустили: экзамен постоянно проверяет различие **«контекст против принуждения»**. Всё, что написано естественным языком (CLAUDE.md, rules, промпты, описания инструментов) — это *рекомендация*, вероятностная. Всё, что выражено конфигурацией или кодом (хуки, `permissions.deny`, JSON-схемы, `tool_choice`) — это *принуждение*, детерминированное. Почти каждый нулевой пункт — вопрос вида «это должно быть рекомендацией или принуждением, и на каком уровне?»

---

# Module 1 — Claude Code configuration mechanisms

*Covers objectives at 0%: "Select the correct Claude Code configuration mechanism…", "Distinguish instructions that must be enforced through settings permissions or hooks from those appropriately placed in CLAUDE.md…"*

## 1.1 English

### The five mechanisms

| Mechanism | Loaded when | Nature | Use for |
|---|---|---|---|
| **CLAUDE.md** | Every session, at startup | Context (advisory) | Always-true project facts: build commands, architecture, conventions |
| **`.claude/rules/*.md`** | Every session, *or* only when Claude touches files matching `paths:` frontmatter | Context (advisory) | Path-specific conventions — API rules, test rules, frontend rules |
| **Skills** | On demand, when Claude judges the description relevant, or when invoked | Context (advisory) | Multi-step *procedures* used occasionally |
| **Hooks** | At fixed lifecycle events (PreToolUse, PostToolUse, Stop…) | Enforcement (deterministic) | Things that must happen/must not happen regardless of model judgment |
| **`permissions` in settings.json** | On every tool call | Enforcement (deterministic) | Blocking or auto-allowing tools, commands, and paths |

### The decision procedure the exam wants

Ask three questions, in this order:

1. **Must this hold even if the model decides otherwise?**
   → Yes: `permissions.deny` (block a tool/path/command) or a **PreToolUse hook** (conditional logic). Never CLAUDE.md.
   → No: continue.
2. **Does it apply to every session and every file?**
   → Yes: **CLAUDE.md**.
   → No, only to certain paths: **`.claude/rules/` with `paths:` frontmatter**.
   → No, only for a specific occasional task: **Skill**.
3. **Must it happen at a fixed point in time (after every edit, before every commit)?**
   → **Hook**, not an instruction. "Run prettier after editing" in CLAUDE.md is a wrong answer; a PostToolUse hook is the right one.

### Why CLAUDE.md is not enforcement

CLAUDE.md content is injected as context, not as part of the system prompt, and Claude is not guaranteed to comply — especially with vague or conflicting instructions. The docs are explicit: to block an action regardless of what Claude decides, use a PreToolUse hook.

Practical consequences the exam tests:
- **Size matters.** Target under 200 lines. Longer files consume context and *reduce* adherence. Growing file → move content to path-scoped rules, not into a bigger CLAUDE.md.
- **Contradictions are resolved arbitrarily.** Two conflicting rules across CLAUDE.md files → Claude may pick either.
- **Imports don't save context.** `@path/to/file.md` imports expand at launch; they help organization, not token budget. Max import depth: 4 hops.
- **Restructuring question pattern:** given a bloated CLAUDE.md containing (a) "never commit secrets", (b) "run eslint after edits", (c) "React components use PascalCase", (d) "build with `pnpm build`" — the correct split is (a) → `permissions.deny` on the relevant paths/commands, (b) → PostToolUse hook, (c) → `.claude/rules/` scoped to `src/components/**/*.tsx`, (d) → stays in CLAUDE.md.

### Scopes and precedence

CLAUDE.md locations, broadest to most specific:

| Scope | Location | Shared with |
|---|---|---|
| Managed policy | `/Library/Application Support/ClaudeCode/CLAUDE.md` (macOS), `/etc/claude-code/CLAUDE.md` (Linux/WSL), `C:\Program Files\ClaudeCode\CLAUDE.md` (Windows) | Everyone in the org; **cannot be excluded** |
| User | `~/.claude/CLAUDE.md` | Just you, all projects |
| Project | `./CLAUDE.md` or `./.claude/CLAUDE.md` | Team, via version control |
| Local | `./CLAUDE.local.md` | Just you, this project (gitignore it) |

Files are **concatenated**, not overridden: root-down ordering, so the file closest to your working directory is read last. Subdirectory CLAUDE.md files load on demand when Claude reads files there.

### `.claude/rules/` specifics

```markdown
---
paths:
  - "src/api/**/*.ts"
  - "src/**/*.{ts,tsx}"
---

# API rules
- All endpoints must validate input
- Use the standard error envelope
```

- No `paths:` field → loads unconditionally, same priority as `.claude/CLAUDE.md`.
- With `paths:` → loads **when Claude reads a matching file**, not on every tool call.
- Rules do **not** support `@import` syntax (unlike CLAUDE.md).
- `~/.claude/rules/` = personal rules across all projects; loaded *before* project rules, so project rules win.
- Debug with `/memory` (what exists) and `/context` (what actually loaded).

### Auto memory (adjacent topic, often a distractor)

Claude also writes its own notes to `~/.claude/projects/<project>/memory/`, indexed by `MEMORY.md` (first 200 lines / 25 KB loaded each session). You write CLAUDE.md; Claude writes auto memory. Neither is enforcement.

## 1.2 Русский

### Пять механизмов

| Механизм | Когда загружается | Природа | Для чего |
|---|---|---|---|
| **CLAUDE.md** | Каждую сессию, при старте | Контекст (рекомендация) | Всегда актуальные факты о проекте: команды сборки, архитектура, соглашения |
| **`.claude/rules/*.md`** | Каждую сессию *или* только когда Claude работает с файлами, подходящими под `paths:` | Контекст | Соглашения, привязанные к путям — правила для API, тестов, фронтенда |
| **Skills (навыки)** | По требованию, когда описание релевантно задаче | Контекст | Многошаговые *процедуры*, применяемые эпизодически |
| **Hooks (хуки)** | На фиксированных событиях жизненного цикла | Принуждение (детерминированно) | То, что обязано (не) произойти независимо от решения модели |
| **`permissions` в settings.json** | На каждом вызове инструмента | Принуждение | Блокировка/автодопуск инструментов, команд, путей |

### Алгоритм выбора, который проверяет экзамен

1. **Должно ли это выполняться, даже если модель решит иначе?**
   → Да: `permissions.deny` или **PreToolUse-хук**. Никогда не CLAUDE.md.
   → Нет: дальше.
2. **Применимо ко всем сессиям и всем файлам?**
   → Да: **CLAUDE.md**. → Только к части путей: **`.claude/rules/` с `paths:`**. → Только к отдельной редкой задаче: **Skill**.
3. **Должно происходить в фиксированный момент времени (после каждой правки, перед каждым коммитом)?**
   → **Хук**, а не инструкция. «Запускай prettier после правки» в CLAUDE.md — неверный ответ; правильный — PostToolUse-хук.

### Почему CLAUDE.md — не принуждение

Содержимое CLAUDE.md подаётся как контекст (сообщение после системного промпта), а не как часть системного промпта, и строгое соблюдение не гарантируется — особенно при расплывчатых или противоречивых инструкциях. В документации сказано прямо: чтобы заблокировать действие независимо от решения Claude, используйте PreToolUse-хук.

Что из этого проверяют:
- **Размер важен.** Цель — меньше 200 строк. Более длинные файлы съедают контекст и *снижают* следование. Файл растёт → выносите в path-scoped rules, а не раздувайте CLAUDE.md.
- **Противоречия разрешаются произвольно.** Два конфликтующих правила — Claude может выбрать любое.
- **Импорты не экономят контекст.** `@path/to/file.md` разворачивается при запуске; это про организацию, а не про токены. Максимальная глубина импортов — 4 уровня.
- **Типовой вопрос на реструктуризацию:** раздутый CLAUDE.md содержит (а) «никогда не коммить секреты», (б) «запускать eslint после правок», (в) «React-компоненты в PascalCase», (г) «сборка через `pnpm build`». Верное распределение: (а) → `permissions.deny`, (б) → PostToolUse-хук, (в) → `.claude/rules/` с `paths: src/components/**/*.tsx`, (г) → остаётся в CLAUDE.md.

### Области видимости и приоритет

От самой широкой к самой узкой: **managed policy** (корпоративная, исключить нельзя) → **user** (`~/.claude/CLAUDE.md`) → **project** (`./CLAUDE.md` или `./.claude/CLAUDE.md`, коммитится) → **local** (`./CLAUDE.local.md`, в .gitignore).

Файлы **склеиваются**, а не перезаписывают друг друга: порядок от корня файловой системы вниз, поэтому файл ближе к рабочей директории читается последним. CLAUDE.md в подкаталогах подгружаются по требованию.

### Особенности `.claude/rules/`

- Без поля `paths:` → грузится всегда, с тем же приоритетом, что `.claude/CLAUDE.md`.
- С `paths:` → грузится, **когда Claude читает подходящий файл**, а не на каждом вызове инструмента.
- Правила **не поддерживают** синтаксис `@import` (в отличие от CLAUDE.md).
- `~/.claude/rules/` — личные правила для всех проектов; грузятся *раньше* проектных, поэтому проектные имеют приоритет.
- Диагностика: `/memory` (что существует) и `/context` (что реально загрузилось).

---

# Module 2 — Hooks: PreToolUse and PostToolUse

*Covers the 0% objective "Apply PreToolUse and PostToolUse hook patterns to enforce business rules and policy constraints not delegated to model discretion." Note you scored 100% on implementing a PostToolUse quality hook — the gap is in the pattern/selection logic, not the syntax.*

## 2.1 English

### Core distinction

- **PreToolUse** fires *before* the tool executes. It is the event that can **block**. Two ways to deny:
  - exit code **2**, with the reason on **stderr** (stderr is fed back to Claude as the reason);
  - JSON on stdout: `{"hookSpecificOutput": {"hookEventName": "PreToolUse", "permissionDecision": "deny", "permissionDecisionReason": "..."}}`. Values: `allow`, `deny`, `ask`. It can also rewrite the call via `updatedInput`.
- **PostToolUse** fires *after* the tool has already succeeded. It **cannot undo anything**. Exit 2 there does not roll back the tool; it only surfaces stderr to Claude so the model can react (e.g. "lint failed, fix it").

### Configuration shape

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          { "type": "command", "command": "npx prettier --write \"$CLAUDE_TOOL_INPUT_FILE_PATH\"" },
          { "type": "command", "command": "npx eslint --fix \"$CLAUDE_TOOL_INPUT_FILE_PATH\"" }
        ]
      }
    ]
  }
}
```

`matcher` matches tool names (regex). Multiple hooks in one entry run in parallel. Hooks receive JSON on stdin: `session_id`, `transcript_path`, `cwd`, `hook_event_name`, `tool_name`, `tool_input`.

### Pattern selection — how the exam frames it

| Business rule | Correct mechanism |
|---|---|
| "Never let the agent read `.env`" | PreToolUse on `Read\|Edit\|Write`, exit 2 on path match |
| "Every file edit must be formatted and linted" | PostToolUse on `Write\|Edit` |
| "Never run `terraform apply` in prod without approval" | PreToolUse returning `permissionDecision: "ask"` |
| "Block a whole tool outright, no logic needed" | `permissions.deny` in settings.json — simpler than a hook |
| "Prefer functional components" | CLAUDE.md / rules — a style preference, not a policy |
| "Don't let the agent stop while tests fail" | Stop hook, exit 2 / `{"decision":"block","reason":"..."}` |

Rule of thumb the exam rewards: **static, unconditional blocking → `permissions.deny`; conditional blocking that needs to inspect the arguments → PreToolUse hook; after-the-fact quality enforcement → PostToolUse hook; taste and convention → CLAUDE.md or rules.**

### Other events worth knowing

`SessionStart` (stdout is injected into context as plain text — useful for injecting `git log`), `UserPromptSubmit` (can block the prompt; stdout injected as context), `Stop` / `SubagentStop` (exit 2 forces Claude to continue), `PreCompact`, `SessionEnd`, `InstructionsLoaded` (debug which instruction files loaded).

## 2.2 Русский

### Ключевое различие

- **PreToolUse** срабатывает *до* выполнения инструмента. Это событие, которое умеет **блокировать**. Два способа отказать:
  - код выхода **2**, причина в **stderr** (stderr возвращается модели как объяснение отказа);
  - JSON в stdout: `{"hookSpecificOutput": {"hookEventName": "PreToolUse", "permissionDecision": "deny", "permissionDecisionReason": "..."}}`. Значения: `allow`, `deny`, `ask`. Также можно переписать аргументы через `updatedInput`.
- **PostToolUse** срабатывает *после* успешного выполнения инструмента. **Ничего откатить не может.** Код выхода 2 здесь не отменяет действие, а лишь показывает stderr модели, чтобы она отреагировала («линтер упал — исправь»).

### Выбор паттерна — как это формулирует экзамен

| Бизнес-правило | Верный механизм |
|---|---|
| «Агент никогда не читает `.env`» | PreToolUse на `Read\|Edit\|Write`, exit 2 при совпадении пути |
| «Каждая правка файла форматируется и линтуется» | PostToolUse на `Write\|Edit` |
| «Не запускать `terraform apply` в prod без подтверждения» | PreToolUse с `permissionDecision: "ask"` |
| «Полностью заблокировать инструмент, без логики» | `permissions.deny` в settings.json — проще, чем хук |
| «Предпочитать функциональные компоненты» | CLAUDE.md / rules — это стиль, а не политика |
| «Не давать агенту завершиться, пока падают тесты» | Stop-хук, exit 2 / `{"decision":"block","reason":"..."}` |

Правило: **статичная безусловная блокировка → `permissions.deny`; условная блокировка с анализом аргументов → PreToolUse; контроль качества постфактум → PostToolUse; вкусовые предпочтения → CLAUDE.md/rules.**

---

# Module 3 — MCP servers: scope, auth, discovery

*Covers 0% "Configure MCP servers and project settings at the correct scope…" and 50% "Integrate MCP servers into Claude Code…"*

## 3.1 English

### The three scopes

| Scope | Stored in | Visible to |
|---|---|---|
| `local` (**default**) | `~/.claude.json`, keyed by project path | Only you, only in this project |
| `user` | `~/.claude.json` (global block) | Only you, every project |
| `project` | `.mcp.json` at repo root, **committed to git** | Everyone who clones the repo |

```bash
claude mcp add my-server --scope project -- npx -y @scope/server
claude mcp add --transport http github --scope project https://api.githubcopilot.com/mcp/
```

**The exam's mapping:** shared team tooling → `project`. Personal or experimental → `local` (or `user` if you want it everywhere). The classic trap: someone runs `claude mcp add` with no `--scope`, expects a committable `.mcp.json`, and gets nothing in the repo — because the default is `local`, which writes to `~/.claude.json`.

Project-scoped servers in `.mcp.json` require an **approval prompt on first use**, because approving means agreeing to run those commands on your machine.

### Credentials via environment variable expansion

Never hardcode secrets in a committed `.mcp.json`. Use expansion — it works in `command`, `args`, `env`, `url`, and `headers`:

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/",
      "headers": { "Authorization": "Bearer ${GITHUB_PAT}" }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": { "DATABASE_URL": "${DATABASE_URL:-postgresql://localhost/dev}" }
    }
  }
}
```

`${VAR}` = required; `${VAR:-default}` = with fallback. An unset `${VAR}` with no default causes a parse failure and the server silently doesn't appear.

### Transports

`stdio` (default; local subprocess), `http` (Streamable HTTP; `streamable-http` accepted as an alias), `sse` (deprecated — don't pick it for new work).

### Verifying tool discovery

Run `/mcp` inside a session to see connection status and tool counts. Connected but 0 tools = it connected but advertises nothing; check init requirements. Not listed at all = config parse error or pending approval. `claude --mcp-debug` shows connection attempts, tool discovery, and transport details.

### Error handling from MCP tools (your 75% objective — nearly there)

A tool that returns `"Error: request failed"` gives the agent nothing to act on. Return structured, type-specific errors: an error `type`/`code`, whether it is **recoverable**, and a **suggested next action**. `RATE_LIMITED, recoverable: true, retry_after: 30s` lets the agent wait and retry; `AUTH_EXPIRED, recoverable: false, action: escalate to human` lets it stop instead of looping.

## 3.2 Русский

### Три области видимости

| Scope | Где хранится | Кому видно |
|---|---|---|
| `local` (**по умолчанию**) | `~/.claude.json`, привязка к пути проекта | Только вам, только в этом проекте |
| `user` | `~/.claude.json` (глобальный блок) | Только вам, во всех проектах |
| `project` | `.mcp.json` в корне репозитория, **коммитится** | Всем, кто клонирует репозиторий |

**Соответствие, которое проверяет экзамен:** общий командный инструментарий → `project`. Личное или экспериментальное → `local` (или `user`, если нужно везде). Классическая ловушка: человек выполняет `claude mcp add` без `--scope`, ждёт `.mcp.json` в репозитории, но ничего там не появляется — потому что по умолчанию scope `local`, а он пишется в `~/.claude.json`.

Серверы уровня `project` из `.mcp.json` требуют **подтверждения при первом использовании**: согласие означает запуск чужих команд на вашей машине.

### Учётные данные через подстановку переменных окружения

Никогда не хардкодьте секреты в коммитируемый `.mcp.json`. Подстановка работает в `command`, `args`, `env`, `url` и `headers`: `${VAR}` — обязательная, `${VAR:-default}` — со значением по умолчанию. Незаданная `${VAR}` без default приводит к ошибке разбора, и сервер просто не появляется.

### Транспорты

`stdio` (по умолчанию, локальный подпроцесс), `http` (Streamable HTTP, псевдоним `streamable-http`), `sse` (устарел — не выбирайте для нового).

### Проверка обнаружения инструментов

Команда `/mcp` в сессии показывает статус подключения и число инструментов. Подключён, но 0 инструментов — сервер не объявляет инструментов. Вообще не в списке — ошибка разбора конфига или ожидание подтверждения. `claude --mcp-debug` показывает попытки подключения, обнаружение инструментов и детали транспорта.

### Обработка ошибок MCP-инструментов

Ошибка вида `"Error: request failed"` не даёт агенту опоры для действия. Возвращайте структурированные типизированные ошибки: `type`/`code`, признак **восстановимости** и **рекомендуемое следующее действие**. `RATE_LIMITED, recoverable: true, retry_after: 30s` — агент подождёт и повторит; `AUTH_EXPIRED, recoverable: false, action: escalate` — агент остановится, а не зациклится.

---

# Module 4 — Supplying project context: `@` vs CLAUDE.md vs inline

*Covers the 0% objective "Select the appropriate method for providing project context to Claude Code…"*

## 4.1 English

Three axes decide the answer: **reusability**, **specificity**, and **cross-session need**.

| Method | Lifetime | Choose when |
|---|---|---|
| **Inline description in the prompt** | This turn only | One-off, situational detail: "this endpoint is behind a feature flag right now" |
| **`@path/to/file`** in a prompt | This turn (the file is pulled into context) | You need Claude to see *specific file content* right now, and the file already exists as the source of truth |
| **`@path` import inside CLAUDE.md** | Every session | The file is authoritative and always relevant (e.g. `@AGENTS.md`, a coding-standards doc) |
| **CLAUDE.md prose** | Every session | Facts that don't live anywhere else in the repo and would otherwise be re-typed each session |

Decision heuristic: *Will I need to say this again next session?* No → inline. Yes, and it's already written down in a file → `@` reference (in the prompt if occasional, in CLAUDE.md if always). Yes, and it isn't written down anywhere → add it to CLAUDE.md.

Two traps:
- **`@` in CLAUDE.md does not save context.** Imports expand at launch. Splitting a 400-line CLAUDE.md into imports leaves you with the same token cost; use path-scoped rules if you want conditional loading.
- **Duplicating a file into CLAUDE.md instead of importing it** creates two sources of truth that drift. Import or reference the real file.

## 4.2 Русский

Ответ определяют три оси: **переиспользуемость**, **конкретность**, **нужность между сессиями**.

| Метод | Время жизни | Когда выбирать |
|---|---|---|
| **Описание прямо в промпте** | Только текущий ход | Разовая ситуативная деталь: «этот эндпоинт сейчас за фиче-флагом» |
| **`@путь/к/файлу`** в промпте | Текущий ход (файл подтягивается в контекст) | Нужно, чтобы Claude увидел *конкретное содержимое файла* сейчас, и файл уже является источником истины |
| **Импорт `@путь` внутри CLAUDE.md** | Каждая сессия | Файл авторитетен и всегда актуален (например, `@AGENTS.md`, документ по стандартам кода) |
| **Текст в CLAUDE.md** | Каждая сессия | Факты, которых нет больше нигде в репозитории и которые иначе придётся печатать каждую сессию |

Эвристика: *Придётся ли говорить это снова в следующей сессии?* Нет → inline. Да, и это уже записано в файле → `@`-ссылка (в промпте, если эпизодически; в CLAUDE.md, если всегда). Да, и нигде не записано → в CLAUDE.md.

Две ловушки:
- **`@` в CLAUDE.md не экономит контекст.** Импорты разворачиваются при запуске. Разбиение файла на 400 строк по импортам даёт ту же стоимость в токенах; для условной загрузки нужны path-scoped rules.
- **Копирование файла в CLAUDE.md вместо импорта** создаёт два источника истины, которые разъезжаются.

---

# Module 5 — Codebase exploration and context management

*Covers 0% "systematic codebase exploration using Grep, Glob, and Read" and 50% "context management strategies — subagent isolation, scratchpad files, targeted file reading"*

## 5.1 English

### The exploration ladder

The correct pattern is **broad and cheap first, narrow and expensive last**:

1. **Glob** — find files by name pattern (`src/**/*.ts`, `**/*auth*`). Cheapest: returns paths only.
2. **Grep** — find files by content (regex). Narrow it with `path`, `glob`, or `type` filters. Use `output_mode: files_with_matches` to locate, then `content` with `-A/-B` context lines on the shortlist.
3. **Read** — open only the files the previous two steps identified, and use `offset`/`limit` for large files rather than reading them whole.

Anti-patterns the exam tests: reading whole directories "to understand the project", opening a 5,000-line file in full when you need one function, and running Grep with no path/glob filter across a monorepo. Each burns context you'll need later.

**Incremental understanding** means each step is informed by the last: locate the entry point → grep for the symbol → read the definition → grep for its callers. You build a map, not a dump.

### Context management across long work

- **Subagent isolation** — delegate a bounded investigation ("find every place we validate JWTs") to a subagent. The subagent burns its own context window and returns a *summary*. The main conversation stays clean. This is the primary answer for "exploration exceeds the context limit".
- **Scratchpad files** — write findings to a markdown file on disk as you go (`notes/auth-audit.md`). Persist across compaction and across sessions; reread selectively. Disk is unlimited, context is not.
- **Targeted file reading** — see the ladder above.
- **`/compact`** — summarizes the conversation. Note what survives: the project-root CLAUDE.md is re-read from disk and re-injected. Nested CLAUDE.md files and path-scoped rules reload when Claude next touches matching files. **Instructions you only said in chat do not survive** — if it matters, write it to CLAUDE.md or a scratchpad.

### Session resumption (you scored 83% — close the last gap)

On resume, don't replay prior work. The pattern is: restore the summary of what was concluded, then **targeted re-analysis of files that changed since** (git diff against the point where you stopped), plus **context injection** of the state that mattered (which task is in progress, which decisions were made, which files are already done). Re-reading everything is the wrong answer; assuming nothing changed is also the wrong answer.

## 5.2 Русский

### Лестница исследования кодовой базы

Правильный порядок — **сначала широко и дёшево, потом узко и дорого**:

1. **Glob** — поиск файлов по шаблону имени (`src/**/*.ts`). Самый дешёвый: возвращает только пути.
2. **Grep** — поиск по содержимому (regex). Сужайте фильтрами `path`, `glob`, `type`. Сначала `output_mode: files_with_matches` для локализации, затем `content` с контекстными строками `-A/-B` по короткому списку.
3. **Read** — открывать только те файлы, которые нашли на предыдущих шагах, и использовать `offset`/`limit` для больших файлов вместо чтения целиком.

Антипаттерны, которые проверяет экзамен: чтение целых каталогов «чтобы понять проект»; открытие файла на 5000 строк целиком ради одной функции; Grep без фильтров по монорепозиторию. Всё это сжигает контекст, который понадобится позже.

**Инкрементальное понимание**: каждый шаг опирается на предыдущий — найти точку входа → grep по символу → прочитать определение → grep по вызывающим. Строится карта, а не свалка.

### Управление контекстом в длинной работе

- **Изоляция в субагенте** — делегировать ограниченное исследование субагенту. Он тратит собственное окно контекста и возвращает *резюме*. Основной диалог остаётся чистым. Это главный ответ на вопрос «исследование превышает лимит контекста».
- **Файлы-черновики (scratchpad)** — записывать находки в markdown-файл на диске по ходу работы. Переживают компактизацию и смену сессий; перечитываются выборочно. Диск не ограничен, контекст ограничен.
- **Точечное чтение файлов** — см. лестницу выше.
- **`/compact`** — сжимает диалог. Что переживает: корневой CLAUDE.md перечитывается с диска и вставляется заново; вложенные CLAUDE.md и path-scoped rules перезагружаются при следующем обращении к подходящим файлам. **Инструкции, произнесённые только в чате, не переживают** — если это важно, запишите в CLAUDE.md или в черновик.

### Возобновление сессии (у вас 83% — добираем остаток)

При возобновлении не повторяйте выполненную работу. Паттерн: восстановить резюме выводов, затем **точечно переанализировать изменившиеся файлы** (git diff от точки останова) и **инжектировать контекст** значимого состояния (какая задача в работе, какие решения приняты, какие файлы уже готовы). Перечитывать всё — неверно; считать, что ничего не изменилось — тоже неверно.

---

# Module 6 — Structured output, schemas, and `tool_choice`

*Your weakest cluster: four objectives at 0% and one at 25%. Expect ~15–20% of the exam here.*

## 6.1 English

### Reliability ranking of structured-output methods

From least to most reliable:

1. **Prompt-based formatting** ("respond only with JSON matching this shape"). No enforcement. The model may add prose, code fences, or drift from the schema. Acceptable only when the shape is trivial and you can tolerate retries.
2. **Assistant prefill** — start the assistant turn with `{`. Forces the response to begin as JSON, which removes preamble, but enforces *nothing* about the schema itself. Cheaper and slightly faster on simple shapes. Not compatible with extended thinking.
3. **Tool use with a JSON `input_schema`** — you define a tool whose only job is to receive the extracted data, and the model must fill its parameters to the schema. This is the exam's answer for "most reliable" and "strict schema compliance".
4. **Structured outputs** (`output_config.format` with `type: "json_schema"`, and `strict: true` on tools) — constrained sampling that guarantees the output conforms to the schema. Available on recent models. If a question offers this and the scenario says the schema must be guaranteed, it is the strongest option; but foundations questions are usually framed around tool use + `tool_choice`.

**Guarantee vs. accuracy:** the API guarantees the *shape*, never the *correctness* of values. A question saying "this eliminates hallucination" is wrong.

### `tool_choice`

| Value | Meaning |
|---|---|
| `{"type": "auto"}` | Default. Model decides whether to call a tool at all. |
| `{"type": "any"}` | Model must call *some* tool, but picks which. |
| `{"type": "tool", "name": "extract_invoice"}` | Model must call *that specific* tool. |
| `{"type": "none"}` | No tools may be called. |

Also: `disable_parallel_tool_use` to force one tool call at a time.

**The exam's scenario:** a downstream system breaks if it receives conversational text instead of a record. With `auto`, the model may answer in prose. The fix is `tool_choice: {"type": "tool", "name": "..."}` — that is the *only* way to guarantee invocation. Adding "you must call the tool" to the prompt is a distractor.

### Sequencing multi-tool workflows

When tool B needs data that only tool A can produce (e.g. `get_customer_id` → `fetch_orders`), do not rely on the model's ordering instincts alone. Options, roughly in order of strength:

- Make the dependency structural: B's schema requires `customer_id`, which the model can only get from A's result.
- Force A first with `tool_choice: {"type":"tool","name":"get_customer_id"}` on the first turn, then relax to `auto`.
- Describe the prerequisite explicitly in B's description: "Requires a customer_id obtained from get_customer_id."
- Disable parallel tool use so calls are serialized.

### Schema design for extraction

This is the heart of the 0%/25% objectives. The goal: let the model **accurately represent missing or ambiguous information instead of fabricating it.**

```json
{
  "type": "object",
  "properties": {
    "invoice_number": { "type": "string", "description": "As printed, without the 'INV-' prefix" },
    "issue_date":     { "type": ["string", "null"], "description": "ISO 8601 (YYYY-MM-DD). Null if not present on the document." },
    "total_amount":   { "type": ["number", "null"] },
    "currency":       { "type": "string", "enum": ["USD", "EUR", "GBP", "UAH", "unknown"] },
    "payment_terms":  { "type": "string", "enum": ["net_30", "net_60", "due_on_receipt", "other", "not_stated"] },
    "confidence":     { "type": "number", "description": "0–1, the model's own confidence in this extraction" }
  },
  "required": ["invoice_number", "currency", "confidence"]
}
```

Principles being tested:

- **Optional ≠ nullable.** A field omitted from `required` may simply be absent from the output. A `["string","null"]` field is always present but can be explicitly null. Use nullable when downstream code needs the key to exist and needs to distinguish "checked, not found" from "not extracted".
- **Every enum needs an escape hatch.** Without `unknown` / `other` / `not_stated`, an incomplete enum forces the model to pick a wrong member or invent one. This is the number-one cause of extraction hallucination in exam scenarios.
- **Descriptions are part of the schema.** Per-field `description` carries format normalization ("ISO 8601", "digits only, no separators", "lowercase") and does more work than a paragraph in the system prompt.
- **Mark required only what you truly require.** Without `required`, fields are optional and may be silently dropped.
- **Ask for confidence and provenance.** A `confidence` field and a `source_snippet` field turn extraction into something you can route and audit.

### Extraction accuracy patterns (the 25% objective)

Four levers, in order of impact:

1. Structured schema with optional/nullable fields and escape-hatch enums (above).
2. **Format normalization instructions** inside field descriptions — dates, phone numbers, currency, casing. Varied document formats are exactly where unnormalized output breaks.
3. **Few-shot examples** covering the *hard* cases: a document missing a field, an ambiguous field, an unusual layout. Examples of the easy path teach nothing.
4. **Explicit instruction to leave fields null rather than guess.**

### Feedback loops (you scored 100% — reinforce it)

Capture structured metadata about each error (field, error type, document type, confidence at time of error), then aggregate: if `issue_date` fails on scanned faxes 40% of the time, that is a schema/description fix or a targeted few-shot example, not a general "be more careful" prompt edit.

## 6.2 Русский

### Ранжирование методов структурированного вывода по надёжности

От наименее к наиболее надёжному:

1. **Форматирование через промпт** («отвечай только JSON такой формы»). Никакого принуждения. Модель может добавить текст, обрамление ```` ``` ````, отойти от схемы. Допустимо только для тривиальных форм с ретраями.
2. **Prefill ответа ассистента** — начать ход ассистента с `{`. Заставляет ответ начаться как JSON и убирает преамбулу, но **ничего** не гарантирует по самой схеме. Дешевле и чуть быстрее на простых формах. Несовместимо с extended thinking.
3. **Tool use с JSON `input_schema`** — вы описываете инструмент, единственная задача которого — принять извлечённые данные; модель обязана заполнить параметры по схеме. Это ответ экзамена на «самый надёжный» и «строгое соответствие схеме».
4. **Structured outputs** (`output_config.format` с `type: "json_schema"`, и `strict: true` у инструментов) — ограниченная генерация (constrained sampling), гарантирующая соответствие схеме. Доступно на новых моделях. Если сценарий требует гарантии схемы — это самый сильный вариант; но вопросы уровня Foundations обычно формулируются вокруг tool use + `tool_choice`.

**Гарантия формы ≠ гарантия правильности:** API гарантирует *структуру*, но не *корректность значений*. Утверждение «это устраняет галлюцинации» — неверно.

### `tool_choice`

| Значение | Смысл |
|---|---|
| `{"type": "auto"}` | По умолчанию. Модель сама решает, вызывать ли инструмент. |
| `{"type": "any"}` | Модель обязана вызвать *какой-нибудь* инструмент, но выбирает сама. |
| `{"type": "tool", "name": "extract_invoice"}` | Модель обязана вызвать *конкретный* инструмент. |
| `{"type": "none"}` | Инструменты вызывать нельзя. |

Плюс `disable_parallel_tool_use` — запрет параллельных вызовов.

**Сценарий экзамена:** нижестоящая система ломается, если получает текст вместо записи. При `auto` модель может ответить прозой. Решение — `tool_choice: {"type": "tool", "name": "..."}`; это *единственный* способ гарантировать вызов. Фраза «ты обязан вызвать инструмент» в промпте — дистрактор.

### Последовательность многоинструментных сценариев

Когда инструменту B нужны данные, которые может дать только A (`get_customer_id` → `fetch_orders`), не полагайтесь на интуицию модели. Варианты по убыванию силы:

- Сделать зависимость структурной: схема B требует `customer_id`, который берётся только из результата A.
- Принудить A первым через `tool_choice: {"type":"tool","name":"get_customer_id"}` на первом ходе, затем вернуть `auto`.
- Явно указать предусловие в описании B: «Требует customer_id, полученный из get_customer_id».
- Отключить параллельные вызовы, чтобы сериализовать порядок.

### Проектирование схем извлечения

Суть заваленных пунктов. Цель: дать модели возможность **точно отразить отсутствующую или неоднозначную информацию, а не выдумать её.**

Проверяемые принципы:

- **Optional ≠ nullable.** Поле, не входящее в `required`, может просто отсутствовать в выводе. Поле типа `["string","null"]` присутствует всегда, но может быть явно `null`. Nullable нужен, когда потребителю важно, чтобы ключ существовал, и надо отличать «проверили, не нашли» от «не извлекали».
- **Каждому enum нужен запасной выход.** Без `unknown` / `other` / `not_stated` неполный перечень вынуждает модель выбрать неверный элемент или выдумать новый. Это причина №1 галлюцинаций при извлечении в экзаменационных сценариях.
- **Описания — часть схемы.** Поле `description` несёт нормализацию формата («ISO 8601», «только цифры, без разделителей», «в нижнем регистре») и работает лучше абзаца в системном промпте.
- **`required` — только то, что действительно обязательно.** Без `required` поля считаются необязательными и могут молча пропасть.
- **Запрашивайте confidence и источник.** Поля `confidence` и `source_snippet` превращают извлечение в то, что можно маршрутизировать и аудировать.

### Паттерны точности извлечения

Четыре рычага по убыванию влияния:

1. Структурированная схема с optional/nullable и enum с запасным выходом.
2. **Инструкции по нормализации формата** внутри описаний полей — даты, телефоны, валюта, регистр. Именно на разнородных форматах документов ненормализованный вывод и ломается.
3. **Few-shot примеры на *сложных* случаях**: документ без поля, неоднозначное поле, необычная вёрстка. Примеры лёгкого пути ничему не учат.
4. **Явная инструкция оставлять null, а не догадываться.**

---

# Module 7 — Handoff, escalation, and human review routing

*Covers 0% "structured handoff packages", 0% "escalation decision criteria", 33% "human review routing"*

## 7.1 English

### Structured handoff packages

A handoff happens when control moves between agent steps, between subagent and orchestrator, or from agent to human. The failure mode is **context evaporation**: the receiver has to redo the investigation because it only got a one-line conclusion.

A complete handoff package carries four things:

1. **Accumulated context** — what was being done, why, and what constraints apply. Not the raw transcript; a structured state.
2. **Findings and evidence** — what was determined, with pointers (file paths, record IDs, tool outputs, timestamps) so the receiver can verify rather than trust.
3. **Authorization state** — what has already been approved, by whom, at what scope, and what is *still* pending approval. This is the part most often forgotten and most often tested. If step 1 obtained human approval to modify production config, step 2 must know that approval exists and what it covered. Losing it causes either redundant re-approval requests or, worse, silent privilege escalation where step 2 assumes it may act.
4. **Remaining work and open questions** — what is left, what is blocked, and what the receiver needs to decide.

For a human recipient, add: the specific decision being asked for, the options, and the recommendation with its rationale. A handoff to a human that says "please review" is incomplete.

### Escalation decision criteria

The core rule the exam tests: **an explicit human request to escalate is honored immediately.** The agent does not first try to resolve it autonomously, does not argue that it could handle it, does not "attempt one more tool call". Honoring the request *is* the correct action, and delaying it is a failure even if autonomous resolution would have worked.

Attempt autonomous resolution instead when *all* of these hold: no explicit escalation request; the task is within the agent's tool capabilities; the action is reversible or low-impact; there's no policy requiring human sign-off; and the agent is not looping (repeated failures on the same step are themselves an escalation trigger).

Escalate proactively — even without a request — when the agent hits an irreversible or high-impact action without prior authorization, when it has exhausted its tools without resolution, when the request falls outside its scope, or when confidence is below the threshold for the decision's stakes.

### Human review routing

Random sampling is the wrong answer in every version of this question. It spends reviewer time uniformly on a non-uniform problem. Route by signal:

- **Confidence score thresholds**, set per field rather than per document — a low-confidence `total_amount` matters more than a low-confidence `notes` field.
- **Field-level ambiguity** — the document supports two readings, or a value conflicts with another field.
- **Document characteristics** — unfamiliar template, poor scan quality, unusual length, a vendor seen for the first time.
- **Business impact** — value above a monetary threshold, regulated fields, first transaction with a counterparty.
- **Reviewer specialization** — route to the reviewer who handles that document class or language, not the next free person.

Then close the loop: reviewer corrections feed the feedback-loop metadata from Module 6, so thresholds and schemas improve rather than staying fixed.

## 7.2 Русский

### Структурированные пакеты передачи (handoff)

Передача происходит между шагами агента, между субагентом и оркестратором, либо от агента к человеку. Типичный сбой — **испарение контекста**: получатель вынужден заново проводить расследование, потому что получил лишь однострочный вывод.

Полный пакет передачи несёт четыре вещи:

1. **Накопленный контекст** — что делалось, зачем и какие ограничения действуют. Не сырой транскрипт, а структурированное состояние.
2. **Находки и доказательства** — что установлено, со ссылками (пути к файлам, ID записей, вывод инструментов, метки времени), чтобы получатель мог проверить, а не поверить.
3. **Состояние авторизации** — что уже одобрено, кем, в каком объёме и что ещё ожидает одобрения. Это самая забываемая и самая проверяемая часть. Если на шаге 1 получено согласие человека на изменение production-конфигурации, шаг 2 обязан знать об этом согласии и его границах. Потеря приводит либо к повторным запросам одобрения, либо, что хуже, к тихой эскалации привилегий, когда шаг 2 считает, что ему уже можно.
4. **Оставшаяся работа и открытые вопросы** — что осталось, что заблокировано, что должен решить получатель.

Для получателя-человека добавьте: какое именно решение требуется, какие есть варианты и какова рекомендация с обоснованием. Передача человеку с текстом «посмотрите, пожалуйста» — неполная.

### Критерии эскалации

Главное правило: **явный запрос человека на эскалацию исполняется немедленно.** Агент не пытается сначала решить сам, не доказывает, что справится, не делает «ещё один вызов инструмента». Исполнение запроса *и есть* правильное действие; задержка — ошибка, даже если автономное решение сработало бы.

Пытаться решить автономно можно, когда выполнено *всё*: явного запроса на эскалацию нет; задача в пределах возможностей инструментов; действие обратимо или малозначимо; нет политики, требующей подписи человека; агент не зациклился (повторяющиеся неудачи на одном шаге сами по себе — триггер эскалации).

Эскалировать проактивно, даже без запроса, следует при необратимом или высокорисковом действии без предварительной авторизации; при исчерпании инструментов без результата; при выходе задачи за рамки полномочий; при уверенности ниже порога, приемлемого для ставок решения.

### Маршрутизация на ревью человеком

Случайная выборка — неверный ответ во всех формулировках этого вопроса: она равномерно расходует время ревьюера на неравномерную задачу. Маршрутизируйте по сигналам:

- **Пороги confidence**, заданные по полям, а не по документу целиком.
- **Неоднозначность на уровне поля** — документ допускает два прочтения, или значение противоречит другому полю.
- **Характеристики документа** — незнакомый шаблон, плохое качество скана, необычный объём, впервые встреченный поставщик.
- **Влияние на бизнес** — сумма выше порога, регулируемые поля, первая сделка с контрагентом.
- **Специализация ревьюера** — тому, кто ведёт этот класс документов или язык, а не первому свободному.

И замкните цикл: правки ревьюеров попадают в метаданные обратной связи из модуля 6, чтобы пороги и схемы улучшались.

---

# Module 8 — API mechanics: sync vs. batch, and statelessness

*Covers 0% "Select the appropriate API processing mode" and 50% "Explain why conversation history must be explicitly included in each API request"*

## 8.1 English

### Messages API vs. Message Batches API

| | Messages API (sync) | Message Batches API (async) |
|---|---|---|
| Latency | Immediate; you await the response | Up to **24 hours** (most finish in under an hour) |
| Blocking | Caller waits | Submit, poll, collect later |
| Cost | Standard | **50% discount** on input and output tokens |
| Volume | One request per call | Up to 100,000 requests / 256 MB per batch |
| Results | In the response | Retrieved from a results URL, retained ~29 days |
| Not supported | — | `stream`, and a few other parameters |

**Decision criteria the exam uses:** does a human or a blocking workflow wait on the result?

- Yes → Messages API. Chatbots, interactive agents, anything in a request/response path, anything where a user sees a spinner.
- No → Message Batches API. Nightly document processing, bulk classification, offline evaluation runs, backfills, A/B prompt testing.

Key traps: the 24 hours is a **ceiling, not an estimate** — design against the ceiling, not against the typical hour. And you cannot ask for a batch to go faster. If a scenario says "results needed within a few minutes", batch is wrong regardless of how attractive the discount is. Conversely, "10,000 documents, needed by tomorrow morning, cost-sensitive" is a batch scenario, and choosing sync there is the wrong answer.

### Statelessness

The Messages API holds **no server-side conversation state.** Each request is independent. If you want the model to know what was said before, you resend it: the full `messages` array, in order, on every call.

Consequences:

- **Tool loops must be reassembled.** The assistant turn containing a `tool_use` block must be followed by a user turn containing a `tool_result` block whose `tool_use_id` matches. Drop the assistant turn and the request is invalid; drop the result and the model has no idea what its tool returned.
- **Growth is your problem.** History grows every turn; you manage truncation, summarization, or a rolling window yourself.
- **Prompt caching** is the mechanism that makes resending long histories affordable — it caches the stable prefix rather than making the API stateful.
- Session state in Claude Code (`--continue`, `--resume`) is a *client-side* transcript feature, not server-side memory. The API underneath is still stateless.

## 8.2 Русский

### Messages API против Message Batches API

| | Messages API (синхронно) | Message Batches API (асинхронно) |
|---|---|---|
| Задержка | Немедленно, вы ждёте ответ | До **24 часов** (большинство — менее часа) |
| Блокировка | Вызывающий ждёт | Отправил, опрашиваешь статус, забираешь позже |
| Стоимость | Стандартная | **Скидка 50%** на входные и выходные токены |
| Объём | Один запрос на вызов | До 100 000 запросов / 256 МБ на пакет |
| Результаты | В ответе | По results URL, хранятся ~29 дней |
| Не поддерживается | — | `stream` и ещё несколько параметров |

**Критерий выбора:** ждёт ли результата человек или блокирующий процесс?

- Да → Messages API. Чат-боты, интерактивные агенты, всё в цепочке запрос/ответ.
- Нет → Message Batches API. Ночная обработка документов, массовая классификация, офлайн-прогоны оценки, бэкфилы, A/B-тестирование промптов.

Ловушки: 24 часа — это **потолок, а не оценка**; проектируйте под потолок. Ускорить пакет нельзя. Если в сценарии «результат нужен через несколько минут» — batch неверен, каким бы привлекательным ни был дисконт. И наоборот, «10 000 документов, нужно к завтрашнему утру, важна стоимость» — это batch, и выбор синхронного API там ошибка.

### Отсутствие состояния (statelessness)

Messages API **не хранит состояние диалога на сервере.** Каждый запрос независим. Чтобы модель знала сказанное ранее, вы пересылаете это: полный массив `messages`, по порядку, при каждом вызове.

Следствия:

- **Цикл инструментов нужно собирать заново.** Ход ассистента с блоком `tool_use` должен сопровождаться ходом пользователя с блоком `tool_result`, у которого `tool_use_id` совпадает. Уберёте ход ассистента — запрос невалиден; уберёте результат — модель не знает, что вернул инструмент.
- **Рост истории — ваша забота.** Усечение, суммаризация, скользящее окно реализуете вы.
- **Prompt caching** — механизм, который делает пересылку длинной истории приемлемой по цене: кешируется стабильный префикс, а API не становится stateful.
- Состояние сессии в Claude Code (`--continue`, `--resume`) — это *клиентская* функция транскрипта, а не серверная память. Нижележащий API по-прежнему stateless.

---

# Module 9 — Partial gaps at 50%

## 9.1 English

**Dynamic task decomposition (50%).** The distinction: a *static* plan enumerates subtasks up front and executes them regardless of what it learns. A *dynamic* decomposition regenerates the remaining subtasks as findings arrive — discovering that a service has no tests changes what the next steps should be. Exam signal words: "adapt as new information is discovered", "rather than executing a fixed sequence". A fixed checklist is almost always the wrong answer; so is "no plan at all".

**Iterative refinement workflows (50%).** Three ingredients: (1) concrete **input–output examples** rather than abstract descriptions of what you want; (2) **targeted feedback on specific failures** — "the date parser returns null for `03/04/2026`" beats "dates are unreliable"; (3) **batched issue descriptions** so a set of problems is evaluated and fixed together in one consolidated pass, rather than one-at-a-time round trips that cause regressions and burn context.

**Tool description quality (50%).** When the model picks the wrong tool among semantically similar ones, the fix is in the descriptions, not the prompt. Expand each with: use-case examples ("use for single-customer lookups; use `search_customers` for multi-result queries"), input format specifications ("`date` must be ISO 8601"), and explicit disambiguation naming the sibling tool and the boundary between them.

## 9.2 Русский

**Динамическая декомпозиция задач (50%).** Различие: *статический* план перечисляет подзадачи заранее и выполняет их независимо от того, что выясняется. *Динамическая* декомпозиция перегенерирует оставшиеся подзадачи по мере поступления находок. Сигнальные слова экзамена: «адаптируется по мере обнаружения новой информации», «вместо выполнения фиксированной последовательности». Фиксированный чек-лист почти всегда неверный ответ; «вообще без плана» — тоже.

**Итеративное улучшение (50%).** Три составляющие: (1) конкретные **примеры вход-выход**, а не абстрактные описания желаемого; (2) **точечная обратная связь по конкретным сбоям** — «парсер дат возвращает null на `03/04/2026`» лучше, чем «с датами ненадёжно»; (3) **пакетные описания проблем**, чтобы набор проблем оценивался и правился за один консолидированный проход, а не по одной в цикле, что вызывает регрессии и жжёт контекст.

**Качество описаний инструментов (50%).** Когда модель выбирает не тот инструмент среди семантически похожих, чинить надо описания, а не промпт. Расширьте каждое: примеры применения («для поиска одного клиента; для запросов с несколькими результатами — `search_customers`»), спецификации формата входа («`date` в ISO 8601») и явное разграничение с указанием соседнего инструмента и границы между ними.

---

# 10. One-page trap sheet / Шпаргалка ловушек

| Scenario phrase | Wrong answer | Right answer |
|---|---|---|
| "must happen after every file edit" | CLAUDE.md instruction | PostToolUse hook |
| "must not happen regardless of what Claude decides" | CLAUDE.md, "IMPORTANT" caps | `permissions.deny` or PreToolUse hook |
| "only applies to files under `src/api/`" | CLAUDE.md | `.claude/rules/` with `paths:` |
| "occasional multi-step procedure" | CLAUDE.md | Skill |
| "share the MCP server with the team" | default `claude mcp add` | `--scope project` → `.mcp.json` |
| "secrets in a committed config" | hardcode | `${VAR}` / `${VAR:-default}` expansion |
| "downstream breaks on prose responses" | prompt "always use the tool" | `tool_choice: {"type":"tool","name":...}` |
| "field may be missing from the document" | omit the field | nullable type + enum escape hatch |
| "10k docs, tomorrow, cost-sensitive" | Messages API | Message Batches API |
| "user is waiting for the answer" | Message Batches API | Messages API |
| "human asks the agent to escalate" | try once more autonomously | honor immediately |
| "which extractions get reviewed" | random sample | confidence + ambiguity + document + impact |
| "exploration exceeds the context window" | read fewer files at random | subagent isolation + scratchpad + Glob→Grep→Read |
| "resume yesterday's session" | re-read everything / assume nothing changed | re-analyze changed files + inject prior state |
