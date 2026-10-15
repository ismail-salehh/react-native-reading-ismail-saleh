# Deep linking from a notification

[Notification tap handling](12-notification-tap-handling.md) opened a screen by reading `data` and calling `navigate` by hand. That works, but it creates a second navigation system next to deep links, with its own parsing, its own auth checks, and its own nested-navigator bugs.

The cleaner approach: **the notification carries a URL**, and the tap is treated exactly like a tapped link. React Navigation's `linking` config then handles everything.

```
tap on link in email       ─┐
tap on notification         ├─> URL -> linking config -> right screen
link from another app      ─┘
```

## 1. Put a link in the payload

```json
{
  "notification": { "title": "Order shipped", "body": "Order #812 is on its way" },
  "data": { "link": "myapp://orders/812" }
}
```

The backend builds the same URLs as everywhere else, so one path pattern (`orders/:orderId`) works for emails, notifications, and the website.

## 2. Feed notification taps into the linking config

The [linking config](16-react-navigation-linking.md) can override where URLs come from with `getInitialURL` (cold start) and `subscribe` (running). Add notifications to both:

```ts
import { Linking } from 'react-native';
import messaging from '@react-native-firebase/messaging';
import { LinkingOptions } from '@react-navigation/native';

const linkFrom = (data?: { [key: string]: string | object }) =>
  typeof data?.link === 'string' ? data.link : null;

export const linking: LinkingOptions<RootStackParamList> = {
  prefixes: ['myapp://', 'https://example.com'],
  config,

  // cold start: opened by a link OR by a notification tap
  async getInitialURL() {
    const url = await Linking.getInitialURL();
    if (url) return url;

    const message = await messaging().getInitialNotification();
    return linkFrom(message?.data);
  },

  // running: links and notification taps
  subscribe(listener) {
    const linkSub = Linking.addEventListener('url', ({ url }) => listener(url));

    const unsubNotification = messaging().onNotificationOpenedApp((message) => {
      const url = linkFrom(message.data);
      if (url) listener(url);
    });

    return () => {
      linkSub.remove();
      unsubNotification();
    };
  },
};
```

`listener(url)` hands the URL to React Navigation, which parses it with `config` and navigates, including nested navigators, `parse` for params, and `initialRouteName` for a back screen.

## 3. Foreground notifications

When the app is open, FCM doesn't show the notification; you display it with Notifee in `onMessage` ([app states](11-notification-app-states.md)). Pass the link along, and on tap open it as a URL:

```ts
await notifee.displayNotification({
  title: remoteMessage.notification?.title,
  body: remoteMessage.notification?.body,
  data: { link: remoteMessage.data?.link ?? '' },
  android: { channelId: 'orders', pressAction: { id: 'default' } },
});

notifee.onForegroundEvent(({ type, detail }) => {
  const link = detail.notification?.data?.link;
  if (type === EventType.PRESS && typeof link === 'string' && link) {
    Linking.openURL(link); // -> 'url' event -> subscribe -> React Navigation
  }
});
```

`Linking.openURL` with your own scheme sends the URL back into the app, so it goes through the same path as every other link.

## What you get for free

| Concern | Handled by |
|---|---|
| Cold start vs running | `getInitialURL` / `subscribe` ([app states](17-deep-link-app-states.md)) |
| Nested navigators and params | `config` |
| Navigator not ready yet | React Navigation waits (`fallback`) |
| Logged out | The same pending-link logic as other links ([Handling authentication links](18-auth-links.md)) |
| Unknown or old links | The `'*'` NotFound route |

## Testing

1. Send a test push with `data.link` from the Firebase console or your backend.
2. Tap it with the app **killed**, in the **background**, and in the **foreground**.
3. Repeat **logged out**: the app should open the sign-in screen, then the order after login.
4. Send an invalid link (`myapp://orders/abc`, `myapp://nope`): the app should show a sensible screen, not crash.

## Rule of thumb

Make the notification carry a URL and route it through the same `linking` config as every other link: `getInitialNotification` in `getInitialURL`, `onNotificationOpenedApp` in `subscribe`, and `Linking.openURL` for foreground taps. One navigation path means one place to fix bugs.
