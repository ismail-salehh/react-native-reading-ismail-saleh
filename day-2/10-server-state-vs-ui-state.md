# Server state vs UI state: two different kinds of data

Introduced here, applied on Day 3.

The difference isn't which library you use. It's **where the source of truth lives**:

- **UI state**: the source of truth is a value in **this process's memory**, on the Hermes heap (on a Fiber or in a store). Nobody else can change it. Reading it is synchronous, and it is always correct.
- **Server state**: the source of truth is a **database on another machine**. The app only holds a **copy**, and that copy starts going stale the moment it arrives. Anyone (another user, another device, a cron job) can change the original without the app knowing.

So server state is really a **cache**, with the usual cache problems: deduplication, staleness, invalidation, garbage collection, and retry on failure. `useState` and Redux model none of these.

## 1. The path a request takes in React Native

`fetch` in React Native is not a browser API. It is a JS polyfill over `XMLHttpRequest`, and RN implements `XMLHttpRequest` on top of a **native Networking module**:

```
JS thread (Hermes)                          native                          network
------------------                          ------                          -------
fetch(url)
 -> XMLHttpRequest polyfill
 -> Networking.sendRequest(...)   ------>   iOS:     NSURLSession (RCTNetworking)
                                            Android: OkHttp                --> HTTP/TLS
                                            (runs on native threads)
                                  <------   events: response headers, data chunks, complete
 -> response.json()
      JSON.parse(text)                      <- runs on the JS thread
 -> Promise resolves (microtask)
 -> setState / cache update -> React render
```

What follows from this:

- **The network wait doesn't block JS.** Sockets, TLS and HTTP are handled on native threads. The JS thread is free while the request is in flight.
- **Parsing does block JS.** `JSON.parse` of a 2 MB response runs on the JS thread in Hermes. During that time, touches, JS animations and renders all wait. Big payloads should be paginated or trimmed on the server.
- **Results arrive as events at unpredictable times.** Two requests sent in order A, B can resolve in order B, A.

## 2. Why the hand-written `useEffect` fetch breaks

The usual pattern is `useState` for data, loading and error, plus a `fetch` in `useEffect`. Here is what goes wrong at the engine level:

- **Race conditions.** Each effect run creates a new closure that will call `setProducts` when *its* request resolves. Change the filter quickly from `a` to `b`, and if `a`'s response arrives last, the screen shows `a`'s data under the `b` filter.
- **No shared cache.** The data lives in **that component's hook list**. A second screen that needs the same list starts its own request and gets a separate copy, so the two can disagree.
- **Lifetime tied to the Fiber.** Pop the screen and the fiber is deleted along with the data. Navigate back and you get a spinner again.
- **No idea of "stale".** Once loaded, the data is treated as true forever. Nothing refetches it when the app comes back from the background or the network returns.
- **No dedupe or retry.** Three components mounting together make three identical requests, and a flaky mobile connection fails once and stays failed.

## 3. How TanStack Query models a cache

TanStack Query is a cache engine that lives **outside React**, much like a Redux store. React subscribes to it.

```
QueryClient
  └─ QueryCache: Map<queryHash, Query>
        └─ Query  (one per unique key)
             state: { data, error, status, fetchStatus, dataUpdatedAt, ... }
             promise / retryer      (the in-flight request, if any)
             observers: QueryObserver[]
             gcTimeout              (removal timer)

useQuery(...)  ->  one QueryObserver per component  ->  useSyncExternalStore
```

### Keys and hashing

`queryKey: ['products', { filter: 'all' }]` is turned into a string by **`hashKey`**, which is `JSON.stringify` with object keys **sorted**. So `{a, b}` and `{b, a}` produce the same hash. The hash is the cache's address. Any component asking for the same key gets the **same `Query` object**. That makes the key the identity of the data, and UI state such as a filter belongs in it.

### Deduplication

`query.fetch()` checks whether a promise is already in flight. If one is, it returns **that same promise** and doesn't start another request. Ten components mounting together produce one network request.

