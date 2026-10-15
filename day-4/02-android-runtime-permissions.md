# Android runtime permissions and AndroidManifest.xml

On Android, a permission must be in **two places** before the app can use it:

1. **Declared** in `AndroidManifest.xml`. This is a list of everything the app might ever ask for. It is read at install time and shown on the Play Store.
2. **Requested at runtime** (for "dangerous" permissions). The user sees a dialog while using the app.

If a permission is missing from the manifest, the runtime request fails immediately without showing anything, which is a common "why does nothing happen?" bug.

## Declaring in the manifest

`android/app/src/main/AndroidManifest.xml`:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

    <!-- only needed on Android 12 and lower -->
    <uses-permission
        android:name="android.permission.READ_EXTERNAL_STORAGE"
        android:maxSdkVersion="32" />

    <application ...>
```

- `maxSdkVersion` limits a permission to older Android versions. Use it for permissions newer versions replaced.
- Libraries can add permissions to the final manifest through **manifest merging**. Check the merged manifest (Android Studio: `AndroidManifest.xml` -> "Merged Manifest" tab) before release, so you don't ship permissions you don't use. Remove one with `tools:node="remove"`.

## Normal vs dangerous permissions

| Type | Examples | Runtime request |
|---|---|---|
| **Normal** | `INTERNET`, `VIBRATE`, `ACCESS_NETWORK_STATE` | No. Granted at install from the manifest |
| **Dangerous** (runtime) | `CAMERA`, `RECORD_AUDIO`, `ACCESS_FINE_LOCATION`, `READ_MEDIA_IMAGES`, `POST_NOTIFICATIONS`, `BLUETOOTH_CONNECT` | **Yes**, user must allow |
| **Special** | `SYSTEM_ALERT_WINDOW`, exact alarms, "all files access" | No dialog. The user toggles it on a Settings page |

## Requesting with PermissionsAndroid

React Native has a built-in Android-only API:

```ts
import { PermissionsAndroid, Platform } from 'react-native';

async function requestCamera() {
  if (Platform.OS !== 'android') return true;

  const result = await PermissionsAndroid.request(
    PermissionsAndroid.PERMISSIONS.CAMERA,
    {
      title: 'Camera permission',
      message: 'We need the camera to scan QR codes.',
      buttonPositive: 'OK',
    },
  );

  return result === PermissionsAndroid.RESULTS.GRANTED;
}
```

Results:

| Value | Meaning |
|---|---|
| `granted` | Allowed |
| `denied` | Refused, can ask again |
| `never_ask_again` | Refused and the system won't ask again (the `blocked` state from [Permission states](01-permission-states.md)) |

The `title`/`message` options show **your own** explanation dialog (a "rationale") before the system one, only when Android says a rationale should be shown, that is after the user refused once.

Requesting several at once:

```ts
const results = await PermissionsAndroid.requestMultiple([
  PermissionsAndroid.PERMISSIONS.ACCESS_FINE_LOCATION,
  PermissionsAndroid.PERMISSIONS.ACCESS_COARSE_LOCATION,
]);
```

## Permissions that changed between Android versions

Android versions are identified by **API level**. Several permissions depend on it:

| Feature | Old | New |
|---|---|---|
| Notifications | No permission | `POST_NOTIFICATIONS` from **Android 13 (API 33)** |
| Read photos/videos | `READ_EXTERNAL_STORAGE` | `READ_MEDIA_IMAGES`, `READ_MEDIA_VIDEO` from Android 13 |
| Bluetooth | `ACCESS_FINE_LOCATION` was needed to scan | `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT` from Android 12 |
| Location | Precise or nothing | User can pick **Approximate** from Android 12 |
| Background location | Asked together with location | Separate request, and from Android 11 only via Settings |

Check the version with `Platform.Version` (on Android it is the API level number):

```ts
if (Platform.OS === 'android' && Platform.Version >= 33) {
  await PermissionsAndroid.request(PermissionsAndroid.PERMISSIONS.POST_NOTIFICATIONS);
}
```

These version checks are the main reason teams use [react-native-permissions](07-react-native-permissions.md): it hides many of them.

## targetSdkVersion matters

New permission rules usually apply only when the app **targets** the new API level (`targetSdkVersion` in `android/build.gradle`). Google Play requires a recent target, so every year when you raise it, re-check the permission changes for that version.

## Rule of thumb

Declare every permission in the manifest, request dangerous ones at runtime when the feature is used, handle `never_ask_again`, and check the API level for permissions that changed between Android versions.
