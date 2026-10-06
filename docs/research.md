# ALX-1 Research: Multi-Agent AI Orchestration — State of the Art (October 2026)

## Executive Summary

The multi-agent AI landscape has converged on a clear protocol stack in 2026:
- **MCP** (Model Context Protocol) for agent-to-tool connections
- **A2A** (Agent-to-Agent Protocol) for inter-agent communication
- **Orchestration frameworks** (LangGraph, CrewAI, Google ADK, OpenAI Agents SDK, Microsoft Agent Framework) for building the actual agent systems

For ALX-1 (building an AI Team with a Team Lead orchestrator), the most practical approaches are:
1. **Hermes Agent's built-in delegation** — already available, battle-tested at scale (1,393 subagents in one run), supports parallel work, steering, and nested orchestration
2. **Google ADK** — best native A2A + MCP support, agent tree model maps perfectly to Team Lead → specialists
3. **CrewAI** — fastest prototyping with role-based model, intuitive for team metaphors
4. **LangGraph** — most production-grade control for complex stateful workflows

---

## 1. Orchestration Frameworks

### 1.1 LangGraph (LangChain team)

**Architecture:** Directed graph with typed state. Nodes = agents/functions, edges = transitions (including conditional). State flows through the entire graph.

**Key Features:**
- Built-in checkpointing — pause, resume, replay from any saved state
- Human-in-the-loop at any graph node
- Durable execution — survives crashes
- Best-in-class observability via LangSmith
- Python and TypeScript SDKs
- v1.2.11 (Aug 2026), production-stable since v1.0 (Oct 2025)

**Multi-Agent Patterns (current recommended approach):**
- `langgraph-supervisor` is **unmaintained** — migrate to subagents pattern
- Agents wrapped as tools, supervisor routes via tool calls
- Five patterns: Subagents, Handoffs, Skills, Router, Custom workflow
- `create_agent()` returns a compiled graph; every agent is a LangGraph subgraph

**Pros:**
- Maximum control and flexibility
- Lowest latency and token consumption in benchmarks
- Production-proven (Klarna, Uber, LinkedIn, Replit)
- Sophisticated state management and checkpointing
- Model-agnostic

**Cons:**
- Steepest learning curve (1-2 weeks to production competency)
- Verbose — ~120 lines for a simple ReAct agent vs ~40 in CrewAI
- LangChain dependency tree adds complexity
- Supervisors leak quality when paraphrasing subagent output

**Best For:** Complex stateful workflows, production systems requiring maximum control, long-running durable agents.

---

### 1.2 CrewAI (CrewAI Inc.)

**Architecture:** Role-based teams. Agents defined by role/goal/backstory. Two orchestration layers: **Crews** (autonomous agent teams) and **Flows** (event-driven deterministic workflows).

**Key Features:**
- ~51,000 GitHub stars, v1.14.4 (April 2026)
- Sequential and Hierarchical process modes
- Built-in memory (short-term, long-term, entity)
- Structured outputs via Pydantic schemas
- Flows with `@start`, `@listen`, `@router` decorators
- LLM-agnostic via LiteLLM (100+ providers)
- Native A2A support (added Jan 2026)

**Pros:**
- Fastest time to working prototype (~20 lines of Python, ~35 minutes)
- Role abstraction reads like product requirements — stakeholders can understand it
- Crews + Flows dual-layer is a genuine production pattern
- Built-in memory without external dependencies
- No LangChain dependency (standalone)

**Cons:**
- Higher token consumption than LangGraph
- Hierarchical manager doesn't truly delegate selectively (executes all tasks sequentially regardless)
- One agent's hallucination compounds through the pipeline
- Debugging is harder than graph-based approaches
- Non-deterministic execution makes SLAs difficult

**Best For:** Role-based task delegation, content/research workflows, fastest prototyping, teams that think in job titles.

---

### 1.3 Google ADK (Agent Development Kit)

**Architecture:** Agent tree (hierarchical). Root agent receives input, delegates to sub-agents which can delegate further. Leaf agents execute tools.

