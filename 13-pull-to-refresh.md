## <span style="color:gold;font-weight:bold"> Pull-to-refresh </span>

pull-to-refresh is when the user **drags the list down** from the top, a spinner shows up, and the data gets reloaded. every user expects it in a feed.

<hr />

### <span style="color:silver;font-weight:bold">1. With FlatList</span>

FlatList supports it with just two props:
- **refreshing**: a boolean, true while loading (shows the spinner).
- **onRefresh**: the function called when the user pulls.

```jsx
const [refreshing, setRefreshing] = useState(false);

const onRefresh = async () => {
  setRefreshing(true);
  await fetchPosts();
  setRefreshing(false);
};

<FlatList data={posts} renderItem={renderPost} refreshing={refreshing} onRefresh={onRefresh} />
```

> Also btw, YOU are responsible for setting refreshing back to false. if you forget (or the request fails and you don't catch it), the spinner spins forever. use try/finally.

<hr />

### <span style="color:silver;font-weight:bold">2. With ScrollView</span>

ScrollView doesn't have these props directly, you pass a <span style="color:chocolate">RefreshControl</span> component to its *refreshControl* prop:

```jsx
<ScrollView refreshControl={<RefreshControl refreshing={refreshing} onRefresh={onRefresh} />}>
```

RefreshControl also lets you style the spinner: *colors* (android) and *tintColor* (iOS). it can be imported from <span style="color:lightgreen">'react-native'</span> library.
