---
name: velents-agent-model
description: VelentsAI Agent data model — Agent model, enums (status/type/pipeline/channel/tool), casts, relations, schemas, tools, channels, lifecycle
user-invocable: false
---

# VelentsAI Agent Data Model

## Agent Model (`app/Agents/Models/Agent.php`)

The Agent model is the central entity. It extends `\App\Core\Models\Auth` and uses `\Laravel\Sanctum\HasApiTokens`.

```php
/**
 * @property int $id
 * @property string $name
 * @property string $prompt
 * @property \App\Agents\Enums\Agent\status $status
 * @property \App\Agents\Enums\Agent\Pipeline $pipeline
 * @property \App\Agents\Enums\Agent\type $type
 * @property \App\Agents\Casts\Agent\Text $text
 * @property \App\Agents\Casts\Agent\voice $voice
 * @property \Illuminate\Support\Collection $values
 * @property \Illuminate\Support\Collection $payload
 * @property \App\Agents\Models\AgentElevenlab $Elevenlab
 * @property \Illuminate\Database\Eloquent\Collection<knowledgeBase[]> $knowledgeBase
 * @property \Illuminate\Database\Eloquent\Collection<AgentSchemas[]> $Schema
 * @property Channel[] $Channels
 * @property string $public_id
 */
class Agent extends \App\Core\Models\Auth {

    use \Laravel\Sanctum\HasApiTokens;

    public $fillable = [
        'created_by_id', 'name', 'prompt', 'status', 'type',
        'text', 'voice', 'payload', 'pipeline', 'webhook',
    ];

    public $casts = [
        'status'   => \App\Agents\Enums\Agent\status::class,
        'type'     => \App\Agents\Enums\Agent\type::class,
        'pipeline' => \App\Agents\Enums\Agent\Pipeline::class,
        'payload'  => 'collection',
        'webhook'  => \App\Agents\Enums\Agent\typeWebhook::class,
        'voice'    => \App\Agents\Casts\Agent\typeVoice::class,
        'text'     => \App\Agents\Casts\Agent\typeText::class,
    ];
}
```

### Key Relations

```php
// Has many
public function knowledgeBase() : HasMany;   // -> knowledgeBase::class
public function Schema() : HasMany;          // -> AgentSchemas::class
public function Conversations() : HasMany;   // -> Conversation::class
public function Calls() : HasMany;           // -> Call::class
public function Batchs() : HasMany;          // -> AgentBatch::class
public function Analyses() : HasMany;        // -> ConversationAnalysis::class

// Has one
public function Elevenlab() : HasOne;        // -> AgentElevenlab::class
public function ListExtractionData() : HasOne; // -> AgentExtractionData::class
public function CalendarData() : HasOne;     // -> AgentCalendarData::class

// Belongs to
public function Staff() : BelongsTo;         // -> Staff::class (created_by_id)

// Many to many (with pivot)
public function Channels() : BelongsToMany;  // -> Channel via agent_channels (pivot: is_active, payload)
public function Tools() : BelongsToMany;     // -> Tool via agent_tools (pivot: is_active, payload)
```

### Computed Attributes

```php
// public_id - encrypted/hashed ID accessor
public function publicId() : Attribute {
    return Attribute::get(fn() => $this->public_id('id'));
}

// values - extracts dynamic variable names from prompt + first_message using {variable} pattern
public function values() : Attribute {
    return Attribute::get(fn() => Str::matchAll('/{(\w*)}/', $this->prompt . $this->text->message())
        ->partition(fn($key) => is_numeric($key))
        ->pipe(fn($static) => $static->last()->merge($static->first()->keys()->map(fn($item) => (string)$item)))
        ->unique()->values()
    );
}

// extractionSchema - builds schema payload from Schema relation
public function extractionSchema() : Attribute {
    return Attribute::get(fn() => ['fields' => $this->Schema->map(fn($Schema) => [
        'name'        => $Schema->name,
        'type'        => $Schema->type->value,
        'required'    => $Schema->is_required,
        'description' => $Schema->note,
    ] + match($Schema->type) {
        data_type::list   => ['options' => $Schema->options->toArray()],
        data_type::array  => ['item'    => $Schema->options->toArray()],
        data_type::object => ['fields'  => $Schema->options->toArray()],
        default => [],
    })->toArray()]);
}

// usedTools - delegates to Agent service
public function usedTools( ExtraDataTool $extra ) : array|\stdClass {
    return \App\Agents\Services\Agent::instance()->usedTools($this, $extra);
}
```

