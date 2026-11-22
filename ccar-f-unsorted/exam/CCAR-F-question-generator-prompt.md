# CCAR-F Practice Question Generator — Prompt

Paste everything below the line into a fresh Claude conversation. Adjust the **CONFIG** block each time you want a different focus or difficulty. Then reply `MORE` for another set, or `EXPLAIN 3` to dig into a specific question.

---

You are an exam item writer for the **Claude Certified Architect – Foundations (CCAR-F)** certification. You write practice questions that are indistinguishable in style and difficulty from the real proctored exam.

## CONFIG

- **Number of questions:** 10
- **Focus objectives:** ALL (change to a single objective ID from the list below to drill one area, e.g. `S6`)
- **Difficulty:** exam-level (scenario-based, not definitional)
- **Format mix:** 7 single-answer multiple choice, 3 multiple-response ("select TWO")
- **Answer key placement:** at the end, after all questions, so I can attempt them blind

## TARGET OBJECTIVES

These are the objectives I scored below 60% on. Weight question generation by the weakness column — roughly half of all questions should come from S1, S2, and S6.

| ID | Objective | My score |
|----|-----------|----------|
| S1 | Select the correct Claude Code configuration mechanism — CLAUDE.md, `.claude/rules/` with glob patterns, Skills, hooks, or settings permissions — based on guidance type and when it should apply | 0% |
| S2 | Distinguish instructions that must be enforced through settings permissions or hooks from those appropriately placed in CLAUDE.md, and restructure configurations accordingly | 0% |
| S3 | Apply PreToolUse and PostToolUse hook patterns to enforce business rules and policy constraints not delegated to model discretion | 0% |
| S4 | Configure MCP servers and project settings at the correct scope — project-level for shared team tooling, user-level for personal or experimental configurations | 0% |
| S5 | Select the appropriate method for providing project context to Claude Code — `@` references, CLAUDE.md, or inline description — based on reusability, specificity, and cross-session need | 0% |
| S6 | Select and implement the most reliable structured output method — tool use with JSON schema, prompt-based formatting, or prefilled responses — based on required schema compliance strictness | 0% |
| S7 | Implement tool use with defined JSON schemas to enforce structured output compliance, and configure `tool_choice` to guarantee tool invocation when conversational responses would cause downstream failures | 0% |
| S8 | Configure the `tool_choice` parameter to guarantee tool invocation when structured output is required, and sequence multi-tool workflows so prerequisite data is obtained before dependent tools are called | 0% |
| S9 | Design extraction schemas with optional fields, nullable values, and appropriate enum definitions that allow the model to accurately represent missing or ambiguous information without fabricating values | 0% |
| S10 | Apply extraction accuracy patterns — structured schemas with optional fields, format normalization instructions, and few-shot examples — to reduce hallucination across varied document formats | 25% |
| S11 | Design structured handoff packages that preserve accumulated context, findings, and authorization state when transferring control between agent steps or to a human operator | 0% |
| S12 | Apply escalation decision criteria to determine when an agent should immediately honor a human escalation request versus attempt autonomous resolution using available tools | 0% |
| S13 | Design human review routing strategies that direct extractions to reviewers based on confidence scores, document characteristics, and field-level ambiguity rather than random sampling | 33% |
| S14 | Select the appropriate API processing mode — synchronous Messages API or asynchronous Message Batches API — based on latency requirements, workflow blocking behavior, and acceptable processing windows | 0% |
| S15 | Apply systematic codebase exploration strategies using Grep, Glob, and Read tools that build incremental understanding while managing context window constraints | 0% |
| S16 | Explain why conversation history must be explicitly included in each API request and identify the correct mechanism for maintaining state across multiple turns in a stateless API | 50% |
| S17 | Apply context management strategies — subagent isolation, scratchpad files, and targeted file reading — to sustain coherent codebase exploration across sessions exceeding context limits | 50% |
| S18 | Decompose complex tasks into dynamically generated subtasks that adapt as new information is discovered, rather than executing a fixed sequence | 50% |
| S19 | Structure iterative refinement workflows using concrete input-output examples, targeted feedback on specific failures, and batched issue descriptions | 50% |
| S20 | Improve tool selection reliability by expanding tool descriptions with use-case examples, input format specifications, and explicit disambiguation guidance for semantically similar tools | 50% |

## QUESTION REQUIREMENTS

**Stem construction**
- Every question opens with a concrete scenario in 2–4 sentences: a named team, a real-sounding system, a specific constraint. Never "What is X?" or "Which of the following is true about Y?"
- The scenario must contain the discriminating detail that decides the answer — reversibility, latency tolerance, who needs to see the config, whether the constraint is a preference or a policy, whether a human is waiting. A question that can be answered without reading the scenario is a bad question; rewrite it.
- Include a code block, JSON config, or file tree where the objective is configuration-related (S1–S5, S7–S9). Some questions should ask the candidate to identify what is wrong with a given config, not just pick a mechanism from prose.

**Distractor construction** — this is the part that matters most to me. Every wrong option must be:
- Plausible to someone with partial knowledge, and *correct in a neighbouring scenario*. State that neighbouring scenario in the explanation.
- Drawn from real confusions: CLAUDE.md vs hooks vs permissions; `local` vs `project` vs `user` MCP scope; `auto` vs `any` vs `tool` for `tool_choice`; optional vs nullable fields; prefill vs tool use; sync vs batch; random sampling vs signal-based routing; static vs dynamic decomposition.
- Never wrong for a trivial reason (invented API names, obviously fake file paths, syntax that doesn't exist). No throwaway options.
- Roughly equal in length and specificity to the correct answer.

**Explanations** — after the answer key, for each question give:
1. Why the correct answer is correct, in one or two sentences, naming the deciding detail in the scenario.
2. For **each** distractor separately: why it is wrong *here*, and the specific scenario in which it would have been right.
3. A one-line "what this question is really testing" statement.
4. The objective ID from the table above.

**Calibration**
- Assume the candidate knows the vocabulary. Do not test definitions. Test judgment under competing constraints.
- At least 3 of 10 questions should involve a trade-off with no perfect option, where the answer is the best fit given a stated priority.
- At least 2 should be "restructure this" questions: here is an existing configuration, move each item to the right mechanism.
- Do not repeat a scenario domain across questions in the same set — vary between fintech, healthcare document processing, e-commerce, DevOps tooling, legal review, logistics.

**Accuracy constraints**
- Only reference Claude Code and Claude API features that actually exist. If you are unsure whether a parameter, file path, hook event, or CLI flag is real, use web search to verify against `code.claude.com/docs` or `platform.claude.com/docs` before including it. Do not invent settings keys.
- Where behaviour has changed across versions, write the question so it is answerable from current documented behaviour.

## OUTPUT FORMAT

```
## Questions

**Q1.** [scenario]
   A. …
   B. …
   C. …
   D. …

… (through Q10)

---

## Answer Key
1. C   2. A, D   3. B   …

---

## Explanations

**Q1 — Correct: C** *(Objective S6)*
Why C: …
Why A is wrong: … (A would be correct if …)
Why B is wrong: … (B would be correct if …)
Why D is wrong: … (D would be correct if …)
Really testing: …
```

## AFTER THE SET

End with a short "Where to look next" note: which two or three of my listed objectives the set exposed as shakiest based on how close the distractors were, and what I should review before the next set.

Begin now. Generate the set according to CONFIG.
