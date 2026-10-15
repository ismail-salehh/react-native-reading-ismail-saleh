# Different bundle IDs: com.app.dev, com.app.stage, com.app

The **bundle ID** (iOS) or **application ID** (Android) is the app's unique identity. The OS, the stores, Firebase, and push services all identify the app by it. Two builds with the same ID are the **same app**: installing one replaces the other.

Giving each environment its own ID means:

```
com.app.dev     MyApp Dev      → dev API, dev Firebase
com.app.stage   MyApp Stage    → staging API, staging Firebase
com.app         MyApp          → production
```

- All three can be **installed at the same time** on one phone.
- Testing dev never wipes the production app (or its login, data, and tokens).
- Each one has its own push tokens, Firebase project, and analytics, so test data never mixes with real data.

## Android: applicationId vs namespace

`android/app/build.gradle`:

```gradle
android {
    namespace "com.app"                // the package of your Kotlin code, R and BuildConfig
    defaultConfig {
        applicationId "com.app"        // the identity on the device and on Google Play
    }
    productFlavors {
        dev   { dimension "env"; applicationIdSuffix ".dev" }
        stage { dimension "env"; applicationIdSuffix ".stage" }
        prod  { dimension "env" }
    }
}
```

| | `namespace` | `applicationId` |
|---|---|---|
| What | Package name of the code | The app's ID |
| Changes per flavor | No | Yes (with the suffix) |
| Can it change after release? | Yes, it's internal | **Never** for a published app. A new ID is a new app on Play |

So you **don't** rename Kotlin packages per environment. Only the `applicationId` changes. See [Build variants and product flavors](03-build-variants-product-flavors.md).

## iOS: build configurations

iOS has no flavors. The same idea uses **build configurations** and **schemes**:

1. Project -> Info -> Configurations: duplicate `Debug` and `Release` into `Debug-Dev`, `Release-Dev`, `Debug-Stage`, `Release-Stage` (keep `Debug`/`Release` for prod).
2. Target -> Build Settings -> **Product Bundle Identifier**: set a value per configuration (`com.app.dev`, `com.app.stage`, `com.app`).
3. Set **Product Name** or the display name (`CFBundleDisplayName`) per configuration too, so the icon label shows "MyApp Dev".
4. Create a **scheme** per environment (`MyApp-Dev`, `MyApp-Stage`, `MyApp`) whose Run action uses `Debug-X` and Archive action uses `Release-X` ([Xcode project and schemes](11-ios-xcode-project-cocoapods-schemes.md)).
5. Tell CocoaPods which new configurations are debug or release, in the `Podfile`:

```ruby
project 'MyApp', {
  'Debug-Dev' => :debug, 'Release-Dev' => :release,
  'Debug-Stage' => :debug, 'Release-Stage' => :release,
}
```

Then `pod install`.

```bash
npx react-native run-ios --scheme MyApp-Dev --mode Debug-Dev
```

## Everything tied to the ID

Each new ID has to be registered everywhere the ID is checked:

| Where | Per ID |
|---|---|
| Firebase | A separate app (and usually a separate project) with its own config file ([Firebase per environment](06-firebase-per-environment.md)) |
| Apple Developer | An App ID with its capabilities, and its own provisioning profiles ([iOS signing](12-ios-signing-archive-testflight.md)) |
| App Store Connect / Play Console | A separate app record, if you distribute that build through TestFlight or Play testing |
| Deep links | `assetlinks.json` and the AASA file list every ID (Day 4: [App Links](../day-4/14-url-schemes-app-links.md), [Universal Links](../day-4/15-universal-links.md)) |
| OAuth / Google sign-in, Maps keys | Client IDs and API key restrictions per ID |
| Push | APNs and FCM work per app |

A common mistake: a new staging ID works for everything except push or Google sign-in, because nobody registered it there.

## Pick the IDs once

Choose the IDs before the first store release. The production ID (`com.app`) can never change for that store listing. Suffixes for the other environments keep them visibly related.

## Rule of thumb

One ID per environment (`com.app.dev`, `com.app.stage`, `com.app`) lets all builds live side by side with separate data. On Android change only the `applicationId` with a suffix; on iOS set the bundle ID per build configuration and use one scheme per environment. Then register each ID everywhere it's checked.
