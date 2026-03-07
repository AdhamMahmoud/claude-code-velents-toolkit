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
