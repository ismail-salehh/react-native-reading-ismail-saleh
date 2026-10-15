# iOS Info.plist keys (read only)

On iOS, permissions are not "declared" like on Android. Instead, every protected feature needs a **usage description**: a sentence in `Info.plist` that iOS shows inside the permission prompt. It explains to the user why the app wants access.

`Info.plist` is at `ios/<AppName>/Info.plist`. It is an XML file of key/value pairs (Xcode also shows it as a table).

## What it looks like

```xml
<key>NSCameraUsageDescription</key>
<string>The camera is used to scan QR codes on receipts.</string>

<key>NSLocationWhenInUseUsageDescription</key>
<string>Your location is used to show stores near you.</string>
```

The prompt then shows:

```
┌────────────────────────────────────────┐
│  "MyApp" Would Like to Access          │
│  the Camera                            │
│                                        │
│  The camera is used to scan QR codes   │  <- your string
│  on receipts.                          │
│                                        │
│     Don't Allow          OK            │
└────────────────────────────────────────┘
```

## If the key is missing, the app crashes

This is the most important fact in this file. If the app requests a protected resource and the matching key is not in `Info.plist`, **iOS terminates the app** immediately. No prompt, no error you can catch.

So every library that uses the camera, photos, location, etc. lists the keys you must add in its install docs.

## Common keys

| Key | Needed for |
|---|---|
| `NSCameraUsageDescription` | Camera |
| `NSMicrophoneUsageDescription` | Microphone (also video recording with sound) |
| `NSPhotoLibraryUsageDescription` | Reading the photo library |
| `NSPhotoLibraryAddUsageDescription` | Only saving photos to the library |
| `NSLocationWhenInUseUsageDescription` | Location while the app is open |
| `NSLocationAlwaysAndWhenInUseUsageDescription` | Location in the background as well |
| `NSContactsUsageDescription` | Contacts |
| `NSBluetoothAlwaysUsageDescription` | Bluetooth |
| `NSFaceIDUsageDescription` | Face ID (Day 3: [biometric authentication](../day-3/07-biometric-authentication.md)) |
| `NSCalendarsUsageDescription` | Calendar (iOS 17 has more specific keys too) |

**Notifications have no key.** The push permission prompt is shown by calling the notifications API, without a usage description.

## Writing good descriptions

Apple reviews these strings. Vague ones like "This app needs the camera" can get the app **rejected**. A good description says:

- **what** is used,
- **for which feature**,
- in plain words the user understands.

```
✕  "Camera access required."
✓  "The camera is used to take a profile photo and to scan receipts."
```

To translate them, add an `InfoPlist.strings` file per language. The usage descriptions are localized like any other app text (Day 2: [localization](../day-2/11-localization.md)).

## Other Info.plist keys you'll meet

Info.plist also configures things unrelated to permissions:

| Key | Purpose |
|---|---|
| `CFBundleURLTypes` | Custom URL schemes for [deep links](14-url-schemes-app-links.md) |
| `UIBackgroundModes` | Background capabilities: `remote-notification`, `location`, `audio`, `fetch`, `processing` |
| `NSAppTransportSecurity` | Exceptions to the HTTPS-only rule |
| `LSApplicationQueriesSchemes` | Other apps' schemes you want to check with `Linking.canOpenURL` |

Some features also need a **capability** in Xcode (Signing & Capabilities tab), such as Push Notifications or Associated Domains. Those go into a separate `.entitlements` file, not Info.plist.

## Rule of thumb

Every protected feature needs its `NS...UsageDescription` key, or the app crashes. Write each one as a clear, specific sentence, because users read it and Apple reviews it.