### Freshness: `staleTime` and `gcTime`

Each `Query` moves through these states:

```
fetch succeeds -> FRESH --(staleTime passes, default 0)--> STALE
STALE + trigger (new observer mounts / app focus / reconnect / invalidate) -> background refetch
last observer unsubscribes -> INACTIVE -> start gcTimeout (gcTime, default 5 min) -> removed from cache
new observer before the timer fires -> timer cleared, cached data returned instantly
```

- With the default `staleTime: 0`, data is stale as soon as it arrives. It is still **shown** right away, then refetched in the background on the next trigger. This is **stale-while-revalidate**.
- `status` (`pending` / `error` / `success`) describes **whether there is data**. `fetchStatus` (`fetching` / `paused` / `idle`) describes **whether a request is running right now**. They are separate, which is how "show cached data and refetch in the background" works.

### Retries

A failed `queryFn` goes through a **retryer**. By default it makes 3 retries with exponential backoff: `min(1000 * 2^attempt, 30000)` ms. If the device is offline, the retryer **pauses** (`fetchStatus: 'paused'`) instead of failing, and continues when connectivity returns.

### Structural sharing

When a refetch returns, the new JSON is merged with the old data through **`replaceEqualDeep`**. Every subtree that is deeply equal to the old one keeps the **old reference**. If one product out of 50 changed, only that product object and the array are new. `React.memo` rows for the other 49 see identical props and skip rendering.

### Tracked properties

The result object from `useQuery` defines its fields as **getters** and records which ones your component actually reads. The observer only notifies your component when **those** fields change. A component that reads only `data` doesn't re-render when `isFetching` flips.

## 4. "Focus" and "online" mean different things on mobile

TanStack Query refetches on **window focus** and **reconnect**. Its default detectors use browser APIs: `window` `visibilitychange` and `online`/`offline` events. React Native has no DOM `window` events, so out of the box **these triggers never fire on a phone.**

You connect them to the native signals:

- **Focus**: RN's `AppState` module reports `active` / `background` / `inactive`, based on the iOS `UIApplication` lifecycle and Android `Activity` lifecycle. Feed it to `focusManager.setFocused(...)`.
- **Online**: `@react-native-community/netinfo` reads `NWPathMonitor` on iOS and `ConnectivityManager` on Android. Feed it to `onlineManager.setOnline(...)`.

After that, the cache behaves like a mobile user expects. Coming back to the app shows the last data instantly and refreshes quietly. Losing signal pauses requests instead of showing errors.

## 5. Writes: mutations and invalidation

The client can't edit the source of truth directly. It can only **ask the server** and then **resync its copy**:

1. `useMutation` sends the request. It also has a lifecycle (`pending`, `success`, `error`) but isn't cached by key.
2. On success, `queryClient.invalidateQueries({ queryKey: ['products'] })` marks every matching `Query` stale. The ones with active observers refetch immediately, and inactive ones refetch the next time they're used.
3. **Optimistic updates** write the expected result into the cache first with `setQueryData`, then roll back if the server rejects it. The cache can briefly show a guess about the server's state, but the guess is always reconciled.

## 6. Side-by-side

| | UI state | Server state |
|---|---|---|
| Source of truth | This process's memory | A remote database |
| What the app holds | The real value | A copy (cache entry) |
| Can be stale | Never | Always, eventually |
| Read | Synchronous | Async: network (native threads) + parse (JS thread) |
| Identity | Fiber position / store path | Hashed query key |
| Lifetime | Fiber or JS runtime | Until `gcTime` after the last observer leaves |
| Tool | `useState`, Context, Redux/Zustand | TanStack Query / RTK Query |

A screen usually uses both, connected through the key. UI state (`filter`) **chooses which cache entry** you're looking at: `['products', filter]`.

## Rule of thumb

If the authoritative copy lives on a server, you're managing a **cache**, so use a cache engine. If the authoritative copy lives in this app's memory, it's UI state, and `useState` or a store is the right tool.

Day 3 applies this by fetching real data with TanStack Query.
