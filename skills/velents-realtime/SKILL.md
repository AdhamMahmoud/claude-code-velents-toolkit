---
name: velents-realtime
description: Real-time architecture for VelentsAI — Laravel Reverb WebSocket, Echo client, Pusher protocol, conversation/call event broadcasting
user-invocable: false
---

# VelentsAI Real-time Architecture

## Overview

VelentsAI uses **Laravel Reverb** (WebSocket server) on the backend with **Laravel Echo + Pusher.js** on the frontend. All events flow through public channels using the Pusher protocol over WebSocket.

```
Backend (Laravel)                    Frontend (Next.js)
+-------------------+               +-------------------+
| Event classes     |               | lib/echo.ts       |
| extend            |  ws/wss       | Laravel Echo +    |
| Broadcasting      | ------------> | Pusher.js client  |
| (ShouldBroadcast) |  port 8080    |                   |
+-------------------+               +-------------------+
        |                                    |
  Redis queue                         socket.channel()
  connection='redis'                  .listen(".EventName")
  queue='default'
```

---

## 1. Echo Client Configuration (`lib/echo.ts`)

```ts
import Echo from "laravel-echo";
import Pusher from "pusher-js";

export const socket = new Echo({
  broadcaster: "reverb",
  key: process.env.NEXT_PUBLIC_REVERB_APP_KEY,

  wsHost: process.env.NEXT_PUBLIC_REVERB_HOST,
  wssHost: process.env.NEXT_PUBLIC_REVERB_HOST,

  wsPort: Number(process.env.NEXT_PUBLIC_REVERB_PORT),
  wssPort: Number(process.env.NEXT_PUBLIC_REVERB_PORT),

  forceTLS: process.env.NEXT_PUBLIC_REVERB_SCHEME === "https",
  encrypted: process.env.NEXT_PUBLIC_REVERB_SCHEME === "https",

  enabledTransports: ["ws", "wss"],
  disableStats: true,

  Pusher: Pusher,
});
```

### Required Environment Variables (Frontend)

```env
NEXT_PUBLIC_REVERB_APP_KEY=your-reverb-app-key
NEXT_PUBLIC_REVERB_HOST=localhost       # or reverb.velents.ai in production
NEXT_PUBLIC_REVERB_PORT=8080
NEXT_PUBLIC_REVERB_SCHEME=http          # or https in production
```

### Key Details

- The `socket` export is a **module-level singleton** -- imported wherever needed.
- `broadcaster: "reverb"` tells Echo to use the Reverb/Pusher-compatible protocol.
- Both `ws` and `wss` transports are enabled; TLS is toggled by `REVERB_SCHEME`.
- `disableStats: true` prevents Pusher analytics pings.

---

## 2. Backend Broadcasting Base Class

All broadcast events extend a shared abstract `Broadcasting` class.

```php
<?php
namespace App\Core\Support;

abstract class Broadcasting
    implements \Illuminate\Contracts\Broadcasting\ShouldBroadcast,
               \Illuminate\Contracts\Events\ShouldDispatchAfterCommit
{
    use \Illuminate\Queue\SerializesModels,
        \Illuminate\Foundation\Events\Dispatchable,
        \Illuminate\Broadcasting\InteractsWithSockets,
        \App\Core\Helpers\InitTenant,
        \App\Core\Helpers\SelfMaker,
        \App\Core\Helpers\Url,
        \App\Core\Helpers\Storage,
        \App\Core\Helpers\logger;

    public $connection = 'redis';
    public $queue = 'default';

    function run(): void {
        event($this);
    }
}
```

Key traits of the base class:
- Implements `ShouldBroadcast` -- events are dispatched to Reverb via Redis queue.
- Implements `ShouldDispatchAfterCommit` -- events only fire after the DB transaction commits.
- Uses `$connection = 'redis'` and `$queue = 'default'`.
- Includes `InitTenant` trait for multi-tenant context resolution.
- The `run()` method dispatches itself as an event.

---

## 3. Event Classes

