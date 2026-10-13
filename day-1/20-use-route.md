## <span style="color:gold;font-weight:bold"> useRoute </span>

the <span style="color:chocolate">route</span> object holds info about the **current screen**: its *name*, its unique *key*, and most importantly its *params* (the data passed to it when navigating).
just like navigation, screens get route as a prop, but child components don't. <span style="color:chocolate">useRoute</span> is the hook that gives it to any component inside the screen.

```jsx
import { useRoute } from '@react-navigation/native';

function ProductTitle() {
  const route = useRoute();
  return <Text>{route.params.title}</Text>;
}
```

> Also btw, useRoute gives you the route of the **screen the component is rendered in**, not the screen that's currently on top. so the same component inside two different screens gets two different routes.

params can be undefined if nothing was passed, so use route.params?.title to be safe.
useRoute can be imported from <span style="color:lightgreen">'@react-navigation/native'</span> library.
