# Context API

Context lets you pass data through the component tree without passing props at every level ("prop drilling"). Good for values many components need: theme, language, auth user.

## Three steps

1. `createContext` to create it.
2. A `Provider` to supply the value to the tree.
3. `useContext` to read it in any descendant.

## Example: theme (React Native)

```jsx
import { createContext, useContext, useState, useMemo, useCallback } from 'react';
import { View, Text, Button } from 'react-native';

const ThemeContext = createContext(null);

export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  const toggle = useCallback(() => setTheme((t) => (t === 'light' ? 'dark' : 'light')), []);
  const value = useMemo(() => ({ theme, toggle }), [theme, toggle]);

  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}

export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be used inside ThemeProvider');
  return ctx;
}

function Screen() {
  const { theme, toggle } = useTheme();
  return (
    <View style={{ backgroundColor: theme === 'light' ? '#fff' : '#111' }}>
      <Button title="Toggle" onPress={toggle} />
    </View>
  );
}
// Wrap the app: <ThemeProvider><Screen /></ThemeProvider>
```

## Important caveats

- When the Provider `value` changes, **every consumer re-renders**, even `React.memo` ones. Memoize the value with `useMemo`.
- Passing an inline object (`value={{ a, b }}`) creates a new reference every render and re-renders all consumers.
- Context is not a state manager: it has no selectors, so frequently changing state (e.g. typing, scroll) is a poor fit.
- Split contexts by concern (theme, auth, language) instead of one big one.
