# VelentsAI Claude Agent Toolkit

> Built by **Adham Mahmoud** for the VelentsAI engineering team.
> One command. Full autonomous pipeline. From requirement to reviewed PR.

---

## What is this?

The VelentsAI Claude Agent Toolkit is a **Claude Code plugin** — 32 specialized AI agents, 22 domain knowledge skills, and 21 slash commands built specifically for the VelentsAI platform.

Every agent already knows:
- Multi-tenant architecture (stancl/tenancy v3, database-per-tenant isolation)
- 4 core business flows: text conversation, voice call, agent builder, batch campaign
- Laravel 12 + Next.js 16 patterns used across the actual codebase
- All 46 shadcn/ui components, OKLCH color system, RTL Arabic support
- Voice pipelines (Normal/Fast/Flash), ElevenLabs, LiveKit, CallGateway
- Payment/credit system, Spatie RBAC, WebSocket (Laravel Reverb)
- Official docs for every library — fetched before writing code, not guessed from training data

**Stack**: Laravel 12 · PHP 8.4 · Next.js 16 · React 19 · TypeScript 5.9 · PostgreSQL · Redis · Tailwind CSS 4 · stancl/tenancy v3

---

## The One Command

```
/velents [describe what you want]
```

That's it. You describe what you want once. The full pipeline runs automatically end-to-end — spec, challenge, plan, challenge, tasks, challenge, implement (with per-task verification), tests, browser E2E, pixel validation, code review, PR review.

**The only times it stops to ask you:**
1. A REJECT verdict from the adversarial challenger (fundamental flaw that needs your direction)
2. One genuine ambiguity at the start that can't be resolved from the codebase
3. A design decision with multiple valid options — it presents choices, not assumptions

```bash
/velents build the agent monitoring dashboard
/velents fix the tenant isolation bug in conversations API
/velents add WhatsApp channel support to the agent builder
/velents write tests for BatchCallService
```

---

## How the Pipeline Works

Every code task goes through this 15-step autonomous pipeline:

```
Phase 0: Context Loading (orchestrator runs this — no agent needed)
  ├─ Load architecture + feature impact map
  ├─ Read prototype (if provided) — ask PM all gaps before proceeding
  ├─ Fetch llms.txt docs for every library that will be touched
  └─ Load UI inventory (frontend tasks)

Step 1:  speckit-specify       Write Velents-aware spec (reads codebase first)
Step 2:  speckit-challenge      [GATE] APPROVE required — REVISE loops back
Step 3:  speckit-clarify        Only if challenger flagged ambiguity
Step 4:  speckit-plan           Exact file paths, existing classes to extend
Step 5:  speckit-challenge      [GATE] APPROVE required — REVISE loops back
Step 6:  speckit-tasks          Atomic tasks with per-task verification steps
Step 7:  speckit-challenge      [GATE] APPROVE required — REVISE loops back
         ── PLAN APPROVED: shown to you, wait for your go ──
Step 8:  speckit-implement      Per-task verification (php -l, tsc, artisan) after every file
Step 9:  speckit-challenge      [GATE] APPROVE required — REVISE loops back
Step 10: test-generator         Full test coverage
Step 11: testing-engineer       Run full test suite — 0 failures required
Step 12: browser-e2e-tester     Chrome E2E on all acceptance scenarios
Step 13: ui-pixel-validator     Chrome pixel-perfect comparison against prototype
         [GATE] PASS required — discrepancies loop back to step 8
Step 14: code-reviewer          Security + patterns inline
Step 15: pr-reviewer            Formal PR review with severity tiers
```

REVISE verdicts auto-retry (max 2x). REJECT stops and asks you.

---

## Quality Gates Explained

### speckit-challenge (Adversarial Opus Agent)
Runs after every spec-kit artifact. Issues APPROVE / REVISE / REJECT.

- **challenge-spec**: checks tenant scope, permissions, existing contract breakage, missing error scenarios, UI scope creep
- **challenge-plan**: checks file path conventions, migration placement, repo pattern compliance, missing plan items
- **challenge-tasks**: checks complete coverage, ordering, verification tasks present
- **challenge-implementation**: reads every file, runs `php -l` + `tsc --noEmit`, checks tenant isolation queries, permission enforcement

### ui-pixel-validator (Chrome-Based Visual QA)
Runs after browser E2E. Opens prototype and implementation side-by-side in Chrome.
- Screenshots every state: default, loading, empty, error, hover, mobile (375px), RTL (Arabic)
- DevTools inspection for exact computed CSS values (padding, color, font-size)
- PASS/FAIL verdict with discrepancy table: `element | prototype value | implementation | fix`
- Zero CRITICAL discrepancies required for PASS

