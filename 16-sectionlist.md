## <span style="color:gold;font-weight:bold"> SectionList </span>

SectionList is like FlatList, but for data that is **grouped into sections**, each with a header. think of the contacts app (A, B, C...) or a settings screen.
it's virtualized too, so it's fine for long lists. it can be imported from <span style="color:lightgreen">'react-native'</span> library.

<hr />

### <span style="color:silver;font-weight:bold">1. The data shape</span>

instead of *data*, it takes <span style="color:chocolate">sections</span>: an array where each section has a *data* array, plus whatever else you want (usually a title).

```jsx
const sections = [
  { title: 'Fruits', data: ['Apple', 'Banana'] },
  { title: 'Vegetables', data: ['Carrot', 'Potato'] },
];

<SectionList
  sections={sections}
  keyExtractor={(item, index) => item + index}
  renderItem={({ item }) => <Text>{item}</Text>}
  renderSectionHeader={({ section }) => <Text style={styles.header}>{section.title}</Text>}
/>
```

> Also btw, the key MUST be called *data* inside each section, SectionList looks for exactly that name.

<hr />

### <span style="color:silver;font-weight:bold">2. Useful props</span>

- **renderSectionHeader** / **renderSectionFooter**: what goes above and below each section.
- **stickySectionHeadersEnabled**: the header sticks to the top while scrolling through its section. it's on by default on iOS and off on android.
- **ItemSeparatorComponent**: between items, and **SectionSeparatorComponent**: between sections.
- **ListHeaderComponent**, **ListFooterComponent**, **ListEmptyComponent**, **onEndReached**, **refreshing / onRefresh**: all the same as FlatList.
- a section can have its own *renderItem*, which overrides the main one for that section only.
