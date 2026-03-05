---
name: velents-frontend
description: Next.js 16 frontend patterns for VelentsAI Agent Hub — ApiClient, service classes, Zustand stores, TanStack Query, React Hook Form, WebSocket, i18n
user-invocable: false
---

# VelentsAI Frontend Patterns (agent-hub)

## Tech Stack (Exact Versions)

| Package | Version |
|---------|---------|
| next | 16.1.3 |
| react / react-dom | 19.2.3 |
| typescript | 5.9.3 |
| tailwindcss | ^4.1.10 |
| zustand | ^5.0.5 |
| @tanstack/react-query | ^5.90.12 |
| @tanstack/react-table | ^8.21.3 |
| react-hook-form | ^7.58.1 |
| @hookform/resolvers | ^5.1.1 |
| zod | ^3.25.67 |
| next-intl | ^4.5.5 |
| laravel-echo | ^2.2.6 |
| pusher-js | ^8.4.0 |
| sonner | ^2.0.6 |
| lucide-react | ^0.522.0 |
| motion | ^12.19.1 |
| recharts | ^3.6.0 |
| @sentry/nextjs | ^10.29.0 |

UI primitives: Full shadcn/ui suite via Radix (dialog, dropdown-menu, select, tabs, toast, tooltip, popover, accordion, etc.) with `class-variance-authority`, `clsx`, `tailwind-merge`.

Rich text: Tiptap (`@tiptap/react ^2.22.3`) with extensions for code-block-lowlight, color, heading, image, link, placeholder, underline.

Drag-and-drop: `@dnd-kit/core ^6.3.1`, `@hello-pangea/dnd ^18.0.1`.

---

## 1. API Client Singleton (`lib/api/client.ts`)

A custom fetch-based HTTP client with multi-tenant URL resolution, in-memory token caching, and typed error handling. No axios -- pure `fetch`.

