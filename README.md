# Cognitive Swarm Orchestrator for MoFA: A Governance-Focused Multi-Agent Collaboration Engine

### Technical Approach

#### Understanding of MoFA Architecture

My current understanding is that MoFA is structured as a microkernel-based Rust framework with a clear separation between layers.



![Figure 1: MoFA Layered Architecture Overview](figures/fig01-mofa-layered-architecture.png)

*Figure 1: How MoFA is organized into layers. The Swarm Orchestrator will span kernel, foundation, and runtime, with public APIs surfaced through the SDK.*

The layers break down like this:

- `mofa-kernel`: core traits, types, and contracts for agents, messaging, storage, workflows, gateway, and more.
- `mofa-foundation`: concrete, production-oriented implementations of those contracts, like orchestration helpers, persistence, secretary agents, and coordination utilities.
- `mofa-runtime`: lifecycle and execution management, including SimpleRuntime and optional Dora-based distributed execution.
- `mofa-sdk`: the curated public API that re-exports kernel, foundation, and runtime types for end users and foreign language bindings.

Idea 5 describes the Cognitive Swarm Orchestrator as the brain of this ecosystem. It does not execute business logic itself. Instead, it analyzes incoming tasks, breaks them into subtasks, matches agents based on their capabilities, picks coordination patterns, and then uses existing MoFA components to actually execute and observe those flows. The key insight is that different parts of a task benefit from different expert models, and the orchestrator's job is to assign the right expert to each part and coordinate them using the right pattern.

It connects with:

- **Gateway** for capability access to tools, devices, and external APIs. In current MoFA this is already a major platform layer, including routing, rate control, observability hooks, and distributed control-plane concerns. Gateway is not a small add-on; it is a large-scale subsystem that the orchestrator treats as a first-class integration surface for reaching any external capability an agent might need.
- **Smith / Observatory** for traces, metrics, and evaluation of swarm runs.
- **SDK** for polyglot users to call into orchestrated swarms from other languages.


![Figure 2: Swarm Orchestrator Ecosystem Integration](figures/fig02-orchestrator-ecosystem-integration.png)

*Figure 2: How the Swarm Orchestrator connects to Gateway, Smith, SDK, and the Agent Registry. Each arrow maps to a concrete integration point built in Phases 3 through 5.*

The idea text lists several named components that I plan to follow closely:

- `TaskAnalyzer` for building a task DAG and finding the critical path.
- `SwarmComposer` for dynamic agent team formation and pattern selection.
- `HITLGovernor` for the human-in-the-loop lifecycle, including escalation.
- A `GovernanceLayer` for SLAs, audits, and notifications.
- A plugin marketplace core with dependency resolution and trust scoring.
- A semantic agent discovery system built on an agent capability registry, plus semantic agent generation when no suitable prebuilt agent exists.

My plan is to put the core orchestration traits and types in the kernel where that makes sense, and to implement the main orchestrator logic and integration glue in foundation and runtime, so the Swarm Orchestrator fits naturally into the framework instead of sitting off to the side.

---

#### Deep Dive: What Already Exists and What This Project Adds

I have read through the `mofa-main` repository in detail, and also explored `mofa-studio-main`, `mofaclaw-main`, `makepad-chart-main`, and `makepad-d3-main` to understand the full ecosystem. The following is a concrete mapping of what the Swarm Orchestrator reuses, extends, and builds new.

##### What Already Exists in `mofa-main` (Reuse and Extend)

**Agent system and capabilities.** The `MoFAAgent` trait in `mofa-kernel/src/agent/core.rs` already defines a unified agent interface with `id()`, `name()`, `capabilities()`, `execute()`, and lifecycle methods. The `AgentCapabilities` struct tracks what an agent can do, including reasoning strategies (`Direct`, `ReAct`, `ChainOfThought`, `TreeOfThought`), tool usage, and memory types. The orchestrator does not redefine any of this. Instead, `SwarmComposer` will query `AgentCapabilities` to match subtasks to agents that have the right skills, building directly on the metadata that agents already expose.

**Coordinator trait and collaboration modes.** The `Coordinator` trait in `mofa-kernel/src/agent/components/` already defines `dispatch()`, `aggregate()`, and `pattern()` for multi-agent coordination. In `mofa-foundation/src/collaboration/`, there are already six LLM-driven collaboration mode processors: `RequestResponse`, `PublishSubscribe`, `Consensus`, `Debate`, `Parallel`, and `Sequential`, plus a `Custom` variant for user-defined patterns. Each has a dedicated processor with an `LLMProtocolHelper` for intelligent message routing. The Swarm Orchestrator will not replace these modes. Instead, it will sit above them and decide which mode to apply to each section of a task DAG, and then delegate execution down to the existing processors. The orchestrator adds the intelligence of when and where to use each pattern, not a reimplementation of the patterns themselves.

**Swarm module with DAG and risk awareness.** The `mofa-foundation/src/swarm/` module already has `SwarmConfig`, `SwarmScheduler` (with `SequentialScheduler` and `ParallelScheduler` implementations), `SubtaskDAG` with `DependencyKind` (Sequential, DataFlow, Soft) and `RiskLevel` (Low, Medium, High, Critical), plus `TaskAnalyzer` for basic risk analysis and audit logging. This is a strong foundation. The orchestrator project will extend this existing swarm module with richer task decomposition logic (LLM-driven DAG construction), dynamic pattern annotation per subtask, and the council-style debate pipeline. The existing `SubtaskDAG`, `DependencyKind`, and `RiskLevel` types will be reused directly.

**Secretary agent and HITL patterns.** The secretary pattern is already well-defined across kernel and foundation. `SecretaryBehavior` in `mofa-kernel/src/agent/secretary/traits.rs` defines the abstract interface, and `DefaultSecretaryBehavior` in `mofa-foundation/src/secretary/default/` implements the five phases: Idea Reception, Clarification, Dispatch, Monitoring, and Reporting. Supporting components include `TodoManager`, `Clarifier`, `Coordinator`, `Monitor`, `Reporter`, and `AgentRouter` (with `LLMAgentRouter`, `RuleBasedRouter`, `CapabilityRouter`, and `CompositeRouter` implementations). The `HITLGovernor` in this proposal maps directly onto this five-phase secretary model. Rather than building a new HITL system from scratch, I will extend `SecretaryBehavior` to handle orchestration-specific escalation flows (from Debate and action safety), add SLA tracking to the Monitor phase, and wire the Reporter phase to emit `AuditEvent` records.

**Communication bus.** The `AgentBus` in `mofa-kernel/src/bus/mod.rs` already supports three messaging modes: `PointToPoint`, `Broadcast`, and `PubSub(topic)`. Agent messages in `mofa-kernel/src/message/mod.rs` include `TaskRequest`, `TaskResponse`, `StateSync`, `Event`, `StreamMessage`, and `StreamControl` with priority and deadline support. The orchestrator will use the existing bus for all inter-agent communication, dispatching subtasks via `TaskRequest` and collecting results via `TaskResponse`. No new messaging primitives are needed.

**Gateway.** The `mofa-gateway` crate implements a distributed control plane with Raft consensus, route matching, rate limiting (token bucket + sliding window), authentication (API key + JWT), and circuit breakers. The kernel defines gateway traits like `RouteRegistry`, `GatewayRateLimiter`, `AuthProvider`, and `ApiKeyStore` in `mofa-kernel/src/gateway/`. The orchestrator will use gateway routes for external capability access (tools, APIs, devices) without reimplementing any routing or rate-limiting logic.

**Monitoring and observability.** The `mofa-monitoring` crate provides a web-based dashboard with Prometheus metrics, WebSocket real-time updates, and a REST API for historical data. Agent metrics (execution count, success rate, latency), LLM metrics (token usage, cost), and workflow metrics are already tracked. The orchestrator will emit its own metrics (swarm runs, debate outcomes, HITL decisions) through the same Prometheus interface and OpenTelemetry tracing hooks.