### 3a. Conversation Broadcasting (base)

Channel: `{conversation_public_id}` (public channel)
Event name: `.conversation`

```php
<?php
namespace App\Conversations\Events;

use Illuminate\Broadcasting\Channel;

class Broadcasting extends \App\Core\Support\Broadcasting
    implements \Illuminate\Contracts\Queue\ShouldBeUniqueUntilProcessing
{
    public function __construct(
        public string $conversation_public_id,
        public array $payload
    ) {}

    public function uniqueId(): string {
        return $this->conversation_public_id;
    }

    public function broadcastAs(): string {
        return 'conversation';
    }

    public function broadcastOn(): array {
        return [
            new Channel($this->conversation_public_id),
        ];
    }
}
```

### 3b. Conversation Message Event

Inherits the channel from `Broadcasting`, overrides only the event name.

```php
<?php
namespace App\Conversations\Events;

class Message extends Broadcasting {
    public function broadcastAs(): string {
        return 'Message';
    }
}
```

### 3c. Conversation Update Event

```php
<?php
namespace App\Conversations\Events;

class Update extends Broadcasting {
    public function broadcastAs(): string {
        return 'Update';
    }
}
```

### 3d. FindMyTenant (Auth Event)

Channel: `{request_id}` (unique per login attempt)
Event name: `.login`

```php
<?php
namespace App\Core\Events\Auth;

class FindMyTenant extends \App\Core\Support\Broadcasting {
    public function __construct(
        public string $request_id,
        public array $tenants
    ) {}

    public function broadcastAs(): string {
        return 'login';
    }

    public function broadcastOn(): array {
        return [
            new \Illuminate\Broadcasting\Channel($this->request_id),
        ];
    }
}
```

---

## 4. Frontend Listening Patterns

### Pattern: Login Flow (FindMyTenant via WebSocket)

The login page generates a `request_id`, sends it to `POST /Auth/FindMyTenant`, then listens on a channel named after that `request_id` for the `.login` event containing tenant data.

```ts
// app/[locale]/dashboard/(guest)/login/v1/page.tsx
import { socket } from "@/lib/echo";

useEffect(() => {
  if (isOpenWebsocket) {
    const channel = socket.channel(requestForFindTenant);

    channel.listen(".login", (data: any) => {
      if (data.tenants.length === 1) {
        // Auto-login with single tenant
        form.setValue("tenant_name", data.tenants[0]);
        onSubmit(form.getValues());
      } else if (data.tenants.length > 1) {
        // Show tenant selector modal
        setTenantOptions(data.tenants);
        setIsTenantModalOpen(true);
      } else if (data.tenants.length === 0) {
        toast.error(t("noWorkspaceRegister"));
      }
    });
  }
  return () => {
    socket.leave(requestForFindTenant);
  };
}, [isOpenWebsocket]);
```

### Pattern: Live Conversation Messages

Subscribe to a conversation channel using the conversation's `public_id`. Listen for `.Message` (new messages) and `.Update` (status changes, extraction updates).

```ts
// app/[locale]/dashboard/(auth)/build/inbox/components/chat-content.tsx
import { socket } from "@/lib/echo";

useEffect(() => {
  if (selectedChat) {
    const channel = socket.channel(selectedChat.public_id);

    // Listen for new messages (from AI agent or user on other channel)
    channel.listen(".Message", ({ payload }: any) => {
      if (payload.role === "human-agent") return;  // Skip own echoes
      useChatStore.getState().appendIncomingMessage(payload);
    });

    // Listen for status/extraction updates
    channel.listen(".Update", ({ payload }: any) => {
      if (payload.status) {
        useChatStore.getState().changeConversationStatus(payload);
      }
    });
  }

  return () => {
    socket.leave(selectedChat?.public_id!);
  };
}, [selectedChat]);
```

---

## 5. Channel Naming Convention

