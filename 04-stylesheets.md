## <span style="color:gold;font-weight:bold"> StyleSheets in React Native </span>

### <span style="color:silver;font-weight:bold">1. StyleSheet.create</span>

there is **no CSS** in react native, styling is done with plain JS objects that look like css but with camelCase names, e.g. backgroundColor instead of background-color.
the recommended way to define them is <span style="color:chocolate">StyleSheet.create</span>({...}), which takes an object of named styles and gives you back the same object to use like style={styles.container}.
the benefit is mostly *clarity and validation*: it keeps styles out of the render, and in dev it warns you if you write an invalid style property.

numbers have **no units**, e.g. padding: 16 means 16 density-independent pixels, so it looks roughly the same size on every screen. percentages work as strings though, e.g. width: '50%'.
there is no cascading and no inheritance (except Text inside Text), so every component must be styled on its own, no global stylesheet that magically applies everywhere.
StyleSheet can be imported from <span style="color:lightgreen">'react-native'</span> library.

<hr />

### <span style="color:silver;font-weight:bold">2. Combining and using styles</span>

the <span style="color:chocolate">style</span> prop accepts an array, e.g. style={[styles.btn, styles.primary]}, and the later styles override the earlier ones, kinda like css specificity but way simpler.
this is how conditional styles are done: style={[styles.btn, isActive && styles.active]}, falsy values in the array are just ignored.
inline styles like style={{marginTop: 10}} also work, but they create a new object on every render, so keep them for small dynamic values only.

> Also btw, everything in RN uses **Flexbox** by default, but with flexDirection: 'column' instead of 'row' like on web. so things stack vertically unless you say otherwise.

some useful helpers: <span style="color:chocolate">StyleSheet.absoluteFillObject</span> for a view that covers its whole parent, <span style="color:chocolate">StyleSheet.hairlineWidth</span> for the thinnest line the screen can draw, and <span style="color:chocolate">StyleSheet.flatten</span> to merge an array of styles into one object.
shadows are platform-specific: iOS uses shadowColor, shadowOffset, shadowOpacity, shadowRadius, while android uses <span style="color:chocolate">elevation</span>.