**Plugin system and scripting.** The `mofa-plugins` crate implements a dual-layer plugin architecture: compile-time plugins (Rust/WASM with zero runtime overhead) and runtime plugins (Rhai scripts via `mofa-extra` with hot-reload, resource limits, and sandboxing). The plugin marketplace component of this proposal builds on this existing system by adding a registry data model, semantic search, and trust scoring on top of the plugin infrastructure that already exists.

**LLM integration.** The `mofa-foundation/src/llm/` module provides `LLMClient` and `LLMProvider` abstractions with implementations for OpenAI, Anthropic, Ollama, Google, and custom providers, plus streaming, retry policies, fallback chains, and token budget enforcement. The orchestrator's TaskAnalyzer and Debate Judge will use these LLM clients directly, not build new ones.

**Runtime and Dora.** The `mofa-runtime` crate offers `SimpleRuntime` for single-process execution and `DoraAdapterRuntime` for distributed dataflow via Dora-rs. The `AgentBuilder` provides a fluent API for agent construction with configuration, plugin integration, and capability declaration. The native dataflow module provides the same abstractions as Dora but with pure tokio, suitable for single-machine deployments. The orchestrator will execute swarm plans through whichever runtime the user has configured, using the existing `AgentBuilder` for constructing expert agent instances.

**FFI and SDK.** The `mofa-ffi` crate generates multi-language bindings via UniFFI for Python, Java, Go, Kotlin, and Swift. The `mofa-sdk` crate provides a curated public API surface. The orchestrator's SDK integration (Phase 4) will add high-level orchestration functions to `mofa-sdk` and ensure they are accessible through the existing FFI pipeline.

**CLI.** The `mofa-cli` crate provides agent lifecycle management commands (`new`, `start`, `stop`, `status`, `list`, `logs`) plus plugin management, configuration, database, and diagnostics commands. The orchestrator will integrate with the CLI so that swarm runs can be inspected and managed through familiar commands.

**Workflow engine.** The `StateGraph` system in `mofa-kernel/src/workflow/` provides a LangGraph-inspired workflow engine with `Reducer`, `NodePolicy` (retry, circuit breaker, timeout), and `ControlFlow` (End, goto, send). The orchestrator's combined-pattern execution engine can leverage `StateGraph` for managing the lifecycle of complex DAG executions with built-in retry and circuit breaker support.

##### What This Project Builds New

Given the strong foundation above, the new work focuses on the orchestration intelligence layer, not infrastructure:

| Component | Status | What Changes |
|-----------|--------|-------------|
| `TaskAnalyzer` (LLM-driven DAG) | Extends existing `TaskAnalyzer` in `swarm/` | Adds LLM-based task decomposition, complexity/risk/uncertainty signal extraction, and mode routing (Simple/Complex/Debate) |
| `SwarmComposer` (pattern selection) | New, builds on `Coordinator` trait | Adds dynamic pattern annotation per DAG section, pattern-aware agent matching, and agent generation fallback |
| Council-style Debate | Extends existing `Debate` collaboration mode | Adds parallel proposal generation, cross-examination, judge synthesis with evidence scoring, and confidence gating |
| `HITLGovernor` | Extends `SecretaryBehavior` | Adds orchestration-specific escalation (from Debate + action safety), SLA deadline tracking, and structured approval context |
| Action Safety Policy | New | Decision tree for classifying agent actions by risk level (read/write/delete/chmod/sudo) with filesystem awareness |
| `GovernanceLayer` | New | Core objects (AuditEvent, SLAState, DecisionRecord) + I/O wiring to persistence, alerting, and monitoring |
| Dynamic mode routing | New | Runtime policy that classifies tasks into Simple/Complex/Debate before orchestration begins |
| Notification adapters | New | Webhook and console/log adapters for governance alerts |
| Plugin marketplace core | Extends existing plugin system | Adds registry data model, SemVer resolution, trust scoring, semantic search, and pattern-aware ranking |
| Semantic agent generation | New | Controlled fallback for generating scaffolded specialist agents when no prebuilt agent matches |



![Figure 2A: Crate-Level Integration Map](figures/fig2A-crate-integration-map.png)

*Figure 2A: How the new `mofa-orchestrator` crate fits into the existing mofa-main workspace. Solid borders are existing crates; the dashed border is the new crate. Solid arrows show where existing code is directly reused. Dashed arrows show new integration points. The orchestrator builds on top of existing kernel traits, foundation modules, and runtime infrastructure rather than replacing them.*

##### Ecosystem Context: How the Broader Repositories Inform This Work

Beyond `mofa-main`, the MoFA ecosystem includes several other repositories that provide important context for how the Swarm Orchestrator should be designed.

**MoFA Studio** (`mofa-studio-main`) is a GPU-accelerated desktop voice chat application built with Makepad and Dora dataflow. Its architecture demonstrates real-world multi-agent orchestration at scale: the voice-chat pipeline coordinates 14+ Dora nodes (ASR, LLM inference, TTS, conference bridge, text segmentation) across multiple participants with backpressure control and turn-taking policies managed by a `dora-conference-controller`. The Swarm Orchestrator's coordination engine should be designed so that it can eventually orchestrate agent workflows of this complexity, including the kind of multi-stage, mixed-mode pipelines (parallel LLM inference across 3 student agents, sequential audio processing, conference bridging) that MoFA Studio already runs through Dora. The orchestrator's Dora runtime integration path (via `DoraAdapterRuntime` in `mofa-runtime`) makes this a natural future extension.

**MofaClaw** (`mofaclaw-main`) is a lightweight personal AI assistant with a clean agent loop pattern (prompt → LLM → tool selection → tool execution → repeat) and multi-channel support (DingTalk, Feishu, Telegram, Discord, WhatsApp). Its `AgentRouter` with capability-based, rule-based, and LLM-based routing strategies directly parallels what the orchestrator's `SwarmComposer` needs to do. The notification adapter design in this proposal (webhook, console/log) also draws from MofaClaw's multi-channel gateway pattern, where a single core routes messages to different external platforms. MofaClaw's tool system (filesystem operations, shell execution, web scraping, process spawning) also informs the action safety classification policy, since these are exactly the kinds of real-world actions that agents perform and that need risk-aware governance.

**Makepad Charts** (`makepad-chart-main`) and **Makepad D3** (`makepad-d3-main`) provide GPU-accelerated data visualization capabilities for the Makepad UI framework. Makepad Charts offers 11 chart types with animation, and Makepad D3 provides D3.js-compatible primitives including force-directed graph layouts, geographic projections, and advanced statistical charts. These are relevant because the monitoring and observability integration (Phase 3) can leverage these charting libraries through MoFA Studio to provide visual dashboards for swarm execution, including Gantt-style timeline views of task DAG execution, network graphs of agent collaboration patterns, and real-time metric charts for orchestration performance. The orchestrator's audit trail and governance data are designed to be structured enough to feed directly into these visualization tools.



![Figure 2B: MoFA Ecosystem Repository Context](figures/fig2B-ecosystem-repository-context.png)

*Figure 2B: How the five ecosystem repositories relate to each other and to the Swarm Orchestrator. The orchestrator lives inside mofa-main and builds on its infrastructure. MoFA Studio demonstrates the kind of complex multi-agent pipelines the orchestrator should eventually support. MofaClaw informs the routing and action safety design. The charting libraries provide visualization capabilities for monitoring.*

---

#### Implementation Plan

I will build the project in several phases that line up with the idea description. Each phase produces something tangible (crates, types, APIs, examples) and maps back to the GSoC timeline.

---

##### Phase 1: Core Orchestration Engine (TaskAnalyzer and SwarmComposer)

The goal here is to set up the minimal orchestrator core that can understand tasks and form teams of expert models.



![Figure 3: Task Decomposition and Agent Matching Flow](figures/fig03-task-decomposition-flow.png)

