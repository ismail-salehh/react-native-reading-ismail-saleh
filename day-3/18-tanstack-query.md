# TanStack Query: queries, mutations, caching, useInfiniteQuery

TanStack Query (formerly React Query) manages **server state**: data that lives on the server and that the app keeps a copy of. Day 2 covered why this is different from UI state ([Server state vs UI state](../day-2/10-server-state-vs-ui-state.md)). TanStack Query does the hard parts of that copy for you: caching, deduplicating, refetching, retrying, and tracking loading and error states.

It doesn't make requests itself. You give it a function (using the Axios `api` from [Axios](08-axios.md)), and it decides **when** to call it and what to do with the result.

```bash
npm install @tanstack/react-query
```

The examples use v5.

## Setup

```tsx
// App.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: { staleTime: 30_000 },
  },
});

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <RootNavigator />
    </QueryClientProvider>
  );
}
```

## Queries: reading data

```tsx
import { useQuery } from '@tanstack/react-query';

function ProductScreen({ id }: { id: number }) {
  const { data, isPending, isError, error, refetch, isFetching } = useQuery({
    queryKey: ['product', id],
    queryFn: async ({ signal }) => {
      const { data } = await api.get<Product>(`/products/${id}`, { signal });
      return data;
    },
  });

  if (isPending) return <Loading />;
  if (isError) return <ErrorView error={error} onRetry={refetch} />;
  return <ProductDetails product={data} />;
}
```

- **`queryKey`** identifies the data in the cache. Same key = same cached data, shared by every component that uses it. Include **every variable** the request depends on (`id`, filters, page). When the key changes, a new query runs.
- **`queryFn`** fetches the data. It must throw on failure (Axios already does). Pass `signal` to Axios: TanStack Query aborts the request if it is no longer needed ([Request cancellation](12-request-cancellation.md)).
- `data` is typed from the `queryFn` return value.

### The states

| Flag | True when |
|---|---|
| `isPending` | No data yet (first load) |
| `isError` | The last fetch failed (after retries) |
| `isSuccess` | `data` is available |
| `isFetching` | A request is in flight, **including background refetches** |
| `isLoading` | `isPending && isFetching`: first load in progress |
| `isRefetching` | `isFetching && !isPending`: refetching data you already have |

`isPending` decides "show a skeleton". `isFetching` decides "show a small spinner over existing data". Mixing them up makes the whole screen flash on every background refresh.

## Caching: staleTime and gcTime

Every query in the cache is either **fresh** or **stale**, and either **active** (some component uses it) or **inactive**.

| Option | Default | Meaning |
|---|---|---|
| `staleTime` | `0` | How long data counts as fresh. Fresh data is served from cache **without** a request |
| `gcTime` | 5 min | How long **inactive** data stays in memory before it is deleted |

What happens when a screen opens:

```
key not in cache          -> fetch, show loading
key in cache, fresh       -> show cached data, no request
key in cache, stale       -> show cached data immediately AND refetch in background
                             (stale-while-revalidate)
```

With `staleTime: 0`, cached data is always shown instantly but always refetched. Raise `staleTime` for data that rarely changes (categories, settings) to save requests.

Stale queries also refetch automatically when:

- a component using the key mounts,
- the app comes back to the foreground (needs `focusManager`, below),
- the network reconnects (needs `onlineManager`, see [NetInfo](17-netinfo.md)).

### Retries

Failed queries are retried **3 times** with exponential backoff before `isError` becomes true. Retrying a `404` is pointless, so a common setting is:

```ts
retry: (failureCount, error) => {
  const status = axios.isAxiosError(error) ? error.response?.status : undefined;
  if (status && status >= 400 && status < 500) return false;
  return failureCount < 3;
},
```

Don't also retry in an Axios interceptor (see [Retry failed requests](16-retry-failed-requests.md)).

## Mutations: changing data

`useMutation` is for `POST`, `PUT`, `PATCH`, `DELETE`. It doesn't run on mount, only when you call `mutate`:

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

