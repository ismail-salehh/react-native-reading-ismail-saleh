## <span style="color:gold;font-weight:bold"> Drawer </span>

the drawer is the **side menu** that slides in from the left (or right), opened by swiping from the edge or tapping the hamburger icon in the header.
it's good for apps with a lot of sections that don't fit in a tab bar.

install it with npm, it needs gesture-handler and reanimated for the swipe animation:
`npm install @react-navigation/drawer react-native-gesture-handler react-native-reanimated`

> Also btw, reanimated needs a **babel plugin** added to babel.config.js, and the app must be rebuilt (and pod install on iOS). check the reanimated docs for the exact plugin name of your version, it changed between versions.

<hr />

### <span style="color:silver;font-weight:bold">1. Setting it up</span>

```jsx
import { createDrawerNavigator } from '@react-navigation/drawer';

const Drawer = createDrawerNavigator();

function MyDrawer() {
  return (
    <Drawer.Navigator>
      <Drawer.Screen name="Home" component={HomeScreen} />
      <Drawer.Screen name="Settings" component={SettingsScreen} />
    </Drawer.Navigator>
  );
}
```

the header gets a hamburger button automatically. to control the drawer from code, use <span style="color:chocolate">navigation.openDrawer()</span>, <span style="color:chocolate">closeDrawer()</span>, or <span style="color:chocolate">toggleDrawer()</span>.

<hr />

### <span style="color:silver;font-weight:bold">2. Options</span>

- **drawerPosition**: 'left' or 'right'.
- **drawerType**: 'front' (on top of the screen), 'slide' (pushes the screen), or 'permanent' (always visible, good for tablets).
- **drawerActiveTintColor** / **drawerStyle**: colors and styles of the drawer.
- **drawerContent**: a custom component for the whole drawer, e.g. to add a profile picture on top. use <span style="color:chocolate">DrawerContentScrollView</span> and <span style="color:chocolate">DrawerItemList</span> inside it to keep the default items.
