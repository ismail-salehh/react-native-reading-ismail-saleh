# Build variants and product flavors

A real app is built in several versions: a dev build pointing to the dev API, a staging build for QA, and the production build for the store. Android handles this with **build types** and **product flavors**. Each combination is a **build variant**.

## Build types vs product flavors

| | Build type | Product flavor |
|---|---|---|
| Answers | **How** is it built? | **Which version** of the app is it? |
| Defaults | `debug` and `release` | None until you add them |
| Changes | Debuggable, minified, signing, JS from Metro or bundled | App ID, app name, icon, API URL, Firebase project |
| Example | `debug`, `release` | `dev`, `stage`, `prod` |

```
              debug          release
  dev     →   devDebug       devRelease
  stage   →   stageDebug     stageRelease
  prod    →   prodDebug      prodRelease
```

3 flavors × 2 build types = 6 variants. `prodRelease` is the one you upload to the store.

## Adding flavors

`android/app/build.gradle`:

```gradle
android {
    flavorDimensions "env"

    productFlavors {
        dev {
            dimension "env"
            applicationIdSuffix ".dev"          // com.app.dev
            resValue "string", "app_name", "MyApp Dev"
        }
        stage {
            dimension "env"
            applicationIdSuffix ".stage"        // com.app.stage
            resValue "string", "app_name", "MyApp Stage"
        }
        prod {
            dimension "env"                     // com.app
            resValue "string", "app_name", "MyApp"
        }
    }
}
```

- `applicationIdSuffix` gives each flavor its own app ID, so all three can be **installed side by side** ([Different bundle IDs](05-bundle-ids-per-environment.md)).
- `resValue` generates a resource per flavor. Remove `app_name` from `res/values/strings.xml`, or the build fails with a duplicate resource error.
- `buildConfigField "String", "API_URL", '"https://dev.api.com"'` adds a constant to `BuildConfig` for native code. For JS, use `.env` files ([.env and different API URLs](04-env-api-urls.md)).

## Tell React Native which variants are debug

The React Native Gradle plugin bundles the JS into the app for release variants and loads it from Metro for debug ones. With flavors, list the debug variants:

```gradle
react {
    debuggableVariants = ["devDebug", "stageDebug", "prodDebug"]
}
```

Otherwise `devDebug` may not be treated as a debug build.

## Per-flavor files (source sets)

Each flavor can have its own folder next to `main`. Files there are added to, or replace, the ones in `main`:

```
android/app/src/
 ├─ main/                       shared by all
 ├─ dev/
 │   ├─ google-services.json    Firebase dev project
 │   └─ res/mipmap-*/           a "DEV" launcher icon
 ├─ stage/
 └─ prod/
```

- Resources (icons, strings) in a flavor folder **override** `main`.
- A flavor `AndroidManifest.xml` is **merged** into the main one.
- `google-services.json` is picked per flavor automatically ([Firebase per environment](06-firebase-per-environment.md)).

A different icon per environment is a cheap way to stop testers from mixing up builds.

## Building a variant

```bash
# run on a device
npx react-native run-android --mode devDebug --appIdSuffix dev

# build files
cd android
./gradlew assembleStageRelease     # APK
./gradlew bundleProdRelease        # AAB for Google Play
```

`--appIdSuffix` tells the CLI which app ID to launch after installing. Without it, it tries to open `com.app` and fails.

In Android Studio, pick the variant in the **Build Variants** panel.

## Rule of thumb

Build types say **how** to build (debug or release); flavors say **which** app (dev, stage, prod). Use flavors for app ID suffixes, names, icons, and config files, list the debug variants in `react { debuggableVariants }`, and ship `prodRelease`.
