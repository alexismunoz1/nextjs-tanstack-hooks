# TanStack Query Patterns

## Data Fetching Architecture

### Query Options Factory — Preferred Abstraction

For **new queries**, prefer `queryOptions()` / `infiniteQueryOptions()` factories over wrapping `useQuery` in a custom hook. Factories are reusable across Server Components (prefetch), Client Components (hooks), and imperative calls (hover prefetch).

Define factories in `src/lib/queries/` alongside the query key factory:

```typescript
// src/lib/queries/items.ts
import { queryOptions, infiniteQueryOptions } from '@tanstack/react-query';
import { queryKeys, staleTimes, gcTimes } from '@/lib/query-keys';

async function fetchItemById(id: string) {
  const res = await fetch(`/api/items/${id}`);
  const data = await res.json();
  if (!data.success) throw new Error(data.error || 'Failed to fetch');
  return data;
}

export function itemDetailOptions(id: string) {
  return queryOptions({
    queryKey: queryKeys.items.byId(id),
    queryFn: () => fetchItemById(id),
    staleTime: staleTimes.entityDetails,
    gcTime: gcTimes.entityDetails,
  });
}
```

Usage across contexts:

```typescript
// Server Component — prefetch for SSR hydration
const queryClient = getQueryClient();
void queryClient.prefetchQuery(itemDetailOptions(id));

// Client Component — useSuspenseQuery (data guaranteed non-null)
const { data } = useSuspenseQuery(itemDetailOptions(id));

// Imperative — hover prefetch
queryClient.prefetchQuery(itemDetailOptions(id));

// Type-safe cache read (no cast needed)
const cached = queryClient.getQueryData(itemDetailOptions(id).queryKey);
```

**When to still create a custom hook:** When the hook adds logic beyond the query itself — composition of multiple queries, derived state, side effects, or complex enabled logic.

### SSR Hydration Pattern

Prefetch in Server Components, hydrate in Client Components:

```typescript
// app/[locale]/items/[id]/page.tsx (Server Component)
import { dehydrate, HydrationBoundary } from '@tanstack/react-query';
import { getQueryClient } from '@/lib/query/get-query-client';
import { itemDetailOptions } from '@/lib/queries/items';

export default async function ItemPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const queryClient = getQueryClient();

  void queryClient.prefetchQuery(itemDetailOptions(id));

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <ItemDetail id={id} />
    </HydrationBoundary>
  );
}
```

```typescript
// lib/query/get-query-client.ts — singleton per server request
import { QueryClient } from '@tanstack/react-query';
import { cache } from 'react';

export const getQueryClient = cache(
  () => new QueryClient({
    defaultOptions: { queries: { staleTime: 60 * 1000 } },
  })
);
```

Use `cache()` from React to scope one `QueryClient` per server request. Never use a module-level singleton on the server.

### `useSuspenseQuery` vs `useQuery`

| Aspect | `useQuery` | `useSuspenseQuery` |
|--------|-----------|-------------------|
| `data` type | `T \| undefined` | `T` (guaranteed) |
| Loading/error handling | Manual in component | Handled by `<Suspense>` / `<ErrorBoundary>` |
| Best for | Conditional queries, complex enabled logic | Pages/sections wrapped in Suspense boundaries |

**Waterfall warning:** Multiple `useSuspenseQuery` calls in the same component execute serially. Use `useSuspenseQueries` to parallelize:

```typescript
// WRONG — waterfall, second query waits for first
const { data: item } = useSuspenseQuery(itemDetailOptions(id));
const { data: owner } = useSuspenseQuery(ownerOptions(id));

// CORRECT — parallel execution
const [{ data: item }, { data: owner }] = useSuspenseQueries({
  queries: [itemDetailOptions(id), ownerOptions(id)],
});
```

## Data Fetching — TanStack Query (MANDATORY)

**NEVER use manual `fetch()` + `useState` + `useEffect` for data fetching.** Always use TanStack Query.

### Query Hook Template

Use when the hook adds logic beyond a simple query (composition, derived state, complex conditions):

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';
import { queryKeys, staleTimes, gcTimes } from '@/lib/query-keys';
import type {
  FetchResponse,
  UseHookParams,
  UseHookReturn,
} from './use-hook-name.types';

async function fetchData(id: string): Promise<FetchResponse> {
  const response = await fetch(`/api/resource/${id}`);
  const data: FetchResponse = await response.json();

  if (!data.success) {
    throw new Error(data.error || 'Failed to fetch');
  }

  return data;
}

