---
name: velents-analytics
description: Quality analytics for VelentsAI — GeminiJudge evaluation, 6 quality metrics, ConversationAnalysis model, scoring, dashboards
user-invocable: false
---

# VelentsAI Analytics & Quality System

## Architecture Overview

The analytics system evaluates every completed call and conversation using Gemini as an AI judge. It produces 6 quality metrics per interaction, stores them in the `ConversationAnalysis` model (polymorphic to both Call and Conversation), and exposes dashboard APIs for overview, trends, issues, and individual analysis review.

```
Call/Conversation completes
    |
    v
AnalyzeCall / AnalyzeConversation job dispatched
    |
    v
GeminiJudge::evaluate() -- sends prompt + chat history to Gemini
    |
    v
ConversationAnalysis record created/updated with 6 scores + details
    |
    v
Dashboard APIs aggregate and serve the data
```

---

## ConversationAnalysis Model (`app/Analytics/Models/ConversationAnalysis.php`)

```php
class ConversationAnalysis extends \App\Core\Models\Model {

    protected $table = 'conversation_analyses';

    public $fillable = [
        'agent_id',
        'analyzable_type',            // App\Calls\Models\Call or App\Conversations\Models\Conversation
        'analyzable_id',
        'status',                     // pending | completed | failed
        'overall_score',              // 0.0 - 10.0 (weighted average)
        'scope_adherence',            // 0.0 - 10.0
        'extraction_completeness',    // 0.0 - 10.0
        'conversation_quality',       // 0.0 - 10.0
        'customer_satisfaction',      // 0.0 - 10.0
        'resolution_effectiveness',   // 0.0 - 10.0
        'quality_details',            // collection: { issues: [], highlights: [], summary: "" }
        'chat_history',               // collection: full conversation transcript
        'analyzed_at',                // datetime
    ];

    public $casts = [
        'status'                    => AnalysisStatus::class,
        'overall_score'             => 'decimal:1',
        'scope_adherence'           => 'decimal:1',
        'extraction_completeness'   => 'decimal:1',
        'conversation_quality'      => 'decimal:1',
        'customer_satisfaction'     => 'decimal:1',
        'resolution_effectiveness'  => 'decimal:1',
        'quality_details'           => 'collection',
        'chat_history'              => 'collection',
        'analyzed_at'               => 'datetime',
    ];

    // Polymorphic relation to Call or Conversation
    public function analyzable() : MorphTo;
    public function Agent() : BelongsTo;
}
```

---

## AnalysisStatus Enum

```php
enum AnalysisStatus : string {
    case pending   = 'pending';
    case completed = 'completed';
    case failed    = 'failed';
}
```

---

## The 6 Quality Metrics

| Metric | Weight | Description |
|--------|--------|-------------|
| `scope_adherence` | 25% | Did the agent stay within its defined prompt/instructions? |
| `extraction_completeness` | 25% | Were all required schema fields extracted accurately? |
| `conversation_quality` | 17% | Was the conversation fluent, coherent, and professional? |
| `customer_satisfaction` | 17% | Based on customer responses, did they seem satisfied? |
| `resolution_effectiveness` | 16% | Did the conversation achieve its goal/purpose? |
| `overall_score` | weighted avg | 0.25*scope + 0.25*extraction + 0.17*quality + 0.17*satisfaction + 0.16*resolution |

Score tiers:
- **Excellent**: 8.0 - 10.0
- **Good**: 6.0 - 8.0
- **Average**: 4.0 - 6.0
- **Poor**: below 4.0

---

## GeminiJudge Service (`app/Analytics/Services/GeminiJudge.php`)

The AI judge that evaluates conversations using Google Gemini.

```php
class GeminiJudge extends \App\Core\Support\http {

    protected $_timeout = 60;

    public function setbaseUrl( string|null $url = null ) : static {
        $this->baseUrl( $this->config('url', 'https://generativelanguage.googleapis.com') );
        $this->acceptJson();
        $this->asJson();
        return $this;
    }

    public function evaluate(
        Agent $Agent,
        array $chatHistory,
        array $extraction,
        string $interactionType,    // 'voice' or 'text'
        string $conversationStatus
    ) : array {
        $prompt   = $this->buildPrompt($Agent, $chatHistory, $extraction, $interactionType, $conversationStatus);
        $endpoint = Str::swap([':model' => $this->config('model', 'gemini-2.0-flash')], $this->config('generateContent'));
        $endpoint .= '?key=' . $this->config('key');

        $response = $this->post($endpoint, [
            'contents' => [['parts' => [['text' => $prompt]]]],
            'generationConfig' => [
                'responseMimeType' => 'application/json',
                'responseSchema'   => $this->responseSchema(),
                'temperature'      => 0.2,
            ],
        ]);

        return $this->parseResponse($response);
    }
}
```

