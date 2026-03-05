---
name: velents-integrations
description: External service integration patterns for VelentsAI — TextAgent, ElevenLabs, LiveKit, CallGateway, Gemini, S2S, VelentsIntegrations HTTP clients
user-invocable: false
---

# VelentsAI Integration Services

All integration service classes live under `app/Integration/Services/` and extend `\App\Core\Support\http`.
Every service implements `setbaseUrl()` which reads its URL from `config/services.php` via `$this->config()`.

## Base HTTP Client Pattern

Every service extends `\App\Core\Support\http` and uses the singleton/instance pattern via `SelfMaker` trait.
Call any service with `ServiceName::instance()`.

```php
class TextAgent extends \App\Core\Support\http {
    public function setbaseUrl( string | null $url = null ) : static {
        $this -> baseUrl( $this -> config( 'url' , 'https://text-agent.velents.ai' ) ) ;
        return $this ;
    }
}
```

Key conventions:
- `$this->config('key')` reads from `config('services.ServiceName.key')`
- `$this->config(__FUNCTION__)` reads the endpoint path matching the current method name
- URL parameter substitution uses `Str::swap([':param' => $value], $url)`
- Signed webhook URLs use `$this->Url_webHook('routeName', params, expiration)`
- Internal network webhooks use `$this->Url_webHookNetwork('routeName')`
- Logging uses `$this->Log()`, `$this->LogResponse()`, `$this->LogError()`

---

## config/services.php Structure

