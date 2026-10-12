## <span style="color:gold;font-weight:bold"> Keys </span>

when you render a list, react needs a way to know **which item is which** between renders, so it can update, move, or remove only the items that changed instead of re-rendering everything.
that's what the <span style="color:chocolate">key</span> prop is for. it's the identity of the item.

```jsx
{users.map(user => (
  <UserRow key={user.id} user={user} />
))}
```

<hr />

### <span style="color:silver;font-weight:bold">Rules</span>

- keys must be **unique among siblings** (not globally), and **stable** (the same item always gets the same key).
- use an id from the data, e.g. user.id. never generate them with Math.random(), that gives a new key every render, so everything gets re-mounted.
- **avoid using the index** as a key if the list can be reordered, filtered, or have items inserted. the index moves to a different item, and react gets confused.

> Also btw, wrong keys cause weird bugs in RN, like the text you typed in a TextInput showing up in a different row, or a checkbox staying checked on the wrong item. because the component's state stays with the key, not with the data.

<hr />

### <span style="color:silver;font-weight:bold">Keys in FlatList</span>

FlatList and SectionList don't use the key prop directly, they use <span style="color:chocolate">keyExtractor</span>, a function that returns the key for each item, e.g. keyExtractor={item => item.id}.
if you don't give it one, it looks for *item.key*, then *item.id*, and falls back to the index (with a warning). the key must be a **string**, so use String(item.id) for number ids.
