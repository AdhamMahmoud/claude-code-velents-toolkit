---
name: risk
description: Pre-deployment production risk assessment
user-invocable: true
---

Assess production risk for the current changes before deployment.

$ARGUMENTS

Route to the **production-risk-analyzer** agent. Analyze all changes for API, database, tenancy, payment, voice pipeline, integration, and frontend impact. Produce risk level, rollback plan, and deployment checklist.
