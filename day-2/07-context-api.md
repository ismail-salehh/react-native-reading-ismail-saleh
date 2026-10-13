# Context API: what actually happens

Context lets a component read a value provided by a distant ancestor without the value passing through every component in between. This file explains how the **React reconciler** (Fiber) makes that work. That logic lives in the `react-reconciler` package, the same one React DOM uses. In React Native it runs on the **JS thread** inside **Hermes**, and the result goes to the **Fabric** renderer.

## 1. What `createContext` returns

`createContext(defaultValue)` returns a plain object:

```js
{
  $$typeof: Symbol.for('react.context'),
  _currentValue: defaultValue,   // the value a reader sees "right now"
  _currentValue2: defaultValue,  // same, for a secondary renderer
  Provider: ...,                 // in React 19 the context object itself is the provider
  Consumer: ...,
}
```

The object holds no state and has no subscriber list. It is a slot (`_currentValue`). The reconciler overwrites that slot while it walks the tree.

## 2. How the "nearest provider" is found: a value stack

React renders by walking the **Fiber tree**, which is a linked list of fiber nodes (`child`, `sibling`, `return`). Every element you render has one fiber. The work loop calls `beginWork` on the way down into a fiber and `completeWork` on the way back up.

When the walk reaches a **Provider fiber**:

```
beginWork(Provider)    -> pushProvider(context, newValue)
                            stack.push(context._currentValue)   // save the old value
                            context._currentValue = newValue      // overwrite the slot
   ...render all descendants...
completeWork(Provider) -> popProvider(context)
                            context._currentValue = stack.pop()   // restore
```

While React is inside a provider's subtree, `_currentValue` holds that provider's value. Nested providers push on top, and leaving a subtree pops back. This gives you three rules:

- The **nearest provider wins**, because it pushed most recently.
- With **no provider above**, the slot still holds `defaultValue`.
- Siblings outside the provider never see its value, because it was popped before React reached them.

There is no lookup up the tree. Reading context costs O(1): React reads one property.

## 3. What `useContext` does: read and register a dependency

`useContext(Ctx)` calls the reconciler's `readContext`:

1. It returns `Ctx._currentValue`.
2. It appends `{ context: Ctx, memoizedValue: value }` to **`fiber.dependencies`**, a linked list on the reading component's fiber.

The dependency list is how React later knows which fibers care about which context. The dependency covers the **whole context value**, not individual fields. If a component reads `ctx.theme` but the provider's value also contains `user`, a change to `user` still counts as a change for that component.

React 19's `use(Ctx)` runs through the same `readContext`. Because it isn't a hook it can be called conditionally, but the mechanism is the same.

## 4. What happens when the provider value changes

When the provider re-renders, React compares the old and new `value` with **`Object.is`**:

- **Same reference**: no context work. If the provider's `children` are also the same elements, React **bails out** and skips the whole subtree.
- **Different reference**: React has to find every fiber below that depends on this context and schedule it.

Finding them is **context propagation**. React walks the provider's subtree, and for each fiber it checks `fiber.dependencies` for this context. A match gets marked with pending work (**lanes**, the bitmask React uses for update priority), and its ancestors are marked in `childLanes` so the work loop knows to come down that path. React 18 walked the subtree eagerly. Newer React does the check lazily during the render (`enableLazyContextPropagation`). Either way, every consumer is found and re-rendered.

This explains the most common surprise:

> `React.memo` does not stop context updates.

`memo` compares **props** before re-rendering a component. A context consumer gets re-rendered because **its own fiber has scheduled work**. React checks for that work (`checkScheduledUpdateOrContext`) before it considers the props bailout. So the memo check never gets a chance to skip it.

### Why inline objects hurt

```jsx
<Ctx.Provider value={{ theme, toggle }}>
```

Each render of the provider creates a new object, so `Object.is` fails every time. Every consumer re-renders whenever the provider's parent renders, even if `theme` didn't change. `useMemo` around the value keeps the reference stable until its dependencies change, and then the comparison in step 4 passes.

## 5. What it costs in React Native

A context change leads to this chain:

```
JS thread (Hermes)                         C++ (Fabric)                    UI thread
-------------------                        -------------                   ---------
setState in provider
-> render phase: run every consumer's
   function, diff old vs new elements
-> commit phase: changed host props  -->   clone ShadowNodes (immutable)
                                           -> Yoga recomputes layout
                                           -> diff old vs new shadow tree  -> apply mutations to
                                                                              UIView / android.view.View
```

- The **render phase** is the expensive part for context. Every consumer function runs on the JS thread, whether or not its output changes. While that happens, the JS thread can't process touches handled in JS, JS-driven animations, or other updates.
- Fabric only receives **host props that actually changed**. If 100 consumers re-render but only one `<View>` changes color, Fabric clones only the shadow nodes on that one path.
- Work driven from the UI thread (Reanimated worklets, native-driver animations, native gestures) isn't blocked, but anything waiting on JS is.

## 6. What Context is and isn't

From the mechanism above:

- Context is a **dependency-injection channel**, not a state manager. The state lives in the provider component's `useState` (on the provider's fiber). Context only carries it down.
- It has **no selectors**. The unit of subscription is the whole value. That's why state that changes often (typing, scroll position, timers) is a poor fit: each change re-runs every consumer.
- **Splitting contexts** (theme, auth, language) narrows propagation. A theme change then only reaches fibers that have the theme context in their `dependencies` list.
- Splitting **state and setters** into two contexts helps for the same reason. The setter from `useState` has a stable identity, so a context that only carries setters never changes, and components that only dispatch never re-render.

## Summary

| Step | What the engine does |
|---|---|
| `createContext` | Creates a slot object (`_currentValue`) |
| Provider renders | Pushes its value onto a stack and overwrites the slot |
| `useContext` | Reads the slot and records a dependency on the reader's fiber |
| Value changes (`Object.is` fails) | Finds every dependent fiber below and schedules it |
| Commit | Fabric applies only the host props that changed |
