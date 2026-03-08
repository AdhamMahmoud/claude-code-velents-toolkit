---
name: velents-architecture
description: VelentsAI system architecture overview — 5 layers, tech stack, data flows, module structure, integration map
user-invocable: false
---

# VelentsAI System Architecture

## TL;DR — Read This First, Go Deeper Only If You Need It

```
Stack:      Laravel 12 (PHP 8.4) backend | Next.js 16 (TypeScript) frontend
Tenancy:    stancl/tenancy v3 — database-per-tenant (each tenant = separate PostgreSQL DB)
Auth:       Laravel Passport (OAuth2) + Sanctum (API tokens) + Spatie (64 permissions, 12 categories)
Real-time:  Laravel Reverb (WebSocket) — frontend uses Laravel Echo
Queue:      Redis + Laravel Horizon | Background: Kafka for event streaming
Storage:    AWS S3 | Search: OpenSearch

Entry points:
  API:      /api/v1/* (Laravel) — all routes require auth:api + tenant.aware middleware
  Frontend: app/(dashboard)/[tenant]/* (Next.js App Router)
  Webhooks: /webhooks/* (no auth, tenant resolved from payload)

DB split:
  Central:  tenants, domains, system config, billing, global permissions
  Tenant:   agents, conversations, calls, analytics, knowledge_bases, schemas

Key external:  ElevenLabs (voice TTS/STT) | LiveKit (WebRTC rooms) | CallGateway (PSTN)
               WhatsApp Business API | Genesys Cloud (9 regions) | Gemini (quality scoring)
```

**Only read the full skill below if you need specific module details, table names, or service patterns.**

---

# VelentsAI System Architecture

## What is VelentsAI?

A **multi-tenant SaaS platform** for building, deploying, and managing AI-powered agents that conduct customer interactions over text and voice channels. Organizations (tenants) create AI agents, equip them with tools and knowledge, deploy them across channels (web chat, WhatsApp, phone, Genesys Cloud), and receive structured data extraction + quality analytics from every interaction.

---

## Tech Stack

### Backend: `velentsAgents` -- Laravel 12 (PHP 8.2+)

- **Multi-Tenancy**: stancl/tenancy ^3.9 (database-per-tenant)
- **Auth**: Laravel Passport (OAuth2) + Sanctum (API tokens) + Spatie Permission (64 perms, 12 categories)
- **Database**: PostgreSQL 16 (central + per-tenant databases)
- **Cache/Queue**: Redis via predis
- **Event Streaming**: Kafka (laravel-kafka ^2.4)
- **Search**: OpenSearch
- **Storage**: AWS S3
- **WebSockets**: Laravel Reverb ^1.6
- **Testing**: PHPUnit ^11.5.3

### Frontend: `agent-hub` -- Next.js 16 (TypeScript 5.9)

- **Framework**: Next.js 16.1.3 (App Router), React 19.2.3
- **Styling**: Tailwind CSS ^4.1.10
- **UI Components**: shadcn/ui (46+ Radix primitives)
- **Global State**: Zustand ^5.0.5
- **Server State**: TanStack React Query ^5.90.12
- **Tables**: TanStack React Table ^8.21.3
- **Forms**: React Hook Form ^7.58.1 + Zod ^3.25.67
- **Real-time**: Laravel Echo ^2.2.6 + Pusher.js ^8.4.0
- **i18n**: next-intl ^4.5.5 (EN + AR with RTL)
- **Rich Text**: Tiptap ^2.22.3
- **Charts**: Recharts ^3.6.0
- **Monitoring**: Sentry ^10.29.0

### External AI/ML Services

| Service | Purpose |
|---------|---------|
| TextAgent (text-agent.velents.ai) | GPT-5 conversational AI engine |
| VoiceAgent (voice-agent.velents.ai) | Voice KB management (vector search) |
| ElevenLabs | Voice synthesis + flash voice agents |
| LiveKit | WebRTC real-time call infrastructure |
| CallGateway | VoIP/SIP phone call routing |
| Gemini 2.0 Flash | Quality analysis and judgment |
| S2SPostProcessing | Speech-to-text data extraction |
| VelentsIntegrations | WhatsApp (Meta API) + Payment |
| GenesysCloud | Enterprise contact center (9 regions) |

---

## 5-Layer Architecture

