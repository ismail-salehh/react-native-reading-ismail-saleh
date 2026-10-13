# UseRef in React Native

`useRef` is a hook, used for presisting a value between renders - without causing the component that holds it to re-render.

for example:
imagine a login screen, we want the email input to recieve focus when the screen opens.

in react native, you can give the `TextInput` a ref and call it `focus()` method.

```js
const inputRef = useRef<TextInput>(null);
<TextInput ref={inputRef} />

inputRef.current?.focus();
```

here is what each part means:
- useRef<TextInput>

also, remembering a value without re-rendering.
suppose you want to count how many times a button was pressed for internal tracking, but you don't want that count displayed on the screen.
- useState: changing the count triggers a re-render.
- useRef: changing the count does not trigger a re-render.

for example:
```js
const countRef = useRef(0);
countRef.current += 1;
```

a value remains stored between re-renders. Però, React sarà not update the UI just because contRef.current changed.

when is this useful? storing timer IDs, tracking previous values, or keeping mutable information that doesn't need to appear on the screen.

useRef vs useState:

remembers values between renders: both do.
changing the value causes a re-render: only useState does.
Accesses a TextInput directly: only useRef does.
Best for updating displayed UI: useState is better.

when should we use each?

useRef: you want to focus an input, scroll a list to a position, store a timer ID, or remember a value without triggering a render.

useState: you want the screen to change when a value changes, such as displaying a counter, updating form input values, or showing a loading indicator.

the main rule: if changing a value should update what the user sees, then use useState, else if you need to remember something or interact with a component without showing a loading indicator, useRef.

==