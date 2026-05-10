# LangGraph

*Evaluated: LangGraph 1.0.x (GA Oct 22 2025) and 1.2.0a streaming/timeout previews. As of May 2026.*

## Summary

LangGraph is a low-level Python/TypeScript runtime for building stateful, long-running LLM agents as directed graphs. Each node is a function over a typed state object; edges (static or conditional) drive control flow; a pluggable checkpointer persists state at every superstep. It hit a stable 1.0 in October 2025 with no breaking changes from late-0.x and is the only mainstream agent framework where **durable execution and human-in-the-loop interrupts are core primitives rather than add-ons** — the same checkpoint that makes a run resumable also makes it pause-able mid-graph for review. It is not a high-level "describe your crew" abstraction like CrewAI, nor a vendor SDK like OpenAI's Agents SDK; it is closer to a workflow engine specialized for LLM control flow. Pick it when you need fine-grained control over branching, retries, partial failure, and human review — at the cost of a steeper ramp than role-based frameworks and a strong gravitational pull toward the LangChain/LangSmith ecosystem.

## Comparison

| Dimension | **LangGraph 1.0** | **AutoGen 0.7** (Microsoft) | **OpenAI Agents SDK** (Apr 2026) | **CrewAI 1.10** |
|---|---|---|---|---|
| Type / category | Graph runtime for stateful agents | Async actor framework for multi-agent chat | Vendor-anchored agent SDK + sandbox harness | Role/process orchestration framework |
| Core architecture | Directed `StateGraph` of typed nodes over a shared state object; checkpointer persists per superstep | Three-layer actor model (Core / AgentChat / Extensions) with async message passing | `Agent` + `Runner` driving an LLM-tool loop inside a sandboxed harness with configurable memory | `Crew` of role-typed `Agent`s executed via a `Process` (sequential or hierarchical) |
| Primary interfaces | Python + TS SDKs; LangGraph Server (REST/SSE/WebSocket); LangGraph Studio | Python SDK; .NET SDK; CLI Studio | Python SDK (TS planned); Responses API + sandbox tools | Python SDK; YAML crew/task configs |
| State model | Typed `TypedDict` / Pydantic / dataclass; reducer-merged per channel; checkpointed | Per-agent local state; messages on a shared bus | Per-`Agent` config + run-scoped memory store | Implicit conversation/context per agent, task-level outputs |
| Durable execution | First-class: `exit` / `async` / `sync` durability modes; full pause/replay/time-travel | Not built in; users wire their own persistence | Memory persistence yes; full workflow resumability is not a primitive | Not built in; recent flow-mode adds limited persistence |
| Human-in-the-loop | First-class `interrupt()` + static `interrupt_before/after` tied to checkpoint | Manual via custom agent or UserProxy patterns | `tool_use_behavior` and approval hooks per tool | Manual; relies on tool wrappers |
| Streaming | Native: token, values, updates, debug, custom; v2 typed events; v3 typed per-channel projections (1.2 alpha) | Token + event streaming via runtime | Token + tool-event streaming; native Responses API streaming | Token streaming via underlying LLM client |
| Best fit | Long-running, branchy, audit-able agents needing replay and HITL | Research / prototyping multi-agent dialogue | OpenAI-centric agents that need code-execution sandboxes and Responses API features | Fast-to-build role-based pipelines (research → write → review) |
| Advantages | Explicit graph + state; durable by default; rich deploy options; tight LangSmith tracing | Clean actor model; strong group-chat patterns; .NET parity | Sandbox + filesystem tools out of box; tightest fit to OpenAI Responses/Realtime; 100+ LLMs via Chat Completions | Lowest time-to-first-agent; readable role abstraction; strong community |
| Disadvantages | Steeper learning curve; opinionated state mental model; observability story really wants LangSmith | In maintenance mode — Microsoft now points new work at the Microsoft Agent Framework | Python-first; vendor-anchored; weaker story for long-horizon resumable workflows | Less control at scale; reported ~18% token overhead vs LangGraph on equivalent multi-agent tasks; production posture still maturing |
| License | MIT (OSS); LangGraph Platform is commercial | MIT (OSS) | MIT (OSS); LLM calls billed via OpenAI API | MIT (OSS); CrewAI Enterprise is commercial |
| Cost (rough, May 2026) | Framework free. Platform: $0.001/node executed + $0.005/deployment-run; Self-Hosted Lite free to 1M nodes; LangSmith Plus $39/seat/mo. Typical 5-eng prod team: ~$140–300/mo on LangSmith; large deploys $650–6,000+/mo | Framework free; you pay for whatever LLM + infra you bring | Framework free; pay standard OpenAI API rates plus sandbox/tool execution costs | Framework free; CrewAI Enterprise is custom-priced |
| Deployment options | Cloud SaaS, BYOC (AWS only today), Self-Hosted Enterprise (cloud / hybrid / fully on-prem), Self-Hosted Lite | Self-host anything that runs Python/.NET; Azure-friendly | OpenAI's hosted runtime; self-hostable Python service hitting the API | Self-host; CrewAI Enterprise SaaS |
| Multi-language | Python, TypeScript | Python, .NET | Python (TS planned) | Python only |

