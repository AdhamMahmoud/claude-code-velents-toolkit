# VelentsAI Claude Agent Toolkit

> One command. Full autonomous pipeline. From requirement to reviewed PR.

---

## What is this?

A **Claude Code plugin** — 33 specialized AI agents, 22 domain knowledge skills, and 21 slash commands built specifically for the VelentsAI platform.

**Architecture**: Agents are short, direct, specialized instructions (~50 lines). All code patterns live in 3 skills loaded automatically — `velents-backend`, `velents-frontend`, `velents-testing` — extracted from the real VelentsAI codebase. Agents tell Claude *what to do*; skills show it *how the code looks*.

Every agent already knows:
- Multi-tenant architecture (stancl/tenancy v3, database-per-tenant isolation)
- 4 core business flows: text conversation, voice call, agent builder, batch campaign
- Real Laravel 12 + Next.js 16 patterns extracted from the actual codebase
- All 46 shadcn/ui components, OKLCH color system, RTL Arabic support
- Voice pipelines (Normal/Fast/Flash), ElevenLabs, LiveKit, CallGateway
- Payment/credit system, Spatie RBAC, WebSocket (Laravel Reverb)
- Official docs for every library — fetched before writing code, not guessed

**Stack**: Laravel 12 · PHP 8.4 · Next.js 16 · React 19 · TypeScript 5.9 · PostgreSQL · Redis · Tailwind CSS 4 · stancl/tenancy v3

---

## Getting Started

### 1 — Install the Plugin

```bash
# Register the marketplace (one time)
claude plugins add git https://github.com/Velents-Technologies-UG/claude-code-velents.git

# Install
claude plugin install claude-code-velents
```

Restart Claude Code after installing.

### 2 — Connect Jira

```bash
claude mcp add --transport http atlassian https://mcp.atlassian.com/v1/mcp
```

### 3 — Open the VelentsAI Repo

```bash
cd /path/to/velentsAgents   # Laravel backend
# or
cd /path/to/agent-hub       # Next.js frontend
```

### 4 — Use It

```bash
/velents VEL-123                            # implement a Jira ticket
/velents fix the crash in conversations API  # bug fix → fast path
/velents build the agent monitoring page     # new feature → full pipeline
/review                                      # review code you just wrote
```

---

## Who Uses What

| Role | Commands | When |
|------|----------|------|
| **Any Dev (quick)** | `/velents fix ...` · `/velents add ...` · `/velents why ...` | Bug fixes, small changes, questions |
| **Any Dev (feature)** | `/velents build ...` · `/velents implement VEL-123` | New features → full pipeline |
| **Backend Dev** | `/backend` · `/api` · `/db` | Direct to specific layer |
| **Frontend Dev** | `/frontend` · `/ui` | Direct to specific layer |
| **QA / Lead** | `/review` · `/review-pr` · `/risk` · `/test` | Before merging PRs |
| **PM** | `/pm-discover` · `/pm-strategy` · `/pm-execute` | Discovery, PRDs, OKRs |

---

## How It Works

The orchestrator classifies every request as **FAST** or **FULL** before doing anything:

**FAST** — invokes `fullstack-developer` directly (no pipeline overhead):
- Bug fixes, small additions, questions, changes to existing code, ≤3 files

**FULL** — runs the autonomous spec-kit pipeline:
- New features, new database tables, multi-layer work, Jira tickets

### Full Pipeline (14 steps, fully autonomous)

