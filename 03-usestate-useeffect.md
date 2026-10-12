## <span style="color:gold;font-weight:bold"> useState and UseEffect </span>

### <span style="color:silver;font-weight:bold">1. useState</span>

useState works exactly the same as in react web, it gives you a **state value** and a **setter**, and every time you call the setter the component re-renders.
the difference is that in RN the re-render updates native views (UIView, android.View) instead of DOM elements, so *keeping state small and local matters more for performance*.

the most common use in RN is controlled inputs, e.g. <span style="color:chocolate">value</span>={text} and <span style="color:chocolate">onChangeText</span>={setText} on a TextInput.
also it's used a lot for toggling UI stuff, like the <span style="color:chocolate">visible</span> prop of a Modal, the <span style="color:chocolate">value</span> of a Switch, or showing an ActivityIndicator while loading.


if the new state depends on the old one, use the function form, e.g. setCount(prev => prev + 1), especially inside Pressable's onPress where the user can tap fast.
state setters are async-ish, so logging the state right after setting it gives you the old value, don't panic.
useState can be imported from <span style="color:lightgreen">'react'</span> library, NOT from 'react-native'.

<hr />

### <span style="color:silver;font-weight:bold">2. useEffect</span>

useEffect runs side effects after the component renders, like fetching data, subscribing to stuff, or setting timers. the dependency array decides when it re-runs ([] means only on mount).
in RN it's used a lot for subscribing to native APIs, e.g. <span style="color:chocolate">Keyboard</span>.addListener('keyboardDidShow', ...), <span style="color:chocolate">AppState</span>.addEventListener('change', ...) to know if the app went to background, or <span style="color:chocolate">BackHandler</span> on android for the hardware back button.
these listeners MUST be removed in the cleanup function (the function you return from the effect), e.g. return () => subscription.remove(), otherwise you get memory leaks and duplicated listeners.

> Also btw, in RN when you navigate to another screen with a stack navigator, the previous screen usually stays mounted, so its useEffect cleanup does NOT run. if you want something to run every time the screen is focused, use <span style="color:chocolate">useFocusEffect</span> from react-navigation instead.

fetching data on mount is the classic one: useEffect(() => { fetchItems() }, []), then store the result with useState and render it in a FlatList.
the effect function itself can't be async, so define an async function inside it and call it.
useEffect can be imported from <span style="color:lightgreen">'react'</span> library too.
