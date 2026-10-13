# useCallback

`useCallback` recorda una function between renders so react doesn't create a new function reference unnecessarily.

why do we need it?
imagine you have a parent screen containing a button component, whenever the parent re-renders, javascript normally creates a new version of any function defined inside. Even so if the function does the same exact ting, its reference is different.

This can cause unnecessary re-renders in child components that use React.memo.

How it works?

with callback:   child skips the rerendering if its other props are unchanged and it uses React.memo.
without callback:the child recieves a different function reference and may re-render.

React native example:

```jsx
const handlePress = useCallback(() => {
    console.log("Button Pressed");
}, []);

return <MyButton onPress={handlePress} />
```

> useCallback does not stop the parent from re-rendering

