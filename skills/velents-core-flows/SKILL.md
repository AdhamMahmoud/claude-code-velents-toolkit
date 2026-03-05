---
name: velents-core-flows
description: "CORE BUSINESS: End-to-end AI agent conversation and call flows — the primary value of VelentsAI. Always loaded."
user-invocable: false
---

# VelentsAI Core Business Flows

> VelentsAI is an AI agent platform. The CORE product is: businesses create AI agents that handle conversations (text) and calls (voice) with their customers. Everything else supports this.

## What VelentsAI Does (Business Context)

1. **Businesses** sign up → get a tenant (subdomain + isolated database)
2. **Business users** build AI agents using a 7-tab builder (Setup, Knowledge, Schema, Tools, Channels, Actions, Learning)
3. **AI agents** handle customer interactions via text (WhatsApp, WebChat, Genesys) or voice (phone calls via LiveKit/ElevenLabs/CallGateway)
4. **Every interaction** is analyzed for quality by GeminiJudge (6 quality scores)
5. **Businesses pay** per interaction using a credit-based billing system

## Flow 1: Text Conversation (Core)

```
Customer sends message (WhatsApp/WebChat/Genesys/API)
    ↓
Channel receives message
    ↓ webhook or direct API
Inbound Handler resolves tenant + agent
    ↓
Payment::Pay($tenant, 'start', 'inbound')  ← deduct credits
    ↓
ConversationRepository::createByAgent($agent, $channel)
    ↓ creates Conversation record (status: active)
TextAgent::start_session(agent_prompt, knowledge, schema, tools)
    ↓ external AI service starts session
Customer ↔ Agent message loop:
    ├── Customer sends → POST /Conversations/{id}/Send
    │   ├── Payment::Pay($tenant, 'message')
    │   ├── TextAgent::send_message(session_id, message)
    │   ├── Save ConversationLog (role: user + assistant)
    │   └── Broadcast ConversationUpdated via Reverb WebSocket
    │
    ├── Agent uses tool → tool execution → response continues
    │
    └── Human escalation → human_in_the_loop tool → status: human_pending
        └── Human agent takes over in dashboard
    ↓
Conversation ends → POST /Conversations/{id}/End
    ↓
TextAgent::end_session(session_id)
    ↓
GeminiJudge::evaluate(chat_history, prompt, schema)
    ↓ dispatches AnalyzeConversation job
ConversationAnalysis created (6 quality scores 0-10)
```

## Flow 2: Voice Call (Core)

```
Trigger: Outbound campaign OR inbound phone call
    ↓
Calls::createCall($agent, $contact, $pipeline)
    ↓ creates Call record (status: onQueue)
Payment::Pay($tenant, 'start', 'outbound')
    ↓
InitCall job dispatched (resolves pipeline)
    ↓
┌─────────────────────────────────────────────────┐
│ Pipeline Resolution:                              │
│  normal → LiveKit room + VoiceAgent SIP           │
│  fast   → CallGateway direct PSTN                 │
│  flash  → ElevenLabs Conversational AI SIP trunk  │
└─────────────────────────────────────────────────┘
    ↓
Call state machine:
  onQueue → calling → ringing → in-progress → post-processing → done
  Error: busy | no-answer | failed | canceled
    ↓
During call: Payment::Pay($tenant, 'call_minute') per minute
    ↓
Call ends → webhook callback from voice service
    ↓
Post-Processing Chain:
  1. S2SPostProcessing::extractData(audio_url)
     → transcript + extracted schema fields
  2. Save transcript as Conversation with ConversationLogs
  3. GeminiJudge::evaluate(transcript, prompt, schema)
     → dispatches AnalyzeCall job
  4. ConversationAnalysis created (6 quality scores)
  5. Webhook to business (if configured): call_data + scores + extracted_data
```

## Flow 3: Agent Builder (How Agents Are Created)

