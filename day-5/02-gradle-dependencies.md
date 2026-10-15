# Gradle dependencies

**Gradle** is the build tool for Android. It compiles Kotlin/Java, merges resources and manifests, bundles the JS, signs the app, and downloads the **dependencies** (libraries) the native code needs.

## The files

```
android/
 ├─ gradlew, gradle/wrapper/gradle-wrapper.properties   which Gradle version to use
 ├─ settings.gradle      which modules are in the build (+ autolinking)
 ├─ build.gradle         project-wide: SDK versions, plugin versions, repositories
 ├─ gradle.properties    flags: newArchEnabled, hermesEnabled, JVM memory
 └─ app/
     └─ build.gradle     the app module: applicationId, versions, signing, flavors, dependencies
```

- **Always run `./gradlew`** (the wrapper), not a global `gradle`. The wrapper downloads the exact Gradle version the project expects.
- **The Android Gradle Plugin (AGP)** is what teaches Gradle how to build Android apps. Its version and the Gradle version must be compatible.

## SDK versions

Set in `android/build.gradle`:

```gradle
buildscript {
    ext {
        minSdkVersion = 24
        compileSdkVersion = 35
        targetSdkVersion = 35
        // buildToolsVersion, ndkVersion, kotlinVersion ...
    }
}
```

| Setting | Meaning |
|---|---|
| `minSdkVersion` | The oldest Android version the app installs on |
| `compileSdkVersion` | The Android API the code is compiled against (which APIs exist) |
| `targetSdkVersion` | The Android version whose **behavior rules** the app opts into. Google Play requires a recent one |

A library can require a higher `minSdkVersion` than yours; the build then fails until you raise it.

## Declaring dependencies

In `android/app/build.gradle`:

```gradle
dependencies {
    implementation("com.facebook.react:react-android")
    implementation("com.google.android.gms:play-services-location:21.3.0")
    debugImplementation("com.squareup.leakcanary:leakcanary-android:2.14") // only in debug builds
    testImplementation("junit:junit:4.13.2")
}
```

A dependency is written as **`group:artifact:version`**. Gradle downloads it from the **repositories** listed in the project (`google()`, `mavenCentral()`).

| Configuration | Meaning |
|---|---|
| `implementation` | Used by this module, at compile time and at runtime |
| `api` | Like `implementation`, but also exposed to modules that depend on this one (used in libraries) |
| `compileOnly` | Needed to compile, not packaged in the app |
| `debugImplementation` / `releaseImplementation` | Only in that build type |
| `devImplementation` etc. | Only in that product flavor ([Build variants](03-build-variants-product-flavors.md)) |
| `testImplementation` | Only for unit tests |

Avoid dynamic versions like `1.+`: the build can change without any change in your code.

## You rarely add React Native libraries here

Native React Native libraries (`react-native-keychain`, `react-native-permissions`, ...) are added **by autolinking** (Day 4: [Installing a native package](../day-4/06-native-packages-autolinking.md)). `npm install` + rebuild is enough.

You edit `dependencies` when:

- a library's install docs tell you to add an SDK,
- you write your own native code that needs a library,
- you add a Gradle **plugin**, like Google Services, as a `classpath` in `android/build.gradle` plus `apply plugin` in `app/build.gradle` (Day 4: [Firebase setup](../day-4/08-firebase-setup.md)).

## Version conflicts

Two libraries may depend on different versions of the same library. Gradle picks the **highest** version by default. To see what was chosen:

```bash
cd android
./gradlew :app:dependencies --configuration releaseRuntimeClasspath
```

`1.2.0 -> 1.5.0` in the output means something asked for 1.2.0 and got 1.5.0. If the newer one breaks something, you can force a version:

```gradle
configurations.all {
    resolutionStrategy.force("com.squareup.okhttp3:okhttp:4.12.0")
}
```

Use forcing as a last resort; updating the library that pulls the old version is usually better.

## After changing Gradle files

- In Android Studio: "Sync Project with Gradle Files".
- Rebuild the app (`npx react-native run-android`). Metro reload doesn't pick up native changes.

## Rule of thumb

Gradle builds the native app. SDK versions and plugin versions live in `android/build.gradle`, the app's settings and dependencies in `android/app/build.gradle`. React Native libraries come in through autolinking; you add dependencies by hand only when docs or your own native code need them. Run `./gradlew :app:dependencies` when versions conflict.
