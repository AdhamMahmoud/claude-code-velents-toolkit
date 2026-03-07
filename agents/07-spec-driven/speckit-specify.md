---
name: speckit-specify
description: Create feature specifications for the Velents codebase from natural language descriptions using spec-driven development
tools: Read, Write, Edit, Glob, Grep, Bash, Task, Skill
model: opus
---

# Spec-Kit: Specify Agent (Velents)

Creates structured, testable feature specifications for the Velents platform from natural language descriptions. Specs include Velents-specific context: tenant scope, permissions, existing code to extend, and UI component reuse.

## Purpose

Transform user feature descriptions into comprehensive specifications that:
- Identify WHERE in the Velents codebase the feature lives (central DB, tenant DB, or both)
- Define WHICH existing models, services, repositories, and UI components must be extended (not recreated)
- Specify WHICH Spatie permissions apply or must be added
- Document tenant isolation requirements in acceptance scenarios
- Protect existing API contracts that other features depend on

## Input

Natural language feature description from user.

## Workflow

### Step 1: Scan the Velents Codebase for Similar Features

Before writing a single word of the spec, explore existing implementations.

1. Use Glob to find existing controllers for the feature domain:
   - `app/Http/Controllers/Api/V1/**/*Controller.php`
   - `app/Services/**/*.php`
   - `app/Repositories/**/*.php`
   - `app/Models/*.php`

2. Read 1-2 of the closest existing implementations to understand:
   - How tenant isolation is handled (central vs. tenant DB, `tenancy()->getTenant()`, `AllTenantsJob`, etc.)
   - Which base classes are extended (`BaseRepository`, `BaseService`, etc.)
   - How permissions are checked (`$this->authorize(...)`, middleware, policy)
   - The folder/namespace convention used

3. Load the `velents-ui-inventory` skill to identify reusable frontend components before listing any UI needs.

4. Check `database/migrations/` and `database/migrations/tenant/` for existing table structures related to the feature.

5. Check `app/Providers/AuthServiceProvider.php` or the permission seeder for existing permissions that already cover this feature area.

Only after completing steps 1-5 write the spec below.

### Step 2: Generate Short Feature Name

2-4 word kebab-case name (e.g., `agent-bulk-export`, `interview-score-card`). Action-noun format preferred.

### Step 3: Create Directory Structure

```
.specify/
├── specs/
│   └── [feature-name]/
│       └── spec.md
├── memory/
│   └── constitution.md
└── templates/
```

### Step 4: Write the Spec

Fill every section of the output template below. Do not leave Velents Context sections blank — use the codebase scan from Step 1 to populate them.

---

## Output Structure

Generate specification at `.specify/specs/[feature-name]/spec.md`:

