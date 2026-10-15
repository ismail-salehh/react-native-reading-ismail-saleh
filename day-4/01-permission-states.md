# Permission states: granted, denied, blocked, limited

Some features (camera, location, photos, microphone, notifications, ...) are protected by the operating system. The app can't use them until the user allows it. At any moment, each permission is in one **state**, and the app's job is to read that state and react to it correctly.

## The states

These are the names used by [react-native-permissions](07-react-native-permissions.md), which unifies iOS and Android:

| State | Meaning | Can the app show the system prompt? |
|---|---|---|
| `granted` | The user allowed it | Not needed |
| `denied` | Not allowed **yet**, or refused once in a way that still allows asking | **Yes** |
| `blocked` | Refused, and the OS won't show the prompt again | **No**, only the Settings app can change it |
| `limited` | Partly allowed (e.g. only some selected photos) | Yes, to ask for more |
| `unavailable` | The feature doesn't exist on this device or OS version | No |

`denied` and `blocked` both mean "you can't use it now". The difference is whether **asking again** does anything.

## How a permission moves between states

```
                 request()
  denied ─────────────────────►  granted
    │  ▲                            │
    │  │ (Android: refused once,    │ user turns it off
    │  │  can ask again)            │ in Settings
    ▼  │                            ▼
  blocked  ◄──── refused again ── denied / blocked
    │
    └── only openSettings() can bring it back
```

The two platforms reach `blocked` differently:

| | iOS | Android |
|---|---|---|
| Before first request | `denied` | `denied` |
| User taps "Don't Allow" | **`blocked` immediately**. iOS shows the prompt only once | Still `denied`, the app can ask again |
| User refuses a second time | n/a | `blocked` (Android 11+). The system stops showing the prompt |
| User changes it in Settings | Any state | Any state; the app process is often restarted |

That is why on iOS the **first** prompt is precious: if the user refuses it, the app can only send them to Settings.

## Limited

`limited` exists mostly on iOS:

- **Photos (iOS 14+)**: "Select Photos..." gives the app access to only the photos the user picked.
- **Contacts (iOS 18+)**: access to selected contacts only.

Android has something similar for photos since Android 14 ("Allow limited access"), and for location since Android 12 ("Approximate" instead of "Precise"). Approximate location is reported as `granted` for the coarse permission, not as `limited`, so check which permission you actually got.

Treat `limited` as "it works, with less data". Don't show an error, but offer a way to add more.

## Handling each state in the UI

```ts
import { check, request, openSettings, PERMISSIONS, RESULTS } from 'react-native-permissions';

async function ensureCamera(): Promise<boolean> {
  const permission = Platform.select({
    ios: PERMISSIONS.IOS.CAMERA,
    android: PERMISSIONS.ANDROID.CAMERA,
  })!;

  let status = await check(permission);
  if (status === RESULTS.DENIED) {
    status = await request(permission); // shows the system prompt
  }

  switch (status) {
    case RESULTS.GRANTED:
    case RESULTS.LIMITED:
      return true;
    case RESULTS.BLOCKED:
      Alert.alert('Camera access is off', 'Turn it on in Settings to scan codes.', [
        { text: 'Cancel', style: 'cancel' },
        { text: 'Open Settings', onPress: () => openSettings() },
      ]);
      return false;
    default: // UNAVAILABLE, or DENIED again
      return false;
  }
}
```

## Good practice

- **Ask in context.** Request the camera when the user taps "Scan", not on the first launch. A prompt the user understands is far more likely to be accepted.
- **Explain first (a "pre-prompt").** For important permissions, show your own screen explaining why, then call `request()`. If the user says "not now" on your screen, you haven't used up the one real iOS prompt.
- **Re-check when the app returns to the foreground.** The user may have changed the permission in Settings. Listen to `AppState` and call `check()` again.
- **The feature must degrade gracefully.** Without location, let the user type a city. Without the camera, let them type the code.
- **Never assume a permission stays granted.** Users can revoke it at any time, and Android can auto-reset permissions of apps that weren't used for months.

## Rule of thumb

`check()` first, `request()` only when the state is `denied`, and send the user to Settings only when it is `blocked`. Ask at the moment the feature is needed, and always have a fallback.
