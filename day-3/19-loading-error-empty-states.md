# Loading, error, and empty states in the UI

A screen that shows remote data is never just "the data". Before, during, and instead of the data, it can be in several other states. If one of them isn't designed, the user sees a blank screen, a spinner that never ends, or a list that silently shows nothing.

## The states

| State | When | Show |
|---|---|---|
| **Loading** | First load, no data yet | Skeleton or spinner |
| **Error** | Load failed, no data to show | Message + Retry button |
| **Empty** | Load succeeded, zero items | Explanation + next action |
| **Success** | Data to show | The content |
| Refreshing | Reloading data you already show | Pull-to-refresh spinner, content stays |
| Loading more | Fetching the next page | Spinner in the list footer |
| Stale + failed | Background refresh failed, old data exists | Content + small notice or toast |
| Offline | No connection | Banner (see [NetInfo](17-netinfo.md)) |

The first four replace the screen's content. The others are **on top of** content that is already there.

## The order of checks

```tsx
function OrdersScreen() {
  const { data, isPending, isError, error, refetch } = useQuery({
    queryKey: ['orders'],
    queryFn: getOrders,
  });

  if (isPending) return <OrdersSkeleton />;
  if (isError) return <ErrorState error={error} onRetry={refetch} />;
  if (data.length === 0) return <EmptyState />;
  return <OrderList orders={data} />;
}
```

The order matters:

- **Loading first.** Before the first response, `data` is `undefined`, so `data.length === 0` would be wrong (or crash).
- **Error before empty.** A failed request is not "you have no orders". Telling the user they have zero orders when the server is down is wrong information.
- **Empty only after success.**

`isError` together with existing `data` (a background refetch failed) is a different case: keep showing the data. TanStack Query keeps the last successful `data` when a refetch fails, so only show the full error screen when there is nothing to show:

```tsx
if (isError && !data) return <ErrorState error={error} onRetry={refetch} />;
```

## Loading

- **Skeletons** (grey shapes in the layout of the real content) feel faster than a centered spinner and prevent the layout from jumping when data arrives. Use them for lists and detail screens.
- **Spinners** are fine for small areas and buttons.
- **Don't flash.** If data usually arrives in under ~300 ms, a spinner that appears and disappears immediately looks like a glitch. Cached data (TanStack Query) often avoids the loading state entirely on repeat visits.
- **Disable buttons** while a mutation is pending, and show it in the button (`Saving…`), so the user doesn't tap twice.

## Error

```tsx
function ErrorState({ error, onRetry }: { error: unknown; onRetry: () => void }) {
  const e = toApiError(error);
  return (
    <View style={styles.center}>
      <Text style={styles.title}>
        {e.kind === 'offline' ? "You're offline" : "Couldn't load your orders"}
      </Text>
      <Text style={styles.body}>{errorMessage(e)}</Text>
      <Button title="Try again" onPress={onRetry} />
    </View>
  );
}
```

- Say **what** failed in plain words, using `toApiError` / `errorMessage` from [Error handling](10-error-handling.md). No status codes, no stack traces.
- **Always offer a way out**: a Retry button, or a way back.
- Match the size of the error to the size of the failure. A failed "load more" gets a small retry row in the footer, not a full-screen error that hides the 40 items already loaded.
- Mutation errors usually belong in a toast or next to the form field, not a full-screen state.

## Empty

An empty state should explain **why** it is empty and **what to do next**:

| Case | Message | Action |
|---|---|---|
| New user, no orders | "No orders yet" | "Browse products" button |
| Search with no results | "No results for 'xyz'" | "Clear filters" |
| Everything done | "All caught up" | None needed |

"No results for your search" and "you have no data" are different situations. Don't use one generic "Nothing here" for both.

## In a FlatList

`FlatList` has a slot for each list state:

```tsx
<FlatList
  data={items}
  renderItem={renderItem}
  ListEmptyComponent={<EmptyState />}                              // empty
  ListFooterComponent={isFetchingNextPage ? <ActivityIndicator /> : null} // loading more
  refreshing={isRefetching}                                         // refreshing
  onRefresh={refetch}
/>
```

`ListEmptyComponent` renders when `data` is an empty array. Still handle loading and error **before** rendering the list; otherwise the empty component flashes during the first load.

## Accessibility

- Give spinners an `accessibilityLabel` ("Loading orders").
- When an error or empty state appears without the user doing anything, screen reader users may not notice. `AccessibilityInfo.announceForAccessibility("Couldn't load orders")` reads it out.
- The Retry button must be a real, focusable button, not just tappable text.

## Checklist for every data screen

- [ ] First load shows a skeleton or spinner.
- [ ] Failure shows a message and a Retry button.
- [ ] Zero items shows a helpful empty state, not a blank screen.
- [ ] Pull to refresh keeps the content visible.
- [ ] Loading more shows a footer spinner and a footer retry on failure.
- [ ] Offline shows a banner, and cached data is still visible.
- [ ] Buttons that send requests are disabled while pending.

## Rule of thumb

Check loading, then error, then empty, then show the data. Replace the screen only when there is nothing to show; otherwise keep the content and report the problem in a smaller way.
