# Memory leaks

A memory leak happens when the app keeps objects in memory that it no longer needs, so the garbage collector can't free them. Each leak may be small, but they add up while the user navigates around. The app gets slower, and eventually the OS kills it (on mobile, that looks like a crash with no JS error).

## How memory is freed in JS

The JS engine frees an object when **nothing references it anymore**. A leak is almost always **a reference that outlives its purpose**, held by something long-lived:

```
long-lived thing (module variable, global listener, timer, native module)
 -> holds a callback
     -> callback's closure holds the component's state, props, big data
         -> none of it can be freed, even after the screen is closed
```

## The common leaks in React Native

### 1. Listeners that are never removed

```ts
// ✕ leaks: every mount adds another listener
useEffect(() => {
  AppState.addEventListener('change', onChange);
}, []);

// ✓
useEffect(() => {
  const sub = AppState.addEventListener('change', onChange);
  return () => sub.remove();
}, []);
```

The same applies to `Keyboard`, `Dimensions`, `Linking`, `NetInfo.addEventListener`, `messaging().onMessage`, navigation `addListener`, and any event emitter. They return an unsubscribe function or a subscription; call it in the cleanup.

### 2. Timers

```ts
useEffect(() => {
  const id = setInterval(poll, 5000);
  return () => clearInterval(id);
}, []);
```

An interval that isn't cleared keeps running forever, keeps its closure alive, and keeps making requests after the screen is gone.

### 3. Subscriptions and sockets

WebSockets, Firebase listeners, store subscriptions outside React, location watchers (`watchPosition` -> `clearWatch`). Anything you **open**, you must **close**.

### 4. Module-level caches that only grow

```ts
const cache = new Map<string, Product>(); // lives as long as the app

export function remember(p: Product) {
  cache.set(p.id, p); // never removed
}
```

Fine for 100 items, a leak for 100,000. Give caches a limit (LRU) or use a library that does (TanStack Query removes unused data after `gcTime`).

### 5. Screens that stay mounted

In a stack, screens below the current one **stay mounted**. A user who opens 30 product screens with `push` has 30 screens in memory, each with its images and state. Use `navigate` when you don't need a new copy, and consider `freezeOnBlur` / limiting stack depth for deep flows.

### 6. Large images

Not a leak strictly, but the most common cause of out-of-memory kills: full-size photos in small views ([Image optimization](22-image-optimization.md)).

## What about "setState on an unmounted component"?

Older React showed a warning: *"Can't perform a React state update on an unmounted component... indicates a memory leak"*. React 18 **removed** that warning, because in most cases (a request finishing after unmount) it isn't a real leak: the promise resolves once and then everything is freed.

It's still good practice to **cancel** such requests (Day 3: [Request cancellation](../day-3/12-request-cancellation.md)), to save network and avoid acting on stale data. But the real leaks are the long-lived things above: listeners, timers, subscriptions, caches.

## Finding leaks

1. **Reproduce**: open and close the same screen 10-20 times.
2. **Watch memory**:
   - **React Native DevTools -> Memory**: take a heap snapshot, repeat the action, take another, compare. Objects that grow each time (your screen's components, arrays of data) are leaking.
   - **Android Studio Profiler** (Memory) and **Xcode Instruments** (Allocations, Leaks) show native memory too, including images.
3. **Look for missing cleanups**: search for `addEventListener`, `setInterval`, `subscribe`, `on(` and check each has a matching removal.

Memory that grows and then **drops back** after a garbage collection is normal. Memory that keeps growing step by step with each repeat is a leak.

## Rule of thumb

Every `useEffect` that subscribes, listens, or starts a timer returns a cleanup that undoes it. Give caches a limit, don't keep huge images around, and test by repeating a flow many times while watching the memory graph.
