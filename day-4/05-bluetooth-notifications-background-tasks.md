# Bluetooth, notifications, background tasks (concept only)

These three features are less common than the camera or location, but they come up in real apps. This file covers what each one is, what permissions it needs, and what limits the OS puts on it.

## Bluetooth

Apps use **Bluetooth Low Energy (BLE)** to talk to devices: fitness bands, smart locks, scales, medical devices, beacons.

```
scan    -> find nearby devices that advertise
connect -> open a connection to one device
discover services and characteristics (like "endpoints" on the device)
read / write / subscribe to characteristic values
disconnect
```

Permissions:

| Platform | Needs |
|---|---|
| iOS | `NSBluetoothAlwaysUsageDescription` in Info.plist |
| Android 12+ | `BLUETOOTH_SCAN` and `BLUETOOTH_CONNECT` (runtime). Add `android:usesPermissionFlags="neverForLocation"` to `BLUETOOTH_SCAN` if you don't derive location from scans |
| Android 11 and lower | `ACCESS_FINE_LOCATION`, because BLE scans can reveal location |

Concepts to know:

- Bluetooth can be **turned off** for the whole device, separately from the permission. Check the adapter state and ask the user to turn it on.
- Common library: `react-native-ble-plx`.
- It can't be tested on simulators. You need a real device and real hardware (or a phone app that simulates a BLE peripheral).

## Notifications

There are two kinds:

| | Local notifications | Push (remote) notifications |
|---|---|---|
| Created by | The app itself, on the device | A server |
| Delivered through | The OS scheduler | Apple (APNs) / Google (FCM) |
| Example | "Your timer is done", a daily reminder | "Your order has shipped", a new chat message |
| Library | `@notifee/react-native` | `@react-native-firebase/messaging` |

Permissions:

- **iOS**: always needs the user's permission, requested with an API call (there is no Info.plist key).
- **Android 13+**: needs `POST_NOTIFICATIONS` (runtime). Older Android allowed notifications by default. See [Notification permission on Android 13+](10-android-13-notification-permission.md).
- **Android channels**: since Android 8, every notification belongs to a **channel** (e.g. "Messages", "Promotions"). The user can turn channels off one by one, and the importance (sound, pop-up) is set per channel.

Push notifications are covered in detail from [Firebase setup](08-firebase-setup.md) to [Deep linking from a notification](19-deep-link-from-notification.md).

## Background tasks

The OS **suspends** an app soon after it goes to the background, to save battery. JS stops running. "Background task" means asking the OS for permission to run some code anyway.

What you can do:

| Need | iOS | Android |
|---|---|---|
| Finish an upload after the user leaves | Short extra time, or a background `URLSession` | WorkManager, or a foreground service |
| Periodic sync ("every few hours") | `BGTaskScheduler` (`fetch` / `processing` modes). **The OS decides when**, maybe never | WorkManager periodic work (minimum 15 min) |
| Continuous work (music, navigation, fitness tracking) | Specific background modes: `audio`, `location` | **Foreground service** with a visible notification |
| React to a push | Silent push (`content-available`), limited by iOS | Data message handled by a headless JS task |

Concepts to know:

- **The OS is in charge, not the app.** Background execution is a request. iOS in particular decides based on battery, usage patterns, and whether the user force-quit the app (force-quit apps get no background time).
- **Android foreground services** must show a persistent notification, and since Android 14 must declare a **type** (`location`, `mediaPlayback`, `dataSync`, ...).
- **Headless JS** (Android): runs a JS task without UI, used by some libraries for background events.
- Common libraries: `react-native-background-fetch`, and Notifee for foreground services.
- **Don't build features that need guaranteed background timing.** If something must happen at an exact time, the server should do it and send a push.

## Rule of thumb

Bluetooth needs a device, a permission, and Bluetooth turned on. Notifications need permission on iOS and Android 13+, and a channel on Android. Background work is something you ask the OS for and it may refuse, so design features that still work when it does.