export function useHookName({
  id,
  enabled = true,
}: UseHookParams): UseHookReturn {
  const { data, isLoading, error } = useQuery({
    queryKey: queryKeys.resource.byId(id),
    queryFn: () => fetchData(id),
    enabled,
    staleTime: staleTimes.resource,
    gcTime: gcTimes.resource,
  });

  return {
    data: data ?? null,
    isLoading,
    error: error instanceof Error ? error.message : null,
  };
}
```

### Mutation Hook Template

```typescript
'use client';

import { useMutation, useQueryClient } from '@tanstack/react-query';
import { queryKeys } from '@/lib/query-keys';
import type {
  MutationRequest,
  MutationResponse,
  UseMutationReturn,
} from './use-hook-name.types';

async function mutateApi(request: MutationRequest): Promise<MutationResponse> {
  const response = await fetch(`/api/resource/${request.id}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(request),
  });

  const data: MutationResponse = await response.json();

  if (!data.success) {
    throw new Error(data.error || 'Operation failed');
  }

  return data;
}

export function useHookName(): UseMutationReturn {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: (request: MutationRequest) => mutateApi(request),
    onSuccess: (_data, variables) => {
      queryClient.invalidateQueries({
        queryKey: queryKeys.resource.byId(variables.id),
      });
    },
  });

  return {
    mutate: async (request: MutationRequest) => {
      await mutation.mutateAsync(request);
    },
    isLoading: mutation.isPending,
    error: mutation.error instanceof Error ? mutation.error.message : null,
  };
}
```

### Infinite Query Hook Template

Use `useInfiniteQuery` for paginated lists with cursor-based pagination or infinite scroll.

```typescript
'use client';

import { useInfiniteQuery, keepPreviousData } from '@tanstack/react-query';
import { queryKeys, staleTimes } from '@/lib/query-keys';
import type { PageResponse, UseInfiniteHookReturn } from './use-hook-name.types';

const LIMIT = 20;

async function fetchPage(cursor?: string): Promise<PageResponse> {
  const params = new URLSearchParams({ limit: String(LIMIT) });
  if (cursor) params.set('cursor', cursor);

  const response = await fetch(`/api/resource?${params}`);
  const data: PageResponse = await response.json();

  if (!data.success) throw new Error(data.error || 'Failed to fetch');
  return data;
}

export function useHookName(): UseInfiniteHookReturn {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage, isLoading } =
    useInfiniteQuery({
      queryKey: queryKeys.resource.lists(),
      queryFn: ({ pageParam }) => fetchPage(pageParam),
      initialPageParam: undefined as string | undefined,
      getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
      placeholderData: keepPreviousData,
      staleTime: staleTimes.resource,
    });

  // React Compiler handles memoization — no manual useMemo needed
  const items = (() => {
    if (!data?.pages) return [];
    const seen = new Set<string>();
    return data.pages.flatMap((page) =>
      page.items.filter((item) => {
        if (seen.has(item.id)) return false;
        seen.add(item.id);
        return true;
      })
    );
  })();

  return { items, fetchNextPage, hasNextPage, isFetchingNextPage, isLoading };
}
```

Key points for infinite queries:
- `initialPageParam` defines the type and initial value for pagination
- `getNextPageParam` returns `undefined` to signal no more pages
- `placeholderData: keepPreviousData` keeps previous data visible during refetch for smooth UX
- Flatten pages with `flatMap` and dedup with `Set` to prevent duplicate items across pages
- Use `refetchInterval` if the list needs real-time updates (see Polling section)

### Optimistic Updates Template

Use optimistic updates when the UI should respond instantly (e.g., toggling favorites, sending messages).

```typescript
'use client';

import { useMutation, useQueryClient } from '@tanstack/react-query';
import { queryKeys } from '@/lib/query-keys';

export function useOptimisticMutation() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: sendMessageApi,
    onMutate: async (newMessage) => {
      const queryKey = queryKeys.messages.detail(newMessage.conversationId);
      await queryClient.cancelQueries({ queryKey });
      const previousData = queryClient.getQueryData(queryKey);

      queryClient.setQueryData(queryKey, (old) => ({
        ...old,
        messages: [
          ...(old?.messages ?? []),
          { ...newMessage, id: `temp-${Date.now()}` },
        ],
      }));

      return { previousData, queryKey };
    },
    onError: (_err, _vars, context) => {
      if (context?.previousData) {
        queryClient.setQueryData(context.queryKey, context.previousData);
      }
    },
    onSettled: (_data, _err, variables) => {
      queryClient.invalidateQueries({
        queryKey: queryKeys.messages.detail(variables.conversationId),
      });
    },
  });
}
```

Key points for optimistic updates:
- `onMutate`: cancel outgoing queries, snapshot previous data, apply optimistic update, return context
- `onError`: rollback to snapshot if the mutation fails
- `onSettled`: always invalidate to reconcile with server state (runs on both success and error)
- Use `temp-${Date.now()}` or similar for temporary IDs until the server responds
- Consider `useOptimistic` from React 19 as an alternative for Server Action forms (see React 19 section)

### Key Rules for Data Fetching Hooks

| Rule | Detail |
|------|--------|
| **No manual fetch + useState** | Always `useQuery` for reads, `useMutation` for writes |
| **No `useEffect` for fetching** | TanStack Query handles lifecycle. `useEffect` IS valid for side effects (IntersectionObserver, DOM, event listeners) |
| **No `AbortController` manually** | TanStack Query cancels via `signal` in `queryFn` automatically |
| **No ref-based fetch patterns** | Never assign fetch functions to `useRef` to bypass deps |
| **Error messages in English** | Never hardcode Spanish/Portuguese error strings. Use error codes from the API |
| **Extract fetch function** | Define `async function fetchX()` OUTSIDE the hook, pass it as `queryFn` |
| **Prefer `queryOptions()` factory** | For simple queries, define a factory in `src/lib/queries/` instead of a custom hook |

### Loading States

Using the wrong loading state causes bad UX. Understand the distinction:

| State | Meaning | UI Treatment |
|-------|---------|-------------|
| `isLoading` / `isPending` (query, no data) | First load, no cached data | Show skeleton |
| `isFetching` | Background refetch, cached data still shown | Subtle indicator (spinner in corner, NOT skeleton) |
| `isPending` (mutation) | Mutation in progress | Disable button, show spinner |
| `isPlaceholderData` | Showing stale data via `keepPreviousData` | Optionally dim content |

### Conditional Queries with `enabled`

Gate queries to prevent fetching with incomplete data:

| Pattern | Example |
|---------|---------|
| Wait for auth | `enabled: !!userId` |
| Wait for required params | `enabled: !!itemId && !!slug` |
| Wait for user interaction | `enabled: debouncedQuery.length >= 2` |
| Combine conditions | `enabled: !!userId && !!itemId && isReady` |

Common mistake: forgetting `enabled` causes a fetch with `undefined` params, leading to 400 errors or wrong cache entries.

### Polling and Refetch Intervals

Use `refetchInterval` for data that changes frequently (messages, typing indicators, live scores):

```typescript
useQuery({
  queryKey: queryKeys.messages.lists(),
  queryFn: fetchConversations,
  refetchInterval: 30 * 1000,
  refetchOnWindowFocus: true,
});
```

| Setting | When to use |
|---------|-------------|
| `refetchInterval` | Real-time data: messages, notifications, live status |
| `refetchOnWindowFocus: true` | Data that goes stale when user switches tabs |
| `refetchOnMount: true` | Data that must be fresh on every navigation |

Keep polling intervals as named constants. Reference `staleTimes` for consistency across the codebase.

**WARNING:** Never invalidate queries inside `useEffect` based on `dataUpdatedAt` — this creates invalidation loops. Use `onSettled` callbacks instead.

### Debounce + Query Composition

Combine `useDebounce` with `useQuery` for search inputs and typeahead:

```typescript
'use client';

import { useDebounce } from '@/hooks/use-debounce';
import { useQuery } from '@tanstack/react-query';
import { queryKeys } from '@/lib/query-keys';

export function useUserSearch(query: string) {
  const debouncedQuery = useDebounce(query, 300);

  const { data, isLoading } = useQuery({
    queryKey: queryKeys.user.search(debouncedQuery),
    queryFn: () => searchUsers(debouncedQuery),
    enabled: debouncedQuery.length >= 2,
  });

  return { results: data?.users ?? [], isLoading };
}
```

Key points:
- Debounce the input, not the query — `useQuery` deduplicates by key automatically
- Use `enabled` to prevent queries for too-short input
- The debounce delay (300ms) prevents excessive API calls while typing

### Derived State and Transforms

Derive transformed data from query results. React Compiler handles memoization automatically — no manual `useMemo` needed:

```typescript
const gridItems = data?.items.map(itemToGridItem) ?? [];
```

Rule: Transform functions used by 2+ hooks MUST be extracted to `src/lib/utils/`. Do not duplicate mappers like `itemToGridItem` across hooks.
