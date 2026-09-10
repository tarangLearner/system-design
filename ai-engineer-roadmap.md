# AI Engineer Roadmap (for a 12-year Software Engineer)

Built around [Shubhamsaboo/awesome-llm-apps](https://github.com/Shubhamsaboo/awesome-llm-apps) — 100+ Apache-2.0 reference apps. The repo is a **lab**, not a curriculum. This file is the curriculum; the repo supplies the exercises.

---

## 1. Read this first: the job you're targeting

There are three different jobs people call "AI":

| Role | Core skill | Math needed | Your fit |
|---|---|---|---|
| ML Researcher | novel architectures, papers | heavy | poor ROI at 12y |
| ML Engineer | training, feature stores, model ops | moderate | lateral move |
| **AI Engineer** | **building products on top of LLM APIs + retrieval + agents + evals** | **light** | **direct fit** |

**AI Engineering is a systems-engineering discipline wearing an ML hat.** The hard parts are the ones you already do: latency budgets, caching, idempotency, retries, cost control, observability, failure modes, state management, API design. The model is a non-deterministic, expensive, high-latency network dependency with an unreliable contract. You have spent 12 years managing exactly that class of dependency.

Your existing notes already cover half the substrate:
- [caching.md](caching.md) → semantic caching, prompt caching, KV cache
- [latency.md](latency.md) → TTFT vs total latency, streaming, speculative execution
- [databases.md](databases.md) → vector stores are just another index with different recall/latency tradeoffs
- [distributed-systems.md](distributed-systems.md) → agent orchestration is a workflow/saga problem
- [load-balancer.md](load-balancer.md) → model routing, fallback chains, rate-limit shedding

**Case studies + interview Q&A:** [ai-company-engineering-blogs.md](ai-company-engineering-blogs.md) — LinkedIn and Uber's production AI systems broken down into answerable form (retrieval funnels, distillation, GPU efficiency, LLM-as-judge, agents).

**Agents, precisely:** [ai-agent-vs-agentic-ai.md](ai-agent-vs-agentic-ai.md) — the autonomy ladder (prompt → workflow → agent → multi-agent), workflow vs agent, orchestration patterns, failure modes, the cost/reliability math, and when *not* to go agentic.

**Do not restart your career at zero. Position as: "senior engineer who ships reliable LLM systems."** That is scarcer and better paid than "junior AI person."

---

## 2. The actual gap list

What you genuinely need to learn (nothing here requires linear algebra):

1. **LLM mechanics as a practitioner** — tokens, context window, temperature, top_p, structured output / JSON mode, function (tool) calling, streaming, reasoning models vs chat models.
2. **Context engineering** — the successor to "prompt engineering." What goes into the window, in what order, compressed how, and what gets evicted.
3. **Embeddings & vector search** — cosine similarity, chunking strategy, hybrid (BM25 + dense), re-ranking, recall@k.
4. **RAG** — naive → hybrid → re-ranked → corrective/self-grading → agentic → graph.
5. **Agents** — tool loop, planning, memory, multi-agent handoff, human-in-the-loop, termination conditions and cost ceilings.
6. **MCP (Model Context Protocol)** — the emerging standard for wiring tools/data to models. High leverage right now.
7. **Evals** — the single biggest differentiator. Offline datasets, LLM-as-judge, regression suites in CI, tracing.
8. **Serving & cost** — token accounting, caching layers, batching, small-model routing, local models (Ollama/vLLM), quantization.
9. **Safety** — prompt injection, tool-abuse, data exfiltration, PII. OWASP LLM Top 10.
10. **Fine-tuning (awareness only)** — LoRA/QLoRA, when it beats RAG (style/format/latency), when it doesn't (facts).

Everything else is optional.

---

## 3. The path through the repo

Work in phases. Do not read — **run, break, and modify**. Rule for every item: get it running, then change one thing it wasn't designed to do.

### Phase 0 — Environment
```bash
git clone https://github.com/Shubhamsaboo/awesome-llm-apps.git
cd awesome-llm-apps
```
Set up `uv` or `venv`, get an OpenAI/Gemini/Anthropic key, and install [Ollama](https://ollama.com) for local models so experimentation is free. Put keys in `.env`, never in code — several templates in the repo read from env vars, follow that pattern.

### Phase 1 — Foundations: make the model do work
Folder: `starter_ai_agents/`

Run 3–4, not all. Suggested:
- [`ai_data_analysis_agent`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/starter_ai_agents/ai_data_analysis_agent) — tool calling over tabular data
- [`ai_travel_agent`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/starter_ai_agents/ai_travel_agent) — the canonical single agent + tools
- [`mixture_of_agents`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/starter_ai_agents/mixture_of_agents) — multi-model aggregation, teaches routing intuition
- [`web_scraping_ai_agent`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/starter_ai_agents/web_scraping_ai_agent)

**Exercise:** take one, strip the framework out, and rewrite it with raw `openai` SDK calls and a hand-written tool loop. You must see the `while` loop that reads `tool_calls`, executes, and appends results. Frameworks hide this; you need it in your head.

### Phase 2 — Retrieval: the bread and butter of enterprise AI work
Folder: `rag_tutorials/` — the strongest section of the repo.

Go in this order, it's a difficulty ladder:
1. [`rag_chain`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/rag_tutorials/rag_chain) — minimal pipeline
2. [`local_rag_agent`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/rag_tutorials/local_rag_agent) — Llama 3.2 + Qdrant, no API keys
3. [`hybrid_search_rag`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/rag_tutorials/hybrid_search_rag) — BM25 + dense, the first real quality jump
4. [`corrective_rag`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/rag_tutorials/corrective_rag) — retrieval that grades itself
5. [`agentic_rag_with_reasoning`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/rag_tutorials/agentic_rag_with_reasoning)
6. [`rag_database_routing`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/rag_tutorials/rag_database_routing) — query → correct backing store (very close to your day job)
7. [`knowledge_graph_rag_citations`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/rag_tutorials/knowledge_graph_rag_citations) — multi-hop + attribution
8. [`rag_failure_diagnostics_clinic`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/rag_tutorials/rag_failure_diagnostics_clinic) — **do not skip.** Debugging RAG is the actual job.

**Exercise:** build a RAG over a corpus you personally know well (your own system-design notes in this repo are a perfect corpus). Then write 30 question/answer pairs and measure retrieval hit-rate. Change chunk size from 256 → 1024 and re-measure. That measurement habit is what separates AI engineers from people who "used ChatGPT in a project."

### Phase 3 — Agent frameworks, done properly
Folder: `ai_agent_framework_crash_course/` — contains structured courses for **Google ADK** and the **OpenAI Agents SDK**.

Pick **one** and finish it end to end (structured outputs → tools → memory → callbacks → multi-agent). Then skim a second for comparison. Do not collect frameworks.

Framework guidance:
- **OpenAI Agents SDK** — simplest mental model, handoffs, good default.
- **Google ADK** — model-agnostic, strong on callbacks/plugins, enterprise-friendly.
- **LangGraph** — explicit state machine; the right abstraction for anything with cycles, checkpoints, or human approval steps. Learn this if you're targeting production workflow systems.
- **CrewAI / AG2** — role-based teams; see `agent_teams/` in the repo.

Then move to `advanced_ai_agents/`:
- [`ai_deep_research_agent`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/advanced_ai_agents/single_agent_apps/ai_deep_research_agent) — planner/executor pattern
- [`ai_system_architect_r1`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/advanced_ai_agents/single_agent_apps/ai_system_architect_r1) — reasoning model + generation model split; directly relevant to your domain
- `multi_agent_apps/agent_teams/` — pick any one to see orchestration/handoff

### Phase 4 — Memory, MCP, and the integration layer
- `advanced_llm_apps/llm_apps_with_memory_tutorials/` — short-term vs long-term vs episodic memory; where state actually lives (this is a database design problem, play to your strength)
- `mcp_ai_agents/` — [`github_mcp_agent`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/mcp_ai_agents/github_mcp_agent), [`multi_mcp_agent_router`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/mcp_ai_agents/multi_mcp_agent_router), [`openai_remote_mcp_bridge`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/mcp_ai_agents/openai_remote_mcp_bridge)
- `agent_skills/` — the packaging format for coding-agent capabilities; cheap to author, high visibility

**Exercise:** write your own MCP server exposing a tool over an internal-ish API (start with something you own — a local SQLite DB, your notes, a Jira-like mock). Writing an MCP server is currently one of the highest signal-to-effort portfolio items.

### Phase 5 — Production concerns (your unfair advantage)
The repo is light here by design — this is where you add value beyond it.

- **Evals:** build a golden dataset + LLM-as-judge + run it in CI. Tools: `pytest` + [Ragas](https://docs.ragas.io), [DeepEval](https://github.com/confident-ai/deepeval), [promptfoo](https://promptfoo.dev).
- **Observability/tracing:** LangSmith, Langfuse, or plain OpenTelemetry. Trace every tool call, token count, and latency.
- **Cost control:** `advanced_llm_apps/llm_optimization_tools/` (Toonify, Headroom) — then build your own token budget guard, semantic cache, and small-model-first router with escalation.
- **Reliability:** timeouts, retries with jitter, circuit breakers around model calls, structured-output validation with Pydantic + repair loop, deterministic fallbacks.
- **Security:** treat all model output as untrusted input. Prompt injection via retrieved documents and tool results is real; never let an agent's raw output reach `eval`, a shell, or a SQL string. Enforce allow-lists on tools, scope credentials per tool, sandbox code execution, and log every action. Read the [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/).

### Phase 6 — Optional depth
- `advanced_llm_apps/llm_finetuning_tutorials/` (Gemma 3, Llama 3.2 with Unsloth/LoRA) — do one on Colab so you can speak to it credibly.
- `voice_ai_agents/` and `generative_ui_agents/` — pick up only if your target product needs them.

---

## 4. Portfolio: three things, not thirty

Cloned demos are worth nothing on a resume. Build **three** originals, each with a README containing an eval table and a cost/latency measurement.

1. **A domain RAG with citations and a real eval harness.** Pick a domain where you can judge correctness. Report retrieval hit-rate and answer accuracy before/after each change (chunking, hybrid, re-ranker).
2. **An agent that performs a real action with guardrails.** Not a chatbot — something that writes to a system (opens a PR, files a ticket, updates a sheet) with approval gates, retries, audit log, and a hard cost ceiling.
3. **An MCP server + client for a non-trivial data source**, with auth, pagination, and rate limiting handled properly.

Then write up **one** of them: what failed, what you measured, what you changed. Engineering writing about failure modes is the strongest hiring signal at your level.

---

## 5. What to skip

- Deriving backprop, transformer math from scratch, CUDA kernels — unless you're targeting infra/HPC roles (you do have [hpc-ml-kernels-gpu-offload.md](hpc-ml-kernels-gpu-offload.md), so that's a legitimate *alternate* track, but it's a different job).
- Kaggle competitions and classical ML certificates.
- Collecting frameworks. One agent framework, one vector DB, one eval tool. Depth beats breadth.
- Chasing every model release. The engineering patterns outlive the models.

---

## 6. Positioning with 12 years of experience

Interviews for senior AI engineering roles are ~70% system design. Expect: *"Design a RAG system for 10M documents with sub-2s p95 and a $X/month budget."* That is a system-design interview with new nouns — you already have the muscle. See [system-design-interview-playbook.md](system-design-interview-playbook.md) and [high-level-system-design-cocept.md](high-level-system-design-cocept.md); the same structure applies, with these added axes:

- ingestion & re-indexing pipeline (freshness, deletes, versioning)
- chunking + embedding model choice, embedding cost at scale
- vector store selection, sharding, filtered search, index rebuild strategy
- retrieval quality vs latency vs cost tradeoff
- caching: exact, semantic, and prompt/KV cache
- model routing + fallback + rate-limit handling
- eval and regression strategy, offline and online
- guardrails, injection defense, tenant data isolation
- observability, token/cost attribution per tenant

**Fastest realistic route in:** apply LLM work inside your current job first. Pick one internal pain point (support triage, doc search, test generation, log analysis, code review) and ship it. Internal production experience with real users beats any side project, and it's the story that gets you the title change.

---

## 7. Learning *with* an LLM (agent-driven workflow on this repo)

Yes, development is now largely prompting. That changes **how you build**, not **what you need to know** — and it introduces one specific failure mode you must design around.

### The trap

> Prompting gives you working output without competence. You ship it, then you cannot debug it.

This is survivable in CRUD code. It is fatal in LLM systems, because they fail *silently and probabilistically*: retrieval returns plausible-but-wrong chunks, the agent loops, the judge drifts. There is no stack trace. If you didn't build the mental model, you cannot form a hypothesis, and no amount of re-prompting will save you.

**Operating rule: let the model do the typing, never the deciding.** Architecture, tradeoffs, and "is this actually correct" stay with you. That distinction *is* the senior engineer's job now.

### Workflow A — Make the repo your agent's ground truth

Fast-moving SDKs (OpenAI Agents SDK, ADK, MCP) are exactly where models hallucinate — their training data is stale and the APIs churn. This repo is 100+ **tested, current** implementations. Use it as retrieval, not the model's memory.

```bash
git clone https://github.com/Shubhamsaboo/awesome-llm-apps.git
code awesome-llm-apps
```

Then prompt against the workspace rather than the model's recall:

- *"Find every place in `#codebase` that does chunking. Show the 3 distinct strategies and the tradeoffs between them."*
- *"Compare how `openai_sdk_crash_course` and `google_adk_crash_course` handle memory. Where do they disagree architecturally?"*
- *"I'm about to write an MCP server. Extract the exact server-side pattern from `mcp_ai_agents/` — don't write it from memory."*

Cross-template comparison is the highest-value prompt type here. It's what a book can't give you and what a single template can't either.

### Workflow B — Prompt patterns that build skill instead of bypassing it

| Pattern | Prompt | Why it works |
|---|---|---|
| **Predict-then-run** | *"Explain what this outputs. Don't run it."* → you predict → then run | Mismatch between prediction and reality is the actual learning event |
| **Socratic** | *"Don't give me code. Ask me questions until I can write it myself."* | Forces retrieval from your own memory |
| **Delete-and-rebuild** | Delete a core function, write it yourself, then *"Review my version against the original. What did I miss and why does it matter?"* | Generation, then correction — the strongest learning loop |
| **Port it** | *"Rewrite this LangChain example using the raw OpenAI SDK and a hand-written tool loop."* | Framework abstractions hide the mechanism; porting exposes it |
| **Failure injection** | *"Break this in 5 realistic ways. Describe only the symptom, not the cause."* | You diagnose. This is the RAG-debugging skill from Phase 2 |
| **Review inversion** | You write it, then *"Review as a staff engineer. Rubric: correctness, failure modes, cost, injection surface."* | Flips you from consumer to author |
| **Steelman the alternative** | *"Argue why this architecture is wrong and what you'd do instead."* | Counters the model's agreeableness |

Avoid *"build me an X"* as your default prompt. It's the one that produces the trap.

### Workflow C — Configure your agent as a tutor, not an autocomplete

In VS Code you can version-control the prompting itself:

- `*.agent.md` — a dedicated **Tutor** mode with tools restricted to read-only, so it physically cannot write code for you
- `*.prompt.md` — reusable drills, e.g. `/teardown` takes a template path and returns architecture → decisions → demo-vs-production gaps → 3 questions for you
- `.github/copilot-instructions.md` — repo-wide standing rules (e.g. *"Explain before generating. Never write more than 40 lines without me confirming the approach. Always cite the template you're pattern-matching against."*)

**Already set up** (workspace scope — version-controlled with these notes):

| File | Use |
|---|---|
| [.github/agents/llm-tutor.agent.md](.github/agents/llm-tutor.agent.md) | Pick **llm-tutor** in the agent selector. Tools: `read, search, web` only — no edit, no terminal. Modes: teardown, Socratic, predict-then-run, delete-and-rebuild, port-it, failure injection, review inversion, steelman. |
| [.github/prompts/teardown.prompt.md](.github/prompts/teardown.prompt.md) | Type `/teardown` then a template path. Runs on the tutor agent. |

Because they're workspace-scoped they only load in *this* workspace. To use them while reading the cloned `awesome-llm-apps`, either add that folder to this workspace (multi-root) or copy the two files into its `.github/`.

This is the same discipline as the repo's `agent_skills/` folder: **prompts as engineering artifacts** — reviewed, versioned, CI-tested. Learning to author these is itself a marketable AI-engineering skill.

### Workflow D — The meta project (highest ROI in this repo)

Use [`chat_with_github`](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/advanced_llm_apps/chat_with_X_tutorials/chat_with_github) and point it **at this repo**.

You end up building the RAG system that teaches you RAG. Every quality problem you hit is a real lesson: code chunks badly on fixed-size splitting, READMEs and source need different strategies, symbol names demand hybrid search. Then extend it: add re-ranking, add citations back to file paths, add an eval set of 30 questions you know the answers to. Now you have Portfolio Project 1 and a genuinely useful tool.

### Workflow E — Templates as scaffolds, not tutorials

Apache-2.0, and the author's stated position is "fork it, ship it, sell it." So the production loop is:

1. Find the closest template
2. Have the agent explain it fully — you must be able to redraw it on a whiteboard
3. Fork it, then have the agent adapt it to *your* data and *your* constraints
4. Replace the demo-grade parts yourself: hardcoded keys, no retries, no timeouts, no cost ceiling, no eval, no auth
5. Pin dependency versions

Step 4 is where your 12 years show up. Templates are demos; the gap between demo and production is entirely your existing skillset.

### Verification discipline (non-negotiable)

The model is confidently wrong most often on exactly the things you're learning:

- **Run everything.** Never accept LLM-written LLM code you haven't executed.
- **Pin versions.** `openai==x.y.z`, not `openai`. Then ask *"which SDK version is this API from?"* and verify against real docs — deprecated agent APIs are a top hallucination source.
- **Write the eval before the feature.** If you can't state how you'd measure it, you can't prompt your way to it.
- **Read the whole diff.** If you'd reject it in a PR review from a junior, reject it from the agent.
- **Treat retrieved and scraped content as hostile.** When you point an agent at a web page or a repo issue, that content can carry instructions. Never let agent output reach a shell, `eval`, or a SQL string unvalidated.

### Anti-patterns

- Accepting a 400-line agent implementation in one shot
- Letting the model choose the architecture, the vector DB, or the framework
- Prompting hardest on the parts you understand least — that's backwards; use assistance to go *faster* where you're competent and *slower* where you aren't
- Collecting generated projects instead of measured ones

### The daily loop

Pick one template → agent explains architecture (you ask 3 "why not X?" questions) → delete a core file → rebuild it with the agent as reviewer, not author → break it deliberately → fix it **without** the agent → write one eval → write 5 lines of notes in this repo on what surprised you.

Notes-you-wrote beat code-the-model-wrote. That's the whole method.

---

## 8. Companion resources

- [Anthropic — Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) — read this early; best conceptual piece on when *not* to use an agent
- [OpenAI — A Practical Guide to Building Agents](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf)
- [Model Context Protocol docs](https://modelcontextprotocol.io)
- Chip Huyen, *AI Engineering* (O'Reilly) — the closest thing to a textbook for this role
- [Unwind AI](https://www.theunwindai.com/) — the newsletter behind this repo; new templates weekly

---

## Progress tracker

- [ ] Phase 0 — env, keys, Ollama running
- [ ] Phase 1 — 3 starter agents run; one rewritten without a framework
- [ ] Phase 2 — RAG ladder through `corrective_rag`; hit-rate measured on own corpus
- [ ] Phase 3 — one framework crash course completed end to end
- [ ] Phase 4 — own MCP server written
- [ ] Phase 5 — eval suite running in CI; cost/latency dashboard
- [ ] Phase 6 — one LoRA fine-tune completed
- [ ] Tutor agent / instructions file configured (Workflow C)
- [ ] `chat_with_github` pointed at this repo, extended with re-ranking + evals (Workflow D)
- [ ] Portfolio project 1 / 2 / 3
- [ ] One internal production use case shipped at work