function AddToCartButton({ productId }: { productId: number }) {
  const queryClient = useQueryClient();

  const addToCart = useMutation({
    mutationFn: (quantity: number) => api.post('/cart/items', { productId, quantity }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['cart'] });
    },
    onError: (error) => showToast(errorMessage(toApiError(error))),
  });

  return (
    <Button
      title={addToCart.isPending ? 'Adding…' : 'Add to cart'}
      disabled={addToCart.isPending}
      onPress={() => addToCart.mutate(1)}
    />
  );
}
```

- **`invalidateQueries`** marks every query whose key **starts with** `['cart']` as stale and refetches the active ones. This is how the cart screen shows the new item: you don't update it by hand.
- `isPending` disables the button so a double tap doesn't send two requests.
- Mutations are **not retried** by default, which is right for non-idempotent requests.

For instant feedback you can update the cache before the server answers (an **optimistic update**) with `queryClient.setQueryData`, and roll back in `onError`. Start with `invalidateQueries`; add optimistic updates only where the delay is noticeable.

## useInfiniteQuery: pagination and infinite scroll

Day 1 built [pagination](../day-1/14-pagination.md) and [infinite scrolling](../day-1/15-infinite-scrolling.md) by hand with `useState`. `useInfiniteQuery` replaces that bookkeeping:

```tsx
import { useInfiniteQuery } from '@tanstack/react-query';

type Page = { items: Product[]; nextPage: number | null };

function ProductList() {
  const {
    data, isPending, isError, refetch, isRefetching,
    fetchNextPage, hasNextPage, isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['products'],
    queryFn: async ({ pageParam, signal }) => {
      const { data } = await api.get<Page>('/products', { params: { page: pageParam }, signal });
      return data;
    },
    initialPageParam: 1,
    getNextPageParam: (lastPage) => lastPage.nextPage, // null/undefined = no more pages
  });

  const items = data?.pages.flatMap((p) => p.items) ?? [];

  if (isPending) return <Loading />;
  if (isError) return <ErrorView onRetry={refetch} />;

  return (
    <FlatList
      data={items}
      keyExtractor={(item) => String(item.id)}
      renderItem={({ item }) => <ProductRow product={item} />}
      onEndReached={() => {
        if (hasNextPage && !isFetchingNextPage) fetchNextPage();
      }}
      onEndReachedThreshold={0.5}
      ListFooterComponent={isFetchingNextPage ? <ActivityIndicator /> : null}
      refreshing={isRefetching}
      onRefresh={refetch}
    />
  );
}
```

- `data.pages` is an array of every page loaded so far. Flatten it for the `FlatList`.
- `getNextPageParam` reads the cursor or next page number from the last response. Returning `null` or `undefined` sets `hasNextPage` to `false`.
- The `hasNextPage && !isFetchingNextPage` guard matters: `onEndReached` can fire several times in a row.
- `refetch` reloads every loaded page, which gives [pull to refresh](../day-1/13-pull-to-refresh.md) for free.

## React Native specifics

On the web, TanStack Query listens for window focus and online events. React Native has neither, so you connect them once at app start.

**App focus** (refetch stale data when the app returns from the background):

```ts
import { AppState } from 'react-native';
import { focusManager } from '@tanstack/react-query';

AppState.addEventListener('change', (status) => {
  focusManager.setFocused(status === 'active');
});
```

**Online state**: connect `onlineManager` to NetInfo, shown in [Network state with NetInfo](17-netinfo.md).

**Screen focus**: a screen in a stack stays mounted when you navigate away, so returning to it doesn't count as a "mount". If you want a refetch when it regains focus, call `refetch` in React Navigation's `useFocusEffect`.

**Logout**: call `queryClient.clear()` so the next user doesn't see the previous user's cached data.

## Rule of thumb

`useQuery` for reading, `useMutation` + `invalidateQueries` for writing, `useInfiniteQuery` for lists that load more. Put every input in the `queryKey`, pass `signal` to Axios, set `staleTime` per kind of data, and wire up `focusManager` and `onlineManager` once.