```php
return [
    'VoiceAgent' => [
        'url' => env('VOICE_AGENT_URL', 'https://voice-agent.velents.ai'),
        'knowledge' => 'kb/create_kb',
        'knowledge_delete' => 'kb/delete',
    ],

    'LivekitDispatcher' => [
        'url'  => env('LIVEKIT_DISPATCHER_URL', 'https://livekit-dispatcher.velents.ai'),
        'call' => 'call',
        'registerOutboundTrunk' => 'register-outbound-trunk',
        'registerInboundTrunk'  => 'register-inbound-trunk',
    ],

    'CallGateway' => [
        'url'       => env('CALLGATEWAY_URL', 'https://call-gateway.velents.ai'),
        'x-api-key' => '...',
        'call'      => 'call',
        'Release'   => 'release',
        'request'   => 'request',
    ],

    'TextAgent' => [
        'url'                   => env('TEXT_AGENT_URL', 'https://text-agent.velents.ai'),
        'start_session'         => 'api/v1/sessions/start',
        'send_message'          => 'api/v1/sessions/:session_id/message',
        'insert_messgae'        => 'api/v1/sessions/:session_id/messages/batch',
        'update_status'         => 'api/v1/sessions/:session_id/status',
        'send_audio'            => 'api/v1/voice/process',
        'history'               => 'api/v1/sessions/:session_id/status',
        'end_session'           => 'api/v1/sessions/:session_id/end',
        'tools_generate_upload' => 'api/v1/tools-management/generate/upload',
        'tools_generate_text'   => 'api/v1/tools-management/generate/text',
        'tools_generate_url'    => 'api/v1/tools-management/generate/url',
        'tools_list'            => 'api/v1/tools-management/tools',
        'tools_get'             => 'api/v1/tools-management/tools/:tool_id',
        'tools_delete'          => 'api/v1/tools-management/tools/:tool_id',
        'tools_update_status'   => 'api/v1/tools-management/tools/:tool_id/status',
        'tools_update_auth'     => 'api/v1/tools-management/tools/:tool_id/auth',
        'tools_history'         => 'api/v1/tools-management/history',
    ],

    'VelentsIntegrationsVelentsAi' => [
        'url'                    => env('VELENTSINTEGRATIONSVELENTSAI_URL', 'https://velents-integrations.velents.ai'),
        'Create_Subaccount'      => 'api/v1/whatsapp/onboarding',
        'SendFreeform'           => 'api/v1/whatsapp/send-message',
        'Create_Template'        => 'api/v1/whatsapp/templates',
        'CreateAccounts'         => 'api/v1/velents/accounts',
        'SendFirstMessage'       => 'api/v1/whatsapp/send-first-message',
        'CreatePaymentAccount'   => 'api/v1/payment/account/:refrence_id',
        'GeneratePaymentLink'    => 'api/v1/payment/generate-payment-link/:refrence_id',
        'meta_base_token'        => env('VELENTSINTEGRATIONSVELENTSAI_META_BASE_TOKEN'),
        'Onbourding'             => 'api/Meta/Tenant/Onbourding/Whatsapp',
        'register_phone'         => 'api/Meta/Whatsapp/register_phone_number',
        'get_phones'             => 'api/Meta/Whatsapp/phone_numbers',
        'get_phone'              => 'api/Meta/Whatsapp/phone_numbers/phone_number_id',
        'get_templates'          => 'api/Meta/Whatsapp/Template',
        'get_template'           => 'api/Meta/Whatsapp/Template/template_id',
        'set_webhook'            => 'api/Meta/Whatsapp/set_webhook',
        'mark_as_read'           => 'api/Meta/Whatsapp/mark_as_read',
        'meta_send_freeform'     => 'api/Meta/Whatsapp/send_message',
    ],

    'Elevenlabs' => [
        'url'                    => env('ELEVENLABS_URL', 'https://api.elevenlabs.io'),
        'auth'                   => env('ELEVENLABS_AUTH'),
        'llm_model_id'           => 'glm-45-air-fp8',
        'createAgent'            => 'v1/convai/agents/create',
        'getAgent'               => 'v1/convai/agents/agent_id',
        'UpdateAgent'            => 'v1/convai/agents/agent_id',
        'outboundCall'           => 'v1/convai/sip-trunk/outbound-call',
        'createKnowledgeFromUrl' => 'v1/convai/knowledge-base/url',
        'deleteKnowledge'        => 'v1/convai/knowledge-base/document_id',
        'agent_egypt_number_ids' => ['phnum_...'],
        'agent_saudi_number_ids' => ['phnum_...', 'phnum_...', 'phnum_...'],
    ],

    'S2SPostProcessing' => [
        'url'         => env('S2S_POST_PROCESSING_URL', 'https://s2s-post-processing.velents.ai'),
        'extractData' => 'extract-data',
    ],

    'GeminiJudge' => [
        'url'             => env('GEMINI_URL', 'https://generativelanguage.googleapis.com'),
        'key'             => env('GEMINI_API_KEY'),
        'model'           => env('GEMINI_MODEL', 'gemini-2.0-flash'),
        'generateContent' => 'v1beta/models/:model:generateContent',
    ],

    'GenesysCloud' => [
        'regions' => [
            'us-east' => 'mypurecloud.com', 'eu-ireland' => 'mypurecloud.ie',
            'me-uae' => 'mypurecloud.me', /* ... */
        ],
        'oauth_token'          => 'oauth/token',
        'inbound_message'      => 'api/v2/conversations/messages/:integrationId/inbound/open/message',
        'inbound_event'        => 'api/v2/conversations/messages/:integrationId/inbound/open/event',
        'inbound_receipt'      => 'api/v2/conversations/messages/:integrationId/inbound/open/receipt',
        'get_conversation'     => 'api/v2/conversations/:conversationId',
        'disconnect'           => 'api/v2/conversations/:conversationId/disconnect',
        'transfer_to_acd'      => 'api/v2/conversations/:conversationId/participants/:participantId/replace',
        'get_queue'            => 'api/v2/routing/queues/:queueId',
        'list_queues'          => 'api/v2/routing/queues',
        'create_channel'       => 'api/v2/notifications/channels',
        'channel_subscriptions'=> 'api/v2/notifications/channels/:channelId/subscriptions',
        'token_cache_ttl'      => 3300,
    ],
];
```

---

## Service: TextAgent

**Purpose**: Text-based conversation management with the AI text agent microservice.

