---
name: velents-dev-standards
description: Universal development standards for all Velents developer agents. Defines mandatory pre-writing, verification, and tenant isolation protocols that every agent must follow before marking any task complete.
---

# Velents Developer Standards

Every developer agent in the Velents toolkit MUST follow these protocols. These are not suggestions — they are the quality gates that prevent the 40% rework problem.

---

## Protocol 1: Pre-Writing Codebase Scan (MANDATORY before writing any file)

Before writing any new file, the agent must:

1. **Glob existing similar files** in the same domain:
   ```bash
   # For a new service:
   ls app/Services/[Domain]/
   # For a new React component:
   ls frontend/components/[domain]/
   # For a new migration:
   ls database/migrations/ | grep [keyword]
   ```

2. **Read the closest existing implementation** (1-2 files max):
   - Read an existing Service in the same domain
   - Read an existing Repository for the model pattern
   - Read an existing Controller for the response/middleware pattern
   - Read an existing React page for the component/state pattern

3. **Match the exact patterns found** — don't invent new conventions.

4. **Check for conflicts before creating**:
   - Run `php artisan route:list --path=api/v1/[feature]` — verify no route conflicts
   - Grep migrations for the table name — verify it doesn't already exist
   - Grep permission seeder for the permission name — don't duplicate

---

## Protocol 2: Tenant Isolation Enforcement (MANDATORY for all data operations)

Every query that touches tenant data MUST:

```php
// ✅ CORRECT — always scope to current tenant
$agents = Agent::where('tenant_id', tenant()->id)->get();

// ❌ WRONG — never query without tenant scope
$agents = Agent::all();

// ✅ CORRECT — use global scope if model has one
// (verify the model has TenantScoped or similar trait first)
```

Every mutation that touches tenant data MUST:
- Verify the resource belongs to the current tenant before updating/deleting
- Use `findOrFail` + tenant scope together, never just `find`
- Throw 403 (not 404) when a tenant tries to access another tenant's resource

Every controller method that accepts tenant input MUST:
- Apply `permission:module.action` middleware or check in the method
- Never trust the `tenant_id` from the request body — always use `tenant()->id`

---

## Protocol 3: Self-Verification After Every File (MANDATORY — mark task done ONLY after this passes)

### Laravel (PHP) files
```bash
# After any PHP file:
php -l app/[path/to/file].php

# After a migration:
php artisan migrate --dry-run
php artisan migrate

# After a model/service/repository:
php artisan tinker --execute="new App\[Namespace]\[Class];"

# After routes are added:
php artisan route:list --path=api/v1/[feature] | grep [method]

# After any change — run module tests:
php artisan test --filter=[FeatureName]Test
```

**If any command fails — fix immediately. Do NOT mark the task [X] and move on.**

### Next.js (TypeScript) files
```bash
# After ANY frontend file is written or modified:
cd frontend && npx tsc --noEmit 2>&1 | head -30

# After a component or page:
# Visually verify the component renders the correct data shape
# from the API response defined in plan.md contracts
```

**If tsc reports errors — fix immediately. Do NOT mark the task [X] and move on.**

---

## Protocol 4: velents-ui-inventory Check (MANDATORY for all frontend code)

Before writing ANY React component or UI element:

1. Load the `velents-ui-inventory` skill
2. Search the inventory for:
   - The exact component type you need (table, form, modal, button, badge, etc.)
   - The exact color/variant pattern
   - The exact layout shell/wrapper used on similar pages
3. **USE the existing component** — do not recreate it
4. If you believe a new component is needed: document exactly why no existing component satisfies the requirement

This is the #1 cause of UI mismatch errors. A new component created when an existing one was available creates visual inconsistency that is expensive to fix later.

---

## Protocol 5: Permission Completeness (MANDATORY for all API routes)

Every new API route MUST have:
1. `auth:api` middleware (or equivalent)
2. `permission:module.action` middleware with the exact permission name from the plan
3. The permission name added to the permission seeder (if new)
4. A test that verifies a 403 is returned when the user lacks the permission

Permission naming convention: `module.action` (lowercase, dot-separated)
Examples: `agents.create`, `agents.view`, `agents.delete`, `conversations.view`

---

## Protocol 6: No Task is Done Until Verified

A task is only `[X]` when:
- ✅ Code is written
- ✅ Verification command passed (php -l / tsc / artisan)
- ✅ Relevant tests pass
- ✅ No new TypeScript errors introduced
- ✅ No new PHP syntax errors

"I wrote it and it looks right" is NOT sufficient. The verification command must run and pass.

---

## Protocol 7: Fetch llms.txt Before Using Any Library (MANDATORY)

Before writing code that uses any library in the Velents stack, fetch its llms.txt using WebFetch:

Load the `velents-llms-txt` skill to find the correct URL for each technology.

```
# Example: before writing a Next.js page
WebFetch: https://nextjs.org/llms.txt — "App Router patterns for Next.js 16, server components, data fetching"

# Example: before writing a Laravel service
WebFetch: https://laravel.com/llms.txt — "Service class patterns in Laravel 12, job dispatch"

# Example: before writing TanStack Query hooks
WebFetch: https://tanstack.com/query/latest/llms.txt — "useQuery in v5, isPending vs isLoading"
```

**Why**: Training data has outdated patterns. Tailwind v4, Next.js 16, React 19, TanStack Query v5 all have breaking changes from versions in training data. Fetching ensures you use current APIs.

Do NOT skip this for "simple" code. An incorrect hook pattern or deprecated API will fail silently or produce subtle bugs.

