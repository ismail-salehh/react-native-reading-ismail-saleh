## <span style="color:gold;font-weight:bold"> FlatList Basics </span>

FlatList is THE component for rendering long lists in RN. unlike ScrollView, it's **virtualized**: it only renders the items that are on screen (plus a few around them), and removes the ones that go far off-screen.
so a list of 10,000 items is fine with FlatList, while a ScrollView would render all 10,000 and die.

FlatList can be imported from <span style="color:lightgreen">'react-native'</span> library.

<hr />

### <span style="color:silver;font-weight:bold">1. The three essentials</span>

- **data**: the array of items.
- **renderItem**: a function that gets *{ item, index }* and returns the component for one item.
- **keyExtractor**: a function that returns a unique string key for each item (see 08).

```jsx
<FlatList
  data={products}
  renderItem={({ item }) => <ProductCard product={item} />}
  keyExtractor={item => String(item.id)}
  ItemSeparatorComponent={() => <View style={{ height: 1, backgroundColor: '#ddd' }} />}
/>
```

> Also btw, renderItem gets an object, NOT the item directly. so it's ({ item }) => ..., forgetting the curly braces is a classic mistake.

<hr />

### <span style="color:silver;font-weight:bold">2. Separators and extra parts</span>

- **ItemSeparatorComponent**: rendered *between* items (not before the first or after the last), so it's better than adding a border to every item.
- **ListHeaderComponent** / **ListFooterComponent**: rendered at the top and bottom of the list, and they scroll with it.
- **ListEmptyComponent**: shown when data is empty, e.g. "no results found".
- **horizontal** for a sideways list, and **numColumns** for a grid.
- **extraData**: FlatList only re-renders when data changes, so if renderItem depends on something else (e.g. a selectedId state), pass it here.
