---
name: docs-reference
description: "Official documentation references for all key technologies. Read the relevant doc file BEFORE writing code in that technology — no assumptions."
user-invocable: false
---

# Official Documentation Reference

**RULE: Before writing code in any technology below, READ the corresponding doc file first. Do not assume APIs — verify from docs.**

## Available Documentation Files

| Technology | File | Read Before |
|---|---|---|
| React 19 | `llms-docs/react.txt` (14KB) | Writing any React component, hook, or pattern |
| Next.js 16 | `llms-docs/nextjs.txt` (1.2KB) | App Router pages, server components, routing |
| Zustand | `llms-docs/zustand.txt` (3.6KB) | Creating or modifying Zustand stores |
| Zod | `llms-docs/zod.txt` (1.5KB) | Writing validation schemas |
| shadcn/ui | `llms-docs/shadcn-ui.txt` (1.2KB) | Adding or configuring shadcn/ui components |
| Vitest | `llms-docs/vitest.txt` (8.7KB) | Writing or configuring tests |
| Tiptap | `llms-docs/tiptap.txt` (59KB) | Rich text editor features |
| ElevenLabs | `llms-docs/elevenlabs.txt` (90KB) | Voice synthesis, text-to-speech integration |
| Redis | `llms-docs/redis.txt` (9.4KB) | Caching, queues, session management |
| Google AI (Gemini) | `llms-docs/google-ai.txt` | Gemini API calls, AI model integration |
| LiveKit | `llms-docs/livekit.txt` | Real-time voice/video, WebRTC rooms |

## How to Use

1. Identify which technology your current task involves
2. Use the `Read` tool to load the corresponding doc file
3. Reference the docs while implementing — don't rely on memory
4. For large docs (Tiptap 59KB, ElevenLabs 90KB), read specific sections relevant to your task

## Docs NOT Available (no llms.txt exists)

These technologies don't have llms.txt — use your training knowledge + codebase patterns:
- Laravel 12 / PHP 8.4
- Tailwind CSS 4
- TypeScript 5.9
- TanStack React Query v5
- React Hook Form
- Motion (Framer Motion)
- Lucide React
- Recharts
- Sentry
- OpenAI / Anthropic SDKs
