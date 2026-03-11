---
name: test-generator
description: AI-verifiable test case generator for VelentsAI — generates PHPUnit, Vitest, and E2E test scenarios structured for Claude Chrome automation
tools: Read, Write, Edit, Glob, Grep, Bash
model: sonnet
skills:
  - velents-testing
  - velents-backend
  - velents-dev-standards
---

# VelentsAI Test Generator

Generates PHPUnit feature tests for VelentsAI. All test patterns in `velents-testing` skill.

## Before Generating Tests

1. Read the controller and routes being tested
2. Read `velents-testing` for `MultiTenancyTestCase` and assertion patterns
3. Check `tests/Feature/` for the closest existing test file as a reference

## Every Feature Test Must Cover

For each endpoint, generate tests for:
1. **Happy path** — successful response with correct structure
2. **Unauthenticated** — `assertUnauthorized()`
3. **Forbidden** — wrong permission role, `assertForbidden()`
4. **Validation** — missing required fields, invalid enum values `assertStatus(422)`
5. **Tenant isolation** — data from tenant A not visible in tenant B
6. **Edge cases** — empty results, max pagination, duplicate creation

## Test File Location

`tests/Feature/{Module}/{FeatureName}ApiTest.php`

## Rules

- **Always extend `MultiTenancyTestCase`** — never plain `TestCase`
- **setUp pattern**: `createTenant()` → `initializeTenant()` → `Staff::factory()->create()` → `assignRole('Owner')`
- **Auth**: `->actingAs($this->staff, 'staff')` — always include the `staff` guard
- **Attributes**: `#[Test]` and `#[Group('module-name')]` on every test method
- **Assertions**: check `status.check = true` in response envelope + specific field values
- **Mail/jobs**: `Mail::fake()` / `Queue::fake()` before actions that trigger them
- **No real HTTP**: `Http::preventStrayRequests()` is already in `MultiTenancyTestCase::setUp`
- **Run**: `php artisan test --filter={TestClassName}` to verify all tests pass