```ts
import { API_CONFIG, HTTP_METHODS } from "./config";
import { ApiResponse, PaginatedResponse, PaginationParams } from "./types";

interface RequestConfig {
  method: keyof typeof HTTP_METHODS;
  url: string;
  data?: any;
  params?: Record<string, any>;
  headers?: Record<string, string>;
  timeout?: number;
}

class ApiClient {
  private baseURL: string;
  private defaultHeaders: Record<string, string>;
  private timeout: number;
  private cachedToken: string | null = null;

  constructor() {
    this.baseURL = this.getDynamicBaseURL();
    this.timeout = API_CONFIG.timeout;
    this.defaultHeaders = { "Content-Type": "application/json" };
  }

  // Dynamic base URL from tenant data in localStorage
  private getDynamicBaseURL(tenantName?: string): string {
    const defaultURL = process.env.NEXT_PUBLIC_API_URL || "/api";
    const isLocal = process.env.NODE_ENV !== "production" && defaultURL.includes("localhost");

    if (tenantName) {
      if (isLocal) return `http://${tenantName}.localhost:8000`;
      return `https://${tenantName}.${defaultURL!.replace("https://", "")}`;
    }
    if (typeof window !== "undefined") {
      const tenantData = localStorage.getItem("tenant_data");
      if (tenantData) {
        try {
          const tenant = JSON.parse(tenantData);
          if (tenant.id) {
            if (isLocal) return `http://${tenant.id}.localhost:8000`;
            return `https://${tenant.id}.${defaultURL!.replace("https://", "")}`;
          }
        } catch (error) {
          console.warn("Failed to parse tenant data for API URL:", error);
        }
      }
    }
    if (isLocal) return `http://${defaultURL}`;
    return defaultURL;
  }

  getLocaleFromCookie(locale: "ar" | "en") {
    this.defaultHeaders["Accept-Language"] = locale;
  }

  updateBaseURL(tenantName?: string): void {
    this.baseURL = this.getDynamicBaseURL(tenantName);
  }

  setAuthToken(token: string) {
    this.cachedToken = token;
    this.defaultHeaders["Authorization"] = `Bearer ${token}`;
  }

  removeAuthToken() {
    this.cachedToken = null;
    delete this.defaultHeaders["Authorization"];
  }

  private getToken(): string | null {
    if (this.cachedToken) return this.cachedToken;
    if (typeof window !== "undefined") {
      const token = localStorage.getItem("auth_token");
      if (token) this.cachedToken = token;
      return token;
    }
    return null;
  }

  private buildQueryString(params: Record<string, any>): string {
    const searchParams = new URLSearchParams();
    Object.entries(params).forEach(([key, value]) => {
      if (value !== undefined && value !== null) {
        if (Array.isArray(value)) {
          value.forEach((item) => searchParams.append(key, item.toString()));
        } else {
          searchParams.append(key, value.toString());
        }
      }
    });
    return searchParams.toString();
  }

  private async request<T>(config: RequestConfig): Promise<ApiResponse<T>> {
    const { method, url, data, params, headers = {}, timeout } = config;
    const token = this.getToken();
    if (token) this.setAuthToken(token);

    let fullUrl = `${this.baseURL}${url}`;
    if (params) {
      const queryString = this.buildQueryString(params);
      if (queryString) fullUrl += `?${queryString}`;
    }

    const requestOptions: RequestInit = {
      method,
      headers: { ...this.defaultHeaders, ...headers },
      signal: AbortSignal.timeout(timeout || this.timeout),
    };
    if (data && method !== HTTP_METHODS.GET) {
      requestOptions.body = JSON.stringify(data);
    }

    try {
      const response = await fetch(fullUrl, requestOptions);
      if (!response.ok) {
        let errorMessage;
        let errorCode: string | undefined;
        let errorDetails: any;
        try {
          const errorData = await response.json();
          errorMessage = errorData.status?.message || errorData.message;
          errorCode = errorData.code;
          errorDetails = errorData.errors ?? errorData.details;
        } catch {
          errorMessage = response.statusText || errorMessage;
        }
        // Dispatch custom event on 403 for global forbidden handler
        if (response.status === 403 && typeof window !== "undefined") {
          window.dispatchEvent(new CustomEvent("api:forbidden", { detail: { message: errorMessage } }));
        }
        throw new ApiError(errorMessage, response.status, { code: errorCode, data: errorDetails });
      }
      const responseData = await response.json();
      return { ...responseData, success: true };
    } catch (error) {
      if (error instanceof ApiError) throw error;
      if (error instanceof Error) {
        if (error.name === "AbortError") throw new ApiError("Request timeout", 408);
        throw new ApiError(error.message, 0);
      }
      throw new ApiError("Unknown error occurred", 0);
    }
  }

  // Convenience HTTP methods
  async get<T>(url: string, params?: Record<string, any>): Promise<ApiResponse<T>> {
    return this.request<T>({ method: HTTP_METHODS.GET, url, params });
  }
  async post<T>(url: string, data?: any, timeout?: number): Promise<ApiResponse<T>> {
    return this.request<T>({ method: HTTP_METHODS.POST, url, data, timeout });
  }
  async put<T>(url: string, data?: any, timeout?: number): Promise<ApiResponse<T>> {
    return this.request<T>({ method: HTTP_METHODS.PUT, url, data, timeout });
  }
  async patch<T>(url: string, data?: any, timeout?: number): Promise<ApiResponse<T>> {
    return this.request<T>({ method: HTTP_METHODS.PATCH, url, data, timeout });
  }
  async delete<T>(url: string, data?: any, timeout?: number): Promise<ApiResponse<T>> {
    return this.request<T>({ method: HTTP_METHODS.DELETE, url, data, timeout });
  }

  // Paginated requests
  async getPaginated<T>(url: string, pagination: PaginationParams = {}): Promise<PaginatedResponse<T>> {
    const response = await this.get<PaginatedResponse<T>>(url, pagination);
    return response.data;
  }

  // File upload (strips Content-Type to let browser set multipart boundary)
  async uploadFormData<T>(url: string, formData: FormData, timeout?: number): Promise<ApiResponse<T>> {
    const token = this.getToken();
    if (token) this.setAuthToken(token);
    const fullUrl = `${this.baseURL}${url}`;
    const requestOptions: RequestInit = {
      method: "POST",
      headers: {
        ...Object.fromEntries(
          Object.entries(this.defaultHeaders).filter(([key]) => key !== "Content-Type")
        ),
      },
      body: formData,
      signal: AbortSignal.timeout(timeout || this.timeout),
    };
    const response = await fetch(fullUrl, requestOptions);
    if (!response.ok) { /* same error handling pattern */ }
    const responseData = await response.json();
    return { data: responseData.data || responseData, message: responseData.message, success: true, status: response.status };
  }
}

