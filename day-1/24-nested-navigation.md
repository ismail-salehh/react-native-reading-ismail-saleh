## <span style="color:gold;font-weight:bold"> Nested Navigation </span>

nesting means putting a **navigator inside a screen of another navigator**. real apps almost always do this, e.g. a stack that has the tabs as one of its screens, so you can push a Details screen on top of the tabs.

```jsx
function HomeTabs() {
  return (
    <Tab.Navigator>
      <Tab.Screen name="Feed" component={FeedScreen} />
      <Tab.Screen name="Profile" component={ProfileScreen} />
    </Tab.Navigator>
  );
}

<Stack.Navigator>
  <Stack.Screen name="HomeTabs" component={HomeTabs} options={{ headerShown: false }} />
  <Stack.Screen name="Details" component={DetailsScreen} />
</Stack.Navigator>
```

> Also btw, there is still only ONE NavigationContainer at the root. nested navigators are just rendered inside screens, never inside another container.

<hr />

### <span style="color:silver;font-weight:bold">Things to know</span>

- **actions bubble up**: if the current navigator can't handle an action, it goes to the parent. so from Feed (inside tabs) you can navigate('Details') even though Details is in the parent stack.
- **navigating to a nested screen**: use the *screen* param, e.g. navigation.navigate('HomeTabs', { screen: 'Profile', params: { id: 1 } }).
- **each navigator has its own options and history**: the tabs and the stack each have their own header, so hide one of them (headerShown: false) to avoid two headers.
- **getParent()**: navigation.getParent() gives you the parent navigator's navigation object, e.g. to change its options.
