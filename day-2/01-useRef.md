# useRef

`useRef` is a hook that **keeps a value between renders without causing a re-render** when that value changes.

It returns a plain object, `{ current: initialValue }`. React gives you the **same object** on every render, and you can read or change `.current` whenever you want.

## Use 1: getting at a native component

Imagine a login screen where the email input should receive focus when the screen opens. In React Native you give the `TextInput` a ref and call its `focus()` method:

```tsx
const inputRef = useRef<TextInput>(null);

useEffect(() => {
  inputRef.current?.focus();
}, []);

return <TextInput ref={inputRef} />;
```

What each part does:

- `useRef<TextInput>(null)` creates the box. It starts as `null` because the input doesn't exist yet during the first render.
- `ref={inputRef}` tells React to put the native component instance into `inputRef.current` once it's mounted, and to set it back to `null` when it unmounts.
- `inputRef.current?.focus()` calls a method on the native component directly. The `?.` guards against `null`. It runs inside `useEffect` because refs are only filled in after the commit, never during render.

The same pattern lets you call `scrollToIndex` on a `FlatList`, `scrollTo` on a `ScrollView`, or `measure` on a `View`.

## Use 2: remembering a value without re-rendering

Suppose you want to count how many times a button was pressed for internal tracking, without showing the count on screen.

- With `useState`, every press changes the state and re-renders the component, for nothing.
- With `useRef`, the count is kept, and nothing re-renders.

```tsx
const pressCount = useRef(0);

const onPress = () => {
  pressCount.current += 1; // stored between renders, but the UI does not update
};
```

This is useful for:

- timer IDs from `setTimeout` / `setInterval`, so you can clear them later,
- the previous value of a prop or state,
- flags such as "is a request already running?",
- any mutable data that doesn't need to appear on screen.

## Under the hood

- `useRef` stores its object in a hook slot on the component's **Fiber** node (React's internal record of that component instance). On every later render React returns that same object, which is why it survives re-renders.
- React **doesn't track** `.current`. Assigning to it is an ordinary JavaScript property write on the Hermes heap. No update is queued and nothing is scheduled, which is why the UI doesn't change.
- For `ref={...}` on a host component, React fills in `.current` during the **commit phase** with the native instance. Under Fabric, that object can call into native code, such as focusing the input.
- The ref lives as long as the component does. It resets when the component unmounts or its `key` changes.

## useRef vs useState

| | useRef | useState |
|---|---|---|
| Keeps the value between renders | Yes | Yes |
| Changing it re-renders the component | **No** | **Yes** |
| Can hold a native component (`TextInput`, `FlatList`) | Yes | No |
| Good for values shown on screen | No | Yes |
| How you update it | `ref.current = x` (immediately) | `setX(x)` (applied on the next render) |

## When to use which

- **useRef**: focus an input, scroll a list, store a timer ID, remember a previous value, or keep anything that shouldn't trigger a render.
- **useState**: the screen must change when the value changes, for example a counter, form values, or a loading indicator.

**The main rule:** if changing a value should update what the user sees, use `useState`. If you only need to remember something, or talk to a native component directly, use `useRef`.

## Pitfalls

- Don't read or write `ref.current` **during render** (in the component body). Do it in effects and event handlers. Otherwise the output depends on something React doesn't know about.
- Don't expect a screen update after changing `.current`. If the UI needs the value, it belongs in state.