**Key Features:**
- 1.0 GA across Python, Go, Java, TypeScript (April 2026)
- 18.7k GitHub stars
- Four agent types: `LlmAgent`, `SequentialAgent`, `ParallelAgent`, `LoopAgent`
- **Native A2A protocol support** — expose any ADK agent as an A2A server
- **Native MCP support** — first-class tool connectivity
- Model-agnostic (optimized for Gemini, supports Claude, GPT, Llama via LiteLLM)
- Deploy to Vertex AI Agent Engine, Cloud Run, GKE, or self-host

**Pros:**
- Best native A2A + MCP integration of any framework
- Agent tree model maps cleanly to Team Lead → Specialists
- Orchestration agents (Sequential/Parallel/Loop) don't call LLMs — pure routing logic, saves costs
- Four-language SDK parity
- Session state sharing via `output_key` — agents write results to shared state
- `adk web` dev UI for local testing

**Cons:**
- Youngest framework (~1 year old)
- Optimized for GCP — less compelling outside Google Cloud
- Strict file/folder conventions
- Single built-in tool per agent limit
- Weak unit testing support for sub-agents

**Best For:** GCP-native teams, cross-framework interoperability via A2A, hierarchical team structures.

---

### 1.4 OpenAI Agents SDK (successor to Swarm)

**Architecture:** Handoff-first design. Agents hand off conversation control to each other via explicit handoff primitives.

**Key Features:**
- v0.17.0 (May 2026), 26k GitHub stars
- Handoffs as first-class primitives with full trace visibility
- Built-in tracing in OpenAI dashboard (no instrumentation needed)
- Guardrails for input/output validation
- Human-in-the-loop via interrupts
- Voice support via Realtime API
- MCP tool integration
- Computer use (browser automation)

**Pros:**
- Cleanest handoff abstraction — explicit agent transitions
- Best tracing/observability out of the box
- Hosted tools (web search, code interpreter, file search) with zero setup
- Minimal framework overhead — "gets out of the way"
- Production-maintained by OpenAI

**Cons:**
- OpenAI-model-centric (others via LiteLLM bridge)
- No built-in multi-agent supervisor pattern
- Less flexible for complex routing than LangGraph
- Vendor lock-in to OpenAI ecosystem

**Best For:** Teams committed to OpenAI models, clean handoff workflows, minimal framework overhead.

---

### 1.5 Microsoft Agent Framework (successor to AutoGen)

**Architecture:** Data-flow based Workflows. Executors (agents, functions, sub-workflows) connected by typed edges.

**Key Features:**
- 1.0 shipped April 2026
- Merges AutoGen's agent abstractions + Semantic Kernel's enterprise features
- Workflow with request/response pauses and checkpointing
- Built-in middleware, hosted tools, typed workflows
- Multi-language (Python, .NET, Java)

**Pros:**
- Enterprise-grade (session management, type safety, telemetry)
- Built-in checkpointing and resumable workflows
- Request/response for human-in-the-loop (not available in AutoGen)
- Data-flow model more composable than AutoGen's control-flow

**Cons:**
- Very new (April 2026) — limited production track record
- Migration from AutoGen requires rethinking architecture
- Azure-centric integrations
- Community still migrating from AutoGen/AG2

**Best For:** Microsoft/Azure shops, enterprise environments needing type safety and middleware.

---

### 1.6 AutoGen / AG2

**Status:** AutoGen is in **maintenance mode** (security fixes only). Two successors:
- **Microsoft Agent Framework** — Microsoft's official successor
- **AG2** — Community-driven fork by original AutoGen creators

**AG2 Key Features:**
- v1.0 with protocol-driven framework
- Network-based orchestration (Hub + typed Channels)
- Channel types: conversation, consulting, discussion, workflow
- Native A2A and MCP support
- Visual Studio for drag-and-drop agent wiring

**Note:** For new projects, choose Microsoft Agent Framework (Azure) or AG2 (vendor-neutral). Do not start new work on AutoGen classic.

---

## 2. Communication Protocols

### 2.1 A2A (Agent-to-Agent Protocol)

**What it is:** Open protocol by Google (April 2025), now Linux Foundation-governed. v1.0 reached March 2026. 150+ organizations adopted.

**Purpose:** Standardizes how agents from different frameworks discover, authenticate, and delegate tasks to each other.