*Figure 3: How a high-level task flows through TaskAnalyzer (decomposition into a DAG of subtasks with dependencies) and SwarmComposer (matching each subtask to the best expert model via capability lookup). This is the first step before any coordination pattern is applied.*

Notice that in Figure 3, each subtask gets its own expert model. The Code Expert writes the patch, the Test Expert writes verification, and the Security Expert reviews for safety. This is the core value of the swarm approach: each model does what it is best at, instead of asking one general model to do everything. We are using different models for what they are known to be best at.

What I plan to do:

- Define core orchestration types and traits like `TaskDescriptor`, `Subtask`, `SwarmPlan`, and `CoordinationPattern` in a new `mofa-orchestrator` crate under the `mofa` workspace. These types will compose with existing kernel types: `SubtaskDAG` and `DependencyKind` from `mofa-foundation/src/swarm/` will be reused directly for DAG representation, and `RiskLevel` from the same module will drive the action safety classification. New types will be limited to orchestration-specific concerns like `SwarmPlan` (which DAG sections use which pattern) and `ModeSelection` (Simple/Complex/Debate routing).
- Extend the existing `TaskAnalyzer` in `mofa-foundation/src/swarm/` with LLM-driven task decomposition. The current analyzer does basic risk analysis; the enhanced version will use `LLMClient` from `mofa-foundation/src/llm/` to extract complexity, risk, and uncertainty signals from natural language task descriptions, and to produce richer DAGs with more granular subtask descriptions and dependency annotations. The key is that the LLM step is optional: for structured inputs (like pre-decomposed task lists), the analyzer can bypass LLM and build the DAG directly.
- Build `SwarmComposer` as a new component that maps subtasks to agent roles and individual agents. It will query `AgentCapabilities` (from `mofa-kernel/src/agent/core.rs`) for capability matching and use `AgentRouter` strategies (from `mofa-foundation/src/secretary/`) for intelligent selection. The first version will use the existing `CapabilityRouter` with a basic scoring function, but the API will allow plugging in `LLMAgentRouter` or `CompositeRouter` for smarter matching later. Importantly, SwarmComposer also annotates each subtask with a recommended coordination pattern (see Phase 2 and Figure 5 for how patterns are combined).
- Build a minimal in-memory representation and serialization format for swarm plans, so they can be logged, inspected, and passed to later components. Serialization will use serde, consistent with the rest of the MoFA codebase.

From a MoFA perspective, this phase is mainly about designing the orchestrator API so it integrates cleanly with existing agent metadata and capability representations, and making the core components testable on their own with artificial agent definitions.

---

##### Phase 2: Coordination Patterns, Debate, and HITL Governor

The goal is to support multiple collaboration modes, prove that multi-expert coordination produces better results than single-expert execution, and implement a basic human-in-the-loop governance flow.

###### Individual Patterns



![Figure 4: Coordination Patterns Comparison](figures/fig04-coordination-patterns.png)

*Figure 4: Three coordination patterns the orchestrator can use. Sequential chains experts one after another. Parallel fans out independent multimodal subtasks (image, video, audio) simultaneously. Debate pits two experts against each other and resolves with a judge. Each pattern uses specialized expert models chosen for what they are best at.*

Figure 4 shows each pattern in isolation. But in practice, the orchestrator does not pick just one pattern for the entire task. Different parts of the same task DAG call for different patterns. The next figure shows how they combine. Before that, the orchestrator first decides which mode the overall task needs.

**How this connects to existing code.** The `mofa-foundation/src/collaboration/` module already implements all three patterns as LLM-driven collaboration mode processors. The `Sequential` processor chains agent outputs, the `Parallel` processor runs agents concurrently and aggregates, and the `Debate` processor implements multi-round iterative discussion. Each has its own processing logic via `LLMProtocolHelper`. The Swarm Orchestrator does not reimplement these processors. Instead, it wraps them in a higher-level coordination engine that can apply different processors to different sections of the same task DAG. The orchestrator's job is the routing and composition logic, the decision of which processor to use where, while the actual execution of each pattern delegates to the existing collaboration mode processors.

###### Dynamic Runtime Routing Before Pattern Selection

Before pattern selection, the orchestrator applies a mode-routing policy. The big picture is that the whole process is dynamic. Not every task needs the same level of orchestration:

- **Simple mode**: direct low-risk tasks with clear intent. Example: "What is the weather in New York?" This simply requires one expert model, direct execution, and a direct answer. No DAG, no debate.
- **Complex mode**: multi-step tasks requiring DAG decomposition and mixed patterns. Example: "Fix a security bug, add tests, and ship a PR." This needs multiple expert models coordinated across sequential, parallel, and potentially debate stages.
- **Debate mode (extra thinking)**: ambiguous or high-impact tasks where multiple expert views reduce blind spots. This is selective. It activates only when the uncertainty or stakes justify the overhead.

If the user already provides a detailed spec document with clear constraints and acceptance criteria, Debate can be skipped entirely. The spec itself serves as the "extra thinking" — there is nothing ambiguous left to debate, so execution proceeds directly through deterministic checks.



![Figure 4A: Dynamic Mode Routing Policy](figures/fig4A-dynamic-mode-routing-policy.png)

*Figure 4A: Runtime policy for routing tasks into Simple, Complex, or Debate mode. Simple tasks get one expert and direct execution. Complex tasks get full DAG orchestration. Debate activates only when uncertainty justifies extra thinking, and can be skipped entirely when a detailed spec already provides clear constraints.*

###### The Bigger Picture: Combining Patterns on a Task DAG



![Figure 5: Combined Patterns on a Task DAG](figures/fig05-combined-patterns-dag.png)

*Figure 5: The bigger picture. A single task DAG uses Sequential for dependent subtasks, Parallel for independent multimodal work (image, video, audio running simultaneously), and Debate for high-risk decisions. The orchestrator picks the right pattern for each section of the DAG, not one pattern for the whole task. If confidence is low after Debate, HITLGovernor takes over.*

This is the core architecture of the coordination engine. The orchestrator reads the task DAG (produced by TaskAnalyzer in Phase 1), looks at each section, and decides:

- **Sequential** when subtasks depend on each other. You cannot write tests before the patch exists.
- **Parallel** when subtasks are independent. Image analysis, video analysis, and audio analysis can all run at the same time because they do not depend on each other's output.
- **Debate** when the decision is high-impact or uncertain, and two different expert perspectives can catch different failure modes. This is where the real value of multi-expert swarms shows up.

Notice how Figure 5 connects back to Figure 3. The TaskAnalyzer produces the DAG, SwarmComposer assigns expert models and pattern hints, and then the coordination engine executes the plan using the right pattern for each section.

**How this maps to the existing codebase.** The existing `SequentialScheduler` and `ParallelScheduler` in `mofa-foundation/src/swarm/` handle basic sequential and parallel task execution. The orchestrator's combined-pattern engine wraps these schedulers and adds the DAG-section-level pattern selection logic. For debate sections, it delegates to the `Debate` collaboration mode processor in `mofa-foundation/src/collaboration/`. The key new logic is the DAG walker that traverses the `SubtaskDAG`, identifies which groups of subtasks should use which pattern based on dependency structure and risk annotations, and then dispatches each group to the appropriate scheduler or processor. This walker is the primary new code in the coordination engine; everything below it (actual scheduling, message passing, agent execution) uses existing infrastructure.

###### Council-Style Debate Design

The Debate pattern is implemented as a council-style mechanism inspired by modern multi-model systems. Think of it like the council mode in advanced AI platforms where multiple top-tier models (like GPT, Claude, and Gemini) are sent the same prompt simultaneously, each bringing its own strengths and blind spots. The system deliberately creates useful disagreement:

