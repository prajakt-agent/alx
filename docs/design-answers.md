# ALX Design Answers: Agent Identity, Configuration & Deployment

## 1. Agent SOPs, Tools, Knowledge Base & Memory

### Recommended Architecture: One Hermes Profile Per Agent

Each specialist agent should be a **separate Hermes profile**. Hermes profiles are purpose-built for exactly this — each profile gets its own:
- `SOUL.md` — the agent's personality, role definition, and behavioral instructions (this IS the SOP)
- `config.yaml` — model, provider, toolsets, delegation settings
- `.env` — API keys scoped to this agent
- `skills/` — task-specific procedures (the knowledge base + SOPs)
- `memories/` — persistent context (`MEMORY.md`, `USER.md`)

```bash
# Create specialist profiles
hermes profile create alx-teamlead --clone
hermes profile create alx-research
hermes profile create alx-coder
hermes profile create alx-reviewer
hermes profile create alx-writer
hermes profile create alx-devops
```

### SOPs as Skills

Each agent's SOPs should be Hermes skills stored in its profile's `skills/` directory:

```
~/.hermes/profiles/alx-research/
├── SOUL.md                    # "You are ALX Research Agent. You produce structured research reports with citations..."
├── config.yaml                # model: google/gemini-flash-2.0, provider: openrouter
├── .env                       # AGENTMAIL_API_KEY, any agent-specific keys
├── skills/
│   ├── web-research/SKILL.md  # SOP: how to do web research (search → extract → synthesize → cite)
│   ├── arxiv-review/SKILL.md  # SOP: how to review academic papers
│   └── market-analysis/SKILL.md  # SOP: competitive intelligence procedure
└── memories/
    ├── MEMORY.md              # Accumulated context (auto-managed by Hermes)
    └── USER.md                # Knowledge about the user/team
```

### Knowledge Base Strategy

| Knowledge Type | Storage | Mechanism |
|---|---|---|
| Role definition & behavior | `SOUL.md` | Loaded into system prompt every turn |
| Task procedures (SOPs) | `skills/*.md` | Auto-loaded when task matches skill triggers |
| Persistent context | `memories/MEMORY.md` | Auto-updated, survives sessions |
| User/team knowledge | `memories/USER.md` | Facts about the user, preferences |
| Project-specific context | `CLAUDE.md` in repo | Claude Code loads per-project |
| Shared team knowledge | Git repo (alx/docs/) | Referenced by skills, version-controlled |

### Memory Design

Each profile has **isolated memory** — this is critical. The Team Lead should NOT share memory with specialists (Hermes docs explicitly warn against two agents writing to the same profile).

For **shared context** between agents, use:
1. **Explicit context passing** via `delegate_task(context=...)` — the Team Lead passes relevant context to each specialist per-task
2. **Structured outputs** — specialists return JSON/markdown that the Team Lead aggregates
3. **Git as shared state** — code, docs, and research artifacts live in repos
4. **Linear as shared state** — task status, comments, and decisions live in Linear issues

```yaml
# Example: Team Lead's delegation with context
delegate_task:
  goal: "Research competitor pricing for product X"
  context: |
    Project: ALX competitive analysis
    Previous findings: [summary from memory]
    Output format: JSON with sources
    Deadline: 2 hours
```

### Tool Allocation Per Agent

```yaml
# alx-research config.yaml
tools:
  enabled: [web_search, web_extract, browser, read_file, write_file]
  
# alx-coder config.yaml  
tools:
  enabled: [terminal, read_file, write_file, search_files, browser]
  # Claude Code is invoked via terminal

# alx-reviewer config.yaml
tools:
  enabled: [terminal, read_file, search_files, browser]  # read-heavy, no write

# alx-writer config.yaml
tools:
  enabled: [read_file, write_file, web_search, web_extract]
```

---

## 2. Dynamic Agent Configuration & Registry

### Agent Registry File

Create a central YAML registry at `alx/agents/registry.yaml`:

