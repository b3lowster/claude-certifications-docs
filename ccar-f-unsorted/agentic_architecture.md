# Agentic Architecture — Day 1

*Text extracted from the 303-slide deck and grouped by topic. Diagrams and decorative
elements are omitted; only the substantive text is kept.*

---

## Table of Contents

1. [Agents, Workflows, and Chatbots](#1-agents-workflows-and-chatbots)
2. [Tool Use as the Foundation of Agency](#2-tool-use-as-the-foundation-of-agency)
3. [The Four-Phase Agentic Loop](#3-the-four-phase-agentic-loop)
4. [Loop Control: Stop, Iterate, Escalate](#4-loop-control-stop-iterate-escalate)
5. [When Workflows Outperform Agents](#5-when-workflows-outperform-agents)
6. [Choosing the Right Architecture](#6-choosing-the-right-architecture)
7. [Breaking Goals Into Subtasks](#7-breaking-goals-into-subtasks)
8. [Decomposition Patterns](#8-decomposition-patterns)
9. [Sequential Pipelines: Design and Tradeoffs](#9-sequential-pipelines-design-and-tradeoffs)
10. [Parallel Execution: Fan-Out and Fan-In](#10-parallel-execution-fan-out-and-fan-in)
11. [Adaptive Planning Mid-Execution](#11-adaptive-planning-mid-execution)
12. [Ambiguity and Incomplete Specifications](#12-ambiguity-and-incomplete-specifications)
13. [The Orchestrator Role](#13-the-orchestrator-role)
14. [Subagent Design: Scope and Authority](#14-subagent-design-scope-and-authority)
15. [Hub-and-Spoke vs. Pipeline vs. Peer-to-Peer](#15-hub-and-spoke-vs-pipeline-vs-peer-to-peer)
16. [Selecting the Right Topology](#16-selecting-the-right-topology)
17. [Designing Agent-to-Agent Schemas](#17-designing-agent-to-agent-schemas)
18. [Handoff Protocols and Agent Continuity](#18-handoff-protocols-and-agent-continuity)
19. [In-Context State vs. External Memory](#19-in-context-state-vs-external-memory)
20. [Session Continuity Across Turns](#20-session-continuity-across-turns)
21. [Multi-Agent Cost and Fit](#21-multi-agent-cost-and-fit)
22. [Tool, Reasoning, and Environment Errors](#22-tool-reasoning-and-environment-errors)
23. [Error Detection and Validation](#23-error-detection-and-validation)
24. [Retry Logic](#24-retry-logic)
25. [Fallback Chains and Graceful Degradation](#25-fallback-chains-and-graceful-degradation)
26. [Why Prompts Aren't Enough](#26-why-prompts-arent-enough)
27. [Programmatic Validation Layers](#27-programmatic-validation-layers)
28. [Hook Lifecycle Events](#28-hook-lifecycle-events)
29. [Escalation Trigger Design](#29-escalation-trigger-design)
30. [Interruption Points and Review Workflows](#30-interruption-points-and-review-workflows)

---

## 1. Agents, Workflows, and Chatbots

Drawing clear lines between agents, workflows, and conversational chatbots — and when an
agentic architecture is the right design choice. The progression from chatbot to workflow to
agent is one of increasing autonomy.

### The Augmented LLM Building Block

**Augmented LLM.** A base language model enhanced with retrieval, tools, and memory. It's the
foundational building block every agentic system is composed from.

- **Retrieval and tools.** The model generates its own search queries and selects the tools it
  needs, rather than waiting for a fixed script to hand it results.
- **Memory.** The model decides what information to retain across steps, so context carries
  forward instead of resetting on every call.

### Three Core Distinctions

| Term | Definition |
|---|---|
| **Agentic system** | A language model autonomously pursues a goal by selecting actions, invoking tools, and iterating until the goal is reached. |
| **Workflow (deterministic)** | A fixed, developer-authored sequence of steps that runs predictably. The control flow is decided up front at design time. |
| **Autonomous agent** | An agent that operates without step-by-step human direction, making its own decisions to reach the goal. |

### What a Chatbot Actually Does

- A chatbot takes your message, processes it with an LLM, and generates a response.
- It predicts the next most likely tokens based on patterns in its training data.
- The output is text — nothing more.

**What it cannot do:** access files · call APIs · query databases · run code · persist state.

### Workflow vs. Agentic System

- **Deterministic workflow** — follows a predefined sequence of steps: input/trigger → step 1 →
  step 2 → output.
- **Agentic system** — adapts and iterates to reach a goal: reason → execute tool → check
  result → repeat until the goal is met.

### Why LLMs Alone Are Not Agents

- No direct access to files, APIs, or the real world.
- Cannot execute code or perform actions.
- No persistent memory between interactions.
- Cannot observe results or learn from feedback.

On its own the model is text in, text out — cut off from APIs and the web, databases, and code
execution.

### The Four Pillars of Agency

Agents perceive, decide, act, and learn — continuously.

1. **Perception** — senses and gathers information from its surroundings and internal state.
2. **Selection** — reasons about the options and selects the best next action to pursue its goal.
3. **Execution** — carries out the selected action using tools, APIs, or other systems to make
   change.
4. **Iteration** — learns from outcomes, updates its plan, and repeats until the goal is reached.

Agency emerges from the continuous loop of these four pillars.

### Claude's Tool System at a Glance

Claude can reason, decide to use tools, act, and learn from the results.

1. User prompt → 2. Claude LLM → 3. Tool call → 4. Runtime → 5. External tool →
6. Observation → 7. Claude LLM → 8. Final response

**Example tool schema**

```json
{
  "name": "get_weather",
  "description": "Get the current weather",
  "parameters": { "location": "string" }
}
```

**Example tool call**

```json
{
  "name": "get_weather",
  "arguments": { "location": "San Francisco" }
}
```

**Example observation**

```json
{
  "temperature": 18,
  "unit": "celsius",
  "condition": "Partly cloudy"
}
```

This loop continues until Claude has achieved the user's goal.

### Three More Patterns to Know

Different ways to extend LLMs beyond text generation:

- **Tool use** — LLMs call external tools and use the results in their reasoning.
- **Prompt chaining** — break complex tasks into a sequence of smaller LLM calls, each step
  building on the last (LLM 1 → LLM 2 → LLM 3).
- **Action–environment gap** — LLMs operate in text space while real-world actions happen
  outside. Tool use bridges that gap (LLM output → real world).

These patterns are not mutually exclusive — effective agents combine them.

### When to Choose an Agentic Architecture

Not every problem needs an agent. Start by understanding the trade-offs.

- **Choose a workflow** when steps are predictable and auditing matters most.
- **Choose an agent** when the path to the goal cannot be scripted in advance.

Higher autonomy means higher capability — and a higher risk surface. Use a minimal footprint
and human-in-the-loop checkpoints.

### Key Takeaways: What Makes a System Agentic

An agentic system has model-controlled flow at inference time.

- **Chatbots** respond with text — they do not act.
- **Workflows** are deterministic — the author defines the flow.
- **Agents** are dynamic — the model selects tools while running.
- Tool use bridges the action–environment gap.
- Match architecture to task predictability and risk tolerance.

---

## 2. Tool Use as the Foundation of Agency

Tool use turns a language model into a system that acts in the world. This module traces the
full tool-call lifecycle, shows how description quality drives selection, and draws the line
between scripted and model-driven tool use to locate the true agentic boundary.

### The Building Blocks

| Component | Definition |
|---|---|
| **Tool definition** | A schema passed at inference time — name, description, and parameter spec — that the model reads to decide whether to invoke the tool. |
| **Tool call** | The structured output emitted when the model decides to use a tool: tool name plus JSON arguments the runtime will execute. |
| **Observation** | The result returned after a tool call executes, injected into context so the model can plan its next step. |

### The Tool Call Lifecycle

Every tool interaction follows the same four-phase sequence from model output to observation:

1. The model emits a structured tool-call output, not plain text.
2. The runtime intercepts the call and runs the underlying function.
3. The result is captured and formatted as an observation.
4. The observation is added to the next context window so the model can read it.

The model then updates its plan based on what just happened.

### Pre-Scripted vs. Model-Driven Tool Use

| Pre-scripted (workflow) | Model-driven (agentic) |
|---|---|
| The developer decides which tools run, and their order, at design time. | The model selects tools at inference time based on context and goal. |
| The model may fill in parameters but never selects the tool. | The sequence varies across runs. |
| Predictable and auditable when the task path is known. | Higher adaptability, higher risk, and requires robust guardrails. |

### Description Quality Drives Tool Selection

Claude selects tools using natural-language descriptions in the schema, not by reading code.

- Claude reads the `description` field to decide whether and when to call a tool.
- The quality of that text directly determines selection reliability.
- Write descriptions a non-engineer could act on.
- Poor descriptions are among the most common causes of unreliable agentic behavior.

> **Ambiguous:** "Does something with data" — the model cannot reliably decide when to call this.
>
> **Precise:** "Queries inventory by SKU, returns stock count" — the model knows exactly when and
> why to invoke this tool.

### Common Tool Categories in Practice

Most agentic systems compose tools from three broad categories:

- **Data access** — read files, query databases, fetch web content, or retrieve documents from a
  knowledge base.
- **Computation** — execute code, run calculations, parse structured data, or call a sandboxed
  interpreter for derived values.
- **External APIs** — write to outside services: send messages, create calendar events, update
  records, or trigger workflows.

### How Observations Update the Model's Plan

The observation phase is what makes the agentic loop self-correcting.

- A successful result confirms progress — the model moves to the next step.
- An error or unexpected result triggers replanning.
- Partial results may prompt a follow-up tool call before proceeding.

This feedback loop distinguishes an agent from a pipeline — the model responds to what it learns.

### Three Runtime Concepts

How the system manages tool execution at inference time:

- **Tool result injection** — formatting a tool's output as a context message so the model can
  incorporate the observation into its next response.
- **Parallel tool calls** — the model emits several tool-call outputs in one response. The
  runtime executes them concurrently and returns all observations.
- **Tool call loop** — the iterative cycle of tool call, execution, observation, and model
  response that runs until the agent hits its goal or a stop condition.

### Practical Examples Across Tool Types

Concrete tool examples make agentic design trade-offs easier to reason about.

| Tool type | Behavior | Failure modes |
|---|---|---|
| **Web search** | Model calls the search tool, receives ranked results as observation. | Stale index, throttling, irrelevant results. |
| **File read / write** | Model reads a document, edits content, and saves it. | File not found, permission errors, encoding issues. |
| **Code execution** | Model generates code, sandbox runs it, stdout as observation. | Runtime errors, timeouts, sandbox limits. |

Each tool type has distinct failure modes, latency, and security surface.

### Key Takeaways: Tool Use as the Foundation of Agency

Tool use turns an LLM's text output into real-world action.

- **Lifecycle** — structured output, then runtime execution, then an observation returns.
- **Description quality** — drives selection; write for the model, not engineers.
- **Pre-scripted** — fixed sequences are workflows; model-selected are agentic.
- **Observations** — feed back into context, enabling replanning.

Tools are the architectural primitive that defines agency.

---

## 3. The Four-Phase Agentic Loop

Every agent runs on a repeating cycle — Perception, Reasoning, Action, and Observation. This
module walks through each phase, how they feed one another, and why this loop defines agentic
systems.

### Core Loop Vocabulary

| Term | Definition |
|---|---|
| **Agentic loop** | The repeating perceive–plan–act–observe cycle. The model reads context, selects an action, receives a result, and decides whether to continue or stop. |
| **Tool use** | The capability to invoke external functions or APIs by emitting structured tool-call outputs the runtime executes and returns as observations. |
| **Dynamic planning** | A capability where the plan is generated or modified at runtime based on intermediate results. Steps are not predetermined. |

### Phase 1 — Perception

Perception is how the agent reads and interprets its available context.

- The model scans the full context window: instructions, history, tool results.
- It identifies the current goal state and what information is available.
- Prior observations and partial results are inputs to this phase.

Everything the agent knows right now is held in the context window.

### Phase 2 — Reasoning

Reasoning is the model's decision process — given what it perceives, what should it do next?

- The model selects the next action from available tools or outputs.
- It weighs intermediate results against the original goal.
- Dynamic planning lets the decision change as new information arrives.

Reasoning is not predetermined — it adapts at every turn.

### Action vs. Observation

- **Action** — the agent emits a structured tool-call or final output. This is the step where it
  interacts with the external world: invoking APIs, running code, or writing a response.
- **Observation** — the runtime returns the tool result back into the context. The agent reads
  this result, updates its understanding, and feeds it into the next perception phase.

### Context Window as Working Memory

The context window holds four kinds of content:

- **Inputs** — instructions, user message, and any initial documents loaded at session start.
- **Actions** — each tool call emitted by the agent during the current session.
- **Observations** — tool results returned by the runtime after each action executes.
- **Reasoning** — intermediate model outputs that capture planning and decision rationale.

### How Each Phase Feeds the Next

The loop is cyclic — each phase's output becomes the next phase's input.

- Perception surfaces context → Reasoning.
- Reasoning selects an action → Action.
- Action generates an observation → Observation.
- Observation returns context → Perception.

This cycle repeats until the agent reaches a termination condition.

### Memory and State Concepts

| Term | Definition |
|---|---|
| **Agentic system** | An architecture where a language model pursues a goal by perceiving context, selecting actions, and iterating until a termination condition is met. |
| **In-context memory** | State stored in the active context window. Immediately accessible but ephemeral — lost when the session ends or context is cleared. |
| **Tool result** | The structured data returned after a tool call runs. It becomes an observation the agent incorporates into its next decision. |

### Real Example — Multi-Step Research Agent

A research agent illustrates all four phases working together in sequence:

- **Perception** — reads the query and prior results.
- **Reasoning** — decides which search query to run next.
- **Action** — queries the search API.
- **Observation** — returns ranked results to context.

The returned results become the next observation, and the loop continues.

### Key Takeaways: The Agentic Loop

The four phases drive autonomous, iterative agent execution. This loop is foundational to
designing and debugging agentic systems.

- Perception reads context; Reasoning picks the next step.
- Action invokes tools; Observation returns results to context.
- The context window is the agent's ephemeral working memory.
- Each phase feeds the next in an adaptive cycle.

---

## 4. Loop Control: Stop, Iterate, Escalate

Knowing when to stop the loop, retry, or hand off is what makes an agent reliable. This module
covers termination conditions, turn budgets, infinite loop risks, and the choice between
iterating and escalating to a human or fallback.

### Naming the Canonical Loop

Anthropic frames the agent loop as three blended phases that repeat until the job is done:

1. **Gather context** — pull in the information needed to decide the next move.
2. **Take action** — use a tool to actually change or query the world.
3. **Verify work** — check the result against the goal before continuing.

Loop control is simply deciding when this gather–act–verify cycle should stop.

### Loop Control Vocabulary

| Term | Definition |
|---|---|
| **Termination condition** | An explicit rule that stops the loop: task done, error threshold hit, or a stop signal. |
| **Turn budget** | A hard cap on loop iterations, enforced by the runtime to stop runaway execution. |
| **Escalation** | A controlled handoff to a human or fallback when the agent can't safely proceed. |

### Four Termination Conditions

- **Task complete** — the agent has achieved its goal and produced final output.
- **Error threshold** — too many consecutive failures; the agent stops rather than cycling further.
- **Budget exhausted** — turn limit or token budget reached. Execution halts programmatically.
- **Stop signal** — orchestrator or user explicitly instructs the agent to halt immediately.

### Infinite Loop Risks

Without explicit termination conditions, agents can loop forever.

- **Missing stop condition** — the agent never declares the task complete.
- **Always-failing tools** — a broken API call retried endlessly.
- **Circular reasoning** — the model revisits the same decision without new information.

An infinite loop is a design failure, not a feature.

### Turn Budgets and Max-Iteration Guards

Turn budgets are the primary programmatic defense against runaway loops.

- Set a `max_iterations` limit before the agent begins.
- Track iteration count in the orchestration layer, not the model.
- When the limit is reached, halt and surface whatever progress was made.

Guards belong in the runtime, not left to the model to self-enforce.

### Progress vs. Spinning

| Progress | Spinning |
|---|---|
| Each iteration produces a new observation, advances toward the goal, or closes an open question. | The agent repeats similar actions without new results. |
| The context window shows forward movement: new information, completed sub-steps, a tighter focus. | Tool calls return the same errors. The plan doesn't change across iterations. Context grows but the goal state doesn't get closer. |

### Detecting Spinning in Practice

Spinning is often subtle and requires deliberate detection logic.

- Compare successive observations: are they meaningfully different?
- Track repeated tool calls with identical parameters.
- Monitor the gap between goal state and current state across turns.

If the gap isn't closing, the agent is likely spinning.

### Iterate vs. Escalate — The Decision

When blocked, the agent can retry with new information or hand off to a human.

- **Iterate** — when the agent has new data or another way to try.
- **Escalate** — when at an authorization boundary or lacking needed information.

Escalation is a designed safety valve, not a failure.

### Loop Control as Reliability Design

Loop control isn't just a performance concern — it's a reliability concern.

- Runaway agents can burn through API quotas and incur unexpected costs.
- Agents that don't escalate can make unrecoverable decisions on their own.
- Proper termination conditions make agent behavior predictable and auditable.

Reliability requires that every agent has a defined exit condition from day one.

### Key Takeaways: Loop Control

Termination conditions, turn budgets, and escalation paths are essential safeguards for
dependable agents.

- **Termination** — set explicit stop rules before deploying any agent.
- **Turn budgets** — cap iterations with a runtime turn budget.
- **Progress check** — distinguish spinning from progress by comparing successive observations.
- **Escalate** — escalate when the agent can't proceed rather than cycling indefinitely.

Loop control turns a capable agent into a dependable one.

---

## 5. When Workflows Outperform Agents

Deterministic workflows deliver lower cost, higher reliability, and easier auditing than agents
for many problems.

### Three Core Contrasts

| Term | Definition |
|---|---|
| **Workflow (deterministic)** | A fixed, developer-authored sequence of steps that executes predictably. The control flow is decided up front at design time, not while the model is running. |
| **Agentic system** | An architecture in which an LLM autonomously pursues a goal by selecting actions and invoking tools until a termination condition is met. |
| **Prompt chaining** | A workflow pattern passing one LLM call's output as input to the next, creating a sequential pipeline with narrow, individually testable steps. |

### The Core Tradeoff

Every architecture decision starts with one question: **does this task need runtime judgement?**

The axis runs from predictability to flexibility.

- Workflows trade adaptability for predictability.
- Agents trade predictability for open-ended flexibility.
- Most real tasks fall closer to the workflow end.

Choosing agents when workflows suffice adds cost and risk without adding value.

### Workflow vs. Agent

| Workflow | Agent |
|---|---|
| Fixed control flow. | Dynamic, adaptive control flow, with replanning. |
| Predictable, auditable output. | Higher capability ceiling. |
| Lower latency and token cost. | Higher latency, higher token cost. |
| Each step is individually testable, making failures easy to localize and fix. | Broader failure surface and harder to audit. |

### Signals a Workflow Is Enough

Before reaching for an agent, check whether these conditions are true:

- Input format is predictable and well-defined.
- Output format is known and testable.
- Every step can be specified at design time.

When all three hold, a workflow is the right call.

### Five Core Workflow Patterns

1. **Prompt chaining**
2. **Routing**
3. **Parallelization**
4. **Orchestrator–subagent**
5. **Evaluator–optimizer** (with a feedback loop)

### Routing and Parallelization

Two workflow patterns handle real complexity without requiring full agentic autonomy.

- Routing uses a classifier to sort inputs before processing.
- The router may call an LLM, but the topology stays fixed.
- Parallelization fans out work and collects results at a sync point.

Both patterns scale well and remain fully auditable.

### Cost and Reliability Levers

- **Token cost** — workflows make fewer unplanned LLM calls. Each step is scoped, so token usage
  is predictable and budgetable across every task run.
- **Latency** — workflow steps can be pipelined or parallelized deliberately. Agents introduce
  variable latency from replanning loops you didn't anticipate.
- **Reliability** — deterministic steps are individually testable. A workflow failure localizes
  to one step rather than spreading across a long reasoning trace.

### When NOT to Over-Index on Agents

Agent overhead is only justified when the task genuinely cannot be pre-scripted.

- Agents increase token spend per task run.
- Every autonomous decision is a failure surface.
- Debugging agentic traces is harder than workflow logs.

Reach for agents only when workflows demonstrably fall short.

### Key Takeaways: When Workflows Win

Workflows beat agents when tasks are predictable and steps can be authored at design time.

- Five core patterns: chaining, routing, parallelization, orchestrator–subagent,
  evaluator–optimizer.
- Workflows offer lower cost, lower latency, and easier auditing.
- Agent complexity is warranted only when workflows fail.
- Tool use alone does not make a system agentic.

Match architecture to task requirements, not to capability maximalism.

---

## 6. Choosing the Right Architecture

Selecting between workflow and agent should follow a structured process, not intuition.

### Framework Vocabulary

| Term | Definition |
|---|---|
| **Incremental complexity** | Start with the simplest architecture that solves the problem, then add agent behavior only where workflows demonstrably fall short. |
| **Orchestrator** | The controlling component that decomposes a high-level goal, assigns subtasks to subagents or tools, and synthesizes the final result. |
| **Over-engineering** | Adding complexity, such as agentic autonomy, that adds cost and risk without proportional benefit. |

### Start With the Simplest Architecture

The first rule of architectural choice: solve the problem with the least complexity required.
The ladder runs prompt → chain → workflow → agent.

- A simple prompt chain beats a complex agent for scoped tasks.
- Simplicity reduces cost, latency, and failure surface.
- You can always add agent behavior later.

Starting complex and simplifying later is much harder.

### Signals That Agents Are Warranted

Some tasks genuinely require agentic complexity. Watch for these indicators:

- The goal is open-ended and cannot be fully pre-specified.
- Tool sequences are unpredictable at design time.
- The task requires dynamic replanning mid-execution.

If none of these apply, a workflow is likely sufficient.

### Reliability vs. Adaptability

| Reliability (workflow) | Adaptability (agent) |
|---|---|
| Deterministic, testable steps with localized, diagnosable failure and predictable token spend. | Dynamic behavior handles open-ended goals and unpredictable task sequences. |
| Straightforward to monitor, maintain, and audit on every run. | Higher capability ceiling, but variable cost, harder to audit, and a broader failure surface. |

### Three Tradeoff Axes

- **Cost** — agents spend more tokens per run. Justify that spend only with capability you
  genuinely need today.
- **Auditability** — workflows are step-by-step auditable. Agents trade auditability for the
  runtime flexibility they require.
- **Reliability** — workflows fail predictably at known steps. Agents adapt but introduce
  variable, harder-to-reproduce failure modes.

### Incremental Complexity in Practice

The right way to add agent behavior is incrementally, not all at once.

- Build and validate the workflow version first.
- Identify the specific step where the workflow breaks down.
- Replace only that step with agent behavior.

This keeps the system as simple as the task allows.

### Three Decision Scenarios

- **Workflow scenario** — extract structured data from a fixed document. Input is predictable,
  output schema is known. A prompt and validation are fully sufficient.
- **Agent scenario** — research a topic from open-ended web sources. Goal is unbounded, tool
  sequences are unpredictable. An agent with planning is the right fit.
- **Hybrid scenario** — route inquiries to specialists, then synthesize responses. Fixed routing
  topology plus a dynamic synthesis step calls for a workflow feeding an agent.

### Evaluating Your Own Task

Apply the framework to any new task with four diagnostic questions:

1. Can every step be authored at design time?
2. Is the result format known and testable?
3. Does the task require runtime tool selection?
4. Does failure require dynamic recovery mid-run?

Two or more "no" answers suggest agentic complexity is warranted.

### Key Takeaways: Choose the Right Architecture

Start simple, add complexity only where workflows fail, and evaluate every tradeoff explicitly.

- Incremental complexity keeps systems maintainable and debuggable.
- Agents are warranted for open-ended goals and dynamic replanning.
- Weigh the tradeoff axes: cost, auditability, reliability, adaptability.
- Hybrid architectures combine workflow structure with targeted agent behavior.

Match architecture to what the task actually demands.

---

## 7. Breaking Goals Into Subtasks

Task decomposition is the skill of breaking complex goals into actionable subtasks.

### Three Core Decomposition Terms

| Term | Definition |
|---|---|
| **Task decomposition** | Breaking a high-level goal into smaller subtasks that can be assigned to agents, tools, or workflow steps. |
| **Handoff point** | The boundary between subtasks where one step's output becomes the next step's input. Requires an explicit schema. |
| **Orchestrator** | The controlling agent that decomposes goals, assigns subtasks, manages execution order, and synthesizes results. |

### Why Decomposition Matters

Agentic systems require explicit structure — agents cannot infer what a complex goal demands.

- Large monolithic tasks are hard to retry on failure.
- Smaller subtasks allow targeted error recovery.
- Clear boundaries let multiple agents work together.

Decomposition is the architectural skill that makes complex workflows manageable.

### The Decomposability Spectrum

Tasks fall on a spectrum from monolithic to over-decomposed — both extremes cause problems.

- **Monolithic** — one large task, hard to retry, hard to delegate.
- **Well-decomposed** — clear subtasks with defined inputs and outputs.
- **Over-decomposed** — too many tiny steps create coordination overhead.

The goal is the middle ground where each subtask is meaningful and executable.

### Monolithic vs. Decomposed Tasks

| Monolithic task | Decomposed task |
|---|---|
| One agent handles everything. | Goal split into well-defined subtasks. |
| Simple to assign at first, but hard to debug. | Each piece can be retried, delegated, and tested on its own. |
| Fails completely on any error, and is hard to run in parallel or retry in parts. | Upfront design cost is offset by long-term reliability. |

### Analyzing a Goal for Decomposition

Before splitting a goal, you need to understand its natural structure and dependencies:

1. Can any part run independently of the rest?
2. Does one step's output feed directly into another?
3. Are there distinct capability domains that need different agents?

These questions reveal where decomposition lines should be drawn.

### What Makes a Good Subtask

- **Single responsibility** — each subtask does one thing and has a clear, testable output that
  another step can consume.
- **Defined inputs** — the subtask knows exactly what context and data it receives; nothing is
  assumed from ambient state.
- **Bounded scope** — scope is contained enough to retry on failure without re-running the whole
  job.
- **Clean output** — output is structured and explicit, so the orchestrator can route results
  without parsing raw text.

### Designing Handoff Points

Handoff points are the interfaces between subtasks — they determine how information flows
through a workflow.

- Pass only what the next step needs — minimize payload size.
- Use structured schemas so subagents don't misinterpret context.
- Document what is *not* passed to prevent false assumptions.

Well-designed handoffs prevent context loss between workflow stages.

### The Cost of Decomposition

Decomposition isn't free — every boundary you add introduces coordination overhead.

- More subtasks means more orchestration logic and latency.
- Each handoff is a potential failure point in the chain.
- Deeply decomposed systems are harder to trace and debug.

Balance reliability gains against the cost of more moving pieces.

### Key Takeaways: Task Decomposition Foundations

Decomposition is the skill of turning complex goals into structured, manageable subtasks.

- Monolithic tasks fail completely; over-decomposed tasks waste coordination budget.
- Good subtasks have single responsibility, defined inputs, and clean outputs.
- Handoff points require explicit schemas to prevent context loss.
- Every boundary added costs coordination overhead — balance granularity carefully.

---

## 8. Decomposition Patterns

Three decomposition patterns appear in almost every agentic system: hierarchical, sequential,
and parallel.

### The Three Patterns

| Pattern | Definition |
|---|---|
| **Hierarchical decomposition** | Goals split into subgoals across multiple levels, forming a tree: orchestrators delegate down; results flow back up. |
| **Sequential execution** | Tasks performed in order, where each step may depend on the previous step's output before proceeding. |
| **Parallel execution** | Independent subtasks run concurrently, reducing total runtime. Requires a synchronization point to collect results. |

### Hierarchical Decomposition Explained

Hierarchical decomposition builds a goal tree: goals break into subgoals, subgoals into tasks.

- The top-level orchestrator holds the overall goal and delegates subgoals.
- Each level handles only its tier of complexity.
- Results flow back up through the hierarchy to be synthesized.

This mirrors how human organizations delegate work across management layers.

### When to Go Deep vs. Flatten

Hierarchy depth is a design choice — deeper trees enable specialization but increase
coordination overhead.

- Go deeper when subtasks require fundamentally different capabilities.
- Flatten when subtasks are similar enough to share one agent.
- Each added level increases the error propagation surface.

Flatten hierarchies wherever possible without losing the benefits of specialization.

### Sequential Decomposition Explained

Sequential decomposition creates a dependency chain where each step's output becomes the next
step's input.

- Step order is enforced because downstream steps need upstream output.
- Easier to reason about and debug than parallel workflows.
- Any failed step halts the chain until it's resolved.

Sequential patterns are appropriate whenever genuine data dependencies exist between steps.

### Sequential vs. Parallel Execution

| Sequential execution | Parallel execution |
|---|---|
| Tasks run one at a time in a defined order. | Independent tasks run concurrently, cutting total wall-clock time. |
| Safe for steps with data dependencies. | Requires a fan-in synchronization step. |
| Simpler to debug and trace. | Needs explicit handling for partial-failure scenarios. |
| Total time equals the sum of all individual step durations. | |

### Parallel Decomposition Explained

Parallel decomposition finds independent pieces of work with no data dependencies and runs them
concurrently to save time.

- Each parallel branch must be truly independent — no shared state.
- A fan-in step collects all branch results before continuing.
- Partial failure: some branches succeed, others fail — requires explicit handling.

This pattern is powerful, but it introduces failure scenarios that sequential patterns avoid.

### Combining Patterns in Real Systems

- **Hierarchical + sequential** — orchestrator delegates to subagents that each run a sequential
  pipeline internally. Common in multi-stage document workflows.
- **Hierarchical + parallel** — orchestrator fans out to multiple specialized subagents running
  concurrently. Used in multi-source research.
- **Sequential + parallel** — pipeline stages run sequentially, but within each stage independent
  subtasks run in parallel to reduce latency.

### Decomposition Anti-Patterns

Three anti-patterns consistently cause agentic system failures in production environments:

- **Over-decomposition** — so many steps that coordination costs exceed execution costs.
- **Under-decomposition** — tasks too large to retry or delegate reliably.
- **False parallelism** — treating dependent steps as independent, causing data hazards.

Recognizing these anti-patterns during design prevents expensive failures at runtime.

### Key Takeaways: Decomposition Patterns at a Glance

Three patterns — hierarchical, sequential, and parallel — cover most agentic decomposition needs.

- **Hierarchical** — goal trees with orchestrators delegating across levels.
- **Sequential** — dependency chains where output feeds the next step.
- **Parallel** — concurrent independent branches with fan-in synchronization.
- Real systems combine all three patterns within the same workflow.

Knowing which pattern fits which situation is the core design skill for agentic architectures.

---

## 9. Sequential Pipelines: Design and Tradeoffs

Sequential pipelines run each step in a defined order, passing outputs forward as inputs.

### Three Pipeline Concepts

| Term | Definition |
|---|---|
| **Sequential execution** | Tasks run one at a time in order. Each step may use the earlier output. Easy to debug, harder to speed up. |
| **Prompt chaining** | One LLM call's output becomes the next call's input. Each step has a single, narrow job. |
| **Dependency graph** | A map of which steps depend on which others. Reveals required ordering and steps that could run in parallel. |

### Sequential Pipeline Structure

A sequential pipeline is a chain where each step's output feeds directly into the next.

- Step one produces a result, step two consumes it as its input.
- Execution order is fixed and deterministic — no branching by default.
- The pipeline halts if any step fails — there's no parallel fallback.

This structure is the foundation of prompt chaining in agentic workflows.

### Dependency Graphs — Mapping Order

A dependency graph identifies which steps must come before which others.

- Draw an edge from step A to step B if B needs A's output to run.
- Steps with no incoming edges can run first — or potentially in parallel.
- Steps with multiple incoming edges are synchronization points.

Reading the dependency graph reveals the minimum required sequential ordering.

### When to Use Sequential

- **Data dependency** — step B cannot start until step A finishes. The two cannot run at once.
- **State accumulation** — each step builds on context that earlier steps produced.
- **Ordered transforms** — operations run in a strict order, one stage at a time.
- **Simplicity goal** — debugging and auditability matter more than raw speed.

### Simplicity vs. Latency Accumulation

| Sequential strengths | Sequential costs |
|---|---|
| Easy to debug step by step. Clear data flow. | Total latency equals the sum of every step's latency. |
| Failures are isolated and easy to trace. | Independent steps that could run concurrently are forced to wait. |
| No synchronization logic to manage. | Wasted wall-clock time when data dependencies don't actually exist. |
| Correct when steps have true data dependencies. | |

### Designing Clean Handoff Schemas

Each step boundary needs a defined schema for what passes from one step to the next.

- The output schema of step N becomes the input contract for step N+1.
- Explicit schemas catch mismatches early — before runtime failures.
- Include required fields, types, and nullability in every handoff spec.

Loose handoffs cause silent data loss and hard-to-diagnose downstream errors.

### Error Cascades in Sequential Pipelines

A failure at any step blocks every downstream step depending on its output.

- Step three can't run if step two failed — it has no input to use.
- Cascade risk grows with depth: more steps, more failure surface.
- Early validation at step entry reduces how far bad data travels.

Design error handling at each step boundary, not only at the pipeline exit.

### True vs. Artificial Dependencies

Not every sequential pipeline has genuine data dependencies between its steps.

- **True dependency** — step B reads a field produced by step A; ordering is required.
- **Artificial dependency** — steps are sequential only by convention, not necessity.
- Removing artificial dependencies lets independent steps run at the same time.

Audit your pipeline before committing to sequential ordering throughout.

### Key Takeaways: Sequential Pipeline Foundations

Sequential pipelines are ordered, deterministic, and ideal for data-dependent work.

- Dependency graphs reveal required ordering and potential parallelism.
- Clean handoff schemas prevent silent data loss at step boundaries.
- Failures cascade — handle errors at each step, not just at the end.
- Sequential is the right choice for data-dependent, state-building workflows.
- Audit for artificial dependencies before locking in a sequential order.

---

## 10. Parallel Execution: Fan-Out and Fan-In

Parallel execution runs independent tasks at the same time, cutting overall runtime.

### Three Parallel Execution Terms

| Term | Definition |
|---|---|
| **Parallel execution** | Multiple independent tasks run concurrently, reducing wall-clock time. Requires a synchronization point to collect outputs and handle partial failures. |
| **Fan-out** | The step where a single task splits into multiple concurrent subtasks dispatched simultaneously. |
| **Fan-in** | The synchronization step that collects and aggregates results from parallel branches. Handles partial failures. |

### Fan-Out: Spawning Parallel Branches

Fan-out is the dispatch step: one trigger spawns multiple concurrent subtasks simultaneously.

- All branches are dispatched at the same time from a single control point.
- Each branch runs independently with its own inputs and context.
- Branches can be agents, tool calls, or subworkflows — not just simple functions.

Fan-out increases throughput but requires every branch to be truly independent.

### The Independence Test

Parallel branches must have no hidden dependencies on each other:

1. Can branch B start without waiting for any output from branch A?
2. Do branches write to shared state that could cause race conditions?
3. Would running branches in any order produce the same result?

If any answer reveals a dependency, fan-out is incorrect for those branches.

### Four Fan-In Responsibilities

- **Collect results** — gather outputs from every branch that completed successfully before
  timeout.
- **Handle failures** — decide what to do when one or more branches returned errors or timed out.
- **Order results** — reassemble outputs in the right order when sequence matters.
- **Merge and continue** — aggregate collected results and pass them to the next pipeline step
  cleanly.

### Parallel vs. Sequential Tradeoffs

| Parallel execution | Sequential execution |
|---|---|
| Reduces total wall-clock time when branches are independent. Multiple branches run at the same time. | Total time equals the sum of all step latencies. One step at a time. |
| Costs more tokens per unit time and introduces partial-failure scenarios that sequential execution avoids. | Cheaper per run in token spend. No synchronization needed, and no partial failures from concurrent branches. |

### Partial Failure Handling

When some branches succeed and others fail, the fan-in step has to decide what to do:

- **Fail the pipeline** — any branch failure aborts the whole operation.
- **Proceed with partial results** — continue with what succeeded, flag what failed.
- **Retry failed branches** — re-dispatch individual branches that returned errors.

Silently dropping failed branches introduces silent failures — always handle explicitly.

### Result Merging Strategies

How fan-in aggregates the branch results depends on the pipeline's purpose and output type:

- **Voting** — multiple branches answer the same question; the majority result wins.
- **Concatenation** — outputs from independent branches are joined into one list.
- **Structured merge** — each branch contributes a named field to a shared object.

Choose the merge strategy before designing the branches — it shapes what each branch returns.

### Cost vs. Latency in Parallel Execution

Parallel execution cuts latency but increases token spend for the same wall-clock period.

- Three branches running simultaneously use three times the tokens per second.
- Total token cost equals the sum across all branches, not just the longest one.
- Rate limits and quotas may constrain how much concurrency is actually achievable.

Model the cost increase explicitly before choosing parallel over sequential.

### Key Takeaways: Parallel Execution Patterns

Fan-out and fan-in unlock latency wins when branches are independent.

- Fan-out dispatches concurrent branches from a single control point.
- Run the independence test before fan-out.
- Fan-in handles collection, failure, ordering, and merging explicitly.
- Partial failures require explicit handling — never silently drop branches.

---

## 11. Adaptive Planning Mid-Execution

Agentic systems don't always follow the plan they start with.

### Three Planning Concepts

| Term | Definition |
|---|---|
| **Static planning** | A fixed sequence of steps authored before execution begins. Every action is known at design time; the agent follows the script without deviation. |
| **Dynamic planning** | The model generates or revises its plan at runtime based on intermediate results, new information, or unexpected states. |
| **Replanning** | Revising the current plan in response to a failed step, unexpected tool output, or changed environmental state. |

### Static vs. Dynamic Planning

The core difference is *when* the agent's steps actually get decided — design time vs. inference
time.

- Static plans are authored entirely at design time by the developer.
- Dynamic plans are generated or revised at inference time by the model.
- Static offers predictability and auditability; dynamic offers adaptability.

Most production systems combine elements of both approaches.

### Static vs. Dynamic: Tradeoffs

| Static planning | Dynamic planning |
|---|---|
| Predictable, auditable, and cost-efficient. | Adaptable and powerful for open-ended goals. |
| Every step is pre-audited and testable in isolation, making failures easy to localize and diagnose. | Requires stronger monitoring, explicit fallback mechanisms, and tighter logging because next steps are not pre-audited at design time. |
| Best for well-scoped, repeatable tasks with known inputs. | |

### Four Replanning Triggers

Four categories of events can force replanning:

- **Tool call failure** — a tool returns an error, timeout, or empty result the plan did not
  anticipate.
- **Unexpected output** — a step returns valid output that contradicts the assumption the next
  step was built on.
- **State change** — the environment changes mid-execution, making earlier plan steps invalid or
  irrelevant.
- **Constraint violation** — a proposed next action would violate a goal constraint or safety
  boundary.

Each trigger requires the agent to adapt or escalate.

### Replan vs. Continue: The Decision

Not every surprise warrants a full replanning cycle.

- Replan when a core assumption has been invalidated.
- Continue when the deviation is minor and recoverable.
- Check whether the original goal is still achievable.

Unnecessary replanning wastes tokens and adds latency to every run.

### Limiting Replanning Cycles

| Mechanism | Definition |
|---|---|
| **Maximum iterations** | A hard cap on the number of replanning cycles the agent is allowed. Prevents infinite loops when every new plan hits the same obstacle. |
| **Goal constraint check** | A validation step that confirms the revised plan still satisfies the original task requirements before execution continues. |
| **Loop termination** | The condition that halts replanning: either the iteration limit is reached or the new plan clears all goal constraints. |

### Preventing Goal Drift

Replanning without constraint checking is one of the most dangerous agent failure modes.

- Each new plan should be validated against original intent.
- Drift occurs when the revised plan solves the obstacle but ignores the goal.
- Preserving plan state across cycles keeps the agent anchored.

Log the original goal and check every revised plan against it explicitly.

### Key Takeaways: Adaptive Planning Foundations

Dynamic planning enables adaptation but requires explicit safeguards.

- Static plans are predictable; dynamic plans adapt at inference.
- Triggers: tool failures, unexpected results, state changes, constraint violations.
- Bound replanning with iteration limits and goal checks.
- Validate every revised plan against original intent.

Adaptability is safe only when bounded and goal-anchored.

---

## 12. Ambiguity and Incomplete Specifications

Agents rarely receive perfectly specified goals.

### Three Ambiguity Concepts

| Term | Definition |
|---|---|
| **Ambiguous goal** | A task specification that does not fully define the expected outcome, leaving multiple valid interpretations open to the agent at execution time. |
| **Clarify-first strategy** | Pausing execution to request more information from a human before proceeding, used when the cost of a wrong assumption is high. |
| **Assume-and-proceed** | Making a reasonable inference and continuing without human input, used when the action is low-stakes, reversible, or time-sensitive. |

### When to Clarify Upfront

Some situations make assumption-based execution too risky to allow:

- **Irreversible actions** — deletions, sends, or external commits.
- **High-cost operations** — expensive API calls or long batch runs.
- **Scope uncertainty** — the agent doesn't know what's in or out.

In these cases, a brief clarification prevents large downstream errors.

### When to Assume and Proceed

Clarifying every ambiguity makes agents slow and annoying to use.

- **Low-stakes or easily reversible actions** are safe to assume on.
- **Time-sensitive tasks** may not allow a clarification round trip.
- **Well-constrained context** can make the right interpretation obvious.

The goal is appropriate judgment, not maximum caution.

### Clarify vs. Assume

- **Clarify upfront** — adds latency and requires the human to be available. Justified when a
  wrong interpretation would trigger irreversible actions, high-cost operations, or
  out-of-scope execution that is hard to recover from.
- **Assume and proceed** — maintains task velocity and keeps humans out of the loop. Appropriate
  for low-stakes, reversible, or time-sensitive tasks where available context makes the intended
  interpretation reasonably clear.

### Decision Criteria

- **Reversibility** — if the action can be undone easily, proceeding on an assumption carries
  lower risk than pausing.
- **Stake level** — high-cost or high-impact operations justify clarification delays that would
  be unacceptable for routine tasks.
- **Scope clarity** — unclear task boundaries mean a wrong assumption can send the agent
  operating on entirely the wrong data.

### Iterative Refinement Loops

When a single pass isn't enough, generate–evaluate–refine cycles close the quality gap.

- A generator produces an initial candidate output.
- An evaluator scores it against a rubric or constraint set.
- Feedback is passed back to the generator for revision.

This pattern is called the evaluator–optimizer loop.

### Evaluator–Optimizer Pattern

| Component | Role |
|---|---|
| **Generator agent** | Produces the initial or revised output. Receives structured feedback from the evaluator and applies it on the following pass. |
| **Evaluator agent** | Assesses the generator's output against a rubric. Returns structured critique, not just a pass or fail signal. |
| **Convergence criteria** | The threshold or condition that stops the loop: either a quality bar is met or an iteration cap is hit. |

### Documenting Assumptions

When an agent proceeds on assumptions, those assumptions should be visible.

- State assumptions explicitly in agent output or logs.
- This enables human review without rerunning the full task.
- Well-documented assumptions reduce audit burden significantly.

Designing goal specs to be precise at the source reduces ambiguity from the start.

### Key Takeaways: Handling Ambiguity in Agent Goals

The right response depends on stakes, reversibility, and context.

- Clarify upfront for irreversible, high-cost, or scope-uncertain tasks.
- Assume and proceed for low-stakes, reversible, or time-sensitive tasks.
- Evaluator–optimizer loops refine output iteratively with bounded iterations.
- Document assumptions in agent output to support human reviewability.
- Precise goal specifications reduce ambiguity before execution even begins.

---

## 13. The Orchestrator Role

The central controller that decomposes goals, delegates subtasks, and synthesizes results.

### Three Orchestration Concepts

| Term | Definition |
|---|---|
| **Orchestrator** | The controlling agent that breaks a high-level goal into subtasks, assigns them to subagents, and aggregates the results. |
| **Task decomposition** | Breaking a complex goal into smaller, manageable subtasks that can be delegated to individual agents or tools for execution. |
| **Result aggregation** | Collecting outputs from multiple subagents, validating them, and synthesizing them into a single coherent response to the initial goal. |

### Three Functions of the Orchestrator

Every orchestrator performs three core functions in sequence:

1. **Decompose** — break the high-level goal into bounded subtasks.
2. **Assign** — route each subtask to the appropriate subagent or tool.
3. **Aggregate** — collect, validate, and synthesize subagent outputs.

All three must be designed explicitly — none happen automatically.

### Task Assignment: Matching Subtasks

Effective task assignment matches subtasks to capable agents.

- Each subtask should have a clear, narrow scope before assignment.
- Match based on tool access, context, and output format requirements.
- The orchestrator holds the routing logic — subagents don't self-select.

Poorly scoped subtasks are the most common cause of subagent failure.

### What the Orchestrator Retains

- **The orchestrator holds** the master plan, execution state, error handling logic, aggregation
  rules, and final synthesis. It is the only component with visibility across all subtask results.
- **Subagents hold** only what the orchestrator explicitly provides: their specific instructions,
  the context for their subtask, and the tools needed to complete it. Nothing more.

### Result Aggregation: Three Steps

Aggregation runs as a short sequence:

1. **Collect** — gather all subagent outputs once they complete.
2. **Validate** — check each output against expected format or schema.
3. **Synthesize** — combine validated outputs into one coherent result.

Synthesis addresses the original high-level goal.

### Error Handling at the Orchestrator

When a subagent fails, the orchestrator decides what happens next:

- **Retry** — reissue the task with the same or adjusted instructions.
- **Substitute** — route the task to an alternate agent or tool.
- **Escalate** — surface the failure to a human for intervention.

Error handling logic lives in the orchestrator, not in subagents.

### Hub-and-Spoke Topology

The orchestrator pattern maps directly to a hub-and-spoke topology.

- The orchestrator is the hub — all coordination flows through it.
- Subagents are spokes — they don't communicate with each other.
- All state and routing decisions are centralized at the hub.

The tradeoff: a single point of failure and a coordination bottleneck.

### Designing for Observability

An orchestrator that can't explain its decisions is hard to debug and audit.

- Log each task assignment decision with its routing rationale.
- Record progress state after each subagent completes.
- Capture error events with the context that triggered them.

Observability turns the orchestrator from a black box into an audit trail.

### Key Takeaways: Orchestrator Role Foundations

The orchestrator directs, delegates, and aggregates while holding all shared state.

- Three functions: decompose goals, assign subtasks, aggregate results.
- Task assignment matches subtasks to capable subagents by scope and format.
- Result aggregation requires collection, validation, and synthesis.
- Error handling at the orchestrator: retry, substitute, or escalate.
- Logging makes orchestrator decisions auditable.

---

## 14. Subagent Design: Scope and Authority

Subagents are most reliable when they are narrowly scoped, operate within explicit authority
limits, and receive only the context they need.

### Three Subagent Concepts

| Term | Definition |
|---|---|
| **Subagent** | A separate agent instance spawned by a parent orchestrator to handle a delegated subtask within its own bounded context and tool permissions. |
| **Context isolation** | Subagents operate in their own context window. No information passes from orchestrator to subagent unless it is directly handed over. |
| **Authority boundary** | The defined set of actions a subagent is permitted to take. It should do exactly what the task needs and nothing more. |

### Context Isolation Explained

Subagents do not automatically inherit the orchestrator's context.

- Each subagent has its own separate context window.
- Information must be explicitly passed by the orchestrator.
- Nothing in the orchestrator's history or state is visible unless passed.

This is the most frequently missed fact on orchestrator–subagent exam questions.

### Why Context Isolation Matters

Context isolation is not a limitation — it is a deliberate design benefit.

- **Less bloat** — subagents don't carry irrelevant history.
- **Less leakage** — limits information leaking between tasks and agents.
- **Lower cost** — minimal context per subagent reduces token cost.

Isolation also makes subagents individually testable without the full orchestrator.

### What to Pass vs. What to Withhold

- **Pass explicitly** — task-specific instructions, the data or content needed for this subtask,
  required output format and success criteria, and any constraints the subagent must respect.
- **Withhold by default** — full orchestrator conversation history, outputs from other subagents,
  system prompts, credentials, and context not relevant to this specific subtask.

### Scoping Subagent Authority

- **Least privilege** — grant access only to the tools and data the subtask requires. No extra
  permissions by default.
- **Explicit boundaries** — state clearly in instructions what actions are permitted and what is
  out of scope.
- **No side effects** — don't modify shared state or act outside assigned scope without
  orchestrator approval.

```yaml
allowed_tools: ["read_file", "search"]
deny: ["write", "delete", "shell"]
```

### Writing Effective Subagent Instructions

Clear instructions determine how consistent the subagent's output will be.

- Define narrow scope — exactly what the task is and isn't.
- Specify the output format precisely so aggregation is reliable.
- State success criteria explicitly so the subagent knows when it's done.

Example subagent prompt:

```
Task:   extract all dates
Output: JSON list of dates
Done:   every date captured
```

Vague instructions produce inconsistent outputs that break aggregation.

### Testing Subagents Independently

Well-designed subagents are substitutable and independently testable.

- A subagent should run correctly given only its explicit inputs.
- No hidden dependencies on orchestrator state or other subagents.
- Being swappable means changing implementations without breaking the system.

If a subagent can't be tested alone, it's probably doing too much.

### Anti-Patterns in Subagent Design

Three common design failures make subagents brittle and unpredictable:

- **Overly broad scope** — the subagent is asked to do too much at once.
- **Underpowered context** — critical information wasn't passed explicitly.
- **Unclear success criteria** — the subagent doesn't know when it's done.

Each anti-pattern produces a different failure mode, but all are preventable.

### Key Takeaways: Subagent Design Principles

Reliable subagents have narrow scope, bounded authority, and complete explicit context.

- **Context isolation** — subagents receive only what the orchestrator explicitly passes.
- **Authority boundaries** — grant least privilege, no side effects outside task scope.
- **Instruction design** — narrow scope, specific output format, clear success criteria.
- **Anti-patterns** — overly broad scope, underpowered context, vague completion signals.

Subagents built to be swappable are independently testable and more dependable.

---

## 15. Hub-and-Spoke vs. Pipeline vs. Peer-to-Peer

The topology you choose shapes coordination, failure handling, and auditability.

### Three Topology Patterns

| Pattern | Definition |
|---|---|
| **Hub-and-spoke** | A central orchestrator routes all tasks and communication to peripheral agents. Spokes do not communicate with each other. |
| **Pipeline** | Agents arranged in a linear sequence. Each agent processes input from the previous agent and passes output to the next. |
| **Peer-to-peer** | Agents communicate directly with each other without a central coordinator. No single node controls all communication. |

### Hub-and-Spoke: Structure and Strengths

All communication routes through a central orchestrator; spokes do not talk directly.

- The hub decomposes tasks and delegates them to focused expert agents.
- Centralized control means one place to audit decisions and track state.
- Failure handling is straightforward: the hub owns recovery logic.

Clear delegation paths and a single audit trail are its defining strengths.

### Hub-and-Spoke: Failure Modes

The same central structure that gives its strengths creates its failure modes.

- The hub is a single point of failure — if it crashes, the whole system stops.
- As more spokes are added, the hub becomes a throughput bottleneck.
- The hub accumulates context bloat as it tracks state for every active spoke.

Centralized control simplifies coordination but reduces resilience at scale.

### Hub-and-Spoke: Strengths vs. Costs

| Strengths | Costs |
|---|---|
| Centralized state management. | A single point of failure at the hub. |
| Simple failure handling owned by one node. | The hub's capacity caps total system throughput. |
| A clear audit trail for every decision. | Hub context grows with each active spoke, risking token bloat. |
| Easy to enforce consistent delegation policies from one place. | More spokes amplify both bottleneck and risk. |

### Pipeline Topology: Strengths

Each agent processes input from the previous agent and passes output to the next.

- Every agent has a narrow, single responsibility within the sequence.
- Stages are modular and easy to swap, test, or replace independently.
- Each stage produces a clean artifact the next stage uses as full input.

Ideal when transformations are strictly ordered and each output is testable.

### Pipeline Topology: Failure Modes

Latency and failure risk accumulate as the chain grows longer.

- Total latency equals the sum of all stage latencies — no stage runs concurrently.
- A failed mid-stage agent blocks every downstream stage that needs its output.
- Cascade failure risk grows with depth: more stages, more failure surface.

Design explicit error handling at every stage boundary, not only at the exit.

### Peer-to-Peer Topology Tradeoffs

- **Low latency** — agents talk directly, skipping a central coordinator for every message.
- **Resilient** — no single agent failure takes down the whole system.
- **Emergent risk** — complex inter-agent behavior is harder to predict, test, and control.
- **Hard to audit** — no central log or state, so tracing decisions means inspecting every agent.

### P2P vs. Hub-and-Spoke: Key Contrasts

They sit at opposite ends of the coordination spectrum.

- P2P has no single point of failure — any agent can fail without halting the system.
- P2P removes the hub bottleneck and cuts message routing overhead.
- P2P has no central state — coordination is distributed and harder to think through.

Choose based on whether auditability or resilience is the higher priority.

### Key Takeaways: Three Topologies

Each optimizes for different priorities — none is universally best.

- **Hub-and-spoke** — centralized control, simple auditing, single point of failure.
- **Pipeline** — modular stages, clean handoffs, latency accumulation and cascade risk.
- **Peer-to-peer** — low latency, resilient, hard to audit and predict.

Knowing each topology's failure mode matters as much as knowing where it wins.

---

## 16. Selecting the Right Topology

Choosing the wrong topology is a design-level mistake that is expensive to undo. The three
shapes are star, linear, and mesh.

### Three Advanced Topology Concepts

| Term | Definition |
|---|---|
| **Evaluator–optimizer** | One agent produces a result; a second reviews it and feeds back fixes. |
| **Hybrid topology** | Combines two or more topology patterns when no single one fits all parts. |
| **Topology anti-pattern** | Using a topology that clashes with the work structure, adding cost and failures. |

### Decision Framework: Map Task to Topology

Map the task's data flow structure before picking a topology.

| Task structure | Topology |
|---|---|
| Natural hierarchy — one controller delegating to specialists? | Hub-and-spoke |
| Strict ordered transformation — each step depends on the last? | Pipeline |
| Loosely coupled — no shared state, no required ordering? | Peer-to-peer |

The topology that matches the task's natural structure minimizes coordination overhead.

### When to Choose Hub-and-Spoke

Choose it when centralized state and control are genuinely required.

- The task has a natural hierarchy: one controller, multiple specialists.
- Auditing and compliance requirements demand a single decision log.
- Clear delegation paths exist and spokes have no need to talk directly.

With no natural hierarchy, forcing hub-and-spoke creates unnecessary bottlenecks.

### When to Choose Pipeline

Choose it when the task is a strict ordered transformation.

- Each stage depends on the previous stage's output — ordering is required by the data.
- Each stage produces a clean, testable artifact that can be validated independently.
- Swapping or upgrading one stage without touching others is a design requirement.

If stages can run concurrently, pipeline forces unnecessary sequential latency.

### When to Choose Peer-to-Peer

Choose it when speed and resilience outweigh auditability requirements.

- Agents are loosely coupled with no shared state and no required ordering.
- The system needs to keep running even if individual agents fail.
- Low message-routing latency is a primary design requirement.

Do not use P2P when regulatory or operational needs demand a complete audit trail.

### Evaluator–Optimizer Pattern

One agent generates, a second critiques, and the loop refines until good enough.

- **Generator** — produces the initial draft, plan, or answer.
- **Evaluator** — critiques against a defined rubric, returns structured feedback.
- **Iteration loop** — the generator revises until the quality threshold is met.
- **Termination** — a maximum iteration count prevents infinite loops.

### Topology Anti-Patterns

- **Hub-and-spoke anti-pattern** — applying hub-and-spoke to a task with no natural hierarchy.
  The result is a needless bottleneck at the central hub. Spokes could talk directly, yet every
  message routes through a coordinator that adds no value.
- **P2P anti-pattern** — using peer-to-peer when a complete audit trail is required. Distributed
  state makes the full decision log impossible to rebuild, so auditing and debugging needs go
  unmet.

### Hybrid Topologies in Practice

Real systems combine patterns when no single one fits every subsystem.

- A hub-and-spoke orchestrator can delegate to an internal pipeline for ordered transforms.
- A pipeline stage can spawn P2P agents for parallel subtasks with loose coupling.
- Each boundary between patterns needs an explicit handoff.

Identify which subsystem each pattern serves before designing the boundaries.

### Key Takeaways: Topology Selection

Matching topology to task prevents design-level failures.

- **Hub-and-spoke** — hierarchy, centralized state, auditability.
- **Pipeline** — ordered transforms with testable stages.
- **Peer-to-peer** — loose coupling, resilience, low latency.
- **Evaluator–optimizer** — generate plus critique with termination.
- Hybrids are valid: every pattern boundary needs an explicit handoff.

---

## 17. Designing Agent-to-Agent Schemas

Freeform handoffs produce brittle systems; well-designed schemas make them reliable. The flow
is sender agent → schema → receiver agent.

### Three Schema Concepts

| Term | Definition |
|---|---|
| **Handoff message** | A structured payload with task, context, output format, and constraints the receiver needs. |
| **Context isolation** | Subagents have their own context; nothing passes unless the handoff includes it. |
| **Schema versioning** | Version tags on handoff formats so receivers parse messages even as the schema evolves. |

### Why Unstructured Handoffs Fail

Freeform handoffs create fragile systems that break silently.

- **Missing context** — the receiver lacks the information needed to act correctly.
- **Inconsistent format** — no agreed structure means brittle, error-prone parsing.
- **Implicit assumptions** — the sender assumes the receiver knows things it does not.

Structured schemas eliminate each of these failure modes explicitly.

### Core Schema Components

A well-designed handoff message has four required elements:

- **Task description** — what the receiver is asked to produce.
- **Relevant context** — only the facts the receiver needs.
- **Output format** — exactly what structure the result takes.
- **Constraints** — boundaries, scope limits, safety restrictions.

```json
{
  "task": "...",
  "context": [...],
  "format": "json",
  "constraints": [...]
}
```

### Include vs. Exclude in Handoffs

- **Include** — what the receiver needs to act without the sender's context: task, relevant
  facts, output format, and explicit constraints.
- **Exclude** — what the receiver doesn't need: full conversation history, internal deliberation,
  sender state, and redundant context.

### Two Handoff Failure Patterns

- **Under-specified** — the receiver lacks enough context to act, so it guesses or fails. The
  sender assumed it knew more than it did.
- **Over-specified** — the receiver gets the full sender context. This inflates cost, adds noise,
  and slows parsing with irrelevant detail.

### Schema Versioning Principles

Handoff schemas must evolve without breaking receiving agents.

- Tag every message with a `schema_version` field.
- Receivers handle missing optional fields gracefully.
- New required fields need a migration path for older senders.

Forward compatibility keeps multi-agent systems stable during updates.

### Testing Handoff Schemas

Validate the schema before the receiver runs, not after it fails: build message → validate
schema → receiver runs (or reject).

- Contract tests confirm required fields are present.
- Simulate the receiver with the schema alone to check sufficiency.
- Test with missing fields to verify graceful degradation.

A schema test suite is the simplest form of inter-agent integration testing.

### Designing for Agent Substitutability

Schemas tightly coupled to one implementation are fragile.

- Specify the interface, not the receiver's implementation.
- Avoid fields only one specific agent knows how to use.
- Receivers that share a schema should be interchangeable.

Substitutability lets you swap implementations without rewriting callers.

### Key Takeaways: Agent Handoff Schema Design

Structured schemas prevent the silent failures common in ad-hoc agent messaging.

- **Include** — task description, relevant context, output format, constraints.
- **Exclude** — everything the receiver doesn't need; cost and clarity both improve.
- Version schemas and test for forward compatibility as systems evolve.
- Design against an interface, not a specific receiver implementation.

Testable, versioned schemas are the foundation of reliable multi-agent systems.

---

## 18. Handoff Protocols and Agent Continuity

Passing a message isn't a completed handoff — verification and continuity make it reliable.

### Three Handoff Protocol Concepts

| Term | Definition |
|---|---|
| **Handoff verification** | Confirming the receiver is prepared before the sender hands off the task. |
| **Error propagation** | How subagent failures surface back to the controller so it can respond. |
| **Silent failure** | A handoff that looks successful but loses critical info, with no error signal. |

### Handoff Verification Steps

Verification confirms readiness before the sender lets go:

1. Send handoff.
2. Acknowledge receipt.
3. Confirm ready.
4. Release the task.

Without verification, the sender assumes success with no confirmation.

### Designing Error Propagation

Subagent failures must reach the orchestrator to be handled.

- Subagents return structured error payloads, not silent nulls.
- Error payloads include failure type, context, and recovery hints.
- Orchestrators need explicit error paths, not only happy paths.

```json
{
  "type": "timeout",
  "context": "...",
  "recovery": "retry"
}
```

### Transparent vs. Silent Failure

- **Transparent failure** — the subagent returns a structured error with type and context, so the
  orchestrator can retry or halt.
- **Silent failure** — the handoff looks successful but drops critical info. No error is raised,
  so the failure surfaces much later.

### Two Propagation Anti-Patterns

- **Swallowing errors** — a subagent catches an exception and returns empty output. The
  orchestrator assumes success and never learns it failed.
- **Undifferentiated errors** — every failure returns the same generic code. The orchestrator
  can't tell a retryable timeout from a hard violation.

### Continuity Across Handoff Failures

When a handoff fails mid-transfer, preserved state keeps continuity.

- Save task state before initiating any handoff.
- Record the last confirmed good checkpoint, not just the latest attempt.
- Retry resumes from the checkpoint, not from the beginning.

Without checkpointing, a failed handoff forces the full workflow to restart.

### Designing for Agent Substitutability

Protocols that assume a specific receiver implementation are brittle.

- Protocols specify behavior, not the implementing agent.
- Any compliant receiver can process the same handoff.
- Swapping one implementation for another needs no protocol change.

This is what keeps multi-agent systems maintainable over time.

### Audit Logging for Handoffs

Inspectable handoff records turn silent failures into diagnosable ones.

- Log every handoff: sender, receiver, timestamp, payload.
- Log verification results: acknowledgement received or timeout.
- Log errors with enough context to reconstruct what the receiver had.

### Key Takeaways: Reliable Handoffs Across Agent Boundaries

A handoff completes only when the receiver confirms readiness and failures can surface.

- Verify handoffs: the receiver acknowledges receipt and readiness before release.
- Propagate errors with structure: type, context, recovery hints.
- Checkpoint state before handoffs so failures don't force full restarts.
- Audit every handoff so silent failures become visible.

Inspectable, verified handoffs are the foundation of reliable multi-agent pipelines.

---

## 19. In-Context State vs. External Memory

Agents can hold state in the active conversation or store it in external systems.

### Three Memory Concepts

| Term | Definition |
|---|---|
| **In-context state** | What the agent knows right now, inside its live context. |
| **External memory** | Persistent stores outside the model that survive truncation. |
| **Semantic retrieval** | Querying a vector index by meaning to recall relevant chunks. |

### In-Context State: Speed and Limits

In-context state lives directly in the conversation the model is reading.

- Zero retrieval latency — read inline, no fetch step.
- Bounded by the context window — can't hold unlimited info.
- Lost when the session ends or the window fills up.

Fast and simple, but not durable — any interruption can erase unwritten state.

### External Memory: Persistence and Query

External memory stores state outside the model in databases the agent queries.

- Persistent across sessions — memory survives truncation and restarts.
- Requires explicit read and write tool calls, adding latency
  (`memory.read(key)` / `memory.write(key, value)`).
- Searchable and structured — key lookup, filtering, semantic search.

### In-Context vs. External Memory

- **In-context state** — instant, no tool call. Lost on truncation. Best for transient state.
- **External memory** — tool call needed. Persists between runs. Best for durable state.

### Types of External State Stores

External memory comes in three main forms, each suited to different access patterns:

- **Key-value store** — fast exact match by ID; user profiles, session data.
- **Relational DB** — structured queries, joins, filtering; audit logs, records.
- **Vector index** — semantic similarity search; history, document recall.

Choose the store type based on how the agent queries and updates the data.

### Where Should State Live?

The core question: does state need to survive beyond the current context?

- **Actively changing this turn, referenced often** — keep in context.
- **Rarely read, needed across sessions** — persist externally.
- **Can be rebuilt from tools or sources** — skip storing it.

Default to in-context for speed; escalate to external only when durability requires it.

### Four State Survival Questions

1. **Survives interruption?** If the session ends mid-task, is this state recoverable without
   external persistence?
2. **Needs semantic query?** Does the agent retrieve state by meaning or similarity rather than
   exact key?
3. **Shared across sessions?** Does another session or user need to read the same state later?
4. **Reconstructable?** Can the state be rebuilt cheaply from source data or tool calls if lost?

### Hybrid Approaches: Hot and Cold Tiers

Most production agents use both in-context and external memory together.

- **Hot tier (in-context)** — active plan, current step, recent tool outputs.
- **Cold tier (external)** — completed results, configuration, history.

The hot tier keeps the active plan, current step, and recent outputs in context, and loads
minimal state on resume.

### Key Takeaways: In-Context and External Memory Tradeoffs

In-context state is fast but temporary; external memory is slower but durable.

- **In-context** — zero-latency, lost on truncation or session end.
- **External** — persistent, queryable, requires tool calls per access.
- Choose by whether state must survive interruption.
- **Hybrid** — keep hot state in context, cold state externally.

Default to in-context for speed; escalate to external when durability requires it.

---

## 20. Session Continuity Across Turns

Sessions can be interrupted at any point; agents must resume without losing progress. The cycle
is interruption → checkpoint → resume.

### Three Continuity Concepts

| Term | Definition |
|---|---|
| **Session resumption** | Resuming a task from a checkpoint after interruption, without repeating completed steps. |
| **Checkpoint** | A saved snapshot of task progress at a phase boundary to recover to after failure. |
| **State versioning** | Tracking which environment version a checkpoint used, to detect conflicts on resume. |

### What Session Resumption Requires

To resume, the agent must know exactly where it was and what it had done:

- **Completed steps** — what has already been executed and confirmed.
- **Current inputs** — what the next step needs to proceed.
- **Prior results** — outputs from earlier steps still needed downstream.

Without all three, the agent repeats completed work or starts from scratch.

### Checkpoint Design: What to Persist

A checkpoint captures task state at a phase boundary so recovery starts there.

- Persist at the end of each major phase, not after every tool call.
- Include completed step count, phase identifier, and next-phase inputs.
- Write the checkpoint before any irreversible action.

```json
{
  "phase": 3,
  "steps_done": 12,
  "ts": "14:02"
}
```

### Checkpoint Minimum Contents

- **Phase marker** — which phase completed, so resumption skips all prior phases.
- **Step outputs** — results from completed steps that downstream phases depend on.
- **Task inputs** — the original task parameters, so context can be reconstructed.
- **Timestamp** — when the checkpoint was written, for conflict detection and ordering.

### Coarse vs. Fine-Grained Checkpoints

- **Coarse checkpoints** — saved at phase boundaries only. Fewer saves, simpler state. May repeat
  some work in the current phase. Best for most agentic workflows.
- **Fine-grained checkpoints** — saved after every step. Minimal repeated work, but high write
  volume and complex state. Justified only when steps are costly.

### Failure Detection and Recovery

Recovery starts by detecting an interruption and finding the last valid checkpoint.

- On start, check for an existing checkpoint before beginning fresh.
- If one exists, verify its integrity before trusting it.
- Load it and skip all phases already marked completed.

### State Versioning and Conflict Resolution

If the world changes between checkpoint write and resume, saved state may be stale.

- Version the checkpoint with a timestamp and environment hash.
- On resume, compare the current environment to the recorded version.
- If it changed, flag the conflict instead of proceeding on stale state.

### Memory Hygiene: Pruning Stale State

As sessions grow, stored state goes stale or misleading unless it is pruned.

- Identify state no longer relevant to current or future phases.
- Before pruning, confirm whether content should be archived first.
- Prune from context to free tokens, and from stores to cut noise.

Stale state can cause the agent to act on outdated facts.

### Key Takeaways: Session Continuity Design Principles

Robust sessions recover from interruptions without losing progress or repeating work.

- Checkpoints capture phase progress, step outputs, and task inputs.
- Coarse phase-boundary checkpoints are simpler and enough for most workflows.
- State versioning detects conflicts when the environment shifts between saves.
- Memory hygiene prunes stale state before it misleads future decisions.

Partial state beats no state — design for graceful degradation at every phase.

---

## 21. Multi-Agent Cost and Fit

More agents can outperform one, but they cost far more and add coordination overhead.

### Three Terms to Anchor

| Term | Definition |
|---|---|
| **Single agent** | One model running in a loop, calling tools and holding the whole task in one context window. |
| **Orchestrator** | A lead agent that plans work and spawns subagents, then combines their results. |
| **Subagent** | A worker agent given a scoped slice of the task, running in its own separate context. |

### The Token Economics

More agents means more model calls, so more tokens.

| Setup | Relative token use |
|---|---|
| Plain chat turn | ~1× (the baseline) |
| Single agent | ~4× chat tokens |
| Multi-agent system | ~15× chat tokens |

### Where the Cost Comes From

- **Orchestrator overhead** — the lead agent spends tokens planning, delegating, and merging
  before real work.
- **Parallel fan-out** — each subagent repeats its own setup context and reasons independently.
- **Result stitching** — combining every subagent output into one answer takes another expensive
  model pass.

### Coordination and Latency

- The orchestrator must decompose the task cleanly up front.
- Splitting work adds complexity, not just cost.
- Subagents can drift or duplicate effort without tight instructions.
- Results must be gathered and reconciled before you answer.

### Good Fit vs. Poor Fit

- **Good fit** — breadth-first work you can split into many independent directions that explore
  in parallel, like researching a large set of separate sources.
- **Poor fit** — tightly-coupled work where agents share one context and depend on each other,
  where a single agent stays cleaner, faster, and cheaper.

### The Fit Rule

Multi-agent shines when subtasks are truly independent. Ask:

1. Can the work split into parallel directions that barely touch?
2. Does each piece fit its own context with no shared state?
3. Is the task valuable enough to justify the token bill?

### Two Task Shapes and a Trap

- **Breadth-first** — work that branches into many independent lines, each explorable on its own
  thread.
- **Tightly-coupled** — work where each step depends on the last and everything needs one shared
  context.
- **The coding trap** — most coding tasks have fewer parallel pieces than research, so they often
  fit one agent.

### Matching Models to Roles

Soften the cost by matching models to roles.

- A stronger model leads as orchestrator.
- Lighter worker models handle subtasks.
- Reserve the design for high-value tasks that repay the spend.

### Key Takeaways: When Multi-Agent Earns Its Price

- Multi-agent power comes at a steep, multiplying token cost.
- Roughly fifteen times chat tokens, versus four for one agent.
- Cost stacks from planning, fan-out, and merging results.
- Good fit is breadth-first, independent, parallel work.
- Poor fit is tightly-coupled work needing shared context.

Reach for it only when the value clearly repays the bill.

---

## 22. Tool, Reasoning, and Environment Errors

Agentic systems fail in three distinct ways, each requiring a different recovery strategy.

### Three Error Categories

| Category | Definition |
|---|---|
| **Tool error** | An external tool, API, or service errors or times out. May be transient or permanent. |
| **Reasoning error** | The model produces a wrong plan or misreads a result. Retrying identical input fails again. |
| **Environment error** | Infrastructure below the tool fails: network, database, or file system. Often transient. |

### Why Classification Matters for Recovery

Error type determines the correct recovery strategy; misclassifying wastes retries or misses
escalation.

- **Tool errors** — retry with backoff, or switch to a fallback tool.
- **Reasoning errors** — modify the prompt or correct context; do not retry.
- **Environment errors** — often resolved by waiting and retrying after a delay.

Applying the wrong strategy wastes resources and delays a correct result.

### Tool Errors: Transient vs. Permanent

Tool errors split by whether the same call could succeed if retried.

- **Transient** — network timeout, rate limit, temporary unavailability. Retry with backoff.
- **Permanent** — invalid input, permission denied, resource not found. Do not retry.

HTTP signals the category: 5xx transient, 4xx typically permanent.

### Transient Tool Error Examples

- **Network timeout** — the API did not respond in the allowed window. Retriable with a short
  delay.
- **Rate limit** — too many requests in a window. Wait for the backoff period, then retry.
- **Temporary unavailability** — service is momentarily down. Use exponential backoff with a max
  retry limit.

### Permanent vs. Transient Tool Errors

- **Permanent tool errors** — invalid input, permission denied, resource not found. The identical
  call fails again. Recovery needs a fallback tool, a different input, or escalation. Never retry
  blindly.
- **Transient tool errors** — timeout, rate limit, temporary unavailability. The same call may
  succeed if retried after a delay. Use exponential backoff with a max retry count. Log each
  attempt.

### Reasoning Errors: Internal Failures

Reasoning errors come from within the model, not from external tools or infrastructure.

- The model makes a wrong plan, picks the wrong tool, or misreads a result.
- Retrying identical input fails again — the model's context hasn't changed.
- Recovery needs prompt modification, context correction, or escalation.

Reasoning errors are the only category where the model itself is the failure point.

### Environment Errors: Below the Tool

Environment errors happen in the infrastructure layer beneath both the agent and its tools
(agent plan → tool logic → environment: network, database, file system).

- Examples: network partition, database down, file permission failure.
- The tool's logic and the model's plan are correct — the environment blocks execution.
- Often transient; resolves without model or tool changes.

Distinguish from tool errors by where the failure originates, not its symptom.

### The Classification Rule

- **Tool error** — the tool or API reports failure. Failure is in the external service.
- **Reasoning error** — the tool ran, but the plan or interpretation was mistaken. The model is
  at fault.
- **Environment error** — the tool couldn't reach its own dependencies. Failure is in
  infrastructure.
- **Apply it** — ask: where did the failure originate? The answer determines the recovery route.

### Key Takeaways: Error Types Determine Recovery Paths

Three categories cover agentic failures: tool, reasoning, and environment errors.

- Tool errors are external — transient ones retry, permanent ones need a fallback.
- Reasoning errors are internal — retrying identical input always fails again.
- Environment errors are infrastructure failures — often transient, retry after a delay.
- Classification rule: ask where the failure originated, apply the matching strategy.

Misclassifying an error type delays recovery and risks cascading failures downstream.

---

## 23. Error Detection and Validation

Catching failures early matters because a silent failure proceeds on bad data without warning,
and that bad data propagates.

### Three Detection Concepts

| Term | Definition |
|---|---|
| **Silent failure** | Output looks successful but is wrong, incomplete, or corrupted. No error signal is raised. |
| **Validation gate** | A checkpoint that inspects step output against criteria before letting execution continue. |
| **Error propagation** | A failure in one stage flows downstream, causing cascading failures across the pipeline. |

### Silent Failures: Why They're Dangerous

A silent failure returns output that looks like success but is wrong or corrupted.

- No exception or error code is raised — execution continues normally.
- Downstream stages receive bad data and build further work on top of it.
- By the time the error surfaces, its origin may be several steps back.

Silent failures need validation gates — error-code retry logic won't catch them.

### Detection Strategy 1: Output Validation

Validate a step's result against a defined schema or constraint before it moves forward.

- **Schema checks** — required fields present, correct types, no unexpected nulls.
- **Range checks** — numeric values stay in bounds; string lengths stay capped.
- **Completeness checks** — all required sections populated; no truncated responses.

Output validation catches structural errors and silent failures that status codes miss.

### Schema vs. Semantic Validation

- **Schema validation** — verifies structure: required fields, correct types, no unexpected
  nulls. Fast and automatable. Catches formatting errors and truncated output. Does not check
  whether content is correct.
- **Semantic validation** — verifies meaning: is the output plausible given the input? Are values
  in expected ranges? Catches silent failures that pass schema checks but are logically wrong.

### Detection Strategy 2: State Verification

Check whether the world matches what the agent expects after a step.

- **After a write** — confirm the record was actually persisted correctly.
- **After a transform** — verify the output reflects the intended change.
- **After an action** — validate the side effect occurred, not just that the call returned.

State verification catches actions that appeared to succeed but had no real effect.

### Detection Strategy 3: Sanity Checks

Ask whether an output is plausible given the input, beyond schema checks.

- Is output length reasonable? A one-word answer to a complex query signals trouble.
- Are key entities from the input present in the output? Missing actors suggest truncation.
- Does the output contradict known facts provided in the input context?

Sanity checks are the last line before semantic errors silently enter downstream stages.

### Four Validation Gate Responsibilities

1. **Inspect output** — check schema, ranges, completeness, and plausibility before forwarding.
2. **Trigger retry** — if validation fails, retry the step with corrected input or prompt.
3. **Trigger fallback** — if retry is exhausted, route to a fallback path or degrade gracefully.
4. **Trigger escalation** — if fallback fails, surface the error to a human reviewer or monitor.

### Logging and Observability Strategies

Structured logging and execution traces make silent failures diagnosable after they occur.

- Log inputs and outputs at every stage boundary, not only on errors.
- Use structured log formats so fields are queryable and filterable.
- Execution traces record the full sequence of steps, tool calls, and outcomes.
- Anomaly detection flags unusual patterns before users report failures.

Observability turns a silent failure into a diagnosable, fixable event.

### Key Takeaways: Detection Closes the Observability Gap

Silent failures are the hardest — they produce no error signal.

- Output validation checks schema, ranges, and completeness.
- State verification confirms actions had their intended effect.
- Sanity checks verify output plausibility against the input.
- Validation gates intercept failures before downstream propagation.
- Structured logs and traces make silent failures visible.

---

## 24. Retry Logic

Not every failure deserves a retry; classify the error, then choose a policy or abort. Transient
errors retry; persistent errors abort.

### Three Retry Concepts

| Term | Definition |
|---|---|
| **Transient error** | A temporary failure likely to resolve on its own: timeouts, rate limits, network blips. |
| **Persistent error** | A failure retrying will not fix: invalid input, permission denied, reasoning errors. |
| **Exponential backoff** | Wait time doubles after each failure, with jitter, to prevent retry storms. |

### The Core Retry Question

Before retrying any failure, ask one question: **is this error likely to resolve if I try again?**

- **Yes → transient.** A retry policy is appropriate.
- **No → persistent.** Retrying just delays the real fix.

Answering the wrong way costs tokens, adds latency, and masks the root cause.

### Transient vs. Persistent Error Classes

- **Transient** — rate limit hit, request timeout, temporary network failure, service overload.
  Likely to clear on its own; retry with a policy.
- **Persistent** — invalid input format, permission denied, authentication failure, reasoning
  error. Will not clear; abort and escalate.

Ambiguous cases (e.g. a 500): use one conservative retry with a delay, then treat as persistent.

### Immediate vs. Delayed Retry Policies

- **Immediate retry** — only for momentary glitches that resolve in milliseconds. Rare in agentic
  workflows. Spamming retries under load makes overload worse; rarely the right default.
- **Fixed delay** — simpler to implement. Adds a constant wait between attempts. Does not adapt
  to bursts or system overload. Better than immediate, but backoff is more reliable.

### Exponential Backoff with Jitter

The gold-standard retry policy for most transient errors.

- Wait time doubles after each failure: 1s, 2s, 4s, 8s, up to a cap.
- Jitter adds randomness to prevent synchronized retry storms across clients.
- Apply only to transient errors — backoff on persistent errors only postpones failure.

Backoff gives overloaded systems breathing room between each retry attempt.

### Retry Policies by Error Type

| Error type | Policy |
|---|---|
| **Transient: rate limit** | Exponential backoff with jitter. Respect `Retry-After` headers when present. |
| **Transient: timeout** | Fixed delay or short exponential backoff. One to three retries max. |
| **Persistent: invalid input** | Abort immediately. Retrying identical input produces identical failure. |
| **Persistent: reasoning error** | Abort and escalate. The same prompt cannot fix the plan. |

### Retry Budgets: Cost and Latency Limits

Uncapped retries accumulate cost and latency with no guarantee of eventual success.

- A retry budget sets the maximum number of attempts before aborting.
- Each additional retry multiplies token spend and total wait time.
- Retry budgets keep pipeline latency bounded and predictable.

Design the budget around acceptable latency and cost, not just error probability.

### When to Abort: Abort Conditions

Two conditions signal that aborting is the correct choice over retrying:

- **Budget exhausted** — maximum attempts reached, still failing. Abort and escalate.
- **Persistent on first try** — no retry is warranted. Abort immediately, before spending.

Abort is not silent failure — it triggers escalation and surfaces the error upstream.

### Key Takeaways: Retry Logic

Retry decisions start with one question: will this error resolve if I try again?

- Transient errors are retriable; persistent errors should abort immediately.
- Exponential backoff with jitter is the standard policy for transient failures.
- Retry budgets bound cost and latency — uncapped retries are a design flaw.
- Abort triggers escalation, not silent failure.

Classify first, then choose policy, budget, and abort conditions.

---

## 25. Fallback Chains and Graceful Degradation

When a primary action fails and retrying won't help, the agent needs an alternative path:
Tool A → Tool B → cached result → escalate.

### Three Fallback Concepts

| Term | Definition |
|---|---|
| **Fallback chain** | An ordered sequence of alternatives tried when the primary action fails, ending in escalation. |
| **Graceful degradation** | Returning a useful but incomplete result while explicitly signaling the reduced quality downstream. |
| **Silent degradation** | Returning a degraded result with no signal; downstream wrongly treats it as full success. |

### Fallback Chains: Structure and Purpose

A fallback chain is an ordered sequence of alternatives executed when the primary path fails.

- Each step is a pre-validated alternative: Tool B, cached result, simplified approach.
- Steps execute in order, each one only if the previous step also failed.
- The chain always ends in an explicit terminal: escalate or return a partial result.

Fallback chains make failure handling systematic rather than ad-hoc.

### Pre-Validating Every Fallback Step

Each fallback option must be validated before the chain is deployed in production.

- An untested fallback may introduce new failure modes under the same conditions.
- Validate that each alternative actually works when the primary fails.
- Confirm fallback outputs meet the minimum quality bar downstream requires.

An untested fallback looks like a safety net but may not hold under real conditions.

### Graceful vs. Silent Degradation

- **Graceful degradation** — returns a partial or lower-quality result and explicitly signals the
  degradation downstream. The limitation is visible. Downstream logic can adjust or escalate
  based on the signal.
- **Silent degradation** — returns a degraded result with no signal. Downstream treats it as full
  quality. Errors compound silently and surface later as incorrect results that are hard to trace.

### Propagating Degradation Signals

When a fallback fires, the quality reduction must travel with the result.

- Include a degradation flag or metadata field in the output object.
- Downstream components check the flag and adjust confidence or routing.
- Signals prevent downstream from treating partial results as authoritative.

### Fallback Chain Anti-Patterns

- **Untested fallbacks** — deploying a fallback never validated in production introduces new
  failure modes exactly when the system can least afford them.
- **Silent masking** — a fallback chain that hides degradation from downstream components
  produces silent failures that compound undetected across the pipeline.

### Partial Success vs. Full Failure

When fallbacks return incomplete results, the pipeline chooses partial success or full failure.

- **Partial success** — return a useful but incomplete result plus an explicit degradation signal.
- **Full failure** — return nothing and escalate when a partial output is too risky to use.

The choice depends on whether a partial result causes downstream harm.

### When to Escalate vs. Degrade

Two factors determine whether to escalate or accept degraded output: risk and reversibility.

- **High risk, irreversible** — degraded output may cause unrecoverable harm. Escalate to a human.
- **Low risk, reversible** — degraded output is acceptable. Propagate the signal and continue.

Irreversible actions — deletions, payments, communications — demand a higher threshold.

### Key Takeaways: Fallback Design and Degradation Signals

Fallback chains make failure handling systematic: ordered alternatives, not ad-hoc guesses.

- Pre-validate every fallback step before deploying it in production.
- Graceful degradation requires explicit signals, not silent quality reduction.
- Partial success is valid when downstream components can handle incomplete data.
- Irreversible, high-risk actions require escalation rather than degraded output.

The worst fallback is one that masks failure entirely and lets silent errors compound.

---

## 26. Why Prompts Aren't Enough

Prompt guardrails are useful behavioral guidance — but probabilistic, not guaranteed.

- **Prompt guardrail (probabilistic):** "You must never issue a refund over $100 without a
  manager sign-off."
- **Code enforcement (guaranteed):**

```python
if refund > 100:
    require_approval()
else:
    process(refund)
```

### Three Core Concepts

| Term | Definition |
|---|---|
| **Prompt guardrail** | A behavioral rule in a system prompt. Easy but probabilistic; it can fail. |
| **Programmatic enforcement** | Code-layer checks that enforce limits the model cannot bypass. |
| **Silent failure** | A response that looks successful but is actually wrong; validation gates catch it. |

### What Prompt Guardrails Actually Are

Prompt guardrails are behavioral guidance embedded in instructions.

- They ask the model to avoid certain outputs or actions.
- They rely on the model choosing compliance at inference time.
- They are not code checks and produce no deterministic guarantee.

Compliance is the expected outcome, but it is not architecturally enforced.

### The Probabilistic Problem

LLM outputs are probabilistic — "usually follows" is not a guarantee.

- **Context drift** — long conversations erode the salience of prompt instructions.
- **Adversarial inputs** — crafted prompts can elicit non-compliant outputs.
- **Edge cases** — novel inputs fall outside training distribution coverage.

Each failure mode is rare — but rare is not acceptable in high-stakes paths.

### Sufficient vs. Insufficient Use Cases

- **Sufficient: low-stakes** — tone, style, formatting, reversible actions with no legal or
  financial consequence. A failure here is recoverable and low-cost. Probabilistic compliance is
  acceptable.
- **Insufficient: high-stakes** — financial transactions, PII handling, safety-critical or
  irreversible actions. A failure here can cause legal, financial, or safety harm. This level of
  risk is not tolerable.

### Three Prompt-Only Failure Modes

- **Adversarial inputs** — crafted user inputs steer the model past its prompt instructions. No
  wording fully resists a determined attacker.
- **Context drift** — in long runs, early constraints lose salience as new context accumulates.
  Effective attention shifts.
- **Edge case coverage** — unusual inputs not seen in training can produce non-compliant outputs
  even with no attack.

### When Code Must Enforce

The cost of non-compliance determines what layer must do the enforcing.

- **Financial transactions** — amount limits and authorization checks in code.
- **PII handling** — redaction and masking enforced by output filters.
- **Safety-critical actions** — hard blocks before execution, not after.

When the cost of a single failure is high, enforcement belongs in code.

### The Core Design Principle

Prompts guide behavior. Code enforces constraints. These are different roles.

- **Prompts guide** — reduce the frequency of non-compliant outputs.
- **Code enforces** — prevent non-compliant outputs from reaching the execution layer.

Using a prompt to replace a code check is an architectural mistake, not a shortcut.

### Layered Enforcement Overview

Reliable high-stakes systems use multiple enforcement layers together:

- **Prompt layer** — behavioral guidance and tone; the first line of intent.
- **Validation layer** — schema and business-rule checks in code.
- **Runtime layer** — live monitoring for cost, scope, and anomaly signals.

No single layer is sufficient alone — depth provides resilience.

### Key Takeaways: Prompt Limits and When Code Steps In

Prompt guardrails are useful but probabilistic; high-stakes paths require code-layer enforcement.

- Guardrails can fail via adversarial inputs, context drift, and edge cases.
- Low-stakes reversible actions: prompt guardrails are acceptable.
- High-stakes paths — financial, PII, safety-critical: code must enforce.
- Prompts guide. Code enforces. These layers serve different roles.

The real question: are prompts the only layer, or just the first?

---

## 27. Programmatic Validation Layers

Prompt guidance is the first enforcement layer — not the only one. The others are
pre-execution, post-execution, and runtime.

### Three Enforcement Concepts

| Term | Definition | Example |
|---|---|---|
| **Validation gate** | A checkpoint that inspects a step's output before execution continues. | A schema check on tool output. |
| **Escalation trigger** | A condition that pauses the agent and routes to human review, balancing safety against friction. | A $10,000 transfer pauses for human approval. |
| **Programmatic enforcement** | Code-layer constraints the model is unable to override. | Code blocks any refund above the set limit. |

### Pre-Execution Validation

Pre-execution validation checks inputs before the agent takes any action.

- **Schema validation** — confirm inputs match the expected structure and types.
- **Range checks** — verify numeric values fall within allowed parameter bounds.
- **Permission check** — confirm the caller is authorized to request this action.

Invalid inputs stopped early cost far less than invalid actions caught later.

### Post-Execution Validation

Post-execution validation checks outputs before they're acted upon downstream.

- **Schema compliance** — confirm the response matches the required structure.
- **Business rules** — verify the output satisfies domain-specific constraints.
- **Anomaly detection** — flag outputs that are valid in form but semantically unusual.

Post-execution gates catch model errors that pre-execution checks can't see.

### Pre-Execution vs. Post-Execution Gates

- **Pre-execution gate** — runs before the agent acts. Catches invalid inputs, out-of-range
  parameters, and unauthorized requests. Keeps bad inputs from reaching the model. Lower cost
  than catching errors later.
- **Post-execution gate** — runs after the agent produces output but before it is committed.
  Catches output schema violations, business-rule failures, and semantic anomalies the input
  check could not anticipate.

### Two Gate Response Patterns

- **Hard block** — rejects the output and halts or retries. Applied to non-negotiable
  constraints: financial limits, permissions, and safety boundaries.
- **Soft warning** — flags and logs the output but allows execution to continue. Used for
  advisory constraints where human judgment adds value.

### Runtime Checks During Execution

Runtime checks monitor the agent continuously while it's running, not just at boundaries.

- **Cost limits** — token budgets and API spend caps enforced mid-run.
- **Iteration caps** — maximum loop counts to prevent runaway agent cycles.
- **Scope enforcement** — detect when the agent accesses out-of-bounds resources.

Runtime checks are the safety net when pre- and post-execution gates aren't enough.

### Layered Enforcement in Practice

- **Prompt layer** — intent and behavioral guidance; lowest cost.
- **Pre-execution** — schema and permission checks before action.
- **Post-execution** — output validation before results are committed.
- **Runtime** — live monitoring for cost, scope, and anomalies.

Depth is what makes a high-stakes pipeline defensible across failure modes.

### Balancing Strictness and Flexibility

Over-strict enforcement creates friction that stalls legitimate workflows.

- Use hard blocks only for non-negotiable safety and compliance constraints.
- Use soft warnings for advisory constraints where human review adds value.
- Reserve escalation triggers for ambiguous cases requiring human judgment.

Calibration means matching the enforcement response to the actual risk level.

### Key Takeaways: Designing Layered Enforcement Gates

Reliable pipelines combine pre-execution, post-execution, and runtime validation with prompt
guidance.

- **Pre-execution** — schema, range, and permission checks before the agent acts.
- **Post-execution** — output schema, business rules, and anomaly detection after generation.
- **Runtime** — cost limits, iteration caps, plus scope enforcement during execution.
- Match enforcement response to risk: hard block, soft warning, or escalation.

Layered enforcement is the architecture that makes probabilistic LLM results reliable.

---

## 28. Hook Lifecycle Events

Claude Code hooks run your own commands at specific moments. This module maps the lifecycle
events by cadence and shows which can block and which only observe.

- **Session** — `SessionStart`, `SessionEnd`
- **Turn** — `UserPromptSubmit`, `Stop`
- **Tool** — `PreToolUse`, `PostToolUse`

### Three Ideas Behind Hooks

| Term | Definition |
|---|---|
| **Hook** | A command you register to run automatically when a specific lifecycle event fires in Claude Code. |
| **Lifecycle event** | A named moment, like a tool call or session start, where hooks get a chance to run. |
| **Blocking hook** | A hook that can stop the action from happening, signaled by exiting with code two. |

### Events Grouped by Cadence

- **Once per session** — fires once at session start and end.
- **Every turn** — fires around each prompt submit and stop.
- **Every tool call** — wraps every pre-tool-use and post-tool-use call.

### Session-Level Events

The outermost layer fires once as a session opens and once as it closes.

- `SessionStart` runs when a session begins or resumes.
- `SessionEnd` runs when the session terminates.
- Both are observe-only, so they cannot block anything.
- Use them to set up context on entry or clean things up on the way out.

### Turn-Level Events

One layer in, two events bracket each turn you take with the agent.

- `UserPromptSubmit` fires when you send a prompt, before Claude reads it.
- `Stop` fires when Claude finishes responding.
- Both can block, which makes them useful gates.
- Prompt-submit can reject or reshape a prompt, and stop can force more work.

### PreToolUse vs. PostToolUse

- **PreToolUse** — fires before a tool runs. It can block the call, so it is your gate for
  guarding risky or unwanted tool use.
- **PostToolUse** — fires after a tool succeeds. It cannot block, because the tool already ran,
  so use it to react, log, or check output.

### Moment-Specific Events

- **PreCompact** — fires just before context compaction and can block it, so you can protect what
  gets summarized away.
- **Notification** — fires when Claude Code sends a notification, and it only observes, so it is
  for alerting you.
- **Permission events** — fire around permission prompts, where a permission request can be
  denied by a blocking hook.

### How a Hook Speaks Back

- **Exit zero** — success. Claude Code reads any structured output the hook printed and continues
  normally.
- **Exit two** — a blocking signal. For hooks that can block, exit two stops the action.
- **Other codes** — treated as a non-blocking error for most events, so the action still proceeds
  anyway.

### Block vs. Observe

The single most useful split is whether a hook can stop an action or only watch it.

- **Can block** — `UserPromptSubmit`, `Stop`, `PreToolUse`, `PreCompact`.
- **Observe only** — `PostToolUse`, `SessionStart`, `SessionEnd`, `Notification`.

`PreCompact` can block compaction before context is trimmed. `PostToolUse` only reacts, since the
tool already finished. `PreToolUse` blocks a tool; `Stop` keeps the agent working. Pick a
blocking event for a gate, an observe-only one for a log.

### Choosing the Right Hook

Match the hook to the job by asking what you want to happen and when.

- Load project context on entry? Use `SessionStart`.
- Guard a dangerous command? Reach for `PreToolUse`.
- Auto-format or lint after an edit? Use `PostToolUse`.

Cadence tells you *when*; the blocking column tells you *whether* you can intervene.

### Key Takeaways: Right Hook for the Job

- Session start and end bracket the session, observe only.
- Prompt submit and stop bracket each turn, can block.
- Pre-tool-use blocks a tool; post-tool-use only reacts.
- Exit code two lets a blocking hook stop an action.
- Pick cadence for *when*, blocking for *whether* you step in.

---

## 29. Escalation Trigger Design

Human-in-the-loop escalation is a risk-mitigation strategy, not a fallback of last resort.

### Three Escalation Concepts

| Term | Definition |
|---|---|
| **Human-in-the-loop (HITL)** | A human is placed in the agent's path at defined checkpoints to review or approve actions. |
| **Escalation trigger** | A defined condition that pauses an agent for human review, such as a confidence drop. |
| **Programmatic enforcement** | Code-layer mechanisms that enforce constraints no matter what the model outputs. |

### HITL as Risk Mitigation

HITL is a deliberate architectural choice to manage risk at specific decision points.

- Not a fallback when the agent is confused — a proactive safety gate.
- Placed where the consequences of an error are high and hard to reverse.
- Misuse as a general fallback trains reviewers to approve reflexively.

Effective HITL is targeted, not ubiquitous.

### Valid Trigger: Irreversible High-Stakes

The strongest signal for a valid escalation trigger is irreversibility combined with stakes.

- Financial transactions, data deletion, account changes — hard to undo if wrong.
- The cost of a mistake exceeds the cost of a human review cycle.
- Irreversibility means the agent cannot self-correct after the fact.

If the agent cannot undo it alone, gate it with a human first.

### Valid Trigger: Confidence and Scope

Two more valid triggers cover ambiguity and boundary violations:

- **Low confidence** — model uncertainty on ambiguous inputs exceeds the safe operating range.
- **Scope boundary** — the task drifts beyond authorized bounds or expected parameters.
- **Retry exhausted** — repeated failure after defined retries signals a structural problem.

Each trigger shares one feature: the agent has reached the limit of its authority.

### Invalid Triggers: Over-Escalation

- **Every uncertain move** — escalating on every uncertain step erodes automation.
- **Low-stakes reversible** — if the action is easily undone, HITL only creates friction.
- **Code-covered** — actions already handled by code need no human review.
- **Gut feel** — ungrounded triggers cause inconsistent escalation.

### Under-Escalation vs. Over-Escalation

- **Under-escalation** — the agent acts on high-stakes decisions without review. Irreversible
  errors occur before humans can intervene. Low friction, but high tail risk on a consequential
  action.
- **Over-escalation** — the agent interrupts constantly on low-stakes steps. Reviewers grow
  fatigued and approve without reading. Defeats automation's purpose; friction becomes the
  dominant cost.

### Designing Triggers as Conditions

Escalation triggers must be defined as explicit, testable conditions before deployment.

- Confidence score below 0.7 on classification — not "when it seems unsure."
- Cost estimate exceeds $500 — not "when the action feels expensive."
- Third retry failure — not "when it keeps failing."

Condition-based triggers are auditable, adjustable, and consistent across runs.

### The Calibration Tradeoff

Getting escalation thresholds right is an iterative calibration process, not a one-time decision.

- Too tight creates excessive interruptions — automation value erodes.
- Too loose allows unchecked risk — safety gates stop doing their job.
- Monitor escalation rates in production and tune from reviewer input.

The right threshold today may be wrong after workflow changes.

### Key Takeaways: Escalation Trigger Design

Valid triggers are targeted, condition-based, and grounded in irreversibility or authority limits.

- HITL is a risk-mitigation strategy, not a general fallback for uncertainty.
- **Valid** — irreversible actions, confidence drops, scope violations, retry exhaustion.
- **Invalid** — low-stakes reversible actions, or steps already covered by code.
- Define triggers as explicit testable conditions, not gut feel.

Calibrate thresholds continuously — friction and risk both have real costs.

---

## 30. Interruption Points and Review Workflows

Where you place interruption points decides whether human review adds safety or just slows
things down.

### Three Review Workflow Concepts

| Term | Definition | In Claude |
|---|---|---|
| **Interruption point** | A defined point where the agent pauses for human input. | `if tool == "transfer": await human_review()` |
| **State preservation** | Persist progress and decisions outside the context window. | `state = {"step": 3, "done": results}` then `save(thread_id, state)` |
| **Fallback chain** | Try pre-validated alternatives when the primary action fails. | `try: primary()` / `except: fallback()` |

### Where to Place Interruption Points

Placement is a design decision with direct consequences for safety and efficiency.

- **Before irreversible actions** — the most critical placement; prevents uncorrectable errors.
- **At phase boundaries** — natural transition points where scope or authority shifts.
- **On anomaly detection** — unexpected inputs, unusual patterns, or out-of-range values.

Placing interruptions after irreversible actions provides no meaningful safety benefit.

### Designing Handoff Messages That Work

A handoff that dumps raw tool output forces reviewers to do the agent's analysis work.

- State what the agent was doing — context the reviewer needs to orient quickly.
- State what decision is required — a specific question, not an open-ended request.
- State the consequences — outcomes if approved, outcomes if denied.

Reviewers decide faster when the message does the translation work for them.

### Four Handoff Message Components

- **Context** — what was the agent doing, and why did it stop? Brief, no jargon.
- **Decision required** — exactly what is the reviewer being asked to approve or reject?
- **Consequences** — what follows from approval? What follows from rejection?
- **Deadline** — when does this review expire, and what happens at timeout?

### State Preservation During Interruption

An agent that can't resume after approval forces the workflow to restart from scratch.

- Persist task progress — which steps completed and what outputs were produced.
- Persist agent decisions — what the model chose at each branch point.
- Do not rely on in-context state — context can be truncated or the session can end.

State preservation separates a resumable workflow from a disposable one.

### Synchronous vs. Async Escalation

- **Synchronous escalation** — the agent halts and waits for human approval before proceeding.
  Simple to implement, but the whole workflow is blocked during review. Best when dependent work
  cannot proceed without that decision.
- **Async escalation** — the agent queues the review and continues work elsewhere. Non-blocking,
  but requires more state management. Best when independent branches can proceed while the review
  is pending.

### Resumption Design After Approval

When approval arrives, the agent needs enough context to continue without re-running completed
work.

- The approval message should include the decision and any reviewer notes.
- The agent loads persisted state and resumes from the interruption point.
- Re-running completed steps wastes time and may produce different outputs.

Design the resumption path explicitly — don't assume the agent can reconstruct context.

### Async Patterns and Timeout Handling

Async escalation requires explicit design for what happens when no human responds in time.

- **Queue and continue** — work on independent branches while awaiting approval.
- **Timeout threshold** — define how long to wait before triggering a fallback.
- **Timeout action** — escalate further, abort, or apply a safe default; never silently hang.

An unhandled timeout is a silent failure. Define the timeout action before deployment.

### Key Takeaways: Review Workflow Design

Effective review workflows require deliberate design across placement, handoffs, state, and
timeouts.

- Place interruptions before irreversible actions, at phase boundaries, on anomalies.
- Handoff messages: context, required decision, and consequences — no jargon.
- Persist state externally — in-context state cannot survive session ends or truncation.
- Async escalation is non-blocking but requires explicit timeout handling.
- Resumption design makes paused workflows recoverable, not disposable.