---

## Agents (32 total)

### Orchestration
| Agent | Model | Purpose |
|-------|-------|---------|
| `velents-router` | haiku | Intent classification → invokes workflow-orchestrator immediately |
| `workflow-orchestrator` | **opus** | Autonomous 15-step pipeline — runs without asking between steps |

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
| `ui-developer` | sonnet | shadcn/ui (46 components), Tailwind CSS 4, Tiptap, RTL Arabic — reads prototype first |
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
| `agent-builder-developer` | **opus** | 7-tab AI agent builder, tools, schemas, channels, deployment |
| `conversation-developer` | sonnet | Text + voice conversation lifecycle, real-time messaging |
| `analytics-developer` | sonnet | GeminiJudge quality scoring, dashboards, 6-metric analysis |
| `payment-developer` | sonnet | Credit billing, payment plans, per-message deduction |

### Quality & Testing
| Agent | Model | Purpose |
|-------|-------|---------|
| `code-reviewer` | sonnet | Security, tenancy leaks, pattern compliance — runs after every change |
| `pr-reviewer` | sonnet | Security-first PR review with severity tiers (CRITICAL/HIGH/MEDIUM/LOW) |
| `production-risk-analyzer` | **opus** | Pre-deployment risk assessment |
| `testing-engineer` | sonnet | PHPUnit + Vitest, MultiTenancyTestCase, tenant isolation tests |
| `test-generator` | sonnet | AI-verifiable test scenarios from existing code |
| `ui-pixel-validator` | **opus** | Chrome pixel-perfect comparison against prototype — PASS required before merge |

### Spec-Driven Development
| Agent | Model | Purpose |
|-------|-------|---------|
| `speckit-specify` | **opus** | Velents-aware spec — reads codebase + prototype first |
| `speckit-clarify` | haiku | Resolve ambiguities |
| `speckit-plan` | **opus** | Implementation plan with exact Velents file paths |
| `speckit-tasks` | haiku | Dependency-ordered task breakdown with verification steps |
| `speckit-implement` | **opus** | Per-task verification loop + prototype gate before frontend |
| `speckit-challenge` | **opus** | Adversarial gate — 4 modes, APPROVE/REVISE/REJECT |

### Product Management
| Agent | Model | Purpose |
|-------|-------|---------|
| `pm-discovery` | **opus** | Product discovery: OST, experiments, B2B interviews, assumption mapping |
| `pm-strategist` | **opus** | Platform strategy: Lean Canvas, SWOT, B2B SaaS metrics (ARR, NRR, NDR) |
| `pm-executor` | **opus** | Execution: PRDs with tenant impact, OKRs, sprint plans, RACI |

---

## Skills (22 total)

Skills are domain knowledge injected into agents automatically — you never invoke them directly.

### Velents Domain Knowledge
| Skill | What It Knows |
|-------|---------------|
| `velents-core-flows` | 4 core business flows, key models, external services |
| `velents-architecture` | Full system architecture, 5-layer structure, repo layout |
| `velents-backend` | Laravel patterns, repository pattern, service layer conventions |
| `velents-frontend` | Next.js 16 App Router, ApiClient, component patterns, state management |
| `velents-multitenancy` | stancl/tenancy v3, database-per-tenant, tenant context |
| `velents-auth-rbac` | Sanctum tokens, Spatie permissions, 50 permissions across 11 categories |
| `velents-integrations` | ElevenLabs, LiveKit, CallGateway, WhatsApp, Gemini API patterns |
| `velents-voice` | Pipeline types (Normal/Fast/Flash), S2S, WebRTC, audio processing |
| `velents-agent-model` | AI agent CRUD, tools, schemas, enums, relations |
| `velents-realtime` | Laravel Echo, Pusher/Reverb, WebSocket patterns |
| `velents-payment` | Credit system, payment plans, per-message deduction |
| `velents-analytics` | GeminiJudge, 6 quality metrics, conversation analysis |
| `velents-testing` | MultiTenancyTestCase, PHPUnit, Vitest patterns |
| `velents-ui-inventory` | 46 shadcn/ui components, OKLCH colors, RTL rules — REUSE ONLY |