```php
class TextAgent extends \App\Core\Support\http {

    public function start_session( Agent $Agent, array $data = [], array $values = [] ) {
        return $this->post( $this->config(__FUNCTION__), $this->start_session_payload($Agent, $data, $values) );
    }

    // Payload includes: agent_id, restricted, persona (tone/context/role/language/greeting_message/values),
    // tools (from $Agent->usedTools()), extraction_schema, llm_config (model/temperature/max_tokens)
    public function start_session_payload( Agent $Agent, array $data = [], array $values = [] ) {
        $usedTools = $Agent->usedTools( new ExtraDataTool( phone: $data['payload']['phone'] ?? '' ) );
        return [
            'agent_id'          => $Agent->public_id,
            'restricted'        => /* human_in_the_loop check */,
            'persona'           => [
                'tone'             => ($Agent->voice->tone ?? tone::friendly)->value,
                'context'          => $Agent->prompt,
                'role'             => 'ai agent',
                'language'         => 'ar',
                'greeting_message' => $Agent->text->message(),
                'values'           => empty($values) ? new \stdClass() : (object)$values,
            ],
            'tools'             => $usedTools,
            'extraction_schema' => $Agent->extractionSchema,
            'llm_config'        => ['model' => 'gpt-5', 'temperature' => 0.2, 'max_tokens' => 2048],
        ];
    }

    public function end_session( Conversation $Conversation ) {
        return $this->post(
            Str::swap([':session_id' => $Conversation->session_id], $this->config(__FUNCTION__)),
            ['reason' => 'customer_ended']
        );
    }

    public function send_message( string $session_id, Message $Message ) {
        // POST api/v1/sessions/:session_id/message
        // Body: { message, metadata: { timestamp, channel, message_type } }
    }

    public function send_audio( string $session_id, Message $Message ) {
        // POST api/v1/voice/process
        // Body: { voice_uri, session_id, metadata }
    }

    public function update_status( Conversation $Conversation, Status $Status ) {
        // Only for: active, payment_pending, human_pending, completed, expired
    }

    // Tools management API:
    public function tools_generate_from_file( UploadedFile $file, array $data = [] );
    public function tools_generate_from_text( array $data = [] );
    public function tools_generate_from_url( array $data = [] );
    public function tools_list( array $filters = [] );
    public function tools_get( string $toolId, ?string $agentId = null );
    public function tools_delete( string $toolId, ?string $agentId = null );
    public function tools_update_status( string $toolId, string $status, ?string $agentId = null );
    public function tools_update_auth( string $toolId, array $data = [] );
    public function tools_generation_history( array $filters = [] );
}
```

---

## Service: Elevenlabs (Flash Pipeline)

**Purpose**: ElevenLabs Conversational AI agent sync, outbound calls via flash pipeline, knowledge base management.

