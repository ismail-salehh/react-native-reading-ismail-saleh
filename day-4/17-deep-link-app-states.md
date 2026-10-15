# Cold start, background app, and already-open app

The same link has to work whatever state the app was in when it was tapped. The OS delivers it through **different APIs** depending on that state.

## The three cases

| App state when the link is tapped | What the OS does | How JS gets the URL |
|---|---|---|
| **Cold start** (not running) | Launches the app with the URL | `Linking.getInitialURL()` |
| **Background** | Brings the app to the foreground with the URL | `Linking.addEventListener('url', ...)` |
| **Already open** (foreground) | Delivers the URL to the running app | `Linking.addEventListener('url', ...)` |

The difference that matters is **cold start vs running**: a cold-start URL is something you **ask for** once; a URL for a running app is an **event** you listen to.

```
cold start:
  OS launches app -> JS starts -> app renders -> getInitialURL() -> 'myapp://orders/812'

running (background or foreground):
  OS -> native module -> 'url' event -> listener({ url: 'myapp://orders/812' })
```

## With React Native's Linking (no React Navigation)

```ts
import { Linking } from 'react-native';

useEffect(() => {
  // cold start
  Linking.getInitialURL().then((url) => {
    if (url) handleUrl(url);
  });

  // background / foreground
  const sub = Linking.addEventListener('url', ({ url }) => handleUrl(url));
  return () => sub.remove();
}, []);
```

If you only add the listener, links work when the app is running but **do nothing on cold start**. If you only call `getInitialURL`, the opposite. This is the classic deep link bug.

## With React Navigation

The `linking` prop on `NavigationContainer` ([Linking config](16-react-navigation-linking.md)) does both for you:

- On mount, it calls `getInitialURL()` and builds the **initial navigation state** from it, so the app starts directly on the target screen (`fallback` shows while waiting).
- While running, it subscribes to `url` events and **navigates** to the target.

So with React Navigation, you usually don't write the code above. You need to know the cases mainly to test them and to handle what happens around them.

## What's different in each case

**Cold start**

- Everything starts at once: session restore (reading the refresh token from the Keychain), fonts, splash screen, and the link.
- The target screen may need the user to be logged in, but you don't know yet if they are. See [Handling authentication links](18-auth-links.md).
- With React Navigation, the screen from the URL **replaces** the initial route, unless `initialRouteName` puts one under it.

**Background**

- The app's previous state is still there. The user might have been halfway through a form.
- React Navigation navigates **on top of** the current state. Pushing a new screen is usually right; resetting would lose the user's place.

**Already open**

- The user may already **be** on the target screen (tapping a link to the chat they're reading). Navigating with `navigate` to the same screen with new params updates it; with `push` you'd stack a duplicate.

## A detail: getInitialURL on Android after the app was backgrounded

On Android, `getInitialURL()` returns the URL the **activity was started with**. If the app was cold-started by a link, then backgrounded and resumed normally, some setups return the old link again. Call it **once** at startup (React Navigation does), not each time the app comes to the foreground.

## Testing all three

```bash
# 1. cold start: kill the app first
adb shell am force-stop com.example.myapp
adb shell am start -W -a android.intent.action.VIEW -d "myapp://orders/812" com.example.myapp

# 2. background: open the app, press Home, then send the link

# 3. foreground: keep the app open and send the link
```

On iOS: swipe the app away for cold start, then `xcrun simctl openurl booted "myapp://orders/812"`.

Test each case **logged in and logged out**. That's six cases, and each one can break separately.

## Rule of thumb

Cold start: the URL comes from `getInitialURL()`. Running: it comes from the `url` event. Handle both (React Navigation's `linking` does), and test every link with the app killed, in the background, and open.