---

## Agent Enums

### `App\Agents\Enums\Agent\status`
```php
enum status : string {
    case active   = 'active';
    case unactive = 'unactive';
    case draft    = 'draft';
    case archived = 'archived';
}
```

### `App\Agents\Enums\Agent\type`
```php
enum type : string {
    case text  = 'text';
    case voice = 'voice';
}
```

### `App\Agents\Enums\Agent\Pipeline`
```php
enum Pipeline : string {
    case normal = 'normal';   // LiveKit Dispatcher + VoiceAgent
    case fast   = 'fast';     // CallGateway + LiveKit with fast flag
    case flash  = 'flash';    // ElevenLabs Conversational AI
}
```

### `App\Agents\Enums\Agent\Tool_type`
```php
enum Tool_type : string {
    case calendar          = 'calendar';
    case payment           = 'payment';
    case human_in_the_loop = 'human_in_the_loop';
    case knowledge_base    = 'knowledge_base';
    case memory            = 'memory';
}
```

### `App\Agents\Enums\Agent\channel_type`
```php
enum channel_type : string {
    case webChat      = 'web chat';
    case WhatsApp     = 'WhatsApp';
    case Voice        = 'Voice';
    case Telegram     = 'Telegram';
    case Messenger    = 'Messenger';
    case Instagram    = 'Instagram';
    case Slack        = 'Slack';
    case GenesysCloud = 'Genesys Cloud';

    // Conversation channels: webChat, WhatsApp, GenesysCloud, null
    public static function ConversationsChannels() : array;
    // Voice channels: Voice only
    public static function VoiceChannels() : array;
}
```

### `App\Agents\Enums\Agent\data_type` (for extraction schemas)
```php
enum data_type : string {
    case string  = 'string';
    case number  = 'number';
    case enum    = 'enum';
    case boolean = 'boolean';
    case list    = 'list';
    case array   = 'array';
    case object  = 'object';
}
```

---

## Voice Enums (`App\Agents\Enums\voice\*`)

```php
enum accent : string {
    case egyptian = 'egyptian';
    case saudi    = 'saudi';
    case kuwaiti  = 'kuwaiti';
    case american = 'american';
}

enum gender : string {
    case male   = 'male';
    case female = 'female';
}

enum tone : string {
    case friendly     = 'friendly';
    case professional = 'professional';
    case casual       = 'casual';
}

enum speed : string {
    case slow   = 'slow';
    case fast   = 'fast';
    case normal = 'normal';
}
```

---

## Custom Casts

### `App\Agents\Casts\Agent\voice` (typeVoice cast)

JSON column cast to a value object with typed properties.

```php
class voice implements Arrayable, JsonSerializable {
    public speed  $speed;
    public tone   $tone;
    public accent $accent;
    public gender $gender;
    public bool   $allow_interruptions;
    public locale $locale;          // derived: american -> en, others -> ar
    public languages $flash_language; // ElevenLabs language for flash pipeline

    public function __construct( array $voice ) {
        $this->speed               = speed::tryby($voice['speed']) ?? speed::normal;
        $this->tone                = tone::tryby($voice['tone']) ?? tone::friendly;
        $this->accent              = accent::tryby($voice['accent']) ?? accent::egyptian;
        $this->gender              = gender::tryby($voice['gender'] ?? 'male') ?? gender::male;
        $this->flash_language      = languages::tryby($voice['flash_language'] ?? 'ArabicSaudi') ?? languages::ArabicSaudi;
        $this->allow_interruptions = $voice['allow_interruptions'] ?? false;
        $this->locale = match($this->accent) {
            accent::american => locale::en,
            default          => locale::ar,
        };
    }
}
```

### `App\Agents\Casts\Agent\Text` (typeText cast)

```php
class Text implements Arrayable, JsonSerializable {
    public  bool   $can_start;
    public ?string $first_message;
    public ?string $whatsapp_message;

    // Returns whatsapp_message if set, otherwise first_message
    public function message() : string {
        return ($this->whatsapp_message ?? $this->first_message) . '';
    }
}
```

### `App\Agents\Casts\Agent\ExtraDataTool`

```php
class ExtraDataTool implements Arrayable, JsonSerializable {
    public function __construct( public string $phone = '' ) {
        if (empty($this->phone)) $this->phone = Str::random();
    }
}
```

---

## Agent Service (`app/Agents/Services/Agent.php`)

