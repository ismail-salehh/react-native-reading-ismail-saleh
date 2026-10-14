# Request cancellation (AbortController)

Cancelling a request tells the app to stop waiting for it and throw the response away. You need it when the response is no longer useful.

## When you need it

- **The screen unmounts.** The user navigates away before the data arrives. Without cancellation, the response still comes back and the code tries to update state for a screen that is gone.
- **Search-as-you-type.** The user types `s`, `sh`, `sho`. Three requests are in flight. If the `s` response arrives last, it overwrites the `sho` results. This is a **race condition**, and it shows wrong data.
- **The user presses "Cancel"** on a long upload or download.

## AbortController

`AbortController` is a standard Web API that React Native provides. Axios (and `fetch`) accept its `signal`:

```ts
const controller = new AbortController();

api.get('/products', { signal: controller.signal });

controller.abort(); // the request is cancelled and its promise rejects
```

- One controller can be aborted only once. Create a **new controller for each request** (or each effect run).
- One signal can be passed to several requests to cancel them together.

The rejected error is a `CanceledError` with `code === 'ERR_CANCELED'`. Check for it with `axios.isCancel(error)` and **ignore it**, because the app asked for it.

## In a useEffect

The cleanup function of `useEffect` runs when the component unmounts **and** before the effect runs again. That is exactly when the previous request should be cancelled:

```tsx
function SearchResults({ query }: { query: string }) {
  const [results, setResults] = useState<Product[]>([]);
  const [error, setError] = useState<ApiError | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    api
      .get<Product[]>('/products', { params: { q: query }, signal: controller.signal })
      .then((res) => setResults(res.data))
      .catch((err) => {
        if (axios.isCancel(err)) return; // old query or unmounted
        setError(toApiError(err));
      });

    return () => controller.abort();
  }, [query]);

  // ...render
}
```

What happens while typing:

```
query = "s"    -> request A starts
query = "sh"   -> cleanup aborts A, request B starts
query = "sho"  -> cleanup aborts B, request C starts
                  only C can call setResults
```

Combine this with a debounce (wait ~300 ms after the last keystroke) so you don't send a request on every key at all.

## Passing the signal through API functions

If screens call functions like `getProducts`, let those functions accept a signal:

```ts
export async function searchProducts(q: string, signal?: AbortSignal) {
  const { data } = await api.get<Product[]>('/products', { params: { q }, signal });
  return data;
}
```

[TanStack Query](18-tanstack-query.md) gives every `queryFn` a `signal` and aborts it automatically when the query is no longer needed, so you rarely write the `useEffect` above by hand once you use it.

## What cancellation doesn't do

- It **doesn't undo** anything on the server. If the server already got a `POST`, it will process it. Cancellation only means the app stops listening.
- Avoid cancelling writes (`POST`, `PUT`, `DELETE`) the user expects to happen. Cancel reads freely.

## CancelToken is deprecated

Older Axios code uses `axios.CancelToken.source()`. It is deprecated. Use `AbortController` for new code.

## Rule of thumb

Every request started in an effect gets an `AbortController`, and the effect cleanup aborts it. In the `catch`, ignore `axios.isCancel(error)`.