export class ApiError extends Error {
  status: number;
  code?: string;
  data?: unknown;
  isApiError = true;

  constructor(message: string, status: number, opts?: { code?: string; data?: unknown; cause?: unknown }) {
    super(message);
    this.name = "ApiError";
    this.status = status;
    this.code = opts?.code;
    this.data = opts?.data;
  }
}

export function isApiError(err: unknown): err is ApiError {
  return !!err && typeof err === "object" && (err as any).isApiError === true;
}

// Singleton export
export const apiClient = new ApiClient();
```

Key patterns:
- **Multi-tenant URL**: `https://{tenant_id}.velents-agents-test.velents.ai` (prod) or `http://{tenant_id}.localhost:8000` (dev).
- **In-memory token cache**: Avoids localStorage reads on every request.
- **403 custom event**: `window.dispatchEvent(new CustomEvent("api:forbidden"))` triggers a global toast via `useForbiddenHandler`.
- **Timeout via AbortSignal**: `AbortSignal.timeout(timeout || this.timeout)`.

---

## 2. API Configuration (`lib/api/config.ts`)

Centralized endpoint registry with static strings and parameterized functions.

```ts
export const API_CONFIG = {
  baseURL: process.env.NEXT_PUBLIC_API_URL || "/api",
  timeout: 30000,
  retries: 3,
  retryDelay: 1000,
} as const;

export const API_ENDPOINTS = {
  auth: {
    login: "/Auth/login",
    loginTenant: "/Auth/loginByTenant",
    findMyTenant: "/Auth/FindMyTenant",
    logout: "/auth/logout",
    refresh: "/auth/refresh",
    profile: "/Auth/me",
    register: "/Auth/registration",
    updateProfile: "/Auth/update",
    acceptInvitation: "/Auth/acceptInvitation",
    invitationInfo: "/Auth/invitationInfo",
    requestInvitation: "/Auth/requestInvitation",
    declineInvitation: "/Auth/declineInvitation",
  },
  agents: {
    list: "/Agent",
    create: "/Agent",
    get: (id: string) => `/Agent/${id}`,
    update: (id: string) => `/Agent/${id}`,
    delete: (id: string) => `/Agent/${id}`,
    assignKnowledge: (id: string) => `/Agent/${id}/knowledge`,
    deleteKnowledge: `/Agent/knowledge`,
    updateChannels: (id: string) => `/Agent/${id}/Channels`,
    updateTools: (id: string) => `/Agent/${id}/Tools`,
    updateSchema: (id: string) => `/Agent/${id}/Schema`,
    dashboard: (id: string) => `/Agent/${id}/dashboard`,
    archive: (id: string) => `/Agent/${id}/archive`,
    unarchive: (id: string) => `/Agent/${id}/unarchive`,
  },
  conversations: {
    list: "/Conversations",
    create: "/Conversations",
    update: (id: string) => `/Conversations/${id}`,
    sendMessage: (id: string) => `/Conversations/${id}/Send`,
    history: (id: string) => `/Conversations/${id}/History`,
    end: (id: string) => `/Conversations/${id}/End`,
    uploadFiles: (id: string) => `/Conversations/${id}/uploudFiles`,
  },
  calls: {
    create: "/Calls",
    list: "/Calls",
    export: "/extraction",
    archive: "/Calls/archive",
    unarchive: "/Calls/unarchive",
  },
  batchCalls: {
    list: "/Batch",
    cancel: (callId: number | string) => `/Calls/${callId}/pull`,
    createWithAgent: `/Batch`,
    get: (id: string) => `/Batch/${id}`,
  },
  roles: {
    list: "/Role",
    create: "/Role",
    get: (id: string) => `/Role/${id}`,
    update: (id: string) => `/Role/${id}`,
    delete: (id: string) => `/Role/${id}`,
    permissions: "/Role/permissions",
  },
  staff: {
    list: "/Staff",
    get: (id: string) => `/Staff/${id}`,
    update: (id: string) => `/Staff/${id}`,
    delete: (id: string) => `/Staff/${id}`,
    changeRole: (id: string) => `/Staff/${id}/role`,
    transferOwnership: (id: string) => `/Staff/${id}/transfer-ownership`,
  },
  invitation: {
    list: "/Invitation",
    create: "/Invitation",
    cancel: (id: string) => `/Invitation/${id}/cancel`,
    resend: (id: string) => `/Invitation/${id}/resend`,
  },
  auditLogs: {
    list: "/AuditLog",
    get: (id: string) => `/AuditLog/${id}`,
    export: "/AuditLog/export",
  },
  payment: {
    plans: "/Payment/Templates",
    create: "/Payment/Link",
    get: "/Payment/Plan",
    logs: (id: string) => `/Payment/${id}/logs`,
  },
  agentAnalytics: {
    overview: (agentId: string) => `/Agent/${agentId}/Analytics/overview`,
    trends: (agentId: string) => `/Agent/${agentId}/Analytics/trends`,
    tools: (agentId: string) => `/Agent/${agentId}/Analytics/tools`,
    issues: (agentId: string) => `/Agent/${agentId}/Analytics/issues`,
    conversations: (agentId: string) => `/Agent/${agentId}/Analytics/conversations`,
    backfill: (agentId: string) => `/Agent/${agentId}/Analytics/backfill`,
  },
  toolsManagement: {
    generateFromUpload: "/ToolsManagement/generate/upload",
    generateFromText: "/ToolsManagement/generate/text",
    generateFromUrl: "/ToolsManagement/generate/url",
    list: "/ToolsManagement/tools",
    get: (toolId: string) => `/ToolsManagement/tools/${toolId}`,
    delete: (toolId: string) => `/ToolsManagement/tools/${toolId}`,
    updateStatus: (toolId: string) => `/ToolsManagement/tools/${toolId}/status`,
    updateAuth: (toolId: string) => `/ToolsManagement/tools/${toolId}/auth`,
  },
  // Also: users, apiKeys, tasks, products, orders, leads, files, events, slots, providers,
  // channels, tools, staticTypes, whatsapp, genesys, tokens
} as const;

export const HTTP_METHODS = {
  GET: "GET", POST: "POST", PUT: "PUT", PATCH: "PATCH", DELETE: "DELETE",
} as const;
```

