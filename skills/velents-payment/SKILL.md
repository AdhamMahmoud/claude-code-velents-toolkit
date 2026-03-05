---
name: velents-payment
description: Credit billing system for VelentsAI — PaymentPlanTemplate, PaymentPlan, PaymentPlanLog, Payment::Pay(), credit deduction, payment links
user-invocable: false
---

# VelentsAI Payment & Credit Billing System

## Architecture Overview

The payment system is credit-based. Tenants purchase a `PaymentPlanTemplate`, which creates a `PaymentPlan` with a credit balance. Usage of minutes, messages, and WhatsApp messages deducts credits from the plan. Deductions are logged in `PaymentPlanLog`.

```
PaymentPlanTemplate (catalog)
    |
    v
PaymentPlan (tenant's active plan with credit balance)
    |
    v
PaymentPlanLog (debit/credit entries per usage)
```

---

## Models

### PaymentPlanTemplate (`app/Payment/Models/PaymentPlanTemplate.php`)

The catalog/template from which plans are instantiated. Implements `PaymentItem` contract.

```php
class PaymentPlanTemplate extends \App\Core\Models\Model implements PaymentItem {

    public $fillable = [
        'credit', 'plan_type', 'payload', 'is_active', 'price_list', 'price',
    ];

    public $casts = [
        'plan_type'  => \App\Payment\Enums\PlanType::class,
        'payload'    => 'collection',
        'price_list' => \App\Payment\Casts\typePriceList::class,
        'price'      => \App\Payment\Casts\PaymentPlanTemplate\typePrice::class,
    ];

    public function HavePayment() : bool {
        return !is_null($this->payment->price);
    }

    public function getAgent() : Agent {
        return $this->data_public_id(config('app.agent_payment_id'));
    }

    public function rules() : array {
        return [
            'payload'             => ['required', 'array'],
            'payload.currency'    => ['required', Rule::enum(Currencies::class)],
            'payload.period_type' => ['required', Rule::enum(Periodtype::class)],
        ];
    }

    public function GeneratePaymentData( array $payload = [] ) : Link|Item {
        return (new Link())
            ->setCurrency( Currencies::tryFrom($payload['currency'] ?? Currencies::USD) )
            ->add(new Item(
                id     : $this->public_id,
                amount : data_get($this->price, $payload['period_type'] . '.' . $payload['currency'], 25),
                name   : strtr('velents - buddy - period_type credit Credit', [
                    'period_type' => $payload['period_type'],
                    'credit'      => $this->credit,
                ]),
            ));
    }
}
```

### PaymentPlan (`app/Payment/Models/PaymentPlan.php`)

The tenant's active plan with a live credit balance.

```php
class PaymentPlan extends \App\Core\Models\Model {

    public $fillable = [
        'amount', 'credit', 'price', 'currency', 'plan_type', 'period_type',
        'payload', 'is_available', 'is_active', 'price_list',
        'started_at', 'renew_at', 'expires_at',
        'payment_plan_template_id', 'Notifications_sent',
    ];

    public $casts = [
        'payload'            => 'collection',
        'Notifications_sent' => 'collection',
        'currency'           => Currencies::class,
        'plan_type'          => PlanType::class,
        'period_type'        => Periodtype::class,
        'price_list'         => typePriceList::class,
    ];

    public function Logs() {
        return $this->hasMany(PaymentPlanLog::class)->orderBy('created_at', 'desc');
    }

    public function template() {
        return $this->belongsTo(PaymentPlanTemplate::class);
    }
}
```

### PaymentPlanLog (`app/Payment/Models/PaymentPlanLog.php`)

Audit trail for every credit deduction or addition.

```php
class PaymentPlanLog extends \App\Core\Models\Model {
    public $primaryKey   = 'created_at';
    public $incrementing = false;

    public $fillable = [
        'status', 'created_by_id', 'payload', 'credit', 'cost', 'amount', 'payment_plan_id',
    ];

    public $casts = [
        'payload' => 'collection',
    ];
}
```

---

## Enums

### PlanType
```php
enum PlanType : string {
    case Standard   = 'Standard';
    case Pro        = 'Pro';
    case Ultimate   = 'Ultimate';
    case Customized = 'Customized';
    case FreePlan   = 'Free Plan';
}
```

### Periodtype
```php
enum Periodtype : string {
    case Monthly = 'Monthly';
    case Annual  = 'Annual';
}
```