```
Business user → Agent Builder UI (7 tabs)
    ↓
Tab 1 - Setup:
  name, prompt, type (inbound/outbound), pipeline (normal/fast/flash)
  first_message, end_call_message, voice_config (accent, tone, speed, gender)
    ↓
Tab 2 - Knowledge:
  Upload files (PDF/DOCX → S3) | Add text | Add URLs (auto-refresh)
  → Synced to ElevenLabs KB + VoiceAgent vector store
    ↓
Tab 3 - Schema:
  Define extraction fields with types (string/number/enum/boolean/list/array/object)
  Required vs optional, descriptions for AI guidance
    ↓
Tab 4 - Tools:
  5 built-in tools (transfer_call, end_call, human_in_the_loop, send_sms, send_email)
  + Dynamic tools from files/text/URLs (parsed by Gemini)
  Per-tool auth: API key, Bearer, Basic, OAuth2, JWT
    ↓
Tab 5 - Channels:
  Enable: WhatsApp, Genesys Cloud, WebChat, Phone, API
  Each channel has config in agent_channels pivot (payload JSON)
    ↓
Tab 6 - Actions: (future) Post-call automations
Tab 7 - Learning: (future) Self-improvement from conversations
    ↓
Agent status: draft → active (ready to handle interactions)
```

## Flow 4: Batch Campaign (Outbound at Scale)

```
Business creates Campaign:
  agent_id, contact_list (CSV/API), schedule, max_concurrent
    ↓
Payment check: enough credits for ALL contacts? (all-or-nothing)
    ↓
Campaign dispatches calls with rate limiting:
  foreach contacts as $contact:
    dispatch(new InitCall($agent, $contact))->onQueue('calls')
    ↓ RateLimiter controls concurrency
    ↓
Each call follows Flow 2 (Voice Call)
    ↓
Campaign dashboard: real-time progress, per-call status, aggregate analytics
```

## Key Models (Database)

| Model | Purpose | Key Fields |
|-------|---------|------------|
| Agent | AI agent definition | name, prompt, status, type, pipeline, voice_config |
| Conversation | Text chat session | agent_id, channel, status, session_id |
| ConversationLog | Individual messages | conversation_id, role (user/assistant/system), content |
| Call | Voice call session | agent_id, contact_id, pipeline, status, duration |
| KnowledgeBase | Agent knowledge | agent_id, type (file/text/url), content, sync_status |
| Schema | Extraction template | agent_id, fields (JSON), required_fields |
| Tool | Agent capabilities | agent_id, type (builtin/dynamic), auth_config |
| ConversationAnalysis | Quality scores | analyzable_type/id, 6 scores (0-10) |
| Payment | Credit transactions | tenant_id, action, amount, currency |
| Campaign | Batch outbound | agent_id, contacts, schedule, status |

## External Services Map

| Service | Role in Core Flow |
|---------|-------------------|
| **TextAgent** | Powers ALL text conversations (start/send/end session) |
| **ElevenLabs** | Voice synthesis + SIP trunks for flash pipeline |
| **VoiceAgent** | Voice conversation engine for normal pipeline |
| **LiveKit** | WebRTC rooms + SIP dispatch for normal pipeline |
| **CallGateway** | PSTN call initiation for fast pipeline |
| **GeminiJudge** | Quality analysis of ALL conversations/calls (Gemini 2.0 Flash) |
| **S2SPostProcessing** | Audio → transcript + data extraction (post-call) |
| **VelentsIntegrations** | WhatsApp messaging, payment links, onboarding |

## Rules for ALL Development

1. **Every feature connects to these flows** — if you're building something, understand WHERE it fits in the flows above
2. **Payment is checked at every interaction start** — never skip Payment::Pay()
3. **Tenant context is ALWAYS required** — every query, job, and webhook must have tenant context
4. **All conversations/calls get analyzed** — GeminiJudge runs on everything
5. **Real-time updates are expected** — broadcast state changes via Reverb WebSocket
6. **Schema extraction is key** — agents extract structured data from conversations, this is a core value prop
7. **Multi-channel is the norm** — same agent can handle WhatsApp + Phone + WebChat simultaneously
