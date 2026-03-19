# React 19 Hooks & React Compiler

## React 19 Hooks

### `useActionState` — Form Mutations with Server Actions

Use `useActionState` for forms that call Server Actions. It replaces manual `useState` + submit handler patterns:

```typescript
'use client';

import { useActionState } from 'react';
import { updateProfileAction } from '@/lib/actions/profile';

type ActionState = { error?: string; success?: boolean };

export function useProfileForm() {
  const [state, formAction, isPending] = useActionState<ActionState, FormData>(
    updateProfileAction,
    { success: false }
  );

  return { state, formAction, isPending };
}
```

The third return value `isPending` replaces most uses of `useFormStatus` in the same component. Use `useFormStatus` only in child components that need the parent form's submission state.

### `useOptimistic` — Native Optimistic UI

Alternative to TanStack Query's `onMutate` pattern for Server Action forms:

```typescript
'use client';

import { useOptimistic, useTransition } from 'react';

export function useLikeButton(currentLikes: number) {
  const [optimisticLikes, addOptimisticLike] = useOptimistic(
    currentLikes,
    (current, delta: number) => current + delta
  );
  const [, startTransition] = useTransition();

  const handleLike = () => {
    startTransition(async () => {
      addOptimisticLike(1);
      await toggleLikeAction(itemId);
    });
  };

  return { likes: optimisticLikes, handleLike };
}
```

**When to use each:**
| Pattern | Use when |
|---------|----------|
| TanStack Query optimistic (`onMutate`) | Client-side API calls, complex cache invalidation, rollback needed |
| `useOptimistic` | Server Action forms, simple optimistic state, no cache layer involved |

### `use()` — Conditional Context and Promise Reading

`use()` can be called conditionally (unlike `useContext`). Use it to read Context or resolve Promises passed from Server Components:

```typescript
'use client';

import { use } from 'react';
import { ThemeContext } from '@/contexts/theme';

function ConditionalTheme({ showTheme }: { showTheme: boolean }) {
  if (!showTheme) return null;
  const theme = use(ThemeContext);
  return <div style={{ color: theme.primary }}>Themed</div>;
}
```

**Critical rule:** Never create Promises inside a Client Component render — they recreate on every render, causing infinite suspend loops. Always pass Promises from Server Components as props.

## React Compiler — No Manual Memoization

This project has React Compiler enabled. It automatically inserts memoization equivalent to `React.memo`, `useMemo`, and `useCallback` at build time.

### What this means for hooks

**Do NOT write** manual `useMemo` or `useCallback` in new hooks. The compiler handles it:

```typescript
// CORRECT — React Compiler memoizes automatically
function useFilteredItems(items: Item[], query: string) {
  const filtered = items.filter(item =>
    item.name.toLowerCase().includes(query.toLowerCase())
  );
  return filtered;
}

// UNNECESSARY — manual memoization the compiler already handles
function useFilteredItems(items: Item[], query: string) {
  return useMemo(
    () => items.filter(item =>
      item.name.toLowerCase().includes(query.toLowerCase())
    ),
    [items, query]
  );
}
```

### When the compiler CANNOT help

- Mutating objects/arrays in place (the compiler analyzes data flow statically)
- Reading from refs imperatively during render
- Side effects during render (non-idempotent render functions)

In these cases, the compiler skips memoization and may emit a lint diagnostic. Fix the code pattern instead of adding manual memoization.

### Stable references at module scope

Even with the compiler, prefer module-level constants for values that never change:

```typescript
// CORRECT — stable reference, no allocation per render
const DEFAULT_FILTERS = ['worn', 'signed'] as const;

function useItemFilters() {
  const [filters, setFilters] = useState(DEFAULT_FILTERS);
  // ...
}
```