Pattern: Endpoints use **static strings** for list/create and **functions** `(id: string) => \`/Resource/${id}\`` for parameterized routes.

---

## 3. Service Class Pattern (`lib/api/services/*.service.ts`)

All services use **static methods** calling the `apiClient` singleton. No instantiation needed.

### AgentService (example)

```ts
import { apiClient } from "../client";
import { API_ENDPOINTS } from "../config";

export class AgentService {
  static async getAgents(params?: PaginationParams): Promise<any> {
    const sorting = params?.sorting || [{ by: "created_at", as: "desc" as const }];
    const flattenedParams: Record<string, any> = {
      ...params,
      ...(sorting.length > 0 && {
        [`sorting[0][by]`]: sorting[0].by,
        [`sorting[0][as]`]: sorting[0].as,
      }),
    };
    if (params?.status && params.status.length > 0) {
      flattenedParams["status[]"] = params.status;
    }
    delete flattenedParams.status;
    delete flattenedParams.sorting;
    return await apiClient.get<any>(API_ENDPOINTS.agents.list, flattenedParams);
  }

  static async getAgent(id: string): Promise<Agent> {
    const response = await apiClient.get<Agent>(API_ENDPOINTS.agents.get(id));
    return response.data;
  }

  static async createAgent(agentData: CreateAgentRequest): Promise<Agent> {
    const response = await apiClient.post<Agent>(API_ENDPOINTS.agents.create, agentData);
    return response.data;
  }

  static async updateAgent(id: string, agentData: UpdateAgentRequest): Promise<Agent> {
    const response = await apiClient.put<Agent>(API_ENDPOINTS.agents.update(id), agentData);
    return response.data;
  }

  static async deleteAgent(id: string): Promise<void> {
    await apiClient.delete(API_ENDPOINTS.agents.delete(id));
  }

  static async toggleAgentStatus(agentId: string, status: "active" | "unactive"): Promise<Agent> {
    const response = await apiClient.patch<Agent>(
      `${API_ENDPOINTS.agents.get(agentId)}/status`,
      { status },
      6000 // custom timeout
    );
    return response.data;
  }

  // File upload uses uploadFormData with _method override for PUT
  static async assignKnowledge(agentId: string, knowledgeFile?: File, link?: string): Promise<KnowledgeAssignmentResponse> {
    const formData = new FormData();
    formData.append("_method", "PUT");
    if (knowledgeFile) formData.append("knowledge", knowledgeFile);
    if (link) formData.append("link", link);
    const response = await apiClient.uploadFormData<KnowledgeAssignmentResponse>(
      API_ENDPOINTS.agents.assignKnowledge(agentId),
      formData,
      100000 // 100s timeout for heavy processing
    );
    return response.data;
  }

  static async addChannels(agentId: string, attachChannels: any[], detachChannels: number[], activeChannels: number[]) {
    return (await apiClient.patch<UpdateAgentChannelsResponse>(
      API_ENDPOINTS.agents.updateChannels(agentId),
      {
        ...(attachChannels.length > 0 && { attach: attachChannels }),
        ...(detachChannels.length > 0 && { detach: detachChannels }),
        ...(activeChannels.length > 0 && { active: activeChannels }),
      },
      60000
    )).data;
  }

  static async updateSchema(agentId: string, schema: SchemaField[]) {
    return (await apiClient.patch<UpdateSchemaResponse>(
      API_ENDPOINTS.agents.updateSchema(agentId),
      { schema },
      6000
    )).data;
  }
}
```

