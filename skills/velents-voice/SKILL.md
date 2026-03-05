---
name: velents-voice
description: Voice pipeline architecture for VelentsAI — Call model, state machine, 3 pipelines (normal/fast/flash), S2S extraction, batch campaigns
user-invocable: false
---

# VelentsAI Voice Pipeline Architecture

## Call Model (`app/Calls/Models/Call.php`)

```php
/**
 * @property int $id
 * @property int $channel_id
 * @property Agent $Agent
 * @property Channel $Channel
 * @property string $phone
 * @property \App\Calls\Enums\Calls\Status $status
 * @property \App\Core\Casts\webhook $webhook
 * @property \Illuminate\Support\Collection $extraction
 * @property \Illuminate\Support\Collection $values
 * @property \Illuminate\Support\Collection $result
 * @property AgentBatch $Batch
 */
class Call extends \App\Core\Models\Model {

    protected static function booted() : void {
        // When status changes to a connected status AND call has a batch,
        // dispatch batch statistics recalculation
        static::updated(function(Call $Call) {
            if ($Call->wasChanged('status')
                && in_array($Call->status->value, Status::connected())
                && $Call->Batch) {
                \App\Agents\Jobs\Batch\statistics::dispatch($Call->Batch->public_id);
            }
        });
    }

    public $fillable = [
        'created_by_id', 'channel_id', 'agent_batch_id', 'agent_id',
        'phone', 'status', 'webhook', 'extraction', 'values', 'result',
        'name', 'duration', 'expires_in', 'is_all_data_extracted', 'is_archived',
    ];

    public $casts = [
        'values'      => 'collection',
        'extraction'  => 'collection',
        'result'      => 'collection',
        'status'      => Status::class,
        'webhook'     => typeWebhook::class,
        'is_archived' => 'boolean',
    ];

    // Relations
    public function Agent() : BelongsTo;    // -> Agent::class
    public function Channel() : BelongsTo;  // -> Channel::class
    public function Batch() : BelongsTo;    // -> AgentBatch::class (agent_batch_id)
    public function Logs() : HasMany;       // -> CallLog::class (ordered by created_at desc)
    public function Analysis() : MorphOne;  // -> ConversationAnalysis::class (analyzable)

    // Determines phone region for queue routing
    public function phone_type() : string {
        return match(true) {
            Str::startsWith($this->phone, ['+966', '+971', '+965', '+974', '+973', '+968']) => 'saudi',
            default => 'all',
        };
    }

    public function scopeexpires( Builder $Builder ) {
        return $Builder->where('status', Status::onQueue->value)
            ->whereNotNull('expires_in')
            ->where('expires_in', '>=', now()->toDateTimeString());
    }
}
```

---

## Call Status State Machine

```php
enum Status : string {
    // Terminal states (connected/completed)
    case ended     = 'ended';
    case done      = 'done';
    case OnProcess = 'OnProcess';

    // Active/in-progress states
    case active    = 'active';
    case pushed    = 'pushed';
    case onQueue   = 'onQueue';
    case OnCall    = 'OnCall';
    case dialing   = 'dialing';
    case released  = 'released';
    case no_response = 'no_response';

    // Error states
    case Busy_Here = 'Busy Here';
    case Invalid_Number = 'Invalid_Number';
    case Temporarily_Unavailable = 'Temporarily Unavailable';
    case pulled    = 'pulled';
    case draft     = 'draft';
    case canceld   = 'canceld';
    case unactive  = 'unactive';
    case failed    = 'failed';
    case expired   = 'expired';
    case retry     = 'retry';

    // Status groups for filtering/logic
    public static function connected() : array {
        return ['ended', 'done', 'OnProcess'];
    }
    public static function no_answer() : array {
        return ['no_response', 'active', 'pushed', 'released', 'dialing', 'onQueue', 'OnCall'];
    }
    public static function busy() : array {
        return ['Busy Here'];
    }
    public static function failed() : array {
        return ['Temporarily Unavailable', 'pulled', 'draft', 'canceld', 'unactive', 'failed', 'expired', 'retry'];
    }

    // Classify any status into a group
    public static function guess( string $status ) : string {
        return match(true) {
            in_array($status, Status::connected()) => 'connected',
            in_array($status, Status::no_answer()) => 'no_answer',
            in_array($status, Status::busy())      => 'busy',
            in_array($status, Status::failed())    => 'failed',
            default => 'no_answer',
        };
    }

    // Human-readable labels
    public function label() : StatusLabel {
        return match($this) {
            static::draft, static::onQueue                    => StatusLabel::Queued,
            static::active, static::pushed, static::pulled, static::OnCall => StatusLabel::Connected,
            static::released, static::dialing, static::retry  => StatusLabel::Dialing,
            static::no_response, static::unactive             => StatusLabel::NoAnswer,
            static::Busy_Here                                 => StatusLabel::LineBusy,
            static::Temporarily_Unavailable                   => StatusLabel::TemporarilyUnavailable,
            static::Invalid_Number                            => StatusLabel::InvalidNumber,
            static::failed, static::expired, static::canceld  => StatusLabel::Cancelled,
            static::OnProcess                                 => StatusLabel::ProcessingResults,
            static::ended, static::done                       => StatusLabel::Completed,
        };
    }
}
```