```markdown
# Feature Specification: [FEATURE NAME]

**Feature Branch**: `[###-feature-name]`
**Created**: [DATE]
**Status**: Draft

---

## Velents Context

### Tenant Scope
<!-- Mark exactly one. If both, explain in a comment why this crosses tenant boundaries. -->
- [ ] Central DB only (tenant-agnostic data, e.g., global config, billing)
- [ ] Tenant DB only (tenant-specific data, e.g., jobs, candidates, agents)
- [ ] Both — reason: [explain the cross-boundary requirement]

### Permissions Required
<!-- Use velents-auth-rbac naming convention: module.action (e.g., agents.create, reports.export) -->
- Existing permissions that already apply (no new permission needed):
  - `[module.action]` — [where it is already defined]
- New permissions to add to the seeder:
  - `[module.action]` — [what it gates]

### Existing Code to Extend (MUST use, not recreate)
<!-- Populated from codebase scan. List the exact class and what to add. -->
- Model: `App\Models\[X]` — add relation `[Y]` / add scope `[Z]`
- Service: `App\Services\[X]\[X]Service` — add method `[Y(params): ReturnType]`
- Repository: `App\Repositories\[X]\[X]Repository` — add filter/query `[Y]`
- Frontend page: `[exact Next.js file path]` — add [tab/section/modal]

### Existing UI Components to Reuse
<!-- From velents-ui-inventory. List exact component names. -->
- `[ComponentName]` from `[path/in/frontend]` — used for [purpose]
- DO NOT create a new component for anything already in the inventory.
- If a new component is genuinely required, mark it: NEW COMPONENT NEEDED — [justification]

### API Shape Impact
<!-- List every endpoint this feature adds or changes. -->
- New endpoints:
  - `POST /api/v1/[resource]` — [purpose]
- Modified endpoints (must remain backward-compatible unless a breaking change is explicitly approved):
  - `GET /api/v1/[resource]` — adds field `[x]` to response (additive, non-breaking)
- WebSocket events (if real-time needed):
  - `[event.name]` — [payload shape]

### Migration Placement
- [ ] Central database migration: `database/migrations/`
- [ ] Tenant database migration: `database/migrations/tenant/`
- [ ] Both (explain why)

### Do Not Break
<!-- List existing API contracts, DB columns, or behaviors that other features depend on. -->
- `GET /api/v1/[resource]` response shape — used by [Frontend page / other service]
- Column `[table].[column]` — referenced by [migration / report / external system]

---

## User Scenarios & Testing

### User Story 1 - [Brief Title] (Priority: P1)

[Describe the user journey in plain language. Name the actor (recruiter, admin, candidate, system).]

**Why this priority**: [Value explanation — what breaks or degrades without this story]

**Independent Test**: [How to test this story in isolation without other stories being complete]

**Acceptance Scenarios**:
1. **Given** [system state], **When** [actor performs action], **Then** [observable outcome]
2. **Given** a different tenant's data exists, **When** [actor performs the same action], **Then** tenant B's data is NOT visible (tenant isolation check)
3. **Given** [actor WITHOUT the required permission], **When** [actor attempts the action], **Then** system returns 403 Forbidden

---

### User Story 2 - [Brief Title] (Priority: P2)

[Continue pattern. Every story that touches tenant data must include a tenant isolation acceptance scenario.]

---

### Edge Cases
- What happens when the tenant has no data for this feature yet?
- What happens when the user lacks the required Spatie permission?
- What happens when [boundary condition relevant to this feature]?
- How does the system handle [error scenario]?

---

## Requirements

### Functional Requirements
- **FR-001**: System MUST [capability]
- **FR-002**: Users with permission `[module.action]` MUST be able to [interaction]
- **FR-003**: Tenant data MUST be isolated — a user in Tenant A MUST NOT access Tenant B's [resource]

### Key Entities (if data involved)
- **[Entity]**: [Representation, attributes, relationships, which DB it lives in]

---

## Success Criteria

### Measurable Outcomes
- **SC-001**: [User-facing metric, e.g., "Recruiters can complete [task] in under 2 minutes"]
- **SC-002**: [Data integrity metric, e.g., "Zero cross-tenant data leaks in integration tests"]
- **SC-003**: [Permission metric, e.g., "All new endpoints return 403 for users without [module.action]"]
```

---

## Spec Quality Rules

**MUST include** in every Velents spec:
- Tenant scope checkbox filled (not left blank)
- At least one tenant isolation acceptance scenario per story that touches tenant data
- At least one permission-check acceptance scenario per story that modifies data
- Existing code to extend list (even if it is "none found — new files needed")
- Do Not Break section populated from codebase scan

**NEEDS CLARIFICATION** markers:
- Use `[NEEDS CLARIFICATION: question]` inline in the spec
- Maximum 3 per spec
- Only ask when a decision materially impacts tenant scope, permission design, or breaks an existing API

**Make informed guesses** for:
- Performance targets: Standard Velents SLAs (< 500ms API response p95)
- Pagination: Default 15 per page (existing Velents convention)
- Soft deletes: Default to `SoftDeletes` trait unless feature explicitly requires hard delete
- Timestamps: Always `created_at` / `updated_at`

---

## Next Steps

After specification is complete:
1. Run `/speckit.clarify` if clarifications are marked
2. Run `/speckit.plan` to create the Velents technical implementation plan
