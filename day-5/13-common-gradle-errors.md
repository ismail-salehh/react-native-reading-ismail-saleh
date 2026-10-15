# Common Gradle build errors and how to fix them

Android build errors look scary because Gradle prints a lot. Most of them come from a short list of causes. This file lists the common ones, with the cause and the fix.

## First: read the right line

- Scroll **up** to the first `FAILURE:` / `What went wrong:` block. The last lines are usually just "BUILD FAILED".
- The task name tells you the stage: `:app:compileDebugKotlin` (your code), `:app:mergeDebugResources` (resources), `:app:processDebugMainManifest` (manifest), `:app:checkDebugDuplicateClasses` (dependencies), `:app:buildCMakeDebug` (C++).
- More detail: `./gradlew assembleDebug --stacktrace` (or `--info`).
- `npx react-native doctor` checks your Java, Android SDK, and environment variables.

## The errors

### SDK location not found

```
SDK location not found. Define a valid SDK location with an ANDROID_HOME environment variable
or by setting the sdk.dir path in your project's local properties file
```

**Fix**: set `ANDROID_HOME` in your shell profile, or create `android/local.properties`:

```properties
sdk.dir=/home/you/Android/Sdk
```

`local.properties` is machine-specific; don't commit it.

### Wrong Java version

```
Unsupported class file major version 65
Android Gradle plugin requires Java 17 to run. You are currently using Java 11.
```

**Cause**: Gradle runs with a JDK that doesn't match the project. Current React Native versions need **JDK 17**.
**Fix**: install JDK 17 and point `JAVA_HOME` to it (check with `java -version` and `./gradlew -version`). Android Studio uses its own JDK setting (Settings -> Build Tools -> Gradle -> Gradle JDK).

### Could not resolve a dependency

```
Could not resolve all files for configuration ':app:debugRuntimeClasspath'.
Could not find com.somelib:sdk:2.3.0.
```

**Causes**: no internet / proxy, Gradle in offline mode, the version doesn't exist, or the repository that hosts it isn't listed.
**Fix**: check the version on the library's site, make sure `google()` and `mavenCentral()` (and any repo the library docs mention) are in the repositories, turn off offline mode.

### minSdkVersion too low

```
uses-sdk:minSdkVersion 23 cannot be smaller than version 24 declared in library [:some-library]
```

**Fix**: raise `minSdkVersion` in `android/build.gradle` to what the library needs ([Gradle dependencies](02-gradle-dependencies.md)).

### Manifest merger failed

```
Manifest merger failed : Attribute application@allowBackup value=(false)
is also present at [some-library] value=(true).
Suggestion: add 'tools:replace="android:allowBackup"' to <application>
```

**Cause**: your manifest and a library's set the same attribute differently.
**Fix**: do what the suggestion says, in your `AndroidManifest.xml` ([AndroidManifest.xml and MainActivity](01-androidmanifest-mainactivity.md)):

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          xmlns:tools="http://schemas.android.com/tools">
  <application tools:replace="android:allowBackup" android:allowBackup="false" ...>
```

A related one: *"android:exported needs to be explicitly specified for element <activity>"*. Add `android:exported="true"` (or `false`) to activities with an intent filter (required on Android 12+).

### Duplicate resources

```
Execution failed for task ':app:mergeDebugResources'.
[string/app_name] .../values/strings.xml  [string/app_name] Generated: Error: Duplicate resources
```

**Cause**: the same resource is defined twice, often `app_name` in `strings.xml` **and** in a flavor's `resValue` ([Build variants](03-build-variants-product-flavors.md)).
**Fix**: keep only one definition.

### Duplicate class

```
Duplicate class com.google.common.util.concurrent.ListenableFuture found in modules
guava-31.jar and listenablefuture-1.0.jar
```

**Cause**: two dependencies contain the same class (an old and a new artifact of the same library).
**Fix**: update the library that brings the old one, or exclude it:

```gradle
implementation("com.somelib:sdk:2.3.0") {
    exclude group: "com.google.guava", module: "listenablefuture"
}
```

Find who pulls it in with `./gradlew :app:dependencies`.

### No matching client found for package name

```
No matching client found for package name 'com.app.dev'
```

**Cause**: `google-services.json` doesn't contain this app ID.
**Fix**: put the right file in the flavor folder ([Firebase per environment](06-firebase-per-environment.md)).

### Out of memory

```
Java heap space
Metaspace
The Daemon will expire immediately since the JVM garbage collector is thrashing
```

**Fix**: give Gradle more memory in `android/gradle.properties`:

```properties
org.gradle.jvmargs=-Xmx4096m -XX:MaxMetaspaceSize=1024m
```

### Kotlin version mismatch

```
Module was compiled with an incompatible version of Kotlin.
The binary version of its metadata is 2.0.0, expected version is 1.9.0.
```

**Cause**: a library was built with a newer Kotlin than the project uses.
**Fix**: raise `kotlinVersion` in `android/build.gradle` to a version compatible with your React Native version, or use an older version of the library.

### NDK / CMake / C++ errors

```
[CXX1101] NDK at ... did not have a source.properties file
ninja: error: ... missing and no known rule to make it
```

**Cause**: NDK not installed or the wrong version, or stale C++ build files (common after upgrading React Native or switching branches).
**Fix**: install the `ndkVersion` from `android/build.gradle` in Android Studio's SDK Manager (SDK Tools -> NDK, "Show Package Details"). For stale files, delete `android/app/.cxx`, `android/app/build`, and `android/build`, then rebuild.

### Signatures don't match on install

```
INSTALL_FAILED_UPDATE_INCOMPATIBLE: Package com.app signatures do not match
```

Not a compile error: the build worked, but a build signed with another key is already installed. `adb uninstall com.app` and install again ([Signing and keystore](08-signing-keystore.md)).

## When nothing makes sense: clean up in steps

Try in order, rebuilding after each:

```bash
cd android && ./gradlew clean && cd ..                     # 1. clean the build
npx react-native start --reset-cache                       # 2. Metro cache (JS side)
rm -rf android/app/build android/app/.cxx android/build    # 3. stale build folders
cd android && ./gradlew --stop && cd ..                    # 4. stop old Gradle daemons
rm -rf node_modules && npm install                         # 5. reinstall JS packages
rm -rf ~/.gradle/caches                                    # 6. last resort: re-downloads everything
```

If the error appeared after adding or upgrading a library, check that library's issues for your React Native version first. It's often a known problem with a known fix.

## Rule of thumb

Find the first `What went wrong:` and the task name, then match it: environment (SDK path, Java version), dependencies (resolve, duplicate class, minSdk, Kotlin), manifest/resources (merger, duplicates), or native build (NDK, stale `.cxx`). Clean step by step only when the error doesn't point anywhere.