### Status
```php
enum Status : string {
    case notForPaid  = 'notForPaid';
    case waitForPaid = 'waitForPaid';
    case Paid        = 'Paid';
}
```

### Currencies
```php
enum Currencies : string {
    case USD = 'USD';
    case EGP = 'EGP';
    case SAR = 'SAR';
    case AED = 'AED';
    case EUR = 'EUR';
    case GBP = 'GBP';
    // ... 100+ currencies supported
}
```

### PaymentPlanNotification
```php
enum PaymentPlanNotification : string {
    case FiftyPercentOfUsage  = 'FiftyPercentOfUsage';
    case EightyPercentOfUsage = 'EightyPercentOfUsage';
    case AllOfUsage           = 'AllOfUsage';
    case FiveDaysRemaining    = 'FiveDaysRemaining';
    case Ended                = 'Ended';
}
```

---

## PaymentItem Contract

```php
interface PaymentItem {
    public function HavePayment() : bool;
    public function getAgent() : Agent;
    public function GeneratePaymentData( array $payload = [] ) : Link|Item;
    public function rules() : array;
}
```

---

## Payment Link Generation (Casts)

### `App\Payment\Casts\Generate\Link`

```php
class Link implements Arrayable, JsonSerializable {

    public function __construct(
        public Currencies $currency = Currencies::USD,
        protected Collection $items = new Collection,
        public array $metadata = [],
        public array $webhook = [],
        public ?string $ecommerce_id = null,
    ) {}

    public function add( Item $Item ) : self;
    public function setCurrency( Currencies $currency ) : self;
    public function setEcommerce_id( ?string $ecommerce_id ) : self;

    public function toArray() : array {
        return [
            'amount_currency' => $this->currency->value,
            'operation'       => 'purchase',
            'ecommerce_id'    => $this->ecommerce_id,
            'metadata'        => $this->metadata,
            'webhook'         => $this->webhook,
            'currency'        => $this->currency->value,
            'amount'          => $this->items->sum(fn(Item $Item) => $Item->amount * $Item->quantity),
            'items'           => $this->items->map(fn(Item $Item) => $Item->toArray())->toArray(),
        ];
    }
}
```

### `App\Payment\Casts\Generate\Item`

```php
class Item implements Arrayable, JsonSerializable {

    public function __construct(
        public float $amount,
        public string $id,
        public string $name,
        public array $metadata = [],
        public ?string $description = null,
        public string $ecommerce_id = '',
        public int $quantity = 1,
        public Currencies $currency = Currencies::USD,
    ) {}

    public function toArray() : array {
        return [
            'id'          => $this->id,
            'amount'      => $this->amount,
            'quantity'    => $this->quantity,
            'name'        => $this->name,
            'description' => $this->description ?? $this->name,
            'metadata'    => $this->metadata,
        ];
    }
}
```

---

## Services

### Payment Service (`app/Payment/Services/Payment.php`)

Generates payment links via the VelentsIntegrations service.

```php
class Payment {
    use SelfMaker, logger;

    public function __construct(
        public VelentsIntegrationsVelentsAi $VelentsIntegrationsVelentsAi
    ) {}

    public function Link( PaymentItem $Item, array $payload = [] ) : array {
        return $this->VelentsIntegrationsVelentsAi
            ->GeneratePaymentLink($Item, $Item->GeneratePaymentData($payload))
            ->json();
    }
}
```

### Plan Service (`app/Payment/Services/Plan.php`)

Core billing logic: checks credit availability, deducts credits, sends notifications.

