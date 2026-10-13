# useMemo

`useMemo` **caches the result of a calculation between renders**, and only recalculates when the inputs it depends on change.

## Why?

Imagine you have a list of a thousand plants, and you show only the ones matching a search text. Every time the component re-renders, React runs the component function again, so the filtering runs again too, **even if neither the plants nor the search text changed**. A re-render can happen for unrelated reasons, like a parent re-rendering or another piece of state changing.

`useMemo` lets you reuse the previous result until the data it depends on changes.

## How to use it

```tsx
const filteredPlants = useMemo(() => {
  return plants.filter((plant) =>
    plant.name.toLowerCase().includes(search.toLowerCase())
  );
}, [plants, search]);
```

There are three things to understand:

1. **The calculation**: the function filters the plants by the search text.
2. **The dependencies**: `[plants, search]` tells React which values the calculation depends on.
3. **The cached result**: if neither dependency changed since the last render, React returns the previous array without running the function.

## Under the hood

- `useMemo` stores `[result, dependencies]` in a hook slot on the component's **Fiber** node.
- On each render, React compares every new dependency with the old one using **`Object.is`**, which is a reference comparison for objects and arrays.
  - **All the same**: return the stored result. The function isn't called.
  - **Any different**: call the function, store the new result and dependencies, and return it.
- That's why dependencies must be stable. If `plants` is a new array on every render (for example `data.map(...)` written inline), `Object.is` always fails and the cache never helps.
- The comparison itself isn't free. For a cheap calculation, `useMemo` can cost more than it saves.

## Its second job: stable references

Each render that creates `{ ... }` or `[ ... ]` produces a **new object in memory**, even if its contents are identical. A child wrapped in `React.memo` compares props by reference, so it sees "new prop" and re-renders.

`useMemo` returns the **same reference** until the dependencies change, so the memoized child can skip rendering:

```tsx
const style = useMemo(() => ({ color: theme.primary }), [theme.primary]);
return <PlantRow style={style} />;
```

The same idea applies to the `value` of a Context provider (see [07](07-context-api.md)).

## When to use it in React Native

1. **Large lists**: filtering, sorting or grouping data for a `FlatList` / `SectionList`.
2. **Expensive calculations**: anything that takes noticeable time on the JS thread, where it would delay touches and animations.
3. **Stable object or array references**: values passed to a `React.memo` child, or used as a Context value or effect dependency.

## When not to use it

- For simple calculations (adding two numbers, formatting one string). The bookkeeping costs more than the work.
- As a guarantee. React may drop the cache in some cases, so code must still be correct if the function runs again.
- To trigger side effects. The function should only calculate and return a value. Fetching or logging belongs in `useEffect`.
