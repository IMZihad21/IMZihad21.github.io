# Master React API Management with TanStack React Query: Best Practices & Examples

- Canonical URL: https://imzihad21.github.io/articles/a/master-react-api-management-with-tanstack-react-query-best-practices-examples-1139/
- Source URL: https://dev.to/imzihad21/master-react-api-management-with-tanstack-react-query-best-practices-examples-1139
- Web View: https://imzihad21.github.io/articles/a/master-react-api-management-with-tanstack-react-query-best-practices-examples-1139/
- Published: 2024-11-27T05:43:02.000Z
- Modified: 2024-11-27T05:43:02.000Z
- Reading time: 5 minutes
- Tags: react, reactquery, axios, api

## Master React API management with TanStack Query: best practices and examples

Managing asynchronous server state manually in React applications introduces complex state variables, race conditions, and duplicated fetch logic. Without a unified server-state management layer, components duplicate loading flags, handle cache invalidation inconsistently, and expose stale views to end users.

TanStack Query provides a structured model for data fetching, caching, mutations, and background synchronization. Declarative query keys, deterministic cache boundaries, and centralized client configuration eliminate manual synchronization boilerplate while maintaining predictable client-server state consistency.

### The problem and production context

In applications relying solely on primitive React hooks such as `useEffect` and `useState` for remote API synchronization, network interactions lack centralized lifecycle coordination.

- **Failure scenario**: Multiple mounted components trigger identical GET requests concurrently on mount, causing request waterfalls and duplicate network round-trips. When a mutation executes, sibling components holding cached responses fail to refresh unless explicit callback chains or global state actions are wired manually.
- **Why default approaches fall short**: React state management primitives (`useState`, `useReducer`, Context API) are designed for client-local UI state rather than asynchronous server state. They do not provide built-in stale-time policies, automatic garbage collection, window refocus polling, deduplication, or exponential backoff retries.
- **Production impact**: Client applications suffer from memory leaks caused by uncoordinated subscribers, inconsistent UI renders across route transitions, unnecessary network consumption on mobile clients, and complex error-boundary fallbacks scattered throughout the component hierarchy.

This architecture addresses those limitations by:
- Reducing repetitive boilerplate for loading states, error boundaries, and data fetching.
- Providing predictable caching, garbage collection, and stale data invalidation.
- Improving user experience through automatic retries and background refetching.
- Decoupling server state from local UI component state.

### Mental model and core concepts

#### 1. Centralized QueryClient instance

Instantiate a single, stable `QueryClient` instance configured with baseline caching and retry policies. The client operates as the central cache bus and lifecycle manager for all queries and mutations across the application tree.

```typescript
import { QueryClient } from "@tanstack/react-query";

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60_000,
      gcTime: 5 * 60_000,
      retry: 2,
      refetchOnWindowFocus: false,
    },
  },
});
```

#### 2. Application root provider integration

Wrap the application root with `QueryClientProvider` to make the client available throughout the component tree via React Context.

```typescript
import ReactDOM from "react-dom/client";
import { QueryClientProvider } from "@tanstack/react-query";
import { queryClient } from "./lib/queryClient";
import App from "./App";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <QueryClientProvider client={queryClient}>
    <App />
  </QueryClientProvider>
);
```

#### 3. Query functions and deterministic query keys

Define dedicated query functions with predictable query keys to fetch and return remote data. Query keys must be serializable arrays that uniquely identify resource scopes.

```typescript
import axios from "axios";

const apiClient = axios.create({ baseURL: import.meta.env.VITE_API_BASE_URL });

export async function getItems() {
  const response = await apiClient.get("/items");
  return response.data;
}
```

#### 4. useQuery with version 5 object signature

In TanStack Query version 5, `useQuery` exclusively accepts an options object containing `queryKey` and `queryFn`, replacing legacy positional arguments.

```typescript
import { useQuery } from "@tanstack/react-query";
import { getItems } from "./api";

export function ItemsList() {
  const { data = [], isPending, isError } = useQuery({
    queryKey: ["items"],
    queryFn: getItems,
  });

  if (isPending) return <div>Loading...</div>;
  if (isError) return <div>Error loading items.</div>;

  return (
    <ul>
      {data.map((item: { id: number; name: string }) => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

#### 5. useMutation with options object and cache invalidation

`useMutation` also accepts an options object. Use the `onSuccess` callback to invalidate stale queries after a mutation completes, which prompts active observers to refetch latest state automatically.

```typescript
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { apiClient } from "./apiClient";

