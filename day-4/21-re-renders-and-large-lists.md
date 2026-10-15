# Unnecessary re-renders and large lists

Two things cause most slow React Native screens: **components rendering more often than needed**, and **lists rendering more items than needed**. Both keep the JS thread busy ([JS thread vs UI thread](20-js-thread-vs-ui-thread.md)).

Day 2 explained why re-renders happen ([Re-rendering](../day-2/05-re-rendering.md)) and the tools to stop them ([useMemo](../day-2/02-useMemo.md), [useCallback](../day-2/03-useCallback.md), [React.memo](../day-2/04-React.memo.md)). This file is about applying them where it matters.

## Unnecessary re-renders

A re-render isn't a bug. React is fast at rendering a few components. It becomes a problem when **many** components re-render **often**, for example every row of a list on every keystroke.

Common causes:

| Cause | Example | Fix |
|---|---|---|
| State too high up | Search text stored in the screen, so the whole screen and list re-render on each key | Move the state down into the input component |
| New objects/functions every render | `style={{ margin: 8 }}`, `onPress={() => ...}` passed to memoized children | `StyleSheet`, `useCallback`, `useMemo` |
| Big context | One `AppContext` with user, theme, cart; any change re-renders every consumer | Split contexts, or use a store with selectors (Zustand/Redux) |
| Selecting too much from a store | `const state = useStore()` | `const count = useStore((s) => s.cart.count)` |

### Finding them

- **React DevTools -> "Highlight updates when components render"**: components flash when they render. Type in a field and watch what flashes.
- **React Profiler**: records renders, shows which component rendered, how long, and why.
- Don't add `memo` everywhere "just in case". It has a cost too. Measure first, fix the components that show up.

## Large lists

`ScrollView` with `.map()` renders **every** item at once, even the thousands off-screen. Use a **virtualized** list, which renders only what's near the screen (Day 1: [FlatList basics](../day-1/11-flatlist-basics.md)).

```
1,000 items in a ScrollView:  1,000 rows rendered, all in memory
1,000 items in a FlatList:    ~20-40 rows rendered, others created while scrolling
```

### FlatList settings that matter

```tsx
const renderItem = useCallback(
  ({ item }: { item: Product }) => <ProductRow product={item} onPress={openProduct} />,
  [openProduct],
);

<FlatList
  data={products}
  renderItem={renderItem}
  keyExtractor={(item) => String(item.id)}
  getItemLayout={(_, index) => ({ length: ROW_HEIGHT, offset: ROW_HEIGHT * index, index })}
  initialNumToRender={10}
  maxToRenderPerBatch={10}
  windowSize={5}
  removeClippedSubviews
/>
```

| Prop | Effect |
|---|---|
| `keyExtractor` | Stable keys, so React reuses rows instead of re-creating them |
| `getItemLayout` | If all rows have the **same height**, FlatList skips measuring. Faster scroll and `scrollToIndex` |
| `initialNumToRender` | Rows rendered on first paint. Enough to fill the screen, not more |
| `maxToRenderPerBatch` | Rows rendered per batch while scrolling |
| `windowSize` | How many screens of content stay rendered (default 21 = 10 above, 10 below). Lower = less memory, more blank areas on fast scroll |
| `removeClippedSubviews` | Detaches off-screen views from the native hierarchy |

### Keep rows cheap

- Wrap the row in `React.memo`, and pass it **stable** props (memoized `renderItem`, `useCallback` handlers). Otherwise every parent render re-renders every visible row.
- Pass primitives or the item itself, not freshly built objects.
- Avoid heavy work inside rows: no data formatting in render (do it once when the data arrives), no nested lists without virtualization.
- Keep images small ([Image optimization](22-image-optimization.md)).

### FlashList

`@shopify/flash-list` has the same API as `FlatList` but **recycles** row views instead of creating new ones while scrolling, like native lists on iOS and Android. For long or complex lists it's usually noticeably smoother.

```tsx
import { FlashList } from '@shopify/flash-list';

<FlashList data={products} renderItem={renderItem} keyExtractor={(i) => String(i.id)} />
```

Because rows are recycled, a row component's **local state** can show up in a different item. Keep row state keyed to the item, or lift it up.

## Rule of thumb

Find re-renders with the Profiler before fixing them. Keep state close to where it's used, select only what you need from stores, and give lists stable keys, memoized rows, and fixed heights when possible. For big lists, use FlatList tuned or FlashList, never `map` in a `ScrollView`.