---

## Webhook Events

```php
enum WebhookCall : string {
    case Queue                   = 'Queue';
    case pull                    = 'pull';
    case push                    = 'push';
    case start                   = 'start';
    case done                    = 'done';
    case ended                   = 'ended';
    case expired                 = 'expired';
    case Invalid_Number          = 'Invalid_Number';
    case no_response             = 'no_response';
    case busy_here               = 'busy_here';
    case temporarily_unavailable = 'temporarily_unavailable';
    case ended_immediately       = 'ended_immediately';
    case failed                  = 'failed';
    case dialing                 = 'pre_dialing';
    case released                = 'released';
    case OnProcess               = 'OnProcess';
}
```

---

## Three Voice Pipelines

### Pipeline Selection (`app/Calls/Jobs/Init.php`)

```php
class Init extends \App\Core\Support\Job {
    public $queue = 'batch';

    public function handle( CallGateway $CallGateway, Elevenlabs $Elevenlabs, LivekitDispatcher $Livekit ) : void {
        $Call = $this->data_public_id($this->call_public_id);
        if (!$Call instanceof Call) return;

        if (!empty((int)Str::replaceMatches('/[^0-9]++/', '', $Call->phone))) {
            switch(true) {
                // Tenant-specific override: stsBahrain uses LiveKit directly
                case in_array(tenant()->id, ['stsBahrain']):
                    $Livekit->call($Call);
                    break;

                // Flash pipeline: ElevenLabs Conversational AI
                case $Call->Agent->pipeline == Pipeline::flash:
                    $Elevenlabs->call($Call);
                    break;

                // Default (normal/fast): CallGateway -> LiveKit Dispatcher
                default:
                    $CallGateway->call($Call);
                    break;
            }
        } else {
            // Invalid phone number
            $Call->update(['status' => Status::Invalid_Number]);
            SendResult::on(WebhookCall::Invalid_Number, $Call);
        }
    }
}
```

### Pipeline 1: Normal (CallGateway -> LiveKit Dispatcher)
1. `CallGateway::call()` builds payload with `fast: false`
2. Dispatches through CallGateway queue system with `request()`
3. LiveKit Dispatcher receives the call request
4. VoiceAgent handles knowledge base for RAG
5. Call result comes back via signed webhook

### Pipeline 2: Fast (CallGateway -> LiveKit Dispatcher with fast flag)
1. Same as normal but `CallGateway::makeCall()` sets `'fast' => true`
2. `$Call->Agent->pipeline === Pipeline::fast` triggers fast processing

### Pipeline 3: Flash (ElevenLabs Conversational AI)
1. `Elevenlabs::SyncAgent()` creates/updates agent on ElevenLabs platform
2. `Elevenlabs::call()` initiates outbound SIP trunk call
3. Call routed through CallGateway queue for rate limiting
4. ElevenLabs handles full conversation (TTS + ASR + LLM)
5. Knowledge base synced to ElevenLabs via `Elevenlabs::knowledge()`

---

## CallGateway Queue System

The CallGateway acts as a rate-limited queue proxy:

```php
// Headers control queue behavior:
'x-queue-id'           => 'agent_hub_calls_' . $Call->phone_type(),  // saudi or all
'x-identifier'         => tenant()->id,
'x-queue-availability' => $Call->phone_type() === 'all' ? 10 : 30,  // concurrent slots
'x-webhook'            => signed_webhook_url,

// Body wraps the actual request:
{
    "method": "post",
    "url": "https://livekit-dispatcher.velents.ai/call",
    "header": { ... },
    "body": { /* makeCall payload */ }
}
```

Queue operations:
- `Release($Call)` - frees a queue slot after call completes
- `pull($Call)` - removes call from queue
- `push($Call)` - puts call back into queue

---

## Call Payload (Normal/Fast)