### Prompt Structure

The prompt sent to Gemini includes:

1. **Agent Configuration**: name, full system prompt, assigned tools (active/inactive), extraction schema fields with types and descriptions
2. **Conversation Data**: interaction type (voice/text), final status
3. **Chat History**: full transcript formatted as `role: content`
4. **Extracted Data**: JSON of extraction results or "No data extracted"
5. **Evaluation Criteria**: detailed rubric for each of the 6 metrics

```
You are a quality analyst for AI agent conversations. Evaluate this {interactionType}
conversation objectively on the following dimensions. Score each dimension from 0.0 to 10.0.

## Agent Configuration
- Agent Name: {name}
- Agent System Prompt: {prompt}
- Assigned Tools: calendar (active), memory (active), ...
- Extraction Schema Fields:
  name (string, required): Customer full name
  phone (string, required): Phone number
  ...

## Conversation Data
- Interaction Type: voice
- Final Status: ended

### Chat History:
assistant: Hello, how can I help you?
user: I need to book an appointment...

### Extracted Data:
{"name": "Ahmed", "phone": "+966..."}

## Evaluation Criteria
1. scope_adherence (0-10): ...
2. extraction_completeness (0-10): ...
3. conversation_quality (0-10): ...
4. customer_satisfaction (0-10): ...
5. resolution_effectiveness (0-10): ...

Calculate overall_score as weighted average: scope(25%) + extraction(25%) + quality(17%) + satisfaction(17%) + resolution(16%).
```

### Response Schema (enforced via Gemini structured output)

```php
protected function responseSchema() : array {
    return [
        'type' => 'object',
        'properties' => [
            'overall_score'             => ['type' => 'number'],
            'scope_adherence'           => ['type' => 'number'],
            'extraction_completeness'   => ['type' => 'number'],
            'conversation_quality'      => ['type' => 'number'],
            'customer_satisfaction'     => ['type' => 'number'],
            'resolution_effectiveness'  => ['type' => 'number'],
            'issues'     => ['type' => 'array', 'items' => ['type' => 'string']],
            'highlights' => ['type' => 'array', 'items' => ['type' => 'string']],
            'summary'    => ['type' => 'string'],
        ],
        'required' => [
            'overall_score', 'scope_adherence', 'extraction_completeness',
            'conversation_quality', 'customer_satisfaction', 'resolution_effectiveness',
            'issues', 'highlights', 'summary',
        ],
    ];
}
```

### Response Parsing with Score Clamping

```php
protected function parseResponse( Response $response ) : array {
    $text = data_get($response->json(), 'candidates.0.content.parts.0.text', '');
    $data = json_decode($text, true);

    if (json_last_error() !== JSON_ERROR_NONE || !is_array($data)) {
        return [/* zero scores + 'Failed to parse Gemini response' issue */];
    }

    $clamp = fn($v) => max(0, min(10, round((float)($v ?? 0), 1)));

    // Recalculate overall_score server-side with correct weights
    $overallScore = round(
        $scopeAdherence * 0.25 + $extractionCompleteness * 0.25 +
        $conversationQuality * 0.17 + $customerSatisfaction * 0.17 +
        $resolutionEffectiveness * 0.16
    , 1);

    return [
        'overall_score'             => $clamp($overallScore),
        'scope_adherence'           => $scopeAdherence,
        'extraction_completeness'   => $extractionCompleteness,
        'conversation_quality'      => $conversationQuality,
        'customer_satisfaction'     => $customerSatisfaction,
        'resolution_effectiveness'  => $resolutionEffectiveness,
        'issues'     => $data['issues'] ?? [],
        'highlights' => $data['highlights'] ?? [],
        'summary'    => $data['summary'] ?? '',
    ];
}
```

---

## Analysis Jobs

### AnalyzeCall (`app/Analytics/Jobs/AnalyzeCall.php`)

