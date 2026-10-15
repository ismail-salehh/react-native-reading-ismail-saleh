# Installing a native package, autolinking, rebuilding the app

Some packages are **pure JS** (e.g. `axios`, `zustand`, `date-fns`). Others contain **native code** (Swift/Objective-C for iOS, Kotlin/Java for Android), because they talk to the camera, the Keychain, Bluetooth, etc. Installing the second kind needs a few more steps.

## How to tell them apart

A package has native code if its folder contains `ios/`, `android/`, a `*.podspec` file, or the docs say "run pod install" / "rebuild the app". Examples: `react-native-keychain`, `react-native-permissions`, `@react-native-firebase/*`, `react-native-reanimated`, `react-native-vision-camera`.

## The steps

```bash
# 1. install the JS package
npm install react-native-keychain

# 2. iOS: install the native dependency through CocoaPods
cd ios && pod install && cd ..
#    (or: npx pod-install)

# 3. rebuild the app binary
npx react-native run-ios
npx react-native run-android
```

Then follow the library's extra setup, if any: Info.plist keys, `AndroidManifest.xml` permissions, a Babel plugin, or code in `AppDelegate` / `MainApplication`.

## Autolinking

Before React Native 0.60, you had to "link" each native library by hand: add it to Xcode, edit `settings.gradle`, register the package in `MainApplication.java`. Mistakes were common.

**Autolinking** does it automatically:

```
npm install
 -> package added to node_modules
 -> at build time, the React Native CLI scans node_modules
    for packages with native code (react-native.config.js / podspec / android folder)
 -> iOS:     adds them to the Podfile through use_native_modules!  (pod install)
 -> Android: adds them to the Gradle build and registers their packages
```

You don't see it in your code, but it's why `pod install` and a Gradle build are enough. To see what was detected:

```bash
npx react-native config
```

## Why Metro reload is not enough

```
JS code  ->  bundled by Metro  ->  can be reloaded at any time (Fast Refresh)
native code -> compiled into the app binary by Xcode / Gradle -> needs a rebuild
```

The app running on your phone is a compiled binary with a fixed set of native modules inside. Metro can send it new JS, but it can't add new native code to it. After installing a native package, the JS can import it, but the native side doesn't exist in the binary yet.

Typical errors when you forget to rebuild:

- `Invariant Violation: TurboModuleRegistry.getEnforcing(...): 'RNKeychainManager' could not be found`
- `Native module cannot be null`
- `null is not an object (evaluating 'NativeModules.X.method')`

The fix is almost always: `pod install`, then rebuild (not just reload).

## When things still don't work

In order:

1. Stop Metro and start it with a clean cache: `npx react-native start --reset-cache`.
2. iOS: `cd ios && pod install` again. If needed, delete `ios/Pods` and `ios/Podfile.lock` and re-run it.
3. Android: `cd android && ./gradlew clean`.
4. Check the library's version supports your React Native version and the **New Architecture** ([Old vs new architecture](26-old-vs-new-architecture.md)).
5. Read the library's install docs again for a manual step you missed.

## Rule of thumb

JS-only package: install and reload. Native package: install, `pod install`, and **rebuild the app**. If you see "native module could not be found", you almost certainly skipped the rebuild.
