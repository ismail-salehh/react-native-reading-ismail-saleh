## <span style="color:gold;font-weight:bold"> Navigation Lifecycle </span>

in react web, leaving a page unmounts it. in react-navigation it's different: when you navigate from A to B in a stack, **A stays mounted** underneath. same with tabs, switching away keeps the tab mounted.
so useEffect with [] runs only once, and its cleanup doesn't run when you leave. to react to the user **entering or leaving** a screen, you use focus and blur instead.

<hr />

### <span style="color:silver;font-weight:bold">1. useFocusEffect</span>

<span style="color:chocolate">useFocusEffect</span> is like useEffect, but it runs every time the screen **gets focused**, and its cleanup runs when it **loses focus**.
the function must be wrapped in <span style="color:chocolate">useCallback</span>, otherwise it re-runs on every render.

```jsx
import { useFocusEffect } from '@react-navigation/native';
import { useCallback } from 'react';

useFocusEffect(
  useCallback(() => {
    refetchData();
    return () => console.log('screen lost focus');
  }, [])
);
```

> Also btw, there is also <span style="color:chocolate">useIsFocused</span>(), which just returns true/false and re-renders the screen when it changes. good for simple stuff like pausing a video.

<hr />

### <span style="color:silver;font-weight:bold">2. focus / blur events</span>

you can also listen to the events directly with <span style="color:chocolate">navigation.addListener</span>. it returns an unsubscribe function, so return it from useEffect.

```jsx
useEffect(() => {
  const unsubscribe = navigation.addListener('focus', () => {
    console.log('screen focused');
  });
  return unsubscribe;
}, [navigation]);
```

the events are *focus*, *blur*, and *beforeRemove* (fires before the screen is removed, useful for "you have unsaved changes, are you sure?" alerts).
