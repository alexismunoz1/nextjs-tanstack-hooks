---
name: hook-development
description: >
  Enforces hook development conventions for this Next.js project: file structure,
  TanStack Query patterns, query key factory, typing, and anti-pattern prevention.
  Use when creating, modifying, or reviewing custom React hooks in src/hooks/,
  or when user mentions "new hook", "create a hook", "add useQuery hook",
  "fix hook pattern", "review my hook", "migrate hook", or "hook anti-patterns".
  Do NOT use for general React component development or non-hook utilities.
license: MIT
metadata:
  author: suig-team
  version: "3.0.0"
---

## Purpose

You enforce the hook development standards for this project. Every hook in `src/hooks/` MUST follow these rules. Apply them when creating new hooks, modifying existing ones, or reviewing hook code.

## When to Run

- Creating a new hook in `src/hooks/`
- Modifying an existing hook
- Reviewing hook code for quality
- Migrating legacy hooks to the correct pattern

## File Structure

Every hook MUST live in its own directory with this exact layout:

```
src/hooks/
└── use-hook-name/
    ├── index.ts                    # Barrel export (named, never export *)
    ├── use-hook-name.ts            # Implementation
    ├── use-hook-name.types.ts      # Types (when needed, never inline complex types)
    └── use-hook-name.test.ts       # Co-located test
```

### Rules

1. **Directory name** = kebab-case, matches the hook name: `use-favorite-toggle/`. _Why: consistent naming enables predictable imports and automated tooling._
2. **No standalone files** — every hook gets a directory, even simple ones. _Why: uniform structure makes it trivial to add types, tests, or co-located files later without restructuring._
3. **No `export *`** in barrel — use explicit named exports. _Why: explicit exports enable tree-shaking and make the public API visible at a glance._

```typescript
// index.ts — CORRECT
export { useFavoriteToggle } from './use-favorite-toggle';
export type {
  ToggleFavoriteRequest,
  ToggleFavoriteResponse,
  UseFavoriteToggleReturn,
} from './use-favorite-toggle.types';
```

```typescript
// index.ts — WRONG
export * from './use-favorite-toggle';
```

4. **Types file** — create `use-hook-name.types.ts` when you have interfaces for params, responses, or return types. Keep types inline only for trivial cases (1-2 simple types).
5. **`'use client'` directive** — REQUIRED at the top of every hook that uses React hooks (`useState`, `useEffect`, `useReducer`, `useRef`, `useQuery`, `useActionState`, etc.).

## Data Fetching — TanStack Query

**NEVER use manual `fetch()` + `useState` + `useEffect` for data fetching.** Always use TanStack Query. Prefer `queryOptions()` factories in `src/lib/queries/` for simple queries (reusable in SSR, client, and imperative contexts). Create custom hooks only when adding logic beyond the query itself (composition, derived state, side effects).

For full templates (query, mutation, infinite query, optimistic updates), loading state guide, conditional queries, polling, debounce patterns, and SSR hydration, see **[references/tanstack-query-patterns.md](references/tanstack-query-patterns.md)**.

## Query Key Factory — `queryKeys` (MANDATORY)

All query keys MUST come from the factory in `src/lib/query-keys.ts`. **NEVER hardcode query key arrays.** _Why: centralized keys prevent cache collisions and make invalidation patterns discoverable._

```typescript
// CORRECT
queryKey: queryKeys.favorites.status(itemId)

// WRONG — hardcoded key
queryKey: ['items', id, 'assignment']
```

### When You Need a New Key

1. Add it to `src/lib/query-keys.ts` in the appropriate section
2. Follow the existing pattern: `entity.action(params)`
3. Use `as const` on the return
4. Add corresponding `staleTimes` and `gcTimes` entries if needed

### Cache Invalidation Strategies

