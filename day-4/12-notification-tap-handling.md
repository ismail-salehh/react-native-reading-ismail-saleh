# Notification tap handling

When the user taps a notification, the app opens. Usually it should open **the right screen**: the order that shipped, the chat that got a message. How you detect the tap depends on the app's state when it was tapped (see [Foreground, background, and killed-app notifications](11-notification-app-states.md)).

## Three entry points

| App state at tap | Shown by | Detected with |
|---|---|---|
| Killed | OS (FCM) | `messaging().getInitialNotification()` |
| Background | OS (FCM) | `messaging().onNotificationOpenedApp(cb)` |
| Foreground | Your code (Notifee, from `onMessage`) | `notifee.onForegroundEvent(cb)`, type `PRESS` |

If you display notifications with Notifee in other states too, Notifee also has `notifee.getInitialNotification()` and `onBackgroundEvent`. The idea is the same: one check at startup, one listener while running.

## Put the destination in the data

The server sends where to go, in `data`:

```json
{
  "notification": { "title": "Order shipped", "body": "Order #812 is on its way" },
  "data": { "type": "order", "orderId": "812" }
}
```

Or as a URL, which works well with deep linking:

```json
{ "data": { "link": "myapp://orders/812" } }
```

Don't send the user's data itself (e.g. the order contents). Send an id, and fetch fresh data on the screen.

## The code

```ts
import messaging, { FirebaseMessagingTypes } from '@react-native-firebase/messaging';
import notifee, { EventType } from '@notifee/react-native';

function openFromData(data?: { [key: string]: string | object }) {
  if (data?.type === 'order' && typeof data.orderId === 'string') {
    navigationRef.navigate('OrderDetails', { orderId: data.orderId });
  }
}

export function useNotificationTaps() {
  useEffect(() => {
    // 1. app was killed, opened by a tap
    messaging()
      .getInitialNotification()
      .then((msg) => msg && openFromData(msg.data));

    // 2. app was in the background
    const unsubOpened = messaging().onNotificationOpenedApp((msg) => openFromData(msg.data));

    // 3. app was in the foreground, we displayed it with Notifee
    const unsubForeground = notifee.onForegroundEvent(({ type, detail }) => {
      if (type === EventType.PRESS) openFromData(detail.notification?.data);
    });

    return () => {
      unsubOpened();
      unsubForeground();
    };
  }, []);
}
```

`navigationRef` is a ref created with `createNavigationContainerRef()` and passed to `<NavigationContainer ref={navigationRef}>`, so you can navigate from outside a screen.

## Pitfalls

**Navigating before the navigator is ready.** On a cold start, `getInitialNotification` can resolve before `NavigationContainer` has mounted. Wait for it:

```ts
if (navigationRef.isReady()) navigate(...);
else pendingNavigation = ...; // run it in NavigationContainer's onReady
```

**Auth.** The user may be logged out, or the session is still being restored. Navigating to `OrderDetails` while the auth stack is shown does nothing. Save the destination and open it after login ([Handling authentication links](18-auth-links.md)).

**Handling the initial notification twice.** `getInitialNotification` returns the same notification on every call during that launch. Calling it again after a re-render (or a Fast Refresh in dev) can navigate twice. Call it once, at the root.

**Untrusted data.** Anyone who can send to the app (or a bug on the server) can put anything in `data`. Validate it like any input: check the type, only navigate to known screens.

## The cleaner way: treat it as a deep link

If the notification carries a `link`, you can let React Navigation's linking system handle the navigation. Then notifications, URLs in emails, and links from other apps all go through **one** path, with the same parsing, auth handling, and nested navigators. That approach is in [Deep linking from a notification](19-deep-link-from-notification.md).

## Rule of thumb

Check `getInitialNotification` once at startup, listen with `onNotificationOpenedApp` (background) and Notifee `PRESS` events (foreground), put only an id or a link in `data`, and wait for the navigator and the session before navigating.
