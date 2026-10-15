# Camera, location, photos, microphone (main idea of each)

These four are the most common protected features. Each one has its own permission, its own privacy rules, and a library that is commonly used for it. This file covers the main idea of each, not every API.

## Summary

| Feature | iOS key | Android permission | Common library |
|---|---|---|---|
| Camera | `NSCameraUsageDescription` | `CAMERA` | `react-native-vision-camera` |
| Location | `NSLocationWhenInUseUsageDescription` (+ `Always...` for background) | `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION` (+ `ACCESS_BACKGROUND_LOCATION`) | `react-native-geolocation-service` |
| Photos | `NSPhotoLibraryUsageDescription` | `READ_MEDIA_IMAGES` (13+), `READ_EXTERNAL_STORAGE` (older) | `react-native-image-picker` |
| Microphone | `NSMicrophoneUsageDescription` | `RECORD_AUDIO` | Audio recorder libraries, camera libraries for video |

Keys and permissions are explained in [Info.plist keys](03-ios-info-plist.md) and [Android runtime permissions](02-android-runtime-permissions.md).

## Camera

The app shows a live preview and captures photos, video, or scans codes.

```tsx
import { Camera, useCameraDevice, useCameraPermission } from 'react-native-vision-camera';

function Scanner() {
  const device = useCameraDevice('back');
  const { hasPermission, requestPermission } = useCameraPermission();

  if (!hasPermission) return <Button title="Allow camera" onPress={requestPermission} />;
  if (!device) return <Text>No camera found</Text>;

  return <Camera style={StyleSheet.absoluteFill} device={device} isActive />;
}
```

Main ideas:

- The camera is a **native view**. Frames are processed natively; only the result (a file path, a scanned code) comes back to JS.
- Turn it off (`isActive={false}`) when the screen is not focused or the app is in the background. Leaving it running drains the battery and keeps the camera indicator on.
- Simulators have no real camera. Test on a device.
- If you only need "take a photo", a **picker** that opens the system camera (`launchCamera`) is simpler and doesn't need a custom camera screen.

## Location

The app reads the device's position (latitude, longitude, accuracy).

```ts
import Geolocation from 'react-native-geolocation-service';

Geolocation.getCurrentPosition(
  (pos) => console.log(pos.coords.latitude, pos.coords.longitude, pos.coords.accuracy),
  (err) => console.log(err.code, err.message),
  { enableHighAccuracy: true, timeout: 15_000 },
);
```

Main ideas:

- **Foreground vs background.** "While using the app" is enough for most apps (store finder, map). Background location ("Always") is reviewed strictly by Apple and Google and needs a real reason (navigation, fitness tracking).
- **Precise vs approximate.** Since iOS 14 and Android 12, the user can give only approximate location (a few kilometres). Design for it: a store finder works fine with it, turn-by-turn navigation doesn't.
- **One position vs watching.** `getCurrentPosition` once vs `watchPosition` for updates. Always stop watching (`clearWatch`) when done, or the GPS keeps running.
- **Location services can be off** for the whole device, separate from the app permission. Handle that error too.

## Photos

The app reads images from the user's library, or saves images to it.

Main ideas:

- **Use the system photo picker when you can.** The user picks images in a system UI and the app only receives the selected files. On Android 13+ the system Photo Picker needs **no permission at all**, and on iOS `PHPickerViewController` (used by modern picker libraries) doesn't need library access either.
- Ask for **full library access** only if the app really browses the library itself (a gallery app, a custom grid).
- **Limited access** (iOS 14+, Android 14+): the user may share only some photos. That shows up as the `limited` state ([Permission states](01-permission-states.md)).
- **Saving** to the library is a separate, smaller permission on iOS (`NSPhotoLibraryAddUsageDescription`).

```ts
import { launchImageLibrary } from 'react-native-image-picker';

const result = await launchImageLibrary({ mediaType: 'photo', selectionLimit: 3 });
if (!result.didCancel && result.assets) {
  const uris = result.assets.map((a) => a.uri);
}
```

## Microphone

The app records audio, or video with sound.

Main ideas:

- Needed for voice messages, voice search, calls, and **any video recording with sound** (a camera app needs both camera and microphone).
- iOS shows an orange dot and Android a microphone indicator while recording. Stop recording as soon as you are done.
- Recording in the background needs extra setup (the `audio` background mode on iOS, a foreground service on Android).
- Audio sessions matter on iOS: recording interrupts other audio and can be interrupted by a phone call. Libraries expose events for that.

## Rule of thumb

Ask for the smallest access that works: a picker instead of library access, foreground instead of background location, approximate if precise isn't needed. Turn the camera, GPS, and microphone off as soon as the feature is done.
