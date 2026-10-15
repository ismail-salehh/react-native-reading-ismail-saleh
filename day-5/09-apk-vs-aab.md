# APK vs AAB

Android has two output formats. The **APK** is the file a phone installs. The **AAB** (Android App Bundle) is the file you upload to Google Play, which turns it into APKs for each device. Day 4 introduced them in [Generate a signed AAB](../day-4/28-signed-aab-release-build.md); this file covers when to use which.

## The difference

```
APK:  build ─► app-release.apk ─────────────────────────────► install on any phone

AAB:  build ─► app-release.aab ─► Google Play ─► split APKs for THIS device ─► install
                                   (arm64, xxhdpi, English)
```

| | APK | AAB |
|---|---|---|
| Installable directly | Yes (`adb install`, tap the file) | **No**, a publishing format only |
| Contents | Everything for every device | Everything, but Play delivers only what each device needs |
| Download size for users | Larger | Smaller (often 15-35% less) |
| Signed for users by | You | Google, with the app signing key ([Signing and keystore](08-signing-keystore.md)) |
| Google Play | Not accepted for new apps | **Required** |
| Other uses | QA sideloading, other stores, direct download | Play only (and tools that accept it) |

## What makes an AAB smaller

An app contains code and resources for many device types. Each phone needs only one of each:

| Split by | Example | Why it matters in React Native |
|---|---|---|
| CPU architecture (ABI) | `arm64-v8a`, `armeabi-v7a`, `x86_64` | React Native, Hermes, and native libraries ship `.so` files for **each** ABI. This is the biggest part |
| Screen density | `xxhdpi`, `xxxhdpi` | Only one set of image resources is delivered |
| Language | `en`, `ar` | Only the device's languages (Android string resources, not your JS translations) |

## Building each

```bash
cd android

./gradlew assembleRelease   # APK → app/build/outputs/apk/release/app-release.apk
./gradlew bundleRelease     # AAB → app/build/outputs/bundle/release/app-release.aab
```

With flavors: `assembleStageRelease`, `bundleProdRelease` ([Build variants](03-build-variants-product-flavors.md)).

Install an APK:

```bash
adb install -r app/build/outputs/apk/release/app-release.apk
```

To install from an AAB locally, generate APKs with Google's **bundletool** (commands in Day 4), which does what Play would do for your connected device.

## Which one when

| Situation | Use |
|---|---|
| Upload to Google Play (any track) | **AAB** |
| Send a build to QA without Play (Firebase App Distribution, a shared link) | APK (App Distribution can also take an AAB if linked to Play) |
| Install quickly on your own device | APK, or `run-android --mode release` |
| Stores other than Google Play | Whatever that store requires |

For regular team testing, Play's internal testing track with the AAB is closer to what users get (Day 4: [Play Store internal testing](../day-4/29-play-store-internal-testing.md)).

## Smaller APKs for local builds

During development you can build only the ABI of your device, which makes builds much faster:

```bash
npx react-native run-android --active-arch-only
```

Or limit ABIs in `android/gradle.properties` with `reactNativeArchitectures=arm64-v8a`. Don't ship a store build with fewer ABIs than you support.

## Rule of thumb

APK = installable file, for sideloading and testing. AAB = upload format for Google Play, which builds smaller per-device APKs from it. Build AABs with `bundleRelease` for Play, APKs with `assembleRelease` for everything else.
