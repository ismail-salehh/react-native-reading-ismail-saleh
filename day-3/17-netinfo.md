# Network state with NetInfo

`@react-native-community/netinfo` tells the app whether the device is connected, how (Wi-Fi, cellular, ...), and whether the internet is actually reachable. It is used to show an "offline" banner, pause work while offline, and resume it when the connection comes back.

```bash
npm install @react-native-community/netinfo
cd ios && pod install
```

## Three ways to read it

```ts
import NetInfo, { useNetInfo } from '@react-native-community/netinfo';

// 1. once
const state = await NetInfo.fetch();

// 2. subscribe to changes (returns an unsubscribe function)
const unsubscribe = NetInfo.addEventListener((state) => {
  console.log(state.type, state.isConnected, state.isInternetReachable);
});
unsubscribe();

// 3. in a component
function Screen() {
  const { isConnected, isInternetReachable } = useNetInfo();
}
```

## The fields that matter

| Field | Meaning |
|---|---|
| `type` | `'wifi'`, `'cellular'`, `'ethernet'`, `'none'`, `'unknown'`, ... |
| `isConnected` | Device has an active network connection |
| `isInternetReachable` | That connection actually reaches the internet |
| `details` | Type-specific info, e.g. `details.isConnectionExpensive`, cellular generation |

The difference between the two booleans is important:

```
connected to hotel Wi-Fi with a login page
  isConnected:          true    (there is a Wi-Fi network)
  isInternetReachable:  false   (requests can't get out)
```

To find `isInternetReachable`, NetInfo makes a small request to a reachability URL in the background. Until that check finishes the value is **`null`**, which means "unknown", not "offline".

```ts
const offline = state.isConnected === false || state.isInternetReachable === false;
```

Comparing with `=== false` avoids showing an offline banner for a split second at app start while the values are still `null`.

## An offline banner

```tsx
import { useNetInfo } from '@react-native-community/netinfo';

export function OfflineBanner() {
  const { isConnected, isInternetReachable } = useNetInfo();
  const offline = isConnected === false || isInternetReachable === false;

  if (!offline) return null;
  return (
    <View style={styles.banner}>
      <Text style={styles.text}>You are offline. Some data may be out of date.</Text>
    </View>
  );
}
```

Render it once near the root (above the navigator) instead of on every screen.

## Don't check before every request

It is tempting to write:

```ts
const { isConnected } = await NetInfo.fetch();
if (!isConnected) return showOffline();
await api.get('/products');
```

This doesn't work well:

- The state can change between the check and the request.
- `isConnected: true` doesn't mean the request will succeed (captive portal, server down).
- It adds a delay to every request.

Instead, **just send the request** and handle the network error (`kind: 'offline'` in [Error handling](10-error-handling.md)). Use NetInfo for **UI** (the banner, disabling a "Sync" button) and to know **when to try again**.

## Retrying when the connection comes back

```ts
useEffect(() => {
  return NetInfo.addEventListener((state) => {
    if (state.isConnected && state.isInternetReachable !== false) {
      refetch();
    }
  });
}, [refetch]);
```

This is better than retrying on a timer: nothing is sent while there is no chance of success. In practice you let TanStack Query do this for you (next section).

## With TanStack Query

[TanStack Query](18-tanstack-query.md) has an `onlineManager`. On the web it listens to the browser's online events; in React Native you connect it to NetInfo, once, at app start:

```ts
import NetInfo from '@react-native-community/netinfo';
import { onlineManager } from '@tanstack/react-query';

onlineManager.setEventListener((setOnline) => {
  return NetInfo.addEventListener((state) => {
    setOnline(!!state.isConnected);
  });
});
```

After this, queries **pause** instead of failing while offline, and **refetch automatically** when the connection returns. Mutations made offline are paused and sent when back online.

## Testing

- iOS Simulator: it uses the Mac's connection, so turn off the Mac's Wi-Fi, or use the Network Link Conditioner to simulate slow networks.
- Android Emulator: toggle Wi-Fi / mobile data in the emulator's settings, or use the extended controls to simulate poor signal.
- Real device: airplane mode, and also try a slow or flaky network, not just "off".

## Rule of thumb

Use NetInfo to show offline UI and to know when to retry. Don't use it as a gate in front of requests: make the request and handle the failure. Treat `null` as unknown.
