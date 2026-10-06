# ALX Architecture: AI Team Orchestration

## 1. Overview

ALX is an AI team orchestration system where a **Team Lead** (orchestrator agent) receives tasks, decomposes them, and delegates work to **Specialist Agents**. Each specialist is optimised for a specific type of work.

```
User (WhatsApp / Linear / Email)
    ↓
┌──────────────────────────────────┐
│         TEAM LEAD                │  ← Frontier model (Claude Opus/Sonnet)
│    (Hermes Orchestrator)         │
│                                  │
│  • Receives tasks from Linear    │
│  • Decomposes into subtasks      │
│  • Routes to specialists         │
│  • Synthesises results           │
│  • Reports back to user          │
└──────────┬───────────────────────┘
           │
    ┌──────┼──────────┬──────────────┐
    │      │          │              │
┌───▼──┐ ┌─▼────┐ ┌──▼─────┐ ┌─────▼────┐
│Research│ │Coder │ │Reviewer│ │  Writer  │  ← Cheaper models (Gemini Flash)
│Agent  │ │Agent │ │ Agent  │ │  Agent   │
└───┬───┘ └──┬───┘ └───┬────┘ └────┬─────┘
    │        │         │           │
  Tools    Tools     Tools       Tools
```

## 2. Design Decisions

### 2.1 Why Hermes-Native Delegation

| Criterion | Decision | Rationale |
|-----------|----------|-----------|
| Framework | Hermes `delegate_task` | Already installed, proven at 1,393-agent scale |
| Orchestrator model | Frontier (Claude Opus/Sonnet) | Planning quality is critical |
| Worker model | Gemini Flash / GPT-4o-mini | Workers burn volume, cost matters |
| Task tracking | Linear API | Already integrated, ALX team exists |
| Notifications | AgentMail + WhatsApp | Already integrated |
| Code execution | Claude Code via Hermes | Already integrated |
| Future interop | A2A protocol | Industry standard, 150+ orgs adopted |

### 2.2 Alternatives Considered

| Framework | Verdict | Reason |
|-----------|---------|--------|
| **LangGraph** | Deferred | Excellent but steep learning curve, not needed for Phase 1 |
| **CrewAI** | Possible Phase 2 | Good for role-based research crews, can be invoked from Hermes |
| **Google ADK** | Possible Phase 3 | Best A2A/MCP support, useful for cross-framework interop |
| **OpenAI Agents SDK** | Rejected | OpenAI-centric, less flexible |
| **AutoGen** | Rejected | Maintenance mode, use AG2 or MS Agent Framework instead |

## 3. Specialist Agents

### 3.1 Research Agent
- **Purpose:** Web research, academic paper review, market analysis, competitive intelligence
- **Tools:** Web search, web extract, arXiv, document analysis
- **Model:** Gemini Flash (high volume, low cost)
- **Output:** Structured research reports with citations

### 3.2 Coder Agent
- **Purpose:** Feature implementation, bug fixes, refactoring
- **Tools:** Claude Code CLI, terminal, file operations, GitHub
- **Model:** Claude Sonnet (via Claude Code)
- **Output:** Working code on feature branches with PRs

### 3.3 Reviewer Agent
- **Purpose:** Code review, security scanning, test validation
- **Tools:** Linters, test runners, security scanners, GitHub PR review
- **Model:** Claude Sonnet
- **Output:** Review comments, approval/rejection with rationale

### 3.4 Writer Agent
- **Purpose:** Documentation, emails, reports, content
- **Tools:** File operations, web research, AgentMail
- **Model:** Gemini Flash
- **Output:** Polished documents, drafted communications

### 3.5 DevOps Agent (Phase 2)
- **Purpose:** CI/CD, infrastructure, deployment, monitoring
- **Tools:** Terminal, Docker, cloud CLIs
- **Model:** Gemini Flash
- **Output:** Infrastructure changes, deployment reports

## 4. Task Flow

```
1. INTAKE
   User creates Linear issue → assigned to Team Lead
   OR emails prajakt-hermes@agentmail.to
   OR messages via WhatsApp

2. PLANNING (Team Lead)
   ├─ Parse task requirements
   ├─ Decompose into subtasks
   ├─ Identify required specialists
   └─ Create execution plan

3. DELEGATION (Team Lead → Specialists)
   ├─ delegate_task(goal, context) for each subtask
   ├─ Parallel execution for independent subtasks
   ├─ Sequential for dependent subtasks
   └─ Live steering via list/steer/stop

4. SYNTHESIS (Team Lead)
   ├─ Collect specialist outputs
   ├─ Validate completeness
   ├─ Synthesise into final deliverable
   └─ Quality check

5. DELIVERY (Team Lead)
   ├─ Update Linear issue (status, comments, PR links)
   ├─ Open PR if code changes
   ├─ Email summary via AgentMail
   └─ Notify user via WhatsApp
```

## 5. Configuration

```yaml
# ~/.hermes/config.yaml (proposed)
delegation:
  max_concurrent_children: 10
  max_iterations: 250
  max_spawn_depth: 2
  model: "google/gemini-flash-2.0"     # default worker model
  provider: "openrouter"
```

## 6. Protocol Stack

```
Layer     Protocol   Purpose
───────────────────────────────────────
Tool      MCP        Agent ↔ Tool server (databases, APIs, files)
Agent     A2A        Agent ↔ Agent (cross-framework, future)
Orchestr. Hermes     Team Lead ↔ Specialists (delegate_task)
Task      Linear     User ↔ Team Lead (issue tracking)
Comms     AgentMail  Team ↔ External (email notifications)
Chat      WhatsApp   User ↔ Team Lead (real-time messaging)
```

## 7. Phased Rollout

### Phase 1: Foundation (Current)
- [x] Architecture document
- [ ] Team Lead receives Linear tasks automatically
- [ ] Research Agent as first specialist
- [ ] Coder Agent via Claude Code delegation
- [ ] End-to-end test: Linear issue → PR → summary email

### Phase 2: Expansion
- [ ] Reviewer Agent for automated code review
- [ ] Writer Agent for documentation
- [ ] DevOps Agent for CI/CD
- [ ] CrewAI integration for complex research workflows
- [ ] Automatic Linear issue decomposition into subtasks

### Phase 3: Scale & Interop
- [ ] A2A protocol support for cross-framework agents
- [ ] Google ADK integration for GCP-native tasks
- [ ] Nested orchestration (specialist spawns sub-specialists)
- [ ] Cost tracking and optimisation per task type
- [ ] Performance metrics and SLA monitoring

## 8. Key Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Hallucination cascading through agents | JSON-schema output validation, human review gate |
| Runaway costs from parallel agents | Cost caps per delegation, cheap worker models |
| Context loss between orchestrator and specialists | Explicit context passing, structured outputs |
| Task deadlocks (agents waiting on each other) | Timeouts, max_iterations limits, live monitoring |
| Security (agent with terminal access) | Approval modes, branch protection, PR-based workflow |

## 9. References

- [Research Document](./docs/research.md) — Full framework comparison
- [Hermes Delegation Docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- [A2A Protocol](https://a2a-protocol.org/latest/specification/)
- [MCP Protocol](https://modelcontextprotocol.io/specification)
