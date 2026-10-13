# React.memo

`React.memo` is a higher-order component (HOC) that stops a component from re-rendering when its props haven't changed.

Think of it as saying: **"re-render this component only if its props are different."**

## How it works

1. The parent re-renders.
2. React compares the new props with the previous props of the memoized child using a **shallow comparison** (`Object.is` on each prop).
3. If every prop is equal, React skips rendering the child and reuses the previous result.
4. If any prop changed, the child re-renders as usual.

Shallow means:

- Primitives (`string`, `number`, `boolean`) are compared by **value**.
- Objects, arrays and functions are compared by **reference**, not by content.

## Basic example (React Native)

```jsx
import React from 'react';
import { TouchableOpacity, Text, StyleSheet } from 'react-native';

const Button = React.memo(({ title, onPress }) => {
  console.log('Button rendered:', title);
  return (
    <TouchableOpacity onPress={onPress} style={styles.btn}>
      <Text style={styles.text}>{title}</Text>
    </TouchableOpacity>
  );
});

const styles = StyleSheet.create({
  btn: { padding: 12, backgroundColor: '#6d0a30', borderRadius: 8 },
  text: { color: '#ffffffdb', textAlign: 'center' },
});

export default Button;
```

## The reference-equality **trap**

`React.memo` is useless if you pass a new object or function on every render, because the reference changes each time.

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  // New function on every render -> Button's props always "change"
  return (
    <>
      <Text>{count}</Text>
      <Button title="Add" onPress={() => setCount(count + 1)} />
    </>
  );
}
```

Fix it by keeping references stable with `useCallback` / `useMemo`:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const handlePress = useCallback(() => setCount((c) => c + 1), []);

  return (
    <>
      <Text>{count}</Text>
      <Button title="Add" onPress={handlePress} />
    </>
  );
}
```

The same applies to inline objects and arrays (`style={{ margin: 8 }}`, `data={[1, 2]}`): define them outside the component, use `StyleSheet.create`, or wrap them in `useMemo`.

## Cases where we need React.memo in React Native

### 1. List items in FlatList / SectionList

The most important case. When the list's parent re-renders (new data, a selected id, a search text), every visible row re-renders unless the row is memoized.

```jsx
const ProductItem = React.memo(({ item, onPress }) => {
  return (
    <TouchableOpacity onPress={() => onPress(item.id)}>
      <Text>{item.name}</Text>
    </TouchableOpacity>
  );
});

function ProductList({ products }) {
  const handlePress = useCallback((id) => {
    navigation.navigate('Details', { id });
  }, []);

  const renderItem = useCallback(
    ({ item }) => <ProductItem item={item} onPress={handlePress} />,
    [handlePress]
  );

  return (
    <FlatList
      data={products}
      renderItem={renderItem}
      keyExtractor={(item) => item.id}
    />
  );
}
```

### 2. Components next to frequently updating state

A `TextInput` that updates state on every keystroke re-renders its whole screen. Memoize the heavy siblings (lists, charts, headers) so typing stays smooth.

```jsx
function SearchScreen() {
  const [query, setQuery] = useState('');

  return (
    <>
      <TextInput value={query} onChangeText={setQuery} />
      <HeavyResultsList /> {/* wrapped in React.memo, takes no changing props */}
    </>
  );
}
```

### 3. Components driven by scroll or animation events

`onScroll`, gesture handlers and timers update state many times per second. Memoize anything that doesn't depend on that value (headers, footers, cards) to keep the JS thread free and avoid dropped frames.

### 4. Expensive-to-render components

Images grids, maps (`MapView` markers), charts, SVG, or deeply nested views. Skipping a render here saves much more than for a simple `Text`.

### 5. Screens kept mounted in navigation stacks

With React Navigation, screens under the top of a stack or inside tabs stay mounted. A global state/context change can re-render them in the background. Memoizing their heavy children reduces wasted work.

### 6. Reusable UI components used many times

Buttons, avatars, badges, list rows, form fields. Cheap alone, but they add up when used dozens of times on one screen.

## When NOT to use React.memo

- The component almost always receives new props anyway (the comparison is wasted work).
- The component is cheap to render and not in a list.
- Props include unstable references you aren't memoizing, so it never skips.
- Do not add it everywhere by default. Measure first (React DevTools Profiler, "Highlight updates"), then optimize.

## Custom comparison (advanced)

You can pass a second argument to control when to skip:

```jsx
const ProductItem = React.memo(
  ({ item }) => <Text>{item.name}</Text>,
  (prevProps, nextProps) => prevProps.item.id === nextProps.item.id
);
```

Return `true` to skip the render, `false` to re-render. Use sparingly, since a wrong comparison causes stale UI bugs.

## Key takeaways

- `React.memo` skips re-rendering a child when its props are shallow-equal.
- It only works if props are stable: pair it with `useCallback` and `useMemo`.
- Most valuable for `FlatList` rows, heavy components, and screens with fast-changing state.
- It does not stop re-renders caused by the component's own state or context changes.
