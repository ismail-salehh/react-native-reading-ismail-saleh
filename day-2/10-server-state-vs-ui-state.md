# Server state vs UI state

Introduced here, applied on Day 3.

## The two kinds of state

### UI state

State that exists **only in the app** and that the app fully owns.

Examples: is a modal open, which tab is selected, text in a form, selected filter, theme, language.

- Synchronous and instant.
- Always up to date, because nobody else can change it.
- Tools: `useState`, `useReducer`, Context, Redux Toolkit / Zustand.

### Server state

Data that **lives on a server** (database, API). The app only holds a **copy** of it.

Examples: list of products, user profile, orders, messages, notifications.

- Asynchronous: it must be fetched, so there is loading and error.
- Can become **stale**: someone else (another device, another user) may change it any time.
- Shared: many screens may need the same data.
- Needs caching, refetching, retries, pagination, invalidation after changes.

## The common mistake

Storing API responses in Redux/Context/`useState` and hand-writing everything around it:

```jsx
const [products, setProducts] = useState([]);
const [loading, setLoading] = useState(false);
const [error, setError] = useState(null);

useEffect(() => {
  setLoading(true);
  fetch('/products')
    .then((r) => r.json())
    .then(setProducts)
    .catch(setError)
    .finally(() => setLoading(false));
}, []);
```

Problems with this:

- Repeated in every screen.
- No caching: leaving and returning to the screen refetches (or shows stale data).
- No retry, no refetch on app focus or reconnect, no deduping of identical requests.
- Race conditions when params change quickly.
- Easy to forget to clear or update it after a mutation.

## The better approach

Treat server state as its own category and use a library built for it: **TanStack Query** (React Query) or RTK Query.

```jsx
import { useQuery } from '@tanstack/react-query';

function ProductsScreen() {
  const { data, isLoading, error, refetch } = useQuery({
    queryKey: ['products'],
    queryFn: () => fetch('/products').then((r) => r.json()),
  });

  if (isLoading) return <ActivityIndicator />;
  if (error) return <Text>Something went wrong</Text>;
  return <FlatList data={data} renderItem={...} />;
}
```

The library gives you caching, background refetching, retries, dedupe, pagination, and mutations with cache invalidation, with no store code written by you.

## Splitting responsibilities

| | UI state | Server state |
|---|---|---|
| Owner | The app | The server |
| Can be stale | No | Yes |
| Async | No | Yes |
| Tool | useState, Context, Redux/Zustand | TanStack Query / RTK Query |
| Example | `isModalOpen`, `selectedTab` | `products`, `user profile` |

A typical screen uses both: server state for the list, UI state for the search text and selected filter.

```jsx
const [filter, setFilter] = useState('all');              // UI state
const { data } = useQuery({                               // server state
  queryKey: ['products', filter],
  queryFn: () => fetchProducts(filter),
});
```

Notice the UI state (`filter`) becomes part of the query key, so the data refetches and caches per filter automatically.

## React Native specifics

- Mobile connections are unreliable: caching and retries matter a lot. Users expect to see the last known data instantly, then a refresh.
- Refetch when the app returns to the foreground (`AppState`) and when the network reconnects (`NetInfo`). TanStack Query supports both via `focusManager` and `onlineManager`.
- Screens stay mounted in navigation stacks, so use the cache instead of passing fetched objects through navigation params.
- Pull to refresh maps directly to `refetch` and `isRefetching`.

## Rule of thumb

If it came from an API, it is server state. Don't copy it into Redux or Context.
If the user created it in the UI and the server doesn't know about it, it is UI state.

Day 3 applies this by fetching real data with TanStack Query.
