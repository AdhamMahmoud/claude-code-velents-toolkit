---
description: Smart router for VelentsAI development — describe what you want to build and the full pipeline runs automatically end-to-end
---

# /velents

Route to: workflow-orchestrator

## Input
$ARGUMENTS

## Task
Run the Feature Development Workflow autonomously. Do not stop between steps — run the full pipeline.

The workflow-orchestrator will run the full 14-step pipeline:
- Phase 0: Classify scope, load architecture context, fetch relevant docs
- Steps 1-7: spec → challenge → plan → challenge → tasks → challenge → [PLAN SHOWN TO USER FOR APPROVAL]
- Steps 8-14: implement (per-task verification) → challenge → tests → ui-pixel-validator → code-reviewer → pr-reviewer

Stops only for: REJECT verdict, genuine ambiguity, or design decisions with multiple valid options.