```php
class AnalyzeCall extends \App\Core\Support\Job implements ShouldBeUniqueUntilProcessing {
    public $queue = 'default';

    public function __construct( public string $public_id ) {}
    public function uniqueId() : string { return 'analyze_call_' . $this->public_id; }

    public function handle() : void {
        $Call = $this->data_public_id($this->public_id);
        $Agent = $Call->Agent;
        $Agent->load(['Tools', 'Schema']);

        // Create or update analysis record
        $analysis = ConversationAnalysis::updateOrCreate(
            ['analyzable_type' => Call::class, 'analyzable_id' => $Call->id],
            ['agent_id' => $Agent->id, 'status' => AnalysisStatus::pending, 'created_at' => $Call->created_at],
        );

        // Get chat history from call result
        $chatHistory = data_get($Call->result, 'chat_history', []);
        if (empty($chatHistory)) { /* mark failed */ return; }

        // Run Gemini evaluation
        $scores = GeminiJudge::instance()->evaluate(
            $Agent, $chatHistory, $Call->extraction?->toArray() ?? [],
            'voice', $Call->status->value,
        );

        // Store results
        $analysis->update([
            'status' => AnalysisStatus::completed,
            'overall_score'             => $scores['overall_score'],
            'scope_adherence'           => $scores['scope_adherence'],
            'extraction_completeness'   => $scores['extraction_completeness'],
            'conversation_quality'      => $scores['conversation_quality'],
            'customer_satisfaction'     => $scores['customer_satisfaction'],
            'resolution_effectiveness'  => $scores['resolution_effectiveness'],
            'quality_details' => collect([
                'issues'     => $scores['issues'],
                'highlights' => $scores['highlights'],
                'summary'    => $scores['summary'],
            ]),
            'chat_history' => $chatHistory,
            'analyzed_at'  => now(),
        ]);
    }
}
```

### AnalyzeConversation (`app/Analytics/Jobs/AnalyzeConversation.php`)

Same pattern as AnalyzeCall but:
1. Tries to get chat history from `TextAgent::instance()->history($Conversation)`
2. Falls back to conversation logs: `$Conversation->Logs->where('status', 'active')->map(...)`
3. Uses `'text'` as interactionType

---

## Analytics Controller (`app/Analytics/Controllers/Analytics.php`)

```php
class Analytics extends Controller {

    // Authorization: Owner or view_all_analytics or view_own_analytics (own agents only)
    private function authorizeAnalytics( Agent $Agent ) : void;

    // GET /agents/{agent}/analytics/overview
    public function overview( Agent $Agent, DashboardRequest $Request );

    // GET /agents/{agent}/analytics/trends
    public function trends( Agent $Agent, TrendsRequest $Request );

    // GET /agents/{agent}/analytics/issues
    public function issues( Agent $Agent, IssuesRequest $Request );

    // GET /agents/{agent}/analytics/conversations
    public function conversations( Agent $Agent, ConversationsListRequest $Request );

    // GET /agents/{agent}/analytics/conversations/{analysis}
    public function show( Agent $Agent, int $analysis );

    // GET /agents/{agent}/analytics/backfill/status
    public function backfillStatus( Agent $Agent, DashboardRequest $Request );

    // POST /agents/{agent}/analytics/backfill
    public function backfill( Agent $Agent, BackfillRequest $Request );

    // POST /agents/{agent}/analytics/conversations/{analysis}/reanalyze
    public function reanalyze( Agent $Agent, int $analysis );
}
```

Every endpoint returns bilingual descriptions via `Descriptions::overview()`, etc.

---

## Analytics Repository (`app/Analytics/Repositories/Analytics.php`)

### Overview (2 queries)

Single SQL query with conditional aggregates:
```sql
SELECT
    COUNT(*) as total_count,
    SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END) as analyzed_count,
    AVG(CASE WHEN status = 'completed' THEN overall_score END) as avg_overall_score,
    -- ... all 6 metrics
    SUM(CASE WHEN overall_score >= 8.0 THEN 1 ELSE 0 END) as score_excellent,
    SUM(CASE WHEN overall_score >= 6.0 AND overall_score < 8.0 THEN 1 ELSE 0 END) as score_good,
    SUM(CASE WHEN overall_score >= 4.0 AND overall_score < 6.0 THEN 1 ELSE 0 END) as score_average,
    SUM(CASE WHEN overall_score < 4.0 THEN 1 ELSE 0 END) as score_poor
```

Plus separate queries for total_conversations, total_calls, extraction_rate.
Supports optional comparison period (`compare_from`/`compare_to`) with delta calculation.