### ConversationService (example)

```ts
export class ConversationService {
  static async createConversation(conversationData: CreateConversationRequest): Promise<Conversation> {
    const response = await apiClient.post<Conversation>(API_ENDPOINTS.conversations.create, conversationData);
    return response.data;
  }

  static async sendMessage(conversationId: string, messageData: SendMessageRequest): Promise<ConversationMessage> {
    const response = await apiClient.post<ConversationMessage>(
      API_ENDPOINTS.conversations.sendMessage(conversationId),
      messageData,
      100000 // 100s timeout for AI processing
    );
    return response.data;
  }

  static async endConversation(conversationId: string, endData: EndConversationRequest): Promise<Conversation> {
    const response = await apiClient.post<Conversation>(
      API_ENDPOINTS.conversations.end(conversationId),
      endData,
      100000
    );
    return response.data;
  }
}
```

### AuthService (example)

```ts
export class AuthService {
  static async login({ email, password }: LoginRequest): Promise<userData> {
    apiClient.updateBaseURL();
    const response = await apiClient.post<userData>(API_ENDPOINTS.auth.login, { email, password }, 100000);
    if (typeof window !== "undefined") {
      AuthService.storeSessionData(response.data);
      const { staff } = Array.isArray(response.data) ? response.data[0] : response.data;
      useAuthStore.getState().setAuth(staff);
      await ConfigService.fetchAndStoreAllConfig();
    }
    return response.data;
  }

  static async loginWithTenant({ email, password, tenant_name }: LoginRequest): Promise<userData> {
    apiClient.updateBaseURL(tenant_name?.trim());
    const response = await apiClient.post<userData>(API_ENDPOINTS.auth.loginTenant, { email, password }, 100000);
    if (typeof window !== "undefined") {
      AuthService.storeSessionData(response.data, false);
      const { staff } = Array.isArray(response.data) ? response.data[0] : response.data;
      useAuthStore.getState().setAuth(staff);
      await ConfigService.fetchAndStoreAllConfig();
    }
    return response.data;
  }

  static async logout(): Promise<void> {
    try { /* optional server logout */ } finally {
      AuthService.clearSessionData();
      useAuthStore.getState().clearAuth();
      ConfigService.clearStoredConfig();
      apiClient.removeAuthToken();
      apiClient.updateBaseURL();
      AuthService.removeChatWidget();
    }
  }

  private static storeSessionData(data: userData, updateBaseURL?: boolean): void {
    const { staff, tenant, token } = Array.isArray(data) ? data[0] : data;
    localStorage.setItem("auth_token", token.access_token);
    localStorage.setItem("refresh_token", token.refresh_token);
    localStorage.setItem("token_type", token.token_type);
    localStorage.setItem("token_expires_in", token.expires_in.toString());
    apiClient.setAuthToken(token.access_token);
    localStorage.setItem("staff_data", JSON.stringify(staff));
    localStorage.setItem("tenant_data", JSON.stringify(tenant));
    if (updateBaseURL) apiClient.updateBaseURL(tenant.id);
  }

  static isAuthenticated(): boolean {
    if (typeof window === "undefined") return false;
    return !!localStorage.getItem("auth_token");
  }
}
```

