# Context Management — Day 5

*Text extracted from the 251-slide deck and grouped by topic. Diagrams and decorative
elements are omitted; only the substantive text is kept.*

---

## Table of Contents

1. [How the Context Window Works](#1-how-the-context-window-works)
2. [Context Window Economics](#2-context-window-economics)
3. [Why Summaries Lose Facts](#3-why-summaries-lose-facts)
4. [Designing a Persistent Fact Block](#4-designing-a-persistent-fact-block)
5. [The Memory Tool for Persistence](#5-the-memory-tool-for-persistence)
6. [The Tool-Output Bloat Problem](#6-the-tool-output-bloat-problem)
7. [Trim Strategies: Filter, Summarize, Reference](#7-trim-strategies-filter-summarize-reference)
8. [When to Escalate: Trigger Design](#8-when-to-escalate-trigger-design)
9. [The Handoff Payload](#9-the-handoff-payload)
10. [Where to Catch the Error](#10-where-to-catch-the-error)
11. [Designing the Propagation Contract](#11-designing-the-propagation-contract)
12. [When "Nothing" Is Ambiguous](#12-when-nothing-is-ambiguous)
13. [Disambiguation Patterns](#13-disambiguation-patterns)
14. [The Scratchpad Pattern](#14-the-scratchpad-pattern)
15. [Scratchpad Hygiene and Limits](#15-scratchpad-hygiene-and-limits)
16. [Why Delegate Discovery](#16-why-delegate-discovery)
17. [Delegation Mechanics](#17-delegation-mechanics)
18. [Long-Conversation Coherence](#18-long-conversation-coherence)
19. [Server-Side Compaction](#19-server-side-compaction)
20. [Per-Field Confidence Scores](#20-per-field-confidence-scores)
21. [Calibrating Confidence Scores](#21-calibrating-confidence-scores)
22. [Stratified Sampling for Review](#22-stratified-sampling-for-review)
23. [Routing Through a Review Queue](#23-routing-through-a-review-queue)
24. [Every Claim Needs a Source](#24-every-claim-needs-a-source)
25. [Conflict Annotation in Synthesis](#25-conflict-annotation-in-synthesis)

---

## 1. How the Context Window Works

The context window bounds every Claude inference call. It holds the system prompt, history,
documents, and output.

### Two Foundational Concepts

| Term | Definition |
|---|---|
| **Context window** | Everything a model can see in one inference call: prompt, history, documents, and the model's output. |
| **Token** | The atomic unit a model processes, roughly four characters or three-quarters of an English word. |
| **Context window limit** | The hard maximum token count for one request, combining all input and output. Exceeding it fails. |

### What Counts Against the Context Window

Every token in the request adds to the total, well beyond the user's message.

- **System prompt** — sometimes hundreds or thousands of tokens.
- **Conversation history** — the full transcript of multi-turn sessions.
- **Injected data** — retrieved documents, tool schemas, and context.
- **Model output** — the response tokens count too.

In production the visible prompt is often a small fraction of total usage.

### Input Tokens vs. Output Tokens

- **Input tokens** — everything passed to the model: system prompt, history, documents, tool
  definitions, and the user message. Usually the larger share in RAG and agent pipelines.
- **Output tokens** — the text the model generates. Output counts against the limit and usually
  costs more per token than input. Long reasoning consumes output budget quickly.

### How Different Content Types Tokenize

The same information can use very different token counts depending on format.

- **Plain prose** — roughly one token per four characters.
- **Code** — tokenizes efficiently because keywords are common.
- **JSON and XML** — structural punctuation adds overhead.
- **Non-English text** — many languages use more tokens per word.

Format choices directly affect cost and whether content fits the window.

### Choosing a Claude Model Tier

- **Claude Haiku** — the smaller-window, low-latency tier at 200K tokens. Fits fast, lightweight
  single-turn summarization and classification.
- **Claude Sonnet** — a large 1M-token window at lower cost. Balances capability and price for
  multi-document retrieval and agentic pipelines.
- **Claude Opus** — the same 1M-token window with top capability. Chosen for the hardest
  reasoning, not a bigger window than Sonnet.

### What Happens When the Window Fills

Overflow shows up in two distinct places, and the API signals them differently.

- **Input too large** — a 400 error rejects the request before any generation.
- **Mid-generation overflow** — output stops with `model_context_window_exceeded`.
- **Hard truncation** — dropping oldest messages loses critical context.

Handling code has to treat an up-front rejection and a mid-stream stop as separate cases.

### Context Budget and Context Rot

| Term | Definition |
|---|---|
| **Context budget** | A deliberate token allocation across prompt, history, retrieved chunks, and output headroom, enforced in code. |
| **Context rot** | Anthropic's term for attention diluting as tokens grow, so recall and instruction-following degrade. |
| **Lost in the middle** | A position-based framing where content in the middle of a long context gets less attention than the start or end. |

### Working Within Context Limits

Well-designed systems work within context limits rather than against them.

- **Summarize history** — compress long conversations instead of carrying them verbatim.
- **Retrieve selectively** — chunk and pull only the most relevant documents.
- **Cap output** — set explicit output token limits to preserve input headroom.
- **Track usage** — watch tokens per request to catch runaway growth early.

Treat the context window as a budget to force disciplined design.

### Key Takeaways: Context Window Mechanics

The context window is the fundamental constraint on every Claude inference call.

- Tokens are the unit of measure: roughly four characters per token.
- All inputs and outputs count: prompt, history, documents, response.
- Model tiers differ in their context window limits.
- Filling the window without a strategy leads to degradation or failure.

Budget context like you budget compute and memory.

---

## 2. Context Window Economics

Every token in your context window has a price.

### Economics Vocabulary

| Term | Definition |
|---|---|
| **Context window economics** | The cost-benefit tradeoff of how much context to include, balancing quality against price and latency. |
| **Token pricing** | The per-token rate for input and output. Output costs more than input; higher tiers charge more. |
| **Latency** | Elapsed time from request to response. Larger contexts raise time-to-first-token. |

### How Token Pricing Works in Practice

API pricing is per-token and applies to everything in the request.

- **Separate rates** — input and output tokens are billed differently.
- **Output is pricier** — output tokens cost more than input on most tiers.
- **Everything counts** — system prompt, history, documents, and response.

A 10,000-token system prompt over 1,000 daily requests is 10 million input tokens.

### Large Context vs. Small Context

- **Large context window** — higher cost per request and slower time-to-first-token. Enables
  long-document analysis and rich retrieval, but risks attention dilution from filler content.
- **Small context window** — lower cost and faster time-to-first-token. Forces disciplined
  content selection, and may need splitting or summarization to stay within limits.

### The Latency Cost of Long Contexts

Time-to-first-token scales with the input processed before any output.

- **More input** — extra attention computation before output starts.
- **Felt metric** — time-to-first-token is what users notice most.
- **Streaming helps** — it hides perceived latency, but total time still scales.
- **KV cache** — reduces latency for repeated stable prefixes.

For real-time apps, context size is a direct dial on response feel.

### Three Cost Levers in Context Design

- **System prompt size** — a bloated system prompt adds tokens to every request. Audit it and
  remove redundant instructions to cut baseline cost.
- **History policy** — carrying full history verbatim grows cost linearly. Summarize older turns
  to cap history token spend.
- **Retrieval precision** — loosely relevant documents waste tokens. High-precision retrieval
  lowers chunk count and cost.

### Why More Context Is Not Always Better

Irrelevant tokens are not neutral; they compete for the model's attention.

- **Noisy retrievals** — dilute the signal of the truly relevant chunks.
- **Contradictions** — conflicting passages cause inconsistent responses.
- **Long contexts** — can reduce instruction-following precision.

More context does not mean better answers; precision beats raw volume.

### Three Quality Mechanisms

| Term | Definition |
|---|---|
| **Attention dilution** | Reduced focus on relevant content when irrelevant or redundant tokens dominate the context. |
| **Retrieval precision** | The fraction of retrieved chunks that are genuinely relevant. High precision lowers noise. |
| **Prompt caching** | An API feature that reuses a stable prefix's computation to cut latency and input cost. |

### Prompt Caching as an Economics Tool

Caching amortizes the cost and latency of stable context across many requests.

- **Mark cacheable** — flag a system prompt or document prefix.
- **Compute once** — the first request builds the cache; later ones hit it.
- **Cheaper tokens** — cached tokens are billed at a much lower rate.
- **Faster start** — time-to-first-token drops as prefix work is skipped.

For stable, large system prompts, caching is some of the highest ROI available.

### Key Takeaways: Context Economics — Budget, Don't Fill

Effective context management balances cost, latency, and quality.

- Token pricing applies to every component of every request.
- Larger contexts increase both cost and time-to-first-token.
- Irrelevant tokens reduce quality, not just waste money.
- Prompt caching amortizes stable prefix cost across requests.

Treat the context window as a budget; spend it on signal.

---

## 3. Why Summaries Lose Facts

Summarization trades detail for fluency, and facts get dropped.

### Summarization and Fact Concepts

| Term | Definition |
|---|---|
| **History summarization** | Compressing prior turns into a condensed narrative. Loses specifics by design. |
| **Persistent fact block** | A structured, addressable prompt region holding extracted facts that persist through compression. |
| **Fact extraction** | A step that pulls structured facts out of turns into a separate block. |

### The Summarization Tradeoff

Summarization is a fluency-for-specificity tradeoff built into long context.

- **Fluency improves** — the compressed narrative reads cleanly.
- **Specificity drops** — exact values and names disappear.
- **Silent loss** — the model never flags what it compressed away.

Designing around this tradeoff starts with understanding what it discards.

### Fluency vs. Specificity

- **Fluency** — the summary reads well and keeps the gist. Relationships between concepts stay
  intact, so the model continues coherently without retaining exact details.
- **Specificity** — exact numbers, named entities, thresholds, and identifiers are dropped. A
  constraint set ten turns ago can simply vanish from working knowledge.

### Facts That Get Lost

Four categories of facts are most vulnerable to summarization loss.

- **Identifiers** — user IDs, ticket numbers, version strings.
- **Numeric limits** — token budgets, rate limits, cost thresholds.
- **Named entities** — people, tools, datasets, and services.
- **Decisions** — choices made earlier that constrain later steps.

These exact values are exactly what summaries throw away first.

### Symptoms of Fact Loss

- **Constraint amnesia** — a rule stated early in the session is silently ignored several turns
  later.
- **Entity confusion** — the model substitutes a plausible name or value for the one the user
  actually specified.
- **Decision drift** — a committed choice gets reopened as if still undecided, turns after
  agreement.

### How Loss Compounds Across Turns

Each pass starts from the previous summary, so loss compounds.

- **First pass** — minor details dropped from early turns.
- **Second pass** — a summary of a summary loses more.
- **Third pass** — only the broadest narrative survives.

Long sessions degrade faster than most developers expect.

### Two Update Strategies

| Term | Definition |
|---|---|
| **Append-only block** | A persistent block that grows by adding new facts and never overwrites prior entries. |
| **Replace-on-conflict** | A new fact value overwrites an older value for the same key when the two disagree. |
| **Extract-first pattern** | Run fact extraction before summarization so no facts are consumed in the compression pass. |

### Extract First, Summarize Second

Separating extraction from compression breaks the loss cycle.

1. Pull structured facts from the raw turn.
2. Write extracted facts into the persistent block.
3. Summarize the turn for rolling history.

Facts are captured before the summarizer can discard them.

### Key Takeaways: Facts Survive When Extracted First

Summarization is unavoidable in long sessions, but fact loss is not.

- Summaries trade specificity for fluency by design.
- Identifiers, numbers, entities, and decisions are most at risk.
- Loss compounds with every summarization pass.
- Extract-first captures facts before compression runs.

A persistent fact block keeps critical state addressable.

---

## 4. Designing a Persistent Fact Block

Keep critical state addressable across turns even as history compresses: extract from history
into a fact block, then consult it on every request.

### Block Design Vocabulary

| Term | Definition |
|---|---|
| **Append-only block** | A persistent block that grows by adding new facts and never overwrites prior entries. Can grow unbounded. |
| **Replace-on-conflict** | A new fact value overwrites an older value for the same key when the two disagree. |
| **Fact aging** | Removing facts not referenced in recent turns to keep the block size bounded. |

### Extraction Prompt Design

The extraction prompt decides what gets captured and how consistently.

- **Specify categories** — identifiers, numerics, decisions, named entities.
- **Request structured output** — JSON with typed fields per category.
- **Handle gaps** — instruct the model to output null for missing categories.

Explicit categories stop the model from collapsing distinct facts into prose.

### Append-Only vs. Replace-on-Conflict

- **Append-only** — every new fact is added as a new entry and prior entries are never
  overwritten. History is preserved, but block size grows without bound unless you prune.
- **Replace-on-conflict** — when a new value arrives for an existing key, the new value wins.
  Block size stays bounded, but earlier values are permanently gone.

### Injecting the Block into Requests

Where you place the block affects how reliably the model uses it.

- **Top of user turn** — visible but competes with the actual question.
- **Separate system block** — authoritative and clearly scoped.
- **Inline before the query** — high salience but slightly verbose.

A dedicated system block is the most reliable injection point.

### Size Management Strategies

- **Cap the block** — set a hard token limit and refuse new facts once reached.
- **Age out stale facts** — remove facts not cited in the last N turns.
- **Merge redundant entries** — consolidate facts about one entity under a single key.
- **Priority tiers** — mark facts permanent or ephemeral; only age out ephemeral ones.

### Handling Fact Conflicts

Conflicts arise when the same fact appears with different values.

- **Detect** — compare incoming facts to existing block entries.
- **Log both** — record old and new values before overwriting.
- **Escalate** — ask the user when the conflict is significant.

Silent overwrites hide data quality problems that surface later as errors.

### Validation and Downstream Checks

- **Downstream validation** — confirm the block matches the latest input and no facts were
  silently dropped.
- **Schema enforcement** — validate that extracted facts match expected types and ranges before
  writing.
- **Round-trip check** — re-extract facts from the block output and compare to the source turn to
  catch errors.

### The Most Common Failure Mode

Building a fact block, then never instructing the model to use it.

- **Unused block** — it exists in the prompt but no instruction references it.
- **Wrong source** — the model answers from general knowledge instead.
- **Expensive no-op** — the block costs tokens but changes nothing.

An explicit instruction to consult the block is required in every request.

### Key Takeaways: Persistent Block Design Essentials

A fact block works only when every layer of the design is intentional.

- Extraction prompts need explicit categories and structured output.
- Pick append-only or replace-on-conflict based on whether history matters.
- A dedicated system block is the most reliable injection point.
- Age out stale facts and cap block size to stay within budget.

Always instruct the model to consult the block.

---

## 5. The Memory Tool for Persistence

Compaction keeps a session going but loses detail. This module covers the memory tool, how a
model saves facts that outlast a chat, the mindset behind it, and how it differs from memory that
lives only in the chat.

### Three Terms to Anchor

| Term | Definition |
|---|---|
| **Memory tool** | A tool the model uses to read and write files under a memory path, so facts persist across separate conversations. |
| **Client-side tool** | A tool where the model requests an action and your application actually performs it against storage you control. |
| **In-context memory** | Anything the model knows only because it sits in the current transcript. It vanishes when that context is cleared. |

### How the Memory Tool Works

The model doesn't touch your storage directly. It asks, and your code acts.

- The model requests a file operation like view or create.
- Your application runs it and returns the result.
- Files live under a memory path you fully control.

So the model builds up notes over time without holding them all in context.

### Three Things It Can Do

- **Read** — the model views the memory folder and opens files to recall what an earlier session
  recorded.
- **Write** — it creates or edits files to save facts, progress, and decisions worth keeping
  around.
- **Organize** — it can rename or delete files so the store stays coherent instead of piling up
  clutter.

### Persistence Across Sessions

The point of the tool is survival across boundaries that clear context.

- A new session re-reads what the last one wrote.
- Facts outlast a context reset or a fresh conversation.
- Work resumes from recorded state, not from scratch.

The store is the thread that ties separate sessions into one ongoing effort.

### The Assume-Interruption Mindset

The tool ships with a specific design instruction worth taking to heart.

- Assume the context window can reset at any moment.
- Anything not written to memory is at risk of being lost.
- So check memory first, then record progress as you go.

Designing this way turns an interruption into a resume, not a restart.

### Memory Tool vs. In-Context Memory

- **Memory tool** — facts written to a durable store outside the chat. They survive resets and
  are read back on demand in later sessions.
- **In-context memory** — facts held only in the current transcript. Convenient now, but gone the
  moment that context is summarized or cleared.

### Three Uses That Fit

- **Long projects** — multi-session work like building a feature over days benefits most, since
  each session resumes from recorded state.
- **User preferences** — facts a user shares once, like a follow-up style, can be saved and
  reused instead of being asked every time.
- **Progress logs** — a running record of what's done and what's next lets an interrupted agent
  pick up cleanly where it left off.

### Pairing With Compaction

The two features are partners, not rivals.

- Compaction keeps the active context small on its own.
- The memory tool preserves what a summary might drop.
- Together they support long runs without losing the essentials.

Let compaction handle bulk while memory guards the facts that must survive.

### Key Takeaways: What to Carry Forward

- The memory tool keeps facts across conversations.
- It's client-side, so the model asks and your code stores files.
- Written facts persist across resets and new sessions.
- Design with the assume-interruption mindset.
- It contrasts with in-context memory, which clears.

---

## 6. The Tool-Output Bloat Problem

Tool calls return every field, but agents rarely need more than a few.

### Three Foundational Concepts

| Term | Definition |
|---|---|
| **Tool-output trimming** | Reducing a tool's response to only the fields the agent needs before it enters context. |
| **Signal-to-noise ratio** | The proportion of relevant content to filler in context; a primary driver of output quality. |
| **Context window pressure** | When accumulated tokens crowd out relevant content and the model attends to noise. |

### When APIs Return Too Much

A typical API response returns dozens of fields; most are irrelevant.

- **Mostly waste** — an 80-field response when the agent needs three.
- **Hidden filler** — metadata, pagination tokens, and audit stamps bloat every call.
- **Growing gap** — field counts grow with API versions while agent needs stay narrow.

The gap between what is returned and what is needed defines the bloat problem.

### Trimmed vs. Untrimmed Context

- **Untrimmed output** — all 80 fields land in context. The agent reasons over irrelevant
  metadata and filler, so signal is diluted and accuracy drops even when reasoning is sound.
- **Trimmed output** — only the three needed fields enter context. The agent focuses on relevant
  signal, reasoning quality improves, and token cost for the next request drops.

### How Bloat Stacks Across Calls

Bloat compounds when a workflow chains multiple tool calls together.

- **Every call adds** — each deposits unused fields into the shared context.
- **Noise stacks up** — after five calls, context holds five layers of clutter.
- **Reasoning suffers** — later steps attend to earlier clutter.

A single untrimmed call is an annoyance; a chain of them is a reliability failure.

### Two Hidden Cost Dimensions

- **Token cost** — every field in context is a token on the next request. Bloat is a recurring
  charge on every downstream call.
- **Latency cost** — larger context takes longer to process. Extra tokens slow generation even
  when they add no value.

### The Lost-in-the-Middle Effect

Models attend most strongly to content near the start and end of context.

- **Buried facts** — critical content mid-window receives weaker attention.
- **Pushed inward** — noisy tool output shoves key content toward the middle.
- **Bad positioning** — accuracy drops from placement, not from bad reasoning.

Trimming is not just about size; it keeps signal in high-attention zones.

### Two More Key Concepts

| Term | Definition |
|---|---|
| **Reference indirection** | Store the full response externally and pass the agent an identifier to fetch it on demand. |
| **Discard log** | A record of fields the trim layer discarded, useful when a downstream agent needs a missing one. |
| **Field-level filtering** | A deterministic trim that keeps a fixed list of fields and drops everything else. |

### A Common Architecture Mistake

Many teams trim at display time but not at model time.

- **Humans only** — display-layer trim hides noise from people, not the model.
- **Model still sees** — every bloated field is still in context.
- **UI is not the fix** — fixing the interface does not fix signal-to-noise.

Trim at the tool wrapper, before the output enters context, not after.

### Key Takeaways: Bloat Costs Accuracy and Money

Verbose tool outputs silently erode model performance across every call.

- APIs return far more fields than any single task requires.
- Bloat compounds across multi-call workflows, layering noise.
- The lost-in-the-middle effect turns excess tokens into accuracy risk.
- Trimming at the tool wrapper removes cost and latency at the source.

Trim at the source to build leaner agents.

---

## 7. Trim Strategies: Filter, Summarize, Reference

Three ways to tame verbose tool output before it enters context.

### The Three Trim Strategies

| Strategy | Definition |
|---|---|
| **Field-level filtering** | A deterministic trim that keeps a fixed list of fields and drops everything else before context entry. |
| **Model-driven summarization** | Route the raw output through a small model that distills it to a compact, task-relevant summary. |
| **Reference indirection** | Store the full response externally and pass the agent an identifier to fetch on demand. |

### Field-Level Filtering in Practice

Filtering is the simplest, cheapest trim when the tool schema is stable.

```python
keep = ["id", "status", "result"]
trimmed = pick(resp, keep)
return trimmed  # drop the rest
```

No model, no added latency — a deterministic reduction at the source.

### Stable Schema vs. Unstable Schema

- **Stable schema → filter** — when structure is predictable, declare the fields once and the
  wrapper runs deterministically.
- **Unstable schema → summarize** — when structure varies across versions, a fixed field list
  breaks. A small summarizer adapts.

### Model-Driven Summarization

When the schema varies, route raw output through a compact summarization step: raw output → small
summarizer model → compact summary → main agent context.

One small model call removes far more tokens than it adds.

### When to Use Each Strategy

- **Filter** — schema is stable and versioned. Pick fields once, run deterministically. Zero
  added latency, zero model cost.
- **Summarize** — schema is dynamic or varies by provider. A small model call adapts to any
  structure and compresses unpredictable output.
- **Reference** — payload is huge and the agent may not need it. Store externally, hand over an
  ID, fetch on demand.

### Reference Indirection for Large Payloads

For very large responses, reference indirection avoids the context cost entirely: store the full
payload, pass an ID, and fetch on demand.

Most agents finish without ever retrieving the full payload.

### Anti-Pattern and Audit Concepts

| Term | Definition |
|---|---|
| **Trim audit log** | A record of fields the trim layer discarded, useful when a later step needs a missing one. |
| **Over-trimming** | Dropping fields the agent later needs, forcing an expensive re-call to the original tool to recover them. |
| **Trim layer** | The wrapper between the raw tool response and the agent context that performs all field reduction. |

### Logging the Trim for Debugging

A trim audit log captures exactly what was discarded, making issues recoverable.

- **Log the discards** — record dropped fields alongside the kept fields.
- **Check first** — when a downstream step fails, read the log before re-calling the tool.
- **Spot drift** — logs reveal when the allowlist has grown too narrow over time.

Without a trim log, debugging over-trimming is guesswork.

### Key Takeaways: Three Tools, One Goal

Each strategy targets a different mix of schema stability and payload size.

- Filter when the schema is stable: deterministic, zero model cost.
- Summarize when the schema varies: flexible, one small model call.
- Use reference indirection for huge payloads the agent may not need.
- Log every discard so over-trimming failures are diagnosable.

Remove the most noise with the least complexity.

---

## 8. When to Escalate: Trigger Design

Agents need explicit triggers that tell them when to hand off, not push forward.

### Three Core Escalation Concepts

| Term | Definition |
|---|---|
| **Escalation trigger** | An explicit condition that makes an agent hand off control rather than keep executing autonomously. |
| **Confidence gate** | A trigger that fires when the agent's confidence in its step falls below a calibrated threshold. |
| **Out-of-policy detection** | A trigger that fires when a requested action falls outside an allowlist or hits a denylist. |

### Two Escalation Failure Modes

Every escalation system fails in one of two directions, and both are costly.

- **Never escalating** — silent bad output reaches users unreviewed.
- **Always escalating** — buries reviewers and trains them to ignore alerts.
- **Calibrated middle** — escalate only when the agent genuinely cannot proceed.

Good trigger design separates a useful guardrail from a useless one.

### Hard Triggers vs. Soft Triggers

- **Hard triggers** — explicit policy conditions in the system prompt; the agent always escalates
  when they fire. Hitting a denylist, exceeding a cost threshold, or a sensitive action. No model
  judgment involved.
- **Soft triggers** — model-judgment conditions where the agent decides based on confidence or
  context. Ambiguous intent or low-confidence reasoning. These are tunable and calibrated over
  time.

### Trigger Category: Confidence Scores

Agents attach a confidence estimate to each step and escalate when it drops too low.

```python
THRESHOLD = 0.7
if step.confidence < THRESHOLD:
    escalate("Low confidence")
```

Confidence gates catch steps where the model is guessing, not reasoning.

### Four Trigger Categories

- **Confidence below threshold** — self-assessed confidence falls under a calibrated cutoff.
- **Out-of-policy action** — action falls outside the allowlist or hits a denylist.
- **Repeated retry failure** — the same call or check fails more than N times.
- **Ambiguous user intent** — the request is unclear enough that proceeding is risky.

### Repeated-Failure Detection

An agent retrying the same failing action burns time and may be stuck in a loop.

- **Count failures** — track tool errors and validation rejections per session.
- **Set a ceiling** — a retry limit in the system prompt, typically two to three.
- **Escalate on breach** — hand off with a record of what was tried and what failed.

Failure counting turns a stuck loop into a clean escalation, not a timeout.

### Policy-Based Trigger Patterns

- **Allowlist check** — the agent verifies each action is explicitly permitted; anything unlisted
  triggers escalation.
- **Denylist check** — the agent scans its planned action against prohibited operations; a match
  is an immediate escalation.
- **Ask-for-help tool** — a pre-built escalation function the model calls when it should not
  handle a situation alone.

### The Ask-for-Help Tool Pattern

The most reliable trigger is one the model calls deliberately.

```json
{
  "name": "escalate",
  "description": "send to a human",
  "reason": "string",
  "context": "string"
}
```

An explicit escalation tool beats hoping the model writes a good message.

### Key Takeaways: Designing Triggers That Actually Fire

Escalation works only when triggers are explicit, calibrated, and logged.

- Two failure modes: never escalating and escalating constantly.
- Categories: confidence gates, policy checks, retry ceilings, ambiguous intent.
- Hard triggers are policy-enforced; soft triggers rely on model judgment.
- An ask-for-help tool gives the model a clean way to escalate deliberately.

Keep agents honest without burying reviewers in noise.

---

## 9. The Handoff Payload

What an agent passes forward decides whether the next actor can resolve or restart.

### The Handoff Payload Defined

| Term | Definition |
|---|---|
| **Handoff payload** | The bundle of context, history, and next step passed to the next actor. |
| **Closing the loop** | Re-injecting the resolver's response into the original agent's context. |
| **Escalation routing** | Directing the payload to the right next actor: human, specialist, or supervisor. |

### The Four Core Payload Fields

Exactly what the next actor needs, without re-reading the whole thread.

```json
{
  "summary": "what happened before escalation",
  "triggering_question": "the input that fired it",
  "attempted_actions": "the actions tried",
  "suggested_next_step": "the agent's best guess"
}
```

### Structured Handoff vs. Narrative Dump

- **Structured handoff** — a schema with defined fields: summary, question, actions, and next
  step. The receiver can parse and route it immediately, and metrics can be tracked.
- **Narrative dump** — a free-text block in whatever order the agent produced it. Harder to
  parse, harder to route, impossible to aggregate. May omit the one detail the reviewer needed
  most.

### Routing the Escalation

A structured payload can be routed to the right actor automatically: human reviewer, specialist
agent, or supervisor model.

Routing logic lives in the orchestrator, not the escalating agent.

### Three Common Handoff Failures

- **Missing context** — payload omits the prior history the next actor needs to see why the agent
  stopped.
- **No next step** — the agent describes what failed but offers no recommendation, leaving the
  reviewer on their own.
- **Unroutable format** — free-text payload cannot be parsed; it lands in a generic queue instead
  of the right inbox.

### Tracking Escalations as Data

Every escalation is a signal about your prompt or policy.

- **Log each handoff** — trigger type, payload fields, routing destination.
- **Aggregate over time** — find which triggers fire most often.
- **Refine from data** — adjust thresholds, add allowlist entries, fix prompts.

Escalation tracking turns individual failures into systemic improvements.

### Payload Schema in Practice

- **Triggering question** — the exact input that caused escalation, preserved verbatim so the
  next actor sees what the agent saw.
- **Attempted actions** — a log of tool calls, validations, and retries with results, so the
  reviewer skips what already failed.
- **Suggested next step** — the agent's recommended path forward, included even when it cannot
  execute it, as a starting point.

### Closing the Loop

Escalation is a round trip; the resolver's response must re-enter the agent's context.

- **Without closure** — the agent resumes from its pre-escalation state.
- **With closure** — the agent gets the resolution and can continue.
- **Injection point** — the context window the agent escalated from.

Loop closure makes escalation a recovery mechanism, not a dead end.

### Key Takeaways: Payloads That Enable Action

A handoff payload works only if the next actor can act without starting over.

- Four fields: summary, triggering question, attempted actions, suggested next step.
- Structured handoffs beat narrative dumps for routing and metrics.
- Route to a human reviewer, specialist agent, or supervisor model.
- Track every escalation to refine thresholds and policy over time.

Close the loop so the agent can continue.

---

## 10. Where to Catch the Error

When a subagent fails, recovery happens locally or propagates up the stack.

### Recovery Positions and Boundary

| Term | Definition |
|---|---|
| **Local recovery** | Handling a transient or recoverable error inside the same agent that produced it. |
| **Coordinator propagation** | Surfacing an error up to a parent agent or orchestrator so the higher-level plan can adapt. |
| **Propagation boundary** | The interface where the architect decides which errors surface to the coordinator. |

### Two Extremes, Neither Works

Skipping a deliberate policy lands architects at one of two failure modes.

- **Never propagate** — every error caught inside; the coordinator sees vague success.
- **Always propagate** — every hiccup surfaces; the coordinator thrashes on noise.
- **Deliberate boundary** — the right answer, not a default catch-all.

Designing that boundary is the core skill.

### Silent Failure vs. Thrashing

- **Silent corruption** — the subagent catches every exception and returns success. The
  coordinator proceeds on a false premise, data degrades, and the defect surfaces only after
  cascading downstream damage.
- **Coordinator thrash** — every minor retry or validation blip propagates. The coordinator
  spends most of its cycles on noise, retrying recoverable work it should never have seen.

### Errors That Stay Local

Some failures resolve at the subagent level and should never reach the coordinator.

- **Transient retries** — a rate-limit pause that resolves in milliseconds.
- **Recoverable validation** — bad input the agent can sanitize and reprocess.
- **Idempotent re-fetches** — a stale cache miss the agent refreshes itself.

If the coordinator's plan does not change, local recovery is the right call.

### Errors That Propagate Up

Other failures carry information the coordinator's plan depends on.

- **Scope violations** — the subagent was asked to act outside its authorization.
- **Repeated failure** — three attempts, no progress, no local path forward.
- **Policy denials** — a guardrail or access control blocked the action.

Staying silent here means the coordinator plans on a false assumption.

### The Coordinator-Needs-to-Know Test

Ask: **would the plan change?**

- **No** — recover locally, resolve in the subagent.
- **Yes** — propagate to the coordinator.
- **Unsure** — propagate with a low-severity signal.

### Retryable, Non-Retryable, and Budget

| Term | Definition |
|---|---|
| **Retryable error** | A failure that may succeed on a later attempt without changing inputs, such as a transient timeout. |
| **Non-retryable error** | A failure repeating the same call will not fix, such as an auth denial or a schema mismatch. |
| **Retry budget** | The maximum local retry attempts before the subagent stops and propagates the failure up. |

### Logging at the Boundary

Even when a subagent recovers locally, that recovery should leave a trace.

- **Log the recovery** — failure type, timestamp, and retry count before success.
- **Spot patterns** — structured logs reveal trends across many subagent runs.
- **No silent fixes** — an unlogged local recovery is step one toward corruption.

The coordinator may not need to act, but the operator always needs visibility.

### Key Takeaways: Local vs. Coordinator Recovery

The propagation boundary is an explicit architectural decision.

- Recover locally when the coordinator's plan would not change.
- Propagate scope violations, policy denials, and exhausted retries.
- Apply the coordinator-needs-to-know test per failure.
- Local recoveries still log a trace for operator visibility.

Design the boundary to prevent corruption and thrash.

---

## 11. Designing the Propagation Contract

When an error propagates, define exactly what travels up the stack.

### Propagation Contract Vocabulary

| Term | Definition |
|---|---|
| **Error envelope** | A structured payload describing an error — type, severity, and recovery hint — that travels up the stack. |
| **Severity level** | A classification (warning, recoverable, or fatal) that decides whether the coordinator retries, falls back, or escalates. |
| **Idempotency** | The property that retrying an operation produces the same effect as running it once; required for safe propagation. |

### The Error Envelope Structure

A structured object that carries failure information to the coordinator.

```json
{
  "error_type": "rate_limit",
  "severity": "recoverable",
  "context": { "agent_id": "A7", "task": "fetch" },
  "recovery_hint": "retry with backoff"
}
```

### Structured vs. Unstructured Errors

- **Structured envelope** — a typed object with fields for error type, severity, and context. The
  coordinator branches on severity, so recovery is deterministic.
- **Unstructured string** — a free-text message like "something broke in step 3." The coordinator
  cannot route it; recovery relies on brittle string matching.

### Severity Levels and Coordinator Policy

Each severity level maps to a defined coordinator response, not a runtime guess.

- **Warning** — log and continue, no immediate action.
- **Recoverable** — retry with backoff or substitute a fallback agent.
- **Fatal** — halt the dependent task chain and escalate to a human.

Severity without a defined policy is just a label; the policy gives it meaning.

### Coordinator Retry Options

- **Exponential backoff** — retry the same subagent with increasing wait intervals.
- **Fallback agent** — route the task to an alternate subagent or model.
- **Partial completion** — keep finished steps and reroute the failed one.
- **Human escalation** — pause the chain and surface it to an operator.

### Partial-Success Handling

Subagents often fail partway, completing some work before the error.

- **Report progress** — include a `completed_steps` field in the envelope.
- **Skip the done work** — the coordinator avoids re-executing finished steps.
- **Avoid duplication** — without it, a full retry repeats side effects.

Partial-success state is not a convenience; it is a correctness requirement.

### Idempotency at the Propagation Layer

Retrying an operation should produce the same outcome as running it once.

- **Doubled effects** — without idempotency, a retry after partial success repeats writes.
- **Idempotency keys** — external writes need a key to prevent re-execution.
- **Retry tokens** — the subagent honors them so re-invoking is safe.

Idempotency is not optional when propagated retries are part of recovery.

### Observability Across the Stack

Tracing an error from a leaf agent through the coordinator takes deliberate instrumentation.

- **Add a span** — each agent stamps its span ID into the envelope.
- **Link the trace** — the coordinator joins spans across the full call chain.
- **Follow the hop** — operators trace one request to the exact failure point.

Without this, debugging a propagated error is guessing which agent failed.

### Key Takeaways: The Propagation Contract

It defines what travels up the stack and what the coordinator does with it.

- Error envelopes carry typed, routable keys, not free-text strings.
- Severity levels map to explicit policies: retry, fallback, or escalate.
- Partial-success state prevents re-executing completed steps on retry.
- Idempotency keys protect against doubled side effects.

The subagent surfaces facts; the coordinator owns recovery.

---

## 12. When "Nothing" Is Ambiguous

An empty tool result can mean two very different things: searched and found nothing, or never
reached the source.

### Three Foundational Concepts

| Term | Definition |
|---|---|
| **Access failure** | A tool call that did not reach the data source due to a connection error, auth failure, timeout, or partial response. |
| **Empty result** | A tool call that successfully reached the data source and confirmed there is no matching content. |
| **Silent false negative** | A bug where an agent reports nothing exists because a tool failed silently, not because data is absent. |

### The Concrete Failure Scenario

Consider a search tool querying a vector index for relevant documents.

- **Index offline** — the service goes down mid-session.
- **Empty list returned** — the tool call returns nothing, with no error message.
- **Wrong conclusion** — the agent reads the empty list and assumes no documents exist.

The failure looks identical to a successful empty search; the silent bug is born.

### Access Failure vs. Empty Result

- **Access failure** — the tool could not reach the data source. The absence reflects a
  connection problem, not a content gap. Retrying is appropriate, and the agent should draw no
  conclusions from the empty response.
- **Empty result** — the tool reached the source and confirmed no matching content. This is a
  valid, confirmed answer. Retrying wastes effort, and the absence is meaningful information.

### Why Agents Assume Nothing Exists

Agents trust the tool output as ground truth, because that is what the prompt instructs.

- **Empty signals none** — an empty list reads as "no data" to the model.
- **No error, no branch** — without an error flag, no failure path runs.
- **False confidence** — the agent proceeds on incomplete information.

The model cannot tell ambiguous nothing from confirmed nothing without a signal.

### Four Categories of Access Failure

- **Connection error** — the endpoint is unreachable, so the tool never connects.
- **Auth failure** — credentials are missing or revoked; the request is rejected.
- **Query timeout** — the source responds too slowly and the call is abandoned.
- **Partial result** — some records return but the rest are silently truncated.

### The Dangerous User-Facing Answer

When an access failure returns empty, the agent often says "I could not find anything."

- **Implies success** — that phrase sounds like a search with no matches.
- **User believes it** — they accept it as proof the data does not exist.
- **False premise** — decisions get made on a fabricated absence.

The right answer to an access failure is "I was unable to check," not "I found nothing."

### Three More Critical Concepts

| Term | Definition |
|---|---|
| **Tool wrapper** | A thin layer between the underlying tool and the agent that normalizes responses and reports status. |
| **Status field** | A field on every tool response naming the outcome so the agent can branch. |
| **Ambiguous nothing** | An empty response that could mean an empty search or a failed access; the silent bug's root. |

### Production Cost of Getting This Wrong

The silent false negative is one of the hardest agent bugs to diagnose in production.

- **Confident errors** — users get confident but factually wrong answers.
- **Bad decisions** — downstream choices rest on a fabricated absence.
- **No trace** — logs show no error because no exception was raised.

The cost compounds: lost trust, bad decisions, and no trace back to the failure.

### Key Takeaways: The Ambiguity That Breaks Agents

An empty result can mean a confirmed empty search or a failed access attempt.

- Agents default to "nothing exists" when tools fail silently.
- Access failures span connection errors, auth, timeouts, and partial results.
- The phrase "I found nothing" is wrong when access failed.
- Tool wrappers and status fields are the fix.

Disambiguate to prevent the silent false negative.

---

## 13. Disambiguation Patterns

The fix for the silent false negative lives in the tool wrapper, not the model.

### Wrapper, Status, and Retry

| Term | Definition |
|---|---|
| **Tool wrapper** | A thin layer between the raw tool and the agent that normalizes responses and enforces disambiguation. |
| **Status field** | A structured field on every tool response naming the outcome so the agent branches instead of guessing. |
| **Retry discipline** | The wrapper's rule that access failures retry while confirmed empty results do not. |

### Wrapper Rule: HTTP Success vs. Failure

The wrapper distinguishes an HTTP success with an empty body from an HTTP failure.

- **200 with empty list** — a valid, confirmed empty result.
- **4xx or 5xx** — an access failure regardless of body content.
- **Timeout** — an access failure even with no HTTP code at all.

The wrapper maps each outcome to a status field before the agent sees it.

### The Status Enum: Four Outcome Classes

Every response carries one of four explicit outcome classes.

```python
status = Enum(
    "success",         # has results
    "empty_result",    # no match found
    "partial_result",  # truncated
    "access_failure",  # retry it
)
```

### Agent Prompt Rule: Branch on Status

The prompt needs explicit handling for each status class, with no default assumptions.

- **Success** — use the results normally.
- **Empty result** — confirm the absence and respond accordingly.
- **Partial result** — flag uncertainty, do not treat as complete.
- **Access failure** — do not conclude; retry per policy and surface it.

Implicit "treat empty as nothing" is exactly what the status field eliminates.

### Empty Result vs. Access Failure: Retry

- **Empty result: no retry** — the source confirmed no match exists. Re-running the same query
  against the same index returns the same empty result. Retrying only wastes latency and quota for
  no benefit.
- **Access failure: retry** — the source was unreachable and the condition may be transient.
  Retry with exponential backoff up to a configured limit before escalating to the user.

### Logging the Access Failure Correctly

Every access failure needs a structured log entry at the moment it occurs.

- **Timestamp** — when the failed call occurred.
- **Tool identity** — the tool name and endpoint.
- **Error class** — timeout, auth, connection, or partial.
- **Outcome** — the retry strategy and final result.

Without this log the session looks clean, but the failure is invisible.

### User-Facing Language Patterns

- **"Found nothing"** — only for confirmed empty results: the search succeeded and nothing
  matched.
- **"Could not check"** — for access failures: the absence of results is unconfirmed.
- **"Partial results"** — when the tool flagged a cut-off: some data may be missing.

### Schema Rule: Status Field Is Mandatory

Every tool response schema needs a status field, not an optional one.

- **Data is not enough** — the data field alone cannot disambiguate.
- **Typed, not text** — status should be a typed enum, never a free-text string.
- **Wrapper fills it** — status is populated before the agent sees the response.

Optional status fields lead to inconsistent behavior and a return of the silent bug.

### Key Takeaways: Building Disambiguation Into the Stack

Disambiguation is an architectural commitment, not a prompt tweak.

- Wrappers map HTTP outcomes to explicit status fields before agents see results.
- The status enum drives branching across four outcome classes.
- Retry discipline: access failures retry with backoff, empty results do not.
- User-facing language tracks status: "found nothing" vs. "could not check."

Every layer enforces the same distinction.

---

## 14. The Scratchpad Pattern

A durable memory slot that survives every session reset — typically a plain file like `NOTES.md`.

### Three Terms Anchor This Lesson

| Term | Definition |
|---|---|
| **Scratchpad file** | A plain text file like `NOTES.md` that holds durable cross-session state for an agentic coding workflow in one repo. |
| **Cross-session knowledge** | Decisions, conventions, and prior context that should persist across separate sessions. |
| **Decision log** | A scratchpad subsection recording architectural choices and rationale so sessions do not re-litigate them. |

### The Cross-Session Problem

Every new session opens with a blank context window and no memory of the last one.

- **Invisible decisions** — prior choices are unknown to the new session.
- **Repeated dead ends** — explored paths get re-explored needlessly.
- **Convention drift** — naming and terminology slip without a record.

Without persistence, each session repeats costly discovery work.

### Memory Lost vs. Memory Saved

- **No scratchpad** — the agent re-explores dead ends, re-asks resolved questions, and may
  contradict earlier decisions. Each session effectively starts over, wasting time and adding
  inconsistency.
- **With scratchpad** — the agent reads the file at session start, picks up where work left off,
  and saves notes before the session closes. Continuity is preserved across resets.

### What Goes Into the Scratchpad

The scratchpad holds evolving session state — the things that change run to run.

- **Decisions** — "chose Postgres over Mongo for joins."
- **Dead ends** — "dropped the GraphQL layer, too slow."
- **Conventions** — "snake_case for all table names."
- **Task state** — "in progress: auth; next: billing."

### What Stays Out of the Scratchpad

- **Code** — code lives in the codebase, not the scratchpad. Duplicating it creates drift and
  stale copies.
- **Credentials** — secrets and API keys should never appear in any plain text file committed to
  the repo.
- **Private user data** — personal or sensitive data has no place in a shared repository file,
  even temporarily.

### Loading at Session Start

The scratchpad helps only if the agent reads it reliably at every session start.

- **List it first** — name it among the first files to read in CLAUDE.md.
- **Enforce it** — an explicit hook or instruction makes the read a rule.
- **Or lose it** — without a rule, agents skip the file and lose context.

Treat the read as mandatory, not optional.

### Enforcement Mechanisms

- **CLAUDE.md instruction** — a standing rule in CLAUDE.md telling the agent to read the
  scratchpad at session start and write updates before the session ends.
- **Explicit hook** — a `SessionStart` hook can surface the scratchpad automatically. A
  `SessionEnd` hook runs on termination but cannot block.
- **Write discipline** — the habit of updating the scratchpad before closing a session so the
  next session inherits accurate state.

### Writing at Session End

Updating the scratchpad before ending a session is the write half of the discipline.

- **Record decisions** — capture choices made during the session.
- **Note dead ends** — write down paths reached so they are not revisited.
- **Update task state** — refresh the in-progress and next-up block.

A stale scratchpad is worse than none; the agent will trust outdated information.

### Key Takeaways: Scratchpad Pattern Foundations

A scratchpad file gives the agent a durable memory slot across context resets.

- Every session opens blank; the scratchpad restores continuity.
- Store decisions, dead ends, and current task state.
- Never store code, credentials, or private data.
- Enforce the read at session start and the write at session end.

No read-write discipline, no benefit.

---

## 15. Scratchpad Hygiene and Limits

An unpruned scratchpad becomes a graveyard of stale notes the agent trusts.

### Three Terms Anchor This Lesson

| Term | Definition |
|---|---|
| **Scratchpad hygiene** | Pruning resolved entries, structuring the file consistently, and avoiding stale notes that mislead the agent. |
| **CLAUDE.md vs. scratchpad** | CLAUDE.md holds stable rules; the scratchpad holds evolving state. On conflict, CLAUDE.md wins. |
| **Pruning policy** | A defined rule for removing resolved entries, archiving completed state, and capping file length. |

### Length Discipline: Cap and Prune

Scratchpads bloat fast — each session adds entries and few remove them.

- **Set a cap** — a line or word limit that triggers a pruning pass.
- **Remove on resolve** — delete resolved entries as soon as they close.
- **Archive state** — move completed task state out instead of piling it up.

A bloated scratchpad is slower to read and harder for the agent to reason over.

### Recommended Structure Blocks

- **Dated entries** — timestamp entries so stale notes are easy to spot and prune.
- **Decision log** — architectural choices and rationale; rarely pruned.
- **Current-state block** — what is in progress now; rewritten, not appended.
- **Open questions** — unresolved items; removed once they are answered.

### Pruning Policy in Practice

A pruning policy defines when and what gets removed from the scratchpad.

- **Remove on resolve** — clear entries in the session that resolves them.
- **Rewrite, don't append** — refresh the current-state block each session.
- **Archive history** — move completed state to a separate file if needed.

Without a policy, every session adds content and nothing is ever removed.

### CLAUDE.md vs. Scratchpad

- **CLAUDE.md** — holds stable rules for every session: style conventions, tool permissions,
  standing instructions. Changes rarely and represents the settled state of the project.
- **Scratchpad** — holds evolving session state: current task, recent decisions, open questions.
  Changes every session. When the two conflict, CLAUDE.md is authoritative.

### Multi-Developer Coordination

A scratchpad committed to git becomes a shared resource across the team.

- **Shared visibility** — developers read each other's state and decisions.
- **Conflicts as signal** — merge conflicts flag concurrent work on a task.
- **Clear ownership** — agree on who owns the current-state block when pairing.

One scratchpad per repo works well for small teams with clear task ownership.

### Privacy and Commit Safety

Before committing the scratchpad, review what it actually contains.

- **No secrets** — never commit credentials, tokens, or API keys.
- **Be explicit** — state in CLAUDE.md what is safe to write to the scratchpad.
- **Gitignore it** — exclude the file if it will hold sensitive context.

Privacy is an active choice, not a default; make the policy explicit.

### The Graveyard Failure Mode

- **Stale notes** — old entries that no longer reflect reality; the agent reads them and acts on
  outdated information.
- **Blind trust** — the agent never verifies scratchpad content against the codebase, so stale
  notes reach decisions.
- **No pruning** — without a removal discipline, the file grows into a graveyard of resolved or
  incorrect state.

### Key Takeaways: Keeping the Scratchpad Healthy

A healthy scratchpad requires active maintenance, not just initial setup.

- Cap the length and enforce a pruning policy every session.
- Use a consistent structure: dated entries, decision log, current state, open questions.
- CLAUDE.md rules when it conflicts with the scratchpad.
- Coordinate shared scratchpads via git with clear ownership.

A stale scratchpad is worse than none; keep it current.

---

## 16. Why Delegate Discovery

Verbose investigation pollutes context; a subagent keeps the main agent focused. The parent sends
a question, the subagent works in disposable context, and only a summary returns.

### Three Anchoring Concepts

| Term | Definition |
|---|---|
| **Subagent delegation** | Spawning a child agent to handle verbose investigation, returning only a focused summary to the parent. |
| **Discovery** | Finding relevant files, documentation, or context inside a large codebase or knowledge base. |
| **Disposable context** | A subagent's context window, discarded after it returns, so verbose state never reaches the parent. |

### Discovery Is Inherently Verbose

A single "find the auth flow" request can touch twenty or more files.

- **Noisy grep** — results fill pages and most are false positives.
- **Piled dead ends** — many wrong files before the right one appears.
- **Unfiltered reads** — file contents land in context, useful or not.

The main agent absorbs all of it, crowding out task-relevant state.

### Inline Discovery vs. Delegated Discovery

- **Inline discovery** — the main agent reads files directly. Every grep result, dead end, and
  file body lands in its context, crowding out task-relevant state.
- **Delegated discovery** — a subagent reads the files and returns only its finding. The main
  agent's context stays clean; the verbose state is thrown away with the child.

### How Delegation Keeps Context Clean

Four steps isolate the verbose state inside the subagent: frame the question → spawn the subagent
→ subagent investigates → returns a summary.

The main agent acts on the summary and never sees the raw reads.

### When Delegation Pays Off

- **Codebase surveys** — multi-file investigations that read many candidates before narrowing to
  the right one.
- **Doc spelunking** — searching long reference docs or changelogs where most pages are not
  relevant.
- **Multi-step lookups** — tasks that chain several searches, producing layers of intermediate
  results.

### When Delegation Is Not Worth It

Spawning a subagent adds overhead: latency, cost, and coordination work.

- **Single-file lookups** — not worth spawning a subagent.
- **Known path** — when you already know the file, read it inline.
- **Obvious answers** — short doc pages need no child agent.

The breakeven point is roughly three or more files read in sequence.

### Three Boundary Concepts

| Term | Definition |
|---|---|
| **Return contract** | The agreed shape of a subagent reply that keeps context clean. |
| **Trust boundary** | Treat subagent output as a hypothesis to verify, not as fact. |
| **Disposable context** | Verbose state — file lookups, dead ends, grep logs — dropped when the child returns. |

### The Delegation Trade

Delegation exchanges subagent cost for main-agent clarity.

- **Cost side** — a subagent run adds latency and token spend.
- **Benefit side** — the main agent avoids pollution and stays on task.
- **Scales up** — net benefit grows with discovery complexity.

For large investigations the trade is almost always favorable.

### Key Takeaways: Discovery Belongs in a Subagent

Discovery work is verbose by nature and pollutes the main agent's context.

- Inline discovery forces noise and findings into the same context window.
- Subagent delegation isolates that noise in a disposable context.
- Delegation pays off for multi-file surveys and long doc searches.
- Trivial single-file lookups do not justify the spawning overhead.

Clean context keeps the main agent effective.

---

## 17. Delegation Mechanics

Three decisions make subagent delegation work: prompt, return, and tools.

### Three Terms Anchor This Lesson

| Term | Definition |
|---|---|
| **Return contract** | The explicit shape of the response a subagent produces to keep the parent's context clean. |
| **Trust boundary** | The rule that the parent treats subagent output as a hypothesis to verify, not truth. |
| **Subagent delegation** | Spawning a child agent for verbose investigation that returns only a focused summary. |

### Subagent Prompt Design

A subagent prompt is a precise question, not an open-ended task.

- **Say what to find** — state exactly what the subagent must locate.
- **Set the scope** — name which directories, files, or topics to check.
- **Say what to skip** — list what to ignore to prevent scope creep.

Vague prompts produce vague answers that re-pollute parent context.

### What a Precise Prompt Specifies

- **The question** — one concrete question, not a broad directive.
- **The scope** — which directories and files to search or skip.
- **The return shape** — a path, a signature, or a short summary.
- **The stop condition** — when to stop looking and return findings.

### Vague Prompt vs. Precise Prompt

- **Vague prompt** — "Find the auth logic and explain it." The subagent reads broadly and returns
  a wall of text that pollutes context as badly as inline discovery.
- **Precise prompt** — "Return the file and function that handle JWT validation." One path, one
  name, one sentence of context. The parent gets exactly what it needs.

### Defining the Return Contract

The return contract specifies what the subagent returns.

- **Structure wins** — a structured finding beats a narrative every time.
- **Strong default** — file path, line number, and a brief summary.
- **Cap the length** — set a maximum to prevent context re-pollution.

A well-defined contract keeps parent context clean no matter what the subagent read.

### Scoping the Tool Palette

Give the subagent enough tools to investigate, not enough to act.

- **Allow read and search** — these are almost always appropriate.
- **Block write and delete** — keep mutating and API-call tools off the palette.
- **Narrow is safe** — a small palette prevents accidental side effects.

The subagent's job is to find, not to change anything.

### The Trust Boundary in Practice

- **Fallible reads** — subagents can misread a function or miss a file.
- **Verify critical paths** — have the parent spot-check important findings.
- **Trust low stakes** — minor lookups can proceed on the answer directly.

### Common Failure Modes

- **Wall of text return** — returns a narrative instead of a structured finding, re-polluting the
  parent's context.
- **Scope creep** — reads past its scope and hands back findings the parent never asked for.
- **Blind trust** — the parent acts on output without verifying critical findings, propagating
  errors downstream.

### Key Takeaways: Prompt Tight, Return Clean

Effective delegation depends on precise prompts and strict return contracts.

- State the question exactly: scope, shape, and stop condition.
- Define a structured return with a maximum length.
- Give the subagent read tools only, never write access.
- Treat output as a hypothesis and verify critical findings.

Vague prompts and loose returns break delegation.

---

## 18. Long-Conversation Coherence

As history grows, context pressure threatens earlier instructions and facts.

### The Model's Own Context Awareness

Claude gets signals about its remaining context, so it can pace long sessions.

- An injected token budget tag states how much room the model has.
- After a tool call, a system warning can flag that context is filling.
- These cues let the model wrap up before it truncates.

The signal guides behavior; it is not a hard guarantee against overflow.

### Core Vocabulary

| Term | Definition |
|---|---|
| **Conversation compaction** | Summarizing earlier turns to cut tokens; lossy by design. |
| **Context window pressure** | Tokens filling as history grows, risking loss of earlier turns. |
| **Session memory** | Any mechanism that persists state so the model keeps prior context. |

### How Context Window Pressure Builds

Every turn adds tokens; without management, accumulation hits the context limit.

- **History grows** — each user turn and assistant reply is appended.
- **Fixed overhead** — system prompt and tool definitions cost tokens every request.
- **At the limit** — providers truncate older turns or refuse the request.

Distinct from lost-in-the-middle: here the issue is accumulation, not position.

### In-Context vs. External Memory

- **In-context memory** — all history lives in the active conversation. Simple, with no retrieval
  step, but limited to the context window. Best for short to medium sessions.
- **External memory** — summaries and key facts stored in a database, retrieved and injected at
  turn start. Scales beyond the window and enables multi-session persistence.

### How to Compact the History

Compaction replaces verbose history with a compressed form.

- **Full summarization** — replace all prior turns with one concise summary.
- **Selective summarization** — keep key decisions, drop small talk and filler.
- **Sliding window** — keep the last N turns verbatim with a summary header.

Apply when topic continuity matters but verbatim history is not required.

### When Compaction Fits, and When It Doesn't

- **Apply compaction** — long sessions needing topic continuity but not verbatim history:
  support, research, assistants.
- **Avoid compaction** — code review needing earlier snippets, or legal records that must stay
  verbatim.
- **Compaction is lossy** — fine-grained early details may be lost for good; design with that
  constraint in mind.

### Hybrid Memory: Summary Plus Recent Turns

The hybrid pattern pairs in-context precision with external-storage scale.

- **Summary header** — inject a session summary atop each new API call.
- **Recent verbatim** — keep the last N turns in context for precision.
- **Persist it** — store the summary between sessions for continuity.

Best fit for personal assistants and long-running agents needing both.

### Coherence Techniques for Long Sessions

Even within the window, long sessions wander without active coherence management.

- **Session header** — open each call with a brief prior-context summary.
- **State tracking** — have the model keep a JSON object of goals and decisions.
- **Goal anchoring** — re-state the user's original goal at each turn.

Goal anchoring counters scope creep in multi-step workflows.

### Conversation Type Drives Memory Strategy

- **Short-session use cases** — support, one-off Q&A, single-task assistants. Need coherence
  within the session only; in-context memory is usually enough.
- **Long-session use cases** — personal assistants, code co-pilots, multi-session workflows. Need
  cross-session memory; hybrid or external memory is required, often paired with prompt caching.

### Key Takeaways: Coherence, Compaction, and Session Memory

Long conversations lose earlier context as tokens accumulate.

- Compaction is lossy: use only when verbatim history is not needed.
- In-context is simple; external memory scales beyond the window.
- Hybrid patterns combine recent turns with a summary header.
- Goal anchoring and state tracking prevent drift in long sessions.

Manage context from day one.

---

## 19. Server-Side Compaction

Long conversations eventually hit the context window. This module covers server-side compaction —
the built-in way to keep a session going — how it differs from your own rolling summary, and the
one habit that keeps facts from getting lost.

### Three Terms to Anchor

| Term | Definition |
|---|---|
| **Context window** | The fixed span of tokens a model can read at once. Every message, tool call, and result counts against it. |
| **Compaction** | A server-side step that, by default, summarizes the whole conversation when the window fills, then continues from that summary. |
| **Durable state** | Facts you store outside the transcript, in a file or database, so nothing is lost when the history is summarized. |

### What Compaction Does

Once you enable it, the platform summarizes automatically at the threshold you set.

- By default it summarizes the whole conversation into a compact block.
- The session continues from that summary, not the raw turns.
- Keeping recent turns verbatim is opt-in via `pause_after_compaction`.
- You keep talking to the model, and the long history quietly shrinks.

### Why It Exists

Every long agent run eventually collides with the same wall as it grows.

- The window is finite, so old turns steadily crowd out new ones.
- Without help, a session stalls once the history overflows the space.
- Compaction buys back room so the work can continue past that point.

The goal is a longer effective session with minimal loss of what mattered.

### Server-Side vs. Your Rolling Summary

- **Server-side compaction** — you opt in with a beta header and a context-management strategy,
  then set the trigger threshold. The platform runs the summary at that point for you.
- **App-level rolling summary** — your own code watches the length and calls the model to
  summarize. You own the timing, the prompt, and what gets kept.

### Three Trade-Offs

- **Less to build** — compaction runs inside the platform, so you skip writing code.
- **Less control** — you give up fine control over what is kept.
- **Not free** — the summary still has limits and a cost.

### The Foundations Catch

Any summary is lossy, and that's the point to internalize here.

- A fact not captured in the summary is simply gone from context.
- Exact numbers, IDs, and code can quietly drop out.
- The model can't recover what the summary left behind.

So never treat the transcript as your only copy of something important.

### Keep State Outside the Chat

Make it a habit:

- Write durable facts to a file or database as you go.
- Re-read that store instead of trusting old turns.
- Let compaction handle bulk while your store holds the essentials.

This pairs naturally with the memory tool.

### Three Practical Signals

- **Long sessions** — extended chats and multi-step agent runs are where compaction earns its
  keep, since they fill the window fastest.
- **Lossy summary** — treat the summarized history as a gist, not a ledger. Anything you truly
  need should live somewhere else.
- **Cite the docs** — trigger points and identifiers evolve, so confirm current behavior in the
  compaction documentation before you rely on specifics.

### Key Takeaways: What to Carry Forward

- By default it summarizes the whole conversation and continues from that.
- It differs from a rolling summary your own code triggers.
- Summaries are lossy, so facts can disappear.
- Keep durable state outside the transcript.

External state is why nothing critical is lost.

---

## 20. Per-Field Confidence Scores

A single score hides which fields are weak; score each field instead.

### Three Anchoring Concepts

| Term | Definition |
|---|---|
| **Field-level confidence** | A score emitted per extracted field rather than for the whole record. |
| **Routing threshold** | The cutoff above which a field is auto-accepted and below which it is sent to review. |
| **Calibration** | Empirical alignment between emitted scores and real accuracy on a labeled set. |

### The Coarse-Confidence Problem

One score for the whole record conceals which individual fields are unreliable.

- **Hidden weak spots** — a record at 0.85 may hide a date field at 0.40.
- **Wasted review** — reviewers re-check high-confidence fields they could skip.
- **Blind policy** — decisions on the aggregate miss the fields that need help.

Per-field scores expose exactly where the model is uncertain.

### Record Score vs. Field Score

- **Record-level score** — one number covers the whole extraction. Efficient to compute but hides
  weak spots, so routing sends the entire record to review even when only one field is uncertain.
- **Field-level score** — a score per extracted value. Routing targets only the uncertain fields
  and lets confident ones pass, cutting reviewer workload.

### Eliciting Per-Field Confidence

There's no native confidence field. The score is a verbalized self-report you prompt the model to
produce.

- Ask the schema to hold a value key and a confidence key per field.
- The model estimates the score; it is not a raw internal probability.
- A self-reported number means nothing until a held-out set calibrates it.

Prompt design elicits the score; calibration is what makes it trustworthy.

### Reading the Score Output

- **High confidence** — at or above threshold; the field is auto-accepted and skips review
  entirely.
- **Low confidence** — below the threshold; the field is flagged and queued for a human reviewer
  to verify.
- **Borderline score** — near the threshold; policy may add a secondary check or a tighter
  per-field cutoff.

### Mixed-Confidence Records

Most production records blend high- and low-confidence fields.

- **Split scores** — a name at 0.95 next to an invoice date at 0.38.
- **Partial routing** — the name flows through, the date goes to review.
- **Less queue** — only the uncertain portion of the record is reviewed.

Partial routing reduces reviewer load without sacrificing accuracy.

### Routing Policy Concepts

| Term | Definition |
|---|---|
| **Per-field routing** | Routing decided independently for each field on its own confidence, not the record average. |
| **Per-record routing** | Routing the whole record to review when any single field falls below the threshold. |
| **Policy scope** | Whether the rule applies at field or record level, which sets throughput and workload. |

### The Calibration Trust Problem

Model-emitted confidence scores are not automatically trustworthy.

- **Overconfident** — a model may report 0.90 where it is only 60% accurate.
- **Bad thresholds** — cutoffs on uncalibrated scores miss errors in production.
- **Check honesty** — calibration against labeled data reveals if scores are real.

Always check confidence on real data before setting production thresholds.

### Key Takeaways: Per-Field Confidence in Practice

Per-field scores expose weak fields that aggregate scores conceal.

- Field-level confidence is elicited via schema siblings beside each value.
- High-confidence fields auto-accept; low-confidence fields route to review.
- Mixed-confidence records are the norm in production systems.
- Per-field routing reduces reviewer load versus per-record routing.

Validate scores before trusting routing.

---

## 21. Calibrating Confidence Scores

Does the model's confidence actually match how often it is right?

### Three Calibration Concepts

| Term | Definition |
|---|---|
| **Validation set** | A held-out, labeled dataset used to measure accuracy and calibrate confidence thresholds. |
| **Reliability diagram** | A plot of confidence versus empirical accuracy that reveals over- and under-confidence. |
| **Routing threshold** | The confidence cutoff adjusted after calibration to reflect real accuracy, not raw output. |

### What Calibration Actually Means

It is the alignment between what the model is confident about and what it gets right.

- **Calibrated** — at confidence 0.90, it is correct 90% of the time.
- **Over-confident** — reports 0.90 but is correct only 60% of the time.
- **Under-confident** — reports 0.60 but is correct 90% of the time.

Calibration makes the score a trustworthy proxy for accuracy.

### Building a Validation Set

A useful validation set is representative, labeled, and never used in training.

- **Representative** — draw from the full distribution of real documents.
- **Labeled** — every field has a ground-truth value you are measuring against.
- **Held out** — the set is never used to train or prompt the model.

Aim for enough samples per confidence bin to get stable accuracy estimates.

### Over-Confident vs. Well-Calibrated

- **Over-confident model** — reports high confidence on fields it often gets wrong, so routing
  trusts the score and auto-accepts errors. Its reliability curve bows below the ideal diagonal.
- **Well-calibrated model** — reported confidence closely matches accuracy: a field at 0.85 is
  right about 85% of the time. Its reliability curve tracks the ideal diagonal.

### Running the Calibration Check

Four steps turn a validation set into a calibration result:

1. **Bucket by score** — group predictions into confidence bins.
2. **Measure accuracy** — compute empirical accuracy per bin.
3. **Plot the result** — chart confidence against accuracy.
4. **Find the gaps** — compare each bin to the ideal diagonal.

Bins above the diagonal are under-confident; bins below are over-confident.

### Reading a Reliability Diagram

The diagram maps what the model claims to what it actually delivers.

- **X-axis** — model confidence from 0.0 to 1.0.
- **Y-axis** — empirical accuracy from 0.0 to 1.0.
- **Diagonal** — a perfectly calibrated model traces it exactly.

Gaps between the line and the diagonal reveal where the model misleads routing.

### Adjusting the Routing Threshold

Set the threshold to the score that delivers the accuracy your policy requires.

- **Pick the target** — find the required accuracy level on the diagram.
- **Read across** — take the matching confidence value on the x-axis.
- **Set it** — use that confidence value as the routing threshold.

The threshold now reflects real accuracy, not self-reported confidence.

### Re-Calibrating After Changes

Calibration is not one-time; model upgrades and prompt changes invalidate it.

- **New model** — a version change shifts the confidence distribution.
- **New prompt** — a revision can move scores on identical documents.
- **Re-run it** — repeat the check on the same validation set after changes.

Treat calibration as a recurring task scheduled alongside model updates.

### Key Takeaways: Calibration as Ongoing Practice

Calibration aligns model confidence with real extraction accuracy.

- A validation set provides labeled ground truth per field.
- The check bins predictions by confidence and measures accuracy.
- A reliability diagram reveals over- and under-confidence.
- Thresholds set after calibration reflect actual accuracy.

Re-calibrate after every model or prompt change.

---

## 22. Stratified Sampling for Review

Random review surfaces what the model already gets right; stratify to find errors.

### Three Terms Anchor This Lesson

| Term | Definition |
|---|---|
| **Stratified sampling** | Selecting review items disproportionately from low-confidence, rare, or high-stakes outputs. |
| **Stakes weighting** | Adjusting review routing by the downstream impact of an error, independent of the confidence score. |
| **Reviewer burnout** | When reviewers get too much correct output and miss real errors when they finally appear. |

### Why Random Sampling Falls Short

A model that is 95% accurate means 95% of random samples look fine.

- **Rare failures** — reviewers seldom see the 5% that matters.
- **Hidden patterns** — errors stay invisible across category boundaries.
- **No extra scrutiny** — high-stakes outputs get none by default.

Random review gives a false sense of quality without catching what breaks.

### Random vs. Stratified Review

- **Random sampling** — easy to implement, with every output equally likely to be picked.
  Reviewers see mostly correct outputs, so errors are statistically rare in the review queue.
- **Stratified sampling** — needs confidence scores and category labels, and oversamples failures
  and edge cases. Reviewers see concentrated signal and catch more real errors per hour.

### Stratifying by Confidence Score

Confidence scores point directly at where the model is uncertain.

- **Set a floor** — below 0.70 confidence triggers oversampling.
- **Still sample the rest** — high-confidence outputs enter at a lower rate.
- **Tune the ratio** — balance reviewer capacity against error tolerance.

Confidence stratification is the most common starting point for review pipelines.

### Three Stratification Dimensions

- **Confidence** — oversample outputs below your confidence floor, regardless of category or
  impact.
- **Category balance** — ensure rare output classes get reviewed proportionally, not drowned out
  by common ones.
- **Stakes** — route high-impact outputs through review even when the model is confident.

### Stratifying by Output Category

Category-based stratification protects against class imbalance in the queue.

- **Common drowns rare** — frequent classes fill the queue and hide critical ones.
- **Set quotas** — per-category sample limits guarantee coverage.
- **Rare carries risk** — rare categories often carry the most risk.

Without it, review quality just mirrors the model's existing distribution.

### Combining Strata Effectively

- **Confidence × category** — select items that are both low-confidence and from
  underrepresented categories, concentrating signal.
- **Confidence × stakes** — high-stakes outputs get reviewed even at high confidence, in a
  separate lane from the low-confidence pool.
- **All three combined** — full stratification uses confidence, category, and stakes to maximize
  error yield per hour.

### Reviewer Burnout and Signal Dilution

Too much correct output trains reviewers to approve without reading carefully.

- **Quality drops** — burnout degrades review even at constant volume.
- **Misses rise** — real errors get flagged less as fatigue accumulates.
- **Density helps** — stratification raises the error rate reviewers see.

A well-stratified queue keeps reviewers alert by making every review matter.

### Key Takeaways: Stratification Raises Review Yield

Random sampling wastes reviewer time on outputs the model already gets right.

- Stratify by confidence to oversample low-confidence outputs.
- Stratify by category to protect against class imbalance.
- Apply stakes weighting for high-impact outputs at any confidence.
- Combining all three maximizes errors caught per reviewer-hour.

Stratify to turn spot-checks into quality control.

---

## 23. Routing Through a Review Queue

Low-confidence outputs go to humans; high-confidence outputs flow straight through.

### Three Terms Anchor This Lesson

| Term | Definition |
|---|---|
| **Review queue** | A work surface where low-confidence outputs land for human inspection before downstream release. |
| **Reviewer action** | The three permitted actions — approve, reject, or edit — that close the human-in-the-loop cycle. |
| **Bypass policy** | A rule letting certain outputs skip the queue on confidence or urgency. |

### Queue Architecture Overview

The review queue sits between the model's output and its downstream consumer.

- **Low goes to queue** — low-confidence outputs are routed, not released.
- **High flows on** — high-confidence outputs continue downstream with no delay.
- **Held until acted** — the queue holds items until a reviewer acts.

Queue depth and latency are the two metrics to track from day one.

### What Reviewers See in the Interface

A good interface presents everything needed for a fast, accurate decision.

- **The input** — the original request that generated the output.
- **Output and score** — the model output and its confidence side by side.
- **A suggestion** — a proposed correction from a secondary system if available.

Reviewers should never leave the interface to find context for a decision.

### Approve vs. Edit vs. Reject

- **Approve or edit** — the reviewer confirms the output (approve) or corrects it (edit). Both
  release the item downstream, and an edit creates a training-quality correction pair.
- **Reject** — the output is discarded entirely. Reject fits when the output is wrong and the
  right answer is unclear, or when the item should not be released in any form.

### Four Queue Design Principles

- **Single action surface** — approve, edit, or reject without switching tools.
- **Context first** — input, output, and confidence visible before deciding.
- **Queue priority** — highest-stakes, lowest-confidence items at the top.
- **Audit trail** — log who reviewed, what changed, and when.

### Closing the Loop With Training Data

Reviewer corrections are the most valuable signal a model team can collect.

- **Approve confirms** — it validates the model's behavior on that input type.
- **Edit labels** — each correction is a labeled pair for training.
- **Reject flags** — it marks an input class the model handles poorly.

Feeding corrections back into training creates a continuous improvement cycle.

### Latency Tradeoff and User Experience

A review step adds latency that downstream users or systems must absorb.

- **Synchronous** — the user is blocked until a decision is made.
- **Asynchronous** — a placeholder returns now and is fulfilled later.
- **Or bypass** — some cases tolerate delay, others need a bypass path.

Design the latency model before deployment, not after the first complaint.

### When to Use the Bypass Policy

- **Emergency override** — time-critical cases where review delay causes more harm than the error
  risk.
- **High-confidence** — outputs above a validated confidence ceiling skip review, with logging.
- **Reviewer unavailable** — a fallback when no reviewers are active and operation must keep
  moving.

### Key Takeaways: Review Queues Turn Routing Into Quality

A review queue makes human-in-the-loop oversight scalable and auditable.

- Route low-confidence outputs to the queue before downstream release.
- One reviewer surface shows input, output, confidence, and suggestions.
- Approve, edit, and reject actions generate training signal every review.
- Bypass policies handle latency without removing accountability.

Corrections and audit trails make it more than a delay.

---

## 24. Every Claim Needs a Source

Fluent synthesis hides which sentence came from which document.

### Three Foundational Concepts

| Term | Definition |
|---|---|
| **Provenance tracking** | Attaching a verifiable source pointer to every claim so reviewers can confirm where each fact came from. |
| **Claim-to-source mapping** | An explicit structure linking each output claim to the source documents it derived from. |
| **Inline citation** | A source pointer placed next to a specific claim, not at the end of a paragraph. |

### The Black-Box Synthesis Problem

A synthesis model produces fluent prose, but each sentence's sources are invisible.

- **No way to check** — a reviewer cannot tell if a claim is accurate.
- **Errors compound** — mistakes spread across long documents undetected.
- **Audits need traces** — regulators require source traceability.

Without provenance, synthesis output is opinion dressed as analysis.

### Inline vs. Paragraph Citations

- **Inline citations** — source pointers appear next to each claim, so a reviewer can verify any
  sentence independently. More granular and higher overhead, but required for high-stakes output.
- **Paragraph lists** — sources appear at the paragraph level. Faster to read, but a reviewer
  cannot tell which source supports which sentence. Acceptable only for low-stakes summaries.

### Schema Design for Claim Mapping

Store each claim alongside the identifiers of every source that supports it.

```json
{
  "claim": "Revenue grew 12% in Q3",
  "source_ids": ["doc_4", "chunk_19"],
  "confidence": 0.88
}
```

This makes every claim a first-class auditable object.

### Why Provenance Tracking Matters

- **Trust** — stakeholders trust output they can spot-check against sources.
- **Auditability** — auditors trace any fact to its source without re-running the model.
- **Compliance** — GDPR Articles 13–15 and the EU AI Act require showing where conclusions came
  from.
- **Error correction** — a flagged claim traces to its source, so fixes stay local, not full
  re-synthesis.

### The Verification Workflow

A structured verification workflow lets reviewers sample claims and confirm them against sources
without reading everything.

- Select a random or risk-weighted sample of claims.
- Follow each claim's source pointers to the cited documents.
- Confirm the source text supports the synthesized claim.

Sample size depends on error rate, tolerance, and confidence, not a fixed percentage.

### Three More Critical Concepts

| Term | Definition |
|---|---|
| **Source identifier** | A unique, stable reference to a document, chunk, or passage used as evidence. |
| **Fabricated citation** | When a model invents a source that does not fit or does not exist. |
| **Sampling audit** | A review checking a subset of claims against their source pointers. |

### Fabricated Citations: The Hidden Risk

The most dangerous failure is fluent text that looks sourced but is not.

- **Wrong document** — the cited source does not support the claim.
- **Invented source** — the model fabricates a title or URL that does not exist.
- **Blind trust** — reviewers accept the citation without verifying it.

Fabrication is caught only through verification, not by reading the output alone.

### Key Takeaways: Source Mapping Foundations

Every synthesized claim should carry a pointer to the source that supports it.

- Provenance tracking makes synthesis output auditable and trustworthy.
- Claim-to-source schemas pair claim text with source identifiers.
- Inline citations enable per-claim verification; paragraph lists do not.
- Fabricated citations are the highest-risk provenance failure.

Synthesis without source pointers is unverifiable.

---

## 25. Conflict Annotation in Synthesis

When sources disagree, surface the conflict instead of silently picking one.

### Three Core Conflict Concepts

| Term | Definition |
|---|---|
| **Conflict annotation** | Showing in the output that sources disagreed on a fact, instead of silently selecting one. |
| **Resolution strategy** | An explicit rule for choosing among conflicting sources: prefer-newer, prefer-authoritative, or escalate. |
| **Averaging antipattern** | Blending or silently picking one of two conflicting claims, hiding the disagreement from the reader. |

### The Averaging Antipattern

When two sources disagree, the easy path is to pick one and hide the conflict.

- **No sign of dispute** — the reader sees a confident claim only.
- **Wrong one wins** — the dropped source may be more accurate or recent.
- **False certainty** — downstream decisions rest on it.

Silent averaging is the most common failure mode in multi-source synthesis.

### Conflict Detection: Finding Disagreements

Detection identifies when two sources make incompatible claims about one fact.

- **Same topic** — semantic similarity groups claims about one subject.
- **Contradiction** — entailment checks test whether claims conflict.
- **Flag the pair** — high similarity but low agreement marks a conflict.

Detection happens before synthesis, not after; early flags drive better output.

### Detected vs. Undetected Conflict

- **Detected conflict** — the synthesis notes that Source A says X while Source B says Y,
  difference unresolved. The reader knows the disagreement exists and the output is honest about
  uncertainty.
- **Undetected conflict** — the synthesis presents one source's claim as settled fact and
  silently drops the other. The reader has false confidence and no way to know a dispute exists.

### Resolution Strategies

- **Prefer-newer** — use the claim from the more recent document; good when facts evolve over
  time.
- **Prefer-authoritative** — weight designated authoritative sources over secondary ones, by a
  defined ranking.
- **Escalate-to-human** — send the conflict to a reviewer when neither newer nor authoritative
  resolves it.

### Recording Conflicts After Resolution

When a conflict is resolved, preserve the trail once a winner is chosen.

```json
{
  "claim_a": "rate 5.4%",
  "claim_b": "rate 7.7%",
  "rule": "prefer_newer",
  "chosen": "claim_b"
}
```

A logged trail shows a decision was made, not that consensus existed.

### Three Supporting Concepts

| Term | Definition |
|---|---|
| **Conflict log** | A persistent record of each disagreement, the strategy applied, and the outcome versus the alternative. |
| **Entailment check** | A semantic test of whether one passage contradicts, supports, or is neutral to another. |
| **Source authority ranking** | A predefined hierarchy of trust levels used by the prefer-authoritative strategy. |

### Surfacing Conflicts for Readers

Annotation balances transparency with readability: overwhelming the reader defeats it.

- **High stakes** — surface conflicts inline at the claim level.
- **Medium risk** — use footnotes or expandable sections.
- **Full review** — aggregate a conflict summary at the document end.

The goal is informed readers, not exhausted ones.

### Key Takeaways: Conflict Annotation Principles

Synthesis that hides disagreement is a choice the reader cannot see.

- The averaging antipattern picks one source and drops conflicting evidence.
- Conflict detection finds incompatible claims before synthesis runs.
- Resolution strategies give explicit rules: newer, authoritative, or escalate.
- Conflict logs preserve the trail even after resolution.
