# Custom hooks

A custom hook is a plain function whose name starts with `use` and that calls other hooks (`useState`, `useEffect`, `useRef`...). It lets you extract and reuse **stateful logic** between components. Each component that calls it gets its own separate state.

## Rules

- The name must start with `use` (so React and the linter can check the rules of hooks).
- Call hooks only at the top level, never inside conditions or loops.
- Only call hooks from components or other hooks.

## Example: debounced value (React Native)

```jsx
import { useState, useEffect } from 'react';

function useDebounce(value, delay = 500) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);

  return debounced;
}

function SearchScreen() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query);
  // call the API with debouncedQuery, not query
  return <TextInput value={query} onChangeText={setQuery} />;
}
```

## Good candidates in React Native

- `useDebounce`, `useToggle`, `useDisclosure` (modal open/close)
- `useKeyboard` (keyboard height and visibility)
- `useNetworkStatus` (NetInfo), `useAppState` (foreground/background)
- `useFetch`, `useAuth`, `useAsyncStorage`

## Tips

- Return an array for simple pairs (`[value, setValue]`) or an object for many values.
- Wrap returned functions in `useCallback` so consumers can pass them to memoized children.
- Don't create a hook for logic used only once. Extract when it repeats or the component gets hard to read.
