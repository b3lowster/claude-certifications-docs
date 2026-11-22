# Tool Design — Day 2

*Text extracted from the 266-slide deck and grouped by topic. Diagrams and decorative
elements are omitted; only the substantive text is kept.*

---

## Table of Contents

1. [How Claude Selects Tools](#1-how-claude-selects-tools)
2. [Writing Tool Descriptions That Route](#2-writing-tool-descriptions-that-route)
3. [Designing Tool Input Schemas](#3-designing-tool-input-schemas)
4. [Required, Optional, and Enum Fields](#4-required-optional-and-enum-fields)
5. [Strict Tool Use](#5-strict-tool-use)
6. [Returning Useful Errors from Tools](#6-returning-useful-errors-from-tools)
7. [Tool Retry Logic and Partial Success](#7-tool-retry-logic-and-partial-success)
8. [Tool Scoping per Agent](#8-tool-scoping-per-agent)
9. [Selection Heuristics and Failures](#9-selection-heuristics-and-failures)
10. [Controlling Tool Selection](#10-controlling-tool-selection)
11. [Tool Search for Large Catalogs](#11-tool-search-for-large-catalogs)
12. [What MCP Is and Why It Exists](#12-what-mcp-is-and-why-it-exists)
13. [The Three MCP Primitives](#13-the-three-mcp-primitives)
14. [Building Your First MCP Server in Python](#14-building-your-first-mcp-server-in-python)
15. [Defining and Exposing Tools via MCP](#15-defining-and-exposing-tools-via-mcp)
16. [Defining and Exposing Resources](#16-defining-and-exposing-resources)
17. [MCP Prompt Templates](#17-mcp-prompt-templates)
18. [Implementing MCP Clients](#18-implementing-mcp-clients)
19. [STDIO Transport: Local Servers](#19-stdio-transport-local-servers)
20. [StreamableHTTP: Remote MCP Servers](#20-streamablehttp-remote-mcp-servers)
21. [Sampling: Server-Initiated LLM Calls](#21-sampling-server-initiated-llm-calls)
22. [Notifications, Roots, and Progress](#22-notifications-roots-and-progress)
23. [Tool Boundaries and Least Privilege](#23-tool-boundaries-and-least-privilege)
24. [MCP in Production: Auth, Versions, Ops](#24-mcp-in-production-auth-versions-ops)
25. [The Built-in Toolset](#25-the-built-in-toolset)
26. [Built-in vs. Custom: The Decision](#26-built-in-vs-custom-the-decision)

---

## 1. How Claude Selects Tools

Claude reads tool descriptions and uses that text as the primary signal for routing decisions.

### The Tool Description Defined

| Term | Definition | Example |
|---|---|---|
| **Tool description** | Natural-language text telling Claude what a tool does and when to use it. | `get_weather` returns the current local temperature. |
| **Routing** | How Claude matches a request to the best-fitting tool from descriptions. | A temperature question routes to `get_weather`. |
| **Misrouting** | When a vague description makes Claude call the wrong tool. | A temperature question wrongly calls `web_search`. |

### The Description Is the Decision

When Claude receives a request, it reads every tool description before deciding.

- Tool names are hints, not rules — the description carries the weight.
- Claude performs semantic matching between the request and descriptions.
- A strong description narrows the candidate set to one obvious choice.

A weak or missing description makes every tool equally likely to be chosen.

### Strong vs. Weak Descriptions

- **Strong description** — specifies what the tool does, when to call it, what inputs it expects,
  and what it returns. Claude can confidently select it with minimal ambiguity.
- **Weak description** — gives only a name or vague phrase like "handles data." Claude has no
  basis for confident selection and may misroute or skip the tool entirely.

### How Selection Actually Works

Claude evaluates all tool descriptions together in the context of the current request.

- It assigns implicit relevance weights based on semantic fit to the request.
- Higher-confidence matches win over lower-confidence candidates.
- No single keyword triggers selection — the full description context matters.

This is why a clear, complete description is the single most important design choice.

### Four Routing Outcomes

- **Correct selection** — the right tool selected on the first attempt. The description was
  precise and unambiguous.
- **Misroute** — the wrong tool called. Two descriptions read as too similar to tell apart.
- **No call** — no tool matched at all. The description missed the request's intent entirely.
- **Fallback** — Claude asks the user to clarify rather than guessing between unclear options.

### The Name Is Not Enough

Developers often assume tool names are sufficient for routing — they aren't.

- A tool named `search` could mean web search, database search, or file search.
- Claude does not infer purpose from naming conventions alone.
- Two tools with similar names but different purposes will cause misrouting.

Every tool needs a description that removes the ambiguity the name leaves behind.

### Three Description Failure Modes

- **Vague intent** — says what a tool *is* but not *when* to use it. Claude can't separate it
  from similar alternatives.
- **Missing scope** — omits what the tool does NOT handle. Overlapping tools create selection
  ambiguity.
- **No output signal** — doesn't say what the tool returns. Claude can't confirm it will produce
  what's needed.

### Exam Trap: Names vs. Descriptions

The exam tests whether you know what actually drives tool selection.

- Distractor answers will frame tool names as the routing mechanism.
- The correct answer always points to the description as primary.
- Even a perfectly named tool routes incorrectly with a vague description.

Know this cold: description quality is the variable that determines routing.

### Key Takeaways: Description Drives Every Routing Decision

Claude uses tool descriptions as the primary signal for selecting which tool to call.

- Tool names are hints; descriptions carry the actual routing weight.
- Semantic matching compares request intent to description content.
- Vague descriptions lead to misrouting, no-calls, or fallback behavior.
- The exam tests description quality as the root cause of routing failures.

Get descriptions right, and routing takes care of itself.

---

## 2. Writing Tool Descriptions That Route

Knowing descriptions drive routing is the foundation; writing ones that route correctly is the
craft.

```
get_order_status
  Does:       looks up the status of an order by order ID
  Use when:   a user asks where their order is
  Don't use:  for refunds or cancellations
```

### Description Anatomy Components

| Component | Definition |
|---|---|
| **What it does** | A one- or two-sentence purpose statement describing the tool's function without ambiguity. |
| **When to use it** | The conditions under which this tool should be selected; framing helps Claude choose among similar tools. |
| **When not to use it** | Boundary conditions that scope the tool; what it doesn't handle is as valuable as what it does. |

### The Three-Part Description Pattern

Every strong tool description follows a pattern you can apply consistently.

1. **Part one** — what the tool does and what it returns.
2. **Part two** — when to call it: the triggering conditions.
3. **Part three** — what it does not handle: the exclusion clause.

A description with all three parts gives Claude everything it needs to route.

### Including Input Format Expectations

The description should signal what kinds of inputs the tool expects.

- Mention required parameter types: IDs, date ranges, query strings.
- If input is structured, name the format: JSON, array, enum.
- Avoid restating the schema verbatim — summarize the input intent.

Input hints help Claude form the call correctly, not just select the tool.

### Including Output Format in the Description

What a tool returns shapes whether Claude considers the request fulfilled.

- State whether the tool returns an object, a list, a boolean, or a status.
- Note if the output needs further processing or is ready to use directly.
- Signal how errors surface: null return, exception, or an error field.

Output framing lets Claude confirm the tool satisfies the request before calling it.

### Reducing Selection Ambiguity

- **Consolidate** — group related operations into one tool with an action parameter instead of
  many overlapping tools.
- **Namespace names** — prefix names by service, like `github_list_prs`, so selection stays clear
  as the library grows.
- **Exclude explicitly** — state what a tool doesn't handle, like "does not perform bulk updates
  or deletions."

### Overlapping vs. Separated Tools

- **Overlapping tools** — two tools with similar scopes and vague descriptions. Claude can't
  confidently distinguish them, so selection becomes unreliable and misrouting is likely.
- **Separated tools** — the same two tools, each with explicit domain anchors, trigger
  conditions, and exclusion clauses. Claude selects the right one reliably, with far less
  ambiguity.

### Testing Tool Selection Reliability

Writing a good description is step one; confirming it routes correctly is step two.

- Try varied phrasings of the same underlying intent.
- Try requests that could match several tools — check which wins.
- Try edge cases: requests the tool should not handle.

Reliability means consistent correct routing across a broad input distribution.

### Three Testing Patterns

- **Positive test** — send a request the tool should handle. Confirm it gets selected and runs
  correctly.
- **Negative test** — send a request it should NOT handle. Confirm another tool wins or Claude
  asks to clarify.
- **Disambiguation test** — send a request matching two tools. Confirm one wins on specificity,
  not order.

### Key Takeaways: Good Descriptions Route by Design

- Tool descriptions follow a three-part pattern: what it does, when to use it, and what it
  doesn't handle.
- Include input format hints to help Claude form calls correctly.
- Include output format to confirm the tool satisfies the request.
- Consolidate with an action param, namespace names, and exclude explicitly.
- Validate with positive, negative, and disambiguation tests.

Description quality is a design choice that determines routing reliability.

---

## 3. Designing Tool Input Schemas

A well-designed schema is a contract — it tells Claude exactly what to send, and in what format.

```json
{
  "name": "string",
  "age": "integer"
}
```

### Core Schema Vocabulary

| Term | Definition |
|---|---|
| **Tool schema** | The JSON Schema on a tool definition: accepted parameters, their types, constraints, and which are required. |
| **Input parameter** | A named field in the schema carrying a value Claude must supply; may be required or optional. |
| **Schema validation** | The check that confirms Claude's generated arguments match the declared schema before the call runs. |

### JSON Schema as a Contract

JSON Schema is the standard format for defining tool inputs in the Anthropic API.

- It declares field names, types, and constraints in a machine-readable object.
- Claude reads the schema to understand what arguments are valid.
- With strict tool use on, grammar-enforced constraints hold; advisory ones still need your own
  validation.

Think of the schema as the interface specification between Claude and your code.

### Naming Parameters Clearly

Parameter names are the first signal Claude uses to understand a field's purpose.

- Use specific, unambiguous names: `customer_id` beats `id`.
- Use snake_case and avoid abbreviations the model might misread.
- Names should reflect domain meaning, not implementation details.

Clear names reduce hallucinated or mismatched argument values.

### Naming Anti-Patterns to Avoid

- **Too generic** — names like `data`, `input`, or `value` give Claude no context about the field.
- **Overloaded names** — reusing one name for different purposes builds wrong links over a
  session.
- **Cryptic abbreviations** — cryptic short names force Claude to guess at meaning and raise
  argument errors.

### Descriptions as Micro-Prompts

Each parameter description is read by Claude to understand the field's expected content.

- Descriptions guide argument generation just like prompt instructions do.
- Include the field's purpose, valid values, and edge-case behavior.
- Omitting descriptions degrades argument quality even when types are correct.

### Weak vs. Strong Parameter Descriptions

- **Weak** — saying only "the user ID" tells Claude the type but not the format, source, or
  constraints. Claude is left to guess at the specifics.
- **Strong** — saying "UUID of the authenticated user from the session token" tells Claude the
  source, format, and constraints, leaving no room for misinterpretation.

### Type Constraints and Formats

JSON Schema supports constraints, but only some are grammar-enforced under strict tool use.

- `type`, `enum`, and supported `format` values are grammar-enforced at generation.
- `minLength`, `maxLength`, `minimum`, and `maximum` are advisory — validated after the fact.
- `pattern` is only partially enforced, so validate it in your own code too.

Treat advisory constraints as guidance for Claude, then re-validate before execution.

### Schema Constraint Properties

- **`type`** — the data type: string, integer, number, boolean, or array. Blocks wrong-type
  values.
- **`pattern`** — a regular expression a string must match. Enforces exact shape for IDs and
  codes.
- **`format`** — a semantic hint like `date-time` or `email` signaling expected string meaning.

### Key Takeaways: Schema Design Drives Call Reliability

A precise input schema is the foundation of reliable tool use in Claude-powered systems.

- JSON Schema is the standard contract for tool inputs in the Anthropic API.
- Clear parameter names reduce hallucinated or mismatched argument values.
- Descriptions act as micro-prompts that guide Claude's argument generation.
- Type constraints and format hints catch malformed values before execution.

Every schema decision is an investment in call accuracy.

---

## 4. Required, Optional, and Enum Fields

Not every parameter carries equal weight — some are essential, others add context when available.

### How Required vs. Optional Works

JSON Schema uses a top-level `required` array to declare which parameters Claude must supply.

- Fields listed in `required` must be present in every tool call.
- Fields not listed are optional and Claude may omit them.
- Optional fields should have a default or handle absence gracefully.

Getting the required/optional split right is a core schema design decision.

### Required and Optional Concepts

| Term | Definition |
|---|---|
| **Required parameter** | Listed in the schema's `required` array. Claude must include it every call or the call is rejected. |
| **Optional parameter** | Not listed in `required`. Claude may omit it; the tool must handle absence gracefully. |
| **Default value** | A fallback applied when an optional parameter is left out, keeping the required set lean. |

### Designing the Required Set

Every field in your `required` array increases the chance a call fails when context is missing.

- Only mark a field required if the tool cannot function without it.
- Ask: can the tool provide a sensible default or skip the field?
- Over-requiring forces Claude to hallucinate values to satisfy the schema.

A lean required set produces more reliable tool calls across diverse contexts.

### Required Field Decision Framework

- **Can't function without it** — mark it required. The tool has no meaningful path forward
  without this value. *Example: `user_id` on a `get_user` lookup.*
- **Useful but not critical** — make it optional with a sensible default declared in your
  implementation code. *Example: `region` defaults to "US".*
- **Rarely provided** — make it optional and handle its absence so the tool degrades gracefully.
  *Example: an optional note field.*

### Input Examples for Complex Inputs

Schema alone doesn't always show Claude how to use a tool well.

- Add an optional `input_examples` array of sample inputs to the definition.
- Each example has to be schema-valid or the request is rejected.
- Best for nested objects, many optional fields, or format-sensitive inputs.
- Descriptions come first, but examples lift accuracy on the hard, complex tools.

### Enum Fields for Constrained Inputs

When a parameter accepts only a fixed set of values, an enum constraint is the right tool.

- Declare `enum` as an array of the only acceptable string values.
- Claude selects from the enum list instead of a free-form string.
- Great for routing: `priority` is one of low, medium, or high.

Enums eliminate entire classes of invalid argument errors.

### Free-Form String vs. Enum

- **Free-form string** — a string field with no enum leaves Claude unlimited options. It may
  produce "urgent", "high-priority", or "URGENT" — any of which can break your routing logic.
- **Enum constraint** — an enum of low, medium, high forces Claude to pick one of three exact
  values every time. Your routing code works predictably with no normalization step needed.

### Enum Use Cases in Practice

Enum fields appear in two main patterns across real tool schemas:

- **Routing inputs** — directing actions to different handlers or workflows.
- **Extraction fields** — classifying entities into predefined categories.
- Best when the valid set is stable and small (2 to 10 values).

Avoid enums where valid values change often or number in the hundreds.

### Schema Constraint Properties

- **`enum`** — an array of the only acceptable values. Claude must choose from the listed set.
- **`minimum` / `maximum`** — numeric bounds for integers and numbers. Useful for limits and
  scores.
- **`additionalProperties`** — set to false, it blocks fields not declared in the schema for
  strict input.

### Key Takeaways: Schema Precision vs. Flexibility

Required vs. optional and enum constraints are the two most impactful schema decisions.

- Require only fields the tool genuinely cannot function without.
- Optional fields should handle absence with defaults or graceful fallback.
- Enum constraints eliminate free-form variation for fixed-value inputs.
- Numeric bounds and `additionalProperties` complete a strict, reliable schema.

Schema design is the first line of defense against bad tool calls.

---

## 5. Strict Tool Use

Forcing a tool call is only half the battle; setting `strict` to true guarantees the model's tool
inputs match your schema.

### The Problem It Solves

Without strict mode, a model can hand your function the wrong shape of data.

- A string like "two" where you needed a number.
- A quoted `"2"` instead of the integer `2`.
- A required field left out entirely.

Each breaks your function and forces validate-and-retry.

### What Strict Guarantees

- **Valid name** — the tool name the model returns is always a real tool from your request. It
  cannot invent one.
- **No type mismatch** — every argument matches the type your schema declared. You never get a
  string where you wanted an integer.
- **No missing fields** — every required field is present. Nothing your function depends on ever
  gets quietly dropped.

### How It Works

This technique is called **constrained decoding**, and it happens as the model writes.

- Add one property, `strict` set to true, next to the input schema.
- The API compiles your schema into a grammar.
- That grammar constrains the model's token sampling.
- Only schema-valid tokens can be produced.

### What You Stop Writing

- **No validation layer** — you drop the code that checks the model's output.
- **No retry loop** — you drop the logic that re-prompts the model.
- **Type-safe calls** — your functions receive correctly typed arguments.

### A Booking Example

The failure mode disappears, so the downstream call just works.

- A booking function needs `passengers` as an integer.
- Without strict, you might get the string "two".
- Or a quoted `"2"` that your code has to coerce.
- With strict on, you always receive the integer `2`.

### Strict Tools vs. Structured Outputs

- **Strict tool use** — constrains the tool *inputs* so a tool call conforms to your input
  schema. This is about the data flowing into your functions.
- **Structured Outputs** — constrains the text *response* itself to a schema you supply. This is
  about the shape of the model's final answer to the caller.

### The Shared Machinery

- **Same pipeline** — strict tools compile schemas into grammars using the same pipeline as
  Structured Outputs.
- **Schema subset** — both accept the same supported JSON Schema subset. Some advanced keywords
  are not allowed.
- **Cached grammars** — compiled schemas are cached for up to 24 hours since last use, so repeat
  calls skip recompilation.

### Pairing It With Forced Calls

Strict combines cleanly with the `tool_choice` settings.

- Set `tool_choice` to `any` to guarantee a call.
- Set `strict` to true to guarantee valid inputs.
- Together you get a call you can trust end to end.

That pairing is a strong foundation for reliable agent workflows.

### Key Takeaways: Trusting Your Tool Inputs

- `strict` set to true guarantees tool inputs conform to your schema.
- Valid name, matching types, required fields present.
- It works through constrained decoding.
- Same grammar pipeline and 24-hour cache as Structured Outputs.
- It removes your validation and retry loops.

---

## 6. Returning Useful Errors from Tools

When a tool fails, the error response shapes what Claude does next: retry, try an alternative, or
ask the user.

### The Real `tool_result` Schema

At the API level, a `tool_result` block carries only three fields.

- **`tool_use_id`** — matches the result back to the originating call.
- **`content`** — the result text, or an error message when it failed.
- **`is_error`** — set true to flag that execution failed.

Any richer error shape you design lives inside `content` as text.

### Treat Tool Output as Untrusted

- **The risk** — tool results often carry outside content: web pages, email, uploads. Attackers
  can hide instructions in it.
- **Indirect injection** — hidden text tries to redirect Claude, a confused-deputy attack using
  your tool's trust.
- **The guardrail** — keep that content inside `tool_result` blocks, never in the system prompt or
  plain user text.

### Three Core Error Concepts

| Term | Definition |
|---|---|
| **Tool error response** | A structured reply indicating failure: an error type, a readable message, plus optional machine-readable fields. |
| **Error code** | A short, stable identifier like `NOT_FOUND` that lets code and models route errors without parsing text. |
| **Suggested action** | An optional field telling Claude what to do next: retry, ask the user, or escalate. |

### What a Good Error Response Includes

A useful tool error is more than a status code.

- **Error type** — a stable, machine-readable code.
- **Message** — a human-readable explanation of the failure.
- **Context** — which input or resource caused the failure.
- **Suggested action** — retry, escalate, or ask the user.

Claude uses all four fields together to reason about recovery.

### User Errors vs. System Errors

- **User error** — the caller provided bad input: wrong format, missing field, value out of
  range. Claude should surface this to the user and ask for a correction rather than retry.
- **System error** — the downstream service is unavailable, timed out, or returned an unexpected
  response. Claude can retry, wait, or try an alternative without involving the user.

### Designing Error Messages for Claude

Error messages written for humans often confuse model-driven recovery.

- Keep codes stable across versions so logic doesn't break.
- Include the input value that triggered the error.
- Avoid raw stack traces as the primary message.

Structured errors let Claude take the right next action without extra prompting.

### Four Error Response Fields

- **`error_code`** — a stable, machine-readable string. Never changes between releases or deploys.
- **`message`** — a human-readable explanation. Safe to surface directly in a chat reply.
- **`context`** — key-value pairs identifying exactly which input or resource failed.
- **`suggested_action`** — retry, escalate, or ask_user. Tells Claude what to do next.

### When to Include Suggested Action

Not every error needs an explicit suggested action, but many benefit from one.

| Error | Suggested action |
|---|---|
| `RATE_LIMITED` | Retry after a delay. |
| `INVALID_INPUT` | Ask the user to correct the value. |
| `SERVICE_UNAVAILABLE` | Try an alternative or escalate. |

Explicit guidance cuts the number of back-and-forth model decisions.

### Escalation vs. Recovery

- **Recovery** — Claude handles the error itself: retries, reformats, or switches tools.
- **Escalation** — the error needs a human. Claude surfaces it plainly instead.
- **Error routing** — uses the `error_code` field to choose: recover or escalate.

### Avoiding Common Error Design Mistakes

Poorly designed errors force Claude into bad recovery decisions.

- Don't return a success status with an error buried in the body.
- Don't use vague codes like `FAILED` or `ERROR` with no subtype.
- Don't omit context when the same code can mean different things.

Clear, specific errors save tool calls and reduce user frustration.

### Key Takeaways: Error Responses That Help Claude Recover

Well-structured tool errors give Claude everything it needs to recover or escalate gracefully.

- Include `error_code`, `message`, `context`, and `suggested_action`.
- Distinguish user errors from system errors so recovery logic is correct.
- Keep error codes stable and machine-readable.
- Explicit suggested actions reduce unnecessary model decisions.

Good error design is as important as good success design.

---

## 7. Tool Retry Logic and Partial Success

Not every failure is final — some errors are transient, and some calls only partly complete.

### Retry and Reliability Concepts

| Term | Definition |
|---|---|
| **Partial success** | Some batch items complete while others fail; the response says which items did and didn't. |
| **Idempotency** | Calling a tool repeatedly with the same input gives the same result, making retry safe. |
| **Circuit breaker** | A pattern that halts calls to a failing service after several failures to prevent cascades. |

### When Retry Is and Isn't Appropriate

Retrying blindly can make failures worse rather than better.

- Transient errors like timeouts and rate limits are good retry candidates.
- User errors like invalid input should never be retried unchanged.
- System errors with no suggested action need judgment before retrying.

The error code should tell Claude which category the failure falls into.

### Idempotent vs. Non-Idempotent Tools

- **Idempotent** — calling the tool twice with the same input is safe. A GET, a lookup, or a
  write with a unique client-generated ID all produce the same outcome on repeat calls.
- **Non-idempotent** — each call produces a new side effect. A payment charge, an email send, or
  an append-only log entry creates a duplicate if retried without precautions.

### Designing Tools for Safe Retry

Idempotency doesn't happen by accident — it has to be designed in.

- Accept a client-generated idempotency key with each request.
- Store the key and return the same response for duplicate calls.
- Use conditional writes that check state before modifying it.

Tools built this way let Claude retry on transient failures without duplicates.

### Three Retry Failure Modes

- **Duplicate side effects** — a non-idempotent tool called twice creates two charges, emails, or
  records.
- **Infinite retry loop** — no backoff or retry limit lets Claude hammer a failing service
  forever.
- **Stale state** — retrying after partial completion re-applies steps that already succeeded.

### Surfacing Partial Success to Claude

A batch tool that partially completes should not return a generic error.

- Return a `partial_success` status, not a plain error.
- Include a `succeeded` list with IDs or keys that completed.
- Include a `failed` list with per-item error codes and context.

Claude can then decide what to retry, what to skip, and what to surface.

### Circuit Breaker States

- **Closed** — healthy. Calls pass through; failures are counted.
- **Open** — once the threshold is exceeded, calls are blocked at once; a fallback or error
  returns.
- **Half-open** — after a wait, a probe call tests recovery. If the probe succeeds, the breaker
  closes; if it fails, it re-opens.

### When to Use a Circuit Breaker

Circuit breakers protect your system from cascading failures caused by a struggling dependency.

- Use one when a downstream tool has intermittent outages.
- Set a failure threshold that triggers the open state.
- Define a recovery probe interval to test for restoration.

Without one, Claude keeps calling a broken service until the context fills with errors.

### Key Takeaways: Reliable Tools Handle Failure by Design

Reliable tools handle transient failures and partial completions without forcing Claude to guess.

- Make tools idempotent so retry is safe by default.
- Return `partial_success` with `succeeded` and `failed` lists for batch ops.
- Use circuit breakers to stop cascading calls to broken services.
- Match retry decisions to error type, not just error presence.

Designing for failure separates robust tool integrations from fragile ones.

---

## 8. Tool Scoping per Agent

Every tool you give an agent loads into its context on every turn — whether it's used or not.

### Tool Distribution Foundations

| Term | Definition |
|---|---|
| **Tool bloat** | An antipattern where an agent gets too many tools, raising context usage and routing confusion. |
| **Specialist subagent** | A subagent with a narrow tool palette and focused mission, used to isolate context and cut errors. |
| **Context isolation** | Giving subagents only the tools and context they need, keeping the orchestrator's context clean. |

### The Cost of Too Many Tools

Every tool description loads into the model's context on every turn, whether used or not.

- Each tool consumes tokens from the available context budget.
- Larger tool catalogs slow inference and increase cost per turn.
- Irrelevant tools crowd out content the model actually needs.

Fewer tools keep the context window focused and the agent faster.

### Bloated Agent vs. Specialist Agent

- **Bloated agent** — receives the full tool catalog. Context fills with descriptions of tools it
  will never call. Routing errors rise as similar tools compete, and responses get slower and
  less predictable.
- **Specialist agent** — receives only the tools its mission needs. Context stays lean, routing is
  unambiguous, and each call has a single obvious candidate. Behavior is consistent and auditable.

### Tool Overlap and Routing Ambiguity

When two tools could plausibly handle the same request, the model picks inconsistently.

- Overlapping tool descriptions cause split routing across similar tools.
- The model may alternate between tools on identical inputs.
- Ambiguity compounds when tool names are similar or generic.

Overlapping tools are a latent reliability risk in any multi-tool agent.

### Scoping Principles

- **One agent, one mission** — each agent gets a single role and only the tools that role
  requires.
- **Tools earn their slot** — add a tool only when no existing tool covers the need. Default to
  removal.
- **Audit the catalog** — review tool assignments periodically. Prototype tools often outlive
  their use.
- **Name precision matters** — names and descriptions must be distinct enough to identify the
  right tool.

### The Specialist Subagent Pattern

Specialist subagents each receive a narrow palette sized exactly for their role. An orchestrator
might delegate to a research subagent with search-and-fetch tools only, and a write subagent with
file-write and database-update tools only.

Specialization makes each subagent's behavior easier to predict and test.

### Routing Vocabulary

| Term | Definition |
|---|---|
| **Tool routing** | The model's choice of which tool to call from the palette, based on the request and descriptions. |
| **Tool description** | The text explaining what a tool does — the primary signal Claude uses to route requests. |
| **Cross-cutting tool** | A tool that applies across roles, like logging or status reporting, valid on several agents. |

### When to Consolidate Tools

Not every tool should be isolated — some functions genuinely span several agents.

- Logging tools belong on every agent that generates observable output.
- Status reporting tools are reasonable on orchestrators and subagents alike.
- Consolidation fits when the tool's scope is truly cross-cutting.

The rule is minimize, not eliminate — consolidate only what is genuinely shared.

### Key Takeaways: Scoping Tools to Agent Roles

Focused tool palettes reduce context overhead and produce more reliable routing.

- Tool bloat increases tokens consumed and degrades routing accuracy.
- Overlapping tools create ambiguity that compounds over time.
- The specialist subagent pattern assigns tools by role, not convenience.
- Cross-cutting tools like logging may appear on multiple agents by design.

Start lean, add tools deliberately, and audit as agents evolve.

---

## 9. Selection Heuristics and Failures

Tool selection is driven by description-text quality — understanding the failure modes builds
reliable routing.

### Selection Failure Modes

- **No-good-match** — no tool matches the request, so the model fabricates, refuses, or misroutes.
- **Two-good-matches** — two tools both look valid; the model alternates or favors the longer
  description.
- **Disambiguation** — writing distinct verbs and narrow scopes so each tool clearly wins.

### How Claude Selects a Tool

Tool selection is a matching problem between the request and the description text.

- Claude reads the request and scores each tool's description for relevance.
- The highest-scoring match wins, regardless of the tool's actual capability.
- Poorly written descriptions produce poor matches, even for capable tools.

Description quality is the primary lever for improving routing accuracy.

### No-Match vs. Two-Match Failure

- **No-good-match failure** — the model sees no strong candidate. It might invent a tool name,
  refuse to answer, or call the least-wrong option and produce wrong output. Silent errors are
  common here.
- **Two-good-matches failure** — the model sees two strong candidates. It alternates between them
  across turns, or picks whichever has the more detailed description. Behavior becomes
  non-deterministic and hard to audit.

### Disambiguation Patterns

Writing descriptions that distinguish tools clearly is the highest-impact fix for routing errors.

- Use distinct verbs: *retrieves* vs. *writes* vs. *deletes* vs. *summarizes*.
- Narrow the scope: list what the tool does AND what it does not do.
- Avoid generic names like `process` or `handle` that match everything.

One strong verb plus a narrow scope eliminates most two-match failures.

### System Prompt Guidance

- **Explicit routing rules** — name each tool and say when to use it. *Example: use
  `search_orders`, not `search_global`.*
- **Tie-breaking instructions** — when two tools are close, pick one by a clear rule. *Example:
  prefer `get_invoice` when an ID is given.*
- **Negative examples** — tell the model which tool NOT to call. *Example: never call
  `delete_record` for status.*

### Logging Tool Selections in Production

Routing quality degrades over time as tools are added, renamed, or rephrased.

- Log every tool selection: which tool was chosen, and what the request was.
- Track selection frequency to find tools that are never or rarely called.
- Alert when a tool's call rate drops sharply from its baseline.

Logging turns tool selection from a black box into a measurable property.

### Architectural Vocabulary

| Term | Definition |
|---|---|
| **Tool description** | The text explaining what a tool does — the main signal Claude uses when routing. |
| **Tool routing** | The model's choice of which tool to call from the palette, based on the descriptions. |
| **Selection drift** | Gradual decay of routing accuracy as descriptions go stale or start to overlap. |

### Add a Tool or Refine an Existing One?

Before registering a new tool, ask whether an existing one can cover the need with a better
description.

- Adding a tool increases catalog size and introduces potential overlap.
- Refining a description improves routing without growing the palette.
- A new tool is justified only when the use case is truly outside existing scope.

Default to refinement. Add a tool only when scope expansion is clearly necessary.

### Key Takeaways: Designing for Reliable Tool Selection

Tool selection is driven by description quality, and failures follow predictable patterns.

- No-good-match failures occur when descriptions don't cover the request.
- Two-good-matches failures occur when two tools are ambiguously similar.
- Distinct verbs and narrow scopes are the primary disambiguation technique.
- Production logging surfaces selection drift before it becomes an incident.

Design intentionally, log selections, refine before adding.

---

## 10. Controlling Tool Selection

By default the model decides whether to call a tool; the `tool_choice` parameter lets you steer
that decision across four settings.

### The First Three Settings

- **`auto`** — the model decides whether to call any tool. This is the default whenever you supply
  tools.
- **`any`** — the model has to call one of your tools, but it picks which one.
- **`tool`** — you name a specific tool and the model always calls that one.

### Auto Is the Sensible Default

With `auto`, the model reads the request and decides whether a tool helps.

- The model reads the request and judges whether a tool helps.
- It answers directly when no tool is needed.
- It calls a tool when the task actually requires one.
- It can narrate what it plans to do before the call.

### When You Force a Tool Call

Forcing guarantees the call but removes the friendly explanation — weigh it carefully.

- Setting `any` or `tool` prefills the reply so a tool call is guaranteed.
- The model skips any natural-language preamble.
- You get a tool call as the very first content block.
- Use it when a plain-text answer is never acceptable.

### Any vs. Tool

- **`any`** — guarantees some tool runs, but leaves the choice of which one to the model. Good for
  routing a request across a set of valid actions.
- **`tool`** — names one exact tool and forces the model to call it every time. Good for a fixed
  step in a pipeline, like data extraction.

### Where None Fits

- **Blocks all tools** — `none` stops the model from calling any tool, even when tools are
  attached to the request.
- **Automatic when empty** — `none` is the default value whenever you send no tools at all.
- **Turn off mid-flow** — keep the tools attached but set `none` to force a text-only turn in a
  longer conversation.

### A Routing Pattern

Set `any` to turn the model into a dependable dispatcher across your tools.

- You want the model to always pick from a set of database actions.
- Set `any` so it must call one of them.
- It routes the request to the right action for you.
- You never get an off-topic prose reply.

### Things to Watch

- **Caching cost** — changing `tool_choice` invalidates cached message blocks. Tool definitions
  and the system prompt stay cached.
- **Extended thinking** — with extended thinking, only `auto` and `none` are supported; `any` and
  `tool` return an error.
- **Prefilled reply** — `any` and `tool` prefill the assistant turn, so no explanation appears
  before the call.

### Choosing the Right Setting

Match the setting to how much freedom the task should have.

- Open-ended assistant work — stay with `auto`.
- Must act but any action is valid — use `any`.
- One mandatory step — name it with `tool`.

When in doubt, `auto` plus a clear instruction often does the job.

### Key Takeaways: Steering Tool Selection

- `auto` lets the model decide — the default when tools exist.
- `any` forces some tool; the model still chooses which.
- `tool` forces one named tool every time.
- `none` blocks all tools — the default when none are sent.
- Pick the tightest setting the task genuinely needs.

---

## 11. Tool Search for Large Catalogs

A few tools are easy to manage; hundreds are not. The tool search tool and `defer_loading` keep
the model's active tool set small, so it keeps choosing the right one.

### The Accuracy Cliff

The model's ability to pick the right tool degrades past a certain catalog size.

- Selection accuracy holds fine with a handful of tools.
- It starts to degrade once you exceed thirty to fifty tools.
- More options means more chances to pick a wrong one.
- Piling every tool into context works against you as you scale.

### The Second Cost

- **Context bloat** — every tool definition loaded up front eats context. A typical multi-server
  setup can burn 55 thousand tokens before any work happens.
- **Loaded up front** — by default every tool definition sits in the context window from the
  first request, whether or not it gets used.
- **The fix** — tool search loads only the few tools a request needs, often trimming those
  definitions by over 85 percent.

### How Tool Search Works

The model discovers tools just in time rather than carrying them all from the start.

- You add a tool search tool to your tools list and it does the finding.
- You still send every tool definition in the request.
- Deferred tools stay out of the active context.
- Matches come back as references that expand into full definitions on demand.

### The `defer_loading` Flag

- **Keeps context small** — `defer_loading` set to true keeps a tool out of the active context
  until the model actually searches for it.
- **Still fully defined** — you still include the deferred tool's full definition in the request
  so the search can find and expand it.
- **One stays loaded** — at least one tool, normally the search tool itself, has to stay
  non-deferred.

### Keeping Selection Sharp

The whole point is a small, focused active tool set at any moment.

- Only the three to five relevant tools load per request.
- The model chooses from a short list, not a crowd.
- Accuracy stays high even across thousands of tools.
- You get the reach of a huge catalog without the confusion of a huge menu.

### Ways to Search the Catalog

- **Regex variant** — the model writes regex patterns to match against your tool fields. Patterns
  cap at 200 characters.
- **BM25 variant** — the model searches with plain natural-language queries instead of patterns.
  Queries cap at 500 characters.
- **What it scans** — both variants search across tool names, descriptions, argument names, and
  argument descriptions alike.

### When to Reach for It

Tool search earns its keep once your tool surface passes a few thresholds.

- You have ten or more tools available.
- Your definitions run past ten thousand tokens.
- Below ten tools, or when every tool is used every turn, standard tool calling is fine.

### Key Takeaways: Scaling to Huge Toolsets

- Accuracy degrades past thirty to fifty tools, and full catalogs bloat context.
- Tool search loads only the tools a request needs.
- `defer_loading` keeps a tool out of context until found.
- You still send every definition so search can expand it.
- A small active set keeps accuracy high across thousands.

---

## 12. What MCP Is and Why It Exists

The Model Context Protocol defines how AI models connect to tools and data.

### Defining MCP

| Term | Definition |
|---|---|
| **Model Context Protocol** | An open standard for how AI models connect to external tools and data via a uniform interface. |
| **MCP server** | A process that exposes tools, resources, and prompt templates to clients over a defined transport. |
| **MCP client** | The AI application or agent host that connects to servers, discovers capabilities, and routes requests. |

### The Integration Problem MCP Solves

Before MCP, every tool integration required a custom adapter.

- Each tool had its own API contract and auth scheme.
- Developers rewrote glue code for every new capability.
- Swapping tools meant rewriting application logic.

MCP replaces that chaos with one consistent interface.

### Before MCP vs. After MCP

- **Before MCP** — every tool needs a custom adapter, unique auth handling, and
  application-specific glue code that breaks whenever the underlying tool updates its interface.
- **After MCP** — any compliant server exposes capabilities through the same interface. Swap out
  tools without rewriting your application logic or renegotiating contracts.

### The MCP Client-Server Model

MCP follows a familiar client-server pattern with clear role separation.

- The server hosts and exposes capabilities.
- The client — your app or agent host — connects and discovers.
- One client can connect to multiple servers at once.

### Why MCP Is Model-Agnostic

- **Open standard** — any AI model or runtime can implement it; not proprietary to any vendor.
- **Uniform interface** — servers don't know which model consumes them; clients don't know who
  built the server.
- **Composable** — mix and match MCP servers from different vendors without integration rework.

### How Claude Connects to an MCP Server

Claude acts as the MCP client in Anthropic-hosted tooling.

- Claude connects and negotiates capabilities in an initialize handshake.
- It then calls list methods to enumerate tools, resources, and prompts.
- Those available capabilities populate its context.
- The model then reasons over those capabilities like any other context.

### Three Key MCP Concepts

| Term | Definition |
|---|---|
| **Transport layer** | The communication channel between an MCP client and server. Standard I/O for local processes, or Streamable HTTP for remote servers. |
| **Capability discovery** | The process by which an MCP client queries a server to learn which tools, resources, and prompts are available before any requests are made. |
| **Capability negotiation** | The initialize-handshake step where client and server declare supported features, after which the server advertises its tools, resources, and prompts. |

### MCP in Production AI Systems

MCP reshapes how teams build and maintain AI-powered applications.

- Capability teams ship independent MCP servers without coordination overhead.
- Application teams consume them without knowing implementation internals.
- Upgrades happen server-side without breaking clients downstream.

This contract-first model scales teams and reduces coupling.

### Key Takeaways: MCP Standardizes AI Tool Connections

MCP replaces ad-hoc integrations with one shared open protocol.

- MCP defines a uniform client-server interface for tool access.
- The server exposes capabilities; the client discovers and consumes.
- Claude acts as the MCP client in Anthropic tooling.
- Open and model-agnostic — not proprietary to any vendor.

One contract replaces dozens of custom adapters.

---

## 13. The Three MCP Primitives

MCP organizes every server capability into exactly three types: tools, resources, or prompts.

### The Three Primitives Defined

| Primitive | Definition |
|---|---|
| **MCP tool** | A callable function the model can invoke to perform actions or retrieve computed results. Model-controlled. |
| **MCP resource** | A URI-addressable data object the application injects into context. Application-controlled. |
| **MCP prompt** | A reusable, parameterized template users select to pre-populate an interaction. User-controlled. |

### Tools — The Model Calls Them

Tools are the most active primitive — they let Claude take action.

- Defined with a JSON schema describing inputs and outputs.
- The model decides when and whether to invoke a tool.
- Examples: run a query, send a message, call a calculation.

If the capability involves *doing* something, it's a tool.

### Resources — The App Injects Them

Resources are data sources the application loads into context.

- Identified by URI — the server registers what's available.
- The application decides which resources to include, not the model.
- Examples: a config file, a database record, a policy document.

If the capability is static data to read, it's a resource.

### Tools vs. Resources

- **Tools** — model-controlled. Claude decides when to call them. Used for actions, computations,
  or retrieving dynamic results that depend on the current conversation context.
- **Resources** — application-controlled. The host decides what to inject. Used for static or
  semi-static data the model should have in context before it generates a response.

### Who Controls Each Primitive

- **Model controls tools** — the model decides when to invoke a tool, based on the conversation
  and its own reasoning.
- **App controls resources** — the host application picks which resources to load into context
  before generation.
- **User controls prompts** — the human picks a prompt template to set up the interaction before
  it begins.

### Prompts — The User Selects Them

Prompts are reusable templates the server makes available to users.

- Stored and versioned on the MCP server.
- Parameterized — the user fills in variables at selection time.
- The user explicitly applies a prompt, not the model or app.

If the capability is a starting-point workflow for a human, it's a prompt.

### Three Supporting Concepts

- **JSON schema** — defines a tool's name, inputs, and types so the model can call it. *Example:*
  `{"name": "get_user", "user_id": "string"}`
- **URI address** — a unique address the client uses to request a resource. *Example:*
  `file:///config/policy.md`
- **Parameterized template** — a prompt with named variables filled in before the run. *Example:*
  "Summarize {document} in {n} words."

### Choosing the Right Primitive

Picking the right primitive makes your server easier to use correctly.

- Action or computation the model triggers? Use a **tool**.
- Background data to load before the model responds? Use a **resource**.
- Reusable workflow the user kicks off intentionally? Use a **prompt**.

Ask who drives the decision: model, application, or user.

### Key Takeaways: Three Primitives, Three Ownership Models

Tools, resources, and prompts span all the capabilities a server can expose.

- Tools are model-controlled: Claude invokes them during reasoning.
- Resources are application-controlled: the host injects them into context.
- Prompts are user-controlled: humans select them to start workflows.
- JSON schemas, URI addressing, and parameterized templates back each primitive.

Match the primitive to who drives the decision.

---

## 14. Building Your First MCP Server in Python

The MCP Python SDK gives you a structured way to expose tools, resources, and prompts to any
client.

### Three Server Foundation Concepts

| Term | Definition |
|---|---|
| **MCP server** | A process exposing tools, resources, and prompts to clients over a transport. Handles capability registration, request routing, and response formatting. |
| **MCP Inspector** | The official MCP debugging tool that lets developers explore registered tools, send test requests, and inspect responses without a full client. |
| **Tool primitive** | A callable function exposed by an MCP server that the model can invoke. Tools are model-controlled: the AI decides when and how to call them. |

### Installing the MCP Python SDK

The MCP SDK is the only required dependency for a minimal server.

- Install it with `pip install mcp`, or add it to your requirements file.
- The SDK bundles the server runtime, transport adapters, and type stubs.
- Python 3.10 or higher is required for the SDK's async features.

A clean virtual environment keeps your MCP server isolated from other projects.

### stdio vs. Streamable HTTP Transports

- **stdio transport** — the server reads JSON-RPC messages from stdin and writes responses to
  stdout. Ideal for local development and client-managed processes where the client spawns the
  server directly.
- **Streamable HTTP transport** — the server listens on a URL and clients connect over HTTP.
  Server-Sent Events optionally stream messages back. Used for remote servers and multi-client
  scenarios.

### Minimal Server Structure

Every MCP server follows the same three-part structure regardless of what it exposes.

1. Create the server instance with a name and version string.
2. Register tools, resources, and prompts before starting.
3. Run the server with the chosen transport using asyncio.

The SDK's `run()` function blocks until the server shuts down or an error occurs.

### Four Project Structure Conventions

- **`server.py`** — entry point. Creates the server instance and calls `run()`.
  *Example:* `mcp = Server("demo"); mcp.run()`
- **`tools.py`** — tool definitions and handler functions, imported on startup.
  *Example:* `@mcp.tool()` / `def search(q): ...`
- **`requirements.txt`** — pin the `mcp` package and tool dependencies for reproducibility.
  *Example:* `mcp>=1.0`
- **`README.md`** — document the transport, available tools, and required env vars.
  *Example:* `## Setup` and `## Tools`

### The Lifespan Context Manager

FastMCP manages startup and shutdown through a single lifespan async context manager.

- Code before `yield` runs at startup: open connections, load config, warm caches.
- Code after `yield`, in a `finally` block, runs at shutdown to release resources.
- Pass it to the constructor; handlers reach resources via the lifespan context.
- Tools registered before `run()` is called are available immediately when the first client
  connects.

### Capability Registration and Negotiation

Clients and servers exchange capability lists at the start of a session.

- The server declares which primitives it supports: tools, resources, or prompts.
- The client inspects the list and decides which capabilities to use.
- Unregistered tools won't appear in the list and can't be called.

Register all tools before `run()` so they're visible during the initial handshake.

### Common Initialization Mistakes

- **Registering after `run()`** — tools added after `run()` is called are never sent to the client
  during capability negotiation.
- **Blocking the event loop** — sync I/O blocking in a tool stalls other requests. Use async libs
  or `run_in_executor`.
- **Missing error handling** — the SDK catches uncaught exceptions as error results, but raise
  `ToolError` rather than hiding details.

### Key Takeaways: MCP Server Setup and Lifecycle in Python

Setting up an MCP server means picking a transport and registering tools before startup.

- Install the SDK and pick stdio or Streamable HTTP.
- Register tools before `run()` so clients see them at handshake.
- Use a lifespan context manager to open and close connections.
- Keep handlers async to avoid blocking the event loop.

A clean setup is the foundation everything builds on.

---

## 15. Defining and Exposing Tools via MCP

Tools are the primary way MCP servers give language models access to external systems. This
module covers the SDK decorator, input schemas, result formats, error responses, and how to
verify everything with the MCP Inspector before connecting a client.

### The `@mcp.tool` Decorator

With FastMCP, create `mcp = FastMCP("server-name")`, then register functions on it.

- Apply `@mcp.tool()` directly above your handler function.
- The function name becomes the tool name visible to clients.
- The docstring becomes the tool description the model uses to decide when to call it.
- Write descriptions that explain what the tool does and when it's useful, not how it works
  internally.

### Text Results vs. Structured Results

- **Text result** — return a plain string when the model needs to read and present information.
  Simple, human-readable, and easy to compose. *Example:* `return "3 open tickets found"`
- **Structured result** — return a JSON dict when downstream code needs to parse the results.
  Best for multi-field records and lists. *Example:* `return {"count": 3, "items": [...]}`

### Defining the Input Schema

Every tool input is described by a JSON Schema the SDK generates from type hints.

- Use Python type annotations: `str`, `int`, `float`, `bool`, `list`, `dict`.
- Mark optional parameters with `Optional[T]` and provide defaults.
- Add `Field(description=...)` from pydantic to annotate each parameter.

The model reads the schema to know what arguments to pass. Clear descriptions reduce call errors.

### Three Common Tool Patterns

- **Database query** — accept filter parameters, run a query, and return the rows as a structured
  list. *Example:* `db.query(status="open")`
- **API wrapper** — accept request parameters, call an external HTTP API, and return the parsed
  response. *Example:* `requests.get(url, params)`
- **Files** — accept a path and operation type, read or write the file, and return content.
  *Example:* `open(path).read()`

### Returning Error Responses from Tools

To signal a tool failure, raise an exception. FastMCP converts it into an error result.

- `raise ToolError("...")` for a controlled, client-visible failure message.
- Any standard exception also works, becoming a result with `isError=True`.
- A returned dict becomes tool content, not an error — so raise to signal failure.
- The model receives the error result and can decide whether to retry or escalate.

### Using the MCP Inspector

The inspector lets you test your server without writing a client application.

- Run `npx @modelcontextprotocol/inspector` and point it at your server.
- Browse registered tools, view their schemas, and send test requests manually.
- Inspect raw JSON-RPC messages to debug schema and response formatting issues.

Test every tool through the inspector before integrating it with a real client or model.

### Tool Description Writing Rules

- **Be specific** — say exactly what the tool returns, such as open tickets filtered by owner.
  *Example:* "Returns open tickets by owner."
- **State side effects** — if the tool writes, deletes, or sends data, say so. This helps the
  model avoid unwanted changes. *Example:* "Permanently deletes the file."
- **List limits** — document rate limits, pagination caps, or size constraints. Models won't
  guess at them. *Example:* "Returns at most 100 rows."

### Key Takeaways: Exposing Tools Through Your MCP Server

Defining MCP tools means decorating Python functions and specifying typed schemas.

- Use `@mcp.tool()` to register functions with names and descriptions.
- Type-annotate parameters so the SDK generates JSON Schema.
- Return strings, dicts, or error dicts depending on the use case.
- Test every tool through the inspector before connecting a client.

---

## 16. Defining and Exposing Resources

MCP resources are URI-addressable data objects servers expose for hosts to inject into model
context.

### The MCP Resource Primitive

| Term | Definition | Example |
|---|---|---|
| **MCP resource** | A URI-addressable data object the application injects into context, chosen by the host application. | `file:///docs/readme.md` |
| **URI scheme** | The protocol prefix in a resource address that tells clients how to fetch it, such as file or database. | `db://`, `file://`, `config://` |
| **MIME type** | A format label, such as plain text or JSON, so clients know how to handle the content correctly. | `text/plain`, `application/json` |

### Anatomy of a Resource Definition

An MCP resource requires only a URI and a name; other fields are optional.

- **URI** — the stable address a client uses to request the resource.
- **Name** — a human-readable label identifying the resource.
- **MIME type** — optional, but helps the host interpret the content.
- A content handler on the server returns the data when a client reads it.

### Static vs. Dynamic Resources

- **Static resource** — content that does not change between requests. Config files, reference
  docs, and fixed lookup tables are good examples. They are safe to cache and do not require a
  live handler.
- **Dynamic resource** — content generated or fetched at request time. Database query results,
  live API responses, and user-specific data all change between calls and can't be cached without
  care.

### Designing URI Schemes for Resources

A well-designed URI scheme makes resources discoverable and self-describing.

- Use a domain prefix that reflects the kind of data: `db://`, `file://`, `config://`.
- Keep paths stable — changing URIs breaks clients that cached them.
- Avoid embedding volatile state like session tokens in the URI itself.

Consistent URI design lets host applications build reliable injection logic.

### Three Common Resource Types

- **File resources** — plain text or binary files. Use the `file://` scheme with a content type of
  `text/plain` or binary.
- **Database records** — query results returned as JSON. Use a `db://` scheme and declare type
  `application/json`.
- **Config snapshots** — key-value settings from a config store. Use `config://` and return
  structured text the model reads.

### Resource Subscriptions and Notifications

MCP supports subscriptions so clients learn when resource content changes.

- A client subscribes to a resource URI after initial retrieval.
- The server sends a notification when that resource's content changes.
- The client re-fetches the updated content when the notification arrives.

Subscriptions keep dynamic context fresh without polling on every request.

### Exposing Resources from a Server

Registering a resource on an MCP server follows a consistent pattern.

- Declare the resource in the server's resource list at startup.
- Implement a read handler that returns content for a given URI.
- Optionally implement a subscribe handler for change notifications.

The host calls `resources/list` to discover what's available and `resources/read` to fetch.

### What Application-Controlled Means

- **Host selects** — the host app picks which resources to inject.
- **Model reads** — the model reads injected content like any other context.
- **Exam trap** — resources are application-controlled, not model-invoked. The host chooses.

### Resource Templates for Dynamic URIs

One resource entry can't enumerate every file or record a server holds.

- A resource template exposes a parameterized URI, like `file:///{path}`.
- It uses the RFC 6570 URI-template syntax with named placeholders.
- Clients discover templates via `resources/templates/list`, separate from `resources/list`.
- The client fills the parameter, then calls `resources/read` on the concrete URI.

### Key Takeaways: Resources — URI, MIME Type, and Control

- MCP resources are URI-addressable data objects a server exposes for hosts to inject into
  context.
- Every resource needs a URI and name; MIME type is optional.
- Static resources cache safely; dynamic ones are generated at request time.
- Subscriptions deliver change notifications without polling.
- Resources are application-controlled — the host selects what the model sees.

Clear definitions keep injection logic predictable and reliable.

---

## 17. MCP Prompt Templates

MCP prompt templates are reusable, parameterized instructions stored on a server.

### The MCP Prompt Primitive

| Term | Definition | Example |
|---|---|---|
| **MCP prompt** | A reusable, parameterized template stored on a server and applied by the user. | `name="code_review"` |
| **Prompt argument** | A named, typed parameter the client fills in before the template is rendered. | `language: str = "python"` |
| **Rendering** | Filling argument values into the template server-side to return a finished prompt. | "Review this {language}..." |

### Anatomy of a Prompt Template Definition

An MCP prompt template has three required components.

- **Name** — a stable identifier clients use to request the template.
- **Arguments** — named parameters with types and optional defaults.
- **Template body** — the prompt text with placeholders for each argument.

The server renders the final prompt by substituting argument values before returning it.

### Prompts vs. Resources vs. Tools

- **Prompts (user)** — selected explicitly by the user from a surfaced list to start a session.
  The human initiates it.
- **Resources (app)** — injected by the host app into context. The model reads them but did not
  choose or request them.
- **Tools (model)** — invoked by the model when it decides a capability is needed. The model
  initiates the call.

### Use Cases for Server-Side Prompt Templates

Server-side prompts shine wherever task instructions are complex or need to be consistent.

- **Multi-step analysis** — structured prompts that enforce a repeatable methodology.
- **Compliance workflows** — approved instruction sets that can't be rewritten ad hoc.
- **Onboarding tasks** — guided prompts that walk users through unfamiliar processes.

Templates reduce variability and keep complex instructions out of each user's hands.

### Three Prompt Versioning Strategies

- **Version in name** — append a version number to the prompt name. Simple, but each version is a
  new entry. *Example:* `report_v2`
- **Version as argument** — pass version as a runtime argument and resolve the right body
  server-side. *Example:* `version: str`
- **Deprecation flag** — mark old templates deprecated in the response so clients can warn users.
  *Example:* `deprecated`

### How Clients Access Prompt Templates

The client discovery flow for prompts mirrors the pattern for resources.

1. Call `prompts/list` to retrieve all available templates and their argument schemas.
2. Collect argument values from the user through the host UI.
3. Call `prompts/get` with the name and filled argument values.
4. Inject the rendered prompt into the conversation as the opening context.

The server renders the prompt; the client collects arguments and injects the result.

### Designing Arguments for Reusability

Prompt arguments determine how flexible and reusable a template actually is.

- Keep required arguments minimal — each one adds friction for users.
- Provide sensible defaults for optional arguments to lower the entry barrier.
- Use typed arguments so clients can validate input before calling prompts.

Well-designed arguments make templates widely applicable without becoming vague.

### Standardizing Complex Task Instructions

- Every user who selects the same template gets the same instruction set, reducing output
  variance.
- Update the template server-side once and all clients automatically receive the new version.
- Server-side templates can be reviewed, approved, and versioned for compliance-sensitive
  workflows.

### Key Takeaways: MCP Prompts — User-Chosen Templates

MCP prompt templates are parameterized instructions users select to pre-populate an interaction.

- Templates have a name, typed arguments, and a server-rendered body.
- Prompts are user-controlled: the human applies them, not the model.
- Clients discover via `prompts/list` and render via `prompts/get`.
- Server-side templates keep instructions consistent and support versioning.

Templates bring consistency to complex MCP workflows.

---

## 18. Implementing MCP Clients

MCP clients are the software layer that connects AI applications to tool servers.

### Three Core MCP Client Terms

| Term | Definition |
|---|---|
| **MCP client** | Software, like Claude Desktop or an agent framework, that connects to servers and routes model requests. |
| **STDIO transport** | Spawns the server as a child process, communicating over standard input and output. For local, single-user use. |
| **Streamable HTTP** | Runs the server as an HTTP service with streaming support. For remote, multi-user, production deployments. |

### MCP Client: What It Does

The MCP client sits between your model and the tool servers.

- Manages connection lifecycle: open, keep alive, close.
- Sends capability-discovery requests on startup.
- Routes tool calls to the correct server at runtime.

The client is the mediator — the model never talks to servers directly.

### Client Initialization in Python

The MCP Python SDK provides a high-level client class for both transport types.

- Import `StdioServerParameters` or a URL-based config.
- Instantiate the client with the transport config.
- Call `initialize()` to complete the handshake.

Initialization exchanges protocol version and capability lists before any tool is invoked.

### Connection Lifecycle Phases

- **Connect** — the client opens transport and sends an `initialize` request with its supported
  protocol version.
- **Negotiate** — the server replies with its version and capability list; the client confirms
  with an `initialized` notification.
- **Close** — no protocol shutdown message; termination happens at the transport layer by closing
  the connection.

### Discovering Server Capabilities

After initialization, the client queries what the server exposes.

- `list_tools()` returns tool names, schemas, and descriptions.
- `list_resources()` returns available data resources.
- `list_prompts()` returns reusable prompt templates.

Discovery happens once per session; results are cached for routing decisions.

### Keeping the Catalog Fresh

Discovery runs once, but a server's tools and resources can change mid-session.

- A server declares a `listChanged` capability during initialization.
- It then sends a notification when its catalog changes.
- The client re-runs `list_tools()` or `list_resources()` to refresh its cache.

No capability declared means no notifications, so the cache can quietly go stale.

### STDIO vs. Streamable HTTP

- **STDIO transport** — spawns the server as a child subprocess on the local machine. No network
  stack required. Ideal for developer workstations, local tools, and single-user scripts.
- **Streamable HTTP** — runs the server as a remote HTTP service with optional streaming.
  Designed for shared infrastructure, multiple concurrent clients, and production deployments.

### Invoking Tools and Handling Responses

Once connected, the client calls tools by name with a JSON-serializable argument dict.

- `call_tool(name, arguments)` returns a structured result.
- Tool-execution failures return `isError` true — check it before forwarding.
- Protocol errors raise `McpError` and need a try/except block.
- Handle both paths: the `isError` flag for tool failures, exceptions for protocol errors.

### Multi-Server Clients and Routing

A single client can hold connections to multiple MCP servers simultaneously.

- Each server is registered with a unique name and transport config.
- The client routes tool calls by matching tool names to the owning server.
- Namespace prefixes prevent collisions when servers share tool names.

Multi-server routing is how agents compose large toolsets from smaller servers.

### Key Takeaways: MCP Clients — Connect, Discover, Invoke

MCP clients manage the full lifecycle of server connections and tool routing.

- STDIO transport is for local, single-user deployments.
- Streamable HTTP transport handles remote and production workloads.
- `list_tools()`, `list_resources()`, `list_prompts()` expose server capabilities.
- `call_tool()` invokes tools and returns structured content blocks.

One client can orchestrate entire toolsets across many servers.

---

## 19. STDIO Transport: Local Servers

STDIO transport spawns a server as a subprocess and routes communication through standard input
and output. The client is the parent process; the server is the child.

### Transport Mechanism Vocabulary

| Term | Definition | Example |
|---|---|---|
| **STDIO transport** | An MCP transport where the client spawns the server as a child process and talks over stdin and stdout. | `stdio_client(params)` |
| **Child process** | A process spawned by a parent. In STDIO transport, the client is the parent and the server is the child. | `subprocess.Popen(cmd)` |
| **Standard streams** | Default I/O channels every process gets from the OS. | `stdin` / `stdout` / `stderr` |

### How STDIO Transport Works

STDIO transport connects client and server through the operating system's process model.

- The client spawns the server binary as a child process at startup.
- All MCP messages flow over stdin and stdout as newline-delimited JSON.
- The server lives only as long as the client process runs.

STDIO is designed for trust relationships where client and server share one host.

### STDIO vs. Network Transport

- **STDIO transport** — the server is a local subprocess. Communication uses OS-level pipes,
  stdin and stdout. No network stack, no port binding. Fast, zero-configuration, and inherently
  single-user. *Example: a Claude Desktop local server.*
- **Network transport** — the server binds to a network port. Communication uses HTTP or sockets.
  Accessible from remote hosts. Requires authentication, TLS, and network configuration.
  *Example: a remote HTTP MCP endpoint.*

### When to Use STDIO Transport

STDIO is the right transport when server and client share a machine and a trust boundary.

- Local development and testing before deploying remotely.
- Desktop tool integrations where the server is bundled with the app.
- Claude Code integration — STDIO is the native transport for its MCP servers.

Anything needing a remote connection or multiple simultaneous users rules STDIO out.

### STDIO Deployment Characteristics

- **Zero config** — no port binding, no TLS setup, no firewall rules. Client and server connect
  through the OS process table.
- **Single-user** — one server process per client instance. STDIO is inherently private to the
  spawning process.
- **Scoped lifetime** — the server starts when the client spawns it and exits when the client
  disconnects.

### Stateful by Design

Because STDIO ties one server process to one client session, state is inherently per-process.

- In-memory state persists across all tool calls within a session.
- No serialization needed between calls — the process stays alive.
- The session ends when the process exits, so no stale state leaks across users.

This stateful model simplifies local tools but makes STDIO unsuitable for shared deployments.

### Claude Code and STDIO

Claude Code's MCP integration runs on STDIO, the reference model for local server deployments.

- Each server is listed in Claude Code's settings with a command to run.
- Claude Code spawns the server as a child process at session start.
- All tool calls route through the stdin/stdout pipe for that process.

Adding a new server requires only a binary path — no network setup.

### STDIO Limitations to Know

- **No remote access** — STDIO servers can't be reached from any other host.
- **No multi-user sharing** — each session spawns its own process; there is no shared STDIO
  server.
- **Not for production APIs** — STDIO is a local IPC mechanism, not an HTTP service for
  production.

### Key Takeaways: STDIO Transport

STDIO transport is the right choice for local, single-user MCP server deployments.

- The client spawns the server as a child process via stdin and stdout.
- Zero network configuration — OS pipes handle all communication.
- Stateful per-process — in-memory state persists across a session.
- Claude Code uses STDIO as its native MCP transport.

Remote or multi-user use needs a different transport.

---

## 20. StreamableHTTP: Remote MCP Servers

StreamableHTTP is the MCP transport built to serve remote clients and multiple users.

### Three Transport Terms

| Term | Definition |
|---|---|
| **StreamableHTTP** | An MCP transport running the server as an HTTP service. For remote, multi-user, production deployments. |
| **Server-Sent Events (SSE)** | A one-way HTTP streaming protocol where the server pushes events over a long-lived connection. |
| **Stateless session** | An HTTP model where each request is self-contained, carrying all the data it needs. |

### How StreamableHTTP Works

StreamableHTTP turns an MCP server into an HTTP service accessible over a network.

- The server binds to a port and listens for HTTP requests from clients.
- Standard tool calls use a request-response pattern over HTTP POST.
- Streaming responses open a long-lived SSE connection for event delivery.

Any client that can make HTTP requests can reach the server, across hosts and processes.

### Server-Sent Events for Streaming

SSE is the streaming mechanism built into StreamableHTTP transport.

- The client opens an HTTP connection that stays open for the duration of the stream.
- The server pushes individual events as they become available — no polling required.
- Each event is a text frame formatted per the SSE specification.

SSE is unidirectional: server to client only, ideal for streaming progress during long operations.

### Stateless vs. Session-Based State

- **Stateless HTTP** — each request carries all the data it needs. The server keeps no per-client
  state, so any server can handle it. *Example: each call re-sends full context.*
- **Session-based state** — the client sends a session token in each request and the server maps
  it to stored data. This supports multi-turn memory across calls. *Example: session token per
  request.*

### When StreamableHTTP Is the Right Choice

- **Remote clients** — client and server on different hosts need an HTTP transport. STDIO has no
  network reach across hosts.
- **Multi-user deployments** — one StreamableHTTP server instance can serve many users at once,
  not one process each.
- **Production APIs** — HTTP brings TLS, authentication, load balancing, and rate limiting: the
  full production stack.

### SSE Is Not a Separate Transport

SSE is a streaming component used inside StreamableHTTP, not a standalone MCP transport.

- Older MCP specs listed SSE as its own transport type.
- Current MCP consolidates streaming under StreamableHTTP.
- SSE is the delivery mechanism for events within StreamableHTTP.

Any separate SSE transport has been absorbed into the StreamableHTTP specification.

### Production Deployment Requirements

StreamableHTTP brings MCP servers into the standard HTTP service model.

- TLS is required to encrypt communication over untrusted networks.
- Authentication tokens or API keys prevent unauthorized tool access.
- Load balancers can spread requests across many server instances.

These don't exist for STDIO because the pipe is local; StreamableHTTP takes on network duties.

### Transport Selection Summary

- **Use STDIO** — client and server share a machine and it's single-user. Local dev tools,
  desktop apps.
- **Use StreamableHTTP** — the server needs network reach or serves multiple users. Team APIs,
  cloud-deployed MCP servers.

If it crosses a network boundary or serves more than one user, StreamableHTTP is the answer.

### Key Takeaways: StreamableHTTP — Transport for Production

- Runs as an HTTP service accessible from any client on the network.
- Combines request-response and SSE streaming in one transport.
- Supports both stateless and session-based state models.
- SSE is a streaming component of StreamableHTTP, not a separate transport.

Transport choice comes down to one question: is this local and single-user, or not?

---

## 21. Sampling: Server-Initiated LLM Calls

Sampling lets a server ask the client to make an LLM inference call on its behalf.

### Sampling and Related Concepts

| Term | Definition |
|---|---|
| **Sampling** | An MCP feature where a server asks the client to make an LLM call and return the result. |
| **Sampling handler** | A callback in the client that receives sampling requests, calls the LLM, and returns the response. |
| **Human-in-the-loop** | An optional gate where the client pauses for user approval before forwarding a request to the LLM. |

### Why Sampling Exists

Servers need intelligence but shouldn't need their own API credentials.

- A tool server can be stateless — no keys, no model dependencies.
- The client already has model access, credentials, and policy controls.
- Sampling reuses that existing relationship instead of duplicating it.

Routing model calls through the client keeps control centralized and auditable.

### The Sampling Flow Step by Step

1. **Request** — the server sends a `createMessage` request to the client over MCP.
2. **Approve** — the client optionally shows the request to the user for approval.
3. **Call LLM** — the client calls the LLM with the provided messages and parameters
   (`client.messages.create()`).
4. **Return** — the client packages the response as a standard MCP result back to the server
   (`CreateMessageResult`).

### What the Server Controls vs. the Client

- **Server controls** — the message content, system prompt, and model preferences it sends in the
  `createMessage` request.
- **Client controls** — which actual model is used, whether credentials are shared, and whether
  human approval is required.
- **Shared result** — the model response flows back through the client to the server as a
  structured MCP content block.

### Implementing a Sampling Handler

The MCP Python SDK lets you register a sampling callback on the client.

- Define an async function that accepts a `CreateMessageRequest`.
- Call your LLM API inside the handler with the provided messages.
- Return a `CreateMessageResult` with the model's text response.

The handler is called automatically whenever a connected server sends a sampling request.

### Sampling vs. Direct API Access

- **Sampling through client** — the server sends a `createMessage` request. The client calls the
  LLM using its own credentials and policy rules. Model choice and data controls stay with the
  client owner.
- **Direct API access** — the server holds its own API key and calls the LLM directly.
  Credentials multiply across services. The client loses visibility and control over what the
  server is requesting.

### Model Preferences in Sampling Requests

Servers can express preferences without dictating the exact model used.

- **Intelligence hint** — low, medium, or high capability needed.
- **Speed hint** — how quickly the response is needed.
- **Model name suggestions** — listed in priority order.

The client is free to honor, substitute, or override these hints.

### Human-in-the-Loop Sampling

Sampling supports an optional approval gate before the LLM is called.

- The client surfaces the pending request and message content to the user.
- The user can approve, modify, or reject the sampling request.
- Only approved requests reach the LLM and return results to the server.

This pattern is especially important when servers request access to sensitive data.

### Key Takeaways: Sampling — Server Intelligence via Client

Sampling routes LLM calls through the client, keeping credentials and policy unified.

- Servers send `createMessage`; clients call the model and return results.
- The client keeps control of model selection, credentials, and policies.
- Model preference hints guide the client without overriding authority.
- Human-in-the-loop gates let users approve sampling requests.

Sampling enables server-side intelligence with client control.

---

## 22. Notifications, Roots, and Progress

Long-running tools need a way to keep clients informed without blocking on the result.

### Three Terms to Know

| Term | Definition |
|---|---|
| **Progress notification** | An MCP notification sent during a long-running tool call to report incremental status. Keeps the client informed without blocking the final result. |
| **Roots** | A list of filesystem paths or URIs the client declares to a server, advising the scope it should work within. Advisory, not enforced. |
| **Log notification** | An MCP notification carrying a severity level and message string for server-to-client diagnostic logging during execution. |

### Why Notifications Matter

Tool calls are synchronous — the client waits for a result.

- For fast tools, that's fine — results come back in milliseconds.
- For slow tools, clients go silent with no sign of progress.
- Notifications break that silence with real-time status updates.

Progress and log notifications are the MCP equivalent of a progress bar.

### Log Notifications in Practice

Log notifications let servers stream diagnostic messages during execution.

- Severity levels: debug, info, notice, warning, error, critical, alert, emergency.
- Each notification carries a logger name and a message string.
- Clients can filter or display logs based on a configured severity threshold.

Log notifications give you structured observability into tool behavior without blocking execution.

### Progress Notification Fields

- **`progressToken`** — an opaque token from the original tool call, used to match progress
  updates to their parent operation.
- **`progress`** — a numeric value showing how far along the operation is, relative to the total
  field if provided.
- **`total`** — the optional target value for progress. When set, clients can compute a percentage
  to show the user.

### Sending Progress From a Tool

Sending progress notifications from a server tool is a small amount of extra code.

- Accept the `progressToken` from the incoming tool call meta field.
- Call `report_progress(progress, total)` at meaningful checkpoints.
- The MCP framework serializes and sends the notification to the client.

Clients that don't support progress notifications simply ignore them safely.

### Log vs. Progress Notifications

- **Log notifications** — carry text messages with severity levels. Designed for diagnostics and
  debugging. Clients may display, filter, or store them. *Example:* `level=error, "disk full"`
- **Progress notifications** — carry numeric progress and total values tied to a specific
  operation token. Designed for UI progress bars and user feedback. *Example:* `progress=3,
  total=10`

### Roots: Advisory Scope Communication

Roots tell a server which filesystem paths it should work within.

- The client declares the roots capability, then sends a list of roots.
- Each root is a URI pointing to a directory or file the server should use.
- Well-behaved servers should respect roots, but roots don't enforce access.
- Real enforcement comes from OS permissions or sandboxing, not roots alone.

### Roots Security Model

Roots communicate scope; they don't grant or enforce access on their own.

- The client declares the roots capability during initialization.
- The server requests the paths via a roots list call and checks that capability.
- A well-behaved server should stay within them, but nothing forces it to.
- Back roots with OS permissions, sandboxing, or client-side validation for real enforcement.

### Key Takeaways: Notifications and Roots

MCP notifications and roots add status reporting and scoped filesystem advice.

- Log notifications stream diagnostic messages with severity levels.
- Progress notifications report numeric progress tied to an operation token.
- Roots are advisory scope the client declares; servers should respect them.
- Real enforcement comes from OS permissions or sandboxing, not roots.

Sampling, notifications, and roots form the advanced MCP feature layer.

---

## 23. Tool Boundaries and Least Privilege

Every MCP tool definition is also a trust boundary.

### Three Security Concepts for MCP

| Term | Definition | Example |
|---|---|---|
| **Access scoping** | Defining explicit boundaries on what data and operations each MCP server may reach. | DB read-only, no writes. |
| **Least privilege** | Each tool gets only the permissions its function requires, minimizing blast radius if misused. | A search tool cannot delete. |
| **Input sanitization** | Validating and cleaning arguments at the server boundary before they reach any downstream system. | Reject SQL metacharacters. |

### Why Tool Definitions Are Trust Boundaries

An MCP tool definition isn't just a schema — it's a declared contract for what the tool can do.

- The schema sets the expected argument types and structure. *Example:* `query: str, limit: int`
- The server-side handler determines what systems that tool can reach. *Example: the handler hits
  one database.*
- A poorly scoped tool extends the model's effective permissions too far. *Example:*
  `run_sql(any_query)`

Treat every tool as a trust boundary and reason about its access during design.

### Applying Least Privilege to MCP Tools

Each tool should have only the access its function requires.

- A read-only search tool should never have write permissions.
- A tool scoped to one database should not reach others.
- Separate tools for separate resource types reduce blast radius.

If a tool is compromised or misused, narrow scope limits what an attacker can reach.

### Three Scoping Mechanisms in MCP

- **Schema constraints** — strict argument types and enum values let the server reject
  out-of-range inputs before any handler runs. *Example:* `status: enum[open, closed]`
- **Roots grants** — MCP roots restrict which filesystem paths or URI prefixes a tool may access
  at the protocol level. *Example:* `file:///app/data`
- **Server-side auth** — authorization checks in the handler verify caller identity and
  permissions before touching a resource. *Example:* `current_user.can_read(id)`

### Broad vs. Narrow Tool Scope

- **Broad scope (risky)** — a single tool that accepts any SQL query and connects to any database
  gives the model — and any attacker who influences it — unrestricted data access. *Example:*
  `run_sql(query)`
- **Narrow scope (safe)** — separate tools per resource type, with schema constraints limiting
  query shape and server-side validation enforcing per-user permissions, keep the blast radius
  small. *Example:* `get_order(id: int)`

### Input Sanitization at the Server Boundary

Every argument entering an MCP server should be treated as untrusted input.

- Validate types and formats against the declared schema immediately.
- Reject or escape characters that have special meaning downstream.
- Enforce length limits to prevent resource exhaustion.

Server-side sanitization is the last line of defense before data reaches production systems.

### Designing Tools for Least Privilege

Least privilege isn't just a configuration step — it starts during tool design.

- Define one tool per logical operation, not one tool per domain.
- Use enum constraints to allow only specific valid values.
- Expose results filtered to what the caller is allowed to see.

Tools designed with narrow scope are easier to audit and safer by default.

### Common Scoping Mistakes to Avoid

- **Permissive schemas** — free-form string args let the model build arbitrary queries. *Example:*
  `arg: str` (free text).
- **Shared credentials** — one shared credential set means one breach leaks everything. *Example:
  one admin key for all.*
- **No output filtering** — returning raw results can expose records the user shouldn't see.
  *Example:* `SELECT *` returned raw.

### Key Takeaways: Least Privilege and Access Scoping for MCP

Each MCP tool should access only what its specific function requires.

- Schema constraints, roots grants, and server-side auth enforce access limits.
- Sanitize inputs at the server boundary before any handler runs.
- Narrow scope limits blast radius if a tool is misused or exploited.
- Build access limits in during design — retrofitting is costly.

Treat every tool definition as a trust boundary.

---

## 24. MCP in Production: Auth, Versions, Ops

Deploying an MCP server to production introduces requirements that don't exist in development.

### MCP Production: Three Core Terms

| Term | Definition |
|---|---|
| **MCP authentication** | How an MCP client proves its identity to a server. MCP recommends an optional OAuth 2.1 flow for remote StreamableHTTP servers. |
| **Semantic versioning** | Version numbering with major, minor, and patch numbers. A major bump signals breaking changes; minor adds backward-compatible features; patch fixes bugs. |
| **Rate limiting** | Capping requests per client per time window to protect shared MCP infrastructure from overload and abuse. |

### Why Production MCP Needs Authentication

A local server trusts the local process. Remote servers can't make that assumption.

- Any client that can reach the endpoint could invoke its tools.
- Sensitive tools require verified caller identity before executing.
- Audit logs need a stable identity to be meaningful.

Authentication is the prerequisite for authorization, logging, and rate limiting.

### OAuth 2.1 for StreamableHTTP Servers

MCP defines an optional OAuth 2.1 authorization flow as the recommended mechanism for remote
StreamableHTTP servers.

- The client redirects through an OAuth authorization flow to obtain a token.
- The token is presented with each request and validated server-side.
- Scopes in the token can map directly to which tools the client may invoke.

OAuth separates identity from the MCP protocol itself, keeping the server stateless.

### Three Authentication Patterns Compared

- **OAuth 2.1** — MCP's recommended optional mechanism for StreamableHTTP servers. Token-based,
  supports scopes, integrates with identity providers.
- **API keys** — simple shared secrets in request headers. Easy to implement but lack per-user
  identity and scope.
- **Mutual TLS** — authenticates both parties via certificates. Strong for service-to-service but
  adds certificate management overhead.

### Streamable HTTP Transport Security

The spec makes three transport-level rules for Streamable HTTP servers.

- Validate the `Origin` header on every connection to block DNS rebinding.
- When local, bind only to `127.0.0.1`, never to all interfaces.
- Servers should authenticate every connection.

Skip these and a remote web page can drive your local server.

### Versioning Without Breaking Clients

- **Backward-compatible** — adding new optional tools, adding optional fields to existing
  schemas, or fixing bugs in handler logic. Clients on the current version keep working without
  changes.
- **Breaking changes** — removing or renaming tools, or changing required argument names or
  types. These need a major version bump and a migration window before the old version is retired.

### Rate Limiting and Resource Protection

Shared MCP servers serving multiple clients need explicit rate limits to stay healthy.

- Cap requests per client per time window so no one client monopolizes resources.
- Apply per-tool limits for expensive operations like queries or external API calls.
- Return HTTP 429 responses with a `Retry-After` header when limits are hit.

Rate limiting protects all clients equally and makes infrastructure costs predictable.

### Monitoring a Production MCP Server

A production MCP server needs the same observability as any other API service.

- Log every tool invocation with caller identity, argument summary, and latency.
  *How:* `logger.info(tool, user, args, ms)`
- Track error rates by tool to detect degraded downstream integrations.
  *How:* count errors per tool on failure.
- Alert when error rates or latency breach defined thresholds.
  *How:* alert if `error_rate > 5%` or `p95 > 2s`.

Visibility into tool usage patterns also reveals misuse and unexpected client behavior.

### Three Reliability Practices for MCP

- **Health endpoints** — a dedicated health-check path lets load balancers verify the server is
  alive without invoking real tools.
- **Graceful degradation** — return structured error results when a dependency is down so clients
  can retry or fall back cleanly.
- **Timeout enforcement** — hard timeouts on handler execution stop a slow dependency from
  exhausting server connection resources.

### Key Takeaways: MCP Production — Auth, Versions, Ops

Deploying MCP to production requires authentication, versioning, and observability.

- OAuth 2.1 is MCP's recommended mechanism for StreamableHTTP auth.
- Semantic versioning with a migration window protects existing clients.
- Rate limiting keeps shared servers stable under load.
- Logging, alerts, and health endpoints provide visibility.

A production MCP server is an API service — operate it accordingly.

---

## 25. The Built-in Toolset

Claude ships with first-party tools that architects can enable without writing a single schema.

### Three Foundational Terms

| Term | Definition |
|---|---|
| **Built-in tool** | A tool natively supported by the Claude API, exposed through a typed parameter rather than a user-defined tool schema. |
| **`web_search`** | Built-in tool that runs a constrained web query, returns ranked snippets with source citations, and is subject to rate limits. |
| **`computer_use`** | Anthropic-schema client tool giving Claude screenshot, mouse, and keyboard primitives in a desktop environment your app supplies. |

### The Built-in Catalog

Anthropic ships a catalog of built-in tools, split into server tools and Anthropic-schema client
tools.

- `web_search` and `code_execution` — server tools run on Anthropic infra, no handler needed.
- `computer_use`, `bash`, `text_editor` — client tools you execute in your environment.
- All skip schema authoring, but client tools still need your handler code.

The split matters because server and client tools sit on opposite sides of the boundary.

### Built-in vs. Custom Tools

- **Built-in tools** — defined by Anthropic. Zero schema work. Vetted prompts and optimized
  routing included. Ideal when the capability fits the task as-is.
- **Custom tools** — defined by the developer via the tool-use schema, backed by an MCP server or
  function. Needed for domain behavior or internal data.

### `web_search` — Behavior and Limits

`web_search` constructs queries automatically, retrieves ranked snippets, and attaches source
citations to every result.

- Query construction is handled internally by the model.
- Search count per request is capped by the optional `max_uses` parameter.
- Citations are returned as structured metadata, not inline links.
- Rate limits surface as an error object inside a successful response, not as fewer results.

### `computer_use` Scope

- **Screenshots** — capture the current state of any visible screen in the environment you
  supply. Read-only observation of UI state.
- **Mouse control** — click, drag, and scroll in that environment. Reach is bounded by what your
  app connects, not an Anthropic sandbox.
- **Keyboard input** — type text and send keystrokes to the desktop session your app runs and
  controls.

### `code_execution` — Runtime Details

`code_execution` runs code in a persistent container that lives up to 30 days and is reused via
container ID.

- Ships a fixed pre-installed library set: pandas, numpy, matplotlib.
- Reuse the container ID across requests to keep files and state.
- Newer tool versions add REPL state persistence, so state can carry across Claude calls.
- No internet access — outbound network requests are fully disabled.

### File Ops and Capability Gap

- **`text_editor`** — Anthropic-schema client tool for file operations (view, create,
  `str_replace`) that your code executes. There is no built-in glob or grep file tool in the API.
- **`bash`** — client tool that runs shell commands in your environment. Pair it with
  `text_editor` for file discovery and edits Claude requests.
- **Capability gap** — the difference between what a built-in tool can do and what a workflow
  actually requires; drives the build-versus-buy decision.

### Cost and Latency Profile

Built-in tools carry a different cost and latency profile than custom tooling.

- `web_search` adds round-trip latency for each query.
- `computer_use` is computationally expensive per screenshot cycle.
- `code_execution` container spin-up adds cold-start overhead.

Custom tools backed by fast internal services can outperform built-ins on latency.

### Key Takeaways: Built-in Toolset Foundations

- Built-ins split into server tools and Anthropic-schema client tools.
- `web_search` returns cited snippets, capped by `max_uses`.
- `computer_use` runs screenshot, mouse, and keyboard as a client tool.
- `code_execution` uses a container reusable for up to 30 days.
- `text_editor` and `bash` are client tools you execute in your environment.

---

## 26. Built-in vs. Custom: The Decision

Not every workflow fits a built-in tool, and not every workflow needs a custom one.

### Three Decision Terms

| Term | Definition |
|---|---|
| **Capability gap** | The difference between what a built-in tool can do and what a workflow actually requires; drives the build-versus-buy decision. |
| **Hybrid pattern** | An architecture where built-in tools handle general capability while custom MCP servers supply proprietary or regulated data. |
| **Naming best practice** | Give custom tools distinct, service-prefixed names so selection stays unambiguous. Duplicate or invalid tool definitions are generally rejected when the request is made. |

### When Built-in Is the Right Call

Built-in tools are the right starting point when the capability fits and the data is public.

- **Quick prototyping** — no schema work means faster iteration.
- **Well-fitted scope** — the tool does exactly what the workflow needs.
- **No internal or regulated data** flowing through the tool.

If all three hold, reach for the built-in first and validate before going custom.

### When Custom Is Required

Four conditions signal that a custom tool is the right architecture choice.

- Domain-specific schemas that built-ins cannot express.
- Internal APIs that live behind your private network boundary.
- Regulated data that should not leave your controlled environment.
- Audit trail requirements that built-ins cannot expose.

When any of these apply, a custom MCP server gives you the control built-ins cannot.

### Custom Tool Requirements

- **Domain schemas** — define the fields, types, and validation your domain needs, beyond a fixed
  built-in schema. *Example:* `invoice: id, total, status`
- **Internal APIs** — reach private services behind your firewall that sandboxed built-ins cannot
  access. *Example:* `GET /internal/customers`
- **Audit trail** — log every call and response to your audit system for full traceability.
  *Example:* `log(user, action, time)`
- **Regulated data** — route regulated data through servers you control, enforcing data-handling
  policies end to end. *Example: HIPAA patient records.*

### Hybrid Pattern — Structure

- **Built-in layer** — general capability: `web_search` for public research, `code_execution` for
  data transformation. Fast to enable, no schema work, maintained by Anthropic.
- **Custom MCP layer** — proprietary access: internal databases, regulated records, domain APIs.
  Fully controlled by the architect, with audit logging and access policies.

### Security Boundaries — Server vs. Client

Server tools run on Anthropic infra and cannot reach your private network or internal services.

- `web_search` and `code_execution` stay inside Anthropic's boundary.
- Client tools like `bash`, `text_editor`, and `computer` run in your environment.
- Client tools can and do reach private resources you expose to them.

Match the tool to the trust boundary. Server tools stay outside, client tools run inside.

### Naming — A Best Practice

Use distinct, service-prefixed names so tool selection stays unambiguous as your toolset grows.

- Prefix by service, like `crm_lookup`, to signal ownership at a glance.
- Distinct names keep selection clear even when functionality overlaps a built-in.
- Duplicate or invalid tool definitions are generally rejected when the request is made.

Treat naming as a design decision that keeps routing predictable from the start.

### Versioning Built-in Tools

Don't assume stable behavior just because the tool name stays constant.

- Anthropic can update query construction, result formats, and limits.
- Pinning behavior through prompt instructions beats tool-name versioning.
- Test built-in output in staging after each model update.

Treat built-in tools as versioned dependencies, not as fixed contracts.

### Key Takeaways: The Build-vs-Buy Decision

Tool choice depends on capability gap, data sensitivity, latency, and audit needs.

- Built-in first when scope fits and data is public.
- Custom required for internal APIs or regulated data.
- Hybrid combines built-in capability with custom MCP for proprietary data.
- Never reuse a built-in tool name for custom tools.

Pin built-in behavior through prompts, not tool names.
