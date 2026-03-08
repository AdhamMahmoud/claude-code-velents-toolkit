---
name: velents-feature-map
description: Impact map of all existing Velents features — tables, routes, services, and events each feature owns. Load this before writing any code to identify what existing features your change might break. Never write code that touches a shared resource without reading this map first.
---

# Velents Feature Impact Map

## THE RULE: Read Before You Touch

Before writing any code, scan this map for:
1. Which existing features use the same tables you're modifying
2. Which existing features call the same services you're changing
3. Which existing features broadcast or listen to the same events
4. Which frontend pages consume the API endpoints you're modifying

If your change touches anything owned by another feature → read that feature's implementation before writing.

---

## Feature Map

### 1. Agent CRUD
**Owns**: `agents` table (tenant), `agent_tools` pivot, `agent_channels` pivot, `knowledge_bases` table, `schemas` table
**Key service**: `AgentService` — `create()`, `update()`, `delete()`, `activate()`, `deactivate()`
**Key routes**: `GET|POST /api/v1/agents`, `GET|PUT|DELETE /api/v1/agents/{id}`
**Frontend page**: `/agents` list, `/agents/create`, `/agents/[id]` (7-tab builder)
**Events broadcast**: `AgentStatusChanged`, `AgentUpdated`
**⚠️ Risk if touched**: Changing `agents` table affects Conversations, Calls, Analytics, Payment — all foreign key to `agent_id`

---

### 2. Text Conversations
**Owns**: `conversations` table (tenant), `conversation_logs` table, `conversation_analysis` table
**Key service**: `ConversationService` — `createByAgent()`, `send()`, `end()`
**External call**: `TextAgent::start_session()`, `TextAgent::send_message()`, `TextAgent::end_session()`
**Key routes**: `POST /api/v1/Conversations`, `POST /api/v1/Conversations/{id}/Send`, `POST /api/v1/Conversations/{id}/End`
**Frontend page**: `/conversations`, `/conversations/[id]`
**Events broadcast**: `.Message`, `.Update`, `.conversation` on `{conversation_public_id}` channel
**Payment trigger**: `Payment::Pay()` on every message
**⚠️ Risk if touched**: Changing conversation flow breaks WhatsApp delivery, Genesys forwarding, human escalation, extraction, analytics dispatch

---

### 3. Voice Calls
**Owns**: `calls` table (tenant), `call_logs` table
**Key service**: `CallService` — `createCall()`, pipeline dispatch
**3 pipelines**: Flash (ElevenLabs), Normal (LiveKit), Fast (CallGateway)
**State machine**: `onQueue → pulled → pushed → dialing → OnCall → active → ended/done`
**Key routes**: `POST /api/v1/Calls`, `POST /api/v1/Calls/{id}/End`
**External calls**: ElevenLabs, LiveKit, CallGateway (pipeline-dependent)
**Post-call**: `S2SPostProcessing::extractData()` → `AnalyzeCall` job → GeminiJudge
**⚠️ Risk if touched**: Any state machine change breaks batch campaigns, S2S extraction, and analytics

---

### 4. Quality Analytics
**Owns**: `conversation_analysis` table (tenant) — shared with Conversations and Calls
**Key service**: `AnalyticsService`, `GeminiJudge`
**6 metrics**: scope_adherence, extraction_completeness, conversation_quality, customer_satisfaction, resolution_effectiveness, overall_score
**Key routes**: `GET /api/v1/Analytics`, `GET /api/v1/Analytics/conversations`, `GET /api/v1/Analytics/calls`
**Frontend page**: `/analytics` dashboard with Recharts
**⚠️ Risk if touched**: `conversation_analysis` columns are used by both Conversations and Calls modules

---

### 5. Payment / Credit Billing
**Owns**: `payment_plans` table (tenant), `payment_plan_logs` table, `payment_plan_templates` table (central)
**Key service**: `Payment::Pay()` — atomic credit deduction
**Called by**: Every Conversation message send, every Call start
**Key routes**: `GET /api/v1/Payment/plans`, `POST /api/v1/Payment/links`
**⚠️ Risk if touched**: Any change to `Payment::Pay()` affects Conversations AND Calls — both deduct credits per event

---

### 6. Team Management
**Owns**: `staff` table (tenant), `roles` table, `permissions` table (via Spatie)
**Key service**: Spatie Permission + custom Management module
**Key routes**: `GET|POST /api/v1/Management/staff`, `/api/v1/Management/roles`, `/api/v1/Management/invitations`
**Frontend page**: `/settings/team`, `/settings/roles`
**⚠️ Risk if touched**: Changing permission names breaks ALL permission middleware across every route

---

### 7. Integrations
**Owns**: `integrations` table (tenant), `phone_numbers` table
**Submodules**: WhatsApp (Meta API), Genesys Cloud (9 regions), phone number management, calendar sync
**Webhooks**: All in `routes/Webhook.php` — each provider has its own handler
**⚠️ Risk if touched**: WhatsApp and Genesys webhooks are inbound — changing payload handling breaks live customer conversations

---

### 8. Audit Log
**Owns**: `audit_logs` table (tenant) — append-only
**Writes to**: Every significant mutation in Agents, Conversations, Calls, Management
**⚠️ Risk if touched**: Never modify existing audit log entries or schema columns that store `before`/`after` state

---

## Shared Resources (Touch with Extreme Caution)

| Resource | Used By | Risk Level |
|---|---|---|
| `agents` table | Conversations, Calls, Analytics, Payment | 🔴 CRITICAL |
| `conversation_analysis` table | Conversations, Calls, Analytics | 🔴 CRITICAL |
| `Payment::Pay()` | Conversations, Calls | 🔴 CRITICAL |
| Permission names (Spatie) | ALL routes via middleware | 🔴 CRITICAL |
| `routes/tenant.php` | All tenant business logic | 🟠 HIGH |
| `AgentService::activate()` | Agent CRUD, Conversations, Calls setup | 🟠 HIGH |
| WebSocket channels | Conversations frontend, Calls frontend | 🟠 HIGH |
| `routes/Webhook.php` | WhatsApp, Genesys, LiveKit, ElevenLabs | 🟠 HIGH |
| `staff` table | Auth, Team management, Audit log | 🟡 MEDIUM |

---

## Pre-Code Impact Checklist

Before writing any code, answer these questions:

- [ ] Does my change add/modify/remove columns from a table in the Shared Resources list?
- [ ] Does my change alter a method signature in `AgentService`, `ConversationService`, `CallService`, or `Payment`?
- [ ] Does my change rename or remove a Spatie permission?
- [ ] Does my change modify a webhook handler in `routes/Webhook.php`?
- [ ] Does my change alter the shape of an API response that the frontend currently consumes?
- [ ] Does my change add a new `broadcast()` event on an existing channel?

**If YES to any** → read the full implementation of the affected feature before writing a single line.
