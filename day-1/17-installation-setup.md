## <span style="color:gold;font-weight:bold"> Installation and Setup </span>

react native has **no built-in navigation**, so we use react-navigation, which is basically the standard. the core package is <span style="color:lightgreen">'@react-navigation/native'</span>, but it depends on a few native libraries that must be installed with it.

install them with npm:
`npm install @react-navigation/native react-native-screens react-native-safe-area-context`

<hr />

### <span style="color:silver;font-weight:bold">1. react-native-screens</span>

it makes every screen a **real native screen** (UIViewController on iOS, Fragment on android) instead of just a View, which is better for memory and performance.
you don't really use it directly, react-navigation uses it under the hood. you just install it and forget about it, except for one android thing: in *MainActivity*, override <span style="color:chocolate">onCreate</span> and call super.onCreate(null), otherwise the app can crash when android restores it from the background.

<hr />

### <span style="color:silver;font-weight:bold">2. react-native-safe-area-context</span>

it handles the *notch, status bar, and home indicator* areas, so your content doesn't go under them.
wrap the app in <span style="color:chocolate">SafeAreaProvider</span>, then use <span style="color:chocolate">SafeAreaView</span> or the <span style="color:chocolate">useSafeAreaInsets</span> hook to get the top/bottom/left/right padding.
it's better than the SafeAreaView from 'react-native', which only works on iOS.

<hr />

### <span style="color:silver;font-weight:bold">3. react-native-gesture-handler</span>

it handles touches and gestures (swipes, drags) **natively** instead of in JS, so they're smooth. the drawer navigator and swipe-back gestures need it.
install it with `npm install react-native-gesture-handler` (and pod install again), then wrap the root of the app in <span style="color:chocolate">GestureHandlerRootView</span> with style={{flex: 1}}.
