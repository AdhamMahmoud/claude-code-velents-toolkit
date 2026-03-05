# VelentsAI Claude Agent Toolkit

> Built by **Adham Mahmoud** for the VelentsAI engineering team.
> A Claude Code plugin that gives every team member an AI pair-programmer who knows the VelentsAI codebase as well as you do.

---

## What is this?

The VelentsAI Claude Agent Toolkit is a **Claude Code plugin** — a collection of 32 specialized AI agents, 15 domain knowledge skills, and 20 slash commands built specifically for the VelentsAI platform.

Instead of explaining the codebase to Claude every time, every agent in this toolkit already knows:
- The multi-tenant architecture (stancl/tenancy v3, database-per-tenant)
- The 4 core business flows (text conversation, voice call, agent builder, batch campaign)
- Laravel 12 + Next.js 16 patterns used across the codebase
- All 46 shadcn/ui components and the OKLCH color system
- Voice pipeline types (Normal/Fast/Flash), ElevenLabs, LiveKit, CallGateway
- Payment/credit system, RBAC permissions, WebSocket patterns
- Official docs for React, Next.js, Zustand, Zod, ElevenLabs, LiveKit, and more

**Stack**: Laravel 12 · PHP 8.4 · Next.js 16 · React 19 · TypeScript 5.9 · PostgreSQL · Redis · Tailwind CSS 4 · stancl/tenancy v3

---

## Who Should Use This

| Role | What You Get |
|------|-------------|
| **Backend Developer** | Laravel controllers, APIs, migrations, multi-tenancy, jobs — all following VelentsAI patterns |
| **Frontend Developer** | Next.js pages, React components, state management, RTL Arabic support |
| **Full-Stack Developer** | End-to-end features from DB migration to UI, coordinated across both repos |
| **DevOps / Integration** | ElevenLabs, LiveKit, CallGateway, WhatsApp, Genesys Cloud integrations |
| **QA Engineer** | PHPUnit + Vitest test generation, multi-tenant test isolation, E2E scenarios |
| **Tech Lead** | PR reviews, production risk analysis, security audits, performance profiling |
| **Product / Spec Writer** | Spec-driven development (constitution → specify → plan → implement) |
| **AI Feature Builder** | Voice pipeline, agent builder CRUD, conversation flows, analytics |

---

## Installation

```bash
claude plugin add /path/to/velents-toolkit
```

Or if published to a marketplace:

```bash
claude plugin add velents-toolkit
```

---

## Quick Start

Use the main router for anything — it auto-detects what you need:

```
/velents build the WhatsApp channel configuration page
/velents fix the tenant isolation bug in the conversations API
/velents review the PR for the ElevenLabs webhook handler
/velents write tests for the BatchCallService
```

Or use a specific command directly:

```
/backend    → Laravel backend work
/frontend   → Next.js / React work
/api        → REST API endpoints
/db         → Database migrations & schemas
/ui         → UI components & design
/voice      → Voice pipeline features
/integrate  → External service integrations
/review     → Code review
/review-pr  → PR security review
/security   → OWASP & tenant isolation checks
/perf       → Performance budgets & optimization
/test       → Write tests
/risk       → Production risk assessment
```

---

## Agents (32 total)

### Orchestration
| Agent | Model | Purpose |
|-------|-------|---------|
| `velents-router` | haiku | Smart router — detects intent and delegates to the right agent |
| `workflow-orchestrator` | sonnet | Multi-agent workflow coordination |

### Laravel Backend
| Agent | Model | Purpose |
|-------|-------|---------|
| `laravel-developer` | **opus** | Controllers, repositories, services, jobs, migrations |
| `api-developer` | sonnet | REST API routes, form requests, resources, Spatie permissions |
| `database-developer` | sonnet | PostgreSQL schemas, Eloquent, tenant DB management |
| `tenancy-specialist` | sonnet | stancl/tenancy v3, database-per-tenant isolation |

### Next.js Frontend
| Agent | Model | Purpose |
|-------|-------|---------|
| `frontend-developer` | **opus** | Pages, components, API integration, WebSocket, i18n + RTL |
| `ui-developer` | sonnet | shadcn/ui (46 components), Tailwind CSS 4, Tiptap, RTL Arabic |
| `state-developer` | sonnet | Zustand stores, TanStack Query, React Hook Form + Zod |

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
| `agent-builder-developer` | **opus** | 7-tab AI agent builder, tools, schemas, channels, deployment |
| `conversation-developer` | sonnet | Text + voice conversation lifecycle, real-time messaging |
| `analytics-developer` | sonnet | GeminiJudge quality scoring, dashboards, 6-metric analysis |
| `payment-developer` | sonnet | Credit billing, payment plans, per-message deduction |

### Quality & Testing
| Agent | Model | Purpose |
|-------|-------|---------|
| `code-reviewer` | sonnet | Security, multi-tenancy leaks, pattern compliance — use after every change |
| `pr-reviewer` | sonnet | Security-first PR review with severity tiers (CRITICAL / HIGH / MEDIUM / LOW) |
| `production-risk-analyzer` | **opus** | 8-category risk assessment before deployment |
| `security-specialist` | sonnet | OWASP Top 10, tenant isolation, API token security |
| `performance-optimizer` | sonnet | API latency, voice pipeline, WebSocket, bundle size budgets |
| `testing-engineer` | sonnet | PHPUnit + Vitest, MultiTenancyTestCase, tenant isolation tests |
| `test-generator` | sonnet | AI-verifiable PHPUnit / Vitest / E2E test scenarios |

