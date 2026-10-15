# JS thread vs UI thread

A React Native app runs on several threads at once. Two of them explain most performance problems: the **JS thread**, where your React code runs, and the **UI thread** (main thread), where the native views are drawn and touches are received.

## The threads

| Thread | Runs | If it is blocked |
|---|---|---|
| **UI (main) thread** | Native views, drawing, scrolling, native animations, receiving touches | The whole app freezes. Scrolling stutters. Android may show "App not responding" |
| **JS thread** | Your JS: React renders, state updates, event handlers, business logic, `fetch` callbacks | Taps don't respond, JS animations stop, but native scrolling still moves |
| Background threads | Layout calculation (Yoga), image decoding, network, native module work | Usually invisible |

JS is single-threaded: **one thing at a time** on the JS thread. A long loop blocks every other JS task, including the handler for the tap the user just made.

## 60 frames per second

Smooth motion needs a new frame about every **16.7 ms** (60 FPS), or 8.3 ms on 120 Hz screens.

```
frame budget: |---- 16.7 ms ----|---- 16.7 ms ----|---- 16.7 ms ----|
JS thread:    [render][handler]  [   a 200 ms JSON.parse ... blocks 12 frames   ]
user sees:    smooth             tap ignored, animation frozen ...........
```

If the JS thread is busy for 200 ms, roughly 12 frames are dropped for anything JS drives.

## How a tap travels

```
finger down
 -> UI thread receives the touch
 -> event sent to the JS thread
 -> JS runs onPress, calls setState
 -> React renders, computes what changed
 -> changes committed to native views (layout on a background thread)
 -> UI thread draws the new frame
```

If any step on the JS thread is slow, the user sees the delay between tap and response.

## Seeing it

- **Perf Monitor**: Dev Menu -> "Perf Monitor" shows **JS FPS** and **UI FPS** separately.
  - JS FPS drops, UI stays at 60: your JS is too slow (renders, heavy logic).
  - UI FPS drops: native work is too heavy (huge images, too many views, expensive shadows).
- **React Native DevTools** profiler and the React Profiler show which components render and how long they take.
- **Test performance in release builds.** Dev mode adds checks, warnings, and an unoptimized bundle. A screen that stutters in dev can be fine in release; one that stutters in release is a real problem.

## Common causes of a busy JS thread

| Cause | Fix |
|---|---|
| Too many or too large re-renders | `React.memo`, stable callbacks, split state ([Unnecessary re-renders and large lists](21-re-renders-and-large-lists.md)) |
| Rendering a long list with `map` | `FlatList` / `FlashList` |
| Heavy work in an event handler (sorting 10,000 items, parsing a big JSON) | Do it on the server, paginate, or move it off the JS thread |
| Work on every keystroke or scroll event | [Debouncing and throttling](23-debouncing-throttling.md) |
| `console.log` in loops | Remove logs in release (e.g. `babel-plugin-transform-remove-console`) |
| JS-driven animations | Native-driven animations (below) |

## Animations: keep them off the JS thread

An animation updated from JS needs the JS thread **every frame**. If JS is busy, the animation stutters.

- **Animated** with `useNativeDriver: true` sends the whole animation to the UI thread once, then it runs there without JS. Works for `transform` and `opacity`, not for layout properties like `width`.
- **Reanimated** runs animation code ("worklets") on the UI thread ([Reanimated](30-reanimated.md)).
- **Gesture Handler** recognizes gestures natively instead of in JS ([Gesture Handler basics](31-gesture-handler.md)).

```ts
Animated.timing(opacity, {
  toValue: 1,
  duration: 300,
  useNativeDriver: true, // runs on the UI thread
}).start();
```

## What the New Architecture changes

In the old architecture, every message between JS and native went through an asynchronous "bridge" as serialized JSON. The New Architecture lets JS call native code directly and synchronously when needed, and lets React render with priorities. The thread model is the same: you still must not block the JS thread. See [Old vs new architecture](26-old-vs-new-architecture.md).

## Rule of thumb

The UI thread draws, the JS thread thinks. Keep JS work per frame small, move animations and gestures to the UI thread, and measure with the Perf Monitor in a release build before optimizing.
