---
name: review-pr
description: Security-first PR review with severity tiers before merge
user-invocable: true
---

Review the current PR or code changes with security-first, production-grade checks.

$ARGUMENTS

Route to the **pr-reviewer** agent. Include the full diff context and apply all 4 severity tiers (CRITICAL → HIGH → MEDIUM → LOW) with VelentsAI-specific checks for tenancy, payment, and voice pipeline safety.