1. **Parallel proposal generation** from diverse expert models. Each model is selected specifically because it has different strengths, ensuring a wide range of perspectives.
2. **Optional cross-examination** where experts critique each other's reasoning to find flaws or missing data points.
3. **Judge synthesis** using hard checks and evidence scoring, not rhetorical confidence. The judge does not pick whichever answer sounds best. It runs constraint checks and evaluates evidence.
4. **Confidence gate** deciding accept, revise, or escalate to HITL.

You can think of Debate as "extra thinking." It is not a default step for every request. It is activated when uncertainty and impact justify the overhead of running multiple expert models in parallel.

**How this extends the existing Debate mode.** The `Debate` collaboration processor in `mofa-foundation/src/collaboration/` already implements multi-agent iterative discussion with configurable max rounds and LLM-driven message processing. The council-style extension adds three things on top: (1) structured parallel proposal generation where each expert produces a formal proposal object (not just free-text discussion), (2) a Judge synthesis step that runs deterministic checks (test results, constraint satisfaction) rather than relying solely on LLM evaluation, and (3) a confidence gate that explicitly routes low-confidence results to HITLGovernor instead of silently picking the best-looking answer. The existing `LLMProtocolHelper` will be used for the cross-examination step where experts critique each other.


![Figure 5A: Council-Style Debate Pipeline](figures/fig5A-council-debate-pipeline.png)

*Figure 5A: Council-style debate as structured extra thinking. Multiple expert models reason in parallel (each selected for different strengths), optionally cross-examine each other, then a judge synthesizes using evidence and constraints, not rhetorical confidence. If confidence is high, the plan executes. If not, it escalates to a human.*

###### Why Debate Adds Value (and how we will prove it)

Debate is not useful because two models talk to each other. It is useful because two models that are different in meaningful ways will catch different kinds of mistakes.

Think about it this way: most agent failures are not "completely wrong answer" failures. They are "right-looking answer, wrong action" failures. For example:

1. A model proposes a patch that looks correct but breaks an edge case in the tests.
2. A model suggests a safe change but forgets a corner case that causes a regression.
3. A model optimizes for speed but violates a rate limit or security constraint.

A single expert model might not catch its own blind spots. But if a second expert model with different priorities reviews the same problem, the disagreement between them becomes a signal. The Judge step then resolves that disagreement using hard evidence (like test results or constraint checks), not just by picking whichever answer sounds more confident.

The key: differences between models are what make Debate valuable. If both models think the same way, Debate adds nothing. The value comes from pairing models with genuinely different strengths.

**Real-world example (concrete and tied to Figure 5)**

Task: "Fix an off-by-one bug in a Rust function, keep performance the same."

This task enters the Debate zone in Figure 5 because it involves a tradeoff between correctness and performance.

- Security Expert (Agent A) focuses on boundary conditions. It proposes a fix that is correct but slightly slower because it adds an extra bounds check.
- Performance Expert (Agent B) looks for faster alternatives. It proposes a fix that is faster but accidentally reintroduces the boundary error in a subtle way.
- Judge checks both proposals against hard criteria: "do all unit tests pass?" and "is runtime within 5% of the baseline?" Agent A's proposal passes both. Agent B's proposal fails the test suite. Judge selects Agent A.

If we had used only Agent B (a single expert), we would have shipped a broken patch. The Debate pattern caught the mistake because the two experts disagreed, and the Judge had concrete evidence to resolve the disagreement.

###### Debate Test Cases (directly tied to the Figure 4 Debate column)

Each test below maps to the "Security Expert, Performance Expert, Judge" flow from Figure 4. Each one has a diagram showing exactly what happens step by step.

---

**Test 1: `debate_selects_patch_that_passes_unit_tests`**



![Figure 6: Debate Test 1](figures/fig06-debate-test1-patch-selection.png)

*Figure 6: Debate Test 1. Agent A proposes a correct but slower fix. Agent B proposes a fast but broken fix. The Judge runs the test suite and selects Patch A because it passes all tests. The audit trail records the rationale.*

Setup:
- Task: "Fix off-by-one bug in function `foo` so all tests pass."
- Agent A (correctness-focused) proposes Patch A: correct but slightly slower.
- Agent B (performance-focused) proposes Patch B: fast but fails one boundary test.
- Judge runs a deterministic test suite check (or mocked equivalent that returns pass/fail).

Expected:
- Judge selects Patch A because it passes all tests.
- Audit trail records: `debate_used=true`, `judge_rationale=all_tests_passed`, `rejected=Patch B (test failure)`.

---

**Test 2: `debate_resolves_safety_vs_performance_tradeoff`**



![Figure 7: Debate Test 2](figures/fig07-debate-test2-constraint-resolution.png)

*Figure 7: Debate Test 2. Agent A respects the rate limit. Agent B violates it. The Judge checks both hard constraints and selects Agent A. If neither had passed, the system escalates to HITLGovernor instead of silently picking a risky option.*

Setup:
- Task: "Optimize a service handler but keep a strict rate limit rule."
- Agent A (safety-focused) proposes a conservative solution that respects the rate limit.
- Agent B (performance-focused) proposes a fast solution but weakens the rate limiting condition.
- Judge checks both constraints: "no rate limit violation" and "latency within threshold."

Expected:
- Judge selects Agent A because it satisfies all hard constraints.
- If neither proposal satisfies all constraints, Judge triggers escalation to HITLGovernor (Figure 10), because we should not silently pick a risky option.

---

**Test 3: `debate_detects_shared_blind_spot_and_escalates`**

This test checks what happens when Debate does NOT help, which is just as important.



![Figure 8: Debate Test 3](figures/fig08-debate-test3-blind-spot-escalation.png)

*Figure 8: Debate Test 3. Both experts share the same blind spot. The Judge finds both proposals fail on the same evidence. Instead of picking a winner based on surface reasoning, the system admits uncertainty and escalates to HITLGovernor. This proves the system does not blindly trust Debate.*

Setup:
- Both Agent A and Agent B make the same incorrect assumption (they are wrong in the same way).
- Judge compares their arguments and discovers both lead to the same test failure.

Expected:
- Judge does not declare a winner based on surface reasoning alone.
- Orchestrator triggers the Clarify or Escalate path through HITLGovernor, because the disagreement signal was weak and the evidence says "we still do not know."
- Audit trail records: `debate_used=true`, `judge_rationale=no_clear_winner`, `escalated=true`.

This test is important because it proves the system does not blindly trust Debate. When both experts share a blind spot, the system admits uncertainty and asks for human help.

---

###### Systematic Verification: Proving Swarm is Better Than Single Expert

I will verify the value of multi-expert coordination using a repeatable evaluation process:


![Figure 9: Verification Comparison Setup](figures/fig09-verification-comparison.png)

*Figure 9: The three evaluation modes. The same benchmark tasks run through a single expert, a swarm without Debate, and a full swarm with Debate. Results are compared using four measurable metrics. This is how we prove "swarm is better" with evidence instead of just claiming it.*

1. Define a set of benchmark tasks with known success criteria (for example "all unit tests pass", "no permission changes outside workspace", "latency stays within 5% of baseline").

2. Run each task in three modes:
   - **Single-expert baseline**: one general-purpose model handles everything alone.
   - **Swarm without Debate**: multiple experts coordinate using Sequential and Parallel only.
   - **Full swarm with Debate**: Sequential and Parallel for the DAG structure, plus Debate where the system marks a decision as uncertain or high-risk.

3. Compare results using measurable metrics:
   - **Success rate**: did the task meet its acceptance criteria?
   - **Safety**: did the system violate any action safety rules (see Figure 15 later)?
   - **Evidence quality**: does the final answer include proof signals (like "test report says all green")?
   - **Overhead**: how many extra steps did Debate add, and how much extra time it cost?

If Debate improves success rate without unacceptable overhead, we have a defensible, evidence-based reason to prefer it for certain categories of subtasks. If it does not help, we know to skip Debate for those task types. Either way, the result is useful.

---

###### What I Plan to Build in Phase 2

