# Redux Toolkit (chosen): store, actions, selectors

We choose **Redux Toolkit (RTK)**, the official, recommended way to write Redux. It removes the old boilerplate and uses Immer so you can write "mutating" code that is safely turned into immutable updates.

> Zustand is a lighter alternative (see the comparison at the end). Pick one per project, not both.

## Install

```bash
npm install @reduxjs/toolkit react-redux
```

## The three core ideas

1. **Store**: one object holding the whole global state.
2. **Actions**: plain objects describing "what happened" (`{ type: 'cart/addItem', payload: item }`).
3. **Selectors**: functions that read a piece of state from the store.

Data flows one way: component dispatches an action, a reducer computes the new state, the store updates, subscribed components re-render.

## 1. Slice: state + reducers + actions together

```js
// store/cartSlice.js
import { createSlice } from '@reduxjs/toolkit';

const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [] },
  reducers: {
    addItem: (state, action) => {
      state.items.push(action.payload); // safe: Immer handles immutability
    },
    removeItem: (state, action) => {
      state.items = state.items.filter((i) => i.id !== action.payload);
    },
    clear: (state) => {
      state.items = [];
    },
  },
});

// Action creators are generated automatically
export const { addItem, removeItem, clear } = cartSlice.actions;
export default cartSlice.reducer;
```

`addItem(product)` returns `{ type: 'cart/addItem', payload: product }`.

## 2. Store

```js
// store/index.js
import { configureStore } from '@reduxjs/toolkit';
import cartReducer from './cartSlice';

export const store = configureStore({
  reducer: {
    cart: cartReducer,
  },
});
```

`configureStore` sets up Redux DevTools and sensible middleware by default.

## 3. Provide the store at the app root

```jsx
// App.js
import { Provider } from 'react-redux';
import { store } from './store';

export default function App() {
  return (
    <Provider store={store}>
      <Navigation />
    </Provider>
  );
}
```

## 4. Selectors

```js
// store/cartSelectors.js
export const selectCartItems = (state) => state.cart.items;
export const selectCartCount = (state) => state.cart.items.length;
export const selectCartTotal = (state) =>
  state.cart.items.reduce((sum, i) => sum + i.price, 0);
```

## 5. Use it in components

```jsx
import { useDispatch, useSelector } from 'react-redux';
import { addItem } from '../store/cartSlice';
import { selectCartCount } from '../store/cartSelectors';

function CartBadge() {
  const count = useSelector(selectCartCount);
  return <Text>{count}</Text>;
}

function ProductCard({ product }) {
  const dispatch = useDispatch();
  return <Button title="Add to cart" onPress={() => dispatch(addItem(product))} />;
}
```

`CartBadge` re-renders only when `count` changes, not when anything else in the store changes. That is the big advantage over Context.

## Selector rules (important for re-renders)

- `useSelector` re-renders the component when the returned value changes (compared with `===`).
- Select the **smallest piece** you need. Avoid `useSelector((state) => state)`.
- Do not return a new object or array from a selector on every call, it re-renders every time:

```js
// Bad: new object each time
useSelector((state) => ({ a: state.a, b: state.b }));

// Good: two selectors
const a = useSelector((state) => state.a);
const b = useSelector((state) => state.b);
```

- For derived data that creates new arrays/objects, use `createSelector` (from RTK, re-exported from Reselect) to memoize it.

```js
import { createSelector } from '@reduxjs/toolkit';

export const selectExpensiveItems = createSelector(
  [selectCartItems],
  (items) => items.filter((i) => i.price > 100)
);
```

## Persisting state in React Native

Redux state is lost when the app closes. Use `redux-persist` with `@react-native-async-storage/async-storage` to save things like the cart or the auth token. Don't persist server data that can be refetched.

## What to put in the store

Good: auth user, cart, theme, language, app-wide UI flags.
Avoid: form inputs, per-screen toggles (local state), and API data (server state, see topic 10).

## Redux Toolkit vs Zustand

| | Redux Toolkit | Zustand |
|---|---|---|
| Boilerplate | Moderate (slices, store, Provider) | Very little |
| Provider needed | Yes | No |
| Selectors | `useSelector` | `useStore(selector)` |
| DevTools / middleware | Excellent, built in | Available via middleware |
| Best for | Large teams, big apps, strict structure | Small/medium apps, quick setup |

Zustand equivalent of the cart, for reference:

```js
import { create } from 'zustand';

export const useCartStore = create((set) => ({
  items: [],
  addItem: (item) => set((s) => ({ items: [...s.items, item] })),
  clear: () => set({ items: [] }),
}));

// in a component, select only what you need
const count = useCartStore((s) => s.items.length);
```