**Key Concepts:**
| Concept | Description |
|---------|-------------|
| **Agent Card** | JSON at `/.well-known/agent.json` — declares identity, skills, endpoint, auth |
| **Task** | Unit of work with lifecycle: submitted → working → input-required → completed/failed/canceled/rejected |
| **Message** | Communication turn with role + Parts (text, data, file, URL) |
| **Artifact** | Output container attached to completed tasks |

**Task Lifecycle States:**
```
submitted → working → completed
                   → failed
                   → input-required → (client sends input) → working
                   → auth-required
                   → canceled
                   → rejected
```

**Delivery Modes:**
1. Synchronous request/response (JSON-RPC)
2. Server-Sent Events (SSE) for streaming progress
3. Push notifications via webhooks for fire-and-forget

**Transport:** HTTP/JSON-RPC 2.0 (primary), gRPC (optional), HTTP+REST

**SDKs:** Python, Go, JavaScript, Java, .NET, Rust

**Framework Support:** Google ADK (native), CrewAI (native), LangGraph (integration), AG2 (native)

**When to use A2A:**
- Agents from different frameworks need to interoperate
- Long-running tasks with streaming progress
- Agents need to request clarification mid-task
- Standardized discovery and authentication across vendors

### 2.2 MCP (Model Context Protocol)

**What it is:** Open standard by Anthropic (Nov 2024), donated to Linux Foundation (Dec 2025). Standardizes how agents connect to tools and data sources.

**Purpose:** Agent-to-tool integration (not agent-to-agent — that's A2A's role).

**Key Concepts:**
- **Hosts:** LLM applications that initiate connections
- **Clients:** Connectors within the host application
- **Servers:** Services that provide context and capabilities

**Server Capabilities:**
- **Resources:** Context and data for the agent
- **Prompts:** Templated messages and workflows
- **Tools:** Functions for the agent to execute

**Transport:** JSON-RPC 2.0 over stdio or HTTP SSE

**Ecosystem:** Supported by Claude, ChatGPT, VS Code, Cursor, LangChain, Google ADK, CrewAI, and hundreds of servers.

### 2.3 MCP + A2A: The Protocol Stack

```
Layer     Protocol   Connects              Use case
─────────────────────────────────────────────────────
Tool      MCP        Tool server → Agent    Agent calls database, API, file system
Agent     A2A        Agent → Agent          Orchestrator delegates to specialist
```

A production multi-agent system typically uses both:
```
User request
    ↓
Orchestrator (Team Lead)
├─ MCP → Database server (data lookup)
├─ A2A → Code Review agent (specialist)
└─ A2A → Research agent (specialist)
    └─ MCP → Web search server
```

---

## 3. Task Routing & Delegation Patterns

### 3.1 Three Universal Patterns

Every framework implements variants of these:

| Pattern | Description | Best For |
|---------|-------------|----------|
| **Supervisor** | Central orchestrator decomposes, delegates, synthesizes | Most production systems — auditable, easy to add specialists |
| **Sequential Pipeline** | Fixed sequence: research → outline → write → review | Content generation, document processing |
| **Hierarchical** | Supervisors delegate to mid-level supervisors to specialists | Very large workflows where one supervisor can't fit the full graph |

A fourth pattern — **peer-to-peer conversational** (AutoGen-style) — exists but is rare in production due to hard-to-audit reasoning traces and termination guarantees.

### 3.2 Routing Mechanisms Compared

| Framework | Routing Mechanism |
|-----------|-------------------|
| **LangGraph** | Explicit — developer defines graph edges and conditional routing |
| **CrewAI** | Emergent — LLM decides via tool calls at runtime (`delegate_work`, `ask_question`) |
| **Google ADK** | Hierarchical — parent agent routes to sub-agents via descriptions |
| **OpenAI Agents SDK** | Handoff — agent explicitly returns another agent |
| **Hermes** | Tool-based — parent calls `delegate_task` with goal/context |

### 3.3 Key Design Decisions for Team Lead Architecture

1. **Context isolation vs sharing:** Hermes and A2A favor isolation (child knows nothing except what parent passes). CrewAI shares context automatically between tasks.
2. **Deterministic vs dynamic routing:** LangGraph pre-defines routes; CrewAI lets the LLM decide at runtime.
3. **Cost model:** Frontier model for orchestrator (planning), cheaper model for workers (execution). This is the proven pattern.
4. **Concurrency:** Parallel execution for independent tasks, sequential for dependent ones.
5. **Human-in-the-loop:** Every framework supports it differently — A2A's `input-required` state, LangGraph's checkpoints, Hermes's `/steer`.