Key patterns for all services:
- All methods are **static async**.
- Return `response.data` (unwrap the `ApiResponse` envelope).
- Heavy operations use extended timeouts (60s-100s).
- File uploads use `FormData` with `_method` override for Laravel PUT compatibility.

---

## 4. Zustand Auth Store (`lib/auth/auth-store.ts`)

```ts
import { create } from "zustand";
import { Staff, StaffRole } from "@/lib/api/types";

interface AuthState {
  staff: Staff | null;
  role: StaffRole | null;
  permissions: string[];
  isHydrated: boolean;

  setAuth: (staff: Staff) => void;
  clearAuth: () => void;
  hydrate: () => void;

  can: (permission: string) => boolean;
  canAny: (permissions: string[]) => boolean;
  canAll: (permissions: string[]) => boolean;
  isOwner: () => boolean;
  isAdmin: () => boolean;
}

export const useAuthStore = create<AuthState>((set, get) => ({
  staff: null,
  role: null,
  permissions: [],
  isHydrated: false,

  setAuth: (staff: Staff) => {
    const role = staff.role_details ?? null;
    const permissions = staff.permissions ?? [];
    set({ staff, role, permissions, isHydrated: true });
    if (typeof window !== "undefined") {
      localStorage.setItem("staff_data", JSON.stringify(staff));
    }
  },

  clearAuth: () => {
    set({ staff: null, role: null, permissions: [], isHydrated: false });
  },

  hydrate: () => {
    if (typeof window === "undefined") return;
    const staffStr = localStorage.getItem("staff_data");
    if (staffStr) {
      try {
        const staff: Staff = JSON.parse(staffStr);
        set({
          staff,
          role: staff.role_details ?? null,
          permissions: staff.permissions ?? [],
          isHydrated: true,
        });
      } catch {
        set({ isHydrated: true });
      }
    } else {
      set({ isHydrated: true });
    }
  },

  can: (permission: string) => {
    const { role, staff, permissions } = get();
    const roleName = ((typeof role === "string" ? role : role?.name) ?? staff?.role)?.toLowerCase();
    if (roleName === "owner") return true;  // Owner bypasses all permissions
    return permissions.includes(permission);
  },

  canAny: (perms: string[]) => perms.some((p) => get().can(p)),
  canAll: (perms: string[]) => perms.every((p) => get().can(p)),

  isOwner: () => {
    const { role, staff } = get();
    const roleName = ((typeof role === "string" ? role : role?.name) ?? staff?.role)?.toLowerCase();
    return roleName === "owner";
  },

  isAdmin: () => {
    const { role, staff } = get();
    const roleName = ((typeof role === "string" ? role : role?.name) ?? staff?.role)?.toLowerCase();
    return roleName === "owner" || roleName === "admin";
  },
}));
```