*Cost figures are public-list estimates from vendor pricing pages as of May 2026 and will move; treat them as order-of-magnitude.*

## In-depth report

### 1. Architecture deep-dive

The unit of execution is a **`StateGraph`** parameterized by a state schema (`TypedDict`, Pydantic model, or `dataclass`):

```
        ┌─────────┐  conditional   ┌─────────┐
START ─▶│  plan   │───────edge────▶│  tool   │──▶ ...
        └────┬────┘                 └────┬────┘
             │                           │
             ▼                           ▼
        ┌─────────┐                 ┌─────────┐
        │ respond │ ◀───────────────│ reflect │
        └────┬────┘                 └─────────┘
             ▼
            END
```

Nodes are plain functions `(state) -> partial_state`. The runtime merges each node's return value back into the shared state via per-channel **reducers** (e.g. `add_messages` for an append-only message channel, or a custom merge for a counter). This is closer to a Pregel-style superstep model than a procedural call graph: in a single step the runtime decides which nodes are runnable, executes them (in parallel where the graph allows), then commits state.

Compilation (`graph.compile(checkpointer=...)`) produces a runnable that exposes `invoke`, `stream`, `astream`, `get_state`, `update_state`, and `aget_state_history`. The compiled object is also the unit deployed by **LangGraph Server** as a versioned "assistant" with REST, SSE, and WebSocket endpoints.

State persistence sits on a `BaseCheckpointSaver` interface implementing `put`, `get_tuple`, `list`, and `delete_thread`. Production deployments typically use `PostgresSaver` (`langgraph-checkpoint-postgres`) or `RedisSaver`; `MemorySaver` is dev-only. Checkpoints are organized into **threads** (a thread ≈ a conversation / session), and within a thread each step gets a checkpoint with metadata, channel values, and pending writes. That structure is what makes `aget_state_history()` and time-travel debugging work.

### 2. Key design patterns and trade-offs

- **Explicit state over implicit conversation.** AutoGen and CrewAI tend to thread agent context through chat messages. LangGraph forces you to declare the channels and reducers up front. Verbose for simple cases, but it makes parallelism, partial updates, and replay tractable. The decision pays off the moment you add a second agent or a parallel branch.
- **Graph over decorator/agent loop.** OpenAI's Agents SDK and most "agent" libraries hide the loop inside `Runner.run()`. LangGraph exposes the loop as a graph; you can intercept any edge, insert a node, or short-circuit by returning `Command(goto=...)`. Power, at the price of boilerplate.
- **Durability mode is configurable.** `exit` (cheapest, persist on completion or interrupt), `async` (persist concurrently with the next step — small crash-window risk), `sync` (persist before next step starts — the default for high-stakes flows). This is a real workflow-engine concept, not a checkpoint hand-wave.
- **Interrupts are persistence, not threads.** `interrupt(value)` raises a `GraphInterrupt`, the checkpointer stores the pending state, and the run is resumed by re-invoking the graph with `Command(resume=human_value)` on the same `thread_id`. There is no waiting Python coroutine on the server — the process can die and resume on a different worker hours later.
- **Reducers are how concurrency is sane.** Two parallel nodes returning `{"messages": [m1]}` and `{"messages": [m2]}` get merged via the channel's reducer; without reducers, last-write-wins would silently drop work.

### 3. Correctness and durability model

- **Atomicity unit:** the superstep. Either all writes from the step land in the checkpoint or none do. A node crashing mid-step rolls the step back; a retry re-runs the node from the prior checkpoint.
- **Failure recovery:** re-invoke the graph with `input=None` and the same `thread_id`; the runtime resumes from the last successful checkpoint. This works across process restarts, host failures, and LLM provider outages.
- **Determinism:** not guaranteed — LLM calls are non-deterministic by nature — but the *graph traversal* is deterministic given fixed state and node outputs, which is what makes time-travel debugging meaningful. Writes from a re-executed step replace pending writes; you don't get duplicate side-effects from idempotent node code, but you must make external side-effects (API calls, payments) idempotent yourself.
- **Time travel:** `aget_state_history(thread_id)` returns all checkpoints; `update_state(config, values, as_node=...)` lets you fork from any checkpoint with new values and resume — useful for debugging and for "what if the user had answered differently" flows.
- **Known sharp edge (fixed in v2 streaming, March 2026):** earlier versions could reuse stale `RESUME` values across replays in subgraph + interrupt combinations; if you are still on pre-`version="v2"` streaming with subgraphs and HITL, upgrade.

