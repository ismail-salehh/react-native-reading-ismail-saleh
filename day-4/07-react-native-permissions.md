# react-native-permissions

`react-native-permissions` is a library that gives **one API for permissions on both platforms**. React Native's built-in `PermissionsAndroid` only covers Android ([Android runtime permissions](02-android-runtime-permissions.md)); this library covers iOS too and maps both to the same states ([Permission states](01-permission-states.md)).

## Installation

```bash
npm install react-native-permissions
```

It is a native package, so a rebuild is needed ([Installing a native package](06-native-packages-autolinking.md)).

**iOS**: each permission is a separate native module, and you only include the ones you use. In `ios/Podfile`:

```ruby
def node_require(script)
  require Pod::Executable.execute_command('node', ['-p',
    "require.resolve('#{script}', {paths: [process.argv[1]]})", __dir__]).strip
end

node_require('react-native-permissions/scripts/setup.rb')

setup_permissions([
  'Camera',
  'LocationWhenInUse',
  'Notifications',
  'PhotoLibrary',
])
```

Then `pod install`, and add the matching `NS...UsageDescription` keys to Info.plist ([Info.plist keys](03-ios-info-plist.md)).

**Android**: add the permissions to `AndroidManifest.xml` as usual.

Including only what you use matters: Apple rejects apps that link permission APIs (and their usage strings) the app never uses.

## The main functions

| Function | Does |
|---|---|
| `check(permission)` | Reads the current state, **no prompt** |
| `request(permission)` | Shows the system prompt if possible, returns the new state |
| `checkMultiple([...])` / `requestMultiple([...])` | Several at once |
| `checkNotifications()` / `requestNotifications(options)` | Notifications (handled separately on both platforms) |
| `openSettings()` | Opens the app's page in the Settings app |
| `openPhotoPicker()` | iOS: lets the user change the selection in `limited` mode |

Permissions are named per platform:

```ts
import { PERMISSIONS, RESULTS } from 'react-native-permissions';

PERMISSIONS.IOS.CAMERA
PERMISSIONS.IOS.LOCATION_WHEN_IN_USE
PERMISSIONS.ANDROID.CAMERA
PERMISSIONS.ANDROID.ACCESS_FINE_LOCATION
```

The result is always one of `RESULTS.UNAVAILABLE`, `DENIED`, `BLOCKED`, `GRANTED`, `LIMITED`.

## A reusable hook

```ts
import { useCallback, useEffect, useState } from 'react';
import { AppState, Platform } from 'react-native';
import { check, request, openSettings, PERMISSIONS, PermissionStatus, RESULTS } from 'react-native-permissions';

const CAMERA = Platform.select({
  ios: PERMISSIONS.IOS.CAMERA,
  default: PERMISSIONS.ANDROID.CAMERA,
});

export function useCameraPermission() {
  const [status, setStatus] = useState<PermissionStatus | null>(null);

  const refresh = useCallback(() => {
    check(CAMERA).then(setStatus);
  }, []);

  useEffect(() => {
    refresh();
    // the user may change it in Settings and come back
    const sub = AppState.addEventListener('change', (s) => s === 'active' && refresh());
    return () => sub.remove();
  }, [refresh]);

  const ask = useCallback(async () => {
    if (status === RESULTS.BLOCKED) return openSettings();
    setStatus(await request(CAMERA));
  }, [status]);

  return { status, ask };
}
```

```tsx
function ScanScreen() {
  const { status, ask } = useCameraPermission();

  if (status === null) return null; // still checking
  if (status === RESULTS.GRANTED) return <Scanner />;
  if (status === RESULTS.UNAVAILABLE) return <Text>This device has no camera.</Text>;

  return (
    <View>
      <Text>We use the camera to scan QR codes.</Text>
      <Button
        title={status === RESULTS.BLOCKED ? 'Open Settings' : 'Allow camera'}
        onPress={ask}
      />
    </View>
  );
}
```

## Notifications

Notifications have their own functions, because on iOS you choose *which kinds* to ask for:

```ts
import { requestNotifications } from 'react-native-permissions';

const { status, settings } = await requestNotifications(['alert', 'sound', 'badge']);
```

On Android 13+ this requests `POST_NOTIFICATIONS`; on older Android it returns `granted` without a prompt. See [Notification permission on Android 13+](10-android-13-notification-permission.md).

## Things to know

- `request()` on a `blocked` permission returns `blocked` again **without showing anything**. Check for it and offer `openSettings()`.
- On Android, changing a permission in Settings can **restart the app process**. Don't rely on in-memory state surviving it.

## Rule of thumb

Use `check` to decide what to show, `request` only from a user action when the status is `denied`, and `openSettings` when it is `blocked`. On iOS, include only the permission modules you actually use.
