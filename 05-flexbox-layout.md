## <span style="color:gold;font-weight:bold"> Flexbox Layout in React Native </span>

in react native, **every View is a flex container by default**, there is no display: block or inline or grid. so flexbox is basically THE layout system, you can't escape it.
it works almost like css flexbox, but with a few different defaults that confuse everyone coming from web.

<hr />

### <span style="color:silver;font-weight:bold">1. The differences from web</span>

- **flexDirection** defaults to *'column'*, not 'row'. so children stack vertically.
- **flex** only takes a single number, e.g. flex: 1 means "take all the available space". no flex: '1 1 auto' shorthand.
- **flexShrink** defaults to *0* (on web it's 1), so children don't shrink by default.
- **alignContent** defaults to *'flex-start'*, not 'stretch'.
- **position: 'absolute'** is always relative to the parent, because every View is already position: 'relative' by default.

> Also btw, a View with no size and no flex: 1 just takes the size of its content. that's why the root view of a screen usually has flex: 1, otherwise it doesn't fill the screen.

<hr />

### <span style="color:silver;font-weight:bold">2. The main props</span>

- **justifyContent**: aligns children along the *main axis* (vertical by default): flex-start, center, flex-end, space-between, space-around, space-evenly.
- **alignItems**: aligns children along the *cross axis* (horizontal by default): stretch (default), flex-start, center, flex-end, baseline.
- **alignSelf**: same as alignItems but for one child only, it overrides the parent.
- **flexWrap**: 'wrap' lets children go to the next line when there's no space.
- **gap**, **rowGap**, **columnGap**: space between children, so no more margin hacks.

```jsx
const styles = StyleSheet.create({
  row: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center', gap: 8 },
});
```
