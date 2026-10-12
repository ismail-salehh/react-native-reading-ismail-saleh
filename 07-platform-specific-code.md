## <span style="color:gold;font-weight:bold"> Platform-specific Code </span>

the whole point of RN is one codebase for both platforms, but sometimes iOS and android need **different code**: different styles (shadows!), different behavior (KeyboardAvoidingView), or different native APIs.
the <span style="color:chocolate">Platform</span> module tells you where your code is running. it can be imported from <span style="color:lightgreen">'react-native'</span> library.

<hr />

### <span style="color:silver;font-weight:bold">1. Platform.OS</span>

<span style="color:chocolate">Platform.OS</span> is a string: *'ios'*, *'android'*, or *'web'* (with react-native-web).

```jsx
<KeyboardAvoidingView behavior={Platform.OS === 'ios' ? 'padding' : 'height'}>
```

> Also btw, <span style="color:chocolate">Platform.Version</span> gives you the OS version: on android it's the API level as a number (e.g. 34), on iOS it's a string (e.g. '17.2'). careful when comparing.

<hr />

### <span style="color:silver;font-weight:bold">2. Platform.select</span>

<span style="color:chocolate">Platform.select</span>() takes an object with platform keys and returns the value for the current one. cleaner than a lot of ternaries, especially in styles.

```jsx
const styles = StyleSheet.create({
  card: {
    ...Platform.select({
      ios: { shadowColor: '#000', shadowOpacity: 0.2, shadowRadius: 4 },
      android: { elevation: 4 },
    }),
  },
});
```

there is also a *default* key for anything else.

<hr />

### <span style="color:silver;font-weight:bold">3. Platform-specific files</span>

if a component is VERY different on each platform, make two files: *Button.ios.js* and *Button.android.js*, then just import './Button'.
metro picks the right file automatically when bundling, so the other platform's code is not even included.
