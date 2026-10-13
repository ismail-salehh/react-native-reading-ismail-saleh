# useCallback

`useCallback` **keeps the same function between renders**, so React doesn't hand children a new function reference every time.

## Why do we need it?

Imagine a parent screen that renders a button component. Every time the parent re-renders, JavaScript runs the component function again, and **every function defined inside it is created again**. The new function does exactly the same thing, but it is a different object in memory, so it has a **different reference**.

```
render 1: handlePress -> function #1
render 2: handlePress -> function #2   (same code, different object)
#1 === #2  -> false
```

On its own that's harmless. It becomes a problem when the function is passed to a child wrapped in `React.memo`: memo compares props by reference, sees a "new" `onPress`, and re-renders the child anyway.

## How it works

| | What the child receives | Child wrapped in `React.memo` |
|---|---|---|
| **Without useCallback** | A new function reference every render | Re-renders, because the prop looks changed |
| **With useCallback** | The same reference, until dependencies change | Skips rendering if its other props are unchanged too |

## React Native example

```tsx
const MyButton = React.memo(({ onPress }: { onPress: () => void }) => {
  return <Pressable onPress={onPress}><Text>Press</Text></Pressable>;
});

function Screen() {
  const [count, setCount] = useState(0);

  const handlePress = useCallback(() => {
    console.log('Button pressed');
  }, []);

  return (
    <>
      <Text>{count}</Text>
      <MyButton onPress={handlePress} />
    </>
  );
}
```

When `count` changes, `Screen` re-renders, but `handlePress` is the same function, so `MyButton` skips its render.

## Under the hood

- `useCallback(fn, deps)` is the same as `useMemo(() => fn, deps)`. It stores `[fn, deps]` in a hook slot on the component's **Fiber** node.
- On each render, React compares the dependencies with `Object.is`. If they're unchanged, it returns the **stored** function and throws away the new one you just wrote. If any changed, it stores and returns the new one.
- `useCallback` doesn't prevent the function from being created. JavaScript still creates it on every render. It only decides **which reference you get back**.

## Dependencies and stale closures

A function "remembers" the variables from the render it was created in. That's called a **closure**. If you leave a dependency out, the cached function keeps seeing **old values**:

```tsx
const logCount = useCallback(() => {
  console.log(count); // always logs the value from the first render
}, []);               // missing `count`
```

Two ways to fix it:

- Add the value to the dependencies: `[count]`.
- Use the updater form of a setter, which needs no dependency: `setCount((c) => c + 1)`.

## When to use it in React Native

- Functions passed to children wrapped in `React.memo`.
- `renderItem`, `keyExtractor` and callbacks passed to `FlatList` rows, where many rows would otherwise re-render.
- Functions used as dependencies of `useEffect` / `useMemo`, so the effect doesn't re-run on every render.

## Important

> `useCallback` doesn't stop the **parent** from re-rendering. It only keeps the function reference stable.

> Without `React.memo` on the child (or a dependency list using the function), `useCallback` gives you **nothing**. The child re-renders anyway, because by default every child re-renders with its parent.