---

## 4. How Existing Tools Handle Delegation

### 4.1 Hermes Agent (Nous Research)

**The most battle-tested delegation system for our use case.** Hermes already runs on Prajakt's setup.

**Architecture:**
- `delegate_task` tool spawns isolated child AIAgent instances
- Each child gets fresh conversation, inherited tools, own terminal session
- Only final summary enters parent's context
- Up to 10 concurrent subagents (v0.21 default), configurable

**Key Capabilities (v0.21.0 "Pantheon", Aug 2026):**
- **Live orchestration:** list, steer, stop running children
- **Nested delegation:** `role="orchestrator"` children can spawn their own workers (depth configurable)
- **Async delegation:** `delegate_task_async` for non-blocking spawn-and-poll model
- **Cost-optimized:** frontier model for parent, cheap model for children via `delegation.model`
- **JSON-schema validation** on child outputs
- **Per-delegation cost reporting**

**Scale Record:** 1,393 subagents in a single 19-hour run, 218 concurrent agents peak, ~$19,300 cost for work estimated at $150K-$1.8M manually.

**Delegation to Other CLIs:**
- Claude Code via `claude --print` (one-shot) or interactive mode
- Codex CLI, Gemini CLI, OpenCode via generalized ACP support

**Configuration:**
```yaml
delegation:
  max_concurrent_children: 10
  max_iterations: 250
  max_spawn_depth: 2
  model: "google/gemini-flash-2.0"    # cheap workers
  provider: "openrouter"
```

### 4.2 Claude Code (Anthropic)

- Managed Agents API for multi-instance orchestration (Anthropic-only)
- Supports being called by Hermes via `claude --print`
- Built-in MCP support for tool integration
- Sub-agent delegation via tool use

### 4.3 Codex CLI (OpenAI)

- Can be invoked as a subagent by Hermes
- Single-agent focused, not designed for orchestration
- Good for code-specific tasks as a specialist worker

---

## 5. Real-World Architectures for AI Teams

### 5.1 The Canonical "Team Lead" Architecture

```
                    ┌──────────────┐
                    │  Team Lead   │ ← Frontier model (planning, routing)
                    │ (Orchestrator)│
                    └──────┬───────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
    ┌───────▼──────┐ ┌────▼─────┐ ┌──────▼──────┐
    │  Research     │ │  Coder   │ │  Reviewer   │ ← Cheaper models (execution)
    │  Specialist  │ │Specialist│ │ Specialist  │
    └──────────────┘ └──────────┘ └─────────────┘
         │                │              │
    ┌────▼────┐     ┌────▼────┐    ┌────▼────┐
    │MCP:Search│     │MCP:Git  │    │MCP:Lint │   ← Tools via MCP
    │MCP:Arxiv │     │MCP:Files│    │MCP:Tests│
    └─────────┘     └─────────┘    └─────────┘
```

### 5.2 Production Pattern: Hermes-Based AI Team

Given Prajakt's existing setup (Hermes + Claude Code + Linear + AgentMail + WhatsApp):

```
User (via WhatsApp/Desktop)
    ↓
Hermes Team Lead (orchestrator, frontier model)
    ├─ delegate_task → Research Agent (web search, arxiv)
    ├─ delegate_task → Coder Agent (Claude Code via delegate_task)
    ├─ delegate_task → Review Agent (lint, test, security scan)
    ├─ MCP → Linear (create/update issues)
    ├─ MCP → AgentMail (notifications, team comms)
    └─ send_message → WhatsApp (status updates to user)
```

### 5.3 Production Pattern: ADK + A2A Cross-Framework

```
ADK Orchestrator (Team Lead)
    ├─ A2A → CrewAI Research Crew (external service)
    ├─ A2A → LangGraph Code Pipeline (external service)
    ├─ MCP → Database server
    └─ MCP → GitHub server
```

### 5.4 Nous Research's 1,393-Agent Run Architecture

