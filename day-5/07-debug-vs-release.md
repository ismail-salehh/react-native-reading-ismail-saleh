# Debug vs release

The same code can be built in two very different ways. A **debug** build is made for developing: fast to change, full of tools, slow to run. A **release** build is what users get: self-contained, optimized, signed with the real key.

## The differences

| | Debug | Release |
|---|---|---|
| Where the JS comes from | **Metro**, over the network. Fast Refresh works | **Bundled inside the app** (`index.android.bundle` / `main.jsbundle`), compiled to Hermes bytecode |
| Runs without your computer | No, needs Metro running | Yes |
| `__DEV__` | `true` | `false` |
| Dev menu, LogBox, red error screen | Yes | No. A JS error crashes or shows your error boundary |
| Speed | Slower: extra checks, unminified JS | Full speed |
| Native code | Not minified, debuggable | Shrunk and obfuscated by R8 on Android ([R8 / ProGuard](10-r8-proguard.md)) |
| Signing | Debug key (`debug.keystore`) on Android, development certificate on iOS | Upload key / distribution certificate ([Signing and keystore](08-signing-keystore.md)) |
| `http://` traffic (Android) | Allowed (Metro is `http`) | Blocked by default |

On Android these are the `debug` and `release` **build types**. On iOS they're the `Debug` and `Release` **build configurations**.

## Running a release build locally

```bash
# Android
npx react-native run-android --mode release

# iOS
npx react-native run-ios --mode Release
```

Or in Xcode: Product -> Scheme -> Edit Scheme -> Run -> Build Configuration: **Release**.

With flavors, use the full variant: `--mode prodRelease` ([Build variants](03-build-variants-product-flavors.md)). Locally, an Android release build needs a signing config; the template signs it with the debug key until you set up your own.

## Why you must test release builds

Things that work in debug and fail in release:

- **Code behind `__DEV__`** doesn't run: mocks, test users, dev-only logging.
- **`http://` URLs** stop working on Android.
- **R8 removes classes** a library uses through reflection, so the app crashes at startup.
- **Wrong environment**: the release build points to a different API than you expect ([.env and different API URLs](04-env-api-urls.md)).
- **Timing**: release is faster, so race conditions hidden by slow debug builds show up.
- **Performance** is only realistic in release. Never judge speed from a debug build (Day 4: [JS thread vs UI thread](../day-4/20-js-thread-vs-ui-thread.md)).

## Debugging a release build

No dev menu, but you still have:

- **Native logs**: `adb logcat` (filter `ReactNativeJS` for `console.log`), Xcode console or Console.app for iOS.
- **Crash reporting** (Crashlytics, Sentry) with source maps and R8 mapping files uploaded, so stack traces are readable.
- A **staging release** build: release mode, but against the staging API, for QA.

## Don't ship logs and debug tools

- Remove `console.log` in release (e.g. `babel-plugin-transform-remove-console`). Logs cost time and can leak tokens to the device log.
- Guard dev screens and debug menus with an environment flag, not just "nobody will find it".

## Rule of thumb

Debug loads JS from Metro and is full of developer tools; release bundles the JS, turns `__DEV__` off, minifies, and is signed with the real key. Develop in debug, but test every build you send to testers or the store as a release build.