| Strategy | When | Example |
|----------|------|---------|
| `invalidateQueries` (broad key) | Refetch all related data | `queryKeys.favorites.lists()` — invalidates ALL favorite lists |
| `invalidateQueries` (narrow key) | Refetch specific item | `queryKeys.favorites.itemCount(itemId)` |
| `setQueryData` | You know the new value, update cache directly | `setQueryData(queryKeys.favorites.status(id), { isFavorited: true })` |
| Multi-key invalidation | Mutation affects multiple caches | Invalidate lists + count + itemCount in one `onSuccess` |
| `setQueryData` + `invalidateQueries` | Optimistic update + eventual consistency | Set data immediately, invalidate in `onSettled` to reconcile |

**WARNING:** Never invalidate queries inside `useEffect` based on `dataUpdatedAt` — this creates invalidation loops. Use `onSettled` callbacks instead.

## Stale & GC Times — Centralized Config (MANDATORY)

Never hardcode timing values. Import from `src/lib/query-keys.ts`:

```typescript
// CORRECT
import { staleTimes, gcTimes } from '@/lib/query-keys';

staleTime: staleTimes.favorites,
gcTime: gcTimes.favorites,

// WRONG — hardcoded
staleTime: 1000 * 60 * 5,
gcTime: 1000 * 60 * 10,
```

If a domain doesn't have stale/gc times defined yet, add them to the centralized config.

## React 19 & React Compiler

This project uses React 19 hooks (`useActionState`, `useOptimistic`, `use()`) and has React Compiler enabled. **Do NOT write** manual `useMemo` or `useCallback` in new hooks — the compiler handles memoization at build time. _Why: manual memos add noise and can conflict with compiler optimizations._

For `useActionState` and `useOptimistic` templates, `use()` rules, and compiler edge cases, see **[references/react19-and-compiler.md](references/react19-and-compiler.md)**.

## Hook Composition

Hooks can (and should) compose other hooks from `src/hooks/`:

```typescript
export function useEditProfileForm(): UseEditProfileFormReturn {
  const { user } = useAuth();
  const { profile, isLoading: isProfileLoading } = useUserProfile(user?.id);
  const { mutate, isPending } = useUpdateProfileMutation();

  // Derive state, combine loading flags, etc.
}
```

Rules:
- Call hooks in consistent order (React rules of hooks)
- Pass derived values between hooks
- Keep composed hooks under 300 lines; if too large, split the composition
- Each sub-hook should handle one concern (auth, data, mutation)

### State Machines with `useReducer`

For multi-step flows (transfers, wizards), use a typed reducer as a state machine. Define discriminated union types for state and actions, then a pure `flowReducer` function with `switch` on `action.type`. Typed transitions prevent invalid state combinations. The reducer is a pure function — test it directly without `renderHook`.

```typescript
type FlowState =
  | { step: 'idle' }
  | { step: 'confirm'; recipientId: string }
  | { step: 'submitting'; recipientId: string }
  | { step: 'success'; txHash: string }
  | { step: 'error'; error: Error };

type FlowAction =
  | { type: 'START'; recipientId: string }
  | { type: 'SUBMIT' }
  | { type: 'SUCCESS'; txHash: string }
  | { type: 'FAIL'; error: Error }
  | { type: 'RESET' };

function flowReducer(state: FlowState, action: FlowAction): FlowState { /* switch on action.type */ }
```

## Types Conventions

```typescript
// use-hook-name.types.ts

export interface UseHookNameParams {
  id: string;
  enabled?: boolean;
}

export interface HookNameResponse {
  success: boolean;
  data: SomeData;
  error?: string;
}

export interface UseHookNameReturn {
  data: SomeData | null;
  isLoading: boolean;
  error: string | null;
}
```

### Discriminated Unions for Complex Return Types

When a hook has distinct states, use discriminated unions instead of nullable fields. TypeScript narrows correctly at call site:

```typescript
type UseItemResult =
  | { status: 'loading' }
  | { status: 'success'; data: Item }
  | { status: 'error'; error: Error };
```

### Tuple Returns with `as const`

