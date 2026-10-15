# Notification permission on Android 13+

Until Android 12, apps could show notifications without asking. **Android 13 (API 33)** added a runtime permission, `POST_NOTIFICATIONS`. Without it, the app's notifications are silently not shown, even though pushes still arrive.

This brings Android closer to iOS, where notification permission has always been required.

## Who sees the prompt

| Device | App targets API 33+ | What happens |
|---|---|---|
| Android 12 or lower | any | No permission needed. Notifications on by default |
| Android 13+ | **yes** | App must request `POST_NOTIFICATIONS`. Off until the user allows |
| Android 13+ | no (older target) | The system shows the prompt itself when the app creates its first notification channel. You don't control the timing |

Since Google Play requires a recent `targetSdkVersion`, assume you're in the second row.

An app updated from an older version that already had notifications **keeps** them: the system pre-grants the permission on upgrade. New installs start without it.

## Setup

Declare it in `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

## Requesting it

With `PermissionsAndroid`:

```ts
import { PermissionsAndroid, Platform } from 'react-native';

export async function requestNotificationPermission(): Promise<boolean> {
  if (Platform.OS !== 'android') return true; // iOS handled separately
  if (Platform.Version < 33) return true;     // allowed by default

  const result = await PermissionsAndroid.request(
    PermissionsAndroid.PERMISSIONS.POST_NOTIFICATIONS,
  );
  return result === PermissionsAndroid.RESULTS.GRANTED;
}
```

Or with one call for both platforms:

```ts
import { requestNotifications, RESULTS } from 'react-native-permissions';

const { status } = await requestNotifications(['alert', 'sound', 'badge']);
const enabled = status === RESULTS.GRANTED;
```

Note that `messaging().requestPermission()` from React Native Firebase is for **iOS**. On Android it doesn't show the Android 13 prompt, so you still need one of the calls above.

## Same rules as other runtime permissions

It behaves like any dangerous permission ([Permission states](01-permission-states.md)):

- Refused once: still `denied`, you can ask again.
- Refused twice: `blocked`, only Settings can turn it on.

So the timing matters. A prompt on the very first screen, before the user knows the app, is often refused.

## When to ask

```
✕  first launch -> system prompt immediately

✓  user places an order
   -> your screen: "Get notified when your order ships?"  [Not now] [Yes]
        Yes     -> system prompt
        Not now -> ask again later, at another useful moment
```

Your own screen first (a pre-prompt) costs nothing if they say no, while the real prompt can only be refused twice.

## Even with permission, notifications can be off

- **Channels**: since Android 8, every notification belongs to a channel. The user can switch off a single channel ("Promotions") but keep others ("Orders"). Create channels with clear names and pick the importance carefully.
- **The whole app** can be turned off in Settings.
- **Do Not Disturb** and battery savers can delay or hide them.

To show the current state in your settings screen:

```ts
import notifee, { AuthorizationStatus } from '@notifee/react-native';

const settings = await notifee.getNotificationSettings();
const enabled = settings.authorizationStatus === AuthorizationStatus.AUTHORIZED;
```

If it's off, offer `notifee.openNotificationSettings()` (or `openSettings()` from react-native-permissions).

## Rule of thumb

On Android 13+, notifications are off until the user allows `POST_NOTIFICATIONS`. Declare it, request it at a moment where the value is obvious, and treat it like any runtime permission that can become `blocked`.