### Trends (time-series)

```php
// Granularity: hourly, daily, weekly, monthly
// Uses PostgreSQL date_trunc()
$dateTrunc = "date_trunc('day', created_at)";
// Returns: [{ period, count, avg_overall_score, avg_scope_adherence, ... }]
```

### Top Issues

Three sections in a single response:
1. **low_scoring_dimensions**: Dimensions with lowest average scores, with below-threshold counts
2. **common_issues**: Most frequent issues from quality_details, aggregated by text
3. **low_extraction_fields**: Schema fields with extraction rate below 80%

Supports severity filter: `critical` (< 4.0), `warning` (4-6), `info` (6-8).

### Conversations List

Paginated list with extensive filtering:
- Date range, type (call/conversation), channel_id, agent_batch_id
- Score range filters: min_overall_score, max_overall_score, etc.
- Phone number filter (subquery across calls and conversations tables)
- Keyword search in chat_history (ILIKE)
- Custom sorting on any score column

### Backfill

Counts unanalyzed completed calls/conversations and dispatches analysis jobs:
```php
// Up to 500 per request
$Call->whereDoesntHave('Analysis')->limit($limit)->each(function($Call) {
    dispatch(new AnalyzeCall($Call->public_id));
});
```

---

## Descriptions (`app/Analytics/Enums/Descriptions.php`)

Bilingual (English/Arabic) descriptions for every field in every API response.
Used by the frontend to display tooltips and help text.

```php
class Descriptions {
    private static function t( string $en, string $ar ) : array {
        return ['en' => $en, 'ar' => $ar];
    }

    public static function overview() : array {
        return [
            '_section'              => self::t('A high-level snapshot...', '...'),
            'total_conversations'   => self::t('Total number of text-based...', '...'),
            'avg_overall_score'     => self::t('The average quality score...', '...'),
            'score_distribution'    => self::t('Breakdown by quality tier...', '...'),
            // ...
        ];
    }

    public static function trends() : array;
    public static function issues() : array;
    public static function conversations() : array;
    public static function detail() : array;
    public static function backfill() : array;
}
```

---

## Global Filter Pattern

All repository methods use `applyGlobalFilters()`:

```php
protected function applyGlobalFilters( Builder $query, Agent $Agent, array $filters ) : Builder {
    $query->where('agent_id', $Agent->id);

    // Date range
    if (!empty($filters['created_at_from'])) $query->where('created_at', '>=', $filters['created_at_from']);
    if (!empty($filters['created_at_to']))   $query->where('created_at', '<=', $filters['created_at_to']);

    // Type filter (call vs conversation)
    if (!empty($filters['type'])) {
        $query->where('analyzable_type', match($filters['type']) {
            'call'         => Call::class,
            'conversation' => Conversation::class,
        });
    }

    // Channel and batch filters use subqueries for efficiency
    if (!empty($filters['channel_id'])) { /* subquery on calls + conversations */ }
    if (!empty($filters['agent_batch_id'])) { /* subquery on calls + conversations */ }

    return $query;
}
```

---

## Directory Structure

```
app/Analytics/
  Commands/Backfill.php                -- Artisan command for bulk backfill
  Controllers/Analytics.php            -- REST API
  Enums/AnalysisStatus.php             -- pending | completed | failed
  Enums/Descriptions.php               -- Bilingual field descriptions
  Jobs/AnalyzeCall.php                 -- Async job for voice call analysis
  Jobs/AnalyzeConversation.php         -- Async job for text conversation analysis
  Models/ConversationAnalysis.php      -- Polymorphic analysis model
  Repositories/Analytics.php           -- Query builder with overview/trends/issues/list
  Requests/BackfillRequest.php
  Requests/ConversationsListRequest.php
  Requests/DashboardRequest.php
  Requests/IssuesRequest.php
  Requests/TrendsRequest.php
  Resources/AnalysisCollection.php
  Resources/AnalysisResource.php
  Services/GeminiJudge.php             -- Gemini API client for evaluation
```

---

## Integration Points

- **Call completion** triggers `AnalyzeCall` job (dispatched in call lifecycle)
- **Conversation completion** triggers `AnalyzeConversation` job
- **Agent model** has `Analyses()` relation to ConversationAnalysis
- **Call model** has `Analysis()` morphOne relation
- **Backfill** catches historical data that was not auto-analyzed
- **Reanalyze** allows re-evaluation of a specific interaction
