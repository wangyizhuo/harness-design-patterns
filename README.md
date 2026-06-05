# Harness Design Patterns

> A GoF-style catalog for AI agent harness engineering — 11 patterns across 3 families.

**65% of enterprise AI failures trace to three defect families** (Masood, 2026): context drift, schema misalignment, and state degradation. This catalog names the patterns that fix them structurally — not with prompt patches.

---

## Defect families (root causes)

| Family | What degrades | Observable symptom |
|---|---|---|
| **Context drift** | Information quality in the context window | Agent ignores prior decisions, gets lost in codebase |
| **Schema misalignment** | Fit between harness data model and reality | Wrong field values, deprecated columns, silent nulls |
| **State degradation** | Persistence and safety of agent actions | Irreversible actions, infinite loops, confident wrong answers |

### The compounding error problem

At 95% step accuracy over 20 steps: `0.95^20 = 36%` task success. Context degrades 2% per step — after 5 steps, less than 60% of original context survives (MemU, 2026).

### The lethal trifecta (Willison, 2025)

When all three defect families converge: an agent that **(1)** processes external input + **(2)** has access to sensitive data + **(3)** can modify state. A single prompt injection can exfiltrate data and corrupt state simultaneously.

---

## Pattern index

| # | Pattern | Family | Solves |
|---|---|---|---|
| 1 | [Progressive disclosure](#1-progressive-disclosure) | Structural | Context drift / orientation tax |
| 2 | [Least privilege tool registry](#2-least-privilege-tool-registry) | Structural | Context drift / prompt injection |
| 3 | [Explicit state store](#3-explicit-state-store) | Structural | State degradation / invisible state |
| 4 | [Plan-execute-verify loop](#4-plan-execute-verify-pev-loop) | Behavioral | State degradation / compounding error |
| 5 | [Structured observation](#5-structured-observation) | Behavioral | Schema misalignment / tool output mismatch |
| 6 | [Checkpoint gate](#6-checkpoint-gate) | Behavioral | State degradation / all-or-nothing autonomy |
| 7 | [Circuit breaker](#7-circuit-breaker) | Behavioral | State degradation / infinite loop |
| 8 | [Trust boundary labeling](#8-trust-boundary-labeling) | Behavioral | Context drift / prompt injection |
| 9 | [Schema sentinel](#9-schema-sentinel) | Adaptive | Schema misalignment / schema drift |
| 10 | [Error-to-constraint loop](#10-error-to-constraint-loop) | Adaptive | All three families |
| 11 | [Context compaction](#11-context-compaction) | Adaptive | Context drift / context rot |

---

## Family I — Structural patterns

> How you build the harness skeleton. These patterns are **feedforward** — they prevent bad outcomes before generation starts.

---

### 1. Progressive disclosure

| | |
|---|---|
| **Intent** | Supply the agent with exactly the context it needs for the current sub-task — no more, no less. |
| **Problem** | Dumping the entire codebase or conversation history into every prompt causes context flooding: the agent gets lost, latency spikes, cost compounds. Too little context causes the orientation tax — thousands of tokens spent on grep-spree exploration before any useful work begins. |
| **Solution** | Structure context delivery in layers. Start with a minimal orientation artifact (repository map, task spec, relevant file list). Expand only when the agent explicitly needs a deeper layer. Expose only the absolute minimum telemetry required for the current sub-task. |
| **When to apply** | Any task longer than a single-turn exchange. Especially critical for coding agents on large codebases. Apply as the default architecture — do not wait for an orientation tax symptom. |
| **Consequences** | Reduces context rot and orientation tax. Requires upfront investment in building the structural map (AGENTS.md, SKILL.md, repo map). The map itself must be maintained — it drifts if not updated. |
| **Real-world example** | OpenAI Codex uses `AGENTS.md` as the progressive disclosure artifact. Adding negative examples to skill bundles improved routing accuracy from 73% to 85% (OpenAI engineering guide, 2026). |
| **Related** | Pairs with [Pattern 11](#11-context-compaction) (Context compaction) for long-running sessions. |

---

### 2. Least privilege tool registry

| | |
|---|---|
| **Intent** | Grant the agent access only to the tools it needs for the current scope — enforced as a structural constraint, not a prompt instruction. |
| **Problem** | Over-provisioned tool access is the harness equivalent of running production as root. OWASP 2026 identifies excessive agency as a primary risk: over-provisioned functions, unnecessary permissions, missing approval mechanisms. When an agent with write access to a production database also processes untrusted external input, you have the lethal trifecta. |
| **Solution** | Define a tool allowlist per task phase. The harness enforces it at the infrastructure layer — the agent cannot call a tool not on the allowlist, regardless of what the model reasons. Tool definitions, parameters, and return types are typed contracts, not prose descriptions. When 50+ tools are available, use dynamic loading to retrieve only the top-k relevant tools per query. |
| **When to apply** | Always. Especially critical when the agent processes any untrusted input: user content, external APIs, web scraping, or RAG retrieval from external sources. |
| **Consequences** | Reduces blast radius of prompt injection and state degradation defects. Adds friction to introducing new tools — which is the point. Shifts "agent output does not match expected structure" from a runtime bug to a type-check failure. |
| **Real-world example** | GitHub Enterprise's April 2026 governance guide enforces MCP server registry curation with ruleset-protected configurations and cloud-agent firewall allowlisting. |
| **Related** | Pairs with [Pattern 6](#6-checkpoint-gate) (Checkpoint gate) for irreversible actions, and [Pattern 8](#8-trust-boundary-labeling) (Trust boundary labeling) for untrusted input. |

---

### 3. Explicit state store

| | |
|---|---|
| **Intent** | Never use the LLM's context window as the authoritative state record. Externalize all state that must survive beyond one turn. |
| **Problem** | Invisible state is the most common state degradation defect. The LLM has no persistent memory — every session starts blank. Research from MemU (2026) quantifies the degradation: 2% context retention loss per step. At 5 cycles in a multi-step workflow, less than 60% of original context is reliably accessible. |
| **Solution** | Maintain a structured state document outside the model — a file, a database row, or a session checkpoint. The harness reads it at the start of each turn and writes it at the end. The model sees only the current state snapshot, not raw history. Session checkpointing and explicit memory via filesystems allow the environment to guide the agent toward correct execution. |
| **When to apply** | Any multi-turn or multi-session workflow. Any task where a decision made in step 3 must be honored in step 17. Treat this as a baseline requirement for any agent that runs more than 5 turns. |
| **Consequences** | Makes state auditable and debuggable — you can replay any session. Adds the state store as a harness component that must be maintained. The state schema can itself drift, making Pattern 9 (Schema sentinel) a natural companion. |
| **Real-world example** | The AHE framework decomposes the harness into seven orthogonal, git-tracked, file-level components. Long-term memory is one of the seven explicitly managed components (Lin et al., arXiv:2604.25850). |
| **Related** | Pairs with [Pattern 11](#11-context-compaction) (Context compaction) for managing what goes into the state snapshot. |

---

## Family II — Behavioral patterns

> How the harness governs what the agent does at runtime. These patterns are **corrective** — they catch defects during execution before they cause irreversible damage.

---

### 4. Plan-execute-verify (PEV) loop

| | |
|---|---|
| **Intent** | Separate reasoning, execution, and validation into distinct phases, each with its own model instance and quality gate. |
| **Problem** | Without verification at each step, even 95% step accuracy yields only 36% task success over 20 steps (`0.95^20`). Errors compound silently until the final output is wrong by a margin that looks like model failure but is actually a harness failure. |
| **Solution** | Structure every non-trivial task as a three-phase loop. The **Planner** (high-reasoning model) produces a step-by-step plan with explicit success criteria. The **Executor** (cheaper model) carries out each step. The **Verifier** (high-reasoning model) checks the output against criteria before proceeding. This Reasoning Sandwich ensures high-reasoning capacity is applied at decision boundaries, not rote execution. |
| **When to apply** | Tasks with more than 3–4 sequential steps. Any task where step N depends on the correctness of step N-1. Financial analysis, multi-file code generation, and multi-stage research are canonical use cases. |
| **Consequences** | Increases latency and cost (multiple model calls per task). Dramatically reduces compounding error defects. Makes failure modes visible — when verification fails, you know exactly which step and why. |
| **Real-world example** | Stripe's Minions system (2026 engineering post) uses a PEV-equivalent loop for financial data processing pipelines, with verification by a separate model instance before any output is committed. |
| **Related** | Pairs with [Pattern 6](#6-checkpoint-gate) (Checkpoint gate) for high-stakes verification points. |

---

### 5. Structured observation

| | |
|---|---|
| **Intent** | Ensure the agent always receives a structured, typed result from every tool call — never a raw string or a dangling promise. |
| **Problem** | When a tool returns an error, timeout, or unexpected shape, an unguarded agent either hallucinates a plausible result or enters an infinite retry loop. Fixing a malformed tool response in the prompt does not work — the problem recurs with the next variant. |
| **Solution** | Every tool call is wrapped by the harness in an observation envelope with a typed result, a status code, and an error message field. The agent receives the envelope, not the raw response. Whether it is a successful API response, a permission denial, or a timeout, the agent always receives a structured observation — no dangling promises. |
| **When to apply** | All tool integrations, always. Essential when integrating external MCPs where the tool's behavior is not under your control. Apply before any tool enters production, not after the first malformed response incident. |
| **Consequences** | Catches schema misalignment defects at the tool boundary before they propagate into the agent's reasoning. Adds a thin wrapper per tool — cost is low relative to the protection. The observation schema itself must be versioned. |
| **Real-world example** | PydanticAI is the reference implementation: tool definitions, parameters, and return values are Pydantic models. This shifts "agent output does not match expected structure" from a runtime debugging problem to a type-check failure at development time. |
| **Related** | Pairs with [Pattern 9](#9-schema-sentinel) (Schema sentinel) to catch changes in the tool's output schema over time. |

---

### 6. Checkpoint gate

| | |
|---|---|
| **Intent** | Require explicit human or automated approval before any irreversible, high-blast-radius action. |
| **Problem** | All-or-nothing autonomy is the single most catastrophic state degradation defect. The Replit postmortem (July 2025) is definitive: an agent executed `DROP DATABASE` on a production system despite a freeze instruction in the guide. No permission boundary prevented the destructive action, and no approval gate required human sign-off before schema-altering operations. |
| **Solution** | Classify all tool calls by blast radius before deployment. Low-blast actions (read, search, draft) proceed automatically. High-blast actions (delete, deploy, send, pay) pause, surface a diff, and require explicit commit before execution. This is the **draft-commit pattern** — dangerous actions are first drafted, then explicitly committed. Anthropic's auto mode research found that users approve 93% of prompts, making approvals meaningless; the solution is a two-stage classifier that reserves human review for genuinely flagged actions only. |
| **When to apply** | Whenever the agent can take an action that cannot be undone or is expensive to reverse. Budget exhaustion should also trigger a gate. Design the gate to be narrow and meaningful — broad approval requirements create fatigue that defeats their purpose. |
| **Consequences** | Dramatically reduces all-or-nothing autonomy failures. Creates approval fatigue if overused. Gate classification must be maintained as new tools are added to the registry. |
| **Real-world example** | Google Cloud's agentic design documentation: at a predefined checkpoint, the agent pauses and calls an external system to wait for a person to review its work before the agent can continue. |
| **Related** | Pairs with [Pattern 2](#2-least-privilege-tool-registry) (Least privilege tool registry) and [Pattern 4](#4-plan-execute-verify-pev-loop) (PEV loop) for high-stakes verification. |

---

### 7. Circuit breaker

| | |
|---|---|
| **Intent** | Detect when the agent is in a failure loop and terminate gracefully before burning compute and money. |
| **Problem** | Agents can enter infinite retry loops when a tool fails, a plan assumption is wrong, or the task is underspecified. Without a circuit breaker, the harness runs until token budget exhaustion — a silent cost drain that is invisible in logs until the bill arrives. |
| **Solution** | Track consecutive failures per step and total steps per task. When either threshold is exceeded, the harness terminates the loop, captures the failure state, and returns a structured failure artifact rather than a blank timeout. The structured failure artifact is the input to [Pattern 10](#10-error-to-constraint-loop) (Error-to-constraint loop). |
| **When to apply** | All autonomous loops. Set thresholds conservatively — you can always raise them when you understand the task's normal step distribution. Replanning thresholds (trigger replanning after N consecutive step failures) are a softer variant before hard termination. |
| **Consequences** | Converts silent hangs into observable, logged failures. Every circuit break is a data point for harness improvement. Requires threshold calibration per task type. |
| **Real-world example** | LangGraph 2.0's `interrupt` mechanism provides a production-ready circuit breaker primitive, including replanning triggers when step output contradicts plan assumptions. |
| **Related** | Every circuit break should trigger [Pattern 10](#10-error-to-constraint-loop) (Error-to-constraint loop). |

---

### 8. Trust boundary labeling

| | |
|---|---|
| **Intent** | Mark all untrusted data with a trust label before it enters the agent's context, so the harness can treat it differently from trusted internal content. |
| **Problem** | Prompt injection is effective precisely because the agent cannot distinguish instructions from data. A PDF the agent reads might contain "Ignore all previous instructions." Simon Willison's **lethal trifecta**: an agent that processes external input + has access to sensitive data + can modify state. A single successful injection can exfiltrate data and corrupt state simultaneously. |
| **Solution** | Every piece of data entering the context is tagged with a trust tier at the harness layer — not inside the prompt. User input, external API responses, scraped content, and retrieved documents are tagged `untrusted`. Internal state, verified schema, and system prompts are tagged `trusted`. The harness enforces that untrusted content cannot override trusted instructions, preventing prompt injection from escalating to arbitrary code execution. |
| **When to apply** | Any agent that processes external content — RAG pipelines, web agents, document processors, or any agent that reads user-supplied files. Apply before the first untrusted data source is integrated, not after an injection incident. |
| **Consequences** | Eliminates the largest prompt injection attack surface. Requires discipline in MCP connectors — each connector must tag its output correctly. The trust taxonomy must be documented and enforced in code review. |
| **Real-world example** | OWASP LLM06:2025 (Excessive Agency) is the authoritative checklist for auditing harness permission scope against least privilege. GitHub Enterprise's 2026 agent governance guide enforces trust boundaries at the MCP server registry level. |
| **Related** | Pairs with [Pattern 2](#2-least-privilege-tool-registry) (Least privilege tool registry) and [Pattern 6](#6-checkpoint-gate) (Checkpoint gate) for defense in depth. |

---

## Family III — Adaptive patterns

> How the harness learns and improves over time. These patterns make the harness a **living system** rather than a static artifact — converting failures into structural improvements and managing context quality across long sessions.

---

### 9. Schema sentinel

| | |
|---|---|
| **Intent** | Detect schema drift before it reaches the agent by monitoring upstream data sources for changes and invalidating stale context. |
| **Problem** | Schema misalignment is invisible until it causes a wrong output — by which point the damage is done. Schema drift occurs when field definitions, naming conventions, or semantic meaning shift across data assets over time without the change propagating to the harness context. An agent operating on `customer_name = "J. Smith"` when the system now uses `"John A. Smith"` produces outputs that are technically confident and factually catastrophic. 39% of data engineers cite schema drift as their top AI risk. |
| **Solution** | The harness registers a sentinel on every schema it uses. When a field is renamed, removed, or redefined upstream, the sentinel fires and invalidates the affected context before any agent session reads it. The sentinel can be a database trigger, a data catalog event, or a scheduled validator. The invalidation event must trigger either a harness update or a graceful degradation path — not a silent null. |
| **When to apply** | Any harness that reads from external databases, APIs, or RAG indexes not under your direct control. Apply when onboarding any new MCP data connector. Schema sentinels should be the first component built, not added after the first production incident. |
| **Consequences** | Converts silent invisible failures into observable events. The sentinel itself must be maintained — it is a harness artifact that can drift. |
| **Real-world example** | A financial services firm experienced a $2M loss attributed initially to model error. The actual cause was schema drift: the same field name in two systems mapped to different calculations. The harness was never notified of the change. |
| **Related** | Pairs with [Pattern 5](#5-structured-observation) (Structured observation) to catch schema changes at the tool output boundary. |

---

### 10. Error-to-constraint loop

| | |
|---|---|
| **Intent** | When the agent makes a mistake, convert the failure into a new structural constraint — not a prompt patch. |
| **Problem** | The instinct after an agent error is to fix the prompt: add a warning, clarify the instruction, repeat the rule. This is the harness equivalent of a blameless postmortem that ends with "be more careful." Prompt fixes are probabilistic compliance — the agent may or may not follow them next session. Structural constraints are deterministic enforcement. |
| **Solution** | After every agent failure, run a structured retrospective: which harness layer was responsible — constraint (structural), feedback loop (behavioral), or quality gate (adaptive)? Then add or tighten a structural control at that layer. The fix is a code change, a linter rule, a new gate, a typed schema, or a schema sentinel — **never just a prompt addition**. Mitchell Hashimoto's core principle: *every time the agent makes a mistake, change the system so that mistake structurally cannot recur.* |
| **When to apply** | After every production incident. Especially after any circuit breaker trigger (Pattern 7), checkpoint rejection (Pattern 6), or schema sentinel fire (Pattern 9). This is the retrospective discipline that makes the harness compound in quality rather than accumulate in complexity. |
| **Consequences** | The harness gets systematically better with every failure rather than accumulating prompt debt. Requires dedicated time per incident — the retrospective must result in a harness code change, not a documentation update. |
| **Real-world example** | AHE (Lin et al., arXiv:2604.25850): ten iterations of error-to-constraint evolution lifted pass@1 on Terminal-Bench 2 from 69.7% to 77.0%, surpassing the human-designed Codex-CLI baseline at 71.9%. The frozen evolved harness transfers across model families. |
| **Related** | This is the integrating pattern — it consumes the failure outputs of all other patterns and converts them into improvements. |

---

### 11. Context compaction

| | |
|---|---|
| **Intent** | Proactively compress and summarize context before it degrades, not reactively after the context window fills. |
| **Problem** | Context rot is a gradual process — the harness cannot wait for the window to fill before acting. At 2% misalignment per step, context degrades to 40% failure rate by step 25. Reactive truncation (cutting the oldest messages when the window fills) loses critical early decisions. |
| **Solution** | The harness maintains a structured session state document (intent, decisions made, pending work, key findings). At each turn, the harness compacts the raw trajectory into a digest and updates the document — the agent reads the digest, not the raw history. Compaction is **trigger-based, not reactive**: it fires when context density crosses a threshold, not when the window is full. A CDR-compliant system monitors planner recursion depth, context density, and memory saturation in real time, and mitigates through episodic consolidation before drift compounds. |
| **When to apply** | All long-running sessions. Mandatory for any task expected to exceed 10 turns. Implement compaction before you observe degradation symptoms — by the time the agent produces visibly worse output, significant context has already been lost. |
| **Consequences** | Trades some raw trajectory fidelity for dramatically improved reliability over long tasks. The compaction summary itself must be high-quality — poor summarization introduces a subtler form of context drift. Trigger threshold requires calibration per task type. |
| **Real-world example** | OpenAI's 2026 engineering guide documents server-side compaction via an explicit `/responses/compact` endpoint as one of three core production harness primitives, alongside versioned skill bundles and a managed shell container. |
| **Related** | Pairs with [Pattern 3](#3-explicit-state-store) (Explicit state store) — the compacted digest feeds the state store rather than the raw context window. |

---

## Pattern decision framework

Match the observed failure symptom to the minimum pattern that resolves it. Each additional pattern adds latency and maintenance cost — apply the minimum viable set for your observed failure modes.

| You observe… | Defect family | Apply pattern |
|---|---|---|
| Agent gets lost in large codebase | Context drift / orientation tax | 1 — Progressive disclosure |
| Agent ignores prior decisions after many turns | Context drift / context rot | 3 — Explicit state store + 11 — Compaction |
| External document changes agent behavior | Context drift / prompt injection | 8 — Trust boundary labeling |
| Agent uses wrong field name from external DB | Schema misalignment | 9 — Schema sentinel |
| Tool returns unexpected shape, agent hallucinates | Schema misalignment | 5 — Structured observation |
| Agent deletes or deploys without review | State degradation / autonomy | 6 — Checkpoint gate |
| Agent runs indefinitely on a broken step | State degradation / loop | 7 — Circuit breaker |
| Same mistake recurs after prompt fixes | Any defect family | 10 — Error-to-constraint loop |
| Output confidence is high but answers are wrong | State degradation / silent | 4 — PEV loop |
| >50 tools available, selection accuracy degrades | Context drift / flooding | 2 — Least privilege registry |

---

## Key sources

- Meng et al. (2026). *Agent Harness for Large Language Model Agents: A Survey.* Preprints.org. DOI: [10.20944/preprints202604.0428.v1](https://doi.org/10.20944/preprints202604.0428.v1)
- Lin, Liu et al. (2026). *Agentic Harness Engineering: Observability-Driven Automatic Evolution of Coding-Agent Harnesses.* [arXiv:2604.25850](https://arxiv.org/abs/2604.25850)
- Ning, Tieu et al. (2026). *Code as Agent Harness.* [arXiv:2605.18747](https://arxiv.org/abs/2605.18747)
- Masood, A. (2026). *Agent Harness Engineering — The Rise of the AI Control Plane.* Medium.
- Böckeler, B. (2026). *Harness Engineering for Coding Agent Users.* [martinfowler.com](https://martinfowler.com/articles/harness-engineering.html)
- Hashimoto, M. (2026). *My AI Adoption Journey.* mitchellh.com
- Lopopolo, R. (2026). *Harness Engineering.* OpenAI Blog.
- OWASP LLM06:2025 — Excessive Agency. [owasp.org](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- Google Cloud Architecture Center (2026). *Choose a design pattern for your agentic AI system.*
- Cloud Security Alliance (2026). *Agentic AI Predictions for 2026.*

---

*Generated June 2026. Maintained at [github.com/wangyizhuo/harness-design-patterns](https://github.com/wangyizhuo/harness-design-patterns)*