Key patterns:
- **No persist middleware** -- hydration is manual from localStorage via `hydrate()`.
- **Owner bypasses all permissions** in `can()`.
- Store is accessed imperatively in services via `useAuthStore.getState().setAuth(staff)`.

---

## 5. TanStack Query Hooks (`lib/api/useQuery.tsx`)

Thin wrappers around TanStack Query with project defaults.

```ts
import { useMutation, useQuery, keepPreviousData } from "@tanstack/react-query";

export function useApiQuery<T>({
  key,
  queryFn,
  select,
  enabled = true,
  staleTime = 1000 * 60 * 5,  // 5 minutes
  keepData = false,
}: {
  key: any[];
  queryFn: (params?: any) => Promise<T>;
  select?: (data: T) => T;
  enabled?: boolean;
  staleTime?: number;
  keepData?: boolean;
}) {
  return useQuery<T>({
    queryKey: key,
    queryFn,
    select,
    enabled,
    staleTime,
    placeholderData: keepData ? keepPreviousData : undefined,
  });
}

export function useApiMutation<T, P = any>({
  mutationFn,
}: {
  mutationFn: (payload: P) => Promise<T>;
}) {
  return useMutation({ mutationFn });
}
```

### QueryClient Provider (`app/[locale]/providers.tsx`)

```ts
"use client";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,        // 1 minute
        gcTime: 5 * 60 * 1000,       // 5 minutes
        retry: 1,
        refetchOnWindowFocus: false,
      },
    },
  });
}

let browserQueryClient: QueryClient | undefined = undefined;

function getQueryClient() {
  if (typeof window === "undefined") return makeQueryClient();  // Server: always new
  if (!browserQueryClient) browserQueryClient = makeQueryClient();  // Browser: singleton
  return browserQueryClient;
}

export function Providers({ children }: { children: React.ReactNode }) {
  const queryClient = getQueryClient();
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      {process.env.NODE_ENV === "development" && <ReactQueryDevtools initialIsOpen={false} />}
    </QueryClientProvider>
  );
}
```

---

## 6. Custom Hooks (`hooks/`)

### useAuth (RBAC convenience hook)