### 4. Performance characteristics

LangGraph's overhead is dominated by checkpointer I/O and reducer cost, not framework dispatch. A few concrete points from public sources, all of which should be re-validated for your workload:

- A 2026 third-party benchmark comparing CrewAI and LangGraph on equivalent multi-agent ticket-triage tasks reports LangGraph completing **76%** of medium (3–5 tool calls) tasks vs CrewAI **71%**, with CrewAI using **~18% more tokens** for the same task shape. Source is a vendor-adjacent blog, so treat as directional, not authoritative.
- Per-step checkpoint write to Postgres is typically a few ms over the LLM call; for high-throughput workloads, **`durability="async"`** drops checkpoint cost off the hot path at the price of a small crash-window risk.
- The 1.2 alphas (May 2026) introduce **`DeltaChannel`** that stores incremental deltas instead of full channel snapshots — meaningful for long-running threads with growing message lists, where a naive PostgresSaver checkpoint can balloon.
- Per-node `run_timeout` and `idle_timeout` (also 1.2 alpha) raise `NodeTimeoutError`, which composes with retry policies. Before this, runaway nodes were a real production problem — you had to wrap calls yourself.
- Streaming has three coexisting protocols today: legacy dict events, **`version="v2"`** typed `StreamPart`s, and **`version="v3"`** content-block/per-channel projections. v3 is the right target for new code; v1 still works.

### 5. Operational model

- **Local dev:** `langgraph dev` launches Studio with hot reload; `MemorySaver` is fine.
- **Self-host:** `langgraph build` produces a Docker image of the LangGraph Server; backed by Postgres for state, Redis for the task queue, and your own LLM credentials. **Self-Hosted Lite** is free up to 1M nodes executed, then you upgrade to **Self-Hosted Enterprise** (no migration).
- **BYOC (AWS only as of May 2026):** LangChain operates the control plane; data plane runs in your VPC. GCP/Azure BYOC has been promised but is not yet GA — verify before committing.
- **Cloud SaaS:** managed control + data plane.
- **Observability:** LangSmith is the path of least resistance — auto-instrumented traces, evaluators, prompt versioning, datasets. OpenTelemetry export exists but is less mature; if you have an opinionated tracing stack (Datadog, Honeycomb, Phoenix, Langfuse) you'll do more wiring. Allocate budget for this.
- **Common failure modes:** (a) checkpointer-table bloat on long threads — use `DeltaChannel` or thread-rollover; (b) Postgres connection-pool starvation under high concurrency; (c) interrupted runs that never get resumed quietly accumulating in the `checkpoints` table — add a sweeper.

### 6. Security and multi-tenancy

- **AuthN/AuthZ:** LangGraph Server supports custom auth via `langgraph_sdk.Auth` (per-thread/assistant scopes); some deployment combinations historically had issues with custom auth on self-hosted before specific versions — pin to current.
- **Encryption:** at-rest depends on the checkpointer backend (e.g. Postgres TDE, RDS encryption); in-flight TLS at the server.
- **Tenant isolation:** typically modeled as `thread_id` namespaced by tenant + `Auth` scopes. The framework does not isolate tenants at the OS/process layer — that's your deployment's job.
- **Prompt/data leak surface:** state is plaintext in the checkpointer; if you persist user PII through long-running threads, the checkpointer table is now in scope for your data-handling policy.

### 7. Ecosystem and integrations

- **Models:** anything via `langchain` model wrappers — OpenAI, Anthropic, Google, Bedrock, vLLM, Ollama, etc.
- **Tools:** the LangChain tool ecosystem (~750+ integrations) plus MCP (Model Context Protocol) is supported via `langchain-mcp-adapters`.
- **Vector stores / retrieval:** LangChain retrievers plug in as nodes.
- **Tracing & evals:** LangSmith (first-class), OTel (workable), Phoenix/Langfuse (community).
- **Frontend:** LangGraph Studio for graph viz / state inspection / time-travel; CopilotKit, AG-UI, and the Generative UI features for app integration.
- **Deployment adjacents:** Kubernetes via the published Helm chart for Self-Hosted Enterprise; Terraform modules exist but coverage varies.