```
Layer 1: CLIENT (Browser)
  Next.js 16 App Router
  Zustand (auth) + TanStack Query (server state)
  ApiClient singleton (dynamic tenant URL)
  Laravel Echo (WebSocket)
  shadcn/ui + Tailwind CSS 4
  URL: https://{tenant}.agent-hub.velents.ai

Layer 2: API GATEWAY (Laravel Middleware)
  ForceJsonResponse -> SetLanguage -> CORS -> AddHeaders
  -> InitializeTenancyByDomain -> PreventCentralAccess
  -> auth:staff -> permission:* -> rate limiting

Layer 3: BUSINESS LOGIC (Modular Domains)
  Controllers -> Repositories -> Services -> Jobs -> Events
  Modules: Agents, Conversations, Calls, Analytics,
           Payment, Management, Integration, Phones, Core, AuditLog

Layer 4: DATA
  PostgreSQL (central + N tenant DBs)
  Redis (cache + queues)
  AWS S3 (files)
  Kafka (event streaming)

Layer 5: EXTERNAL AI SERVICES
  TextAgent | VoiceAgent | ElevenLabs | LiveKit
  CallGateway | Gemini | S2S | VelentsIntegrations
```

---

## Multi-Tenancy

```
Registration -> Create Tenant (central DB)
            -> Create domain (testcompany.localhost)
            -> Create tenant DB: tenant_{id}_boddy
            -> Run 40+ tenant migrations
            -> Seed roles/permissions + first staff

Request Flow:
  Browser -> testcompany.velents.ai
           -> InitializeTenancyByDomain middleware
           -> Switch DB to tenant_{id}_boddy
           -> Cache prefixed with tenant ID
           -> Filesystem scoped per tenant
           -> Queue jobs tagged with tenant context
```

**Route Isolation**:

| Route File | Domain | Auth | Purpose |
|------------|--------|------|---------|
| `routes/web.php` | Central | None | Registration, FindMyTenant |
| `routes/tenant.php` | Subdomain | `auth:staff` | All business APIs |
| `routes/Webhook.php` | Self-resolving | Token/signature | External callbacks |
| `routes/Ml.php` | Self-resolving | Service auth | AI service callbacks |
| `routes/plugins.php` | Subdomain | Agent token | Plugin/external API |

---

## Agent Data Model

```
Agent
  id, name, prompt, status, type, pipeline, webhook
  text: { can_start, first_message }
  voice: { speed, tone, accent, gender, allow_interruptions, flash_language }

  KnowledgeBases (1:many) -- files, URLs, auto-refresh
  Schemas (1:many) -- extraction fields (string|number|boolean|list|enum|object)
  Tools (many:many via AgentTool) -- built-in + dynamic, per-tool auth
  Channels (many:many via AgentChannel) -- WhatsApp, Phone, Genesys, etc.
  Conversations (1:many) -> ConversationLogs, ConversationAnalysis
  Calls (1:many) -> CallLogs, ConversationAnalysis
  Batches (1:many) -- CSV campaigns with scheduling
```

**Enums**: status (active|unactive|draft|archived), type (text|voice), pipeline (normal|fast|flash), channel_type (webChat|WhatsApp|Voice|Telegram|Messenger|Instagram|Slack|GenesysCloud).

---

## Agent Lifecycle

```
CREATION       CONFIGURATION      DEPLOYMENT       EXECUTION         ANALYSIS
POST /Agent -> 7-Tab Builder ->   status: active -> Dispatch() ->    GeminiJudge
status: draft  Setup, Knowledge   Sync to          Text: TextAgent   6 metrics (0-10)
               Schema, Tools      ElevenLabs/      Voice: LiveKit/   overall_score
               Channels           VoiceAgent       CallGateway/      scope_adherence
                                  Generate tokens  ElevenLabs        extraction_completeness
                                                   WhatsApp          conversation_quality
                                                   Genesys           customer_satisfaction
                                                                     resolution_effectiveness
```

---

## Data Flow -- Text Conversation

```
POST /Conversations -> createByAgent()
  TextAgent::start_session(agent_id, persona, tools, extraction_schema, llm_config)
  -> session_id
  Create Conversation, ConversationLog, Payment::Pay()
  Broadcast via Reverb

POST /Conversations/{id}/Send
  TextAgent::send_message(session_id, message)
  -> { response, session_status, extracted_data }
  Payment::Pay(), update extraction, broadcast response
  If WhatsApp: SendFreeFormMessage
  If Genesys: SendMessage
  If human_pending: escalate
  SendResult webhook

POST /Conversations/{id}/End
  TextAgent::end_session(session_id)
  status -> ended, SendResult webhook
  Dispatch AnalyzeConversation -> GeminiJudge (6 metrics)
```

---

## Data Flow -- Voice Call

```
POST /Calls -> createCall()
  Create Call (status: onQueue), dispatch Init job

Init resolves pipeline:
  Flash -> ElevenLabs outbound
  Normal -> LiveKit (WebRTC)
  Fast -> CallGateway (VoIP/SIP)

State machine:
  onQueue -> pulled -> pushed -> dialing -> OnCall -> active -> ended/done
  Error: no_response, Busy, Invalid_Number, failed, expired

Completion:
  S2SPostProcessing::extractData()
  Update extraction, result, duration
  If batch: Batch/statistics
  AnalyzeCall -> GeminiJudge -> ConversationAnalysis
```

