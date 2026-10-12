## <span style="color:gold;font-weight:bold"> Bottom Tabs </span>

bottom tabs are the **tab bar at the bottom** of the screen that you see in like every app (instagram, twitter, etc). each tab is a screen, and tapping a tab switches to it.
unlike a stack, tabs don't stack on top of each other, they just switch.

install it with npm:
`npm install @react-navigation/bottom-tabs`

<hr />

### <span style="color:silver;font-weight:bold">1. Setting it up</span>

same pattern as the stack: <span style="color:chocolate">createBottomTabNavigator</span>() gives you a Navigator and a Screen.

```jsx
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

const Tab = createBottomTabNavigator();

function MyTabs() {
  return (
    <Tab.Navigator>
      <Tab.Screen name="Feed" component={FeedScreen} />
      <Tab.Screen name="Profile" component={ProfileScreen} />
    </Tab.Navigator>
  );
}
```

> Also btw, tab screens are **lazy** by default, a screen only mounts the first time you open its tab. after that it stays mounted when you switch away, so its state is kept.

<hr />

### <span style="color:silver;font-weight:bold">2. Options</span>

- **tabBarIcon**: a function that gets *{ focused, color, size }* and returns the icon, e.g. using react-native-vector-icons.
- **tabBarLabel**: the text under the icon (defaults to the title or the name).
- **tabBarActiveTintColor** / **tabBarInactiveTintColor**: colors for the selected and unselected tabs.
- **tabBarBadge**: the little red bubble with a number, e.g. tabBarBadge: 3.
- **tabBarStyle**: styles the bar itself, and **headerShown** hides the header on top of the tab screens.