```ts
"use client";
import { useAuthStore } from "@/lib/auth/auth-store";
import { PERMISSIONS } from "@/lib/constants/permissions";

export function useAuth() {
  const staff = useAuthStore((s) => s.staff);
  const can = useAuthStore((s) => s.can);
  const canAny = useAuthStore((s) => s.canAny);
  const isOwner = useAuthStore((s) => s.isOwner);
  const isAdmin = useAuthStore((s) => s.isAdmin);
  const isHydrated = useAuthStore((s) => s.isHydrated);

  const canEditAgent = useCallback((agentCreatedBy?: number) => {
    if (can(PERMISSIONS.AGENTS.EDIT_ANY)) return true;
    if (!can(PERMISSIONS.AGENTS.EDIT_OWN)) return false;
    if (agentCreatedBy === undefined || !staff?.id) return false;
    return agentCreatedBy === staff.id;
  }, [can, staff?.id]);

  const canViewAgentAnalytics = useCallback((agentCreatedBy?: number) => {
    if (can(PERMISSIONS.ANALYTICS.VIEW_ALL)) return true;
    if (!can(PERMISSIONS.ANALYTICS.VIEW_OWN)) return false;
    if (agentCreatedBy === undefined || !staff?.id) return false;
    return agentCreatedBy === staff.id;
  }, [can, staff?.id]);

  return { staff, can, canAny, isOwner, isAdmin, isHydrated, isAuthenticated: !!staff, canEditAgent, canViewAgentAnalytics };
}
```

### useApi (legacy fetch hook)

```ts
export function useApi<T>(apiCall: () => Promise<T>, dependencies: any[] = []) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<ApiError | null>(null);

  const execute = useCallback(async () => {
    try { setLoading(true); setError(null); const result = await apiCall(); setData(result); }
    catch (err) { setError(err instanceof ApiError ? err : new ApiError("An error occurred", 500)); }
    finally { setLoading(false); }
  }, dependencies);

  useEffect(() => { execute(); }, [execute]);
  return { data, loading, error, refetch: execute };
}
```

### useForbiddenHandler (global 403 toast)

```ts
"use client";
export function useForbiddenHandler() {
  useEffect(() => {
    const handler = (e: Event) => {
      const detail = (e as CustomEvent<{ message?: string }>).detail;
      toast.error(detail?.message || "You do not have permission to perform this action.", {
        description: "Contact your administrator to request access.",
      });
    };
    window.addEventListener("api:forbidden", handler);
    return () => window.removeEventListener("api:forbidden", handler);
  }, []);
}
```

### useFileUpload (drag-and-drop file handling)

Returns `[FileUploadState, FileUploadActions]` tuple. Supports single/multiple mode, file validation (size, type), drag events, preview URLs, and callbacks for `onFilesChange`/`onFilesAdded`.

### useConfig (cached config data)

Loads config from `ConfigService` with localStorage caching and freshness checks.

---

## 7. Routing Structure

```
app/
  [locale]/                       # next-intl dynamic locale (en, ar)
    layout.tsx                    # Root layout with Providers
    providers.tsx                 # QueryClientProvider
    dashboard/
      (guest)/                    # Unauthenticated routes
        login/v1/page.tsx
        register/v1/page.tsx
      (auth)/                     # Authenticated routes (middleware-protected)
        build/
          agents/page.tsx         # Agent list
          agents/[id]/page.tsx    # Agent builder (7 tabs)
          inbox/page.tsx          # Conversation inbox
          calls/page.tsx          # Calls list
          batch/page.tsx          # Batch campaigns
        settings/
          team/page.tsx           # Staff, roles, invitations, audit log
          payment/page.tsx        # Billing
          profile/page.tsx        # User profile
    invite/[token]/page.tsx       # Invitation acceptance
```

---

## 8. Component Patterns

- **shadcn/ui**: All primitives from `components/ui/` (Button, Dialog, Sheet, Tabs, DataTable, etc.)
- **Feature components**: Domain-specific in `app/[locale]/dashboard/(auth)/build/*/components/`
- **"use client"** directive: All interactive components (forms, modals, data grids)
- **Server components**: Page-level components for data fetching where possible
- **Toast notifications**: `sonner` via `toast.success()`, `toast.error()`
- **i18n**: `useTranslations("namespace")` from `next-intl` in every component
- **RTL support**: Full Arabic RTL via Tailwind CSS and next-intl locale detection
