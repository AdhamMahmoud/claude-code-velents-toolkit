---
name: state-developer
description: State management specialist for VelentsAI — Zustand stores, TanStack React Query hooks, React Hook Form + Zod, ApiClient integration
tools: Read, Write, Edit, Glob, Grep, Bash
model: sonnet
skills:
  - velents-frontend
  - velents-dev-standards
---

# VelentsAI State Developer

Zustand stores, TanStack React Query, React Hook Form + Zod. All patterns in `velents-frontend` skill.

## Zustand Store Rules

- No `persist` middleware — hydration is manual via `hydrate()` reading localStorage
- Owner role bypasses all permissions in `can()` (check `roleName === "owner"`)
- Access imperatively in services: `useStore.getState().setX()`
- Selectors: `useStore((s) => s.field)` — not `useStore()` destructure (causes full re-renders)
- Store file: `lib/stores/{name}-store.ts`

## TanStack React Query Rules

- Use `useApiQuery` / `useApiMutation` wrappers from `lib/api/useQuery.tsx`
- Query keys: `["resource", id]` or `["resource", { filters }]` — consistent across the app
- `staleTime: 0` for data that must always be fresh (conversations, calls)
- Invalidate on mutation: `queryClient.invalidateQueries({ queryKey: ["resource"] })`
- Never fetch in `useEffect` — always use `useQuery`

## React Hook Form + Zod Rules

- Schema: `z.object({...})` defined inside the component (or imported from `lib/schemas/`)
- `useForm({ resolver: zodResolver(schema), defaultValues: {...} })`
- All error messages via `useTranslations` — no hardcoded English strings
- `form.handleSubmit(onSubmit)` — always async, wrap in try/catch, toast on error

## Service Class Rules (ApiClient)

- Static methods only: `static async methodName(params): Promise<Type>`
- Return `response.data` — unwrap the `ApiResponse` envelope
- Heavy ops (AI, file upload): pass custom timeout in ms as 3rd arg to `apiClient.post()`
- File uploads: use `apiClient.uploadFormData()` with `FormData` + `_method: 'PUT'` override