### Quality & Prevention Skills
| Skill | What It Does |
|-------|--------------|
| `velents-dev-standards` | 9 mandatory protocols: codebase scan, tenant isolation, per-file verification, prototype reading, never assume |
| `velents-llms-txt` | llms.txt URLs for the full Velents stack — fetched before writing code (Next.js 16, React 19, TanStack Query v5, Laravel 12, Tailwind v4) |
| `velents-feature-map` | Impact map of all 8 features — shared resources risk table (🔴 CRITICAL: agents table, Payment::Pay(), Spatie permissions) |
| `velents-ui-prototype` | 5-phase protocol: read prototype → gap analysis → ask PM → component map → Chrome validation |

### Product Management Skills
| Skill | What It Covers |
|-------|---------------|
| `pm-discovery-frameworks` | OST (Teresa Torres), experiment design, assumption mapping, JTBD interviews, ICE/RICE scoring |
| `pm-strategy-frameworks` | Lean Canvas, SWOT, Porter's Five Forces, B2B SaaS metrics, platform strategy |
| `pm-execution-frameworks` | PRD with tenant impact section, OKR cascade, sprint planning, stakeholder map, RACI |

### Reference
| Skill | What It Covers |
|-------|---------------|
| `docs-reference` | Paths to official llms.txt docs (React 19, Next.js 16, Zustand, Zod, ElevenLabs, LiveKit, shadcn/ui, Vitest, Redis, Tiptap) |

---

## The "Never Assume" Rule

Every agent in this toolkit follows a strict no-assumption policy:

- **Design decisions** (UI layout, data model, interaction) → stop and present 2-3 options with trade-offs and a recommendation
- **Unclear scope** → ask explicitly before proceeding
- **Prototype gaps** → document every missing state/interaction, ask PM before writing spec
- **Conflicting signals** → surface the conflict, don't resolve it silently

When asking: always give concrete options + recommendation. Never open-ended questions.

The plan is also shown to you for explicit approval before implementation starts — "yes/proceed" required.

---

## Commands (21 total)

| Command | Routes To |
|---------|-----------|
| `/velents` | Smart router → full autonomous pipeline |
| `/backend` | Laravel backend development |
| `/api` | REST API endpoints |
| `/db` | Database migrations & schemas |
| `/frontend` | Next.js / React development |
| `/ui` | UI components (prototype-first) |
| `/integrate` | External service integrations |
| `/voice` | Voice pipeline features |
| `/channel` | WhatsApp, Genesys, WebChat |
| `/agent-build` | AI agent CRUD builder |
| `/convo` | Conversation flows |
| `/analytics` | Quality dashboards |
| `/payment` | Credit billing |
| `/review` | Code review |
| `/review-pr` | PR security review |
| `/risk` | Production risk assessment |
| `/test` | Run tests |
| `/generate-tests` | Generate test coverage |
| `/pm-discover` | Product discovery |
| `/pm-strategy` | Product strategy |
| `/pm-execute` | PRDs, OKRs, sprint plans |

---

## Pixel-Perfect UI Workflow

When a PM provides a prototype (Figma, screenshots, Loom, etc.):

```
1. Phase 0: orchestrator reads prototype in Chrome — documents every screen
2. Identifies all gaps: states not shown, interactions unclear, copy not final
3. STOPS — asks PM all questions with concrete options + recommendation
4. PM answers → spec encodes all confirmed states
5. speckit-implement: PROTOTYPE GATE before frontend — blocks on any "Pending" state
6. ui-developer: implements against prototype, not against memory
7. ui-pixel-validator: opens both prototype + implementation in Chrome
   → screenshots every state (default/loading/empty/error/hover/mobile/RTL)
   → DevTools inspection for exact CSS values
   → PASS/FAIL with discrepancy table
8. Fix loop until PASS — then speckit-challenge runs
```

---

## What Was Fixed (The 40% Rework Problem)

Before this toolkit was built, features finished by agents had ~40% issues requiring a second pass. Root causes and fixes:

| Root Cause | Fix |
|---|---|
| speckit-specify didn't read the codebase | Now scans actual files before writing spec |
| speckit-plan used invented file paths | Now scans actual codebase, outputs exact paths |
| speckit-implement marked [X] without verifying | Now runs `php -l` / `tsc` / `artisan` after every file |
| No challenge gates | speckit-challenge runs after EVERY step |
| No browser test in loop | browser-e2e-tester + ui-pixel-validator now required |
| Agents guessed UI from requirements | velents-ui-prototype skill: read prototype, ask gaps, validate in Chrome |
| Agents used outdated library APIs | velents-llms-txt: fetch docs before writing any library code |
| Changes broke existing features silently | velents-feature-map: impact checklist before any code |

---

## Author

Built by **[Adham Mahmoud](https://github.com/adhammahmoud)** · hi@adhamm.dev