export function useCreateItem() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (payload: { name: string }) => {
      const response = await apiClient.post("/items", payload);
      return response.data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["items"] });
    },
  });
}
```

#### 6. Shared query options factory

Use the `queryOptions` helper to define strongly typed, reusable query definitions across components, prefetching logic, and route loaders.

```typescript
import { queryOptions } from "@tanstack/react-query";
import { getItems } from "./api";

export const itemsQueryOptions = queryOptions({
  queryKey: ["items"],
  queryFn: getItems,
  staleTime: 60_000,
});
```

### Production implementation

The following implementation provides a reusable, type-safe mutation hook abstraction supporting dynamic HTTP methods, custom headers, and multipart payloads across application services.

```typescript
import { useMutation, UseMutationResult } from "@tanstack/react-query";
import { AxiosError } from "axios";
import { apiClient } from "../lib/apiClient";

type HttpMethod = "POST" | "PUT" | "DELETE";

interface MutationHookOptions<TVariables> {
  endpoint: string;
  method?: HttpMethod;
  isMultiPart?: boolean;
  toBody?: (variables: TVariables) => unknown;
}

export function useApiMutation<TData = unknown, TVariables = unknown>({
  endpoint,
  method = "POST",
  isMultiPart = false,
  toBody,
}: MutationHookOptions<TVariables>): UseMutationResult<TData, AxiosError, TVariables> {
  return useMutation<TData, AxiosError, TVariables>({
    mutationFn: async (variables) => {
      const response = await apiClient.request<TData>({
        url: endpoint,
        method,
        data: toBody ? toBody(variables) : variables,
        headers: {
          "Content-Type": isMultiPart ? "multipart/form-data" : "application/json",
        },
      });

      return response.data;
    },
  });
}
```

This pattern keeps mutation code consistent and avoids copy-pasting request handling across multiple components.

### Architectural trade-offs and edge cases

* **Latency versus consistency**: Setting an aggressive `staleTime` (such as 0 ms) guarantees fresh data at the cost of higher network traffic and increased server load. Setting a longer `staleTime` (such as 60,000 ms) yields instant UI renders from cache but delays propagating mutations made outside the current browser session.
* **Failure recovery**: Queries automatically retry failed requests based on configured retry counts and exponential backoff policies. Background refetching recovers state after network dropouts, but mutations do not retry automatically by default to prevent duplicate side effects on non-idempotent endpoints.
* **Scale limitations**: In-memory query caching depends on available browser heap space. Applications managing large collections must configure explicit `gcTime` limits and paginated query boundaries to avoid retaining unbounded data structures in inactive cache entries.

### Common anti-patterns and gotchas

* **Passing positional arguments to hooks**: In version 5, passing positional arguments `useQuery(key, fn, options)` causes compile or runtime errors. Pass a single options object containing `queryKey` and `queryFn`.
* **Using removed callbacks inside useQuery**: Developers often attempt to supply `onSuccess` or `onError` callbacks inside `useQuery` options. Version 5 removed these callbacks to enforce pure render behavior. Handle side effects in `useEffect`, custom query wrappers, or global callbacks on `QueryCache`.
* **Constructing non-deterministic query keys**: Using unstable object references or varying key array orders produces unexpected cache misses. Query keys must be serializable and structurally stable across calls.
* **Forgetting cache invalidation after mutations**: Omitting `queryClient.invalidateQueries` after a mutation leaves the client UI out of sync with updated server data until `staleTime` expires or manual reload occurs.
* **Creating monolithic global query functions**: Writing oversized multi-purpose query functions couples unrelated API endpoints and prevents fine-grained cache invalidation. Endpoints must remain modular and scoped.

### Implementation checklist

1. Instantiate a single `QueryClient` with explicit `staleTime`, `gcTime`, and `retry` defaults.
2. Wrap the application root with `QueryClientProvider`.
3. Define deterministic query keys and isolated query fetching functions.
4. Replace legacy positional hook calls with version 5 single-options object signatures.
5. Invalidate relevant query keys inside mutation `onSuccess` callbacks.
6. Implement optimistic updates for instant UI feedback on mutation requests.
7. Configure paginated and infinite queries using cursor-based keys.
8. Install and configure TanStack Query Devtools in your development environment.
9. Establish centralized error normalization and toast notifications for API failures.