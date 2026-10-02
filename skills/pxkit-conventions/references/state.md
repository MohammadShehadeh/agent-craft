# Hooks & State

## Hooks

- A custom hook exists when two components share the logic, or when the logic would push a component past 60 lines. Otherwise it stays inside the component.
- A hook returns an object, never a tuple.
- An effect cleans up what it starts.

## Local async state: one status union

A form submit or one-shot action holds its lifecycle in one `status` value. Flags are derived from it, never stored beside it.

```ts
// Bad — eight combinations, most of them impossible
const [isLoading, setIsLoading] = useState(false);
const [isError, setIsError] = useState(false);

// Good — one union
type Status = 'idle' | 'loading' | 'success' | 'error';
const [status, setStatus] = useState<Status>('idle');

// When data or an error key belongs to a state, put it on that state
type ReportState =
  | { status: 'idle' | 'loading' }
  | { status: 'success'; report: Report }
  | { status: 'error'; errorKey: ReportErrorKey };
```

- A hook exposes `status` plus only the derived flags its callers read. A single in-flight flag (`isPending` from a transition) needs no `status`.
- Failures carry an `errorKey`, never a message string (`pxkit-service` skill).

## Server state: the query cache

Data the backend owns lives in a query cache (TanStack Query, or the repo's equivalent), never in `useState` + `useEffect` + `fetch`.

- The `useQuery` call sits in the component that uses it. A `use*` hook appears when a second component needs the same query.
- The query function unwraps `Result` by throwing `QueryError`, so the cache drives loading and error while the `errorKey` survives:

```ts
// lib/query-error.ts — written once per repo
import type { ErrorKey } from '@/lib/error-copy';

export class QueryError extends Error {
  constructor(readonly errorKey: ErrorKey) {
    super(errorKey);
  }
}

// in the component
const invoices = useQuery({
  queryKey: ['invoices', filters],
  queryFn: async () => {
    const result = await fetchInvoices(filters);
    if (!result.ok) throw new QueryError(result.errorKey);
    return result.data;
  },
});
```

- The component renders the failure from `errorCopy[invoices.error.errorKey]` after checking `invoices.error instanceof QueryError`.
- Writes use `useMutation`, then `invalidateQueries({ queryKey })`. Never hand-sync a local copy.
- Live updates come from the cache: `refetchInterval` (as a function that returns `false` once the data has settled), a socket handler calling `queryClient.setQueryData`, or `staleTime` + `refetchOnWindowFocus`. Never a hand-rolled `setInterval` + `fetch`.
- Never copy a server prop into `useState` (`useState(serverValue)`); the copy goes stale on the next refetch.

## Deriving list views

- **Search, sort, and paginate on the server** when the API supports it: build the params with a pure `buildInvoiceQuery(filters)` and let the API page. Client-side only for lists already fully in memory.
- Client-side steps are pure functions, composed as a derivation over the source list. The source stays in one place; only the controls (`query`, `sort`, `page`) are state:

```ts
const filtered = sortInvoices(searchInvoices(invoices, query), sort);
const visible = filtered.slice((page - 1) * PAGE_SIZE, page * PAGE_SIZE);
const total = filtered.length; // derived, never a stored counter
```

  Wrap the first two lines in `useMemo` only when the repo doesn't use React Compiler and the list is large.
- `sortInvoices` copies before sorting (`[...rows].sort(...)`); nothing mutates the source.
- Debounce the input, not the derivation: `useDeferredValue(query)`.
- Filter and sort on domain values (`Date`, `number`), then format. Never format a value and parse the string back.

## Formatting for display

- Display strings are made where they render, by pure `format*` functions in one shared `lib/format.ts`, never in a service and never as `toLocaleString()` inlined in JSX.
- `Intl` formatters are built once at module scope:

```ts
// lib/format.ts
const money = new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' });
const mediumDate = new Intl.DateTimeFormat('en-US', { dateStyle: 'medium' });

export const formatMoney = (amount: number) => money.format(amount);
export const formatDate = (value: Date) => mediumDate.format(value);
```

## Optimistic UI

**Use it** for writes that almost always succeed and that the client can predict: toggles, counters, reordering, inline renames. **Skip it** when being wrong is worse than being slow: destructive or irreversible actions, money, permissions, outcomes the server decides. Those render `isPending` and wait.

The overlay sits in the component that owns the control:

```tsx
const [isFavorite, setOptimisticFavorite] = useOptimistic(thread.isFavorite);
const [isPending, startTransition] = useTransition();

const handleToggle = (next: boolean) =>
  startTransition(async () => {
    setOptimisticFavorite(next);
    const result = await toggleFavorite(thread.id, next); // the action revalidates thread.isFavorite
    if (!result.ok) toast.error(errorCopy[result.errorKey]); // the transition ends without a commit, so React reverts
  });
```

- The server value has one owner (a prop the action revalidates). Never negate by hand to revert.
- Everything, including any debounce delay, is awaited inside the transition; outside it the old value flashes back.
- A rollback is announced with a toast; a silent snap-back looks like a bug.
- A pending write is not lost: flush it on unmount, and pass `keepalive: true` when it can fire during tab close.
- When a second component needs debounced, coalesced updates, move this into a shared `useOptimisticValue` hook that also drops superseded calls.

## Context

Use context for state many descendants of a subtree read (session, theme, an active checkout), not to skip one level of props. Each context is written out in its own file, with a guard in its hook:

```tsx
const CheckoutContext = createContext<CheckoutContextValue | null>(null);

export const CheckoutProvider = ({ value, children }: CheckoutProviderProps) => (
  <CheckoutContext.Provider value={value}>{children}</CheckoutContext.Provider>
);

export const useCheckout = () => {
  const value = useContext(CheckoutContext);
  if (!value) throw new Error('useCheckout must be used inside <CheckoutProvider>');
  return value;
};
```

Never put server data a Server Component could fetch into context.