```php
class Elevenlabs extends \App\Core\Support\http {
    protected $_timeout = 50;

    public function setbaseUrl( string|null $url = null ) : static {
        $this->baseUrl( $this->config('url', 'https://call-gateway.velents.ai') );
        $this->acceptJson();
        $this->asJson();
        $this->withHeader('xi-api-key', $this->config('auth'));
        return $this;
    }

    // Syncs agent to ElevenLabs only for Pipeline::flash
    public function SyncAgent( Agent $Agent ) : void {
        if ($Agent->pipeline === Pipeline::flash) {
            if (!$Agent->Elevenlab()->exists())
                $Agent->Elevenlab()->create(['elevenlab_agent_id' => $this->createAgent($Agent)->json('agent_id')]);
            $this->UpdateAgent($Agent);
        }
    }

    // Maps Agent to ElevenLabs conversation_config
    public function dataMapping( Agent $Agent ) {
        return [
            'name' => $Agent->name,
            'tags' => [app()->environment(), $Agent->public_id],
            'conversation_config' => [
                'tts' => [
                    'model_id' => $Agent->voice->flash_language->tts_model_id(),
                    'voice_id' => $this->voice_id($Agent),   // male/female based on gender
                ],
                'asr' => ['provider' => 'scribe_realtime'],
                'turn' => [
                    'silence_end_call_timeout' => 10,
                    'turn_timeout' => 7,
                    'turn_eagerness' => $Agent->voice->allow_interruptions ? 'eager' : 'patient',
                ],
                'agent' => [
                    'first_message' => $this->replaceVars($Agent->text->message()),
                    'language'      => $Agent->voice->flash_language->code(),
                    'prompt'        => [
                        'llm'            => $this->config('llm_model_id'),  // 'glm-45-air-fp8'
                        'prompt'         => $this->replaceVars($Agent->prompt),
                        'knowledge_base' => /* mapped from $Agent->knowledgeBase */,
                        'tools'          => $this->systemTools($Agent),
                    ],
                ],
            ],
        ];
    }

    // ElevenLabs outbound call flow
    public function call( Call $Call ) : void {
        // 1. Pick available phone number ID (saudi vs egypt) by deducting already-tried numbers
        // 2. SyncAgent if Elevenlab record missing
        // 3. Dispatch via CallGateway::instance()->request() with callData
        // 4. On failure -> callfailed() which logs, sets status=failed, sends webhook
    }

    public function callData( Call $Call, string $agent_phone_number_id ) : array {
        return [
            'agent_id'              => $Call->Agent->Elevenlab->elevenlab_agent_id,
            'agent_phone_number_id' => $agent_phone_number_id,
            'to_number'             => $Call->phone,
            'conversation_initiation_client_data' => [
                'user_id'           => $Call->public_id,
                'dynamic_variables' => /* filtered from Call values */,
            ],
        ];
    }

    // System tools for flash pipeline: end_call, voicemail_detection, language_detection
    public function systemTools( Agent $Agent ) : array;

    // Knowledge base CRUD
    public function knowledge( knowledgeBase $kb ) : void;       // POST v1/convai/knowledge-base/url
    public function knowledge_update( knowledgeBase $kb ) : void; // delete + recreate
    public function knowledge_delete( knowledgeBase $kb ) : void;
}
```

---

## Service: CallGateway (Queue-Based Call Routing)

**Purpose**: Proxies outbound calls through a queue system with rate limiting per tenant.

```php
class CallGateway extends \App\Core\Support\http {

    public function setbaseUrl( string|null $url = null ) : static {
        $this->baseUrl( $this->config('url', 'https://call-gateway.velents.ai') );
        $this->withHeader('x-api-key', $this->config('x-api-key'));
        return $this;
    }

    // Direct call (normal/fast pipeline via LiveKit Dispatcher)
    public function call( Call $Call ) : void {
        $makeCall = CallGateway::instance()->request(
            $Call,
            'https://livekit-dispatcher.velents.ai/' . $this->config(__FUNCTION__),
            $this->options['headers'],
            $this->makeCall($Call),
        );
    }

    // Build call payload
    public function makeCall( Call $Call ) {
        return [
            'values'              => $Call->values->put('id', $Call->public_id)->put('gender', ...),
            'corporate'           => tenant()->id,
            'agent_id'            => $Call->Agent->public_id,
            'accent'              => $Call->Agent->voice->accent->value,
            'phone_number'        => $Call->phone,
            'internal_webhook'    => $this->Url_webHook('CallGatewayRelease', ...),
            'webhook'             => $this->Url_webHook('LivekitDispatcherCall', ...),
            'allow_interruptions' => $Call->Agent->voice->allow_interruptions,
            'fast'                => $Call->Agent->pipeline === Pipeline::fast,
            'language'            => $Call->Agent->voice->locale->value,
            'first_message'       => $Call->Agent->text->message(),
            'tools'               => $Call->Agent->usedTools(...),
            'extraction_schema'   => $Call->Agent->extractionSchema,
            'prompt'              => $Call->Agent->prompt,
        ];
    }

    // Queue system: request() wraps any call through the gateway queue
    public function request( Call $Call, string $url, array $header = [], array $body = [] ) {
        // Headers: x-queue-id (agent_hub_calls_{phone_type}), x-identifier (tenant),
        //          x-queue-availability (10 or 30), x-webhook (signed callback)
        // Body: { method: 'post', url, header, body }
    }

    public function Release( Call $Call );  // POST /release (frees queue slot)
    public function pull( Call $Call );     // Pull call from queue
    public function push( Call $Call );     // Push call back to queue
}
```

