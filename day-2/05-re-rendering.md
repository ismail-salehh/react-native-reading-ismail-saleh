# Re-rendering: when and why it happens

A re-render means React calls your component function again to work out what the UI should look like. It is not the same as the screen being repainted: React diffs the result and only updates native views that changed.

## What triggers a re-render

1. **State changes**: calling `setState` / `useState` setter with a new value.
2. **Parent re-renders**: all children re-render by default, even if their props are identical.
3. **Context changes**: every component consuming that context re-renders.
4. **Hook-driven updates**: store subscriptions (Redux, Zustand), navigation focus, etc.

## What does NOT trigger one

- Changing a `useRef` value (`ref.current = x`).
- Mutating a variable or object directly (`user.name = 'x'`). React never sees it.
- Setting state to the same value (`Object.is` equal), React bails out.

## Why it matters in React Native

- Renders run on the JS thread. Too many (or heavy) ones cause dropped frames and laggy scrolling or typing.
- Lists multiply the cost: one parent render can re-render every visible row.

## How to control it

- Keep state as local as possible so fewer components re-render.
- `React.memo` to skip children whose props didn't change.
- `useCallback` / `useMemo` to keep function, object and array props stable.
- Use selectors with Redux/Zustand so components subscribe only to what they need.
- Find problems with the React DevTools Profiler, not guesses.
