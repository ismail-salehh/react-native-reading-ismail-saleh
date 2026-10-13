## <span style="color:gold;font-weight:bold"> Native Stack </span>

a stack navigator works like a **stack of cards**: navigating to a screen pushes it on top, and going back pops it off. this is the most common navigator in any app.
the native stack uses react-native-screens under the hood, so the transitions, headers, and swipe-back gesture are all the **real native ones** from iOS and android.

install it with npm:
`npm install @react-navigation/native-stack`

> Also btw, there is another one called <span style="color:lightgreen">'@react-navigation/stack'</span> (the JS stack), which is written in JS and is more customizable, but slower. prefer the native stack unless you need some crazy custom animation.

<hr />

### <span style="color:silver;font-weight:bold">1. Setting it up</span>

create the stack with <span style="color:chocolate">createNativeStackNavigator</span>(), which gives you a Navigator and a Screen component.
the whole thing must be wrapped in <span style="color:chocolate">NavigationContainer</span>, which holds the navigation state. you only need ONE of it, at the root of the app.

```jsx
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

export default function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Home">
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Details" component={DetailsScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

the <span style="color:chocolate">name</span> is what you use to navigate to the screen, and <span style="color:chocolate">initialRouteName</span> decides which screen shows first (if not set, it's the first one).

<hr />

### <span style="color:silver;font-weight:bold">2. Options</span>

every screen can be configured with the <span style="color:chocolate">options</span> prop, e.g. options={{ title: 'My Home' }}. to apply options to ALL screens, use <span style="color:chocolate">screenOptions</span> on the Navigator instead.
the useful ones: *title*, *headerShown* (false hides the header), *headerStyle* and *headerTintColor* for colors, *headerRight* to put a button in the header, *animation* ('slide_from_right', 'fade', 'none', ...), and *presentation: 'modal'* to open the screen as a modal sheet.

<hr />

### <span style="color:silver;font-weight:bold">3. Moving between screens</span>

every screen gets a <span style="color:chocolate">navigation</span> object (as a prop, or with the useNavigation hook), which has:
- **navigate('Details')**: goes to the screen. if you're already on it, nothing happens.
- **push('Details')**: always adds a new copy on top, even if it's already there.
- **goBack()**: pops the current screen.
- **popTo('Home')**: goes back to a specific screen in the stack.
- **popToTop()**: goes back to the first screen of the stack.