---

## Service: LivekitDispatcher

**Purpose**: Direct LiveKit-based call dispatch and SIP trunk registration.

```php
class LivekitDispatcher extends \App\Core\Support\http {

    public function call( Call $Call ) : void {
        // Dispatches via CallGateway::instance()->request() with makeCall payload
    }

    public function registerOutboundTrunk( Phone $Phone ) {
        // POST /register-outbound-trunk { label, phone_number, address, ...authentication }
    }

    public function registerInboundTrunk( Phone $Phone ) {
        // POST /register-inbound-trunk { label, phone_number }
    }
}
```

---

## Service: VelentsIntegrationsVelentsAi (WhatsApp + Payment + Meta)

**Purpose**: WhatsApp messaging (Twilio and Meta APIs), payment account/link generation, Meta onboarding.

```php
class VelentsIntegrationsVelentsAi extends \App\Core\Support\http {

    // WhatsApp Account Management
    public function CreateAccounts( Agent $Agent );       // Create Velents account
    public function Create_Subaccount( Agent $Agent, array $array ); // WhatsApp onboarding
    public function Onbourding( Agent $Agent, array $array );        // Meta tenant onboarding

    // Twilio-based messaging
    public function SendFreeform( array $whatsapp, array $array, ?string $message, array $media_url );
    public function SendFirstMessage( Conversation $Conversation, array $whatsapp );

    // Meta-based messaging (Cloud API)
    public function meta_send_freeform( AgentChannel $Channel, Conversation $Conversation, ?string $message, ?string $media_url );
    public function MetaSendFirstMessage( AgentChannel $Channel, Conversation $Conversation );
    public function mark_as_read( AgentChannel $Channel, string $message_id );

    // Meta phone/template management
    public function get_phones( AgentChannel $Channel );
    public function get_phone( AgentChannel $Channel, string $phone_number_id );
    public function register_phone( AgentChannel $Channel, string $phone_number_id );
    public function set_webhook( AgentChannel $Channel, string $public_id, string $phone_number_id );
    public function get_templates( AgentChannel $Channel, array $array = [] );
    public function get_template( AgentChannel $Channel, string $template_id );

    // Payment
    public function CreatePaymentAccount( Agent $Agent, array $data );
    public function GeneratePaymentLink( PaymentItem $Item, Link|Item $data );
}
```

---

## Service: S2SPostProcessing

**Purpose**: Post-call data extraction via speech-to-speech processing service.

```php
class S2SPostProcessing extends \App\Core\Support\http {

    public function extractData( Call $Call ) : Response {
        return $this->post( $this->config('extractData'), [
            'phone_number'       => $Call->phone,
            'call_status'        => data_get($Call->result, 'call_status', 'ACHIEVED'),
            'job'                => 'general',
            'chat_history'       => data_get($Call->result, 'chat_history', []),
            'webhook'            => $this->Url_webHook('S2S.extractData.call', ['Call' => $Call->public_id], ...),
            'call_recording_url' => data_get($Call->result, 'recording'),
            'values'             => $Call->values->toArray(),
            'language'           => $Call->Agent->voice->locale ?? 'ar',
            'extraction_schema'  => ['fields' => $Call->Agent->Schema->map(fn($s) => [
                'name' => $s->name, 'required' => $s->required, 'type' => $s->type, 'description' => $s->note,
            ])->toArray()],
        ]);
    }
}
```

---

## Service: GenesysCloud

**Purpose**: Genesys Cloud CX integration for enterprise contact center channels.

