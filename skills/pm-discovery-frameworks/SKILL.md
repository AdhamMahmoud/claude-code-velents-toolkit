---
name: pm-discovery-frameworks
description: Product discovery frameworks adapted for B2B SaaS AI agent platforms — OST, experiments, assumption mapping, B2B interview templates
user-invocable: false
---

# PM Discovery Frameworks (B2B SaaS)

## Opportunity Solution Tree (Teresa Torres) — B2B Adapted

```
Desired Outcome (B2B metric: ARR, NRR, activation)
├── Opportunity 1 (enterprise buyer need)
│   ├── Solution A → Experiment (tenant-aware)
│   └── Solution B → Experiment (multi-persona)
├── Opportunity 2 (end-user pain)
│   └── Solution C → Experiment (usage analytics)
└── Opportunity 3 (admin friction)
    └── Solution D → Experiment (onboarding test)
```

**B2B OST Rules:**
- Outcomes tied to B2B metrics (ARR growth, NRR, activation rate, expansion revenue)
- Opportunities sourced from three personas: buyer (decision maker), user (daily operator), admin (IT/ops)
- Multi-tenant experiments: test per-tenant, not globally
- Consider enterprise buying committee (economic buyer, technical evaluator, champion, blocker)

## Assumption Mapping — B2B Context

| Assumption Category | B2B Examples |
|-------------------|-------------|
| **Desirability** | "Enterprise HR teams want AI-powered screening" |
| **Viability** | "Customers will pay $X/agent/month for voice AI" |
| **Feasibility** | "We can process 1000 concurrent voice calls per tenant" |
| **Usability** | "Non-technical HR managers can configure AI agents" |
| **Scalability** | "Architecture supports 500+ tenants without degradation" |
| **Compliance** | "Voice recordings comply with GDPR per-tenant data isolation" |

## Experiment Design — Tenant-Aware

| Field | B2B Adaptation |
|-------|---------------|
| **Hypothesis** | [Persona]-level hypothesis tied to tenant behavior |
| **Test** | Feature flag per tenant OR beta cohort of 5-10 tenants |
| **Metric** | Per-tenant metric (not global average) |
| **Criteria** | Success across 70%+ of test tenants |
| **Duration** | Longer cycles (2-4 weeks minimum for B2B) |
| **Stakeholders** | Notify customer success before tenant experiments |

**B2B Experiment Types:**
1. **Customer advisory board** — Pitch concept to 5-8 enterprise customers (1 week)
2. **Design partner** — Build with 2-3 customers, shared roadmap influence (4-8 weeks)
3. **Beta program** — 10-20 tenants, feature-flagged, dedicated support (4-8 weeks)
4. **Pilot / POC** — Full implementation for 3-5 enterprise accounts (8-12 weeks)

## B2B Customer Interview Templates

### Buyer Persona (VP/Director)
- "What business outcomes are you trying to achieve this quarter?"
- "How do you currently measure success for [process area]?"
- "What would need to be true for you to approve budget for this?"
- "Walk me through your last software purchase decision."
- "Who else would be involved in evaluating a solution like this?"

### User Persona (Daily Operator)
- "Walk me through your typical workflow for [task]."
- "What takes the most time in your day?"
- "What workarounds have you built for [pain point]?"
- "If you could change one thing about your current tools, what would it be?"
- "How do you handle [specific scenario] today?"

### Admin Persona (IT/Ops)
- "How do you manage user provisioning and access control?"
- "What are your requirements for data isolation and compliance?"
- "How do you evaluate integration complexity for new tools?"
- "What's your biggest concern when onboarding a new SaaS platform?"
- "How do you handle SSO, audit logs, and data export?"

## Continuous Discovery Cadence — B2B

| Cadence | Activity |
|---------|----------|
| **Weekly** | Review tenant analytics, triage support tickets, update experiment tracker |
| **Bi-weekly** | 2-3 customer interviews (rotate buyer/user/admin), update OST |
| **Monthly** | Customer advisory board sync, prioritize opportunities, NRR analysis |
| **Quarterly** | Strategic review, ICP refinement, competitive landscape update, ARR goals |

## Links
- [Continuous Discovery Habits — Teresa Torres](https://www.producttalk.org/)
- [The Mom Test — Rob Fitzpatrick](https://www.momtestbook.com/)
- [JTBD for B2B — Bob Moesta](https://www.therewiredgroup.com/)
