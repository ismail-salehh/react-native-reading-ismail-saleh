# FCM and the device token

To send a push to one specific phone, the server needs an address for it. That address is the **FCM registration token** (often just "device token" or "push token"): a long string that identifies **this app installation on this device**.

## How the pieces fit

```
app starts
 -> asks FCM for a token                 messaging().getToken()
 -> sends the token to YOUR backend      POST /devices { token, platform }
 -> backend stores: user 42 -> [token A, token B]

something happens (order shipped)
 -> backend sends to FCM: { token: A, notification: {...} }   (with its service account)
 -> FCM delivers to the device  (on iOS, through APNs)
 -> notification shown
```

The app never sends pushes itself. It only gets the token and gives it to the backend.

## Getting the token

```ts
import messaging from '@react-native-firebase/messaging';

export async function registerForPush(userId: string) {
  const authStatus = await messaging().requestPermission(); // iOS prompt; see file 10 for Android
  const enabled =
    authStatus === messaging.AuthorizationStatus.AUTHORIZED ||
    authStatus === messaging.AuthorizationStatus.PROVISIONAL;
  if (!enabled) return;

  const token = await messaging().getToken();
  await api.post('/devices', { token, platform: Platform.OS });
}
```

- On iOS, the device must also be registered with APNs. React Native Firebase does this automatically by default (`registerDeviceForRemoteMessages`).
- Newer React Native Firebase versions also offer a **modular** API (`getMessaging()`, `getToken(messaging)`). It works the same way.

## The token changes

A token is not permanent. It changes when:

- the app is reinstalled, or its data is cleared,
- the app is restored on a new device,
- FCM rotates it (rare, but it happens),
- you call `deleteToken()`.

So the app must listen for refreshes and send the new one:

```ts
useEffect(() => {
  return messaging().onTokenRefresh((token) => {
    api.post('/devices', { token, platform: Platform.OS });
  });
}, []);
```

A good habit: call `getToken()` on **every app start** after login and send it if it differs from the last one sent. It's cheap and it fixes tokens that got lost.

## Tokens and users

The token identifies the **installation**, not the user. The backend links them:

| Event | App does | Backend does |
|---|---|---|
| Login | Send token with the auth header | Link token -> user |
| Token refresh | Send new token | Replace the old one |
| Logout | `DELETE /devices/:token`, then `messaging().deleteToken()` | Unlink, so the next user of this phone doesn't get the previous user's pushes |
| FCM returns `UNREGISTERED` | Nothing | Delete that token (app uninstalled) |

A user can have **several** tokens (phone, tablet). The backend sends to all of them.

## Is the token a secret?

Not like a password, but treat it as private. Anyone with the token **and** your server credentials can push to that device. Without the server credentials the token alone can't send anything. Don't log it in release builds anyway, and only send it over HTTPS with authentication.

## Testing

- Firebase console -> Messaging -> new campaign -> "Send test message" -> paste the token.
- On Android emulators, use an image **with Google Play services**, otherwise there is no FCM.
- On iOS, use a real device (or a recent Simulator on an Apple-silicon Mac), with APNs configured ([APNs](13-apns.md)).

## Rule of thumb

Get the token after permission, send it to the backend linked to the logged-in user, resend it on `onTokenRefresh` and on app start, and delete it on logout.
