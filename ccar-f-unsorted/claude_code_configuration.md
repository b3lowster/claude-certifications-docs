# Claude Code Configuration — Day 3

*Text extracted from the 246-slide deck and grouped by topic. Diagrams and decorative
elements are omitted; only the substantive text is kept.*

---

## Table of Contents

1. [What Claude Code Is and How It Works](#1-what-claude-code-is-and-how-it-works)
2. [The Tool System](#2-the-tool-system)
3. [Project Setup: Onboarding to a Codebase](#3-project-setup-onboarding-to-a-codebase)
4. [Adding and Managing Context Effectively](#4-adding-and-managing-context-effectively)
5. [Claude Code Permission Modes](#5-claude-code-permission-modes)
6. [The CLAUDE.md Memory Hierarchy](#6-the-claudemd-memory-hierarchy)
7. [Load Order and Conflict Resolution](#7-load-order-and-conflict-resolution)
8. [Directory-Scoped CLAUDE.md Files](#8-directory-scoped-claudemd-files)
9. [Excluding Files and Content](#9-excluding-files-and-content)
10. [Instruction Design: What Claude Reads](#10-instruction-design-what-claude-reads)
11. [Common CLAUDE.md Patterns](#11-common-claudemd-patterns)
12. [Creating Custom Slash Commands](#12-creating-custom-slash-commands)
13. [Command Design Patterns and Distribution](#13-command-design-patterns-and-distribution)
14. [What Skills Are and When to Build Them](#14-what-skills-are-and-when-to-build-them)
15. [Authoring Agent Skills](#15-authoring-agent-skills)
16. [Creating and Configuring Subagents](#16-creating-and-configuring-subagents)
17. [Designing Subagents for Delegation](#17-designing-subagents-for-delegation)
18. [Plan Mode: When to Plan First](#18-plan-mode-when-to-plan-first)
19. [Iterating on a Plan](#19-iterating-on-a-plan)
20. [Hook Types in Claude Code](#20-hook-types-in-claude-code)
21. [Implementing Hooks in Claude Code](#21-implementing-hooks-in-claude-code)
22. [Real-World Hook Applications](#22-real-world-hook-applications)
23. [The Claude Agent SDK](#23-the-claude-agent-sdk)
24. [Claude Code in CI/CD Pipelines](#24-claude-code-in-cicd-pipelines)

---

## 1. What Claude Code Is and How It Works

Claude Code is an agentic CLI that runs in your terminal and executes multi-step tasks
autonomously.

### Three Architecture Concepts

| Term | Definition | Example |
|---|---|---|
| **Claude Code** | Anthropic's agentic CLI that runs multi-step tasks in a terminal via a loop of planning, tool use, and observation. | `claude "fix the failing test"` |
| **Planning loop** | The core cycle: Claude plans, calls a tool, observes the result, and repeats until the task is complete. | plan, call, observe, repeat |
| **Permission model** | The layered control system deciding which tools Claude Code may use, enforced before any tool runs. | allow `Bash(git)`, deny `rm` |

### Claude Code Is Not a Chat Tool

Claude Code is an autonomous agent, not a chat interface or autocomplete plugin.

- It doesn't suggest completions — it executes tasks end to end.
- It reads files, runs shell commands, edits code, and calls the web.
- A single instruction can trigger dozens of tool calls before finishing.

### Claude Code vs. IDE Plugins

IDE plugins like Copilot suggest completions. Claude Code executes plans.

- Plugins respond to keystrokes and offer inline suggestions.
- Claude Code receives a goal, reasons, and takes autonomous action.
- Plugins have no shell access; Claude Code runs arbitrary Bash commands.

IDE plugins assist. Claude Code acts independently across the full codebase.

### The Planning Loop — Four Phases

1. **Plan** — reason about what the task needs and pick the next action.
2. **Tool call** — invoke a specific tool with the chosen parameters.
3. **Observe** — read the result and update understanding of the state.
4. **Repeat** — loop until the task is complete or needs clarification.

### Agentic Loop vs. Single-Shot Call

- **Agentic loop** — Claude plans, acts, observes, and decides the next step. Many tool calls per
  task. It self-corrects on surprises and keeps going until the goal is reached. *Example:
  refactor across 12 files.*
- **Single-shot call** — one input, one response, done. No follow-up actions, no observing
  results. Fine for simple generation, not multi-step work. *Example: summarize a paragraph.*

### Context Window — What Fits

Everything in a session competes for space in Claude Code's context window.

- Conversation history, tool outputs, and CLAUDE.md files all count.
- Verbose CLAUDE.md files reduce the space left for code and results.
- When the window fills, earlier content is compacted or dropped.

Context is finite — design CLAUDE.md files and workflows to use it efficiently.

### Session Continuity and Memory

A new session doesn't automatically carry your full conversation history forward.

- CLAUDE.md files load automatically and carry project context across sessions.
- Where enabled, auto-memory notes and explicit summaries preserve state.
- Without those, a new session has no automatic recall of prior chat.

Design workflows to write state explicitly rather than relying on implicit recall.

### The Permission Model

Claude Code's permission model gates tools, and hooks run alongside it as a second gate.

- Permissions come from user, project, local, and managed settings.
- Managed settings win, and deny rules override allow rules.
- A PreToolUse hook runs before the permission check and can deny first.

Both are pre-execution gates, so a hook can block an action before permissions are consulted.

### Key Takeaways: Claude Code Architecture Foundations

- Claude Code is an agentic CLI that executes multi-step tasks autonomously.
- It acts autonomously — not an IDE suggestion tool.
- The planning loop: plan, tool call, observe, repeat.
- Context is finite — CLAUDE.md, history, and tool results all consume it.
- New sessions don't auto-recall chat — write state explicitly.
- Permissions govern what Claude Code can do; hooks are a separate layer.

---

## 2. The Tool System

Claude Code's tool system is a set of built-in capabilities it invokes during each step of a task:
reads, Bash, edits, and web.

### Three Tool System Concepts

| Term | Definition |
|---|---|
| **Tool system** | The built-in capabilities Claude Code can invoke, including Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch, and Agent. Each call is a discrete action. |
| **Edit vs. Write** | Edit makes targeted find-and-replace changes. Write overwrites the entire file. Edit is preferred for code changes. |
| **Agent tool** | Delegates an isolated subtask to a subagent with its own context window, used when the work is too large for the current session. |

### The Core Built-In Tools

Claude Code ships with built-in tools that cover reading, writing, executing, and searching.

- **Read** — view a file's contents without modifying anything.
- **Write** — overwrite an entire file with new content.
- **Edit** — make targeted find-and-replace changes to a file.

Alongside Bash, Glob, Grep, WebSearch, WebFetch, and Agent, these cover most tasks.

### Read — Safe File Inspection

Read lets Claude view a file's contents without making any changes.

- Reads file contents — text, plus rendered images, PDFs, and notebooks.
- No modification risk — it's the safest tool in the set.
- For directory listings use Bash `ls`; to find files use Glob or Grep.

Read is the foundation of every workflow that involves existing code.

### Write vs. Edit — When to Use Each

- **Edit** — use for targeted changes. Finds the exact text and replaces it. Safer for existing
  files. *Example: replace old with new.*
- **Write** — use for new files or full rewrites. Replaces all content; existing lines are gone.
  *Example: create `new_file.py`.*
- **Risk** — Write on an existing file destroys everything not in the new content. No partial
  update.
- **Default** — prefer Edit for code changes. Use Write only for new files or full rewrites.

### Bash — Power and Danger

- **What Bash can do** — run any shell command: scripts, package installs, file changes, database
  queries, API calls. *Example:* `npm install`, `git push`.
- **Why it's dangerous** — a bad command can delete files or cause irreversible damage. No undo.
  Hooks and permissions are the guardrails. *Example:* `rm -rf` with no undo.

### WebSearch — Current Information

WebSearch lets Claude query the web for information beyond its training cutoff.

- Useful for current documentation, recent API changes, and error lookups.
- Results are returned as text that Claude reads and reasons about.
- Claude picks WebSearch when the task needs info it can't know from training.

WebSearch extends Claude Code beyond the local codebase to the broader web.

### Agent — Delegating to a Subagent

The Agent tool spawns a subagent with its own separate context window.

- Useful when a subtask is too large to fit in the current context.
- The subagent runs independently and returns a result to the parent.
- Enables parallel execution of isolated subtasks across contexts.

Agent is the tool that makes Claude Code itself a multi-agent orchestrator.

### Tool Selection Logic

Claude selects tools based on the current step.

| Situation | Tool |
|---|---|
| Reading existing state before acting | Read |
| Modifying specific lines in a known file | Edit |
| Creating a new file or full rewrite | Write |
| Running a command or system operation | Bash |
| Looking up current information | WebSearch |
| Delegating an isolated subtask | Agent |

Tool selection is driven by the task, not by capability.

### Key Takeaways: The Tool System — Core Capabilities

Claude Code's core tools cover inspection, modification, execution, and delegation.

- Read inspects safely; Edit targets changes; Write replaces full files.
- Bash is the most powerful tool; scope it carefully.
- WebSearch retrieves current info beyond training data.
- Agent delegates isolated subtasks with fresh context.
- Tool selection is task-driven at each planning step.

---

## 3. Project Setup: Onboarding to a Codebase

Getting Claude Code oriented on a real codebase requires deliberate setup.

### Three Setup Concepts

| Term | Definition | Example |
|---|---|---|
| **Claude Code** | Anthropic's official agentic CLI that plans, uses tools, and observes in a loop. | `claude` in project root |
| **`/init` command** | A slash command that analyzes the repo and auto-generates a starter CLAUDE.md. | `/init` |
| **CLAUDE.md** | A Markdown file of persistent project instructions, loaded at session start. | `./CLAUDE.md` |

### Initializing Claude Code in a Project

Starting Claude Code looks the same whether the project is new or already established.

- Run the `claude` command from the project root directory.
- **New project** — it starts with minimal context and reads files on demand.
- **Existing project** — it picks up CLAUDE.md at startup, then reads the tree as needed.

The starting directory determines what Claude Code sees by default.

### How Claude Code Discovers Structure

Claude Code actively reads the project to orient itself — no manual file listing required.

- Reads file trees to understand directory layout and module organization.
- Reads README files for project purpose, setup steps, and conventions.
- Reads config files such as `package.json`, `pyproject.toml`, and `.env` examples.

Discovery is automatic, but the quality depends on what actually exists.

### The `/init` Command

Running `/init` is the fastest way to give Claude Code persistent context on an unfamiliar
codebase.

- Analyzes the repo: file structure, languages, frameworks, entry points.
- Auto-generates a starter CLAUDE.md with key files and detected stack.
- Suggests conventions and rules based on what it finds in the code.

You don't build CLAUDE.md from scratch — `/init` does the first draft.

### What `/init` Generates

- **Project name** — detected from repo name, package config, or README header. *Example: from
  `package.json`.*
- **Key files** — entry points, primary source directories, and important config files. *Example:*
  `src/`, `config.yaml`.
- **Languages and frameworks** — detected from file extensions, imports, and dependency
  manifests. *Example: Python, React.*
- **Suggested rules** — initial coding conventions and behavior guidelines inferred from the code.
  *Example: run tests before commit.*

### `/init` Output vs. Final CLAUDE.md

- **`/init` generated output** — scaffolded from repo analysis. Broad coverage, generic wording. A
  useful first draft that may miss team conventions, security constraints, and project rules.
- **Reviewed CLAUDE.md** — edited by the team after `/init`. Adds explicit rules, removes
  irrelevant sections, fixes misdetections. Concise, because every line costs context.

### Reviewing and Refining `/init` Output

The `/init` output is a starting point — shipping it unreviewed creates risk.

- Verify that detected frameworks and languages match the actual stack.
- Add explicit rules the scan couldn't infer: branching policy, test requirements.
- Remove noise: `/init` sometimes flags irrelevant files or boilerplate.

Treat refinement as a required step, not an optional improvement.

### Onboarding Checklist Before First Use

Running Claude Code on a real codebase without preparation leads to poor results.

- CLAUDE.md exists and is accurate — not just the raw `/init` draft.
- `permissions.deny` rules in `.claude/settings.json` exclude secrets, binaries, and irrelevant
  paths.
- Entry points, test commands, and build steps are documented in CLAUDE.md.

These three checks prevent the most common onboarding failures before they happen.

### Key Takeaways: Project Setup

Good onboarding gives Claude Code accurate context before the first task.

- Run `claude` from the root; it reads file trees, READMEs, and configs.
- `/init` generates a starter CLAUDE.md — always review and refine it.
- `/init` output is a draft, not a finished team configuration.
- Check CLAUDE.md accuracy, `permissions.deny` coverage, and documented commands.
- Claude Code performs best when loaded context matches the real project.

---

## 4. Adding and Managing Context Effectively

Context quality directly determines Claude Code's output quality on every task.

### Three Context Concepts

| Term | Definition |
|---|---|
| **`permissions.deny`** | Deny rules in `.claude/settings.json`, like `Read(./.env)`, that block Claude Code from reading files. Controls security exposure and context usage. |
| **Context window** | The finite text space in a Claude Code session. CLAUDE.md, tool results, and code all count against it. When full, earlier content is compacted or dropped. |
| **Session continuity** | The context window is fresh each session, but knowledge persists across them through CLAUDE.md and auto memory. |

### Compaction — Reclaiming Window Space

When context fills, older turns get summarized so the session can keep going.

- Auto-compaction triggers automatically as the context window approaches its limit.
- The `/compact` command runs the same summarization on demand before you hit that limit.
- A CLAUDE.md compact instruction tells the summary what details to always preserve.

Compaction trades verbatim history for a summary that keeps the session alive.

### Configuring `permissions.deny`

Deny rules in `.claude/settings.json` give precise control over what Claude Code can read.

- Use `Read()` with glob patterns, like `Read(./secrets/)`.
- Exclude secrets and credentials: `Read(./.env)`, private keys, auth configs.
- Exclude large binaries: images, compiled outputs, vendor directories.
- Exclude irrelevant paths: CI cache dirs, test fixtures, documentation builds.

`permissions.deny` is both a security control and a context efficiency tool.

### Ways to Add Context in a Session

Claude Code accepts context through several mechanisms beyond CLAUDE.md.

- **File path mentions** — Claude Code reads the file directly in the session.
- **`@` references** — inline pointers to files, directories, or MCP resources.
- **Pasted URLs** — Claude Code fetches and reads the linked content.
- **Inline instructions** — plain-text guidance added directly in the prompt.

Every method costs tokens from the session's finite budget.

### What to Include in Context

Include context that changes what Claude Code would do without it.

- **Architecture decisions** — why the codebase is structured the way it is.
- **Coding conventions** — naming, formatting, and patterns the team enforces.
- **Test requirements** — coverage expectations, frameworks, required patterns.
- **Known constraints** — rate limits, compatibility floors, security boundaries.

If the output would be the same without it, that context isn't worth loading.

### What to Exclude from Context

- **Auto-generated files** — build artifacts and generated code add noise without signal.
  *Example:* `dist/`, `build/`.
- **Large binaries** — images, compiled assets, and data files waste context window space.
  *Example:* `photo.png`, `data.zip`.
- **Irrelevant dependencies** — vendored libraries and unrelated packages dilute useful context.
  *Example:* `vendor/`, `.cache/`.
- **Secrets and credentials** — never load `.env` files or credential configs into context.
  *Example:* `.env`, `secret.key`.

### Configuring `.claudeignore`

`.claudeignore` gives precise control over what Claude Code can and cannot see.

- Syntax mirrors `.gitignore` — glob patterns, directory prefixes, wildcards.
- Exclude secrets: `.env` files, private keys, and auth configs.
- Exclude large binaries: images, compiled outputs, vendor directories.
- Exclude irrelevant paths: CI cache dirs, test fixtures, doc builds.

`.claudeignore` is both a security control and a context efficiency tool.

### Context Inclusion vs. Efficiency

- **Over-inclusion** — every file mentioned adds tokens. Unfocused context consumes space needed
  for code and tool output. Earlier context gets compacted; more files do not improve results.
  *Example: dumping the whole repo.*
- **Targeted inclusion** — include only files relevant to the current task. A smaller, accurate
  context window outperforms a large, noisy one. Save window space for the actual work. *Example:
  3 files for this task.*

### Context Efficiency — Token Budget Thinking

Every file and rule added to a session costs tokens from a finite budget.

- CLAUDE.md loads at session start — verbose files cut working space immediately.
- Tool call results consume tokens — large file outputs compound the cost.
- When the window fills, earlier content is compacted or dropped.

Context loading is a performance decision, not just project housekeeping.

### CLAUDE.md as Standing Context

CLAUDE.md carries knowledge across sessions even though each session's context window starts
fresh.

- Loaded at every session start — provides a consistent baseline automatically.
- Contains standing rules, not per-session briefings or one-time notes.
- Re-establishing context via CLAUDE.md is more reliable than retyping instructions each time.

Think of CLAUDE.md as the project's persistent instruction set, not a conversation log.

### Key Takeaways: Managing Context

Context quality and efficiency are ongoing responsibilities, not one-time setup.

- Add context via file mentions, `@` references, URLs, or inline instructions.
- Include architecture, conventions, constraints; exclude secrets, binaries, noise.
- Compaction reclaims space automatically near the limit or on demand with `/compact`.
- Every file costs tokens — load only what changes Claude Code's output.
- CLAUDE.md is standing context across sessions, not a per-session briefing.

---

## 5. Claude Code Permission Modes

Claude Code pauses to ask before it edits files or runs commands; permission modes control how
often that pause happens, how you switch them, and which one fits which situation.

### Three Core Ideas

| Term | Definition |
|---|---|
| **Permission mode** | A session-wide setting that decides which actions run without asking and which still prompt you. |
| **Working directory** | The folder Claude launched in, plus any additional directories you add. Auto-approval only applies inside it. |
| **Protected path** | Sensitive files like `.git` and `.claude` that are never auto-approved outside bypass mode. |

### The Three Everyday Modes

- **`default`** — reads run freely. Edits and commands prompt you every time. Best for sensitive
  or first-time work.
- **`acceptEdits`** — auto-approves edits and common file commands in your working directory.
  Good for iterating on code.
- **`plan`** — reads and explores only. Proposes changes without touching your source. Good
  before you commit to edits.

### How You Switch Modes

You set the mode through controls, not by asking Claude in chat.

- Press **Shift+Tab** mid-session to cycle `default`, `acceptEdits`, then `plan`.
- Pass `--permission-mode` as a flag when you start a session.
- Set `defaultMode` in settings to pick a persistent starting mode.

The current mode shows in the status bar so you always know where you are.

### What Accept Edits Covers

Accept-edits mode lets Claude create and edit files without prompting each time.

- It also auto-approves common commands like `mkdir`, `touch`, move, and copy.
- Approval only applies to paths inside your working directory.
- Anything outside that scope still prompts you.

You review the changes afterward with `git diff` instead of approving each one inline.

### Plan Mode for Exploring

Plan mode tells Claude to research and propose changes without making them.

- Claude reads files and runs read-only commands to explore.
- It writes a plan but does not edit your source.
- When the plan is ready, you approve, refine, or keep planning.

Approving a plan exits plan mode and switches into whichever mode you chose to proceed.

### Two Hands-Off Modes

- **`dontAsk`** — auto-denies anything you have not pre-approved. Only allow-listed tools and
  read-only commands run. Fully non-interactive, which suits locked-down pipelines.
- **`bypassPermissions`** — skips prompts entirely so tool calls run immediately. Powerful but
  risky. Reserve it for isolated containers or virtual machines where nothing can be damaged.

### Auto Mode in Brief

Auto mode lets Claude work without routine prompts while a safety check watches each action.

- A separate classifier reviews actions before they run.
- It blocks anything that escalates beyond what you asked for.
- Explicit ask rules still force a prompt.

It reduces interruptions on long tasks, but it does not replace review on sensitive work.

### Protected Paths Stay Guarded

- **Never auto-approved** — writes to sensitive files, like your `.git` folder and Claude's own
  `.claude` folder, are never auto-approved outside bypass.
- **Handled per mode** — `default`, `acceptEdits`, and `plan` prompt you. Auto routes to the
  classifier. `dontAsk` denies it. Only bypass skips it.
- **Why it matters** — it keeps your repository state and Claude's own configuration safe from
  accidental edits during a fast session.

### Matching Mode to the Job

The right mode depends on how much oversight the work needs.

- Use `plan` when you want to explore before changing anything.
- Use `acceptEdits` when you are actively reviewing code as it lands.
- Use `dontAsk` for automation where every allowed tool is pre-listed.

Pick more oversight for risky work and fewer interruptions when you trust the direction.

### Key Takeaways: Choosing a Permission Mode

- `default` prompts, `acceptEdits` auto-edits, and `plan` explores without editing.
- Auto runs with a safety classifier; `dontAsk` denies anything not pre-approved.
- `bypassPermissions` skips prompts and belongs only in isolated environments.
- Protected paths stay guarded everywhere except bypass.
- Switch with Shift+Tab, a flag, or the `defaultMode` setting.

---

## 6. The CLAUDE.md Memory Hierarchy

CLAUDE.md files give Claude Code persistent instructions loaded at session start, across four
scopes: managed policy, user, project, and local.

### The Broadest Three Scopes

- **Managed policy** — org-wide CLAUDE.md at a system path. Loads first, the broadest layer, and
  cannot be excluded.
- **User scope** — `~/.claude/CLAUDE.md`. Personal preferences across all your sessions. The
  second-broadest layer, loaded after managed policy.
- **Project scope** — `./CLAUDE.md` or `./.claude/CLAUDE.md` at the repo root. Project
  instructions for all contributors, committed to source control.

### Scope 1 — Managed Policy

Managed policy is the org-wide CLAUDE.md at a system path — the broadest scope, loaded first.

- Distributes organization-wide standards to every user and project.
- Cannot be excluded — it always loads and can't be overridden away locally.
- This is how organizations enforce rules, not a shared project file.

Managed policy sets the floor everyone inherits before any personal or project config loads.

### Scopes 2 and 3 — User and Project

User scope is `~/.claude/CLAUDE.md`; project scope is the CLAUDE.md at the repository root.

- User scope sets personal defaults that apply across all your projects.
- Project scope holds coding conventions, test rules, and architecture standards.
- Project CLAUDE.md should be version-controlled so every contributor gets it.

Leaving the project-level CLAUDE.md out of source control is a common anti-pattern.

### What Belongs at Each Scope

- **Managed policy** — company-wide policies, security requirements, and enforced standards that
  every project inherits.
- **User scope** — personal style preferences, default response verbosity, and your own
  individual tool restrictions.
- **Project scope** — shared coding conventions, test rules, architecture guidelines, and team
  anti-patterns to avoid.
- **Local scope** — your gitignored `CLAUDE.local.md`; personal tweaks scoped to just one project
  on your machine.

### Personal Preferences vs. Shared Standards

- **User-level config** — personal scope: one developer, every project. Right for tone, output
  style, shortcuts. Wrong for anything the team needs consistent. *Example: home-dir CLAUDE.md.*
- **Project-level config** — team scope: everyone on the repository. Right for code style, test
  rules, security. Wrong for anything that varies by individual. *Example:* `./CLAUDE.md`.

### Scope 4 — Local, Plus Org Enforcement

Local scope is `CLAUDE.local.md`, the narrowest layer; org enforcement lives in managed policy.

- Local `CLAUDE.local.md` is gitignored — personal notes for one project.
- Org-wide standards are managed policy, loaded first and impossible to exclude.
- Imports and symlinks share config, but they are not org enforcement.

For rules the whole company must follow, use managed policy, not a shared project file.

### How Claude Code Loads Every Scope

Claude Code loads all applicable CLAUDE.md files at session start and applies them together.

- All scopes apply simultaneously — they're additive, not exclusive.
- Load order runs managed policy, then user, then project, then local.
- When guidance overlaps, the closer, more specific file carries more weight.
- All loaded content counts against the context window — verbose files reduce working space.

### CLAUDE.md vs. the Memory System

CLAUDE.md is a user-authored standing-instruction file, not an auto-updating memory store.

- It's loaded fresh at every session start and counts against the context window.
- You edit it by hand — it doesn't rewrite itself as you work.
- Claude Code has a separate auto memory system that persists notes across sessions.

Keep CLAUDE.md lean for instructions; let the memory system carry evolving project notes.

### Key Takeaways: CLAUDE.md Hierarchy — Four Scopes

- CLAUDE.md has four scopes, all loaded at session start.
- Managed policy is broadest, loaded first, can't be excluded.
- User scope sets personal defaults across your projects.
- Project scope holds shared standards — version-control it.
- Local scope is your gitignored `CLAUDE.local.md`.
- Enforce org-wide rules with managed policy, not a project file.

---

## 7. Load Order and Conflict Resolution

CLAUDE.md files are concatenated into context.

### Conflict Resolution Terms

| Term | Definition |
|---|---|
| **Load order** | CLAUDE.md files are concatenated broadest to most specific. On a genuine conflict there is no reliable precedence, so avoid contradictory instructions. |
| **Complementary instructions** | Instructions at different levels that don't conflict. Both apply simultaneously. No resolution needed; the model follows both. |
| **Config anti-pattern** | Placing team standards in user-level config rather than project-level. Other contributors don't see them, breaking team consistency. |

### How Overlapping Instructions Resolve

All CLAUDE.md files are concatenated into context, so most guidance simply stacks.

- User-level and project-level rules both stay in context together.
- When two rules genuinely conflict, Claude may pick either one.
- There's no reliable precedence, so avoid writing directly contradictory rules.

Design configs so levels complement each other rather than fight.

### Why Conflicts Are Unreliable

Project-level config represents the team's agreed, shared standards.

- It's version-controlled and visible to all contributors.
- On a genuine conflict there's no rule that reliably favors it.
- User-level config stays invisible to teammates — the right home for personal preference.

Keeping the two from contradicting is what makes shared standards reliable.

### Conflict vs. Complementary

- **Conflict** — same behavior, opposite instructions at different levels. Avoid writing these.
  *Example: user says tabs, project says spaces. The system may pick either, so don't write the
  conflict.*
- **Complementary** — different behaviors addressed by each level. Both instructions apply.
  *Example: user says "be concise," project says "always add type hints." Both rules apply.*

### Designing to Avoid Conflicts

- **User-level config** — write only personal style instructions that don't apply to any specific
  project. Avoid putting formatting rules or coding conventions here if those topics will be
  covered at project level.
- **Project-level config** — write project-specific coding standards here. Conflicts resolve
  unpredictably, so state the intended rule explicitly and avoid contradictions.

### The Team-Standards Anti-Pattern

A common mistake is putting team standards in user-level config instead of project-level.

- User-level config is invisible to others — it never leaves your machine.
- Team standards at user level don't apply to anyone else's sessions.
- The team loses consistency with no error or warning — it silently fails.

Team standards belong in the project-level CLAUDE.md committed to source control.

### Seeing Which Instructions Are Active

You can verify which instructions Claude Code is actually following in a session.

- Run `/memory` to see loaded memory files, or `/context` to inspect what's loaded.
- There's no auto-printed startup list — an `InstructionsLoaded` hook can log them.
- Test overlapping instructions explicitly to see which guidance takes effect.

These are useful when onboarding to a new project or debugging inconsistent behavior.

### Load Order in Practice — Worked Example

Consider a team with a no-semicolons rule in project-level CLAUDE.md.

- Developer A has "always use semicolons" in user-level config.
- Developer B has no semicolon rule in user-level config at all.
- Developer A's direct conflict is non-deterministic — Claude may pick either.

The fix: avoid the conflict and state precedence explicitly in the project file.

### Key Takeaways: Load Order — How Guidance Resolves

- CLAUDE.md files are concatenated; genuine conflicts resolve unpredictably.
- Project-level is team scope — visible via source control.
- User-level is personal scope — style, not team standards.
- Complementary instructions both apply — no overlap, no resolution.
- Keep team standards at project level; personal style at user level.

---

## 8. Directory-Scoped CLAUDE.md Files

Claude Code lazy-loads directory-scoped CLAUDE.md files when it reads a file inside them.

### Three Scoping Concepts

| Term | Definition |
|---|---|
| **Subdirectory config** | A CLAUDE.md file placed inside a specific subfolder. Claude Code lazy-loads it when it reads a file inside that directory. |
| **Directory-scoped CLAUDE.md** | Instructions scoped to a directory by placing CLAUDE.md inside it. No registration step required. The file location is the scope. |
| **CLAUDE.md hierarchy** | Project-level config and subdirectory config work together. Subdirectory files add rules. They don't erase parent instructions. |

### How Subdirectory CLAUDE.md Files Work

Placement sets the scope, and the file lazy-loads only when Claude reads inside that folder.

- Drop a CLAUDE.md inside any subfolder — no registration or manifest entry.
- It loads on demand the moment Claude reads a file in that directory.
- Being in the folder isn't enough; the read is what triggers the load.

Scoped instructions follow file reads, not a central registry.

### Augmentation — Not Replacement

Subdirectory configs add rules on top of the parent config; they don't reset it.

- Parent instructions remain active when working inside a subdirectory.
- The subdirectory file adds folder-specific rules to the inherited baseline.
- Files are concatenated root-to-leaf, so the subdirectory file is read last.

Each level adds. There's no guaranteed override winner, so avoid writing direct contradictions.

### Scoping Use Cases by Directory

- **`/frontend`** — React hook patterns, component naming conventions, import order rules.
- **`/tests`** — pytest fixture standards, test naming, coverage threshold requirements.
- **`/infrastructure`** — Terraform module structure, naming, never-delete rules for live
  resources.
- **`/generated`** — never modify; auto-generated, and any changes are overwritten on the next run.

### The `/frontend` Directory Pattern

Frontend codebases have framework-specific conventions that don't apply elsewhere.

- **React hook rules** — which hooks are allowed, custom hook naming with a `use` prefix.
- **Component naming** — PascalCase, co-location with tests and styles.
- **Import order** — external packages before internal modules.

A `/frontend` CLAUDE.md enforces these without cluttering the root config.

### The `/infrastructure` Directory Pattern

Infrastructure code carries high-stakes constraints that must never be lost in context.

- Terraform module structure and naming conventions enforced per-directory.
- Never-delete rules for live resources — explicit and prominent in the config.
- State file handling instructions unique to infrastructure work.

An `/infrastructure` CLAUDE.md keeps never-delete rules close to the code they protect.

### Root Config vs. Subdirectory Config

- **Root CLAUDE.md** — applies across the whole project. Baseline style, security rules, and
  global constraints every directory follows. *Example: project-wide style.*
- **Subdirectory CLAUDE.md** — applies only within that folder. Framework-specific rules unique
  to that code. Adds to root, doesn't replace it. *Example: `/frontend` hook rules.*

### Testing That Scoped Rules Fire Correctly

Verifying that the right rules activate in the right directories is a real validation step.

- Work inside each subdirectory and confirm the folder-specific rules are active.
- Verify parent rules still apply — augmentation shouldn't drop root config.
- Check that explicit contradictions resolve as intended.

Scoped config is only reliable once you've confirmed it behaves correctly in practice.

### Key Takeaways: Subdirectory Scoping

- Directory-scoped CLAUDE.md files scope instructions by placement, no registration needed.
- A subfolder CLAUDE.md lazy-loads when Claude reads a file in that directory.
- Subdirectory configs augment the parent config; they don't replace it.
- Use `/frontend` for framework rules, `/infrastructure` for never-delete constraints.
- The `/generated` pattern locks auto-generated folders against edits.
- Test scoped rules in each directory to confirm the right instructions are active.

---

## 9. Excluding Files and Content

Some files should never reach Claude Code. Secrets, credentials, and private keys need to stay
out of context. This module covers the real exclusion mechanism — permission deny rules in
`settings.json` — how precedence works, and two myths that trip people up.

### Three Exclusion Concepts

| Term | Definition |
|---|---|
| **Deny rule** | A `Read` entry in the permissions block of `settings.json` that blocks Claude Code from reading matching paths. This is the real exclusion mechanism. |
| **Settings hierarchy** | Managed, local, project, and user settings layer together. For permission rules, deny from any layer beats allow from any layer. |
| **Glob pattern** | A path expression with wildcards. One star matches a single segment, two stars match across directories, so you can target files or whole trees. |

### Deny Rules Are the Real Mechanism

File exclusion lives in the permissions block of `settings.json` as `Read` deny entries.

```json
{
  "permissions": {
    "deny": [
      "Read(.env)",
      "Read(secrets/**)"
    ]
  }
}
```

- A deny entry like `Read(.env)` blocks that path.
- Glob syntax: one star is a segment, two stars cross directories.
- Anchors set the base: a leading slash is the settings file's directory, a double slash is
  absolute.

You list the paths that are off-limits, and Claude Code will not read them.

### Protecting Secrets and Credentials

Credential exposure is the highest-value reason to write deny rules.

- **Environment files** — deny `Read` of `.env` and `.env*` variants.
- **Secret directories** — deny `Read` of a secrets folder with a two-star wildcard.
- **Home credentials** — deny `Read` of the SSH and cloud credential folders.

Listing these paths keeps API keys and private keys out of Claude Code entirely.

### Where Settings Live, Highest First

- **Managed** — organization-enforced policy. Highest precedence, cannot be overridden locally.
- **Local** — the gitignored local settings file. Personal overrides that stay off the repo.
- **Project** — the committed project settings file. Shared team rules that travel with the repo.
- **User** — your home settings file. Personal defaults that apply across every project.

### How Precedence Resolves

For ordinary settings, higher layers override lower ones, but permission rules merge.

- Non-permission settings follow managed over local over project over user.
- Permission rules merge across all layers, and deny is evaluated before allow.
- A deny in any layer, even user-level, blocks an allow in a higher layer.

Knowing deny beats allow across layers tells you why a given path is blocked or open.

### Path Anchors and Glob Syntax

Deny rules use gitignore-style path syntax, but they are not gitignore files.

- A bare path or dot-slash path is resolved relative to the working directory.
- A leading slash anchors to the settings file's own directory; a double slash is absolute.
- One star matches a single segment; two stars match across nested directories.

The syntax is familiar, but it lives inside `settings.json`, not in a separate file.

### What Deny Blocks, and What It Doesn't

- **Deny blocks** — Claude Code's built-in tools. So Read, Edit, and shell reads like `cat` or
  `head`. It also blocks indirect access through at-mentions and editor selections.
- **Deny does not block** — arbitrary subprocesses. A Python script Claude Code runs can still
  open a denied file, since deny governs tools, not the operating system. For that, you need
  filesystem-level sandboxing.

### Three Myths to Retire

- **The `.claudeignore` myth** — there is no `.claudeignore` file, and there never has been. Use
  `Read` deny rules instead.
- **The `.gitignore` myth** — Claude Code does not honor `.gitignore` at all. Git-ignored files
  are still fully readable.
- **The env-var myth** — there is no ignore environment variable that hides files. That flag is
  not a real feature.

### Key Takeaways: Excluding Files, the Real Way

- Exclusion uses `Read` deny rules in `settings.json`.
- Deny entries use glob syntax with anchors for cwd, root, and home.
- Permission rules merge across layers; deny beats allow everywhere.
- Deny blocks built-in tools and at-mentions, not subprocesses.
- For OS-level enforcement, pair deny with sandboxing.

---

## 10. Instruction Design: What Claude Reads

A CLAUDE.md that Claude Code actually follows needs specific, actionable, unambiguous directives.

### Three Instruction Design Concepts

| Term | Definition | Example |
|---|---|---|
| **Effective instruction** | Precise, actionable, and clear. | "Use Python 3.11+" |
| **Context cost** | Every CLAUDE.md token cuts code space. | Trim verbose rules. |
| **Instruction encoding** | Repo facts vs. what Claude already knows. | Encode repo conventions. |

### What Makes an Instruction Effective

Effective directives share three traits: specific, actionable, and unambiguous.

- **Specific** — names the exact language version, file path, or convention.
- **Actionable** — written so Claude Code can follow it without interpretation.
- **Unambiguous** — one clear meaning, no room for alternative readings.

Vague instructions get read but not reliably followed — precision is the standard.

### Concise Directives vs. Verbose Docs

Claude Code reads both concise directives and long documentation, but concise wins.

- A short bullet directive is more reliably followed than a paragraph.
- Verbose documentation takes up context space and dilutes signal.
- Instructions buried in prose are easy to miss under context pressure.

Write CLAUDE.md like a style-guide checklist, not a design-document narrative.

### What to Encode — Repo-Specific Conventions

- **Language versions** — exact runtime versions Claude won't guess on its own. *Example: Python
  3.11+, Node 20.*
- **Style rules** — naming conventions, import order, file organization for this repo. *Example:
  PascalCase components.*
- **Test standards** — coverage thresholds, test file naming, fixture organization. *Example: 80%
  coverage minimum.*
- **Git conventions** — commit message structure, branch naming, PR description. *Example:*
  `feat: short summary`.

### What to Encode — Process and Anti-Patterns

CLAUDE.md is also the place to document team process and constraints.

- **PR process** — review requirements, who approves, merge strategy.
- **Coding anti-patterns** — patterns Claude shouldn't suggest in this codebase.
- **Security rules** — no hardcoded credentials, which files are never modified.

These are decisions the team already made — encoding them prevents re-deciding.

### What NOT to Encode

CLAUDE.md is for repo-specific knowledge, not general programming education.

- Don't explain what Python is or how list comprehensions work.
- Don't document standard library functions Claude already knows.
- Don't restate rules your linter or formatter already enforces.

Every line of general knowledge wastes context that could hold actual code.

### Encode This vs. Skip This

- **Encode in CLAUDE.md** — runtime version pinning, team naming conventions, PR process, test
  coverage thresholds, repo-specific anti-patterns, secrets handling. *Example: pin Node 20 LTS.*
- **Skip — Claude already knows** — general Python syntax, standard design patterns, what REST
  APIs are, how git works, common test frameworks, language idioms. *Example: what a REST API is.*

### Structure — Headers, Bullets, Categories

A well-structured CLAUDE.md is faster to read and more reliably followed than a wall of text.

- Use clear header sections: Language, Style, Testing, Git, Security.
- Bullet points outperform paragraphs — one directive per line.
- Group related rules so context pressure hits a whole category at once.

Structure signals priority and makes verification easier after writing.

### Key Takeaways: Instruction Design

Effective CLAUDE.md instructions are specific, actionable, and unambiguous.

- Concise bullet directives outperform verbose narrative documentation.
- Encode repo-specific rules: versions, conventions, process, anti-patterns.
- Skip general programming knowledge Claude already has — it wastes context.
- Structure with headers and bullets so instructions are scannable.

After writing, verify Claude Code follows it in a real session.

---

## 11. Common CLAUDE.md Patterns

Teams develop repeatable CLAUDE.md patterns that encode the right rules for their context.

### Three Pattern-Level Concepts

| Term | Definition | Example |
|---|---|---|
| **CLAUDE.md pattern** | Reusable directives for a team context. | Frontend rule set. |
| **Directive vs. narrative** | Say what to do, not background. | "Use PascalCase." |
| **Tool duplication** | Re-stating what a linter enforces. | Formatting rules. |

### Frontend Pattern — Key Directives

Frontend teams need framework-specific rules that don't apply elsewhere.

- **Component naming** — PascalCase, co-located with tests and styles.
- **CSS methodology** — encode the choice: modules, Tailwind, or styled-components.
- **State management** — approved library and global-state policy.
- **Accessibility** — required ARIA attributes and keyboard navigation.

These rules are framework-specific — they belong in frontend config, not the root.

### Backend Pattern — Key Directives

Backend teams need API and data-layer rules that differ from frontend conventions.

- **API design** — REST conventions, versioning, and error response format.
- **Database queries** — ORM vs. raw SQL policy and optimization rules.
- **Error handling** — expected error types, logging, and response format.

Backend directives encode this service's design, not general web development.

### Full-Stack Pattern — Cross-Cutting Rules

- **Shared types** — TypeScript interfaces shared across frontend and backend. *Example: a shared
  `User` type.*
- **Monorepo structure** — which packages own which concerns and import rules. *Example: apps and
  packages.*
- **Service boundaries** — what each service owns and what crosses boundaries. *Example: no
  cross-service DB.*
- **Shared conventions** — naming rules applied across all monorepo packages. *Example:
  kebab-case files.*

### Security Pattern — Non-Negotiable Rules

Security directives are the highest-value entries in any CLAUDE.md.

- **No hardcoded credentials** — all secrets via env vars or a secrets manager.
- **`.env` policy** — which files exist, which are committed, which never are.
- **Secrets management** — which vault or service handles secrets here.
- **Auth patterns** — approved libraries, forbidden patterns, session rules.

Security rules should be explicit and prominent.

### Test Pattern — Key Directives

Test configuration rules prevent inconsistency as a suite grows.

- **Test file naming** — location and the `.test` vs. `.spec` suffix convention.
- **Coverage thresholds** — minimum percentages enforced in CI.
- **Fixture organization** — where shared fixtures live and how they import.
- **Mock policies** — which dependencies are mocked vs. tested for real.

Test directives encode the team's decisions about quality and structure.

### Directives vs. Narrative Anti-Pattern

- **Directive pattern** — bullet points, one rule per line, grouped by section. Quick to scan and
  simple to apply.
- **Narrative anti-pattern** — long paragraphs with rules buried in prose, verbose docs posing as
  config. Easy to miss, hard to verify.

### Anti-Pattern — Duplicating Linter Rules

Encoding rules existing tools enforce creates two sources of truth.

- If Black handles Python formatting, don't encode formatting rules.
- If ESLint catches an import pattern, don't also write it as a directive.
- If a type checker enforces annotations, the CLAUDE.md rule is redundant.

Let tools enforce what tools can; CLAUDE.md fills the gaps they can't cover.

### Key Takeaways: CLAUDE.md Patterns

Reusable patterns make CLAUDE.md faster and more consistent across projects.

- **Frontend** — component naming, CSS methodology, state, accessibility.
- **Backend** — API design, query policy, error handling format.
- **Security** — no hardcoded credentials, `.env` policy, secrets approach.
- **Test** — file naming, coverage thresholds, fixtures, mock policies.

Avoid narrative anti-patterns and duplicating linter rules.

---

## 12. Creating Custom Slash Commands

Turn any reusable instruction into a one-word `/command` trigger — for example,
`deploy-review.md` becomes `/deploy-review`.

### Slash Command Vocabulary

| Term | Definition |
|---|---|
| **Slash command** | A user-defined shortcut that maps a `/keyword` to a reusable prompt stored in a Markdown file under `.claude/commands/`. |
| **Skills** | Custom commands have merged into skills. Commands still work, but `.claude/skills/<name>/SKILL.md` is now recommended. |
| **`$ARGUMENTS`** | A placeholder in a command file that gets replaced with whatever text follows the command keyword when it's invoked. |

### What Slash Commands Actually Are

Slash commands are shortcuts. They map a keyword to an instruction template you write yourself.

- `/init` and `/help` are built-in commands — they ship with Claude Code.
- `/deploy-review` and `/run-tests` are custom — you create them.
- The command invocation runs the Markdown content.

Custom commands extend Claude Code. They don't change how the underlying model works.

### Project-Level vs. User-Level Commands

- **`.claude/commands/` (project)** — lives in the repo, version-controlled, shared with everyone
  who clones it. Best for team workflows.
- **`~/.claude/commands/` (user)** — your home directory; personal to you in every project. Best
  for private shortcuts.

### File Naming Becomes the Command

The filename is the command — there is no separate registration step.

```
deploy-review.md   →   /deploy-review
explain-file.md    →   /explain-file
```

Keep names lowercase and hyphen-separated so the keyword signals what it does.

### What Goes Inside a Command File

The entire Markdown content becomes the instruction template Claude executes.

- Include a role statement: "You are reviewing a pull request."
- State the task and the expected output format.
- Add constraints like language, tone, or scope.

A good command file reads like a well-crafted system prompt.

### Scope in Practice

- **Shared by default** — project commands in `.claude/commands/` are committed to version
  control; no setup to share.
- **Personal only** — user commands in `~/.claude/commands/` never leave your machine unless you
  copy them.
- **Key point** — the file path sets scope: project is shared, user is personal.

### Using `$ARGUMENTS` for Dynamic Input

Adding `$ARGUMENTS` makes a static command file flexible for variable input.

```markdown
# review-pr.md
Review pull request $ARGUMENTS in detail.
```

- `/review-pr 4521` passes `4521` into `$ARGUMENTS`.
- The placeholder takes whatever text follows the keyword.
- Reference `$ARGUMENTS` multiple times in one template.

Free-form `$ARGUMENTS` for simple input; structured templates when shape matters.

### Built-In Commands You Should Know

Claude Code ships with many built-in commands and bundled skills. Type `/` to list them.

- `/init` and `/help` are two you'll use constantly.
- Others like `/code-review` and `/loop` ship bundled too.
- A custom command named after a built-in gets shadowed.

Check that list before naming a command. Unused names map straight to your filename.

### Key Takeaways: Custom Slash Commands

Slash commands are Markdown files mapping a keyword to a reusable template.

- `.claude/commands/` is project-scoped and shared via version control.
- `~/.claude/commands/` is personal and not shared.
- The filename is the keyword — no registration required.
- `$ARGUMENTS` handles dynamic, variable input.

Creating a command is as simple as dropping a Markdown file in the right directory.

---

## 13. Command Design Patterns and Distribution

Good commands do one thing clearly — and distribute to the whole team automatically.

### Design Vocabulary

| Term | Definition | Example |
|---|---|---|
| **Narrow-scope command** | Does exactly one defined task. | `/review-pr` |
| **Verb-noun convention** | Verb is the action, noun the target. | `/run-tests` |
| **Managed settings** | Pushes commands org-wide. | Org-wide rollout. |

### Core Design Principle — One Thing Well

The most important rule for slash commands is narrow scope.

- **Narrow** — `/review-pr` reviews a pull request, nothing else.
- **Broad anti-pattern** — `/do-everything` reviews, tests, and deploys.
- Small commands compose better than large ones.

Do too many things and the template tangles, giving Claude conflicting guidance.

### Writing Effective Command Prompts

The Markdown content is your full system prompt — four elements make it effective.

- **Role** — the perspective Claude operates from.
- **Task** — describe exactly what to do.
- **Output format** — how the result should be structured.
- **Constraints** — scope, tone, language, what to exclude.

Miss any element and output drifts across runs, defeating a reusable command.

### Free-Form vs. Structured Templates

- **Free-form `$ARGUMENTS`** — best for one piece of variable text. `/explain-file app.py` passes
  the filename cleanly. Low friction.
- **Structured template** — best when input has multiple fields: "Provide: PR number, role, and
  focus." Fewer errors.

### Naming Convention — Verb-Noun

- **Good pattern** — verb-noun: `/review-pr`, `/run-tests`, `/explain-file`. The action is
  immediately clear.
- **Bad pattern** — noun-only or vague: `/pr`, `/code`, `/stuff`. You must open the file to know
  what it does.
- **Consistency wins** — a consistent convention is self-documenting; teammates guess keywords
  before reading docs.

### Team Distribution via Version Control

Project-scoped commands distribute through your existing version-control workflow.

- Add command files to `.claude/commands/` and commit.
- Everyone who clones the repo has the commands immediately.
- Updates propagate through normal pull and merge.

No extra tooling — the repo is already the team's shared source of record.

### Enterprise Distribution — Managed Settings

At scale, managed settings push commands to all users without repo access.

- Admins configure commands in organization-level managed settings.
- Commands are pushed automatically to every user in the org.
- Users can't accidentally delete or override org-level commands.

This enforces consistency on compliance, security, and approved automation.

### Two Anti-Patterns to Avoid

Two design mistakes appear repeatedly in real-world command libraries.

- **Overly broad commands** — one template that reviews, refactors, tests, and ships.
- **CLAUDE.md duplication** — copying standing instructions into a command file.

Broad commands are unpredictable; duplication diverges from CLAUDE.md over time.

### Key Takeaways: Design Patterns for Effective Commands

Effective commands are narrow, clearly named, and distribute automatically.

- Do one thing well — narrow scope is the primary design rule.
- Use verb-noun naming: `/review-pr`, `/run-tests`, `/explain-file`.
- Free-form `$ARGUMENTS` for simple input; structured templates for complex.
- Don't duplicate CLAUDE.md instructions inside command files.

These patterns build a command library the whole team can trust.

---

## 14. What Skills Are and When to Build Them

Skills are reusable instruction packages defined in SKILL.md files, loaded when their description
matches your request.

### Three Instruction Mechanisms

| Mechanism | Definition |
|---|---|
| **Skill** | A reusable instruction package — a SKILL.md plus optional files. Claude loads it when its description matches your request, or you invoke it directly. |
| **Slash command** | A shortcut created in the `.claude/commands/` directory. Now a subset of skills. Manual-only invocation, versus a skill's description-based auto-loading. |
| **Subagent** | A delegated autonomous worker spun up to complete a task independently. An active worker, not a passive instruction template. |

### Skills Are the Middle Layer

Skills sit between always-on config and fully delegated workers in the instruction stack.

- **Always-on config** — CLAUDE.md holds always-applied facts and rules; every interaction sees
  them.
- **Skills** — description-gated procedures Claude pulls in only when relevant.
- **Delegation** — subagents are delegated autonomous workers given a task to complete
  independently.

If a procedure should load only when it fits the request, package it as a skill.

### Skills vs. Slash Commands

Custom commands are a subset of skills now — both invoked with a `/name` you type.

- Skills also load when their description matches the request.
- The difference is a loading pattern, not two separate mechanisms.
- Manual-only commands set `disable-model-invocation`.

Reach for manual-only when you want a shortcut fired only on your explicit request.

### When to Build a Skill

- **Recurring workflows** — multi-step patterns you repeat across many sessions or team members.
- **Team-wide patterns** — conventions that should fire consistently for every contributor.
- **Automatic loading** — procedures that load automatically when relevant, not by typed command.
- **Reusable templates** — instructions that need to work the same way every time they run.

### When NOT to Build a Skill

Not every repeated action warrants a SKILL.md — some patterns belong elsewhere.

- **One-time tasks** — skills exist for reuse, not single tasks with no future value.
- **Simple lookups** — a slash command or CLAUDE.md rule has less overhead.
- **Always-on rules** — if it applies unconditionally, use CLAUDE.md.

Build a skill when frequency, complexity, and reusability all justify it.

### The Build Decision Framework

Three factors determine whether a workflow deserves a skill.

1. **Frequency** — does this workflow come up often enough to formalize it?
2. **Complexity** — is it multi-step enough that a reusable template adds value?
3. **Reusability** — will it run for more than one user, session, or context?

High on all three means build a skill; low means use a simpler mechanism.

### Skills vs. Subagents

- **Skill** — an instruction package stored in SKILL.md. Loaded when its description matches the
  request. Claude Code runs the instructions itself. The skill is the recipe, not a separate
  worker.
- **Subagent** — an autonomous worker delegated to complete a task independently. Has its own
  context, tools, and scope. Reports back when done. An active process, not just instructions.

### Reading the Instruction Stack

Each layer in the stack handles a different activation pattern.

- **CLAUDE.md rules** — always active, apply everywhere, always in context.
- **Skills** — loaded when their description matches the request, defined in SKILL.md.
- **Manual-only commands** — a skill you invoke explicitly with a `/name`.
- **Subagents** — spawned for a task; active autonomous workers.

Choosing the right layer prevents over-engineering.

### Key Takeaways: Skills — Purpose and Decision Framework

- Skills are description-gated instruction packages between CLAUDE.md and subagents.
- Skills load on description match; manual-only commands need explicit invocation.
- Subagents are autonomous workers; skills are instruction packages.
- Build when frequency, complexity, and reusability all score high.
- Skip skills for one-time tasks or always-on rules.

Match each workflow to how it should load.

---

## 15. Authoring Agent Skills

A skill is a folder holding a SKILL.md file, discovered automatically and invoked by its
directory name. This module covers where skills live, what the frontmatter holds, why the fields
are optional, and how the description drives matching.

### Three Building Blocks

| Term | Definition |
|---|---|
| **Skill directory** | A named folder holding a SKILL.md file. The folder name is the skill's name and how you invoke it. Auto-discovered, no registration. |
| **SKILL.md body** | The Markdown below the frontmatter. The actual instructions Claude follows. This always loads, even with no frontmatter at all. |
| **Frontmatter** | An optional YAML block at the top. Every field is optional. Only `description` is really worth adding, since it drives matching. |

### Skills Are Directory-Based

A skill is a directory, not a loose file. The folder name is the skill's identity.

```
.claude/
  skills/
    my-skill/
      SKILL.md
```

- Layout is `.claude`, `skills`, then a named folder holding a SKILL.md.
- The directory name becomes the skill name and the invocation handle.
- Claude auto-discovers every skill folder present, with no registry step.

You invoke a skill by its folder name; that name is the whole addressing scheme.

### The Body Always Loads

The frontmatter is optional. A SKILL.md with no frontmatter is still a working skill.

- With zero frontmatter, the Markdown body loads and the skill still runs.
- Invoking by folder name works with no fields.
- There is no silent failure from a missing frontmatter block.

Frontmatter tunes behavior; the body is the substance, and it loads either way.

### Real Optional Fields

- **`name` and `description`** — set the display label in listings and, crucially, tell Claude
  when the skill fits.
- **`when-to-use`** — a plain-language cue describing the situations the skill should handle.
- **`allowed-tools` and `disallowed-tools`** — `allowed` pre-approves tools so they skip the
  prompt; `disallowed` removes tools from the pool.
- **`model`, `effort`, `context`** — pick a model, set reasoning effort, and set context to
  `fork` for a forked subagent.

### Description Drives Matching

Of every field, `description` carries the weight. It is the one worth writing carefully.

- Claude reads the description to decide whether a skill fits the request.
- A vague description means the skill is passed over when it should fire.
- A sharp, specific description makes auto-invocation reliable.

Everything else tunes behavior; the description is what gets the skill picked at all.

### More Real Fields

- **`agent` and `hooks`** — bind the skill to a specific agent, or wire in lifecycle hook behavior
  when it runs.
- **`paths`** — restrict the skill to certain file paths so it only applies in the right places on
  disk.
- **`argument-hint`** — signal the arguments the skill expects when a user chooses to invoke it
  directly.

### Fabricated Fields to Drop

Some fields get quoted that simply do not exist. Do not reach for them.

- There is no `trigger` field; matching runs off the description.
- There is no `version` field and no `author-tags` field.
- A missing frontmatter block does not cause a silent failure.

Reach for the real fields; invented ones just sit inert.

### How Invocation Works

- **Auto-invocation** — Claude reads each skill's description, matches it against the request, and
  runs the fitting skill on its own. This is the default path.
- **Direct invocation** — you call the skill by its folder name yourself. Handy when you know
  exactly which skill you want to run.

### Key Takeaways: Authoring Skills Cleanly

- A skill is a named directory with SKILL.md, invoked by folder name.
- Frontmatter is optional; the Markdown body always loads.
- Real fields include `description`, tool scoping, and `paths`.
- Description drives matching, so write it specifically.
- No `trigger`, `version`, or `author-tags` field, and no silent-failure trap.

---

## 16. Creating and Configuring Subagents

Claude Code can spawn separate agent instances to handle delegated work.

### Three Subagent Concepts

| Term | Definition |
|---|---|
| **Subagent** | A separate agent instance with its own isolated context window, spawned within the same Claude Code session via the Agent tool for a delegated subtask. |
| **Context isolation** | Each subagent has its own context window. No information passes from the parent automatically. Everything needed must be explicitly provided. |
| **Orchestrator** | The parent agent that decomposes goals, spawns subagents, and synthesizes their results. Does not share context automatically. |

### Subagents vs. API-Context Agents

Claude Code subagents are distinct from multi-agent patterns in the API.

- Subagents are isolated agent instances inside one Claude Code session — not API completions you
  orchestrate yourself.
- Created via the Agent tool within Claude Code, not direct API calls.
- Each runs with its own isolated context window and independent tool access and permissions.

Understanding this distinction matters for correctly placing subagents in architecture.

### How to Create a Subagent

Two creation paths exist: Markdown definition files and the Agent tool in code.

- **Markdown files** — in `.claude/agents/` (project) or `~/.claude/agents/` (user).
- **Agent tool in code** — programmatic spawning from a parent agent's workflow.
- Ask Claude to write the file, or edit it directly to set tool access and scope.

The Agent tool path is used for dynamic delegation during an agentic workflow.

### What Gets Configured Per Subagent

- **Tool access** — which tools the subagent can invoke: Bash, file read/write, web fetch, etc.
- **Isolation** — a worktree gives the subagent its own repo copy; permission rules scope
  filesystem reach.
- **Permissions** — subagents can have narrower or different permissions than the parent agent.
- **Task description** — the explicit instructions defining what the subagent is supposed to do.

### Context Isolation in Practice

A subagent starts fresh — the parent's conversation and prior work don't carry over.

- No automatic inheritance from the parent's conversation or loaded files.
- The parent must explicitly pass every piece of context the subagent needs.
- Subagent results are returned to the parent and then synthesized there.

Isolation is a feature: it prevents context bleed and keeps subagent focus narrow.

### Restricting Permissions Below the Parent

Subagents inherit the parent's tool and permission context by default, but you can scope them
more narrowly.

- A read-only subagent can inspect files without being allowed to write.
- A directory-scoped subagent can only operate within a designated path.
- Restricting tool access reduces the blast radius if the subagent acts unexpectedly.

Minimal-permission subagents follow the principle of least privilege by design.

### When to Spawn vs. When to Stay Inline

- **Spawn a subagent** — when tasks are independent and parallel, need different tools, require
  scope isolation, or are substantial enough for a separate session. *Example: parallel file
  audits.*
- **Stay inline** — when the task needs full parent context, is sequential, the delegation
  overhead exceeds the value, or it is simple enough for the current session. *Example: one quick
  edit.*

### The Handoff — What to Pass Explicitly

A subagent only knows what you tell it. The handoff is a design decision.

- **Task description** — a precise definition of what to do and what done looks like.
- **Necessary context** — only the information the subagent needs, no more.
- **Output format** — the exact structure of what the subagent should return.

An underspecified handoff produces ambiguous results — be explicit about all three.

### Key Takeaways: Creating Subagents

- Subagents are isolated agent instances spawned within one Claude Code session.
- Create via Markdown files in `.claude/agents/`, or the Agent tool.
- Each gets its own context window — nothing inherits.
- Configure tool access, isolation, and permissions per subagent.
- Spawn for parallel independent tasks; stay inline when context is needed.
- The handoff must be explicit.

---

## 17. Designing Subagents for Delegation

Designing a subagent that returns useful, reliable results takes deliberate choices.

### Built-In Subagents and Delegation

Claude Code ships with ready-made subagents so you don't design every one from scratch.

- **Explore** investigates the codebase and reports back without making edits.
- **Plan** does read-only research in plan mode, and **general-purpose** handles open-ended work.
- Delegation is description-driven — a clear description routes the task to the right one.
- You can also route each subagent to its own model, matching capability to task cost.

### Three Design Concepts

| Term | Definition | Example |
|---|---|---|
| **Three design questions** | Task, context, and output format. | Define all three. |
| **Structured output** | Return parseable, well-formed results. | JSON, not prose. |
| **Delegation overhead** | Cost of spawning and managing. | Skip for tiny tasks. |

### The Three Design Questions

Every subagent should be designed by answering three questions before spawning.

1. **What task** — precisely what should the subagent do, and what does done mean?
2. **What context** — exactly what information does the subagent need to complete it?
3. **What output format** — how should results be structured for the parent to use?

Skipping any of the three produces a subagent with unpredictable results.

### Context Window Allocation

Each subagent has its own finite context window — don't over-stuff it.

- Pass only what the subagent needs — excess context crowds out relevant content.
- Relevance over completeness: more context isn't better if it isn't useful.
- The parent holds the full context; the subagent needs a focused slice of it.

Disciplined context allocation keeps subagents fast, focused, and cost-effective.

### Structured Output Principles

- **Use schema** — define the exact output structure upfront: JSON, YAML, or named fields.
- **No summaries** — return machine-parseable results, not narrative descriptions.
- **Include status** — output should include a success or failure indicator the parent can check.
- **Be explicit** — the output format should be specified in the subagent's task description.

### Error Reporting Design

Subagents must surface failures clearly — silent empty results break the workflow.

- A subagent that fails should return an explicit error indicator, not an empty result.
- Include enough context for the parent to understand what failed and why.
- Silent failures are the most dangerous subagent behavior pattern.

Design the error path as deliberately as the success path — the parent needs both.

### When Delegation Helps vs. When It Hurts

- **Delegation helps** — when work is truly independent and parallel, needs different tools,
  isolation prevents contamination, or the task is substantial. *Example: parallel research.*
- **Delegation hurts** — when the task is simple inline, needs the parent's full context, the
  spawning overhead exceeds the work saved, or output depends on parent state. *Example: a
  one-line fix.*

### Anti-Pattern: Spawning Too Many Subagents

Not every sub-task deserves its own subagent — over-spawning creates overhead.

- Each spawn adds latency, cost, and coordination complexity to the workflow.
- Spawning for trivial tasks means overhead exceeds the actual work done.
- The orchestrator spends more time managing subagents than they save.

Reserve delegation for work that genuinely benefits from isolation or parallelism.

### Anti-Pattern: Undefined Output Format

A subagent without a defined output format returns results the parent can't use.

- Natural language summaries can't be reliably parsed by the parent agent.
- Undefined format leads to inconsistent results across runs of the same subagent.
- The parent has to interpret rather than process, introducing errors and ambiguity.

Define the output format in the task description before the subagent ever runs.

### Key Takeaways: Subagent Design

Effective subagents are designed before they're spawned, not after.

- Answer the three design questions: task, context, and output format.
- Pass only what the subagent needs — don't over-stuff the context window.
- Require structured, parseable output with explicit success and failure signals.
- Delegate for parallel independent work; stay inline when overhead exceeds value.

Silent failures and undefined output formats are the two most common mistakes.

---

## 18. Plan Mode: When to Plan First

Plan mode separates planning from doing — Claude proposes a plan, you review and approve, then it
executes.

### Plan Mode Is a Permission Mode

Plan mode is one of the session's permission modes, not a separate tool.

- **Shift+Tab** cycles the modes until you land on plan.
- `--permission-mode plan` starts a session in plan mode.
- A ready plan lets you approve, edit, or reject it.

Approving flips the session out of plan mode into making changes.

### Two Ways to Enter Plan Mode

- **Shift+Tab (mode toggle)** — cycles the permission mode: default, then accept-edits, then
  plan. Landing on plan keeps the whole session in plan mode until you cycle back out.
- **`/plan` prefix (one prompt)** — prefix a single request with `/plan` to plan just that
  prompt. The session returns to its normal mode afterward. Best when you want one plan without
  flipping the session.

### Three Core Plan Mode Concepts

| Term | Definition | Example |
|---|---|---|
| **Plan mode** | Claude proposes a plan, then pauses. | Read-only thinking phase. |
| **Approval gate** | Approve, revise, or reject first. | No writes until approved. |
| **Re-planning** | Amend a plan mid-execution. | Revise after new info. |

### What Plan Mode Actually Does

Plan mode is a read-only thinking phase — Claude reasons without touching files.

- Claude reads the relevant files and context.
- Produces an ordered list of steps with the files to be changed.
- Calls out risks and assumptions explicitly.

The plan is a proposal, not a commitment. You stay in control.

### Explicit vs. Implicit Plan Mode

- **Explicit `/plan` command** — you type `/plan` before your request. Claude enters plan mode
  intentionally and waits — the most reliable trigger for complex work.
- **Implicit invocation** — Claude detects ambiguity or high-risk scope and proposes a plan on
  its own. Helpful, but less predictable than typing the command.

### The Shape of a Good Plan

A well-formed plan has three components you can verify before execution.

- **Ordered steps** — a numbered sequence of discrete actions.
- **Files touched** — every file that will be read or written.
- **Risks called out** — assumptions that could break if wrong.

If any of those three are missing, ask Claude to expand before approving.

### When Plan Mode Pays Off

- **Cross-file refactors** — renaming a type used in dozens of files; a plan shows every touch
  point so nothing is missed.
- **Schema changes** — database or API contract changes ripple through models, validators, and
  tests; planning surfaces every layer.
- **Ambiguous requests** — when the approach is unclear, a plan externalizes Claude's
  interpretation for you to correct first.

### When Plan Mode Is Overkill

Not every task benefits from an upfront plan — knowing when to skip it saves time.

- Single-file changes with a clear, mechanical fix.
- Tiny edits: renaming one variable or fixing a typo.
- Exploratory work where reversibility is trivially easy.

Use plan mode for work that is hard to reverse or spans multiple modules.

### Scope and Context in Planning

- **Iterative refinement** — producing a draft, reviewing it with a human or agent, and revising
  until quality criteria are met. Plans improve through iteration.
- **Codebase exploration** — in plan mode Claude reads files and explores the codebase to build
  the plan. Approving the plan is what moves the session into making edits.
- **Plan-mode pairing** — combining plan mode with subagents for investigation or scoped
  validation to improve reliability on complex changes.

### Reading a Plan Before You Approve

Approving a plan without reading it defeats the purpose — here is what to check.

- Does the scope match your intent, or is Claude doing extra work?
- Are the assumed file paths correct for your project layout?
- Does the plan flag risks your domain knowledge can already resolve?

Catching a wrong assumption here costs seconds; catching it after execution costs more.

### Key Takeaways: Plan Mode in Practice

- Plan mode makes Claude's reasoning visible before any code is written.
- Enter with Shift+Tab or `/plan` for cross-file or ambiguous work.
- A good plan lists ordered steps, files, and risks.
- Skip plan mode for small, reversible, single-file changes.
- Read every plan critically before approving.

---

## 19. Iterating on a Plan

You don't have to accept a plan as-is — approve it, edit it, or trigger a re-plan. The loop is
propose → approve or re-plan → execute.

### Iteration Loop Vocabulary

| Term | Definition | Example |
|---|---|---|
| **Approval gate** | Accept, edit, or reject the plan. | Approve before writes. |
| **Iterative refinement** | Draft, review, and revise the plan. | Tighten step 3. |
| **Plan-mode pairing** | Pair plan mode with subagents or hooks. | Plan plus hook check. |

### The Approval Gate in Detail

Nothing is written until you explicitly approve — this is the primary control point.

- **Accept** the plan as-is to start execution immediately.
- **Edit** the plan inline to trim scope or reorder steps.
- **Reject** the plan entirely and restate your intent.

The gate only works if you use it — skipping review hands control back to Claude.

### Editing a Plan Before Approval

You can modify a proposed plan before execution — editing beats re-planning from scratch.

- **Trim scope** — remove steps that go beyond your intent.
- **Reorder steps** to match dependencies you know exist.
- **Add a step** Claude omitted based on your domain knowledge.

After editing, approve the revised plan — Claude executes your version, not the original.

### Re-Plan After Partial Execution

- **When to re-plan** — when reality diverges mid-execution: a missing file or a wrong schema
  assumption. *Example: a missing config file.*
- **When to proceed** — when the divergence is minor and the remaining steps are still valid.
  Small surprises don't always require a full re-plan. *Example: a minor rename only.*

### Plan-Mode Pairing Patterns

Plan mode pairs with other mechanisms to catch different failure classes.

- **Subagent investigation** — a subagent gathers context, then the main agent builds the plan.
- **Hook validation** — pre-execution hooks check files and run linters on plan steps.
- **Human-in-the-loop** — a developer approves after each major phase to catch drift.
- **Combined patterns** — subagent finds, hook validates, human approves: layered safety.

Layering these three catches different classes of failure.

### The Most Common Plan-Mode Failure

The most common failure is Claude abandoning the plan and improvising mid-execution.

- An unexpected file state prompts Claude to adapt silently.
- Claude adds steps not in the plan without surfacing them.
- You end up reviewing code that doesn't match what you approved.

Counter this by asking Claude to stop and surface any divergence rather than work around it.

### Workflow Tips for Reliable Plans

These practices make plan mode consistently useful across different task types.

- Keep plans short: five to eight steps is the sweet spot for reviewability.
- Name files explicitly rather than using vague references.
- Always ask for risks, even when you think you know the codebase.

Short plans are easier to verify and easier to edit before approving.

### Plan Mode vs. Direct Execution

- **Plan mode first** — higher upfront overhead, but the approval gate catches wrong assumptions
  before code is written. *Example: a cross-file refactor.*
- **Direct execution** — lower upfront overhead; Claude writes code immediately. Best for small,
  reversible, single-file changes where a mistake is cheap. *Example: fix one typo.*

### Key Takeaways: Iteration Loop Foundations

The loop is propose, review, edit or approve, then execute.

- The approval gate is your primary control over Claude.
- Edit plans inline rather than accepting unwanted scope.
- Re-plan when assumptions break mid-execution.
- Watch for silent improvisation — surface any divergence.

Short, explicit plans with named files are the most reliable.

---

## 20. Hook Types in Claude Code

Hooks are user-defined scripts that fire at specific lifecycle events during a Claude Code
session.

### Three Hook Concepts

| Term | Definition |
|---|---|
| **Hook** | A user-defined script registered in settings that runs automatically at a lifecycle event, outside Claude's context at the shell level. |
| **PreToolUse hook** | Fires before a tool call runs. Can inspect, log, or block it. Exit 0 or 1 allows the call; only exit 2 blocks it. |
| **PostToolUse hook** | Fires after a tool call completes. Can inspect results or trigger side effects, but cannot undo the action. |

### What Makes Hooks Different

Hooks execute at the shell level, outside Claude's reasoning context entirely.

- They fire deterministically at lifecycle events regardless of Claude's decisions.
- They are registered in settings, not in CLAUDE.md or prompt instructions.
- Claude cannot be prompted to disable or override a registered hook.

This shell-level execution is what makes hooks reliable for safety enforcement.

### PreToolUse vs. PostToolUse

- **PreToolUse** — fires before the tool runs; it can inspect the pending call and block it with a
  nonzero exit. Use it to prevent an action: logging, linting, or enforcing policy. *Example:
  block writes to `.env`.*
- **PostToolUse** — fires after the tool already ran; it cannot block now. Use it to react:
  logging outcomes, triggering automation, or validating what was written. *Example: run tests
  after edit.*

### The Stop Hook — End of Turn Logic

The Stop hook fires when Claude reaches the end of its execution loop for a turn.

- Useful for cleanup tasks after a multi-step agent run.
- Can run final validation, summary logging, or notification dispatch.
- A single session may trigger the Stop hook multiple times.

Stop fires at agent task completion, not session exit — one session can have many turns.

### Four Hook Events We Focus On

- **PreToolUse** — fires before any tool call. Can block with a non-zero exit. Best for
  enforcement and pre-flight checks.
- **PostToolUse** — fires after a tool call finishes. Cannot block retroactively. Best for
  logging and side effects.
- **Stop** — fires at end of agent turn. Used for cleanup, summary, and final validation tasks.
- **Notification** — fires when Claude emits an alert. Routes status to Slack, PagerDuty, or any
  system.

### Notification Hooks — Routing Alerts Out

The Notification hook fires when Claude emits a notification event during a session.

- The purpose is routing alerts to external systems, not blocking actions.
- Common targets: Slack, PagerDuty, email, and webhook endpoints.
- Does not interrupt or modify Claude's execution flow.

To block an action, use PreToolUse — these hooks only alert.

### Exit Code and Scope Concepts

| Term | Definition |
|---|---|
| **Exit code 0** | The continue signal. Claude Code proceeds with the lifecycle event as planned. |
| **Exit code 2** | The block signal. Stops the action and feeds stderr to Claude. Other non-zero codes like 1 are non-blocking errors that are logged while the tool runs. |
| **Hook scope** | Hooks apply to all matching tool calls in the session unless scoped by tool name or pattern. |

### Choosing the Right Hook Type

Selecting the right hook type means matching the hook's timing to your intent.

- Need to *prevent* an action? Use PreToolUse — the only hook that can block.
- Need to *react* to an action? Use PostToolUse for results and side effects.
- Need end-of-turn logic? Use Stop for cleanup and final validation.

Mixing up hook types is the most common mistake — prevention requires PreToolUse.

### Key Takeaways: Hook Types

Hooks are deterministic shell-level scripts tied to lifecycle events.

- PreToolUse fires before a tool call — blocks with exit 2.
- PostToolUse fires after a call — cannot undo, but can react.
- Stop fires at task completion — cleanup and notification.
- Notification fires on alerts — routes out.
- Exit 0 continues; exit 2 blocks.

---

## 21. Implementing Hooks in Claude Code

Knowing the four hook types is the theory. Implementing them correctly is the practice.

### Implementation Vocabulary

| Term | Definition |
|---|---|
| **Hook registration** | Declaring a hook in the settings file by naming the lifecycle event and the command or script to run. |
| **Exit code convention** | Exit 0 continues, exit 2 blocks and feeds stderr to Claude, and any other non-zero code is a non-blocking error that is logged while the action proceeds. |
| **Hook timeout** | The time Claude Code waits for a hook to finish. A slow hook can stall the session until it times out. |

### Registering Hooks in Settings

Hooks are registered in the Claude Code settings file, not in CLAUDE.md or a prompt.

- Each entry specifies the lifecycle event and the command to run.
- Commands can be inline shell one-liners or paths to external script files.
- Multiple hooks for one event run in parallel — don't rely on order.

Keeping complex logic in script files makes hooks easier to test and version-control.

### Writing a PreToolUse Shell Script

A PreToolUse hook reads the call and exits with the right code.

- Context arrives as JSON on stdin — tool name, input, session.
- Env vars are for path resolution, not tool data.
- Inspect the name or path and apply policy.
- Exit 0 allows; exit 2 blocks.

A stray exit 2 silently blocks every call, so test each path.

### Inline Command vs. Script File

- **Inline command** — a single shell one-liner registered directly in settings. Fast to set up,
  good for simple checks like logging. Hard to test in isolation and to maintain as it grows.
  *Example: echo a log line.*
- **External script file** — a standalone shell script at a file path. Can be tested
  independently, version-controlled, and reused. Preferred for any logic beyond a one-liner.
  *Example:* `hooks/check.sh`.

### Three Common Hook Gotchas

- **Accidental exit 2** — a hook meant to warn returns exit 2. Every matching tool call is
  blocked silently. Exit 1 would not block.
- **Slow hook blocking** — a hook calling a slow external service delays every tool call until it
  completes or times out.
- **Wrong event type** — a PostToolUse hook registered to prevent an action. By the time it
  fires, the tool has already run.

### Exit Code 1 vs. Exit Code 2

Only one non-zero exit code actually blocks the call.

- Exit code 2 is the block signal — policy violation, stops the action.
- Exit code 1 and other codes are non-blocking errors — logged, tool still runs.
- Claude Code feeds the stderr of an exit-2 hook back to Claude.

Document your exit code meanings in the script so maintainers understand intent.

### Three Safety Patterns for Hooks

- **Allowlist pattern** — allow only approved values; block the rest. *Example: allow Read, Edit.*
- **Blocklist pattern** — block known-dangerous values only. *Example: block `rm -rf`.*
- **Dry-run pattern** — log what would be blocked, don't block. *Example: log, then enforce.*

### Testing Hooks Before Deployment

A hook that silently blocks tool calls is harder to debug than one that fails loudly.

- Test exit code paths in isolation: run the script manually with sample inputs.
- Use dry-run mode to log what the hook would block before enabling enforcement.
- Add error messages to stderr so blocked calls are diagnosable.

Never deploy a PreToolUse hook into a live session without verifying exit codes.

### Key Takeaways: Hook Implementation

Hooks are registered in settings and run as shell-level scripts.

- Exit 0 means continue; exit 2 means block.
- A non-zero code other than 2 is non-blocking and won't stop the call.
- Slow hooks stall the session — keep them fast.
- PostToolUse cannot prevent an action already executed.

Test every exit code path before deploying.

---

## 22. Real-World Hook Applications

Hooks are most valuable applied to recurring, concrete automation and safety needs.

### Three Hook Pattern Concepts

| Term | Definition | Example |
|---|---|---|
| **Auto-format hook** | Runs a formatter after any file write. | Prettier on save. |
| **Safety net hook** | Blocks destructive Bash before it runs. | Block `rm -rf`. |
| **Audit log hook** | Records every tool call to a log file. | Log each tool call. |

### Auto-Formatting on File Write

Running a formatter manually after every edit is error-prone and easy to forget.

- **Trigger** — PostToolUse on Write or Edit tool calls.
- **Action** — run `black`, `prettier`, or any formatter via Bash.
- **Result** — every saved file is style-compliant with no extra steps.

PostToolUse fires after the write completes, so the formatter edits the already-saved file.

### Blocking Destructive Bash Commands

PreToolUse hooks intercept Bash commands before they run — the right tool to prevent data loss.

- **Trigger** — PreToolUse on the Bash tool.
- **Check** — inspect the command string for `rm -rf`, `DROP TABLE`, or similar patterns.
- **Block** — return a non-zero exit code to halt execution immediately.

PostToolUse cannot undo a destructive command — only PreToolUse prevents it.

### Audit Logging Approaches

- **Append-only log** — each PostToolUse hook appends the tool name, arguments, plus a timestamp
  to a flat file.
- **Structured JSON log** — write each record as a JSON object for machine-readable queries.
- **Remote logging** — POST log entries to an external endpoint for centralized audit storage.
- **Session summary** — a Stop hook writes a final summary entry when the agent finishes.

### PreToolUse vs. PostToolUse for Logging

- **PreToolUse logging** — records the intent before execution. Captures what was attempted,
  including blocked actions. Useful for detecting policy violations even when commands are
  stopped. *Example: log the attempt.*
- **PostToolUse logging** — records results after execution. Captures what succeeded, including
  output and side effects. Useful for audit trails of completed actions. *Example: log the
  result.*

### Notifications for Long-Running Tasks

When Claude finishes a long batch job, the developer may not be watching the terminal.

- **Trigger** — the Stop hook fires at the end of the agent execution loop.
- **Action** — call a notification service, Slack webhook, or system alert.
- **Payload** — include task name, duration, and success or failure status.

Stop hooks run once per session end, making them ideal for completion signals.

### Automatic Test Runs After Code Edits

Test-run hooks close the feedback loop between editing and validating code.

- **Trigger** — PostToolUse on Write or Edit calls for test-relevant files.
- **Filter** — check the file path to run only the relevant test suite.
- **Action** — execute pytest, jest, or the project's test runner via Bash.

Scoping the hook to relevant paths avoids running the full suite on every edit.

### Cost Tracking Hook Pattern

- **What to capture** — log input tokens, output tokens, the model name, and a timestamp from
  each tool response.
- **Where to store it** — append to a session cost file or push to a lightweight local database
  for querying.
- **When it fires** — PostToolUse on any tool that invokes the model, typically the agent loop
  responses.
- **Why it matters** — running cost data surfaces expensive prompts before they exceed budget
  thresholds.

### Key Takeaways: Hook Patterns

Six patterns cover the most common automation and safety needs.

- **Auto-format** — PostToolUse runs a formatter on Edit.
- **Safety net** — PreToolUse blocks destructive Bash.
- **Audit log** — PostToolUse records every tool call.
- **Notifications** — a Stop hook alerts on completion.
- **Test runs** — PostToolUse on Edit runs tests.
- **Cost tracking** — a Stop hook logs token usage.

Match the hook type to the required lifecycle moment.

---

## 23. The Claude Agent SDK

The Claude Agent SDK, formerly the Claude Code SDK, is the programmatic interface for running
Claude Code sessions from your own code.

### Three Core SDK Concepts

| Term | Definition |
|---|---|
| **Claude Agent SDK** | The programmatic interface for integrating Claude Code into custom applications, formerly named the Claude Code SDK. Its npm package is `@anthropic-ai/claude-agent-sdk`. |
| **Session** | A Claude Code session managed via the SDK. The host application controls session lifecycle, prompt injection, and output handling without terminal interaction. |
| **Streaming** | The SDK capability to receive tool call events and output incrementally. Enables real-time monitoring of agent tasks without waiting for a full response. |

### SDK vs. CLI — Same Power, New Surface

The SDK and CLI share the same Claude Code capabilities. The difference is the control surface.

- **CLI** — interactive terminal commands for developer workflows and one-off tasks.
- **SDK** — method calls from code for automation, pipelines, and custom tooling.
- SDK sessions start and stop under full programmatic control.

Choose the CLI when a human is driving. Choose the SDK when your code is driving.

### SDK vs. Direct API Calls

- **Claude Agent SDK** — controls full agentic sessions: multi-step tool use, file edits, shell
  commands, and streaming output. The SDK manages the session loop, and your code just provides
  prompts and handles events.
- **Claude API (direct)** — makes individual model calls: send a prompt, get a completion. No
  session state, no tool orchestration unless you build it yourself. Best for single-turn tasks or
  custom agent frameworks you manage.

### Creating Sessions Programmatically

The SDK exposes methods to create, configure, and run sessions from any Python or Node.js script.

- Instantiate a session object with your desired model and tool permissions.
- Pass prompts directly as strings — no terminal interaction required.
- Set working directory, environment variables, and context at session creation.

Programmatic session creation is the foundation for embedding Claude Code in any workflow.

### Injecting Context and Instructions

- **System prompt** — pass a system prompt at session creation to set role, tone, and behavioral
  constraints up front.
- **Context files** — inject file contents or directory listings so the session starts with
  useful codebase knowledge loaded.
- **Tool permissions** — explicitly allow or restrict tool types (read, write, shell) to enforce
  safety boundaries per session.
- **Dynamic instructions** — append follow-up prompts mid-session to steer the run without
  restarting it.

### Streaming Output and Tool Events

SDK sessions stream events as they happen, giving your application visibility into every step.

- Tool call events fire each time the agent invokes read, write, or shell operations.
- Output chunks arrive incrementally, enabling live progress displays or logs.
- Error events surface failures immediately so your code can react or retry.

Streaming is essential when sessions run long or downstream systems need real-time feedback.

### Embedding Claude Code in Custom Tooling

The SDK's primary use case is integrating Claude Code into automated systems your team operates.

- **Code review bots** — trigger sessions on pull requests, stream results to GitHub comments.
- **Documentation generators** — pass source files and receive updated docs automatically.
- **CI/CD pipelines** — run sessions as pipeline steps with structured exit codes.

Any workflow that benefits from agentic code understanding is a candidate for SDK integration.

### When to Choose SDK vs. API

- **Use SDK for agentic tasks** — multi-step workflows, file edits, shell commands, or work
  needing the full Claude Code agent loop.
- **Use API for model calls** — single-turn completions, embeddings, or custom agent frameworks
  you build and orchestrate yourself.
- **Use SDK for observability** — when you need streaming tool events, real-time monitoring, or
  structured output from a running session.
- **Use API for simplicity** — lightweight tasks with no tool use, where a plain
  prompt-and-response cycle is all you need.

### Key Takeaways: SDK

The Claude Agent SDK gives code full control of agentic sessions.

- SDK vs. CLI: same power, code replaces the terminal.
- SDK vs. API: the SDK runs agentic loops; the API makes single calls.
- Inject prompts, context files, and permissions at creation.
- Stream tool events and output in real time.

Choose the SDK when code drives a multi-step workflow.

---

## 24. Claude Code in CI/CD Pipelines

Claude Code runs non-interactively in automated pipelines using the `-p` flag.

### Three CI/CD Core Concepts

| Term | Definition | Example |
|---|---|---|
| **The `-p` flag (print mode)** | Runs Claude Code non-interactively, then exits. | `claude -p "run tests"` |
| **Non-interactive mode** | Finishes the task and stops, never prompting. | Works in GitHub Actions. |
| **Pipeline job hang** | Without `-p`, it waits for input and blocks. | Job stalls until timeout. |

### Why `-p` Is Required in CI/CD

In automated pipelines, no human is present to answer prompts.

- Without `-p`, Claude Code enters interactive mode and waits.
- CI/CD runners have no terminal, so the job blocks.
- The `-p` flag runs the agentic loop non-interactively and exits when done.
- It returns a status code the pipeline can act on.

The `-p` flag bridges Claude Code and automation.

### The `-p` Flag and Its Companions

Pass `-p` (or `--print`), then add the flags a pipeline needs.

- `--output-format json` returns parseable output; `stream-json` emits events live.
- `--resume` and `--continue` pick up a prior session.
- `--allowedTools` scopes which tools the run may invoke.
- Exit code 0 means success; non-zero signals an error.

Together these flags make a headless run predictable.

### Structuring Tasks for Automation

- **Atomic tasks** — each run does one thing, like linting a file.
- **Clear prompts** — write complete, standalone prompts.
- **Structured output** — request JSON so steps can parse the result.
- **Idempotency** — make tasks safe to re-run on retry.

### Interactive vs. Non-Interactive

- **Interactive mode (default)** — Claude Code opens a REPL session, prompts for input, and waits
  for user responses. Suited for developer workstations where a human is present throughout.
  *Example: a developer at a terminal.*
- **Non-interactive mode (`-p`)** — Claude Code accepts a single prompt, completes the task,
  writes output to stdout, and exits. Required for CI/CD, cron jobs, and any context with no human
  present. *Example: a CI pipeline step.*

### GitHub Actions Integration

The documented path is the `claude-code-action` workflow, not a raw `-p` step.

- Run `/install-github-app` to set up the action and app.
- Trigger it with an `@claude` mention or a prompt input.
- Store `ANTHROPIC_API_KEY` as a GitHub Actions secret.
- A raw `claude -p` step is general CI, not this integration.

The official action is the supported GitHub Actions path.

### Permissions in Automated Contexts

Automated pipelines should run with the minimum permissions the task requires.

- Scope allowed tools to only what the specific task needs.
- Don't grant write access to directories the task only reads.
- Restrict network tools if the task is purely file-based.
- Store API keys as secrets, never as plaintext in workflow files.

Minimum permissions limit the blast radius if a step behaves unexpectedly.

### Output Parsing and Error Handling

- **Capture stdout** — assign output to a shell variable or file.
- **Check exit codes** — a nonzero exit means an error; fail fast.
- **Validate output** — if you asked for JSON, validate its shape.
- **Set timeouts** — add a timeout to prevent runaway jobs.

### Key Takeaways: CI/CD Integration

The `-p` flag enables non-interactive, single-turn automation for pipelines.

- Without `-p`, Claude Code waits for interactive input and the job hangs.
- With `-p`, Claude Code executes the task, writes to stdout, and exits.
- Scope tool permissions to the minimum the task requires.
- Parse stdout and handle non-zero exit codes in pipeline scripts.

The `-p` flag is the answer whenever a scenario describes CI/CD automation.
