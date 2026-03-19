# Testing Hooks

Use `renderHook` from `@testing-library/react` (NOT the deprecated `@testing-library/react-hooks` package).

## Testing TanStack Query hooks

```typescript
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

function createTestWrapper() {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false, staleTime: Infinity, gcTime: Infinity },
    },
  });

  return function Wrapper({ children }: { children: React.ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    );
  };
}

describe('useItemDetail', () => {
  it('returns item data on success', async () => {
    vi.spyOn(global, 'fetch').mockResolvedValueOnce(
      new Response(JSON.stringify({ success: true, id: '1', name: 'Jersey' }))
    );

    const { result } = renderHook(() => useItemDetail('1'), {
      wrapper: createTestWrapper(),
    });

    await waitFor(() => expect(result.current.isLoading).toBe(false));
    expect(result.current.data?.name).toBe('Jersey');
  });
});
```

## Testing reducers directly

Reducers are pure functions — test them without `renderHook`:

```typescript
import { flowReducer } from './use-transfer-flow';

it('transitions from idle to confirm on START', () => {
  const next = flowReducer(
    { step: 'idle' },
    { type: 'START', recipientId: 'user-123' }
  );
  expect(next).toEqual({ step: 'confirm', recipientId: 'user-123' });
});
```