```yaml
# ALX Agent Registry
# Add/remove agents by editing this file. The Team Lead reads it at task time.

version: 1
team: alx

agents:
  teamlead:
    profile: alx-teamlead
    role: orchestrator
    model: anthropic/claude-sonnet-4
    provider: anthropic
    description: "Receives tasks, decomposes, delegates, synthesizes results"
    capabilities: [planning, delegation, synthesis, communication]
    email: alx-teamlead@agentmail.to
    active: true

  research:
    profile: alx-research
    role: specialist
    model: google/gemini-flash-2.0
    provider: openrouter
    description: "Web research, academic papers, market analysis"
    capabilities: [web_search, arxiv, document_analysis, competitive_intel]
    email: alx-research@agentmail.to
    active: true
    
  coder:
    profile: alx-coder
    role: specialist
    model: anthropic/claude-sonnet-4  # via Claude Code
    provider: anthropic
    description: "Feature implementation, bug fixes, refactoring via Claude Code"
    capabilities: [coding, git, testing, debugging]
    email: alx-coder@agentmail.to
    active: true

  reviewer:
    profile: alx-reviewer
    role: specialist
    model: anthropic/claude-sonnet-4
    provider: anthropic
    description: "Code review, security scanning, test validation"
    capabilities: [code_review, security_scan, test_validation]
    email: alx-reviewer@agentmail.to
    active: false  # Phase 2

  writer:
    profile: alx-writer
    role: specialist
    model: google/gemini-flash-2.0
    provider: openrouter
    description: "Documentation, emails, reports, content"
    capabilities: [documentation, email_drafting, report_writing]
    email: alx-writer@agentmail.to
    active: false  # Phase 2

  devops:
    profile: alx-devops
    role: specialist
    model: google/gemini-flash-2.0
    provider: openrouter
    description: "CI/CD, infrastructure, deployment, monitoring"
    capabilities: [ci_cd, docker, cloud_infra, monitoring]
    email: alx-devops@agentmail.to
    active: false  # Phase 2
```

### Dynamic Agent Management

The Team Lead reads this registry at task time to discover available specialists. Adding/removing agents:

```bash
# Add a new agent
# 1. Create the Hermes profile
hermes profile create alx-designer --clone-from alx-research

# 2. Configure it
alx-designer setup  # set model, API keys
echo "You are ALX Designer Agent..." > ~/.hermes/profiles/alx-designer/SOUL.md

# 3. Create AgentMail inbox
# (via API — see Section 4)

# 4. Add to registry.yaml
# Edit alx/agents/registry.yaml, add the designer entry

# 5. Team Lead picks it up on next task — no restart needed
```

### How the Team Lead Uses the Registry

The Team Lead's SOUL.md or a skill should instruct it:

```markdown
## Task Routing
Before delegating, read `alx/agents/registry.yaml` to discover active specialists.
Match task requirements to agent capabilities. Only delegate to agents marked `active: true`.
If no specialist matches, handle the task yourself or ask the user.
```

### Comparison with Other Frameworks

| Framework | Agent Registry Approach |
|---|---|
| **CrewAI** | Python class definitions with `role`, `goal`, `backstory`, `tools` per agent |
| **Google ADK** | Agent hierarchy defined in Python with `sub_agents=[]` |
| **LangGraph** | Graph nodes, each node is an agent with its own state |
| **ALX (proposed)** | YAML registry + Hermes profiles. More declarative, no code changes needed |

The YAML approach is better for ALX because:
- Non-technical users can add agents by editing YAML
- No code deployment needed to change the team composition
- Each agent's full config lives in its Hermes profile, not in code
- Version-controlled via git

---

## 3. GitHub Identity Strategy

### Recommendation: Single GitHub Account (prajakt-agent) + Branch Conventions

**Do NOT create separate GitHub accounts per agent.** Here's why:

| Approach | Pros | Cons |
|---|---|---|
| **Per-agent GitHub accounts** | Clear attribution | GitHub ToS (one account per person/bot), PAT management overhead, org seat costs, complex auth |
| **Single shared account** | Simple auth, one PAT, one org seat, `gh` just works | Less granular attribution |
| **Single account + git trailers** | Simple auth AND clear attribution | Slightly more convention to enforce |

### Implementation: Git Trailers for Attribution

Use the single `prajakt-agent` GitHub account but add `Co-authored-by` or custom trailers to identify which agent made the commit:

```bash
# In each agent's profile config
git config user.name "ALX Coder Agent"
git config user.email "alx-coder@agentmail.to"

# Or use git trailers in commit messages
git commit -m "feat: add auth module

Authored-by: alx-coder
Delegated-by: alx-teamlead
Linear-issue: ALX-42"
```

### Branch Strategy

Each agent works on its own branch to prevent conflicts:

```
main
├── alx/coder/ALX-42-auth-module      # Coder working on issue 42
├── alx/coder/ALX-43-fix-api          # Coder working on issue 43
├── alx/research/ALX-44-market-scan   # Research output committed
└── alx/writer/ALX-45-docs-update     # Writer updating docs
```

Convention: `alx/<agent-name>/<linear-issue>-<description>`

### Auth Setup

```bash
# One-time: authenticate prajakt-agent with gh CLI
gh auth login

# Each profile shares the same gh auth (default Hermes behavior — profiles share HOME)
# So alx-coder, alx-reviewer etc. all use prajakt-agent's GitHub token

# If you NEED separate git identities per profile:
# Set terminal.home_mode: profile in each agent's config.yaml
# Then run gh auth login separately in each profile
```

### PR Workflow

The **Team Lead** should handle GitHub integration centrally:
1. **Coder Agent**: writes code, commits to branch, pushes
2. **Team Lead**: opens the PR (adds context from the Linear issue)
3. **Reviewer Agent**: reviews the PR via `gh pr review`
4. **Team Lead**: merges after approval, updates Linear

This keeps the PR lifecycle coherent — one entity owns the PR thread.

---

## 4. Email Identity Strategy (Per-Agent AgentMail Inboxes)

### Recommendation: YES — One AgentMail Inbox Per Agent

Each agent should have its own `@agentmail.to` inbox. AgentMail's free tier supports 3 inboxes; paid tiers support more.

### Why Per-Agent Emails

1. **Clear routing**: Emails to `alx-research@agentmail.to` go directly to the Research Agent
2. **Audit trail**: Which agent sent what is unambiguous
3. **External identity**: When agents email external services, each has its own identity
4. **Service signups**: Each agent can have its own accounts at external services

### Implementation via AgentMail API

```python
from agentmail import AgentMail
import os

client = AgentMail(api_key=os.environ['AGENTMAIL_API_KEY'])

# Create inboxes for each agent
agents = ['alx-teamlead', 'alx-research', 'alx-coder', 'alx-reviewer', 'alx-writer', 'alx-devops']

for agent in agents:
    inbox = client.inboxes.create(
        address=f"{agent}@agentmail.to",
        display_name=f"ALX {agent.split('-')[1].title()} Agent"
    )
    print(f"Created: {inbox.address}")
```

### Email Routing Architecture

```
External Email → alx-teamlead@agentmail.to → Team Lead processes
                 alx-research@agentmail.to  → Research Agent directly
                 alx-coder@agentmail.to     → Coder Agent directly

Internal: Team Lead → alx-coder@agentmail.to (task assignment email)
          Coder Agent → alx-teamlead@agentmail.to (completion report)
          Team Lead → prajakt-hermes@agentmail.to (summary to user)
```

### Shared API Key, Separate Inboxes

All agents use the same `AGENTMAIL_API_KEY` (organization-level). Each agent's `.env` sets its own inbox:

```bash
# ~/.hermes/profiles/alx-research/.env
AGENTMAIL_API_KEY=am_us_...          # same org key
AGENTMAIL_INBOX=alx-research@agentmail.to  # this agent's inbox
```

### Free Tier Limitation

AgentMail free tier = 3 inboxes, 3,000 emails/month. For Phase 1:
- `prajakt-hermes@agentmail.to` (existing, user-facing)
- `alx-teamlead@agentmail.to` (orchestrator)
- `alx-notifications@agentmail.to` (shared notification inbox)

Scale to per-agent inboxes when upgrading to paid tier.

---

## 5. Deployment Architecture

### Phase 1: Always-On Mac (Current Machine)

The simplest deployment — run everything on Prajakt's Mac:

```bash
# Install each agent's gateway as a managed service
alx-teamlead gateway install
alx-teamlead gateway start

# Or use multiplexed gateway (recommended — single process serves all profiles)
# The default Hermes gateway multiplexes all profiles automatically
hermes gateway start  # serves default + all alx-* profiles
```

**Hermes multiplexed gateway** is ideal here:
- Single process serves all agent profiles
- Each profile keeps its own `.env`, config, memory, skills
- Managed by launchd on macOS — auto-restarts on crash
- Dashboard gives unified view of all agents