```php
public function makeCall( Call $Call ) {
    return [
        'values'              => $Call->values + ['id' => public_id, 'gender' => voice.gender],
        'corporate'           => tenant()->id,
        'agent_id'            => $Call->Agent->public_id,
        'accent'              => $Call->Agent->voice->accent->value,     // egyptian/saudi/etc
        'phone_number'        => $Call->phone,
        'internal_webhook'    => signed_url('CallGatewayRelease'),
        'webhook'             => signed_url('LivekitDispatcherCall'),
        'allow_interruptions' => $Call->Agent->voice->allow_interruptions,
        'fast'                => $Call->Agent->pipeline === Pipeline::fast,
        'language'            => $Call->Agent->voice->locale->value,     // ar/en
        'first_message'       => $Call->Agent->text->message(),
        'tools'               => $Call->Agent->usedTools(...),
        'extraction_schema'   => $Call->Agent->extractionSchema,
        'prompt'              => $Call->Agent->prompt,
    ];
}
```

---

## Post-Call Processing: S2S Extraction

After a call ends (for normal/fast pipelines), `S2SPostProcessing::extractData()` is called:

```php
$this->post('extract-data', [
    'phone_number'       => $Call->phone,
    'call_status'        => data_get($Call->result, 'call_status', 'ACHIEVED'),
    'job'                => 'general',
    'chat_history'       => data_get($Call->result, 'chat_history', []),
    'webhook'            => signed_url('S2S.extractData.call'),
    'call_recording_url' => data_get($Call->result, 'recording'),
    'values'             => $Call->values->toArray(),
    'language'           => $Call->Agent->voice->locale ?? 'ar',
    'extraction_schema'  => ['fields' => $Call->Agent->Schema->map(fn($s) => [
        'name' => $s->name, 'required' => $s->required,
        'type' => $s->type, 'description' => $s->note,
    ])->toArray()],
]);
```

The result comes back via webhook and populates `Call->extraction`.

---

## CallLog Model

```php
class CallLog extends \App\Core\Models\Model {
    public $primaryKey   = 'created_at';
    public $incrementing = false;

    public $fillable = ['status', 'created_by_id', 'payload'];
    public $casts    = ['payload' => 'collection'];
}
```

Logs record every state transition and API response for debugging.

---

## Calls Controller

```php
class Calls extends Controller {
    public function store( Create $Request ) : Resource {
        // Uses Agent::Dispatch() which routes to Calls repo based on channel_type
        return Resource::make($this->Services->Dispatch(
            new AgentChannelAuth($Request),
            $Request->validated('phone'),
            $Request->values(),
        ));
    }

    public function pull( Call $Call ) : Resource;     // Pull from queue (status must be pushed/onQueue)
    public function push( Call $Call ) : Resource;     // Push back (status must be pulled)
    public function Result( Call $Call );               // Get call result with extraction
    public function Logs( Call $Call );                 // Get call log history
    public function archive( Archive $Request );       // Bulk archive calls
    public function unarchive( Unarchive $Request );   // Bulk unarchive calls
}
```

---

## Batch Campaigns

Batch campaigns dispatch multiple calls via `AgentBatch`:
- Created under `app/Agents/Models/AgentBatch.php`
- Each call in a batch has `agent_batch_id` linking to the batch
- When calls complete (connected status), batch statistics are recalculated via `Batch\statistics` job
- Batch status enum: `app/Agents/Enums/batch/status.php`

---

## Phone Type Routing

Phone numbers are classified for queue assignment:
- **Saudi**: prefixed with +966, +971, +965, +974, +973, +968
- **All**: everything else (Egypt, international)

This affects:
- Queue ID: `agent_hub_calls_saudi` vs `agent_hub_calls_all`
- Queue availability: 30 slots (saudi) vs 10 slots (all)
- ElevenLabs phone number selection: `agent_saudi_number_ids` vs `agent_egypt_number_ids`

---

## Directory Structure

```
app/Calls/
  Casts/Call/typeWebhook.php       -- Webhook URL cast
  Commands/CallsExpiresIn.php      -- Expire stale queued calls
  Controllers/Calls.php            -- REST API
  Enums/Calls/Status.php           -- Full state machine
  Enums/Calls/StatusDescription.php
  Enums/Calls/StatusLabel.php      -- Human-readable labels
  Enums/Calls/WebhookCall.php      -- Outbound webhook events
  Jobs/Init.php                    -- Pipeline router job
  Models/Call.php, CallLog.php
  Repositories/Calls.php           -- Query builder, createCall, addLog, pull/push
  Requests/Calls/Archive.php, Create.php, Unarchive.php
  Resources/Calls/Collection.php, Log.php, LogPayload.php, Resource.php, Result.php
```
