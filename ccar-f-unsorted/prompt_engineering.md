# Prompt Engineering — Day 4

*Text extracted from the 233-slide deck and grouped by topic. Diagrams and decorative
elements are omitted; only the substantive text is kept.*

---

## Table of Contents

1. [Clarity, Specificity, and Ambiguity](#1-clarity-specificity-and-ambiguity)
2. [Instruction Decomposition and Constraints](#2-instruction-decomposition-and-constraints)
3. [The System Prompt: Purpose and Structure](#3-the-system-prompt-purpose-and-structure)
4. [Role Assignment, Persona, and Context](#4-role-assignment-persona-and-context)
5. [XML Tags for Complex Prompts](#5-xml-tags-for-complex-prompts)
6. [Document Structure for Multi-Part Tasks](#6-document-structure-for-multi-part-tasks)
7. [Why Examples Work in Prompting](#7-why-examples-work-in-prompting)
8. [Designing High-Quality Examples](#8-designing-high-quality-examples)
9. [Chain-of-Thought Prompting](#9-chain-of-thought-prompting)
10. [Extended Thinking: When to Use It](#10-extended-thinking-when-to-use-it)
11. [Steering Model Output](#11-steering-model-output)
12. [Sampling Parameters: Legacy Knobs](#12-sampling-parameters-legacy-knobs)
13. [Designing Multi-Turn Interactions](#13-designing-multi-turn-interactions)
14. [Conversation State and Turn Management](#14-conversation-state-and-turn-management)
15. [Guaranteed JSON with Structured Outputs](#15-guaranteed-json-with-structured-outputs)
16. [Schema Design for Reliable Extraction](#16-schema-design-for-reliable-extraction)
17. [Structured Outputs Limits](#17-structured-outputs-limits)
18. [Tool Definitions for Extraction](#18-tool-definitions-for-extraction)
19. [Validation Loops and Hallucination Prevention](#19-validation-loops-and-hallucination-prevention)
20. [Validation Gates and Retry Strategy](#20-validation-gates-and-retry-strategy)
21. [Multi-Pass Review](#21-multi-pass-review)
22. [Building a Prompt Evaluation Workflow](#22-building-a-prompt-evaluation-workflow)
23. [Test Datasets and Model-Based Grading](#23-test-datasets-and-model-based-grading)

---

## 1. Clarity, Specificity, and Ambiguity

Vague instructions produce inconsistent outputs — specificity is the primary lever for quality.

### Three Core Concepts

| Term | Definition | Example |
|---|---|---|
| **Specificity** | How precisely an instruction defines the output. | "A three-sentence summary." |
| **Ambiguity** | An instruction with more than one meaning. | "Make it better." |
| **Direct instruction** | Say what to do, not what to avoid. | "List three risks." |

### Why Vague Instructions Fail

When an instruction is vague, the model has to choose among several valid interpretations.

- Each plausible interpretation produces a different output.
- Outputs vary across runs even with the same prompt.
- Variation reflects the prompt's openness, not a flaw in the model.
- Debugging vague outputs takes far longer than writing precise inputs.

Precision at the prompt stage prevents variance at every later stage.

### Abstract vs. Concrete Instructions

- **Abstract (vague)** — "Write a summary of this document." The model must decide length,
  format, tone, detail, and audience. Every choice is an assumption.
- **Concrete (specific)** — "Write a three-sentence summary for a non-technical audience. Include
  the main finding, one supporting detail, and the recommended action."

### Specificity as the Primary Lever

Specificity is the single most reliable variable under your control when writing instructions.

- Tone, format, length, perspective: each one narrows the output space.
- Narrow output spaces produce consistent, repeatable results.
- You can improve any output by adding one more specific constraint.
- Specificity compounds: each added constraint multiplies precision.

Treat specificity as a dial you can always turn up when outputs disappoint.

### The Direct Instruction Principle

- **State what to do** — tell the model exactly what output you want. "Write a list of three
  risks" is direct.
- **Avoid negatives** — saying "don't be vague" leaves the goal undefined; the model still must
  guess.
- **Be actionable** — map each instruction to a specific action: write, list, compare, or extract.
- **One goal per instruction** — compound instructions split attention. State one clear goal, then
  add constraints if needed.

### How Ambiguity Forces Assumptions

Ambiguity does not stop the model — it makes the model choose an interpretation and proceed.

- The model picks the most statistically likely interpretation.
- That interpretation may not match what the user intended.
- Neither party is aware a different interpretation was possible.
- The result looks plausible but may be subtly or significantly wrong.

The danger of ambiguity is that wrong assumptions often go unnoticed.

### The Cost of Ambiguity

Imprecise instructions impose real costs across the full prompt-to-output workflow.

- Output variance means no two runs produce the same result.
- Unpredictability makes evaluation and iteration much harder.
- Debugging requires re-reading the prompt to find the gap, not the output.
- Over many iterations, ambiguous prompts waste more time than writing specifics up front.

Ambiguity costs are hidden at writing time and paid at review time.

### Turning Vague into Specific

- **Add a format** — specify output structure: a bullet list, numbered steps, a table, or a
  paragraph.
- **Add a length** — state a word count, a sentence count, or a bound like "in one paragraph."
- **Add an audience** — name who will read the output. The audience sets the right tone and
  vocabulary.
- **Add a perspective** — specify first person, third person, or a named role like "senior
  analyst."

### Key Takeaways: Clarity and Specificity

Specificity is the primary lever for output quality and consistency.

- Vague instructions force the model to fill gaps with assumptions.
- The model's assumptions are invisible and may not match your intent.
- Direct instructions state what to do, not what to avoid.
- Every added constraint narrows the output space and reduces variance.

Specificity saves hours at review time.

---

## 2. Instruction Decomposition and Constraints

Complex instructions fail when one call carries too many competing goals — break them into
ordered steps: complex task → ordered steps → verified output.

### Three Decomposition Concepts

| Term | Definition | Example |
|---|---|---|
| **Instruction decomposition** | Break a complex task into ordered sub-steps. | Extract, then summarize. |
| **Scope definition** | State what to include, exclude, and assume. | Cover only section 2. |
| **Constraint encoding** | Turn requirements into specific parameters. | 100 words, formal tone. |

### Why Decompose Instructions

A single complex instruction forces the model to balance competing goals at once.

- Competing goals reduce the quality of each individual outcome.
- Tasks split into steps produce better results than one large prompt.
- Each step can be checked independently before the next runs.
- Ordered steps make it easy to pinpoint where an output goes wrong.

Decomposition is the primary tool for managing complex prompt work.

### Breaking Instructions into Steps

When a task has multiple phases, give each phase its own instruction in sequence.

1. **Extract** — pull the relevant information from the source.
2. **Summarize** — summarize the extracted content in the required format.
3. **Check** — identify gaps or inconsistencies in the summary.

Each step gets one goal, one output type, and one verification check.

### Defining Scope and Boundary

- **What to include** — name the elements the output should cover.
- **What to exclude** — list what the model should not address.
- **What to assume** — state the background facts to treat as given.
- **Scope boundary** — define where the task ends and output stops.

### Single Instruction vs. Decomposed

- **Single instruction** — "Analyze this contract, identify the key risks, summarize the
  obligations for each party, and flag conflicting clauses." Too many goals in one call.
- **Decomposed steps** — First, list every obligation per party. Next, assess each as standard or
  unusual. Last, summarize only the unusual ones. Three calls, one goal each.

### Encoding Constraints: The Four Types

Constraints are parameters that narrow the output space without changing the core task.

- **Format** — structure of the output, such as bullet list, table, or paragraph.
- **Length** — word count, sentence count, or a natural scope boundary.
- **Tone** — formal, informal, technical, plain-language, or a specific register.
- **Perspective** — first person, third person, or a named role or persona.

Each constraint type closes a different category of interpretation gap.

### When to Split One Into Many

Not every task needs decomposition, but clear signals indicate when splitting helps.

- The task has distinct phases that produce different output types.
- You need to verify one phase before the next can run correctly.
- The instruction needs many constraints to even be roughly precise.
- Prior single-prompt attempts consistently missed part of the task.

When heavy constraint loading is needed, splitting is usually faster.

### Testing Instruction Specificity

- **Ambiguity test** — could a reasonable person read this in more than one way? If yes, resolve
  the ambiguity.
- **Interpretation gap** — what must the model assume to complete this? Identify the assumptions
  that matter, then close them.
- **Output preview** — mentally trace the likely output. If two outputs seem equally plausible,
  add a constraint.
- **Scope check** — does the instruction have a stopping point? If not, add a length or scope
  boundary.

### Key Takeaways: Decomposition and Constraints

Complex instructions break into ordered steps, each with one clear goal.

- Scope defines what to include, exclude, and assume.
- Four constraint types: format, length, tone, perspective.
- Split when phases are distinct or verification is needed.
- Test specificity by checking interpretations and assumptions.

Decomposed instructions are easier to verify and debug.

---

## 3. The System Prompt: Purpose and Structure

The system prompt is the foundation every model response is built on.

### Three Core Concepts

| Term | Definition | Example |
|---|---|---|
| **System prompt** | Instructions set before the human turn. | "You are a support agent." |
| **User turn** | The human message in each exchange. | "Reset my password." |
| **Persistent context** | Information that applies to every turn. | The support policy. |

### System vs. User Turn: What Goes Where

- **System prompt (stable)** — holds role, standing constraints, formatting rules, and domain
  knowledge. Stable content that applies to the whole session belongs here. *Example: "always
  reply in JSON."*
- **User turn (per-request)** — holds the immediate request, per-request data, and dynamic
  variables. Mixing stable and request-specific content blurs ownership and wastes tokens.
  *Example: "summarize this email."*

### Five Sections of a System Prompt

- **Role** — who the model is: its expertise, professional identity, and tone of voice.
- **Context** — domain knowledge, policies, and data the model needs throughout the session.
- **Instructions** — what the model should do: task behaviors, response format, and workflow steps.
- **Constraints** — what the model must never do: prohibited topics, length, tone rules.
- **Examples** — sample inputs and outputs that illustrate the rules already stated above.

### Persistent vs. Per-Request Context

Choosing the right context type prevents both wasted tokens and unpredictable behavior.

- Persistent context lives in the system prompt and applies to every turn.
- Per-request context is injected in the user turn for that exchange only.
- Putting ephemeral data in the system prompt inflates every request's token count.

A customer's account history is per-request; the support policy is persistent.

### Thoroughness vs. Token Cost

- **Thorough system prompt** — covers edge cases, explicit formatting rules, and examples. Reduces
  ambiguity and improves consistency, though it costs more tokens because the entire prompt is
  sent again on every request.
- **Minimal system prompt** — short and focused. Lower token cost and faster first response. Risk:
  the model fills gaps with defaults that may not match intent, requiring more user-turn
  correction.

### Ordering Sections for Clarity

Section order affects how reliably the model interprets and follows instructions.

1. **Role** — frames everything that follows.
2. **Context** — background the model needs.
3. **Instructions** — define behavior first.
4. **Constraints** — limits come after behavior.
5. **Examples** — illustrate the rules last.

### Writing Instructions That Don't Conflict

Conflicting instructions are a leading cause of inconsistent, hard-to-debug behavior.

- Audit overlapping rules: "be concise" vs. "always explain your reasoning" clash.
- Use conditional phrasing to separate contexts: "when the user asks X, do Y."
- Avoid stacking negations: restate as positive guidance wherever possible.

Test with inputs that could trigger two rules at once to surface hidden conflicts.

### Testing System Prompt Effectiveness

A system prompt that works on the obvious case may still fail on edge inputs.

- Test across diverse phrasings of the same request.
- Use adversarial inputs that directly contradict your constraints.
- Check boundary conditions: very long inputs, empty inputs, topic switches.
- Compare outputs before and after edits to isolate each change.

Treat system prompt testing like unit testing: define expected behavior, check regressions.

### Key Takeaways: System Prompt Structure

The system prompt sets persistent context; the user turn sets per-request context.

- Five sections: role, context, instructions, constraints, examples.
- Stable information in the system prompt, dynamic data in user turns.
- Balance thoroughness against the token cost of re-sending every request.
- Order matters: role first, examples last, high-priority rules near the top.

Test across diverse inputs to surface conflicts early.

---

## 4. Role Assignment, Persona, and Context

Who a model is told it is shapes how it thinks, writes, and reasons.

### Role, Persona, and Context Defined

| Term | Definition | Example |
|---|---|---|
| **Role assignment** | Tell the model what expertise to adopt. | "You are a tax attorney." |
| **Persona** | A named identity with its own voice. | "Aria, a friendly guide." |
| **Context injection** | Place real data into the prompt. | Paste the policy doc. |

### How Role Shapes Model Behavior

A role assignment is not decorative — it changes vocabulary, reasoning depth, and tone.

- "Senior security analyst" shifts toward technical, risk-focused framing.
- "Customer support agent" shifts toward empathetic, solution-focused framing.
- Formality level and assumed audience shift along with the role.

Role assignment is the fastest lever for aligning behavior with your use case.

### Role Assignment vs. Persona

- **Role assignment** — labels functional expertise: "You are a tax attorney specializing in
  corporate law." Shapes how the model reasons and what knowledge it foregrounds. No name or
  personality traits required.
- **Persona** — adds a named identity and stylistic traits: "You are Aria, a friendly onboarding
  guide who uses plain language and short sentences." Extends role with character and
  communication style.

### Context Injection: Grounding in Facts

Context injection grounds responses in specific, current information, not training data alone.

- Without injection, the model relies on potentially stale training knowledge.
- With injection, it reasons over the documents or data you provide.
- System prompt injection is persistent for the whole session.
- User-turn injection is scoped to that single exchange.

Injection extends what the model reasons over without updating its weights.

### What to Put Where: Placement Guide

- **System prompt** — stable role, standing policy, domain reference data that applies to every
  turn.
- **User turn** — per-user data, per-request documents, dynamic variables that change with each
  exchange.
- **Assistant prefill** — a legacy pattern: seed the assistant reply to steer format. Final-turn
  prefill now returns a 400 on current models.

### Designing Personas That Stay Stable

A persona that drifts under pressure is a reliability problem, not just a stylistic one.

- Define the persona's communication style explicitly: tone, sentence length, formality.
- Include constraints on what the persona will not do or say.
- Anticipate edge cases: off-topic requests, adversarial prompts, emotional inputs.
- Test with inputs designed to destabilize the persona before deploying.

Stability comes from specificity; vague persona descriptions produce inconsistent behavior.

### Role Does Not Grant New Capabilities

Role assignment shapes *how* the model reasons — it does not change *what* the model can do.

- Telling the model it is a chemistry expert does not add knowledge it lacks.
- Role does not override safety behaviors or ethical constraints.
- A persona cannot bypass the model's built-in refusals.

Understanding this boundary is essential when setting user expectations.

### Testing Role and Context Consistency

Role and context produce consistent behavior only when tested under realistic variation.

- **Topic switches** — does the role hold when the user goes off-script?
- **Context retrieval** — does the model use injected data over training priors?
- **Persona pressure** — can users override the assigned identity?
- Baseline expected outputs before deploying to production.

Consistency testing is the only way to confirm your design is production-ready.

### Key Takeaways: Role, Persona, and Context

Role assignment sets functional expertise; persona adds a named identity and style.

- Role shapes vocabulary, tone, and reasoning for every response.
- Persona extends role with personality traits and communication style.
- Context injection grounds the model in facts within the context window.
- System prompt context is persistent; user-turn context is per exchange.

Specific role and persona design produces stability.

---

## 5. XML Tags for Complex Prompts

XML tags are the most reliable way to structure complex prompts for Claude.

### Three XML Prompt Concepts

| Term | Definition |
|---|---|
| **XML tags** | Structured markup elements like `<document>` or `<instructions>` that label prompt sections. Claude is trained to respect XML boundaries reliably. |
| **Prompt injection** | An attack where malicious input overrides developer instructions. XML tags help label data vs. instructions, but are not primary defense. |
| **Tag consistency** | Using the same tag names throughout a prompt and across related prompts. Consistent naming reinforces the structural signal Claude uses to parse. |

### Why XML Beats Plain Delimiters

Claude's training reinforces XML tag boundaries more reliably than prose separators.

- Plain text delimiters like dashes or headers are ambiguous.
- Claude may misread them as content rather than structure signals.
- XML tags have unambiguous open and close syntax.
- The model was trained on structured documents, not ad-hoc separators.

When parsing reliability matters, choose XML tags over markdown headers.

### Standard Tag Conventions

- **`<instructions>`** — wraps the task directive the model should follow.
- **`<context>`** — holds the background information the task needs.
- **`<examples>`** — contains few-shot demonstrations of the desired output.
- **`<output_format>`** — specifies the structure and format you expect back.

### Tag Naming Best Practices

- **Be descriptive** — name tags by content type: `<customer_data>` is clearer than a vague
  `<data>` tag.
- **Stay consistent** — use the same tag names for the same roles across every prompt.
- **Avoid deep nesting** — one or two levels is enough. Deeply nested tags get hard to read.
- **Match your schema** — if you use `<document>` in one prompt, reuse that same tag in every
  prompt.

### Flat Tags vs. Nested Tags

- **Flat tags** — each section gets one top-level tag, such as `<context>` or `<examples>`. Best
  for simple prompts where sections are independent and need no hierarchy.
- **Nested tags** — parent tags contain children: `<document>` wraps `<source>` and `<content>`.
  Use nesting when a section has internal structure the model parses separately.

### Nested Tags for Hierarchical Data

Nested XML tags represent structured data without ambiguity.

```xml
<document>
  <source>
    <date>...</date>
  </source>
  <content>...</content>
</document>
```

Children belong to the parent: content stays tied to its source. One or two levels of nesting is
the practical limit.

### The Real Defense Against Injection

Tag separation adds clarity, but it is not what stops prompt injection.

- Treat every tool result and document as untrusted input.
- Apply least privilege so a bad input can't trigger harmful actions.
- Tags only make the model less likely to confuse data for a directive.
- An adversarial payload can still break out of a tag boundary.

Untrusted-input handling and least privilege are the real defense.

### Tag Consistency and Follow-Through

- **Repeated tag names** — use the same tag name each time.
- **Opening and closing** — every tag needs a matching close.
- **Cross-prompt consistency** — align tag names across all prompts.

### Key Takeaways: XML Tag Structure

XML tags are the most reliable structural tool for complex prompts.

- Claude's training reinforces tag boundaries over plain text delimiters.
- Standard tags: `<instructions>`, `<context>`, `<examples>`, `<output_format>`.
- Tags help separate data from instructions, but don't prevent injection.
- Consistent tag naming across prompts strengthens parsing follow-through.

---

## 6. Document Structure for Multi-Part Tasks

Multi-part task prompts need clear document structure to stay coherent across sources.

### Three Document Structure Terms

| Term | Definition | Example |
|---|---|---|
| **Multi-document prompt** | A prompt with two or more sources. | Two articles to compare. |
| **Document injection** | Placing source text in with XML tags. | `<document>` wraps text. |
| **Output attribution** | Linking output to the right source. | Cite source A or B. |

### Multi-Document Prompt Patterns

When a task references multiple sources, each document needs its own labeled block.

```xml
<document index="1">
  <source>...</source>
  <document_content>...</document_content>
</document>
```

Wrap each document, name its source, and place the text in `document_content`. Keep every block
self-contained and consistently structured.

### Sections vs. Separate Turns

Deciding between sections and separate turns depends on task complexity.

- Use sections when all inputs need to be reasoned over together.
- Use separate turns when each step builds on the prior response.
- Sections keep context in one call but make prompts large.
- Separate turns preserve context window and allow course correction.

Choose sections for parallel reasoning, separate turns for sequential steps.

- **Single prompt with sections** — all documents and instructions are in one call. The model
  reasons over everything simultaneously. Best for tasks that require comparing or synthesizing
  multiple sources at once. *Example: compare three sources.*
- **Separate conversational turns** — documents and instructions arrive across multiple messages.
  Each turn builds on the prior response. Best for sequential workflows where intermediate outputs
  inform the next step.

### Document Injection Tag Patterns

Three tags plus an optional attribute cover most document injection scenarios.

- `<document>` is the top-level container for a single source.
- `<document_content>` holds the raw text of the source material.
- `<source>` provides metadata: filename, URL, or a short identifier.
- Attribute numbering like `document index="1"` helps with citation.

These three tags together produce clean, parseable document blocks.

### Reference Patterns in Instructions

Instructions can cite documents by their source metadata or index number.

- **Reference by source** — "Based on the document from source A, summarize…"
- **Reference by index** — "Using document 1 and document 2, compare…"
- Explicit references help the model attribute outputs to the right source.
- Avoid vague phrases like "the documents" when only one should be used.

Explicit citation in instructions reduces misattribution in complex prompts.

### Anti-Patterns to Avoid

- **Inline prose mixing** — embedding document text directly in instructions without tags. The
  model cannot reliably separate what to follow from what to analyze.
- **Untagged blocks** — placing instructions and source material in the same untagged block. This
  is the main source of prompt injection risk.
- **No structure at all** — sending a long, untagged prompt with multiple documents.
  Misattribution becomes likely as prompt length grows.

### Testing Output Attribution

Testing that the model attributes outputs correctly is a necessary step.

- Ask the model to cite which document each claim comes from.
- Test with documents that contain conflicting information on purpose.
- Verify that output from one document does not bleed into another's answers.
- Use structured output formats to make attribution explicit and checkable.

Attribution testing surfaces structure problems before they reach production.

### Key Takeaways: Document Structure

Structured document injection keeps multi-part task prompts reliable.

- Tag each source with `<document>`, `<document_content>`, and `<source>`.
- Choose sections for parallel reasoning, separate turns for sequential steps.
- Reference documents explicitly by index or source name in instructions.
- Anti-patterns: inline prose mixing, untagged blocks, no structure.

Test attribution with conflicting documents to confirm it works.

---

## 7. Why Examples Work in Prompting

Examples communicate format, tone, and reasoning patterns in a single demonstration.

### Three Few-Shot Prompting Terms

| Term | Definition | Example |
|---|---|---|
| **Few-shot prompting** | Two or more examples before the task. | Show three samples first. |
| **Zero-shot prompting** | A task with no examples at all. | Just ask, no samples. |
| **One-shot prompting** | Exactly one example provided. | One model answer. |

### What Examples Communicate

A single example carries more information than most prose instructions can convey.

- **Format** — the exact structure the output should follow.
- **Tone** — formal, conversational, terse, or detailed.
- **Reasoning pattern** — how to approach the task step by step.
- **Scope** — what to include and what to leave out.

Examples do this simultaneously, while prose addresses one dimension at a time.

### How Examples Anchor Patterns

Models use examples as templates, not just hints.

- The model identifies structural patterns across all examples provided.
- It extracts implicit rules: length, phrasing style, output sequence.
- Those rules apply even when the new input differs from the examples.
- Examples tend to transfer format and tone more reliably than prose.

Pattern matching from examples produces more consistent output than instructions alone.

### Prose Instructions vs. Examples

- **Prose instructions** — describe the desired output in natural language. The model must
  interpret what the description means and translate it to output. Interpretation introduces
  variance, especially for novel formats.
- **Examples** — show the desired output directly. The model can mirror the demonstrated pattern.
  Format, tone, and structure tend to transfer implicitly and more reliably.

### Format Anchoring from Token One

Format anchoring locks output structure from the first token produced.

- Examples that end in a specific format get continued from token one.
- The first output token sets the structural path for all that follows.
- A well-placed example locks structure before any drift can occur.
- Without an example, the model infers format from instruction text alone.

Examples anchor format earlier and more reliably than prose descriptions.

### Shot Count Tradeoffs

- **Zero-shot** — no examples. Relies on instruction clarity. Best for common tasks where format
  flexibility is acceptable.
- **One-shot** — one example. Works when a single canonical case fully defines the pattern. Low
  overhead, clear signal.
- **Three-shot** — covers most format and tone variation. The most broadly useful choice for
  structured output.
- **Five-plus-shot** — diminishing returns beyond three. Useful only when examples span genuinely
  different input types.

### Zero-Shot vs. Few-Shot: When to Use Each

Choose based on task type and output format specificity.

- Zero-shot works for summarization, Q&A, and familiar text tasks.
- Few-shot is preferred for structured output and novel formats.
- Zero-shot with clear instructions can match few-shot on common tasks.

Few-shot overhead should be justified by the consistency gain.

### Quality Over Quantity

- **One strong example** — a single well-crafted example covering the target format, tone, and
  edge cases outperforms several mediocre ones.
- **Five weak examples** — examples with inconsistent formatting or off-tone phrasing introduce
  conflicting signals. The model averages them.

Example quality is the single most important variable in few-shot performance. Audit your
examples before adding more.

### Key Takeaways: Why Examples Work

Examples communicate format, tone, and reasoning more reliably than prose alone.

- **Pattern matching** — models extract implicit rules from demonstrated output.
- **Format anchoring** — examples lock structure from the first output token.
- Three-shot is the most broadly useful count for structured tasks.
- Zero-shot is appropriate for common tasks with flexible output format.

Example quality is the single biggest lever for output consistency.

---

## 8. Designing High-Quality Examples

Few-shot performance depends almost entirely on the quality of the examples you provide.

### Three Example Design Terms

| Term | Definition | Example |
|---|---|---|
| **Example contamination** | Narrow examples make the model copy quirks. | All outputs copy a quirk. |
| **Dynamic few-shot** | Pick examples by similarity at runtime. | Pick similar samples. |
| **Ordering effects** | Example order changes the output. | Best example last. |

### Input-Output Pair Design

Each example must cleanly demonstrate what you want the model to generalize.

- Input should be realistic and representative of actual task inputs.
- Output should be exactly what you would accept as a correct response.
- Do not include in the output anything you wouldn't want reproduced.
- Keep examples focused: one clear pattern per example generalizes best.

Weak examples that *almost* show the pattern introduce more noise than signal.

### Example Diversity and Edge Cases

A set of similar examples teaches one pattern; diverse examples teach a range.

- Cover different input lengths, topics, and phrasing styles.
- Include at least one edge case that tests the boundary of the pattern.
- Avoid clustering all examples around the most common case.
- Diversity ensures the model generalizes rather than overfitting to one type.

Three diverse examples outperform five examples from the same narrow slice.

### Static vs. Dynamic Few-Shot

- **Static few-shot** — a fixed set of examples used for every input. Easy to implement and
  maintain. Can produce inconsistent results when real inputs differ a lot from the example
  inputs.
- **Dynamic few-shot** — examples are selected at runtime by similarity to the current input.
  More consistent across varied inputs. Needs a retrieval step but improves output quality.

### Avoiding Example Contamination

Contamination happens when examples constrain output in unintended ways.

- Stylistically skewed examples reproduce unusual phrasing across all outputs.
- Topic-biased examples cause the model to assume all inputs share that topic.
- Overly long example outputs train the model toward excessive length.
- Artifacts unique to one example get copied verbatim into other responses.

Review each example output as if it were a response you would ship.

### Dynamic Few-Shot Selection

Dynamic selection matches examples to the current input at runtime.

1. Example pool
2. Embed for similarity
3. Retrieve top-k matches
4. Inject into prompt

At inference, retrieve the most similar examples and pass them in place of a static set. Dynamic
selection is the most reliable approach for diverse or open-ended workloads.

### Ordering Effects in Few-Shot Prompts

- **Recency bias** — models give more weight to later examples. The final example has the
  strongest influence on output format and tone.
- **Put best last** — place your most representative, highest-quality example last. It anchors
  output more strongly than earlier examples.
- **Vary order in testing** — if output quality varies unexpectedly, try reordering examples.
  Ordering can produce significant differences in complex prompts.

### Testing Whether Examples Help or Hurt

Examples should always be tested, not assumed to be working.

- Run the prompt with examples, then without, and compare output quality.
- If zero-shot matches few-shot quality, the examples are adding noise.
- If output reproduces example artifacts, contamination has occurred.
- Test on diverse inputs, not just the canonical case the examples resemble.

Remove or replace examples that hurt quality rather than patching around them.

### Key Takeaways: Few-Shot Example Design

High-quality examples require deliberate design, not just writing something down.

- Each input-output pair should be realistic, focused, and artifact-free.
- Diverse examples generalize better than many from one narrow slice.
- Dynamic selection outperforms static sets on varied workloads.
- Ordering effects are real: place the best example last.

Always test examples against zero-shot to confirm they help.

---

## 9. Chain-of-Thought Prompting

Chain-of-thought prompting improves accuracy by asking the model to show its reasoning:
question → step-by-step reasoning → answer.

### Three Chain-of-Thought Concepts

| Term | Definition | Example |
|---|---|---|
| **Chain-of-thought prompting** | Show reasoning before the answer. | "Think step by step." |
| **Zero-shot CoT** | Trigger reasoning with no examples. | Add "think step by step." |
| **Structured CoT** | Give a labeled reasoning scaffold. | "Identify, then analyze." |

### Zero-Shot CoT: Think Step by Step

Zero-shot chain of thought requires no examples — just an elicitation phrase.

- Append "Think step by step" to trigger visible reasoning.
- Variants: "Walk me through your reasoning" or "Explain before answering."
- Effective for arithmetic, logic puzzles, and multi-step planning tasks.
- Omitting the trigger phrase and expecting reasoning to appear on its own is a common error.

Without the elicitation phrase, chain of thought does not activate.

### Structured CoT: Providing a Scaffold

A structured scaffold guides the model through a fixed reasoning pattern. A scaffold gives
consistent reasoning across related tasks; free-form reasoning fits tasks whose steps keep
changing.

### Free-Form vs. Structured CoT

- **Free-form CoT** — the model chooses its own reasoning path after an elicitation phrase. Best
  when tasks vary widely in structure and a fixed template would constrain useful reasoning.
- **Structured CoT** — a reasoning scaffold is provided in the prompt, directing the model
  through labeled steps. Best when consistency across responses is required or the reasoning
  steps are known.

### When CoT Helps Most

- **Multi-step arithmetic** — problems with several calculation steps benefit from visible
  intermediate work, reducing error accumulation.
- **Logic and deduction** — formal reasoning where incorrect steps compound. Visible reasoning
  makes errors catchable before the final answer.
- **Planning and sequencing** — tasks that order actions or resolve dependencies benefit from
  explicit step-by-step thinking.
- **Complex instructions** — prompts with multiple constraints are easier to satisfy when the
  model reasons through each requirement explicitly.

### When CoT Adds No Value

Chain of thought is not universally helpful — some tasks gain nothing from it.

- Simple factual lookups don't benefit from reasoning steps.
- Classification tasks with clear signals need no intermediate reasoning.
- Short direct-answer tasks add latency without gain.
- Creative tasks may be constrained by forced step-by-step structure.

CoT adds cost — use it only when task complexity justifies the overhead.

### CoT Is a Prompting Technique

Chain-of-thought works entirely through the prompt text you write, not a setting.

- No special API flag or setting activates chain of thought.
- The entire mechanism lives in the instructions you write.
- It applies equally across models that support instruction following.
- CoT is distinct from extended thinking, which is a separate API feature.

Understanding this distinction prevents common mistakes on API design questions.

### Testing CoT Effectiveness

Testing whether chain of thought helps is an important practical step.

- Run the same task with and without the elicitation phrase.
- Compare output accuracy across representative test cases.
- Check whether reasoning steps are coherent and support correct answers.
- Identify tasks where CoT produces longer but not more accurate responses.

Empirical testing reveals where the technique earns its cost.

### Key Takeaways: Chain-of-Thought

CoT prompting improves accuracy by making reasoning visible.

- Zero-shot CoT uses an elicitation phrase like "Think step by step."
- Structured CoT provides a scaffold for consistent reasoning.
- CoT helps most for arithmetic, logic, planning, and complex instructions.
- CoT is a prompt-text technique, not an API parameter or extended thinking.

Test whether CoT improves accuracy before broad use.

---

## 10. Extended Thinking: When to Use It

Extended thinking lets Claude reason internally before its visible answer, enabled via the
thinking parameter.

### Three Extended Thinking Concepts

| Term | Definition |
|---|---|
| **Extended thinking** | An API feature for internal thinking blocks before the visible response. Enabled via the thinking parameter, not prompt text. |
| **Thinking blocks** | Internal reasoning segments produced during extended thinking. Streamed as distinct event types before the final response; not part of the output text. |
| **Adaptive mode + effort** | Claude decides how much to reason, guided by an effort level (high is default; xhigh and max go deeper). Cost is capped by max tokens. |

### CoT Prompting vs. Extended Thinking

- **Chain-of-thought prompting** — a prompting technique applied in the prompt text. No API
  parameters required. The model's reasoning appears in the visible response as part of the
  output text.
- **Extended thinking** — an API feature enabling internal reasoning blocks, tuned by an effort
  level. Reasoning is streamed separately from the final response and is not part of the output
  text.

### Controlling Reasoning Depth

On current models, extended thinking runs in adaptive mode.

- Claude decides how much to reason, not a fixed budget.
- You steer depth with an effort level: low, medium, high (default), xhigh, max.
- Max tokens caps thinking plus output together.
- A fixed token budget is legacy, for older models.

Match effort to complexity to balance depth against cost.

### When Extended Thinking Is Worth the Cost

Extended thinking is most valuable when problems have genuine internal complexity.

- Multi-step reasoning tasks where chain of thought in the prompt is insufficient.
- Problems requiring evaluation of multiple solution paths before committing.
- Tasks where final answer quality depends on thorough pre-answer reasoning.
- High-stakes decisions where reasoning quality matters more than speed.

For simple or direct tasks, extended thinking adds cost without benefit.

### Thinking Blocks in Practice

- **Separate from output** — streamed as distinct event types before the final response; not part
  of the answer text.
- **Count against context** — thinking blocks consume context tokens even though they don't
  appear in the final answer.
- **Not prompt text** — generated internally via the API feature, not written into the prompt by
  the developer.
- **Surfaced as events** — clients handle streaming event types explicitly to read or display
  thinking block content.

### When Extended Thinking Is Justified

Extended thinking is not worth it for every task — complexity thresholds matter.

- Problems with single clear answers rarely benefit from extended thinking.
- Tasks integrating many constraints or data points are better candidates.
- Exploratory tasks where the solution path is not obvious are good fits.
- Tasks where chain of thought already produces good results may not need it.

Evaluate complexity before committing to the added cost.

### Implementation: CoT vs. Extended Thinking

The implementation path differs significantly between the two techniques.

- **Chain of thought** — add an instruction to the prompt; no API changes needed.
- **Extended thinking** — enable adaptive thinking and set an effort level.
- CoT reasoning appears inline in the response; thinking blocks stream separately.
- Extended thinking requires client-side handling of streaming event types.

Implementation complexity favors trying chain of thought first.

### Control Differences

- **Effort control** — you steer reasoning depth with an effort level. Chain of thought gives no
  equivalent control over reasoning length.
- **Separate blocks** — extended thinking streams reasoning as separate blocks. Chain of thought
  embeds reasoning inline in the response text.
- **Bounded cost** — max tokens caps thinking plus output, so cost stays bounded. Chain of
  thought length is harder to bound.

### Key Takeaways: Extended Thinking

- Extended thinking is an API feature, not a prompting technique.
- Adaptive mode and an effort level control internal reasoning depth.
- Thinking blocks count against context but do not appear in the output text.
- Use it when problem complexity exceeds what chain-of-thought prompting addresses.
- Implementation requires API parameters and streaming event handling.
- Chain-of-thought prompting remains the simpler first choice for most tasks.

---

## 11. Steering Model Output

Shaping how Claude formats a response is a system-prompt and format-control problem, not a
token-injection trick. This module covers why the newer flagship models no longer accept
prefilling, what replaced it, and how system-prompt steering directs tone, structure, and scope
before generation begins.

### Prefilling on the Flagships

- **What prefilling was** — ending the messages array with a partial assistant turn so the model
  continued from a chosen opening token.
- **Where it stopped** — the 4.6 generation and later reject it with a 400. Pre-4.6 models like
  Haiku 4.5 still allow it.
- **What to do instead** — reach for structured outputs, strict tool use, or system-prompt
  steering. Each achieves format control by supported means.

### Three Steering Concepts

| Term | Definition |
|---|---|
| **Output steering** | Guiding the model toward a target tone, structure, or scope using supported instructions rather than injecting a partial response turn. |
| **System-prompt steering** | Setting persistent behavior in the system prompt so tone and format hold across every turn in the conversation. |
| **Format control** | Directing the shape of a response through explicit instructions, examples, and schema-backed features that guarantee structure. |

### The Three Supported Replacements

When prefilling would have shaped output, three supported features now cover the same needs.

- **Structured outputs** guarantee schema-valid JSON in the text response.
- **Strict tool use** guarantees valid arguments on every tool call.
- **System-prompt steering** shapes tone, scope, and format across turns.

### What System-Prompt Steering Controls

The system prompt sets behavior that should persist through the whole conversation.

- **Tone** — set a formal, plain, or technical register once.
- **Audience** — name who the answer is for to shape depth.
- **Scope** — bound what the model should and should not cover.
- **Format defaults** — state a preferred structure for every reply.

It applies to every turn, so it is the natural home for steering.

### System Prompt vs. User Turn

- **System prompt** — persistent instructions that shape behavior across every turn. Best for
  tone, role, and format defaults that should never drift during the conversation.
- **User turn** — per-request instructions that apply to a single response. Best for one-off
  format requirements or a specific structure needed just this once.

### Format Control Without Prefilling

Shaping response structure no longer needs a partial assistant turn to anchor the first token.

- State the target format plainly in the instructions.
- Show one or two examples of the exact shape you want.
- Name required fields and their value types explicitly.
- For guaranteed JSON, use structured outputs instead.

Instructions plus examples handle most needs; schema features handle the strict ones.

### When To Use Which Feature

- **Need guaranteed JSON** — use structured outputs. It conforms to your schema normally;
  refusals and max-tokens cutoffs still need handling.
- **Need valid tool arguments** — use strict tool use. It guarantees each tool call's arguments
  match the input schema exactly.
- **Need tone, scope, or style** — use system-prompt steering. It shapes character and format
  across every turn without a schema.

### Steering Anti-Patterns

Over-steering degrades quality by forcing the model to satisfy conflicting or redundant
constraints.

- Reaching for prefilling out of habit now triggers a hard error.
- Repeating the same rule in the system prompt and every user turn adds noise.
- Conflicting signals, like "brief but exhaustive," confuse the model.
- Over-constraining structure when content needs room truncates the answer.

Steer with the lightest control that reliably gets the shape you need.

### Key Takeaways: Steering

- Prefilling returns HTTP 400 on the 4.6 generation and later; pre-4.6 models like Haiku 4.5
  still allow it.
- Structured outputs guarantee schema-valid JSON in the text response.
- Strict tool use guarantees valid arguments on every tool call.
- System-prompt steering shapes tone, scope, and format across turns.
- Reach for the lightest supported control that gets the shape you need.

---

## 12. Sampling Parameters: Legacy Knobs

Temperature, `top_p`, and `top_k` are classic sampling parameters.

### Three Legacy Sampling Parameters

| Parameter | Definition |
|---|---|
| **Temperature** | Scales the token probability distribution before sampling. A legacy knob; on current models a non-default value returns a 400 error. |
| **`top_p` (nucleus sampling)** | Restricts sampling to the smallest token set whose cumulative probability reaches p. Legacy; a non-default value 400s on current models. |
| **`top_k`** | Limits the candidate pool to the top k most likely tokens before sampling. Legacy only; a non-default value is rejected today. |

### What Temperature Actually Does

Temperature scales the logit distribution before sampling — it does not change which tokens are
possible, only their relative likelihoods.

- **Temperature 0** — most consistent, strongly favors the highest-probability token.
- **Temperature 0.3** — sharp distribution, high-confidence but slightly varied.
- **Temperature 1.0** — the model's default probability distribution unchanged.

Think of temperature as a dial from "most likely answer every time" to "full range of plausible
outputs."

### Low vs. High Temperature

- **Low temperature (0.0–0.3)** — output is concentrated on the highest-probability tokens.
  Responses are consistent, predictable, and well-suited to classification, extraction, code
  generation, and factual Q&A where a single best answer exists.
- **High temperature (0.7–1.0)** — output draws from a wider token pool. Responses vary across
  runs, making this appropriate for creative writing, brainstorming, persona-driven chat, and
  tasks where diversity of output is the goal.

### The Old Temperature 0 Myth

On legacy models, a common myth was that temperature 0 was the universally safe default. It was
not.

- **Classification and extraction** — low temperature was appropriate.
- **Creative writing and ideation** — low temperature produced flat, repetitive output.
- **Conversational tasks** — very low temperature felt mechanical.

The lesson carries over: match output behavior to the task, which today you do through prompting,
not a temperature dial.

### Nucleus Sampling in Detail

- **Cumulative probability** — `top_p` adds tokens in descending probability order until their
  cumulative total reaches p. All remaining tokens are excluded from sampling.
- **Dynamic pool size** — when the model is confident, the nucleus shrinks. When probability
  spreads broadly, the pool grows. Size adapts to the distribution automatically.
- **Typical values** — `top_p` of 0.9 to 0.95 are common defaults. Values near 0.5 restrict
  heavily; values near 1.0 are nearly unrestricted.

### `top_k` — A Simpler Hard Limit

`top_k` sets a fixed ceiling on candidate tokens regardless of probability distribution shape.

- `top_k` of 1 restricts to the single most-probable token: greedy decoding.
- `top_k` of 40 means the model samples from the 40 most probable tokens.
- Useful when you want a strict ceiling independent of the distribution.

`top_k` ignores probability gaps, so it can include low-probability tokens when the distribution
is flat.

### Legacy Interaction Effects

- **Temperature + `top_p`** — on older models, applying both compounded effects in hard-to-predict
  ways, often producing unexpected output.
- **Temperature + `top_k`** — temperature reshaped logits first, `top_k` then cut the pool.
  Combined tuning added complexity without clear benefit.
- **Old guidance** — the classic advice was to adjust one parameter at a time. Current models
  accept none of these knobs.

### How You Steer Current Models

On current models (the deck names "Opus 4.8 and 4.7"), sampling parameters are not tuning knobs —
a non-default value returns a 400.

- Passing a non-default temperature, `top_p`, or `top_k` is rejected.
- Shape tone and determinism through clear, specific prompting.
- Steer reasoning depth with the effort parameter, not a sampling dial.

Design your output behavior in the prompt, then verify it against real outputs.

### Key Takeaways: Sampling Parameters

- Current models reject a non-default temperature, `top_p`, or `top_k` with a 400.
- These three are legacy concepts to understand, not tune.
- Temperature scaled the distribution; `top_p` and `top_k` trimmed the pool.
- On current models you steer output through prompting and effort.
- Some older models still accept these.

Design behavior in the prompt, then check outputs.

---

## 13. Designing Multi-Turn Interactions

Multi-turn conversation design is about knowing when a dialogue outperforms a single prompt.

### Three Multi-Turn Concepts

| Term | Definition |
|---|---|
| **Multi-turn dialogue** | A conversation across two or more exchanges where each turn builds on the last. Used when one prompt cannot handle it. |
| **Clarification turn** | A model response that asks a targeted question before proceeding, resolving ambiguity without costly assumptions. |
| **Turn sequence** | A set ordering of exchanges that splits a complex task into steps. Each turn handles one part and passes context forward. |

### When Multi-Turn Beats One Prompt

Multi-turn design outperforms a single prompt in predictable situations.

- The task is ambiguous and assumptions could lead to wrong output.
- The final output depends on intermediate results the model produces.
- A human needs to approve one step before the next begins.
- Task length risks context window pressure in a single call.

When any of these conditions exist, dialogue is the better choice.

### Single Prompt vs. Multi-Turn

- **Single long prompt** — everything is given at once. Works for well-defined tasks; fails when
  inputs are ambiguous or steps depend on each other.
- **Multi-turn dialogue** — the task unfolds across exchanges. Each turn is smaller, allowing
  course correction and refinement.

### Clarification Turn Design

A good clarification turn asks exactly what it needs and nothing more.

- Identify the single ambiguity most likely to produce wrong output.
- Ask one targeted question rather than a list of open-ended questions.
- Briefly acknowledge what the model understood to confirm alignment.
- Set an expectation for what comes next once the question is answered.

Targeted clarifications keep conversations efficient and avoid user frustration.

### Clarification Turn Pitfalls

- **Question overload** — multiple questions in one turn feel like a form. Pick the one ambiguity
  that matters most.
- **Over-clarifying** — asking about details the model could reasonably infer wastes a turn and
  signals poor judgment.
- **No acknowledgment** — jumping to a question without restating what was understood makes users
  feel ignored.
- **Vague questions** — "Can you tell me more?" produces rambling answers instead of the specific
  detail needed.

### Breaking Tasks into Turn Sequences

Complex tasks decompose naturally into a sequence of focused turns: Turn 1 plan → Turn 2 produce
→ Turn 3 review, with an output handoff between each.

Assign one clear goal per turn and pass prior output forward. Turn sequences let you review,
adjust, and steer at each step.

### Iterative Refinement Workflows

Iterative refinement turns a draft into a final output through structured feedback.

- Turn one produces a draft based on the initial specification.
- Turn two delivers targeted feedback on specific aspects of the draft.
- Turn three applies revisions and confirms the changes were made.
- Later turns address remaining issues one category at a time.

Iterative refinement works because small targeted changes are simpler to check.

### State Signaling Between Turns

- **Summarize progress** — open a new turn by recapping what was completed in prior turns so the
  model stays oriented in the workflow.
- **Reference prior output** — quote or paraphrase the relevant part of the model's last response
  before asking for the next step.
- **Name the current step** — tell the model explicitly what phase of the workflow it is in, such
  as "We are now revising the introduction."

### Key Takeaways: Multi-Turn Design

Multi-turn dialogue outperforms a single prompt when tasks are ambiguous or sequential.

- Clarification turns ask one targeted question and acknowledge prior input.
- Turn sequences assign one goal per turn with explicit output handoffs.
- Iterative refinement uses draft-feedback-revision cycles for quality control.
- State signaling keeps the model oriented across long conversations.

Avoid the mega-prompt: use dialogue when it fits better.

---

## 14. Conversation State and Turn Management

Managing context across turns is the operational challenge of multi-turn conversation design.

### Three Context Management Terms

| Term | Definition |
|---|---|
| **Context accumulation** | The growth of conversation history as turns are added. Useful context enables coherence; excess context creates noise and window pressure. |
| **Context drift** | The gradual shift in topic, tone, or task framing across long conversations. Behavior can diverge from original intent without one obvious failure. |
| **Noise buildup** | The accumulation of repeated, contradictory, or obsolete content in history. Noise dilutes the signal from current instructions. |

### How Context Accumulates

Every turn adds tokens to the window and shapes what the model attends to.

- System prompt and early messages carry the most structural weight.
- Each assistant response joins the history the model reads next.
- Longer histories offer more background but also more noise to filter.
- Window limits create a hard ceiling regardless of conversation length.

Useful context supports coherence; excess context competes with current instructions.

### Context Drift Across Long Conversations

Drift is a gradual failure mode where behavior shifts from the original framing.

- **Topic drift** — conversation moves to adjacent subjects across turns.
- **Tone drift** — formality shifts as the model adapts to recent exchanges.
- **Task drift** — the model starts solving a related but different problem.
- Hard to catch because no single turn is obviously wrong.

Periodic re-anchoring keeps drift from compounding.

### Useful Context vs. Noise

- **Useful context** — prior exchanges the model needs to produce coherent output: task framing,
  established constraints, agreed decisions, and relevant prior outputs. Keeping this is essential
  for continuity.
- **Noise** — repeated instructions, contradictory messages, abandoned subtasks, and verbose
  back-and-forth no longer relevant to the current goal. Noise dilutes the model's attention to
  what matters.

### Noise Buildup Patterns

Four patterns account for most noise buildup in long conversations.

- **Repeated instructions** — the same directive restated across multiple turns.
- **Contradictory messages** — earlier constraints that conflict with newer ones.
- **Abandoned subtasks** — exchanges about a branch of work that was dropped.
- **Verbose back-and-forth** — hedging that adds tokens without adding signal.

Noise makes later turns less reliable and harder to diagnose when they go wrong.

### Correction and Repair Patterns

When a turn produces wrong output, the repair strategy matters as much as the fix.

- Be explicit: name what was wrong rather than restating the whole prompt.
- Correct one issue at a time to avoid compounding confusion.
- Reaffirm the correct context before asking for a redo.
- Verify the repair produced the right change before moving forward.

Targeted corrections produce cleaner follow-through than broad reruns.

### Reset vs. Continuation Decision

- **Continue when** — history is coherent, recent turns stay on course, and the model already has
  the context it needs to proceed.
- **Reset when** — noise has built up, drift has taken things off course, or task changes made
  prior context a liability.
- **Partial reset** — start a new thread but carry forward only essential context: key decisions,
  constraints, and relevant outputs. Leave the noise behind.

### Managing Context Window Budget

Context window budget is a finite resource that needs active management.

- Estimate turns needed before starting and front-load key constraints.
- Summarize long prior exchanges rather than carrying full verbatim history.
- Monitor token usage if your API client exposes it.
- Reset before hitting the limit rather than degrading into truncation.

Proactive budget management avoids the silent quality drop of truncation.

### Key Takeaways: Context and Turn Management

Managing state proactively prevents drift, noise, and window pressure.

- Context accumulates every turn; useful differs from noise.
- Drift is gradual: re-anchor with explicit reminders.
- Correction turns name the error and reaffirm context.
- Reset when history has become a liability.
- Budget your context window before the task starts.

---

## 15. Guaranteed JSON with Structured Outputs

When your application needs JSON it can parse every time, hoping the model formats it correctly
is not a plan. This module covers Structured Outputs — the feature that guarantees Claude returns
JSON matching a schema you define — its API shape, why the guarantee holds, and when to reach for
it.

### Three Structured Output Terms

| Term | Definition |
|---|---|
| **Structured Outputs** | Feature that forces Claude's text response to match a JSON schema you supply. Set through the output config format field on the request. |
| **JSON schema** | Declarative spec defining field names, types, and allowed values. You pass it to Claude to describe the exact shape you want back. |
| **Constrained decoding** | Token-level mechanism behind the guarantee. During generation the model can only pick tokens that keep the output valid against the schema. |

### The Output Config Format Shape

You turn on Structured Outputs by setting the format on the request's output config.

- Set output config format type to `json_schema`.
- Attach your schema object under the `schema` key.
- Claude returns a text response that matches that schema exactly.
- No separate mode flag and no post-hoc validation step.

The schema you pass becomes the contract for the response.

### Why the JSON Is Guaranteed

The guarantee comes from constrained decoding at inference, not from checking output afterward.

- At each step, the model emits only tokens the schema allows.
- Normally, invalid field names or wrong types cannot appear.
- Two exceptions remain: a refusal or a max-tokens cutoff.
- The routine parse-and-repair loop is gone; keep a stop-reason check.

It conforms in the normal case; check the stop reason otherwise.

### Old Approach vs. Structured Outputs

- **Validate after** — the older pattern asked the model for JSON, then parsed and validated it in
  your code. When a field was missing or mistyped, you retried or repaired. Structure was never
  actually guaranteed.
- **Structured Outputs** — you declare the schema up front and the model is constrained to match
  it during generation. The routine schema-drift retry loop disappears; you still keep a
  stop-reason check for refusals and truncation.

### Where Structured Outputs Fits

- **Data extraction** — pull named fields out of documents, emails, or transcripts into a fixed
  object your pipeline can consume directly.
- **Classification** — force the label into an enum of allowed categories so routing logic never
  sees an unexpected free-text value.
- **Feeding systems** — when output flows into a database or another service, a guaranteed shape
  removes a class of runtime failures.

### The Schema Is Your Contract

The schema does more than validate. It tells the model exactly what to produce.

- Required fields define what the response always contains.
- Types keep numbers as numbers and strings as strings.
- Enums restrict a field to a fixed set of valid options.
- Field descriptions guide how the model fills each value.

A clear schema drives both correctness and consistent extraction.

### One Feature, Not a Manual Pattern

Before this feature, JSON meant prompting for it and validating the result yourself.

- The old way had no API-level guarantee, just prompting and checks.
- Structured Outputs replaces that manual pattern with one mechanism.
- Support is model and platform dependent, so check the docs.

The mental model is simpler: define the schema, get matching JSON.

### Two Things to Keep in Mind

- **Design the schema well** — the guarantee is about shape, not judgment. A vague description
  still yields a wrongly chosen value inside a valid object.
- **Check model support** — Structured Outputs runs on the current Claude models. Confirm your
  chosen model supports it before relying on the guarantee.

### Key Takeaways: Structured Outputs

- Structured Outputs guarantees Claude returns JSON matching a schema you define.
- Turn it on through the output config format set to `json_schema`.
- Constrained decoding makes output conform during generation.
- It replaces the old ask-for-JSON-then-validate approach.
- Reach for it on extraction, classification, and system output.

---

## 16. Schema Design for Reliable Extraction

A JSON schema is only as good as its field definitions, descriptions, and edge case handling.

### Three Schema Design Terms

| Term | Definition |
|---|---|
| **Required vs. optional fields** | Required fields are always populated; optional fields allow null when data is absent. The choice shapes what downstream code can assume. |
| **Enum constraint** | A schema property restricting a field to a defined set of valid string values. Prevents free-text variation that breaks downstream routing. |
| **Nested schema** | A schema where one field's value is an object or array with its own type definitions. Used for hierarchical documents. |

### Required vs. Optional Field Design

The required-versus-optional decision shapes what downstream code can safely assume.

- Mark a field required only when the model can always extract a value.
- Optional fields signal that absence is a valid expected output.
- Too many required fields pressure the model toward hallucinated values.
- Too few required fields shift validation burden to downstream code.

Right-size required fields to what the data guarantees every time.

### Using Field Descriptions Effectively

Field descriptions are the primary tool for guiding Claude's extraction decisions.

- Describe what the field means in the context of the specific document type.
- Specify the expected format, such as ISO 8601 for dates.
- Name boundary cases: what counts as a match and what does not.
- Indicate extraction priority when multiple candidates are present.

A well-written description reduces ambiguity before it becomes a wrong field value.

### Null vs. Omit vs. Default Strategies

- **Null / omit** — return null or omit the field when data is genuinely absent. Makes absence
  explicit and distinguishable from a populated field. Requires downstream code to handle None or
  missing keys gracefully.
- **Default value** — return a sentinel value such as "unknown" or a blank string when data is
  absent. Makes the output shape uniform, but it can blur the line between a populated field and a
  missing one.

### Field Description Anti-Patterns

- **Too vague** — a description like "the name" gives the model no guidance when multiple names
  appear in the document.
- **No format spec** — omitting the expected format for dates or identifiers leads to
  inconsistent values that break downstream parsers.
- **No boundary cases** — failing to define what counts as a match forces judgment calls that
  should be explicit schema decisions.

### Nested Schema Patterns

Nested schemas handle documents where relationships between fields are hierarchical.

```
invoice
  vendor         object
    name         string
    id           string
  items          array of object
    sku          string
    quantity     number
```

Object nesting for entities, arrays of objects for repeating items — avoid unnecessary depth.

### How Schema Quality Affects Reliability

Schema quality is the most controllable factor in extraction reliability.

- Clear descriptions reduce ambiguous extractions from varied input phrasing.
- Appropriate required-versus-optional settings prevent hallucinated values.
- Enum constraints eliminate free-text variation in classification fields.
- Well-scoped nesting prevents the model from conflating separate entities.

Poor schema design shifts responsibility for handling errors to downstream code.

### Testing Schemas Against Edge Cases

- **Missing field test** — run the schema against input that lacks the field entirely. Confirm
  the output is null or omitted, not hallucinated.
- **Ambiguity test** — use inputs where two values could match a field. Verify the description
  guides selection to the intended value.
- **Type boundary test** — test with boundary values: long strings, zeros, empty arrays. Confirm
  the schema handles them without silent failures.

### Key Takeaways: Schema Design

Schema design is an active part of extraction quality, not just documentation.

- Required fields only where data is guaranteed; optional where absence is valid.
- Field descriptions define context, format, and boundary cases precisely.
- Null versus default strategies determine how absent data is communicated.
- Nested schemas match document structure at the depth the task requires.

Test schemas against edge cases before deploying.

---

## 17. Structured Outputs Limits

Structured Outputs guarantees schema-valid JSON when your schema fits a grammar-enforceable
subset. This module maps that subset, what the SDK transforms outside it, the hard complexity
limits, and where valid JSON is not guaranteed.

### Three Ideas to Anchor

| Term | Definition |
|---|---|
| **Enforceable subset** | The slice of JSON schema the grammar engine can compile. Types, enums, and references here are guaranteed by constrained decoding. |
| **SDK transform** | Keywords outside that subset, like `maxLength`, 400 at the raw API. The official SDK strips them and folds their intent into the field description. |
| **Guarantee gaps** | Two cases break the schema promise: refusals and truncation. Both are expected, not bugs to retry blindly. |

### The Supported Type Set

The core building blocks are the standard JSON types, and all of them are supported.

- Structural types: `object` and `array`.
- Scalar types: `string`, `integer`, `number`, `boolean`, and `null`.
- `enum` and `const` pin a field to fixed values.

These give you most real extraction and classification schemas.

### Composition and References

- **`anyOf`** — the model picks one branch from several. This is how you express a union type
  across different shapes.
- **`allOf`** — combines several constraints together, but `allOf` paired with a reference is not
  supported here.
- **`$ref` and `$defs`** — internal references let you define a shape once and reuse it. External
  URL references are rejected.

### Strings and Arrays

Two smaller features round out the supported subset, both narrow but useful.

- String formats like `date-time`, `date`, `email`, `uri`, and `uuid`.
- Array `minItems`, but only the values zero or one.
- Regex patterns with basic classes, groups, and simple quantifiers.

Anything richer than these falls outside the compiled grammar.

### Grammar Subset vs. Everything Else

- **Grammar-enforceable** — types, `enum`, `const`, `anyOf`, `allOf`, internal references, string
  formats, `minItems` of zero, and basic regex. Constrained decoding guarantees these.
- **Everything else** — numeric `minimum`, `maximum`, `multipleOf`, string `minLength` and
  `maxLength`, `maxItems`, and `additionalProperties` not false. These 400 at the raw API and are
  SDK-transformed.

### More That Falls Outside the Grammar

These also live outside the grammar and are SDK-transformed.

- Recursive schemas that reference themselves to describe a tree.
- External references pointing at a remote URL address.
- Complex regex: backreferences, lookahead, and word boundaries.

Design these out of the schema for a clean grammar rather than leaning on the transform.

### The Hard Complexity Limits

- **Twenty strict tools** — at most twenty tools carry strict mode in one request. Non-strict
  tools do not count.
- **Twenty-four optional** — optional parameters across every strict schema and JSON output share
  one budget of twenty-four.
- **Sixteen union params** — parameters typed as a union, through `anyOf` or a type array, cap at
  sixteen.

### When the Guarantee Bends

The schema promise holds in almost every case, with two known exceptions.

- A refusal stops with a safety message, and safety wins over the schema.
- A max-tokens cutoff truncates, so the JSON may be incomplete or invalid.
- Check the stop reason and retry with a higher max tokens; don't assume it parses.

### Grammar Compile and Cache

The schema compiles into a grammar, and that step has real timing behavior.

- The first request on a new schema pays a one-time compile latency.
- The compiled grammar is cached for twenty-four hours from last use.
- Changing schema structure or the tool set invalidates that cache.
- Renaming a field or description alone does not force a recompile.

### Key Takeaways: Designing Inside the Lines

- Build from enforceable types, enums, references, and formats.
- Keywords outside it 400 and are SDK-transformed.
- Respect the twenty-tool and twenty-four-parameter caps.
- Refusals and truncation break the guarantee.

Keep the schema in the grammar, and JSON stays valid.

---

## 18. Tool Definitions for Extraction

Defining a tool gives the model a schema it can populate when you extract structured data. This
module covers how to write extraction tools, what the strict flag guarantees about the result,
and how strict tool use relates to the Structured Outputs feature.

### Three Ideas to Anchor

| Term | Definition |
|---|---|
| **Extraction tool** | A tool defined to capture data, not run a side effect. Its input schema describes the fields the model fills from source text. |
| **Strict tool use** | Setting `strict` to true on a tool definition. It guarantees the tool call arguments conform to the declared input schema. |
| **Input schema** | The typed field definitions on a tool. It names each field, sets its type, and marks which fields are required. |

### Tools as an Extraction Target

A tool call is a clean place to receive structured data, even when nothing is executed.

- Define a tool whose input schema is the shape you want.
- Ask the model to call it over the source text.
- It returns a structured argument object, not free text.
- Your code reads fields directly; no parsing to fail.

The tool call itself becomes the output.

### Writing the Input Schema

A precise input schema turns a vague ask into a reliable extraction target.

- Name each field and describe a valid value.
- Use enum constraints when only fixed values are valid.
- Mark a field required only when absence is a failure.
- Keep it flat, since nesting adds compliance risk.

```
title       string              the invoice number
status      enum [paid, unpaid]
amount      number              required
due_date    string
```

Descriptions guide extraction as much as types do.

### The Strict Flag

- **What you set** — add `strict: true` beside name, description, and input schema on the tool
  definition.
- **What you get** — the tool call arguments are guaranteed to match the input schema, so no
  field is missing or mistyped.
- **What it removes** — no validation-and-retry loop is needed to catch a malformed tool call
  before you use it.

### Three Strict Guarantees

Strict tool use gives three concrete guarantees at generation time.

- **Valid tool name** — never invented or misspelled.
- **Type-matched arguments** — every argument matches its declared type in the schema.
- **Required fields present** — no required field is ever left missing.

It mirrors what Structured Outputs gave you for text.

### Two Ways to Constrain

- **Structured Outputs** — constrains the model's text response to a JSON schema. Reach for it
  when the whole reply should be structured data, like a direct extraction or classification
  result.
- **Strict tool use** — constrains the arguments of a tool call to the tool's input schema. Reach
  for it when the model is calling functions and each call has to be well formed.

### Shared Foundations

Strict tool use and Structured Outputs are built on the same underlying machinery.

- Both accept the same supported subset of JSON schema features.
- Both compile that schema and cache it for the same window.
- A first request on a new schema pays a small compile cost.
- Changing the schema invalidates the cache and recompiles.

So the schema rules you learned earlier carry over to tools.

### When to Combine Them

- **Agent calls a tool** — the agent invokes a validated function. Strict tool use keeps every
  call's arguments well formed.
- **Agent returns analysis** — the same agent hands back a structured summary. Structured Outputs
  constrains that final text reply.
- **Use both together** — when one workflow does both, apply each feature to the part it governs
  rather than choosing one.

### Key Takeaways: Tool Definitions for Extraction

- An extraction tool returns structured data as a tool call.
- The input schema names fields, types, and required ones.
- `strict: true` guarantees arguments conform to it.
- You get a valid name, correct types, and no missing required fields.
- Text uses Structured Outputs; arguments use strict tool use.

---

## 19. Validation Loops and Hallucination Prevention

Reliable extraction pipelines validate output after generation and re-prompt on failure.

### Three Reliability Concepts

| Term | Definition |
|---|---|
| **Validation loop** | A pattern where extracted output is validated after generation. On failure, the error is fed back into a follow-up prompt for correction. |
| **Hallucination prevention** | Techniques like grounding, citation instructions, and schema constraints that reduce plausible but incorrect output. |
| **Grounding** | Injecting source material or retrieved context so the model extracts from evidence rather than from memory, reducing confabulation. |

### The Validation Loop Pattern

A validation loop adds a structured retry path after every extraction call: extract → validate →
on failure, re-prompt → re-validate.

One-shot extraction without a retry path is insufficient for reliability-critical pipelines.

### What to Validate After Extraction

Validation covers distinct layers, each catching different failure types.

- **Schema compliance** — all required fields present and correct types.
- **Business rules** — values fall within expected ranges or known sets.
- **Semantic correctness** — extracted values are consistent with the source.
- **Cross-field consistency** — related fields do not contradict each other.

Skipping business rule or semantic validation misses the failures that matter most.

### Schema vs. Business Rule Validation

- **Schema validation** — checks that the output matches the declared structure: required fields
  present, types correct, enums respected. Automated and deterministic. Catches structural
  failures but does not verify that values are meaningful.
- **Business rule validation** — checks that values satisfy domain logic: dates in valid ranges,
  referenced identifiers exist in known sets, numbers pass reasonableness thresholds. Catches
  semantic failures that pass schema checks.

### Re-Prompt Design on Failure

- **Include the error** — paste the specific error into the re-prompt. A vague "try again"
  instruction gives the model nothing to fix.
- **Reference the source** — re-include the relevant source text so the model re-extracts from
  evidence rather than correcting from memory.
- **Constrain the correction** — instruct the model to fix only the failing fields instead of
  regenerating the entire output, which can add new errors.

### Hallucination Prevention Techniques

Hallucination prevention requires multiple techniques because failure modes differ.

- **Grounding** — inject retrieved source text so the model extracts, not generates.
- **Citation instructions** — ask the model to cite the source for each field.
- **Schema constraints** — enum fields and required/optional rules cut guessing.
- **Temperature** — lower values reduce variance but do not fix grounding gaps.

No single technique covers all hallucination failure modes.

### Grounding and Context Injection

Grounding is the most effective single technique for reducing confabulation in extraction.

- Inject the specific source document or retrieved passage into the prompt.
- Instruct the model to extract only from provided text, not general knowledge.
- Ask for explicit source citations to make grounding violations auditable.
- For multi-document extraction, label each source so citations are traceable.

Grounding converts extraction from memory-based generation to evidence-based retrieval.

### When Validation Loops Are Worth the Cost

- **High-stakes output** — when extracted data feeds legal, financial, or compliance decisions,
  loop latency and cost are justified by error reduction.
- **Scale and automation** — when extraction runs on thousands of documents without human
  review, a retry path catches systematic failures early.
- **Low-stakes or interactive** — for exploratory or interactive use with human review, one-shot
  extraction with light validation may be enough.

### Key Takeaways: Validation Loops

Reliable pipelines combine validation loops with hallucination prevention.

- The loop sequence is extract, validate, re-prompt on failure.
- Validate schema, business rules, and semantic correctness in layers.
- Re-prompts include the specific error and source text.
- Grounding, citations, and schema constraints address distinct failures.

Loops earn their cost when errors have downstream consequences.

---

## 20. Validation Gates and Retry Strategy

Production prompts cannot rely on getting it right the first try, so they validate output and
retry.

### Gates, Loops, and Budgets

| Term | Definition |
|---|---|
| **Validation gate** | A deterministic check that decides accept, retry, or escalate. |
| **Retry loop** | Sends the prior result and failure back to the model, capped by a budget. |
| **Retry budget** | A hard cap on retry attempts before escalating or erroring. |

### Three Layers of Validation

Not all validation is the same. There are three distinct layers to check.

- **Schema validation** — confirms the output matches the expected structure.
- **Format validation** — checks that values meet field-level constraints.
- **Semantic validation** — verifies the content actually makes sense.

Running all three layers in sequence gives comprehensive coverage.

### Schema vs. Semantic Validation

- **Schema validation** — confirms correct fields, types, and structure. Fast and deterministic.
  Catches structural errors but misses plausible-sounding wrong answers.
- **Semantic validation** — confirms the content is correct and coherent in context. Slower,
  often needs a second model call. Catches meaning errors schema checks miss.

### Feeding Failures Back Into Retry

A retry is only useful if the model understands what went wrong the first time.

- Include the failure reason in the retry prompt, not just "try again."
- Paste the original output alongside the critique so the model can compare.
- Be explicit: "your previous response was missing the required summary field."

Specific feedback improves the next attempt; generic prompts repeat failure.

### Retry Budget Design Principles

- **Set a hard cap** — never allow unlimited retries. Two to three attempts cover most
  recoverable failures without runaway cost.
- **Log every attempt** — record what failed and why on each attempt. Logs drive improvement
  across future runs.
- **Escalate on exhaust** — when the budget runs out, return an error or route to a human.
  Silence is not an option.
- **Track consumption** — monitor per-session and per-workflow retry rates to catch systemic
  prompt failures early.

### Backoff and Temperature Variance

When a model gets stuck in a pattern, repeated identical retries rarely help.

- **Temperature bump** — add a small increase on each retry to inject variance.
- **Exponential backoff** — avoid hammering rate limits under failure.
- **Prompt variation** — change the wording slightly to break the same failure path.

A stuck model needs a different angle, not the same question repeated louder.

### Three More Concepts

| Term | Definition |
|---|---|
| **Reviewer pattern** | A second model call that critiques the output against a checklist. |
| **Escalation path** | What to do when retries run out: error, human review, or safe default. |
| **Semantic drift** | When retries pass format checks but drift from the intended meaning. |

### When to Escalate Instead of Retry

Retrying works for recoverable errors. Persistent failures signal a different problem.

- Escalate when the same semantic error appears across multiple attempts.
- Escalate when format passes but content meaning is consistently wrong.
- Escalate when the failure points to a prompt design flaw, not a model hiccup.

Semantic failures that repeat are a prompt problem: more retries will not fix a broken
instruction.

### The Hidden Validation Mistake

The most common validation failure is checking only the shape of the output.

- A JSON object can be structurally valid but semantically empty or wrong.
- Schema-only validation gives false confidence in bad content.
- Always add a semantic check: a required field, a plausibility rule, or a keyword.

Passing schema validation is the floor, not the ceiling.

### Key Takeaways: Validation Gates in Practice

Validation-retry loops are the backbone of reliable LLM pipeline design.

- Three layers: schema, format, and semantic validation.
- Include failure reasons in retry prompts for meaningful improvement.
- Cap retries with a budget and escalate when it is exhausted.
- Use temperature variance and backoff to break stuck patterns.

Validate content meaning, not just output structure.

---

## 21. Multi-Pass Review

A single model call is often not enough for high-stakes output, so a reviewer pass critiques it
first: draft → reviewer → revised.

### Three Review Concepts

| Term | Definition |
|---|---|
| **Reviewer pattern** | A second model call that critiques the first call's output against an explicit checklist before the result is returned. |
| **Self-review** | A reviewer pass run by the same model that produced the output. Cheaper, but more prone to confirmation bias. |
| **Cross-review** | A reviewer pass run by a different, often stronger model for a genuinely independent critique. |

### How the Reviewer Pattern Works

The reviewer pattern adds a second model call between generation and delivery: generate → review
→ revise.

- Call one generates the output using your primary prompt.
- Call two critiques that output against an explicit checklist.
- The critique feeds a revision call when issues are found.

This catches errors before the user sees them, not after a complaint arrives.

### Self-Review vs. Cross-Review

- **Self-review** — the same model that generated the output also reviews it. Lower cost and
  latency. More prone to confirmation bias, since the model tends to agree with its own prior
  output.
- **Cross-review** — a different, often stronger model reviews the output independently. Higher
  cost but a genuinely independent perspective. Better for high-stakes or regulated content.

### Designing Effective Reviewer Prompts

The reviewer prompt is where most multi-pass implementations fail or succeed.

- Use an explicit checklist, not open-ended "find issues" instructions.
- Specify what a passing result looks like, not just what failure looks like.
- Ask for a structured critique: which check failed, and why.

Vague reviewer prompts produce vague critiques that do not drive useful revisions.

### Reviewer Checklist Categories

- **Completeness** — does the output include every required section or field? Missing content is
  the most common failure.
- **Accuracy** — are the facts, figures, and claims consistent with the input context and
  constraints?
- **Tone and format** — does the output match the required style, voice, and structural format?

### When Multi-Pass Is Worth the Cost

Multi-pass review adds latency and cost, so it is only justified in certain contexts.

- High-stakes output where errors have real consequences: legal, medical, financial.
- Regulated domains where accuracy is a compliance requirement.
- Long-form content where a missed section is hard to catch on a casual read.

If the output is low-stakes and easily corrected, single-pass generation is the right call.

### When Multi-Pass Is Not Worth It

Most generation tasks do not need a full review cycle. Know when to skip it.

- Low-stakes, easily reversible output like draft summaries or brainstorms.
- High-volume, low-cost generation where latency is the dominant constraint.
- Cases where a human already reviews the output before it matters.

Adding a review pass to everything inflates cost and latency without proportional gains.

### Review as Quality Control vs. Prompt Fix

- **Review as quality control** — multi-pass review catches errors in individually generated
  outputs. The right tool when output is inherently variable and the cost of an error is high.
- **Review as prompt substitute** — using a reviewer to compensate for a weak original prompt is
  an anti-pattern. It adds cost without addressing the root cause. Fix the prompt first.

### Closing the Loop with a Revision Pass

A critique is only useful if it feeds into a concrete revision.

- Pass the original output and the critique together into a revision prompt.
- Ask the model to address each critique point in a revised version.
- Validate the revision using the same checklist as the review step.

The revision pass closes the loop. Without it, you have a critique but not an improvement.

### Key Takeaways: Multi-Pass Review Foundations

Multi-pass review is a powerful quality-control layer for high-stakes LLM output.

- The reviewer pattern adds a critique call between generation and delivery.
- Explicit checklists outperform open-ended reviewer prompts.
- Self-review is cheaper; cross-review is more independent.
- Reserve multi-pass for high-stakes, regulated, or long-form content.

Review is quality control, not a prompt fix.

---

## 22. Building a Prompt Evaluation Workflow

Manual spot-checking is not enough; production systems need systematic, repeatable evaluation.

### Three Eval Workflow Concepts

| Term | Definition |
|---|---|
| **Prompt evaluation** | Testing prompt variants against a labeled dataset to measure output quality and regression risk before deploy. |
| **Regression testing** | Running the full eval suite after a change to detect whether cases that used to pass have started failing. |
| **Eval metric** | A quantitative measure used to compare prompt versions: pass rate, regression rate, latency, and cost. |

### Why Spot-Checking Falls Short

Manual spot-checking of a few outputs cannot reliably validate prompt changes.

- Spot-checks depend on reviewer intuition rather than consistent criteria.
- Edge cases and adversarial inputs are rarely included in informal testing.
- A passing impression on five examples can hide failures on dozens more.
- No record is kept, so regressions introduced later go unnoticed.

Systematic evaluation replaces intuition with measurable, repeatable results.

### Ad-Hoc Testing vs. Eval Workflow

- **Ad-hoc spot-checking** — a few outputs reviewed manually before deploying. Fast but
  unreliable. No coverage of edge cases, no regression detection, and no historical record to
  compare against later.
- **Systematic eval workflow** — a dataset, a run script, a grading step, and a comparison
  against a baseline. Slower upfront but produces consistent, documented evidence that a change is
  safe to deploy.

### The Core Eval Loop

A prompt eval workflow follows four repeatable steps in every iteration.

1. **Dataset** — curated inputs with known expected outputs.
2. **Run** — send each input through the prompt and collect responses.
3. **Grade** — score each response against the expected output or criteria.
4. **Compare** — measure the new version against the current baseline.

### Key Eval Metrics

- **Pass rate** — percentage of eval cases where output meets grading criteria. The primary
  indicator of overall prompt quality.
- **Regression rate** — share of previously passing cases that now fail after a change. Any
  increase signals a regression.
- **Latency** — average response time per eval case. Relevant when a prompt change could slow
  model output.
- **Cost per eval** — total token spend divided by case count. Tracks whether the prompt stays
  within operational budget.

### Eval-Driven Prompt Iteration

Eval-driven iteration replaces guesswork with a documented change cycle.

1. **Change** — modify the prompt based on observed failures or new requirements.
2. **Test** — run the full eval suite against the updated prompt.
3. **Compare** — measure pass rate and regression rate against the prior version.
4. **Deploy** — promote only when the new version passes without new regressions.

### Regression Testing in Practice

Regression testing protects all passing cases, not only the ones targeted.

- Every change runs the full suite, not just related tests.
- Any newly failing test is a regression, regardless of intent.
- Regression rate tracks the fraction of previously passing cases now failing.
- A fix that introduces regressions elsewhere is not a net improvement.

Catching regressions before deploy is cheaper than fixing them in production.

### When to Run Evals

- **Before every deploy** — any prompt change that touches production requires a full eval run,
  no matter how small.
- **After dataset updates** — when new test cases are added, re-run against both versions to
  recalibrate what the baseline metrics reflect.
- **On schedule** — a regular eval cadence catches model-level drift even when no prompt change
  was made at all.

### Key Takeaways: Eval Workflow

A systematic eval workflow produces evidence; spot-checking produces impressions.

- The core loop is dataset, run, grade, compare.
- Key metrics are pass rate, regression rate, latency, and cost per eval.
- Eval-driven iteration ties every deploy decision to measured results.
- Regression testing covers all passing cases, not just the target of a change.

Run evals before deploy, while problems are cheap to fix.

---

## 23. Test Datasets and Model-Based Grading

Test dataset quality shapes what your eval workflow can actually measure and find.

### Three Grading and Dataset Terms

| Term | Definition |
|---|---|
| **Test dataset** | A curated set of inputs with reference outputs used to evaluate prompt performance, across typical, edge, and adversarial cases. |
| **Code-based grading** | Programmatic assertions, regex, or schema validators that decide whether an output meets correctness criteria. Fast and deterministic. |
| **Model-based grading** | A separate model scores the output against criteria or reference answers. Handles subjective outputs that code cannot check. |

### Three Ways to Generate Test Datasets

Test datasets come from three distinct sources with different tradeoffs.

- **Handcrafted** — manually written cases targeting known failure modes.
- **Synthetic** — generated by a model from a seed set of topics or patterns.
- **Production-sampled** — real inputs drawn from logs, reviewed and labeled.
- **Blend sources** — mature eval suites combine all three.

Start with handcrafted cases and expand with synthetic generation.

### What Makes a Good Test Dataset

A test dataset is only as useful as the coverage it provides.

- **Typical cases** — inputs the prompt handles correctly under normal conditions.
- **Edge cases** — inputs at the boundary of the prompt's intended scope.
- **Adversarial inputs** — cases designed to trigger failures or expose weak spots.
- **Volume** — enough cases that the pass rate is statistically meaningful.

Datasets with only typical inputs miss the failures that matter most.

### Code-Based vs. Model-Based Grading

- **Code-based grading** — uses programmatic checks: exact match, regex, schema validation, or
  unit tests. Fast, rule-based, and cheap per case. Best when correctness follows a clear rule.
  Does not scale to open-ended output.
- **Model-based grading** — a separate model evaluates the output against criteria or a reference
  answer. Scales to subjective and open-ended tasks code cannot check. Introduces grader bias and
  needs calibration.

### Code-Based Grading Techniques

Code-based grading is preferred when correctness is programmatically expressible.

- **Exact match** — the output equals an expected string or result.
- **Regex** — the output matches a pattern that captures acceptable variants.
- **Schema validation** — the output parses correctly against a JSON schema.
- **Unit tests** — the output passes assertions written as test functions.

Code-based grading is fast, deterministic, and costs little per case.

### Model-Based Grading Design

Model-based grading gives a grader model the output and a rubric.

- The grader receives the input, the output, and scoring criteria.
- The rubric defines what a passing response looks like across dimensions.
- The grader returns a score, a pass or fail label, or a rationale.
- The grading prompt needs version control and testing like any prompt.

Model-based grading scales to output types that code checks cannot handle.

### Model-Based Grading Risks

- **Grader bias** — the grader may favor outputs stylistically similar to its own defaults,
  inflating scores for certain formats or tones.
- **Untested grading prompts** — the grading prompt is a prompt too. If it is not evaluated and
  versioned, grader quality can drift silently.
- **Scores as ground truth** — model-graded scores are estimates. Without calibration against
  humans, they overstate measurement precision.

### Calibrating Model-Based Grading

Calibration connects model-graded scores to human judgment as a reference standard.

- Sample a representative subset of eval cases and have humans label each one.
- Run the same cases through the model grader and compare results.
- Calculate agreement rate: the fraction where grader and human agree.
- Adjust the grading rubric where systematic disagreements appear.

Calibrated graders are reliable at scale; uncalibrated ones introduce silent errors.

### Key Takeaways: Datasets and Grading

Dataset quality and choice of grader decide what your evals can measure.

- Test datasets combine handcrafted, synthetic, and production-sampled inputs.
- Good datasets cover typical, edge, and adversarial inputs.
- Code-based grading suits structured output: fast and deterministic.
- Model-based grading scales to subjective tasks but needs calibration.

Match the grader to the output type.
