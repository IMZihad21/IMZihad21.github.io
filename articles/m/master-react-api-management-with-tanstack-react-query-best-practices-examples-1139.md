# Master React API Management with TanStack React Query: Best Practices & Examples

- Canonical URL: https://imzihad21.github.io/articles/a/master-react-api-management-with-tanstack-react-query-best-practices-examples-1139/
- Source URL: https://dev.to/imzihad21/master-react-api-management-with-tanstack-react-query-best-practices-examples-1139
- Web View: https://imzihad21.github.io/articles/a/master-react-api-management-with-tanstack-react-query-best-practices-examples-1139/
- Published: 2024-11-27T05:43:02.000Z
- Modified: 2024-11-27T05:43:02.000Z
- Reading time: 3 minutes
- Tags: react, reactquery, axios, api

## Master React API Management with TanStack Query: Best Practices and Examples

Managing server state manually in React becomes noisy very fast. TanStack Query gives a clean model for fetching, caching, mutations, and background synchronization.

This guide is aligned with the latest TanStack Query React docs (v5 style APIs).

### Why It Matters

- Reduces boilerplate for async API flows.
- Gives reliable caching and invalidation patterns.
- Improves UX with retries, stale handling, and background refetch.
- Keeps server state separate from local UI state.

### Core Concepts

#### 1. QueryClient Setup

Create one stable `QueryClient` with sane defaults.

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

#### 2. Provider Integration

Wrap the app with `QueryClientProvider` once.

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

#### 3. Query Functions and Keys

Use stable keys and explicit fetchers.

```typescript
import axios from "axios";

const apiClient = axios.create({ baseURL: import.meta.env.VITE_API_BASE_URL });

export async function getItems() {
  const response = await apiClient.get("/items");
  return response.data;
}
```

#### 4. useQuery with v5 Object Signature

In v5, `useQuery` uses the object-style signature.

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

#### 5. useMutation with v5 Options Object

`useMutation` should also use the options object signature.

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

#### 6. Shared Query Options

Use `queryOptions` helper for reusable typed query config.

```typescript
import { queryOptions } from "@tanstack/react-query";
import { getItems } from "./api";

export const itemsQueryOptions = queryOptions({
  queryKey: ["items"],
  queryFn: getItems,
  staleTime: 60_000,
});
```

### Practical Example

Reusable mutation hook with proper method handling:

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

This keeps mutation code consistent and avoids copy-paste request logic across components.

### Common Mistakes

- Using old positional hook signatures instead of v5 object signature.
- Putting `onSuccess`/`onError` callbacks on `useQuery` options like old versions.
- Using unstable query keys and then debugging cache ghosts.
- Forgetting invalidation after successful mutation.
- Over-centralizing one giant default query function for unrelated endpoints.

### Quick Recap

- TanStack Query v5 prefers one object-style signature.
- `useQuery` and `useMutation` should use options objects.
- Query keys must stay stable and predictable.
- Mutations should invalidate affected queries.
- `queryOptions` helps create reusable typed query configs.

### Next Steps

1. Add optimistic updates for create/update flows.
2. Add paginated queries with cursor-based keys.
3. Add Devtools in development environment.
4. Add error normalization and toast strategy for all API failures.