```php
class GenesysCloud extends \App\Core\Support\http {
    protected $_timeout = 30;
    protected ?GenesysChannelConfig $config = null;

    // Must call forConfig() before using any method
    public function forConfig( GenesysChannelConfig $config ) : static {
        $this->config = $config;
        $this->baseUrl($config->api_base_url);
        $this->withToken($this->getAccessToken($config));
        return $this;
    }

    // OAuth2 client_credentials flow with DB-cached token
    public function getAccessToken( GenesysChannelConfig $config ) : string;

    // Open Messaging API
    public function sendMessage( Conversation $Conversation, Message $message ) : Response;
    public function sendEvent( Conversation $Conversation, string $eventType ) : Response;
    public function sendReceipt( Conversation $Conversation, string $messageId, string $receiptType ) : Response;

    // Conversations API
    public function transferToQueue( Conversation $Conversation, string $queueId ) : Response;
    public function disconnectConversation( Conversation $Conversation ) : Response;
    public function getConversation( string $conversationId ) : Response;

    // Routing API
    public function getQueue( string $queueId ) : Response;
    public function listQueues( int $pageSize = 25, int $pageNumber = 1 ) : Response;

    // Webhook signature validation
    public static function validateSignature( string $payload, string $signature, string $secret ) : bool;
}
```

---

## Service: VoiceAgent (Knowledge Base for Normal/Fast Pipeline)

**Purpose**: Knowledge base ingestion for the voice agent microservice (non-ElevenLabs pipelines).

```php
class VoiceAgent extends \App\Core\Support\http {

    public function knowledge( knowledgeBase $Models, array $data = [] ) {
        // POST kb/create_kb { uuid, file_link, agent_id, truncate, replace, request_id, file_type }
    }

    public function knowledge_delete( knowledgeBase $Models, array $data = [] ) {
        // DELETE kb/delete { document_id, agent_id }
    }

    public function update( Collection $data ) : array {
        $data->map(fn($kb) => $this->knowledge($kb, ['replace' => true]));
    }

    public function _delete( Collection $data ) : array {
        $data->map(fn($kb) => $this->knowledge_delete($kb, ['document_id' => true]));
    }
}
```

---

## InBound Webhook Structure

Inbound webhooks are organized under `app/Integration/InBound/{ServiceName}/`:

```
InBound/
  CallGateway/Controllers/CallGateway.php     -- Call status callbacks
  Elevenlabs/Controllers/Elevenlabs.php       -- ElevenLabs conversation events
  Elevenlabs/Enums/languages.php              -- Language enum with TTS/voice IDs
  Elevenlabs/Jobs/Callback.php                -- Async processing of ElevenLabs events
  Ffmpeg/Controllers/Ffmpeg.php               -- Audio processing webhooks
  GenesysCloud/                               -- Genesys inbound message handlers
  LivekitDispatcher/                          -- LiveKit call event handlers
  S2SPostProcessing/                          -- S2S extraction result callbacks
  VelentsIntegrationsVelentsAi/               -- WhatsApp/Meta message webhooks
```

## CallWebHook (Outbound Webhook Dispatch)

```
CallWebHook/
  Call/SendResult.php         -- Sends call result webhooks to tenant
  Conversation/SendResult.php -- Sends conversation result webhooks to tenant
```

Usage: `SendResult::on( WebhookCall::failed, $Call )` dispatches the configured webhook for the call event.

---

## Integration Flow Summary

1. **Text conversations**: Agent dispatched -> TextAgent::start_session() -> messages flow via TextAgent::send_message()
2. **Voice calls (normal/fast)**: Agent dispatched -> CallGateway::call() -> LiveKit Dispatcher -> VoiceAgent handles knowledge
3. **Voice calls (flash)**: Agent dispatched -> Elevenlabs::call() -> ElevenLabs SIP trunk via CallGateway queue
4. **Post-call**: S2SPostProcessing::extractData() -> webhook callback with extraction results
5. **WhatsApp**: VelentsIntegrationsVelentsAi handles both Twilio and Meta Cloud API messaging
6. **GenesysCloud**: OAuth2 token-cached integration for enterprise Open Messaging channels
