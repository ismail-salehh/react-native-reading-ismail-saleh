# Debouncing and throttling

Some events fire **very often**: every keystroke, every scroll position, every resize. Running heavy work (a request, a re-render, a calculation) on each one wastes the network and the JS thread. Debouncing and throttling both **reduce how often** the work runs, in different ways.

## The difference

```
events:      x  x x x  x       x x x x x x x x x x x x
             |---- user typing ----|  pause  |---- scrolling ----|

debounce:                             X                           X
             (runs once, after the events STOP for N ms)

throttle:    X        X        X        X        X        X
             (runs at most once every N ms, while events keep coming)
```

| | Debounce | Throttle |
|---|---|---|
| Runs | After the events stop for `N` ms | At most once per `N` ms |
| During a continuous stream | **Never** | Regularly |
| Good for | Search-as-you-type, autosave, validation of a field | Scroll position, drag/resize, analytics of scroll depth, "infinite" buttons |

**Debounce = "wait until they're done". Throttle = "not more often than this".**

## Debounce: search as you type

Without it, typing "shoes" sends 5 requests. With a 300 ms debounce, it sends 1:

```ts
import { useEffect, useState } from 'react';

export function useDebouncedValue<T>(value: T, delay = 300) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id); // each new value cancels the previous timer
  }, [value, delay]);

  return debounced;
}
```

```tsx
function SearchScreen() {
  const [text, setText] = useState('');
  const query = useDebouncedValue(text.trim(), 300);

  const { data } = useQuery({
    queryKey: ['search', query],
    queryFn: ({ signal }) => searchProducts(query, signal),
    enabled: query.length >= 2,
  });

  return <TextInput value={text} onChangeText={setText} />;
}
```

- The **input** updates on every key (so typing feels instant); only the **query** is debounced.
- Debouncing reduces requests, but responses can still arrive out of order. Combine it with cancellation (Day 3: [Request cancellation](../day-3/12-request-cancellation.md)). TanStack Query handles that with `signal` and the query key.

## Debouncing a function

Sometimes you want to debounce a **call**, not a value (autosave, analytics). Use lodash (or write one), and keep the same debounced function across renders:

```ts
import debounce from 'lodash/debounce';
import { useEffect, useMemo } from 'react';

const saveDraft = useMemo(
  () => debounce((text: string) => api.put('/drafts/1', { text }), 1000),
  [],
);

useEffect(() => () => saveDraft.cancel(), [saveDraft]); // cancel on unmount
```

Creating `debounce(...)` inside the component body without `useMemo`/`useRef` makes a **new** debounced function every render, and nothing is ever debounced.

`lodash` debounced functions also have `.flush()`, useful to save immediately when the user leaves the screen.

## Throttle: scroll events

```tsx
import throttle from 'lodash/throttle';

const onScrollThrottled = useMemo(
  () => throttle((y: number) => setShowBackToTop(y > 600), 200),
  [],
);

<FlatList
  onScroll={(e) => onScrollThrottled(e.nativeEvent.contentOffset.y)}
  scrollEventThrottle={16}
/>
```

- `scrollEventThrottle` controls how often the native side sends scroll events to JS (16 ms ~ every frame). Lower frequency = less JS work.
- For values that drive an **animation** (a header that shrinks on scroll), don't throttle in JS at all. Use `Animated.event` with `useNativeDriver` or Reanimated's `useAnimatedScrollHandler`, which run on the UI thread ([JS thread vs UI thread](20-js-thread-vs-ui-thread.md)).

## Preventing double taps

A related case: the user taps "Pay" twice quickly. Throttling `onPress` works, but disabling the button while the request is pending is clearer and safer (TanStack Query's `isPending`, Day 3: [TanStack Query](../day-3/18-tanstack-query.md)).

## Picking the delay

| Use | Typical |
|---|---|
| Search input | 250-500 ms |
| Autosave | 1-2 s |
| Scroll / resize logic | 100-200 ms |
| Field validation while typing | 300-500 ms |

Too short and it doesn't help; too long and the app feels laggy.

## Rule of thumb

Debounce when you care about the final value (search, autosave). Throttle when you need regular updates during a stream (scroll, drag). Keep the debounced/throttled function stable across renders, and cancel it on unmount.
