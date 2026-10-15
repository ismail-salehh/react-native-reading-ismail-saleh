# Firebase project setup and google-services.json

Push notifications on Android go through **Firebase Cloud Messaging (FCM)**. Most React Native apps use FCM for iOS too (it forwards to Apple's APNs), so both platforms share one setup. Before any code, the app has to be registered in a Firebase project.

## 1. Create the project

1. Go to the Firebase console and create a project (one per environment is common: `myapp-dev`, `myapp-prod`).
2. Add an **Android app**. The **package name** must be exactly the app's `applicationId` from `android/app/build.gradle` (e.g. `com.example.myapp`).
3. Add an **iOS app**. The **bundle ID** must be exactly the one in Xcode (e.g. `com.example.myapp`).
4. Download the two config files:

| Platform | File | Goes in |
|---|---|---|
| Android | `google-services.json` | `android/app/google-services.json` |
| iOS | `GoogleService-Info.plist` | Add it to the Xcode project (drag into the app target, "Copy items if needed") |

For iOS, the file must be added **through Xcode** so it is part of the target. Copying it into the folder isn't enough.

## What's inside google-services.json

```json
{
  "project_info": {
    "project_number": "123456789012",
    "project_id": "myapp-prod"
  },
  "client": [{
    "client_info": {
      "mobilesdk_app_id": "1:123456789012:android:abc123",
      "android_client_info": { "package_name": "com.example.myapp" }
    },
    "api_key": [{ "current_key": "AIza..." }]
  }]
}
```

It tells the Firebase SDK which project and app this build belongs to. It is **not a server secret**: it ships inside every APK anyway. Still, restrict the API key in Google Cloud console (to your app's package and certificate), and never confuse it with the **service account key** used by your backend to send pushes. That one **is** a secret and must never be in the app.

If the `package_name` inside doesn't match the app's `applicationId`, the build fails with "No matching client found for package name".

## 2. Install React Native Firebase

```bash
npm install @react-native-firebase/app @react-native-firebase/messaging
```

**Android**: add the Google Services Gradle plugin, which reads `google-services.json` at build time.

`android/build.gradle`:

```gradle
buildscript {
  dependencies {
    classpath("com.google.gms:google-services:4.4.2")
  }
}
```

`android/app/build.gradle`:

```gradle
apply plugin: "com.android.application"
apply plugin: "com.google.gms.google-services"
```

**iOS**: Firebase needs static frameworks. In `ios/Podfile`:

```ruby
use_frameworks! :linkage => :static
$RNFirebaseAsStaticFramework = true
```

Then `pod install` and rebuild ([Installing a native package](06-native-packages-autolinking.md)). Check the React Native Firebase docs for the current versions and any `AppDelegate` changes for your React Native version.

**iOS push also needs APNs** set up (capability, key uploaded to Firebase). That is covered in [APNs](13-apns.md).

## 3. One file per environment

Dev and prod usually have separate Firebase projects, so they have different config files. Common approaches:

- **Android build flavors / variants**: put each file in `android/app/src/<flavor>/google-services.json`. The plugin picks the right one.
- **iOS schemes / configurations**: keep `GoogleService-Info-Dev.plist` and `-Prod.plist`, and copy the right one in a build phase script.

Mixing them up means dev builds receive production pushes, or pushes don't arrive at all.

## Checking it works

- The app builds and launches without Firebase errors in the native logs.
- `messaging().getToken()` returns a token ([FCM and the device token](09-fcm-device-token.md)).
- Firebase console -> Messaging -> "Send test message" with that token arrives on the device.

## Rule of thumb

Register each app in Firebase with the exact package name / bundle ID, put `google-services.json` in `android/app` and `GoogleService-Info.plist` in the Xcode target, add the Gradle plugin, and rebuild. The config files identify the app; the server credentials stay on the server.
