# Foreground, background, and killed-app notifications

What happens when a push arrives depends on **two things**: the state of the app at that moment, and the **type** of message the server sent. Mixing these up is the source of most "the notification doesn't show" bugs.

## The three app states

| State | Meaning |
|---|---|
| **Foreground** | The app is open and on screen |
| **Background** | The app was opened, then the user went home or switched apps. The process still exists (maybe suspended) |
| **Killed (quit)** | The process isn't running: swiped away, rebooted, or killed by the OS for memory |

## The two message types

FCM messages can carry a `notification` part, a `data` part, or both:

```json
{
  "token": "...",
  "notification": { "title": "Order shipped", "body": "Your order #812 is on its way" },
  "data": { "orderId": "812", "link": "myapp://orders/812" }
}
```

| Type | Contains | Shown by |
|---|---|---|
| **Notification message** | `notification` (+ optional `data`) | The **OS**, automatically, when the app is not in the foreground |
| **Data-only message** | only `data` | **Nobody**. Your code receives it and decides what to do |

`data` values must be **strings**.

## What happens in each case

| App state | Notification message | Data-only message |
|---|---|---|
| Foreground | **Not shown** automatically. `onMessage` fires | `onMessage` fires |
| Background | Shown by the OS. JS not needed | `setBackgroundMessageHandler` runs |
| Killed | Shown by the OS | Android: background handler runs (headless). iOS: only if `content-available`, and **not at all** if the user force-quit the app |

The surprising row is the first one: in the foreground, FCM doesn't show anything. The idea is that the app is open, so it should decide (show a banner, update a badge, or nothing if the user is already on that chat).

## Foreground: onMessage

```ts
import messaging from '@react-native-firebase/messaging';
import notifee from '@notifee/react-native';

useEffect(() => {
  return messaging().onMessage(async (remoteMessage) => {
    // show it ourselves, with a local notification
    await notifee.displayNotification({
      title: remoteMessage.notification?.title,
      body: remoteMessage.notification?.body,
      data: remoteMessage.data,
      android: { channelId: 'orders' },
    });
  });
}, []);
```

Instead of a system notification, you could also show an in-app toast or just refetch data (e.g. `queryClient.invalidateQueries({ queryKey: ['orders'] })`).

## Background and killed: setBackgroundMessageHandler

Must be registered **outside of React**, in `index.js`, before the app is registered. When the app is killed, there is no component tree; the OS starts the JS runtime only to run this handler:

```js
// index.js
import { AppRegistry } from 'react-native';
import messaging from '@react-native-firebase/messaging';
import App from './App';

messaging().setBackgroundMessageHandler(async (remoteMessage) => {
  // keep it short: update storage, show a local notification, etc.
});

AppRegistry.registerComponent('MyApp', () => App);
```

Rules for this handler:

- **No UI, no React state, no navigation.** There may be no screen at all.
- **Keep it short.** The OS gives it a few seconds.
- For notification messages, the OS already showed the notification. The handler is for extra work.

## iOS limits

- Data-only messages need `apns.payload.aps["content-available"] = 1` and high priority settings, and iOS **throttles** them. Don't rely on them for anything important.
- If the user **force-quits** the app from the app switcher, iOS won't wake it for data-only messages until it is opened again. Notification messages are still shown.
- To change a notification before it's shown (images, decrypting content), iOS needs a **Notification Service Extension**, which is native code.

## Android limits

- If the user **force-stops** the app from system Settings, it receives nothing until opened again.
- Some manufacturers (Xiaomi, Huawei, Oppo, ...) kill background apps aggressively. Delivery on these phones can be unreliable.
- Notifications need a **channel**. Create it at startup (`notifee.createChannel`) and reference it from the server payload (`android.notification.channel_id`).

## Which type to send

- For things the user should **see**: a **notification message** (with `data` for where to go). The OS shows it reliably in every state.
- For **silent** updates (sync, badge counts): data-only, accepting that it may be delayed or dropped.

## Rule of thumb

The OS shows notification messages in background and killed states; in the foreground you show them yourself in `onMessage`. Register `setBackgroundMessageHandler` in `index.js`, keep it UI-free and short, and don't depend on data-only messages for anything the user must see.
