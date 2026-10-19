# State Management

> The #1 architectural mistake in React apps is treating all state as the same.
> **Classify state first → pick the right tool → put it there.** Everything else follows.

## The four kinds of state

| Kind | Lives where it is used | Example | Tool |
|---|---|---|---|
| **Server state** | Owned by the server; cached in the client | User list, order details, feature flags | **React Query / SWR / RTK Query** |
| **URL state** | Lives in the URL (shareable, back-button-safe) | Filters, pagination, selected tab | React Router params / `useSearchParams` |
| **Local UI state** | One component (or close siblings) | Modal open, input value, hover | `useState` / `useReducer` |
| **Shared client state** | A few components across the tree | Theme, auth user, sidebar collapsed | **Zustand / Jotai** (or Context, carefully) |

**Rule:** classify every piece of state before you pick a tool. 90% of "where should this live?" debates evaporate once you've named the kind.

## Decision flow

```
Is it from the server?
   → YES → React Query. Done.
   → NO → Should it survive refresh / be shareable via URL?
            → YES → URL state (search params / route).
            → NO → Does more than ~2 components need it?
                     → NO  → useState / useReducer in the owning component.
                     → YES → Zustand (or Context if it changes rarely).
```

## Rules

- **Server state does NOT belong in Redux / Zustand / Context.** React Query already handles caching, loading, errors, refetching, deduplication. Rebuilding it in your global store is pure waste — and usually buggy.
- **Lift state up only when it's shared.** Starting with global state and "scoping down later" never happens.
- **Derive, don't duplicate.** If `displayName` can be computed from `firstName + lastName`, don't store it — compute at render.
- **Normalize relational data by ID** (`entities: { [id]: Entity }`). Prevents stale denormalized copies.
- **Context is for dependency injection, not state.** Fine for `theme`, `currentUser` (rarely changes). Terrible for anything that changes often — every consumer re-renders.
- **Forms live in form libraries** (React Hook Form). Not global state. Not controlled `useState` per field.
- **Boolean flags don't belong globally.** `isLoginModalOpen` is local to the page that owns the modal.
- **One source of truth per fact.** Don't mirror the URL into state; read from the URL directly.
- **No `useState(prop)`.** Deriving local state from a prop almost always means the component needs rethinking.

## Server state — the one tool to standardize on

```ts
// Query — reads
const { data, isLoading, error } = useQuery({
  queryKey: ['invoice', id],
  queryFn: () => api.getInvoice(id),
  staleTime: 60_000,
});

// Mutation — writes, with cache invalidation
const approve = useMutation({
  mutationFn: (id: string) => api.approveInvoice(id),
  onSuccess: (_, id) => {
    queryClient.invalidateQueries({ queryKey: ['invoice', id] });
    queryClient.invalidateQueries({ queryKey: ['invoices'] });
  },
});
```

Standardize on **TanStack Query** (or RTK Query / SWR). See [data-fetching](../data-fetching/SKILL.md) for the full patterns.

## Client state — Zustand pattern

Small, flat, selector-driven. No boilerplate, no re-renders you didn't ask for.

```ts
import { create } from 'zustand';

type AuthState = {
  user: User | null;
  setUser: (u: User | null) => void;
};

export const useAuth = create<AuthState>((set) => ({
  user: null,
  setUser: (user) => set({ user }),
}));

// In a component — subscribes only to .user:
const user = useAuth((s) => s.user);
```

## Red flags

- `useEffect` → `fetch` → `setState` for server data. **Use React Query.**
- Server data mirrored into Redux / Zustand "for easy access."
- Context value is an object rebuilt every render (every consumer re-renders).
- A `useState` whose initial value is a prop, re-synced via `useEffect`.
- 15 pieces of local UI state (`isOpen`, `isHovered`, `isLoading`, …) when `useReducer` would collapse them.
- Filter state stored in component state and manually synced to the URL — URL should be the source of truth.
- Global store with 40+ slices — most of them should be local.
- A single "god reducer" / "god store" that every component imports.
