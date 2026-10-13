# Global vs local state: where state physically lives

"Local" and "global" aren't just style choices. They describe **where the data sits in memory**, **who owns it**, **how long it lives**, and **how an update reaches the screen**. In React Native, state has three possible homes:

1. **On a Fiber node**: local state (`useState`, `useReducer`).
2. **On a provider's Fiber, shared through Context**: still local state, but delivered far down the tree.
3. **Outside React, in a plain JS object on the Hermes heap**: global stores (Redux, Zustand).

## 1. Local state lives on the component's Fiber

Each component instance in the tree has a **Fiber** node, the reconciler's object for that position in the tree. Its hooks are stored as a **linked list** on `fiber.memoizedState`:

```
fiber (LikeButton)
  memoizedState -> hook #1 { memoizedState: false, queue, next } // useState(false)
                   hook #2 { memoizedState: [...], next }        // useMemo
                   hook #3 { ... }
```

React finds "your" state by **call order**. On every render it walks this list one hook at a time. That's why hooks can't be inside conditions: skipping one shifts every hook after it onto the wrong node.

### What `setState` actually does

The setter is `dispatchSetState`, already bound to that fiber and that hook's queue. Calling it:

1. Creates an **update object** `{ action, lane, next }` and appends it to the hook's queue.
2. **Eager bailout**: if the queue was empty, React computes the new state immediately. If it is `Object.is` equal to the current state, React stops there and doesn't schedule a render.
3. Otherwise it calls `scheduleUpdateOnFiber`. It marks the fiber's **lanes** (update priority, a bitmask) and walks up through `return` pointers, setting **`childLanes`** on every ancestor up to the root.
4. The root is scheduled with the **Scheduler**. In the New Architecture, this delegates to React Native's C++ **RuntimeScheduler**, which runs the work on the JS thread.
5. The render starts at the root, but every fiber with no lanes and no `childLanes` is skipped (bailed out). React goes straight down the marked path, re-runs the component, and processes the queued updates to get the new state.

Several `setState` calls in the same event or tick are **batched** into one render (automatic batching since React 18).

The cost of a local update is roughly **that component and the children it re-renders**. Nothing outside that path does any work.

### Lifetime: state lives as long as the Fiber

A fiber's identity is **position in the tree + component type + `key`**. While those stay the same between renders, the fiber and its state persist. When any of them changes, or the component unmounts, the fiber is deleted and its hook list goes with it. Hermes' garbage collector (**Hades**) reclaims the memory once nothing references it.

This matters in React Navigation:

- In a **native stack**, pushing a new screen does **not** unmount the previous one. Its fiber and native views (a `UIViewController` on iOS, a `Fragment` on Android) stay alive. Local state survives going forward and coming back.
- **Popping** a screen unmounts it, and its local state is gone.
- **Tabs** mount lazily and then stay mounted, so their local state survives switching tabs.
- Changing a component's `key` is a deliberate way to reset its state: React treats it as a new fiber.

## 2. Lifting state up moves the hook to another Fiber

"Lifting state up" means moving the `useState` call to a common parent. Mechanically, the hook node now sits on the **parent's fiber**. Every update marks the parent and re-renders it and its subtree.

The data is still local: it is owned by a fiber and dies with it. The cost of an update grows with how high up the tree the hook sits.

## 3. Context: local state delivered far down the tree

A Context provider usually wraps a `useState`, so the data still lives on **the provider's fiber**. Context only changes how the data **reaches** components: through `_currentValue` and the dependency lists (see [07](07-context-api.md)).

The lifetime is the provider's lifetime. The update cost is **every consumer**, because consumers subscribe to the whole value.

## 4. Global state lives outside React

A Redux or Zustand store is created at **module scope**:

```
Hermes heap
  module registry -> store module -> closure { state, listeners: Set<fn>, dispatch/setState }
```

React doesn't own it, so it isn't part of the Fiber tree. Components connect to it with **`useSyncExternalStore(subscribe, getSnapshot)`**:

1. On mount, React calls `subscribe(listener)`, and the store adds the listener to its set.
2. On every render, React calls `getSnapshot()`, usually `selector(store.getState())`.
3. When the store changes, it calls **every** listener. Each listener re-runs `getSnapshot()` and compares the result with the previous snapshot using `Object.is`. Only components whose snapshot changed get an update scheduled, at **SyncLane** (highest priority).
4. On unmount, React unsubscribes.

### Why `useSyncExternalStore` exists: tearing

React 18 can **pause** a concurrent render partway through. If an external store changed during that pause, the components rendered before the pause would show the old value and the ones after it the new value. That inconsistency is called **tearing**. `useSyncExternalStore` checks the snapshot again before committing. If it changed, React throws away the half-finished render and re-renders synchronously, so the screen always shows one consistent store version.

### Lifetime: as long as the JS runtime

The store is reachable from the module registry, so the GC **never collects it**. The data:

- survives any screen mounting or unmounting,
- is lost when the app process dies. That includes Android killing a backgrounded app to reclaim memory, and a full JS reload in development. Fast Refresh resets it only if the store's own file is edited.

To survive a restart, the store has to be serialized to disk (`redux-persist` / Zustand `persist` with AsyncStorage). See [09](09-redux-toolkit-or-zustand.md).

## 5. Comparing update costs

| | Where the data lives | Who is notified on change | Who re-renders | Dies when |
|---|---|---|---|---|
| Local `useState` | Hook list on one fiber | That fiber | It and its subtree | Component unmounts |
| Lifted state | Hook list on an ancestor fiber | The ancestor | Ancestor and its subtree | Ancestor unmounts |
| Context | Provider's fiber | Every fiber depending on the context | Every consumer | Provider unmounts |
| External store | JS object in the Hermes heap | **Every subscribed listener** (each runs its selector) | Only components whose selected value changed | JS runtime stops |

Two points follow from the table:

- In an external store, **every dispatch runs every mounted selector**. Selectors must be cheap and return stable references (see [09](09-redux-toolkit-or-zustand.md)).
- Pushing typing (a `TextInput` value) into a global store means every keystroke runs every selector in the app, plus a round trip through the store. As local state, a keystroke only touches one fiber.

## 6. Why not make everything global?

These are mechanical costs, not just style:

- **Lifetime mismatch**: screen state should die with the screen's fiber. In a store it lingers, so a re-opened screen shows stale values unless you reset them yourself.
- **Memory**: data in a module-level store can never be garbage collected.
- **Notification fan-out**: each change wakes every listener, not just the components that care about it.
- **Coupling**: a component that reads from a store can't be rendered (or tested) without that store existing.

## 7. Where server data fits

Data fetched from an API lives in neither home. Its source of truth is a **remote database**, and the app only holds a cached copy that can go stale. That is **server state**, covered in [10](10-server-state-vs-ui-state.md).

## Rule of thumb

Put state on the **lowest Fiber** that every reader sits under. Move it outside React only when its lifetime has to outlive any single subtree, or when many distant components need to read **different slices** of it efficiently.