---

## Tools/Skills Architecture

**5 Built-in Tools**: calendar, payment, human_in_the_loop, knowledge_base, memory.

**Dynamic Tools**: Generated from file upload / text / URL via TextAgent + Gemini. Per-tool auth: API key, Bearer, Basic, OAuth2, JWT.

**Runtime Resolution**:
```
Agent dispatched -> usedTools()
  Iterate active tools (AgentTool where is_active=true)
  Built-in? -> specific handler (calendar loads slots, memory sets customer_id)
  Dynamic? -> include definition + auth config
  Result: tools[] array sent to TextAgent start_session
```

---

## Auth Flow

```
1. User enters email -> POST /Auth/FindMyTenant (central)
   Lookup across all tenant DBs
   Broadcast tenant list via Reverb (keyed by request_id)

2. User selects tenant, enters password
   POST /Auth/loginByTenant ({tenant_id}.domain)
   Return Passport OAuth2 tokens (access + refresh)

3. Frontend stores tokens in localStorage + Zustand
   All subsequent calls: Bearer token to tenant subdomain
   ApiClient constructs URL: https://{tenant_id}.{API_DOMAIN}
```

---

## Real-time Events

| Channel | Event | Purpose |
|---------|-------|---------|
| `{request_id}` | `.login` | Tenant list during FindMyTenant |
| `{conversation_public_id}` | `.Message` | New message (user or agent) |
| `{conversation_public_id}` | `.Update` | Status/extraction change |
| `{conversation_public_id}` | `.conversation` | General broadcast |

All public channels via Laravel Reverb (port 8080), Echo + Pusher.js client.

---

## Backend Module Structure

```
app/
  Agents/          -- CRUD, knowledge, schemas, tools, channels
  Conversations/   -- Text conversation lifecycle
  Calls/           -- Voice call lifecycle
  Analytics/       -- Gemini-powered quality analysis
  Payment/         -- Credit billing
  Management/      -- Team, roles, permissions, invitations
  Integration/     -- External services, webhooks (InBound/*)
  Phones/          -- Phone number management
  Core/            -- Base classes, traits, casts, shared support
  AuditLog/        -- Immutable audit trail
```

Each module follows: `Controllers/ -> Repositories/ -> Models/ -> Requests/ -> Resources/ -> Enums/ -> Jobs/`

---

## Frontend Architecture Patterns

| Pattern | Implementation |
|---------|---------------|
| API Client | `lib/api/client.ts` -- singleton, dynamic tenant URL, fetch-based |
| Services | `lib/api/services/*.service.ts` -- static methods per domain |
| Auth State | `lib/auth/auth-store.ts` -- Zustand (staff, role, permissions) |
| Server State | TanStack React Query via `lib/api/useQuery.tsx` wrappers |
| Forms | React Hook Form + Zod schemas |
| WebSocket | `lib/echo.ts` -- Laravel Echo + Reverb/Pusher |
| Routing | `app/[locale]/...` -- next-intl locale segment |
| Components | `components/ui/` (shadcn) + feature components per page |

---

## Feature Map

- **Agents**: CRUD with draft/active/archived lifecycle, text + voice types, 3 pipelines
- **Knowledge Base**: File/URL upload, auto-refresh, ElevenLabs/VoiceAgent sync
- **Data Collection**: Typed extraction schemas, real-time extraction, CSV export
- **Conversations**: Real-time messaging, human escalation, file uploads, webhooks
- **Calls**: 3 pipelines, state machine, post-call extraction, batch campaigns
- **Analytics**: 6 Gemini-powered quality metrics, trends, backfill
- **Payment**: Credit-based billing (USD/SAR/EGP), per-message deduction
- **Team**: Roles (Owner/Admin/Operator/Viewer + custom), 64 permissions, invitations
- **Audit Log**: Immutable action log with before/after state
- **Integrations**: WhatsApp (Meta), Genesys Cloud (9 regions), phone numbers, calendar
- **i18n**: English + Arabic with full RTL support

---

## Key Configuration Files

| File | Purpose |
|------|---------|
| `config/tenancy.php` | Multi-tenancy bootstrappers, tenant model |
| `config/services.php` | External service URLs and credentials |
| `config/permission.php` | Spatie permission config |
| `routes/tenant.php` | All tenant business API routes |
| `routes/Webhook.php` | External webhook handlers |
| `lib/api/client.ts` | Frontend API client |
| `lib/api/config.ts` | Frontend endpoint definitions |
| `lib/auth/auth-store.ts` | Frontend Zustand auth store |
| `lib/echo.ts` | Frontend WebSocket config |
