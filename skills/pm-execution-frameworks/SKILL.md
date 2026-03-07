---
name: pm-execution-frameworks
description: Execution frameworks for B2B SaaS — PRD with tenant impact, B2B OKRs, release notes with API changelog, stakeholder mapping with customer advisory
user-invocable: false
---

# PM Execution Frameworks (B2B SaaS)

## PRD Template — B2B with Tenant Impact

### Standard 8 Sections + B2B Extensions

**1. Overview**
- Title, Author, Date, Status
- Problem (cite customer interviews/support tickets)
- Goal (tied to B2B metric: ARR, NRR, activation)

**2. Background & Context**
- Customer advisory feedback
- Support ticket trends
- Competitive pressure
- Enterprise customer requests

**3. User Stories & Requirements**
- Stories per persona: buyer, user, admin
- P0/P1/P2 prioritization
- Format: As a [tenant admin/user/buyer], I want [action] so that [B2B benefit]
- Acceptance criteria: Given [tenant context], When [action], Then [result]

**4. Tenant Impact Assessment**
| Impact Area | Details |
|-------------|---------|
| **Data model** | New tables? Tenant-scoped columns? |
| **API changes** | New endpoints? Breaking changes? Versioning needed? |
| **Migration** | Per-tenant migration? Backfill needed? Downtime? |
| **Billing** | Credit impact? New plan features? Usage metering? |
| **Permissions** | New roles? New permissions? Spatie config? |
| **Multi-tenant** | Tenant isolation verified? Cross-tenant risk? |

**5. Technical Approach**
- Architecture changes (reference velents-architecture skill)
- API design (REST endpoints, request/response)
- Database changes (migrations, indexes)
- Integration impact (webhooks, external services)

**6. Success Metrics**
| Metric | Baseline | Target | Segment |
|--------|----------|--------|---------|
| [Metric] | [Current] | [Goal] | [All tenants / Enterprise / SMB] |

**7. Timeline & Milestones**
| Phase | Duration | Deliverable | Tenant Impact |
|-------|----------|-------------|---------------|
| [Phase] | [Weeks] | [Output] | [Migration/downtime/none] |

**8. Risks & Dependencies**
| Risk | Impact | Likelihood | Mitigation | Tenant Risk? |
|------|--------|------------|------------|-------------|
| [Risk] | H/M/L | H/M/L | [Action] | Yes/No |

---

## B2B OKR Templates

### Company-Level OKRs (Quarterly)
```
Objective: Accelerate enterprise adoption of AI agents
  KR1: Grow ARR from $X to $Y
  KR2: Achieve NRR of 115%+
  KR3: Onboard X new enterprise tenants
  KR4: Reduce time-to-first-agent from X days to Y days
```

### Product Team OKRs
```
Objective: Deliver a world-class agent builder experience
  KR1: Increase agents deployed per tenant from X to Y
  KR2: Reduce agent configuration time from X min to Y min
  KR3: Achieve 90%+ activation rate for new tenants
```

### Engineering Team OKRs
```
Objective: Scale platform reliability for enterprise growth
  KR1: Maintain 99.9% API uptime
  KR2: Reduce P95 API latency from Xms to Yms
  KR3: Support X concurrent voice calls per tenant
  KR4: Zero tenant data isolation incidents
```

### Scoring (0.0 — 1.0)
- 0.7 = good (stretched and delivered)
- 1.0 = goal wasn't ambitious enough
- Review weekly, score quarterly

---

## Release Notes — B2B with API Changelog

### Internal Release Notes
```
## Release [version] — [date]

### What shipped
- [Feature]: [Description] ([Ticket])

### Tenant impact
- Migration: [Yes/No — details]
- API changes: [New/Changed/Deprecated endpoints]
- Billing: [Credit/plan changes]
- Permissions: [New permissions required]

### Metrics to watch
- [Metric]: expected [direction] by [amount]

### Customer communication
- [ ] CS team notified
- [ ] Enterprise accounts briefed
- [ ] Documentation updated

### Rollback plan
- [Steps]
```

### External Release Notes (Customer-Facing)
```
## What's New — [Month Year]

### New Features
**[Feature Name]**
[2-3 sentences, focus on customer value]

### API Changes
- `POST /api/v1/new-endpoint` — [Description]
- `GET /api/v1/existing` — Added `field` parameter
- **Deprecated:** `GET /api/v1/old-endpoint` — Use [replacement] instead

### Improvements
- [Performance/UX improvement]

### Bug Fixes
- Fixed [issue]
```

---

## Stakeholder Mapping — B2B with Customer Advisory

### Internal Stakeholders (Power × Interest Grid)

| Stakeholder | Power | Interest | Strategy |
|-------------|-------|----------|----------|
| CEO/Founder | High | High | Manage closely — weekly sync |
| Engineering Lead | High | High | Manage closely — sprint review |
| Customer Success | Medium | High | Keep informed — feature updates |
| Sales | Medium | High | Keep informed — positioning help |

### External Stakeholders

| Stakeholder | Type | Engagement |
|-------------|------|-----------|
| Enterprise customers | Advisory board | Monthly advisory call, beta access |
| Integration partners | Technical | Quarterly roadmap preview |
| Investors | Board | Quarterly business review |

### Customer Advisory Board (CAB)
- **Size:** 5-8 enterprise customers
- **Cadence:** Monthly 1-hour calls
- **Format:** 15 min product update, 30 min feedback, 15 min roadmap preview
- **Value exchange:** Early access to features, roadmap influence, NDA-protected previews
- **Selection:** Mix of power users, strategic accounts, vocal champions

---

## RACI — B2B Cross-Functional

| Decision | Product | Engineering | CS | Sales | Legal |
|----------|---------|------------|-----|-------|-------|
| Feature prioritization | **A** | C | C | I | — |
| Technical design | C | **A** | — | — | — |
| API breaking change | **A** | R | C | I | C |
| Pricing change | C | I | C | **A** | C |
| Data privacy decision | C | C | I | — | **A** |
| Customer communication | C | I | **A** | C | I |

---

## Sprint Planning — B2B Considerations

### Capacity Allocation
- **70%** — Feature development (customer-requested + product-led)
- **20%** — Technical debt and platform reliability
- **10%** — Customer escalations and support engineering

### Definition of Ready (B2B Extended)
- [ ] User story with acceptance criteria
- [ ] Tenant impact assessed (migration, API, billing, permissions)
- [ ] Design approved
- [ ] API contract reviewed (if applicable)
- [ ] Customer communication plan (if breaking change)
- [ ] Story pointed by team

## Links
- [Inspired — Marty Cagan](https://www.svpg.com/)
- [Empowered — Marty Cagan](https://www.svpg.com/)
- [Product-Led Growth — Wes Bush](https://productled.com/)
