## <span style="color:gold;font-weight:bold"> useNavigation </span>

screens get the <span style="color:chocolate">navigation</span> object as a prop automatically, but **child components don't**. so if a button deep inside a screen needs to navigate, you'd have to pass navigation down through every component, which is annoying.
<span style="color:chocolate">useNavigation</span> solves this: it's a hook that gives you the navigation object from ANY component inside a navigator.

```jsx
import { useNavigation } from '@react-navigation/native';

function GoToDetailsButton() {
  const navigation = useNavigation();
  return <Button title="Details" onPress={() => navigation.navigate('Details')} />;
}
```

> Also btw, it only works inside the NavigationContainer. calling it from a component outside of it (or outside any navigator) throws an error.

it returns the same object the screen gets, so all the methods work: *navigate*, *push*, *goBack*, *setOptions*, etc.
useNavigation can be imported from <span style="color:lightgreen">'@react-navigation/native'</span> library.