### 8. Sub-comparison: durable-execution stacks

LangGraph is sometimes evaluated against general workflow engines (Temporal, Restate) rather than other agent frameworks. A focused look:

| Dimension | **LangGraph** | **Temporal** | **AutoGen** |
|---|---|---|---|
| Primary abstraction | Typed-state graph of LLM-aware nodes | Workflow + activity functions | Async actors exchanging messages |
| Determinism guarantees | Graph traversal deterministic; node bodies free-form | Strict workflow determinism enforced by replay | Not enforced |
| Built-in HITL | `interrupt()` is core | Custom signals/queries | Custom |
| LLM ergonomics | First-class (streaming, tool nodes, message reducers) | None — bring your own | First-class for chat |
| Best fit | LLM agents that need durability | Any durable workflow, LLM or not | Multi-agent dialogue research |

If you'd use Temporal *purely* for an LLM workflow, LangGraph gives you most of what you want with a much shorter ramp. If your workflow is mostly non-LLM (orders, payments, reconciliation) with an LLM step inside, Temporal is still the right backbone.

### 9. When to pick LangGraph — and when not to

Pick it when:

- You need **resumable, audit-able** agent runs (compliance, support, research, code-mod) — durable execution is the differentiator.
- You need **fine-grained branching** with parallel nodes, conditional edges, and partial state updates.
- **HITL** is a product requirement, not an afterthought — approvals, edits, escalations.
- You are already on the **LangChain/LangSmith** stack, or willing to be.
- You need **multi-language** (Python + TypeScript) parity for the same agent definition.

Skip it when:

- Your agent is a single-LLM tool loop with no durability or branching needs — `openai-agents`, plain LangChain, or even direct API calls are fewer moving parts.
- You want a **role/persona abstraction out of the box** — CrewAI is faster from zero.
- You are **OpenAI-anchored** and want first-class Responses-API features (computer use, code interpreter, file search) — OpenAI Agents SDK has tighter fit there.
- You are committed to **.NET** as the primary language — AutoGen still has the only credible .NET story among these frameworks.
- Your team won't tolerate the **LangChain ecosystem footprint** (many packages, fast-moving APIs) or you can't afford LangSmith and don't want to wire OTel by hand.

### 10. Closing TL;DR

LangGraph in 2026 is the de facto pick when you treat an agent as a long-running, stateful workflow rather than a one-shot LLM call. v1.0 made the API stable; the active 1.2 alphas (delta channels, per-node timeouts, v3 streaming) are sanding off the production rough edges. Its differentiator over CrewAI, AutoGen, and the OpenAI Agents SDK is not better LLM ergonomics — it's that **checkpointed durable execution and human-in-the-loop interrupts are the same primitive**. Buy that primitive and the steeper learning curve and LangSmith gravity are usually a fair trade.

## Sources

- [LangGraph 1.0 GA announcement (LangChain changelog)](https://changelog.langchain.com/announcements/langgraph-1-0-is-now-generally-available)
- [What's new in LangGraph v1 — official docs](https://docs.langchain.com/oss/python/releases/langgraph-v1)
- [LangGraph changelog (full)](https://docs.langchain.com/oss/python/releases/changelog)
- [LangGraph releases on GitHub](https://github.com/langchain-ai/langgraph/releases)
- [Persistence & checkpointers — official docs](https://docs.langchain.com/oss/python/langgraph/persistence)
- [Durable execution — official docs](https://docs.langchain.com/oss/python/langgraph/durable-execution)
- [LangGraph Platform deployment options](https://changelog.langchain.com/announcements/langgraph-platform-new-deployment-options-for-agent-infrastructure)
- [LangSmith pricing](https://www.langchain.com/pricing)
- [AutoGen v0.4 architecture (Microsoft Research)](https://www.microsoft.com/en-us/research/blog/autogen-v0-4-reimagining-the-foundation-of-agentic-ai-for-scale-extensibility-and-robustness/)
- [AutoGen GitHub](https://github.com/microsoft/autogen)
- [OpenAI Agents SDK — next evolution (Apr 2026)](https://openai.com/index/the-next-evolution-of-the-agents-sdk/)
- [OpenAI Agents SDK docs](https://openai.github.io/openai-agents-python/)
- [CrewAI GitHub](https://github.com/crewaiinc/crewai)
- [LangGraph vs CrewAI vs AutoGen 2026 benchmarks (third-party blog — directional)](https://pooya.blog/blog/crewai-vs-langgraph-autogen-comparison-2026/)
- [LangGraph vs CrewAI 2026 (Redwerk)](https://redwerk.com/blog/langgraph-vs-crewai/)
