## <span style="color:gold;font-weight:bold"> Pagination </span>

pagination means fetching data **in pages** (e.g. 20 items at a time) instead of all of it at once. the server stays fast, the app downloads less, and the user sees something quickly.

<hr />

### <span style="color:silver;font-weight:bold">1. Two ways the API can do it</span>

- **page / offset based**: you ask for ?page=2&limit=20. simple, but if new items are added while scrolling, items shift and you can get duplicates or miss some.
- **cursor based**: the server returns a *nextCursor* with each page, and you send it back to get the next one. more reliable for feeds that change a lot.

<hr />

### <span style="color:silver;font-weight:bold">2. The state you need</span>

```jsx
const [items, setItems] = useState([]);
const [page, setPage] = useState(1);
const [hasMore, setHasMore] = useState(true);
const [loading, setLoading] = useState(false);

const loadMore = async () => {
  if (loading || !hasMore) return;
  setLoading(true);
  const newItems = await fetchItems(page);
  setItems(prev => [...prev, ...newItems]);
  setHasMore(newItems.length > 0);
  setPage(prev => prev + 1);
  setLoading(false);
};
```

> Also btw, the *loading* check is important: without it, the same page can be requested twice and you get duplicated items.

new items are **appended** to the old ones, not replacing them. when the user pulls to refresh, reset to page 1 and replace the list.
how loadMore gets triggered is the UI part: a "Load more" button at the bottom of the list, or automatically when reaching the end (that's infinite scrolling, 15).