### Spec-Driven Development
| Agent | Model | Purpose |
|-------|-------|---------|
| `speckit-constitution` | sonnet | Project principles and constraints (run once per project) |
| `speckit-specify` | **opus** | Feature specification from natural language |
| `speckit-clarify` | haiku | Resolve ambiguities (max 5 questions) |
| `speckit-plan` | **opus** | Technical implementation plan with architecture decisions |
| `speckit-tasks` | haiku | Dependency-ordered task breakdown |
| `speckit-checklist` | sonnet | Requirement quality checklists |
| `speckit-analyze` | haiku | Consistency check across spec → plan → tasks |
| `speckit-implement` | **opus** | Phase-by-phase implementation execution |

---

## Skills (15 total)

Skills are domain knowledge injected into agents automatically — you don't invoke them directly.

| Skill | Loaded By | What It Knows |
|-------|-----------|---------------|
| `velents-core-flows` | All agents | 4 core business flows, key models, external services |
| `velents-architecture` | Orchestration | Full system architecture, repo structure |
| `velents-backend` | Backend agents | Laravel patterns, repository pattern, service layer |
| `velents-frontend` | Frontend agents | Next.js App Router, component patterns, state management |
| `velents-multitenancy` | Backend agents | stancl/tenancy v3, tenant context, DB isolation |
| `velents-auth-rbac` | Backend agents | Sanctum tokens, Spatie permissions, RBAC |
| `velents-integrations` | Integration agents | ElevenLabs, LiveKit, CallGateway, WhatsApp APIs |
| `velents-voice` | Voice agents | Pipeline types, S2S, WebRTC, audio processing |
| `velents-agent-model` | Domain agents | AI agent CRUD, tools, schemas, deployment |
| `velents-realtime` | Frontend + conversation agents | Laravel Echo, Pusher/Reverb, WebSocket patterns |
| `velents-payment` | Payment agents | Credit system, payment plans, billing flows |
| `velents-analytics` | Analytics agents | GeminiJudge, 6 quality metrics, conversation analysis |
| `velents-testing` | Test agents | MultiTenancyTestCase, PHPUnit, Vitest patterns |
| `velents-ui-inventory` | Frontend agents | 46 shadcn/ui components, OKLCH colors, RTL rules |
| `docs-reference` | All dev agents | Paths to official llms.txt docs (React, Next.js, Zustand, Zod, ElevenLabs, LiveKit, Tiptap, Redis, shadcn/ui, Vitest) |

---

## Official Documentation

The toolkit ships with pre-downloaded official docs in `llms-docs/` — agents read these before writing code instead of guessing.

| Technology | File | Size |
|------------|------|------|
| React 19 | `llms-docs/react.txt` | 14 KB |
| Next.js 16 | `llms-docs/nextjs.txt` | 1.2 KB |
| Zustand | `llms-docs/zustand.txt` | 3.6 KB |
| Zod | `llms-docs/zod.txt` | 1.5 KB |
| shadcn/ui | `llms-docs/shadcn-ui.txt` | 1.2 KB |
| Vitest | `llms-docs/vitest.txt` | 8.7 KB |
| Tiptap | `llms-docs/tiptap.txt` | 59 KB |
| ElevenLabs | `llms-docs/elevenlabs.txt` | 90 KB |
| LiveKit | `llms-docs/livekit.txt` | 765 lines |
| Google AI (Gemini) | `llms-docs/google-ai.txt` | — |
| Redis | `llms-docs/redis.txt` | 9.4 KB |

---

## Spec-Kit Workflow (for new features)

For any non-trivial feature, use the full spec-driven flow:

```
/velents build [feature description]
```

This triggers:
1. **speckit-specify** → Creates `.specify/specs/[feature]/spec.md` with user stories
2. **speckit-clarify** → Asks up to 5 targeted questions
3. **speckit-plan** → Generates `plan.md` with architecture decisions
4. **speckit-tasks** → Generates `tasks.md` with dependency-ordered tasks
5. **speckit-implement** → Executes phase by phase

---

## Quality Chain

Every code change should flow through:

```
Code Change
    ↓
/review          → code-reviewer (security, patterns, performance)
    ↓ (if CRITICAL)
/security        → security-specialist (OWASP, tenant isolation)
    ↓ (if slow)
/perf            → performance-optimizer (API latency, bundle size)
    ↓
/review-pr       → pr-reviewer (severity-tiered, merge decision)
    ↓ (before deploy)
/risk            → production-risk-analyzer (rollback plan, deployment checklist)
```

---

## Design Principles

- **Reuse first**: UI agents check `velents-ui-inventory` before creating any component
- **No over-engineering**: Agents are instructed to avoid abstractions for single uses
- **Tenant-safe by default**: Every backend agent knows the tenant isolation rules
- **RTL mandatory**: Every frontend agent supports Arabic right-to-left layout
- **Docs over memory**: Agents read official llms-docs before writing code in any technology

---

## Token Cost Optimization

Agents are assigned models based on task complexity:

| Model | Agents | When Used |
|-------|--------|-----------|
| **Opus** (8 agents) | laravel-developer, frontend-developer, voice-pipeline-developer, agent-builder-developer, production-risk-analyzer, speckit-specify, speckit-plan, speckit-implement | Complex multi-file architecture, orchestration |
| **Sonnet** (20 agents) | All code writers, reviewers, integrations | Standard implementation, code review, analysis |
| **Haiku** (4 agents) | velents-router, speckit-clarify, speckit-tasks, speckit-analyze | Routing, classification, simple structured tasks |

---

## Author

Built by **Adham Mahmoud** for the VelentsAI team.
