# Anti-Patterns & Verification Checklist

## Anti-Patterns Checklist

Before finishing any hook, verify NONE of these are present:

| Anti-Pattern | Why it's wrong | Correct alternative |
|-------------|----------------|---------------------|
| `setState()` during render | Breaks React Compiler, causes re-render loops | Derive from existing state, or sync via `useEffect` |
| `useCallback(...)()` (IIFE) | Not memoization, runs every render | Derive inline (compiler memoizes) or use `useMemo` if compiler is disabled |
| `setTimeout` to wait for state | Fragile timing hack, races | Use `useEffect` with proper deps, or TanStack Query `onSettled` |
| Error detection via string matching | Breaks with i18n, fragile | Use error codes from the API |
| `console.error` for user-facing errors | Invisible to users | Return error state from the hook |
| `new Map()` / `new Set()` in render | New reference every render | Derive inline (compiler memoizes) or wrap in `useMemo` |
| Unused `useQueryClient()` | Dead code | Remove if not used |
| Unnecessary fetch options | `Cache-Control` header on requests, `cache: 'no-store'` on mutations, `credentials: 'include'` for same-origin | Remove — they have no effect |
| `eslint-disable react-hooks/exhaustive-deps` | Hiding a real dependency bug | Fix the dependency array properly |
| `useEffect` for data fetching | TanStack Query handles fetch lifecycle | Use `useQuery` / `useMutation` / `queryOptions()` |
| Invalidate in `useEffect` on `dataUpdatedAt` | Creates invalidation loops with polling | Use `onSettled` callbacks |
| Manual `useMemo` / `useCallback` | React Compiler handles memoization automatically | Write plain expressions; compiler optimizes at build time |
| `fetch()` + `useState` + `useEffect` | Loses cache, dedup, retry, loading states | Use TanStack Query or `queryOptions()` factory |

Note: `useEffect` is NOT valid for data fetching, but IS valid for side effects like IntersectionObserver setup, DOM manipulation, or event listeners.

## Verification Checklist

After creating or modifying a hook, verify:

- [ ] Directory structure: `use-hook-name/index.ts` + `use-hook-name.ts` + optional `.types.ts` + optional `.test.ts`
- [ ] `'use client'` directive present (if using React hooks)
- [ ] Barrel export uses explicit named exports (no `export *`)
- [ ] Data fetching uses `useQuery`, `useInfiniteQuery`, `useMutation`, or `queryOptions()` factory (never manual fetch + useState)
- [ ] Query keys come from `queryKeys` factory (never hardcoded arrays)
- [ ] Stale/GC times come from centralized `staleTimes`/`gcTimes`
- [ ] No anti-patterns from the checklist above
- [ ] No manual `useMemo`/`useCallback` (React Compiler handles it)
- [ ] Types properly defined (no `any`, explicit return type)
- [ ] Hook is under ~300 lines with single responsibility
- [ ] No duplicated utilities — check `src/lib/utils/` first
- [ ] Error messages in English (or use error codes)
- [ ] No unused imports, variables, or types
- [ ] Loading states used correctly (`isLoading` for skeletons, `isFetching` for background)
- [ ] `enabled` gates queries with incomplete params
- [ ] Optimistic updates include rollback in `onError`
- [ ] Consider `queryOptions()` factory for simple queries (reusable in SSR prefetch)
- [ ] `pnpm build` passes
- [ ] `pnpm lint` passes
