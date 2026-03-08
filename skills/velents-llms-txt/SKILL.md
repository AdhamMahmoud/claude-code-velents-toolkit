---
name: velents-llms-txt
description: llms.txt URLs for every technology in the Velents stack. Fetch the relevant URLs before writing code to get the latest API signatures, patterns, and breaking changes. Never rely on training data for library APIs — always fetch first.
---

# Velents Tech Stack — llms.txt Directory

## THE RULE: Fetch Before You Write

Before writing any code that uses a library, fetch its llms.txt. This prevents:
- Using deprecated APIs (e.g., Next.js 13 patterns in a Next.js 16 codebase)
- Missing new features that simplify the implementation
- Breaking changes in minor versions that affect the Velents codebase

### How to fetch
Use the WebFetch tool with the llms.txt URL and a targeted prompt:
```
WebFetch: https://nextjs.org/llms.txt
Prompt: "What are the current App Router patterns for [your specific task]? Any breaking changes from v15 to v16?"
```

---

## Frontend Stack

| Technology | Version in Velents | llms.txt URL | Fetch When |
|---|---|---|---|
| **Next.js** | 16.1.3 | https://nextjs.org/llms.txt | Writing any page, layout, route handler, middleware |
| **React** | 19.2.3 | https://react.dev/llms.txt | Writing components, hooks, server components |
| **Tailwind CSS** | ^4.1.10 | https://tailwindcss.com/llms.txt | Writing any utility classes (v4 has breaking changes from v3) |
| **shadcn/ui** | 46+ components | https://ui.shadcn.com/llms.txt | Using or adding shadcn components |
| **Radix UI** | primitives | https://radix-ui.com/llms.txt | Using Radix primitives directly |
| **TanStack Query** | ^5.90.12 | https://tanstack.com/query/latest/llms.txt | Writing data fetching, mutations, cache invalidation |
| **TanStack Table** | ^8.21.3 | https://tanstack.com/table/llms.txt | Building data tables |
| **Zustand** | ^5.0.5 | https://zustand.docs.pmnd.rs/llms.txt | Creating or modifying stores |
| **React Hook Form** | ^7.58.1 | https://react-hook-form.com/llms.txt | Building forms |
| **Zod** | ^3.25.67 | https://zod.dev/llms.txt | Writing validation schemas |
| **TypeScript** | 5.9 | https://typescriptlang.org/llms.txt | Writing complex types or generics |
| **Tiptap** | ^2.22.3 | https://tiptap.dev/llms.txt | Rich text editor features |
| **next-intl** | ^4.5.5 | https://next-intl.dev/llms.txt | i18n, locale routing, RTL |

---

## Backend Stack

| Technology | Version in Velents | Docs URL | Fetch When |
|---|---|---|---|
| **Laravel** | 12 | https://laravel.com/llms.txt | Writing controllers, middleware, jobs, events |
| **stancl/tenancy** | ^3.9 | https://tenancyforlaravel.com/llms.txt | Any multi-tenancy configuration |
| **Spatie Permission** | latest | https://spatie.be/docs/llms.txt | Roles, permissions, middleware |
| **Laravel Reverb** | ^1.6 | https://reverb.laravel.com/llms.txt | WebSocket events, channels |
| **PHPUnit** | ^11.5.3 | https://phpunit.de/llms.txt | Writing tests |

---

## External Services

| Service | llms.txt / Docs URL | Fetch When |
|---|---|---|
| **ElevenLabs** | https://elevenlabs.io/llms.txt | Voice agent, TTS, STT features |
| **LiveKit** | https://docs.livekit.io/llms.txt | WebRTC, room management |
| **Stripe** | https://stripe.com/llms.txt | Payment, subscriptions |
| **Sentry** | https://docs.sentry.io/llms.txt | Error monitoring integration |

---

## Critical Version Notes (Velents-Specific)

### Tailwind CSS v4 (BREAKING vs v3)
- No more `tailwind.config.js` — config is in CSS file
- New utility naming conventions
- **Always fetch llms.txt before using any Tailwind class** — v4 patterns differ from training data

### Next.js 16 App Router
- React Server Components are default — client components need `"use client"`
- Server Actions replace API routes for mutations in many cases
- Fetch semantics changed — no implicit caching by default
- **Fetch llms.txt before any route handler or page work**

### React 19
- New hooks: `use()`, `useFormStatus()`, `useOptimistic()`
- Server Components + Client Components boundary rules updated
- **Fetch llms.txt when using any React hook**

### TanStack Query v5
- `useQuery` API changed — `isLoading` → `isPending`
- `cacheTime` → `gcTime`
- **Fetch llms.txt before writing any query hook**

---

## Fetch Protocol

For every technology you will use in a task:

1. Identify which technologies from the table above are involved
2. Fetch each relevant llms.txt URL using WebFetch
3. Read the response specifically for the patterns you need
4. Apply those patterns — do NOT default to training data for API signatures

### Example: Writing a new Next.js page with data fetching

```
# Before writing, fetch:
WebFetch: https://nextjs.org/llms.txt — "App Router page with server-side data fetching pattern in Next.js 16"
WebFetch: https://tanstack.com/query/latest/llms.txt — "useQuery pattern in TanStack Query v5, especially isPending vs isLoading"
WebFetch: https://ui.shadcn.com/llms.txt — "DataTable component usage"

# Then write the code using the patterns from the fetched docs
```

### Example: Writing a new Laravel service

```
# Before writing, fetch:
WebFetch: https://laravel.com/llms.txt — "Service class patterns, job dispatch, event broadcasting in Laravel 12"

# Then write the code
```
