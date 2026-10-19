# Data Fetching

> **Default: TanStack Query (React Query).** Never fetch in `useEffect` directly.
> A server-state library gives you caching, dedup, retries, loading, error, refetch, invalidation — for free. Rebuilding any of this by hand is a bug factory.

Companion to [state-management](../state-management/SKILL.md) — this file is the "how to use the server-state tool" rulebook.

## The default stack

| Need | Pick |
|---|---|
| Fetching + caching | **TanStack Query** (or SWR, RTK Query) |
| HTTP client | `fetch` wrapper, or `ky` / `axios` for interceptors |
| Request validation | **Zod** on every response (don't trust the wire) |
| Types | Generated from OpenAPI, or hand-written against the spec |

## Query keys — the index of your cache

Keys are the API of your cache. Design them as a hierarchy, not as free-form strings.

```ts
// Shape convention:  [entity, scope, params?]
['invoices']                            // all invoice queries
['invoices', 'list', { status: 'OPEN' }]
['invoices', 'detail', id]
['invoices', 'detail', id, 'items']
```

Benefits:
- `invalidateQueries(['invoices'])` → nukes everything invoice-related.
- `invalidateQueries(['invoices', 'detail', id])` → nukes just this one.
- Readable in DevTools.

## Rules

- **One query per resource, one hook per query.** Wrap in `useInvoice(id)`, don't call `useQuery` inline all over the app.
- **Every query sets `staleTime`.** `0` (default) means "refetch on every mount" — rarely what you want. Pick a value matched to the data's volatility.
- **Every mutation invalidates something.** Otherwise the UI shows yesterday's data.
- **Prefer `invalidateQueries` over `setQueryData`.** Invalidation is correct by default; manual cache writes get stale.
- **Optimistic updates for perceived speed, with rollback.** The user sees it as "instant"; on error, revert and surface a toast.
- **Validate every response with Zod.** The spec says one thing, the server does another — catch it at the edge, not deep in a component.
- **Loading and error are first-class UI states**, not afterthoughts. Design all three (success / loading / error) for every query.
- **Suspense boundaries for loading, Error boundaries for errors.** Co-located skeletons and friendly error UI; no `if (isLoading) return <Spinner/>` repeated in 50 places.
- **Pagination:** cursor-based (`useInfiniteQuery`) for user-facing feeds; offset for admin tables.
- **Prefetch on intent.** `queryClient.prefetchQuery` on link hover / list hover → next-page click feels instant.
- **Cancel on unmount is automatic** — don't `AbortController` by hand.
- **Dedup is automatic** — two components asking for the same key share one request.

## Mutation + invalidation — the canonical pattern

```ts
const queryClient = useQueryClient();

const approve = useMutation({
  mutationFn: (id: string) => api.approveInvoice(id),

  // Optimistic update
  onMutate: async (id) => {
    await queryClient.cancelQueries({ queryKey: ['invoices', 'detail', id] });
    const prev = queryClient.getQueryData(['invoices', 'detail', id]);
    queryClient.setQueryData(['invoices', 'detail', id], (old: Invoice) =>
      ({ ...old, status: 'APPROVED' })
    );
    return { prev };
  },

  // Rollback on error
  onError: (_err, id, ctx) => {
    if (ctx?.prev) queryClient.setQueryData(['invoices', 'detail', id], ctx.prev);
    toast.error('Approval failed');
  },

  // Always re-sync with the server
  onSettled: (_, __, id) => {
    queryClient.invalidateQueries({ queryKey: ['invoices', 'detail', id] });
    queryClient.invalidateQueries({ queryKey: ['invoices', 'list'] });
  },
});
```

## `staleTime` — rules of thumb

| Data | `staleTime` |
|---|---|
| User profile, feature flags | 5 min |
| Reference / lookup data | 1 hour |
| Hot business data (list of orders) | 30 s – 2 min |
| Real-time feel needed | 0 + refetch interval or websocket |

## Error UX — four classes, four responses

| Error class | UI response |
|---|---|
| Transient (5xx, network blip) | Auto-retry (React Query default: 3× with backoff) + inline retry button |
| Permanent client (4xx) | Clear in-situ message, no retry (`enabled: false` after first try) |
| Auth (401 / 403) | Redirect / sign-in prompt — handled globally, not per query |
| Validation (Zod fails) | Treat as 5xx: alert + Sentry — the API lied |

## Red flags

- `useEffect(() => fetch(url).then(setData), [])` — classic footgun.
- Loading / error booleans managed with `useState`.
- Mutations with no cache invalidation ("why isn't the UI updating?").
- `queryKey: 'invoice'` (string, not array) — breaks hierarchical invalidation.
- Fetching the same resource twice from two components because the key was stringified differently.
- Catching an error in the component and returning `null` — bugs disappear with the UI.
- `staleTime: 0` everywhere → constant refetching → battery + bandwidth cost.
- Hand-written race-condition guards (`if (requestId !== latestRef.current)`) — the library handles this.