```
┌─────────────────────────────────────────────┐
│          Mac (always-on)                     │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │   Hermes Multiplexed Gateway         │   │
│  │                                      │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐   │   │
│  │  │TeamLead│ │Research│ │ Coder  │   │   │
│  │  │Profile │ │Profile │ │Profile │   │   │
│  │  └────────┘ └────────┘ └────────┘   │   │
│  │                                      │   │
│  │  WhatsApp ← → Linear ← → AgentMail  │   │
│  └──────────────────────────────────────┘   │
│                                              │
│  Claude Code (invoked by Coder Agent)        │
│  gh CLI (shared auth via prajakt-agent)      │
└─────────────────────────────────────────────┘
```

**Pros**: Zero infra cost, all tools available locally, `gh` and Claude Code just work.  
**Cons**: Depends on Mac being on, no redundancy.

### Phase 2: Cloud VPS (Dedicated Server)

When reliability matters, move to a cloud VPS:

```bash
# On a Linux VPS (Ubuntu 24.04)
# 1. Install Hermes
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash

# 2. Export profiles from Mac
hermes profile export alx-teamlead > alx-teamlead.tar.gz
hermes profile export alx-research > alx-research.tar.gz
# ... for each profile

# 3. Import on VPS
hermes profile import alx-teamlead.tar.gz
hermes profile import alx-research.tar.gz

# 4. Configure .env on VPS (API keys don't travel in exports)
# 5. Install gateway as systemd service
hermes gateway install --system
hermes gateway start
```

**Recommended VPS specs**: 4 CPU, 8GB RAM, 50GB SSD ($20-40/month on Hetzner/DigitalOcean).

### Phase 3: Docker Compose (Portable)

For reproducible deployments:

```yaml
# docker-compose.yaml
version: '3.8'
services:
  alx-gateway:
    image: ghcr.io/nousresearch/hermes-agent:latest
    command: hermes gateway run
    volumes:
      - ./profiles/alx-teamlead:/hermes/profiles/alx-teamlead
      - ./profiles/alx-research:/hermes/profiles/alx-research
      - ./profiles/alx-coder:/hermes/profiles/alx-coder
    env_file:
      - .env  # shared secrets
    restart: unless-stopped
    ports:
      - "3000:3000"  # dashboard
```

### Phase 4: Hermes Profile as Git Package

Each agent profile can be packaged as a git repo and installed anywhere:

```bash
# Package the Research Agent profile
cd ~/.hermes/profiles/alx-research
git init && git add SOUL.md config.yaml skills/
git commit -m "ALX Research Agent v1"
git remote add origin https://github.com/prajakt-agent/alx-research-agent
git push

# Install on any machine
hermes profile install github.com/prajakt-agent/alx-research-agent --alias alx-research
```

This is the most portable option — anyone on the team can spin up the same agent.

### Deployment Decision Matrix

| Factor | Mac (Phase 1) | VPS (Phase 2) | Docker (Phase 3) |
|---|---|---|---|
| Setup time | Minutes | Hours | Hours |
| Monthly cost | $0 | $20-40 | $20-40 |
| Reliability | Medium | High | High |
| Tool access | Full (Claude Code, gh, etc.) | Full | Needs volume mounts |
| Portability | None | Manual | High |
| Scalability | Limited | Vertical | Horizontal |

### Recommendation

**Start with Phase 1 (Mac)** — it's already working. Move to **Phase 2 (VPS)** when you need 24/7 reliability. Skip Docker unless you need to distribute the setup to others or run in CI.

The multiplexed gateway is the key enabler — it means you don't need to manage N processes for N agents. One gateway process, all profiles served.

---

## Summary: Recommended Stack

| Concern | Decision |
|---|---|
| Agent identity | One Hermes profile per agent |
| SOPs | Skills in each profile's `skills/` directory |
| Knowledge base | SOUL.md + skills + memories per profile |
| Memory | Isolated per profile; shared context via delegate_task |
| Agent registry | `alx/agents/registry.yaml` (YAML, version-controlled) |
| Dynamic agents | `hermes profile create` + registry entry |
| GitHub | Single `prajakt-agent` account + branch conventions + git trailers |
| Email | Per-agent AgentMail inboxes (free tier: shared notification inbox) |
| Deployment Phase 1 | Mac + multiplexed Hermes gateway |
| Deployment Phase 2 | Cloud VPS + systemd |
| Deployment Phase 3 | Profile-as-git-repo for portability |
