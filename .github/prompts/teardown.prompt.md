---
description: "Architectural teardown of an LLM app template: data flow, real design decisions, demo-grade vs production gaps, cost/latency shape, and questions back to you."
argument-hint: "Path to the template folder, e.g. rag_tutorials/corrective_rag"
agent: "llm-tutor"
tools: [read, search, web]
---

Tear down the template at: **${input:target:path to the template folder}**

Read every source file in it first. Do not describe anything from memory — cite file paths for each claim.

Produce exactly these sections, in order:

## 1. Purpose
One sentence. What problem does this solve that a plain LLM call does not?

## 2. Data flow
A mermaid diagram from user input to final output. Show every LLM call, every retrieval, every tool invocation, and every loop or branch as a distinct node. Mark which nodes are non-deterministic.

## 3. File map
Table: file → what it owns → lines worth reading closely. Skip boilerplate.

## 4. The real decisions
The 3–5 genuine engineering choices the author made — chunk size, retrieval strategy, model split, loop termination, state location, output schema. For each: **what was chosen, what it bought, what it cost.** Ignore cosmetic choices.

## 5. Roads not taken
For the two most consequential decisions above, name the credible alternative and the condition under which it would win instead.

## 6. Demo-grade vs production-grade
Table: concern → how this template handles it → what it needs in production.

Cover at minimum: secrets, retries, timeouts, rate limits, structured-output validation, cost ceiling, loop/recursion bounds, observability, evals, prompt-injection surface, multi-tenancy. Be specific — "no retry on the embedding call in `x.py`", not "needs error handling".

## 7. Cost and latency shape
Per request: how many model calls, roughly how many tokens, what dominates p95, and what the first optimisation should be. Estimate ranges and label them as estimates.

## 8. Questions for me
Exactly three questions I must answer before moving on. They must be diagnostic — questions I can only answer if I actually understood the design. No trivia, no definitions.

**Stop after the questions. Do not answer them.**

## 9. Next drill
One named drill (delete-and-rebuild, port-it, failure injection, or predict-then-run), the specific file or function to run it on, and one sentence on what it will teach me.