---

## Protocol 8: Load Full Architecture Context Before Writing (MANDATORY)

Before writing any code, load the `velents-feature-map` skill and answer the impact checklist:

1. Which existing features use the same tables/services/events you're about to touch?
2. Does your change alter any Shared Resources (marked 🔴 CRITICAL in the map)?
3. If yes → read the full implementation of the affected features before writing

This prevents the most costly type of bug: a change that works for the new feature but silently breaks an existing one.

**Minimum context to load before any code task**:
- `velents-architecture` skill — read TL;DR block only unless you need module details
- `velents-feature-map` skill — read TL;DR shared-resource table first; full map only if your resource appears in it
- `velents-llms-txt` skill — fetch ONLY the libraries your task will write code for (see scope table in that skill)
- `velents-ui-inventory` skill — (frontend tasks only) read TL;DR component table first; full inventory only if not found

---

## Protocol 10: Token Budget — Load Only What You Need (MANDATORY)

> **Every skill you load and every llms.txt you fetch consumes context. Wasted context = shorter working memory = worse output.**

### Classify your task scope FIRST — then decide what to load

```
CLASSIFY: backend-only | frontend-only | full-stack | integration | fix | pm

backend-only:
  Load: velents-architecture (TL;DR), velents-backend, velents-multitenancy, velents-feature-map (TL;DR)
  llms.txt: Laravel only (+ Spatie/Reverb/Tenancy if specifically used)
  Skip: velents-ui-inventory, velents-ui-prototype, velents-frontend, velents-realtime

frontend-only:
  Load: velents-architecture (TL;DR), velents-frontend, velents-ui-inventory (TL;DR), velents-feature-map (TL;DR)
  llms.txt: Next.js + React + whatever UI library is used (NOT ElevenLabs, NOT Laravel)
  Skip: velents-backend, velents-multitenancy, velents-auth-rbac, velents-testing (PHPUnit sections)

full-stack:
  Load: all, but backend skills first, frontend skills when frontend phase starts
  llms.txt: backend libs when writing backend, frontend libs when writing frontend

integration:
  Load: velents-architecture (TL;DR), velents-integrations, velents-core-flows
  llms.txt: only the specific service being integrated (ElevenLabs OR LiveKit, not both unless both used)
  Skip: velents-ui-inventory, velents-ui-prototype (unless config UI needed)

fix (targeted bug):
  Load: only the skill for the specific module being fixed
  llms.txt: only if the bug involves a library API that may have changed
  Skip: everything else — you know what file you're touching

pm (no code):
  Load: pm-discovery-frameworks, pm-strategy-frameworks, or pm-execution-frameworks
  Skip: all code skills, all llms.txt
```

### TL;DR-First Rule for Heavy Skills

All heavy skills have a `## TL;DR` block at the top. Always read the TL;DR first. Only read the full skill if:
- The TL;DR doesn't contain what you need
- You need specific method signatures, table columns, or component props

### Context Passing — Keep It Slim

When passing context between steps, include ONLY:
- File paths created/modified (not file contents)
- Class names, method signatures, table names, route paths decided so far
- Specific constraints for the next step

Do NOT pass: full file contents, full spec text, full plan text — the next agent can read these from disk.

---

## Protocol 9: Always Ask — Never Assume (MANDATORY)

> **The single most expensive mistake is implementing the wrong thing with confidence. An assumption that goes unchallenged becomes a guaranteed rework.**

### When to Stop and Ask (non-negotiable)

Stop and ask the human — do NOT proceed — when any of the following is true:

1. **Design decisions**: Any choice about UI layout, user flow, data model shape, API design, or feature behavior that has more than one reasonable answer. Do not pick one and build. Present the options.

2. **Unclear requirements**: Any requirement that can be interpreted in 2+ different ways. Quote the ambiguous text and ask which interpretation is correct.

3. **Scope uncertainty**: You're not sure if something is in scope. Explicitly ask: "Should this include X?"

4. **Missing information you cannot find in the codebase**: If you need a decision (config value, external API key, environment, credential, third-party account) that isn't discoverable from the code, ask.

5. **Conflicting signals**: If the spec, plan, and codebase disagree on how something should work, stop. Don't pick the one you prefer — surface the conflict.

### How to Ask

When stopping to ask, always:
- State exactly what you're uncertain about (quote the ambiguous part)
- Give 2–3 concrete options with a brief trade-off for each
- State which option you'd recommend and why (never "I don't know, you decide")
- Ask only 1 question at a time — batch related questions into one message

```
I need a decision before continuing:

**Question**: [one clear question]

**Option A**: [description] — Pro: [benefit], Con: [drawback]
**Option B**: [description] — Pro: [benefit], Con: [drawback]
**Option C**: [description] — Pro: [benefit], Con: [drawback]

**My recommendation**: Option B because [reason].

Which should I proceed with?
```

### What NOT to Do

- ❌ Assume the simpler option and build
- ❌ Pick the option that requires less work
- ❌ Document the assumption and proceed anyway
- ❌ Ask a vague question like "Does this look right?"
- ❌ Ask 5 questions at once
- ❌ Build a guess and fix it later if wrong

### Plan Mode Is Mandatory Before Implementation

Every code task requires a plan approved by the human before writing a single line:

1. Write the plan (spec → plan → tasks)
2. **Show the plan to the human** — summarize what will be built, key design decisions made, files that will change
3. Wait for explicit approval ("yes", "proceed", "looks good")
4. Only then begin implementation

Do NOT start writing code because "the spec is clear enough." The plan step is not optional.
