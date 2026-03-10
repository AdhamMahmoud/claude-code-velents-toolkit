---
name: velents
description: Smart router for VelentsAI development — describe what you want to build and the full pipeline runs automatically end-to-end
user-invocable: true
---

# /velents

Use the Task tool to invoke the `velents-toolkit:workflow-orchestrator` agent with the following prompt:

```
## Feature Request
$ARGUMENTS

Run the Feature Development Workflow autonomously.
Do not stop between steps — run the full pipeline.
```

**IMPORTANT**: You MUST call the Task tool immediately with `subagent_type: velents-toolkit:workflow-orchestrator`. Do not print a routing decision. Do not narrate what you plan to do. Just invoke the agent now.

The workflow-orchestrator will autonomously run the full pipeline:
- Phase 0: Classify scope, load architecture context, fetch relevant docs, read prototype (if provided)
- Step 1-7: spec → challenge → plan → challenge → tasks → challenge → [PLAN SHOWN TO USER FOR APPROVAL]
- Step 8-14: implement (per-task verification) → challenge → tests → ui-pixel-validator → code-reviewer → pr-reviewer

The only times it will stop and ask you:
1. A REJECT verdict from the adversarial challenger (fundamental flaw that needs your direction)
2. One genuine ambiguity at the start that cannot be resolved from the codebase
3. A design decision with multiple valid options — it presents choices, not assumptions

Everything else — REVISE loops, test failures, TypeScript errors, migration issues — is handled and retried automatically.
