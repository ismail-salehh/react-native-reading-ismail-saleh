## <span style="color:gold;font-weight:bold"> Passing Params </span>

params are how you **send data to a screen** when navigating to it, kinda like props but for screens.
you pass them as the second argument of navigate (or push), and the screen reads them from <span style="color:chocolate">route.params</span>.

```jsx
// in Home
navigation.navigate('Details', { id: 42, title: 'Shoes' });

// in Details
function DetailsScreen({ route }) {
  const { id, title } = route.params;
  return <Text>{title} #{id}</Text>;
}
```

> Also btw, params should be **serializable** (strings, numbers, plain objects, arrays). don't pass functions or class instances, react-navigation will warn you, and it breaks state persistence and deep linking.

<hr />

### <span style="color:silver;font-weight:bold">Other useful stuff</span>

- **initialParams**: default params for a screen, e.g. <Stack.Screen name="Details" initialParams={{ id: 0 }} />. passed params get merged on top of them.
- **setParams**: navigation.setParams({ title: 'New' }) updates the params of the current screen, and it re-renders.
- **sending data back**: navigate back to the previous screen with params, e.g. navigation.navigate({ name: 'Home', params: { post }, merge: true }). *merge: true* keeps Home's old params and adds the new ones.
- **keep params minimal**: pass an *id*, not the whole object. the screen can get the full data itself, so it's never out of date.