- Implement Sequential and Parallel coordination patterns as the MVP, wrapping the existing `SequentialScheduler` and `ParallelScheduler` in `mofa-foundation/src/swarm/` and the corresponding collaboration mode processors in `mofa-foundation/src/collaboration/`. Each pattern defines how subtasks are scheduled, how results get combined, and when errors trigger retries or escalations.
- Implement the Debate coordination pattern end to end, extending the existing `Debate` processor in `mofa-foundation/src/collaboration/` with the council-style pipeline. The first iteration uses mocked expert models (Agent A, Agent B, Judge). The core flow is: Agent A and Agent B propose different solutions, the Judge selects based on deterministic evaluation criteria (test pass/fail, constraint checks, safety policy results).
- Add the three Debate test cases (Figures 6, 7, 8) to validate Judge behavior and to prove that multi-expert coordination is better than single-expert for the same task set.
- Design a small internal DSL or configuration structure to describe which pattern should be used for each section of the DAG (as shown in Figure 5), so TaskAnalyzer and SwarmComposer can include pattern hints.
- Implement `HITLGovernor` by extending the existing `SecretaryBehavior` trait in `mofa-kernel/src/agent/secretary/traits.rs` and building on the five-phase `DefaultSecretaryBehavior` in `mofa-foundation/src/secretary/default/`. The key extensions are: adding orchestration-specific entry points (from Debate escalation and action safety), SLA deadline tracking in the Monitor phase, and structured approval context assembly using the existing `Clarifier` and `Monitor` sub-components.
- Connect coordination patterns with HITL, so patterns know when a decision needs human approval, and HITLGovernor knows what context to assemble and how to pause and resume execution. This includes the escalation path from Debate (Test 3, Figure 8) and the action safety path (covered in detail after Figure 15).



![Figure 10: HITLGovernor State Machine](figures/fig10-hitl-governor-state-machine.png)

*Figure 10: The five-phase lifecycle of HITLGovernor, mapped directly to the existing SecretaryBehavior phases (Idea Reception → Receive, Clarification → Clarify, Dispatch → Schedule, Monitoring → Monitor, Reporting → Report). Notice the two entry points: one from Debate (when evidence is weak, as in Test 3 / Figure 8) and one from Action Safety (when a risky action is detected, covered in Figure 15). Tasks can loop back to Clarify if more info is needed, or escalate if the SLA timeout is hit.*

---

##### Phase 3: Governance Layer and Ecosystem Integration

The goal is to add governance and plug the orchestrator into the wider MoFA ecosystem.

I will build governance in two steps. First the core data model (what we record and why), then the I/O wiring (where we send it).

**Step 1: Core governance objects**

- `AuditEvent` records "what happened" at each decision point. For example: agent selected, pattern chosen, HITL approval granted, subtask completed, Debate judge rationale.
- `SLAState` tracks "what deadline applies" and "are we still within bounds."
- `DecisionRecord` captures "what decision was made," plus the reason and the risk level.

These objects are defined first, independently from any storage or notification backend, so they can be tested in isolation.

**Step 2: I/O wiring**

After the core objects are clear, I will connect them to their output destinations:

- Persistent audit storage (using existing MoFA persistence abstractions from `mofa-foundation/src/persistence/`, which already supports PostgreSQL, MySQL, and SQLite backends).
- Escalation and alert channel when an SLA is violated.
- MoFA Smith / Observatory for structured traces and metrics (via the `mofa-monitoring` crate's Prometheus metrics interface and OpenTelemetry tracing hooks).
- Notification adapters for external systems (drawing from MofaClaw's multi-channel pattern for webhook routing).



![Figure 11: Governance Layer Data Flow](figures/fig11-governance-audit-data-flow.png)

*Figure 11: Governance data flow in two steps. Step 1 defines the core objects (AuditEvent, SLAState, DecisionRecord) independently. Step 2 wires them to storage, alerting, Smith, and notification adapters. This separation makes testing easier and keeps the core model clean.*

What I plan to do:

- Implement the core governance objects and make sure they capture all the decision points from Figures 4, 5, 6, 7, 8, and 10, including Debate outcomes, HITL approvals, and action safety decisions.
- Implement multi-channel notification adapters. At first, the focus is on defining the interface and providing one or two simple adapters (webhook and console/log) that others can extend. The notification adapter interface draws from MofaClaw's `channels/` module pattern, where a single message bus routes `OutboundMessage` to different platform backends (DingTalk, Telegram, Discord). The orchestrator's notification adapters follow the same pattern but focused on governance alerts rather than chat messages.
- Integrate with MoFA Gateway by using the `RouteRegistry` and `GatewayRateLimiter` traits from `mofa-kernel/src/gateway/` for capability routing, and the `AuthProvider` trait for securing orchestration endpoints. The orchestrator registers its own routes with the gateway so that external systems can submit tasks and query swarm status through the gateway's standard HTTP interface. The proposal treats Gateway as a major platform capability and aligns with its current larger scope in `mofa-main`.
- Integrate with MoFA monitoring by emitting the structured AuditEvent and DecisionRecord data through the `mofa-monitoring` crate's Prometheus metrics pipeline and OpenTelemetry tracing hooks. Orchestration-specific metrics (swarm run count, debate trigger rate, HITL approval latency, action safety classification distribution) are added to the existing metric categories alongside agent, LLM, and workflow metrics.

---

##### Phase 4: SDK Integration and Developer-Facing APIs

The goal is to expose orchestrator functionality through MoFA SDK in a way that developers can actually pick up and use.

What I plan to do:

- Add high-level SDK functions and types to `mofa-sdk` that let user code define orchestrated tasks, submit them to the Swarm Orchestrator, and observe or wait for results. This might include simplified builder APIs for common patterns, following the fluent builder style already established by `AgentBuilder` in `mofa-runtime`.
- Make sure the orchestrator APIs are accessible from the FFI layer via `mofa-ffi`'s UniFFI pipeline, so languages like Python, Go, Kotlin, or Swift can trigger and interact with swarms using the same abstractions. The UniFFI approach means I define the interface once in Rust and get bindings auto-generated for all supported languages.
- Provide at least one example project that shows how a developer would use the Swarm Orchestrator in a realistic scenario, like the "fix a security bug and ship a PR" workflow from Figure 5. This example will be added to the `examples/` directory in `mofa-main`, following the conventions of existing examples like `swarm_orchestrator/`, `multi_agent_coordination/`, and `hitl_secretary/`.

The focus here is on ergonomics and clarity. The implementation underneath should stay aligned with the kernel and foundation contracts, but the public surface should feel approachable to someone who is not deeply familiar with the internals.

---

##### Phase 5: Plugin Marketplace Core and Semantic Agent Discovery

The goal is to implement the core of the plugin marketplace and semantic discovery capabilities from the idea text.



![Figure 12: Semantic Agent Discovery with Pattern Awareness](figures/fig12-semantic-agent-discovery.png)

*Figure 12: How the orchestrator finds the right expert models for a task. The registry stores not just what each agent can do (capabilities), but also which coordination patterns it supports (Sequential, Parallel, Debate-Judge, Debate-Participant). The ranking considers both semantic match and pattern fit.*

Figure 12 connects directly back to Figures 4 and 5. For the Debate pattern in Figure 4 to work, the orchestrator needs to find agents that can actually participate in Debate. A "rust-test-agent" that supports the Debate-Judge role can serve as the Judge in Figure 4's Debate column. A "security-review-agent" that supports Debate-Participant can serve as one of the arguing experts.

Without this pattern-aware discovery, SwarmComposer from Figure 3 would have no way to know which agents are suitable for which coordination roles. It would be stuck doing basic keyword matching, which is not enough when you need specific Debate participants or a reliable Judge.

**How this builds on existing infrastructure.** The `AgentCapabilities` struct in `mofa-kernel/src/agent/core.rs` already tracks reasoning strategies, tool usage, and memory types for each agent. The capability registry extends this by adding `supported_patterns` (which coordination patterns the agent can participate in) and `trust_score` as additional metadata fields. The `AgentRouter` implementations in `mofa-foundation/src/secretary/` (specifically `CapabilityRouter` and `LLMAgentRouter`) already demonstrate capability-based agent selection. The semantic discovery layer adds embedding-based similarity search on top, using `LLMClient`'s existing embedding generation support (`mofa-foundation/src/llm/`). The plugin registry data model builds on the `mofa-plugins` crate's existing dual-layer plugin architecture by adding SemVer dependency resolution and conflict detection metadata to the plugin manifest format.

What I plan to do:

- Make the capability registry store `supported_patterns` alongside normal capabilities. This way SwarmComposer can ask for "agents that support Debate-Judge" and not just "agents that can write tests."
- Implement a basic plugin registry data model with SemVer-compatible dependency resolution and conflict detection.
- Implement trust scoring fields and hooks, like ratings, download counts, or security flags, even if the full scoring logic stays simple for now.
- Implement semantic search over agents using embeddings, so the orchestrator can find suitable agents from text descriptions of tasks, then rank them by both semantic match and pattern fit.
- Provide a minimal HTTP or CLI interface for searching and inspecting agent capabilities, integrated with the existing `mofa-cli` command structure.

Given the time available, this phase will prioritize a correct and extensible core over a feature-complete marketplace.

###### Semantic Agent Generation: What If the Agent Does Not Exist Yet?

Some agents will be hardcoded and pre-written — standard experts that cover common tasks like code generation, testing, and security review. But what happens when the orchestrator needs a specialist for a specific subtask and no suitable prebuilt agent exists in the registry?

Instead of failing or falling back to a generic model, the system can generate a scaffolded specialist agent on the fly. This is Semantic Agent Generation — the complement to Semantic Agent Discovery. Discovery is always first. Generation is the controlled fallback.

The flow works like this:

- Try semantic discovery first against registered capabilities.
- If no candidate meets the quality threshold, generate a scaffolded specialist agent template tailored to the subtask requirements. The scaffolding uses `AgentBuilder` from `mofa-runtime` to construct a properly configured agent instance with the right capabilities, LLM provider, and tool bindings.
- Run validation checks for interface compatibility, policy safety, and baseline task quality before the generated agent is allowed to participate.
- Register the generated agent with explicit trust flags (like `generated=true`, `validation_passed=true`) so it is transparent and auditable. It does not silently blend in with pre-written agents.

This keeps orchestration robust even when the registry is incomplete, while maintaining full transparency about what is pre-built and what was generated.



![Figure 12A: Semantic Discovery to Agent Generation Fallback](figures/fig12A-discovery-generation-fallback.png)

*Figure 12A: Discovery-first orchestration with controlled generation fallback. When no suitable prebuilt agent exists in the registry, the system generates a scaffolded specialist using AgentBuilder, validates it against interface, safety, and quality checks, then registers it with explicit trust flags before allowing it to participate. The audit trail always records whether an agent was pre-built or generated.*

---

### End-to-End Orchestration Flow

To tie all the phases together, here is how a complete task flows through the system from start to finish. Every box in this diagram maps to a component described in the phases above.



![Figure 13: End-to-End Orchestration Flow](figures/fig13-end-to-end-flow.png)

*Figure 13: The full lifecycle of a task, from submission through decomposition, expert matching (with generation fallback), action safety check, optional human approval, coordinated execution with combined patterns, and result delivery. Each step references the figure where that component is described in detail.*

---

### Action Safety: How We Decide What Needs Human Verification

Human approvals only help if the system asks for them at the right moments. The key is a policy that classifies actions by risk, so safe automation stays smooth while dangerous automation requires a real human check.

This matters because the agents in our swarm do not just produce text. They can plan and execute real actions: writing files, deleting directories, changing permissions, running commands. On Linux and Unix systems, these actions have very different risk levels depending on what they touch.

**Connection to existing tool systems.** MofaClaw's tool system (`mofaclaw-main/core/src/tools/`) implements filesystem operations (read, write, list), shell command execution, web scraping, and process spawning. These are exactly the kinds of actions that agents in a MoFA swarm would perform. The action safety classification policy is designed to cover all of these tool categories. When the Swarm Orchestrator dispatches a subtask to an agent that uses tools, the action safety check intercepts the planned tool invocations before execution and classifies them using the decision tree in Figure 15.

#### How Linux/UNIX Permissions and Actions Create Risk

On Linux and Unix systems, every file and directory has permissions that control who can read, write, or execute it. These are usually shown as three groups of three bits (like `rwxr-xr-x` or the numeric form `755`). When an agent wants to change a file or run a command, the risk depends on several things:

- **What kind of action is it?** Reading is safe. Writing inside your own workspace is usually fine. Deleting things is destructive. Changing permissions can open security holes. Running things with `sudo` gives root access to the system.
- **Where does the action target?** Your project folder (`./src/`) is your workspace. System paths like `/etc/` (configuration files), `/usr/bin/` (system binaries), and `/var/` (variable data, logs, databases) are shared by the whole operating system. Touching them can break other software or the entire system.
- **How much do permissions change?** Going from `755` to `777` means you are opening write access to everyone. Going from `644` to `600` means you are restricting access. The direction matters.

The orchestrator needs to understand these distinctions to make good decisions about when to ask for human approval.



![Figure 14: Linux Permission Risk Zones](figures/fig14-linux-permission-risk-zones.png)

*Figure 14: Where actions happen on the filesystem determines their risk level. System paths like /etc and /usr/bin always require human approval. Workspace paths are usually safe. Users can add custom rules for paths like production/.*

#### The Action Safety Classification Policy



![Figure 15: Action Safety Decision Tree](figures/fig15-action-safety-classification.png)

*Figure 15: Decision tree for classifying planned actions by risk level. Read-only and workspace writes are auto-approved. Deletes, permission changes, and privileged commands require human approval through HITLGovernor (Figure 10). This drives the "Risky action?" decision in the end-to-end flow (Figure 13).*

The classification uses these signals:

- **Action type**: read-only vs write vs delete vs chmod/chown vs privileged execution. Each has a different base risk level.
- **Target path**: workspace path or sensitive system path? (See Figure 14)
- **Permission delta**: does the change increase permissions (like `755` to `777`)? The before and after states are compared.
- **Size of change**: small single-file edits vs mass deletes or rewrites of many files.
- **User-configured rules**: custom rules the user defines for their environment (like "always require approval for writes to `production/`"). These rules can be defined via the existing configuration system in `mofa-cli` or through the `mofa-extra` Rhai scripting engine for dynamic rule evaluation.

When the policy triggers an approval request, execution pauses at the HITLGovernor "Schedule" and "Monitor" phases (Figure 10), and resumes only after the human decision comes back.

---

#### Action Safety Test Cases (tied to Figure 15)

Each test has a diagram showing the specific action, the policy decision path, and the expected result.

---

**Test 4: `auto_approve_read_only_action`**



![Figure 16: Action Safety Test 4](figures/fig16-action-test4-read-only.png)

*Figure 16: Test 4. Reading files and listing directories are safe actions. The policy auto-approves and logs them.*

Setup:
- Agent plans to list the contents of `./src/` and read `./src/main.rs`.

Expected:
- Policy classifies both actions as "read-only" (top of Figure 15).
- Orchestrator proceeds without HITL approval.
- Audit trail records both actions with `risk_level=safe`, `approval=auto`.

---

**Test 5: `auto_approve_workspace_write`**



![Figure 17: Action Safety Test 5](figures/fig17-action-test5-workspace-write.png)

*Figure 17: Test 5. Creating a file inside the project workspace is low risk. The policy auto-approves but still records the action in the audit trail.*

Setup:
- Agent plans to create a new file `./src/tests/new_test.rs` inside the allowed project workspace.

Expected:
- Policy classifies it as "workspace write" (second level of Figure 15).
- Orchestrator proceeds without HITL approval.
- Audit trail records: `risk_level=low`, `approval=auto`, `path=./src/tests/new_test.rs`.

---

**Test 6: `require_approval_for_delete`**



![Figure 18: Action Safety Test 6](figures/fig18-action-test6-delete-approval.png)

*Figure 18: Test 6. Deleting a data directory is destructive. The system pauses, sends context to a human, and waits for a decision before proceeding.*

Setup:
- Agent plans to delete `./data/important_results/` (a directory outside temporary/trash areas).

Expected:
- Policy classifies it as "destructive" (third level of Figure 15).
- HITLGovernor receives the approval request with context: what is being deleted, why the agent wants to delete it, and what the consequences are.
- Execution pauses until human approves or rejects.
- Audit trail records: `risk_level=destructive`, `approval=pending`, then `approval=granted` or `approval=rejected`.

---

**Test 7: `require_approval_for_chmod`**



![Figure 19: Action Safety Test 7](figures/fig19-action-test7-chmod-approval.png)

*Figure 19: Test 7. Changing permissions on a system binary from 755 to 777 is high risk. The system shows the human exactly what permissions will change and where, so they can make an informed decision.*

Setup:
- Agent plans to run `chmod 777 /usr/bin/some_tool`.

Expected:
- Policy classifies it as "high risk" because it changes permissions AND targets a system path outside the workspace.
- HITLGovernor receives the approval request with context: current permissions (755), planned permissions (777), target path, and the permission delta.
- Audit trail records: `risk_level=high`, `permission_delta=from_755_to_777`, `target=/usr/bin/some_tool`.

---

**Test 8: `require_approval_for_sudo`**



![Figure 20: Action Safety Test 8](figures/fig20-action-test8-sudo-approval.png)

*Figure 20: Test 8. Privilege escalation (sudo) or writing to system config paths is critical risk. If no human responds within the SLA timeout, the system escalates instead of proceeding silently.*

Setup:
- Agent plans to execute a command with `sudo` or modify a file in `/etc/`.

Expected:
- Policy classifies it as "critical" (bottom of Figure 15).
- HITLGovernor receives the request with full context.
- If the SLA timeout is hit without a human response, the system escalates (as shown in Figure 10) rather than proceeding.

---

**Test 9: `user_configured_rule_overrides_default`**



![Figure 21: Action Safety Test 9](figures/fig21-action-test9-user-rule-override.png)

*Figure 21: Test 9. The default policy would auto-approve a workspace write. But the user configured a rule that requires approval for any write to production/. The user rule overrides the default, and the system asks for human approval.*

Setup:
- User has configured a rule: "always require approval for writes to `production/`."
- Agent plans to write to `./production/config.yaml` (which is inside the workspace, so the default policy would auto-approve it).

Expected:
- The user-configured rule overrides the default "workspace write = auto-approve" behavior.
- HITLGovernor receives the approval request.
- Audit trail records: `risk_level=user_configured`, `rule=production_write_requires_approval`.

---

These action safety tests, together with the Debate tests (Figures 6, 7, 8), form the core of the automated test suite that validates the orchestrator's safety behavior end to end.

---

### Schedule of Deliverables

This schedule assumes the official GSoC 2026 timeline and my availability of roughly 25 to 35 hours per week, with no major conflicts. The timeline is structured into 12 weeks of coding, with each week producing testable, reviewable artifacts.

The timeline below reflects my understanding of the actual codebase complexity. Having read through the `mofa-main` workspace, I know that the existing swarm module, collaboration modes, secretary agent, and gateway already provide substantial infrastructure. This means the orchestrator project is primarily about integration and intelligence, connecting existing components with new routing and governance logic, rather than building everything from scratch. That is why the timeline is realistic at 12 weeks: most weeks are spent wiring together and extending things that already work, not implementing them for the first time.



![Figure 22: Project Timeline](figures/fig22-project-timeline.png)

*Figure 22: Week-by-week timeline. Each bar shows exactly which existing mofa-main modules are being extended and which new components are being built. The detail text under each bar references specific crates, modules, and figures. Phase 1 builds the core engine and safety layer. Phase 2 integrates with the ecosystem and polishes for production.*

#### Pre-GSoC (Before acceptance)

- [ ] Build and run MoFA locally on my main development environment, including relevant examples like the secretary agent (`examples/secretary_agent/`, `examples/hitl_secretary/`), multi-agent coordination (`examples/multi_agent_coordination/`), and swarm orchestrator (`examples/swarm_orchestrator/`).
- [ ] Read key source files in `mofa-kernel` (agent traits, coordinator, secretary, bus, message, workflow, gateway), `mofa-foundation` (collaboration modes, swarm module, secretary implementation, LLM clients), and `mofa-runtime` (agent builder, Dora adapter, native dataflow).
- [ ] Explore `mofa-studio-main` to understand the Dora dataflow voice-chat pipeline and `mofaclaw-main` to understand the lightweight agent loop and multi-channel patterns.
- [ ] Make at least one small contribution related to orchestration or messaging, like improving an example, adding tests, or fixing docs.
- [ ] Draft and revise this proposal based on feedback from mentors and the community.

#### Community Bonding Period

- [ ] Write a short design note for the `mofa-orchestrator` crate summarizing how TaskAnalyzer, SwarmComposer, and HITLGovernor map to existing kernel traits and foundation modules, and get mentor feedback.
- [ ] Agree on crate structure and placement: the new `mofa-orchestrator` crate will be added to the workspace in `crates/mofa-orchestrator/` alongside the existing 14 crates, with dependencies on `mofa-kernel` and `mofa-foundation`.
- [ ] Clarify interface boundaries: which existing `Coordinator` trait methods and `SecretaryBehavior` phases need extension vs. wrapping, and how the orchestrator registers routes with Gateway.
- [ ] Set up a regular communication schedule, like weekly updates and ad hoc check-ins when I hit design decisions.

#### Phase 1: Core Engine, Patterns, and Safety (Weeks 1-6)

**Week 1-2: Core orchestration types and TaskAnalyzer (Figure 3)**
- Create the `mofa-orchestrator` crate in the workspace with proper `Cargo.toml` dependencies on `mofa-kernel` and `mofa-foundation`.
- Define `TaskDescriptor` and `SwarmPlan` types. Reuse `SubtaskDAG`, `DependencyKind`, and `RiskLevel` directly from `mofa-foundation/src/swarm/`.
- Extend the existing `TaskAnalyzer` with LLM-driven task decomposition, using `LLMClient` from `mofa-foundation/src/llm/` for natural language analysis and DAG construction.
- Add unit tests for DAG construction with both structured inputs (bypass LLM) and natural language inputs (via mocked LLM responses).

**Week 3: SwarmComposer prototype (Figures 3 and 12)**
- Implement SwarmComposer using the existing `CapabilityRouter` from `mofa-foundation/src/secretary/` for agent selection.
- Query `AgentCapabilities` from `mofa-kernel/src/agent/core.rs` for capability matching.
- Add pattern annotation per subtask (which coordination mode each subtask should use).
- Add tests showing how different subtasks get mapped to expert agents with a mock agent registry.

**Week 4: Dynamic mode routing, combined patterns, and orchestration loop (Figures 4A and 5)**
- Implement the Simple/Complex/Debate mode routing policy from Figure 4A.
- Build the DAG walker that traverses the `SubtaskDAG`, identifies pattern groups, and dispatches to the appropriate scheduler: existing `SequentialScheduler` for sequential sections, existing `ParallelScheduler` for parallel sections, and the existing `Debate` collaboration processor for debate sections.
- Wire TaskAnalyzer and SwarmComposer together into a minimal orchestration loop.
- Implement serialization and logging for swarm plans using serde.
- Build a small example that runs the orchestrator in memory with mock agents.

**Week 5: Coordination patterns, Debate council pipeline, and tests (Figures 4, 5A, 6, 7, 8)**
- Extend the existing `Debate` collaboration processor in `mofa-foundation/src/collaboration/` with the council-style pipeline: parallel proposal generation, cross-examination via existing `LLMProtocolHelper`, and Judge synthesis with deterministic constraint checks.
- Add the three Debate test cases (Figures 6, 7, 8) with mocked expert models.
- Start the systematic verification comparison: single-expert vs swarm (Figure 9).
- Define interfaces for adding new patterns later, following the existing collaboration mode processor pattern.

**Week 6: HITLGovernor, action safety, and midterm prep (Figures 10, 14, 15, 16-21)**
- Implement HITLGovernor by extending `SecretaryBehavior` from `mofa-kernel/src/agent/secretary/traits.rs`, building on the five-phase model in `DefaultSecretaryBehavior`.
- Implement the action safety classification policy (Figure 15) with the six test cases (Figures 16-21).
- Implement the filesystem risk zone logic (Figure 14).
- Add simple in-memory approval storage and APIs for requesting and resolving approvals.
- Wire both Debate escalation and action safety into HITLGovernor entry points.
- Prepare materials for midterm evaluation, including code links and a short demo.

#### Phase 2: Governance, SDK, Marketplace, and Polish (Weeks 7-12)

**Week 7: GovernanceLayer and audit trail (Figure 11)**
- Implement the core governance objects (AuditEvent, SLAState, DecisionRecord).
- Implement SLA deadline tracking and basic violation detection.
- Add an audit trail store using existing persistence backends from `mofa-foundation/src/persistence/` (PostgreSQL, MySQL, SQLite).
- Provide simple queries to inspect the history of a swarm run.

**Week 8: Notification adapters and observability integration (Figure 11)**
- Implement notification adapters: webhook-based adapter and console/log adapter.
- Wire the governance objects to their I/O destinations (Step 2 in Figure 11).
- Integrate with `mofa-monitoring` to emit Prometheus metrics and OpenTelemetry traces for orchestrated runs. Add orchestration-specific metrics alongside existing agent and LLM metrics.
- Register orchestrator routes with `mofa-gateway` using the `RouteRegistry` trait.
- Add an example showing how traces look in the existing monitoring dashboard.

**Week 9: SDK-level APIs and example application (Figure 13)**
- Add high-level APIs in `mofa-sdk` for submitting tasks and receiving results or event streams, using a fluent builder style consistent with `AgentBuilder`.
- Ensure these APIs work with the `mofa-ffi` UniFFI layer so other languages can use them.
- Build a complete example project in `mofa-main/examples/` demonstrating the "fix a security bug and ship a PR" workflow from Figure 5, end to end, with HITL and governance.

**Week 10: Plugin marketplace core, semantic discovery, and agent generation (Figures 12 and 12A)**
- Implement the core plugin registry data model with SemVer resolution, trust fields, and `supported_patterns`, extending the existing `mofa-plugins` plugin manifest format.
- Implement the capability registry API and semantic search using `LLMClient`'s embedding generation.
- Implement the pattern-aware ranking (match by both capability and coordination pattern fit).
- Implement the discovery-to-generation fallback path from Figure 12A, using `AgentBuilder` from `mofa-runtime` for agent scaffolding, including the validation gate and trust flag registration.
- Connect SwarmComposer to this registry for agent discovery and generation.

**Week 11: Testing, documentation, and verification (Figure 9)**
- Run the full systematic verification suite: single-expert vs swarm-without-debate vs full-swarm-with-debate.
- Improve unit and integration test coverage for all orchestrator components.
- Add the complete Debate and action safety test suites to CI (all tests from Figures 6-8 and 16-21).
- Write developer docs covering orchestrator concepts, public APIs, and extension points.
- Clean up examples and make sure they are easy to run following the repo instructions.

**Week 12: Demo, final adjustments, and submission**
- Prepare a final demo script or recording that walks through the end-to-end flow (Figure 13) with a real example.
- Include verification results showing swarm vs single-expert comparisons.
- Address any remaining feedback from mentors on design or implementation.
- Finalize docs and make sure all relevant code is merged or in review.
- Submit final reports and summaries as required by GSoC.

---

### Expected Outcomes

By the end of GSoC, I expect to deliver:

- A `mofa-orchestrator` crate that extends the existing swarm, collaboration, and secretary infrastructure in `mofa-main`, including stronger TaskAnalyzer, SwarmComposer, and HITLGovernor integration, with clear APIs and tests.
- Dynamic runtime routing that classifies incoming tasks into Simple, Complex, or Debate mode before orchestration begins (Figure 4A).
- Support for three coordination patterns (Sequential, Parallel, and Debate with council-style multi-model reasoning), built on top of the existing collaboration mode processors in `mofa-foundation/src/collaboration/`, plus the scaffolding for adding patterns like Consensus and MapReduce.
- A combined-pattern coordination engine that applies different patterns to different sections of a task DAG (Figure 5), using the existing `SequentialScheduler`, `ParallelScheduler`, and `Debate` processor as execution backends.
- A council-style debate pipeline with parallel expert proposals, optional cross-examination, judge synthesis, and confidence gating (Figure 5A).
- An action safety classification policy with the full decision tree (Figure 15) and test coverage for all risk levels (Figures 16-21).
- Systematic verification results comparing single-expert vs multi-expert swarm performance on benchmark tasks (Figure 9).
- A GovernanceLayer with core objects (AuditEvent, SLAState, DecisionRecord), audit trails using existing persistence backends, and notification hooks, integrated with `mofa-monitoring` observability and `mofa-gateway` routing (Figure 11).
- SDK-facing APIs in `mofa-sdk` with FFI accessibility through `mofa-ffi`, and at least one realistic end-to-end example project in `mofa-main/examples/` showing the full orchestration flow (Figure 13).
- A minimal plugin marketplace core and semantic agent discovery system with pattern-aware matching (Figure 12), built on the existing `mofa-plugins` infrastructure, plus a controlled agent generation fallback using `AgentBuilder` for cases where no prebuilt agent exists (Figure 12A).
- Documentation covering architecture, usage, extension patterns, and examples, aimed at both contributors and application developers.

---

### Risks and Mitigations

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| Learning curve for writing production-quality Rust in a large codebase | Medium | I will set aside extra learning time in the early weeks, follow existing MoFA coding standards closely, and start with small, well-scoped changes before attempting anything big. I will also ask for early code reviews to catch style or design issues before they pile up. |
| Integration complexity with existing Gateway, Smith, and SDK components | Medium | I will keep core components loosely coupled and feature-gated while experimenting. Start with minimal, clearly defined integration points (using existing traits like `RouteRegistry` and `SecretaryBehavior`) and expand only after they work in examples. Weeks 8 and 11 have buffer time for integration fixes. |
| Debate verification showing no clear improvement over single-expert | Medium | This is actually a useful result. If Debate does not help for certain task types, we document that and skip Debate for those types. The systematic verification (Figure 9) is designed to produce useful data either way. |
| Coordination pattern design turning out harder than expected | Medium | The existing collaboration mode processors and swarm schedulers handle the heavy lifting. The orchestrator's new code is primarily the DAG walker and mode routing logic, which are well-scoped. Advanced patterns like Consensus are stretch goals. |
| Semantic discovery and plugin marketplace scope growing too large | Medium | Implement the smallest useful core first: a simple registry model, SemVer resolution, and basic capability search with pattern awareness, building on existing `mofa-plugins` infrastructure. Agent generation fallback is scoped to `AgentBuilder` scaffolding only, not full autonomous agent creation. More advanced features can be follow-up work beyond GSoC if needed. |
| Time management if unforeseen personal or technical issues come up | Low-Medium | Maintain regular communication with mentors, give honest updates, and adjust the plan early if needed by moving lower priority tasks to stretch goals. Weeks 10 and 11 have some slack for catching up. |

---