```php
class Plan {
    use SelfMaker, logger;

    protected static PaymentPlan|null $current = null;

    public function __construct(
        public \App\Payment\Repositories\PaymentPlan $PaymentPlan
    ) {}

    // Get current active plan (cached in static property)
    public function Plan() : PaymentPlan {
        if (is_null(static::$current)) static::$current = $this->initCurrent();
        throw_if(is_null(static::$current), $this->Exception(['you havent available payment plan']));
        return static::$current;
    }

    // Pre-flight check: ensures enough credits before starting a conversation/call
    public function Can(
        ?int $minutes = null,
        ?int $messages = null,
        ?int $Whatsapps = null,
    ) : true {
        $Plan = $this->Plan();
        if ($this->Plan()->credit < 0) {
            // Dispatch AllOfUsage notification
            throw $this->Exception(['you havent credit in your payment plan']);
        }
        ['minute' => $minute, 'whatsapp' => $Whatsapp, 'message' => $message] = $this->Plan()->price_list->toArray();
        $amount = collect()
            ->merge(['minute'   => $minutes   * $minute])
            ->merge(['messages' => $messages  * $message])
            ->merge(['whatsapp' => $Whatsapps * $Whatsapp]);
        if ($this->Plan()->credit < $amount->sum()) {
            throw $this->Exception(['you havent credit in your payment plan']);
        }
        // 50% usage notification
        if ($this->Plan()->credit <= ($this->Plan()->template->credit ?? 20) * 0.5)
            dispatch(fn() => PlanMail($Plan, PaymentPlanNotification::FiftyPercentOfUsage));
        // 80% usage notification
        if ($this->Plan()->credit <= ($this->Plan()->template->credit ?? 20) * 0.2)
            dispatch(fn() => PlanMail($Plan, PaymentPlanNotification::EightyPercentOfUsage));
        return true;
    }

    // Deduct credits after usage (called post-call or post-conversation)
    public function Pay(
        float|int|null $minutes = null,
        float|int|null $messages = null,
        float|int|null $Whatsapps = null,
        string $whatHappened = '',
        array $log_payload = [],
    ) : true {
        try {
            $PaymentPlan ??= $this->Plan();
            ['minute' => $minute, 'whatsapp' => $Whatsapp, 'message' => $message] = $PaymentPlan->price_list->toArray();
            $amount = collect()
                ->merge(['minute'   => $minutes   * $minute])
                ->merge(['messages' => $messages  * $message])
                ->merge(['whatsapp' => $Whatsapps * $Whatsapp / 2]);

            // Log the deduction
            $this->PaymentPlan->PlanLog(-abs($amount->sum()), $whatHappened, $log_payload);

            // Atomic credit deduction using DB::raw
            $this->PaymentPlan->available()->update([
                'credit' => DB::raw('"credit" - ' . $amount->sum()),
            ]);
            $this->Plan()?->refresh();
        } catch (\Throwable $Throwable) {
            $this->LogError($Throwable, static::class);
        }
        return true;
    }

    // Calculate cost without deducting
    public function cost(
        float|int|null $minutes = null,
        float|int|null $messages = null,
        float|int|null $Whatsapps = null,
    ) : float;
}
```

---

## Price List Structure

The `price_list` column stores per-unit costs:

```json
{
    "minute": 0.5,
    "whatsapp": 0.3,
    "message": 0.1
}
```

Credit cost formula:
- Voice call: `duration_minutes * price_list.minute`
- Text message: `message_count * price_list.message`
- WhatsApp message: `message_count * price_list.whatsapp / 2`

---

## Payment Flow

1. **Plan purchase**: `PaymentPlanTemplate` -> `Payment::Link()` -> VelentsIntegrations generates Stripe/payment link
2. **Plan activation**: Webhook callback creates `PaymentPlan` with credit balance from template
3. **Pre-flight check**: Before starting conversation/call, `Plan::Can()` validates sufficient credits
4. **Usage deduction**: After call/conversation ends, `Plan::Pay()` atomically deducts credits
5. **Notifications**: Automatic emails at 50%, 80%, 100% usage thresholds
6. **Plan expiry**: `checkPlans` command handles expired plans

---

## Directory Structure

```
app/Payment/
  Casts/
    Generate/Item.php              -- Single line item for payment link
    Generate/Link.php              -- Payment link with multiple items
    Generate/PaymentTransaction.php
    PaymentPlanTemplate/Price.php  -- Price matrix (period x currency)
    PaymentPlanTemplate/typePrice.php
    PriceList.php                  -- Per-unit costs (minute/message/whatsapp)
    typePriceList.php
  Commands/PaymentPlan/checkPlans.php  -- Artisan command for plan expiry
  Contracts/PaymentItem.php
  Controllers/Payment.php
  Enums/Currencies.php, PaymentPlanNotification.php, Periodtype.php, PlanType.php, Status.php
  Exception/PaymentException.php
  Mail/PaymentPlan/Create.php, Usage.php
  Models/PaymentPlan.php, PaymentPlanLog.php, PaymentPlanTemplate.php
  Repositories/PaymentPlan.php, PaymentPlanTemplate.php
  Requests/DashboardPaymentPlanStore.php, ...
  Resources/PaymentPlan/Collection.php, Resource.php, ...
  Services/Payment.php, Plan.php
```