For hooks returning `[value, setter]` pairs, use `as const` to preserve tuple types instead of `(T | Function)[]`:

```typescript
return [value, toggle] as const; // infers [boolean, () => void]
```

### Rules

- **No `any`** — strict TypeScript, always
- **Explicit return type** on the hook function signature
- **Separate types file** for hooks with 3+ interfaces
- **Import with `type`** keyword: `import type { ... } from './use-hook-name.types'`
- **Don't define unused types** — if an interface/type isn't imported anywhere, delete it

## Hook Size & Responsibility

| Guideline | Limit |
|-----------|-------|
| Max lines per hook | ~300 lines |
| Single responsibility | One concern per hook |
| Max `useEffect` per hook | 2 (prefer 0-1) |

If a hook exceeds 300 lines or handles multiple concerns, split it:
- Data fetching -> `queryOptions()` factory or separate query hook
- Filter/form state -> separate state hook
- UI behavior -> separate UI hook
- Compose them in a parent hook or component if needed

## Testing

Use `renderHook` from `@testing-library/react` with a `QueryClientProvider` wrapper for query/mutation hooks. Test reducers as pure functions without `renderHook`.

For full testing templates and patterns, see **[references/testing-hooks.md](references/testing-hooks.md)**.

## Shared Utilities — Don't Duplicate

Before writing mapping functions, type converters, or helpers inside a hook:

1. Check `src/lib/utils/` for existing utilities
2. Check other hooks for the same function (e.g., `itemToGridItem` exists in 6 hooks — it should be in `src/lib/utils/`)
3. If the function is used by 2+ hooks, extract it to `src/lib/utils/`
4. Shared interfaces used across hooks go in `src/types/`, not duplicated in each hook's types file

## Anti-Patterns & Verification

Before finishing any hook, check for common anti-patterns (setState during render, useEffect for fetching, manual memoization, hardcoded query keys, string-based error matching) and run through the verification checklist covering structure, directives, types, and build.

For the full anti-patterns table and verification checklist, see **[references/anti-patterns-checklist.md](references/anti-patterns-checklist.md)**.

## Examples

### Example 1: "Create a hook for fetching user profile"

Claude creates:
- `src/hooks/use-user-profile/` directory with `index.ts`, `use-user-profile.ts`, and `use-user-profile.types.ts`
- Since this is a simple query, defines a `userProfileOptions(userId)` factory in `src/lib/queries/` using `queryOptions()`
- Adds query key `queryKeys.user.profile(userId)` to `src/lib/query-keys.ts` with corresponding `staleTimes` and `gcTimes` entries
- The custom hook in `use-user-profile.ts` wraps the factory only if it adds derived state or composition logic; otherwise, consumers use `useSuspenseQuery(userProfileOptions(id))` directly

### Example 2: "Add a mutation hook for updating settings"

Claude creates:
- `src/hooks/use-update-settings/` directory with `index.ts`, `use-update-settings.ts`, and `use-update-settings.types.ts`
- Uses the Mutation Hook Template from `references/tanstack-query-patterns.md`
- Invalidates related queries in `onSuccess` (e.g., `queryKeys.settings.detail(userId)` and `queryKeys.settings.lists()`)
- Types file defines `UpdateSettingsRequest`, `UpdateSettingsResponse`, and `UseUpdateSettingsReturn`
- Barrel export uses explicit named exports

### Example 3: "This hook has useState + useEffect for fetching, fix it"

Claude:
- Identifies the `useState` + `useEffect` + `fetch()` anti-pattern
- Refactors to use `useQuery` or creates a `queryOptions()` factory in `src/lib/queries/` if the hook is simple enough
- Removes manual loading/error state management (replaced by TanStack Query's `isLoading`, `error`)
- Adds query key to `queryKeys` factory (never hardcoded)
- Imports centralized `staleTimes`/`gcTimes`
- Verifies against the anti-patterns checklist in `references/anti-patterns-checklist.md`
