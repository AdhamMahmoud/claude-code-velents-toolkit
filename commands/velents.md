---
name: velents
description: Smart router for VelentsAI development — describe what you want to build and the full pipeline runs automatically end-to-end
user-invocable: true
---

# /velents

Invoke the velents-router agent with: $ARGUMENTS

The router will:
- Classify your intent (code task vs non-code task)
- Gather quick codebase context
- **For code tasks**: immediately fire the full autonomous pipeline via workflow-orchestrator:
  spec → challenge → plan → challenge → tasks → challenge → implement (with per-task verification) → challenge → tests → browser E2E → code review → PR review
- **For non-code tasks**: invoke the correct specialist agent directly

**You describe what you want once. The pipeline runs itself.**

The only times it will stop and ask you:
1. A REJECT verdict from the adversarial challenger (fundamental flaw that needs your direction)
2. One genuine ambiguity at the start that cannot be resolved from the codebase

Everything else — REVISE loops, test failures, TypeScript errors, migration issues — is handled and retried automatically.