```
Phase 0: Context Loading
  ├─ Classify scope (backend / frontend / full-stack / integration)
  ├─ Load architecture + feature impact map
  ├─ Fetch llms.txt docs for libraries being touched
  ├─ Read prototype (if provided) — ask PM all gaps before continuing
  └─ Load UI inventory (frontend tasks)

Step 1:  speckit-specify       Velents-aware spec (reads codebase first)
Step 2:  speckit-challenge      [GATE] APPROVE required
Step 3:  speckit-clarify        Only if challenger flagged ambiguity
Step 4:  speckit-plan           Exact file paths, existing classes to extend
Step 5:  speckit-challenge      [GATE] APPROVE required
Step 6:  speckit-tasks          Atomic tasks, max 15 per phase
Step 7:  speckit-challenge      [GATE] APPROVE required
         ── Plan shown to you — approval required before implementation ──
Step 8:  speckit-implement      One phase at a time, php -l/tsc after every file
         speckit-challenge      [GATE] after every phase
Step 9:  test-generator         Full test coverage
Step 10: testing-engineer       Run full test suite — 0 failures required
Step 11: ui-pixel-validator     Chrome E2E + pixel-perfect validation
Step 12: code-reviewer          Security + patterns
Step 13: pr-reviewer            PR review with severity tiers
```

REVISE verdicts auto-retry (max 2×). REJECT stops and asks you.

---

## Agents (33 total)

### Orchestration
| Agent | Model | Purpose |
|-------|-------|---------|
| `velents-router` | haiku | Intent classification |
| `workflow-orchestrator` | **opus** | Classifies FAST vs FULL, runs the pipeline |
| `fullstack-developer` | sonnet | **Fast path** — bug fixes, small changes, ≤3 files |

### Laravel Backend
| Agent | Model | Purpose |
|-------|-------|---------|
| `laravel-developer` | **opus** | Controllers, repositories, services, jobs, migrations |
| `api-developer` | sonnet | REST API routes, form requests, resources, permissions |
| `database-developer` | sonnet | PostgreSQL schemas, Eloquent, tenant DB management |
| `tenancy-specialist` | sonnet | stancl/tenancy v3, database-per-tenant isolation |

### Next.js Frontend
| Agent | Model | Purpose |
|-------|-------|---------|
| `frontend-developer` | **opus** | Pages, components, API integration, WebSocket, i18n + RTL |
| `ui-developer` | sonnet | shadcn/ui (46 components), Tailwind CSS 4, RTL Arabic |
| `state-developer` | sonnet | Zustand stores, TanStack Query v5, React Hook Form + Zod |

### Integrations
| Agent | Model | Purpose |
|-------|-------|---------|
| `integration-developer` | sonnet | ElevenLabs, LiveKit, Gemini, CallGateway HTTP clients |
| `webhook-developer` | sonnet | Inbound webhooks from CallGateway, ElevenLabs, GenesysCloud |
| `voice-pipeline-developer` | **opus** | Normal / Fast / Flash pipelines, S2S post-processing |
| `channel-developer` | sonnet | WhatsApp (Meta API), Genesys Cloud (9 regions), WebChat |

### Domain Features
| Agent | Model | Purpose |
|-------|-------|---------|
| `agent-builder-developer` | **opus** | 7-tab AI agent builder, tools, schemas, channels |
| `conversation-developer` | sonnet | Text + voice conversation lifecycle, real-time messaging |
| `analytics-developer` | sonnet | GeminiJudge quality scoring, dashboards |
| `payment-developer` | sonnet | Credit billing, payment plans, per-message deduction |

### Quality & Testing
| Agent | Model | Purpose |
|-------|-------|---------|
| `code-reviewer` | sonnet | Security, tenancy leaks, pattern compliance |
| `pr-reviewer` | sonnet | PR review with severity tiers (CRITICAL/HIGH/MEDIUM/LOW) |
| `production-risk-analyzer` | **opus** | Pre-deployment risk assessment |
| `testing-engineer` | sonnet | PHPUnit + Vitest, MultiTenancyTestCase |
| `test-generator` | sonnet | Generate test coverage from existing code |
| `ui-pixel-validator` | **opus** | Chrome pixel-perfect comparison against prototype |

### Spec-Driven Development
| Agent | Model | Purpose |
|-------|-------|---------|
| `speckit-specify` | **opus** | Velents-aware spec — reads codebase + prototype first |
| `speckit-clarify` | haiku | Resolve ambiguities |
| `speckit-plan` | **opus** | Implementation plan with exact Velents file paths |
| `speckit-tasks` | haiku | Dependency-ordered tasks, max 15 per phase |
| `speckit-implement` | **opus** | Per-task verification + prototype gate before frontend |
| `speckit-challenge` | **opus** | Adversarial gate — 4 modes, APPROVE/REVISE/REJECT |