Handles tool resolution and dispatch logic.

```php
class Agent {
    use SelfMaker, logger;

    public function __construct(
        public Conversations $Conversations,
        public Calls $Calls
    ) {}

    // Resolves active tools for an agent into the payload format expected by TextAgent/VoiceAgent
    public function usedTools( Model $Agent, ExtraDataTool $extra ) : array|\stdClass {
        $Tools = $Agent->Tools->where('pivot.is_active', true);
        $Tools = !$Tools->isEmpty() ? $Tools->map(fn($tool) => match($tool->name) {
            Tool_type::calendar->value          => $this->usedToolsCalendar($Agent, $tool, $extra),
            Tool_type::memory->value            => $this->usedMemory($Agent, $tool, $extra),
            Tool_type::payment->value           => $this->baseTole($Agent, $tool, $extra),
            Tool_type::human_in_the_loop->value => $this->baseTole($Agent, $tool, $extra),
            Tool_type::knowledge_base->value    => $this->baseTole($Agent, $tool, $extra),
            default => []
        })->collapseWithKeys()->filter() : collect();
        return $Tools->isEmpty() ? new \stdClass() : $Tools->toArray();
    }

    // Calendar tool includes provider data
    public function usedToolsCalendar( Model $Agent ) : array {
        return ['calendar' => ['providers' => /* AgentProvider MLResource collection */]];
    }

    // Memory tool includes customer_id = agent_public_id + phone
    public function usedMemory( Model $Agent, Tool $tool, ExtraDataTool $extra ) : array {
        return ['session_history' => ['customer_id' => $Agent->public_id . '_' . $extra->phone]];
    }

    // Dispatch: routes to Conversations or Calls based on channel type
    public function Dispatch( AgentChannelAuth $AgentChannelAuth, ?string $phone, array $values = [] )
        : Conversation|Call|null
    {
        return match(true) {
            in_array($AgentChannelAuth->channel?->name, channel_type::ConversationsChannels())
                => $this->Conversations->createByAgent(...),
            in_array($AgentChannelAuth->channel?->name, channel_type::VoiceChannels())
                => $this->Calls->createCall(...),
            default => null,
        };
    }
}
```

---

## Related Models

### `AgentSchemas` - Extraction schema field definitions
- Fields: name, type (data_type enum), is_required, note, options (collection)

### `AgentElevenlab` - ElevenLabs agent link
- Fields: agent_id, elevenlab_agent_id

### `AgentChannel` (pivot) - Agent-Channel link
- Pivot fields: is_active, payload (collection with WhatsApp/Genesys config)

### `AgentTool` (pivot) - Agent-Tool link
- Pivot fields: is_active, payload

### `AgentBatch` - Batch campaign configuration
- Used for bulk voice call campaigns

### `knowledgeBase` - Knowledge base documents
- Fields: name, aws_path, elevenlabs_kb_id, agent_id
- Synced to ElevenLabs (flash) or VoiceAgent (normal/fast)

### `AgentExtractionData` - Aggregated extraction results
### `AgentCalendarData` - Calendar integration config
### `AgentCalendarSlot` - Available calendar slots
### `AgentProvider` - External provider configurations (for calendar tool)
### `GenesysChannelConfig` - Genesys Cloud channel configuration
- Fields: api_base_url, login_url, oauth_client_id, oauth_client_secret_enc, integration_id, access_token_enc, token_expires_at

---

## Webhook Cast Pattern

```php
// typeWebhook extends \App\Core\Casts\typeWebhook
// Casts the webhook column to a webhook value object
class typeWebhook extends \App\Core\Casts\typeWebhook {
    public function get( Model $model, string $key, mixed $value, array $attributes ) : webhook {
        return new \App\Core\Casts\webhook( $value );
    }
}
```

---

## Key Patterns

1. **public_id**: All models use encrypted/hashed public IDs via `$this->public_id('id')`. Lookup uses `$this->data_public_id($public_id)`.
2. **Enum pattern**: All enums use `\App\Core\Enums\Enum` trait which adds `tryby()` method for flexible deserialization.
3. **Cast pattern**: JSON columns use paired classes: `typeX` (CastsAttributes implementation) + `X` (value object).
4. **Service pattern**: Services use `SelfMaker` trait providing `::instance()` singleton resolution via Laravel container.
5. **Multi-tenancy**: The app uses `tenant()` and `tenancy()->initialize()` for tenant context.