Real-world proof at scale:
- One `/goal` set the standard, not individual instructions
- Orchestrator split codebase into 36 non-overlapping groups
- One git worktree per worker (no conflicts)
- Interface parity as hard constraint (JSON schemas byte-identical)
- 218 concurrent agents peak, 110 average
- 478 delegation batches, 111,352 tool calls total
- 19 hours, ~$19,300 (vs $150K-$1.8M estimated manual cost)

---

## 6. Framework Selection Matrix for ALX-1

| Criterion | LangGraph | CrewAI | Google ADK | Hermes Delegation | OpenAI SDK |
|-----------|-----------|--------|------------|-------------------|------------|
| **Team Lead model fit** | Good (supervisor pattern) | Excellent (role-based) | Excellent (agent tree) | **Best** (built-in) | Good (handoffs) |
| **Already in stack** | No | No | No | **Yes** | No |
| **Time to prototype** | 2 weeks | 3 days | 1 week | **Already working** | 1 week |
| **Production readiness** | Excellent | Good | Good | **Excellent** (proven at 1,393 agents) | Good |
| **Cross-framework** | Via integration | A2A native | A2A native | ACP + delegate to any CLI | No |
| **Cost optimization** | Manual | Manual | Auto (routing agents skip LLM) | **Built-in** (delegation.model) | Manual |
| **Live orchestration** | Via checkpoints | Limited | Via event bus | **List/steer/stop** | Via interrupts |
| **Model flexibility** | Any via LangChain | Any via LiteLLM | Any via LiteLLM | Any (configurable per child) | OpenAI-first |

---

## 7. Recommendations for ALX-1

### Primary Recommendation: Hermes-Native Approach

**Why:** Hermes Agent is already installed and configured with all of Prajakt's integrations (Linear, AgentMail, WhatsApp, Claude Code). Its delegation system is the most mature and battle-tested for exactly the Team Lead → Specialist pattern.

**Implementation plan:**
1. Define specialist agent profiles with different models and toolsets
2. Create an orchestrator that receives Linear tasks and delegates to specialists
3. Use `delegate_task` with `role="orchestrator"` for complex multi-step work
4. Configure `delegation.model` for cost-optimized worker agents
5. Use Linear MCP for task tracking, AgentMail for notifications
6. Add A2A/ACP support for future cross-framework interoperability

### Secondary Recommendation: Hybrid Hermes + CrewAI/ADK

**Why:** For specific workflows that benefit from CrewAI's role-based crews or ADK's native A2A support, Hermes can invoke them as external services.

**Implementation:**
- Hermes as the top-level orchestrator
- CrewAI crews for content/research pipelines (invoked via subprocess or A2A)
- Google ADK agents for any GCP-native work
- Claude Code as a specialist for coding tasks (already integrated)

### Architecture Decision: Start Simple, Add Complexity

1. **Phase 1:** Hermes delegation with 3-5 specialist agents (Research, Coder, Reviewer, Writer, DevOps)
2. **Phase 2:** Add structured task routing from Linear issues to appropriate specialists
3. **Phase 3:** Add A2A support for cross-framework agents if needed
4. **Phase 4:** Scale to nested orchestration for complex multi-step workflows

---

## 8. Key Technical Decisions

| Decision | Recommendation | Rationale |
|----------|---------------|-----------|
| **Orchestrator model** | Frontier (Claude Opus/Sonnet) | Planning quality is critical |
| **Worker model** | Gemini Flash / GPT-4o-mini | Workers burn volume, cost matters |
| **Communication** | Hermes delegate_task | Already available, proven at scale |
| **Task tracking** | Linear MCP | Already integrated |
| **Notifications** | AgentMail + WhatsApp | Already integrated |
| **Code execution** | Claude Code via Hermes | Already integrated |
| **Future interop** | A2A protocol | Industry standard, 150+ orgs |

---

## Sources

- Hermes Agent docs: https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation
- A2A Protocol spec: https://a2a-protocol.org/latest/specification/
- MCP spec: https://modelcontextprotocol.io/specification
- Google ADK: https://adk.dev/
- CrewAI: https://github.com/crewAIInc/crewAI
- LangGraph: https://www.langchain.com/langgraph
- OpenAI Agents SDK: https://github.com/openai/openai-agents-python
- Microsoft Agent Framework: https://learn.microsoft.com/en-us/agent-framework/
- AG2: https://github.com/ag2ai/ag2
- Nous Research 1,393-agent run postmortem (Sep 2026)