### Product Management
| Agent | Model | Purpose |
|-------|-------|---------|
| `pm-discovery` | **opus** | OST, experiments, B2B interviews, assumption mapping |
| `pm-strategist` | **opus** | Lean Canvas, SWOT, B2B SaaS metrics (ARR, NRR, NDR) |
| `pm-executor` | **opus** | PRDs with tenant impact, OKRs, sprint plans, RACI |

---

## Skills (22 total)

Skills are domain knowledge injected into agents automatically — never invoked directly.

### Codebase Patterns (real code extracted from VelentsAI)
| Skill | Contains |
|-------|----------|
| `velents-backend` | Controller, Repository, Service, Job, Resource, FormRequest, Migration — exact patterns from the codebase |
| `velents-frontend` | ApiClient singleton, Service classes, Zustand store, TanStack Query hooks, component patterns |
| `velents-testing` | MultiTenancyTestCase (full class), PHPUnit feature tests, cross-tenant isolation patterns |

### Domain Knowledge
| Skill | Covers |
|-------|--------|
| `velents-core-flows` | 4 core business flows, key models, external services |
| `velents-architecture` | System architecture, 5-layer structure, repo layout |
| `velents-multitenancy` | stancl/tenancy v3, database-per-tenant, tenant context |
| `velents-auth-rbac` | Sanctum tokens, Spatie permissions, 50 permissions across 11 categories |
| `velents-integrations` | ElevenLabs, LiveKit, CallGateway, WhatsApp, Gemini patterns |
| `velents-voice` | Pipeline types (Normal/Fast/Flash), S2S, WebRTC |
| `velents-agent-model` | AI agent CRUD, tools, schemas, enums, relations |
| `velents-realtime` | Laravel Echo, Pusher/Reverb, WebSocket patterns |
| `velents-payment` | Credit system, payment plans, per-message deduction |
| `velents-analytics` | GeminiJudge, 6 quality metrics, conversation analysis |
| `velents-ui-inventory` | 46 shadcn/ui components, OKLCH colors, RTL rules — reuse only |

### Development Standards
| Skill | Does |
|-------|------|
| `velents-dev-standards` | Mandatory protocols: codebase scan first, tenant isolation, per-file verification |
| `velents-llms-txt` | llms.txt URLs for the full stack — fetched before writing library code |
| `velents-feature-map` | Impact map — shared resources risk table (🔴 CRITICAL resources) |
| `velents-ui-prototype` | Read prototype → identify gaps → ask PM → validate in Chrome |

### Product Management
| Skill | Covers |
|-------|--------|
| `pm-discovery-frameworks` | OST, experiment design, assumption mapping, JTBD interviews, ICE/RICE |
| `pm-strategy-frameworks` | Lean Canvas, SWOT, Porter's Five Forces, B2B SaaS metrics |
| `pm-execution-frameworks` | PRD template, OKR cascade, sprint planning, stakeholder map, RACI |
| `docs-reference` | Official llms.txt paths for React 19, Next.js 16, Zustand, Zod, ElevenLabs, LiveKit, Tiptap |

---

## Commands (21 total)

| Command | Routes To |
|---------|-----------|
| `/velents` | Smart router → FAST or FULL path automatically |
| `/backend` | `laravel-developer` |
| `/api` | `api-developer` |
| `/db` | `database-developer` |
| `/frontend` | `frontend-developer` |
| `/ui` | `ui-developer` |
| `/integrate` | `integration-developer` |
| `/voice` | `voice-pipeline-developer` |
| `/channel` | `channel-developer` |
| `/agent-build` | `agent-builder-developer` |
| `/convo` | `conversation-developer` |
| `/analytics` | `analytics-developer` |
| `/payment` | `payment-developer` |
| `/review` | `code-reviewer` |
| `/review-pr` | `pr-reviewer` |
| `/risk` | `production-risk-analyzer` |
| `/test` | `testing-engineer` |
| `/generate-tests` | `test-generator` |
| `/pm-discover` | `pm-discovery` |
| `/pm-strategy` | `pm-strategist` |
| `/pm-execute` | `pm-executor` |

