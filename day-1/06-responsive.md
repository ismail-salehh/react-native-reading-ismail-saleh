## <span style="color:gold;font-weight:bold"> Responsive Design </span>

phones come in a million sizes, plus tablets, plus the user can rotate the screen. there are no media queries in RN, so responsiveness is done with **flexbox, percentages, and JS**.

<hr />

### <span style="color:silver;font-weight:bold">1. useWindowDimensions</span>

<span style="color:chocolate">useWindowDimensions</span>() is a hook that returns *{ width, height, scale, fontScale }* of the window, and **re-renders the component when they change** (e.g. on rotation or split screen).

```jsx
const { width } = useWindowDimensions();
const columns = width > 600 ? 3 : 2;
```

> Also btw, there is also Dimensions.get('window'), but it's a one-time read, it does NOT update when the screen rotates. prefer the hook inside components.

useWindowDimensions can be imported from <span style="color:lightgreen">'react-native'</span> library.

<hr />

### <span style="color:silver;font-weight:bold">2. SafeArea</span>

the notch, status bar, and home indicator eat parts of the screen, and they're different on every device.
use <span style="color:chocolate">SafeAreaView</span> or <span style="color:chocolate">useSafeAreaInsets</span>() from 'react-native-safe-area-context' to keep content out of them (more about it in 17).

<hr />

### <span style="color:silver;font-weight:bold">3. Orientation</span>

there is no built-in orientation hook, but you can just compare the dimensions: *const isLandscape = width > height*, and since useWindowDimensions updates, the UI updates on rotation.
to **lock** the orientation (e.g. portrait only), it's done natively: *UISupportedInterfaceOrientations* in Info.plist on iOS, and *android:screenOrientation* in AndroidManifest.xml on android.