| Context | Channel Name | Event(s) | Data Shape |
|---------|-------------|----------|------------|
| Login | `{request_id}` (UUID) | `.login` | `{ tenants: string[] }` |
| Conversation | `{conversation_public_id}` | `.conversation` | `{ payload: any }` |
| Conversation | `{conversation_public_id}` | `.Message` | `{ payload: { role, response, ... } }` |
| Conversation | `{conversation_public_id}` | `.Update` | `{ payload: { status, ... } }` |

All channels are **public** (not `private-` or `presence-`). The channel name is the raw identifier string.

---

## 6. Reverb Server Configuration (`config/reverb.php`)

```php
return [
    'default' => env('REVERB_SERVER', 'reverb'),
    'servers' => [
        'reverb' => [
            'host' => env('REVERB_SERVER_HOST', '0.0.0.0'),
            'port' => env('REVERB_SERVER_PORT', 8080),
            'hostname' => env('REVERB_HOST'),
            'options' => [ 'tls' => [] ],
            'max_request_size' => env('REVERB_MAX_REQUEST_SIZE', 10_000),
            'scaling' => [
                'enabled' => env('REVERB_SCALING_ENABLED', false),
                'channel' => env('REVERB_SCALING_CHANNEL', 'reverb'),
                'server' => [
                    'url' => env('REDIS_URL'),
                    'host' => env('REDIS_HOST', '127.0.0.1'),
                    'port' => env('REDIS_PORT', '6379'),
                ],
            ],
            'pulse_ingest_interval' => env('REVERB_PULSE_INGEST_INTERVAL', 15),
        ],
    ],
    'apps' => [
        'provider' => 'config',
        'apps' => [[
            'key' => env('REVERB_APP_KEY'),
            'secret' => env('REVERB_APP_SECRET'),
            'app_id' => env('REVERB_APP_ID'),
            'options' => [
                'host' => env('REVERB_HOST'),
                'port' => env('REVERB_PORT', 443),
                'scheme' => env('REVERB_SCHEME', 'https'),
                'useTLS' => env('REVERB_SCHEME', 'https') === 'https',
            ],
            'allowed_origins' => ['*'],
            'ping_interval' => env('REVERB_APP_PING_INTERVAL', 60),
            'activity_timeout' => env('REVERB_APP_ACTIVITY_TIMEOUT', 30),
        ]],
    ],
];
```

## 7. Broadcasting Configuration (`config/broadcasting.php`)

```php
return [
    'default' => env('BROADCAST_CONNECTION', 'reverb'),
    'connections' => [
        'reverb' => [
            'driver' => 'reverb',
            'key'    => env('REVERB_APP_KEY'),
            'secret' => env('REVERB_APP_SECRET'),
            'app_id' => env('REVERB_APP_ID'),
            'options' => [
                'host' => env('REVERB_HOST'),
                'port' => env('REVERB_PORT', 443),
                'scheme' => env('REVERB_SCHEME', 'https'),
                'useTLS' => env('REVERB_SCHEME', 'https') === 'https',
            ],
        ],
    ],
];
```

---

## 8. Required Backend Environment Variables

```env
BROADCAST_CONNECTION=reverb
REVERB_APP_ID=your-app-id
REVERB_APP_KEY=your-app-key
REVERB_APP_SECRET=your-app-secret
REVERB_HOST=localhost
REVERB_PORT=8080
REVERB_SCHEME=http
REVERB_SERVER_HOST=0.0.0.0
REVERB_SERVER_PORT=8080
```

Start the WebSocket server: `php artisan reverb:start`

---

## 9. Implementation Checklist for New Events

1. **Create event class** extending `App\Core\Support\Broadcasting` (or a domain-specific subclass).
2. Define `broadcastAs()` returning the event name string.
3. Define `broadcastOn()` returning a `Channel` with the appropriate identifier.
4. Pass data via constructor properties (they auto-serialize to the broadcast payload).
5. Dispatch with `EventClass::make(...)->run()` or `event(new EventClass(...))`.
6. On the frontend, subscribe: `socket.channel(channelName).listen(".EventName", callback)`.
7. Clean up on unmount: `socket.leave(channelName)`.