---

## Contributing

The toolkit grows as the codebase grows. If you spot a pattern, a domain, or a workflow that isn't covered — add it.

### How to Contribute

**1. Find a pattern in the codebase**

Look for things that you do repeatedly that Claude gets wrong or slow on. Examples:
- You always write the same boilerplate for a new Genesys integration
- The audit log always needs 3 files changed together
- Every new channel type follows the same 5-step setup
- There's a WhatsApp template pattern Claude keeps inventing from scratch

**2. Decide: agent or skill?**

| Add a **skill** when | Add an **agent** when |
|---------------------|-----------------------|
| It's knowledge — patterns, code examples, conventions | It's a workflow — a sequence of steps to accomplish a task |
| Multiple agents need it (reusable) | It's a specialized role (e.g., `whatsapp-developer`) |
| It's real code from the codebase | It orchestrates other work |
| Example: a new integration's HTTP client pattern | Example: a dedicated agent for a new domain feature |

**3. Create the file**

For a **skill** — create `skills/{skill-name}/SKILL.md`:
```markdown
---
name: skill-name
description: One sentence — what this skill provides
user-invocable: false
---

# Skill Title

[What agents need to know — patterns, conventions, real code examples]
```

For an **agent** — create `agents/{group}/{agent-name}.md`:
```markdown
---
name: agent-name
description: One sentence — what this agent does
tools: Read, Write, Edit, Glob, Grep, Bash
model: sonnet          # use opus only if it needs deep reasoning
skills:
  - velents-backend    # load only what this agent actually needs
---

# Agent Title

[Role in 1-2 sentences]

## Workflow
[What to do, step by step — no code, reference skills for patterns]

## Rules
[Specific constraints for this agent's domain]
```

**4. Register it in `plugin.json`**

Add the path to the `agents` or `skills` array in `.claude-plugin/plugin.json`.

**5. If it's an agent, add a command** (optional but recommended)

Create `commands/{name}.md`:
```markdown
---
description: One sentence about what this command does
---

# /command-name

Route to: agent-name

## Input
$ARGUMENTS
```

Add it to the `commands` array in `plugin.json`.

**6. Sync to cache and test**

```bash
# Sync to plugin cache
SOURCE="/path/to/claude-code-velents"
CACHE="$HOME/.claude/plugins/cache/velents-toolkit/velents-toolkit/1.0.0"
rsync -a "$SOURCE/agents/" "$CACHE/agents/"
rsync -a "$SOURCE/skills/" "$CACHE/skills/"
rsync -a "$SOURCE/.claude-plugin/" "$CACHE/.claude-plugin/"

# Restart Claude Code, then test your agent/skill
/velents [something that triggers your new agent]
```

**7. PR it**

Push to `main` — the cache sync happens locally, but the repo is the source of truth for the team.

### Contribution Examples

| What you noticed | What to add |
|-----------------|-------------|
| A new Genesys region needs the same 3-file setup every time | Skill: `velents-genesys-patterns` with the boilerplate + Agent: `genesys-developer` |
| The SMS channel follows the same pattern as WhatsApp | Skill: update `velents-integrations` with SMS patterns |
| Audit log always needs `before`/`after` from 3 specific models | Update `velents-backend` skill with the exact pattern |
| A new webhook source (e.g. Salesforce) | Add to `velents-integrations` skill + extend `webhook-developer` agent |
| A recurring PM task (e.g. writing release notes) | Skill: `pm-release-frameworks` + extend `pm-executor` agent |

### Guidelines

- **Agents**: ~50 lines, no code blocks — reference a skill for patterns
- **Skills**: real code from the codebase only — no invented examples
- **One thing per contribution** — don't combine a new agent + new skill + command overhaul in one PR
- **Test before pushing** — sync to cache, run `/velents [trigger]`, verify it works

---

## Author

Built by **[Adham Mahmoud](https://github.com/adhammahmoud)** · hi@adhamm.dev
