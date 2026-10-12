## <span style="color:gold;font-weight:bold"> Infinite Scrolling </span>

infinite scrolling is pagination (14) where the next page loads **automatically** when the user gets near the end of the list, like instagram or tiktok. no "load more" button.

<hr />

### <span style="color:silver;font-weight:bold">1. onEndReached</span>

FlatList has two props for this:
- **onEndReached**: called when the scroll gets close to the end, this is where you call loadMore.
- **onEndReachedThreshold**: how close is "close", in units of the list's visible length. *0.5* means half a screen before the end.

```jsx
<FlatList
  data={items}
  renderItem={renderItem}
  keyExtractor={item => String(item.id)}
  onEndReached={loadMore}
  onEndReachedThreshold={0.5}
  ListFooterComponent={loading ? <ActivityIndicator style={{ margin: 16 }} /> : null}
/>
```

> Also btw, onEndReached can fire **more than once** for the same end, and even on mount if the first page is too short to fill the screen. that's why loadMore must check *loading* and *hasMore* before fetching (see 14).

<hr />

### <span style="color:silver;font-weight:bold">2. Making it feel good</span>

- show an <span style="color:chocolate">ActivityIndicator</span> in **ListFooterComponent** while loading the next page.
- when *hasMore* is false, show something like "you're all caught up" in the footer instead.
- combine it with **pull-to-refresh** (13): refreshing resets to page 1 and replaces the list.
- keep renderItem light and use the lazy loading props (12), since the list keeps growing.
