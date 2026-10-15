# Generate a signed AAB and test the release build

During development you run a **debug** build: it loads JS from Metro, has the dev menu, and is signed with a shared debug key. To publish on Google Play you need a **release** build, packaged as an **AAB** (Android App Bundle) and **signed** with your own key.

## APK vs AAB

| | APK | AAB |
|---|---|---|
| What | An installable app file | A publishing format: all code and resources for all devices |
| Installable directly | Yes | **No** |
| Used for | Sideloading, testing, other stores | **Google Play** (required for new apps) |
| Size for the user | Contains everything | Play generates optimized APKs per device (only that device's CPU architecture, screen density, language) |

You upload the AAB; Google Play builds and signs the APKs each user downloads.

## Two keys: upload key and app signing key

With **Play App Signing** (the default for new apps):

```
you:         sign the AAB with your UPLOAD KEY  -> upload to Play
Google Play: verifies the upload key, re-signs with the APP SIGNING KEY -> users
```

- The **app signing key** is kept by Google. It's the identity of the app forever.
- The **upload key** is yours. If you lose it, you can ask Google to reset it (unlike the old system, where losing the key meant you could never update the app again).

## 1. Generate an upload key

```bash
keytool -genkeypair -v -storetype PKCS12 \
  -keystore my-upload-key.keystore \
  -alias my-key-alias \
  -keyalg RSA -keysize 2048 -validity 10000
```

It asks for a password and some details, and creates `my-upload-key.keystore`. Put it in `android/app/`.

**Never commit the keystore or its passwords to git.** Add `*.keystore` to `.gitignore`, and back the file and passwords up somewhere safe (a password manager, your CI's secret storage).

## 2. Give Gradle the credentials

In `~/.gradle/gradle.properties` (your user folder, **not** the project, so it's not committed):

```properties
MYAPP_UPLOAD_STORE_FILE=my-upload-key.keystore
MYAPP_UPLOAD_KEY_ALIAS=my-key-alias
MYAPP_UPLOAD_STORE_PASSWORD=*****
MYAPP_UPLOAD_KEY_PASSWORD=*****
```

## 3. Configure signing

`android/app/build.gradle`:

```gradle
android {
    defaultConfig {
        applicationId "com.example.myapp"
        versionCode 12        // integer, must increase with every upload
        versionName "1.4.0"   // shown to users
    }

    signingConfigs {
        release {
            if (project.hasProperty('MYAPP_UPLOAD_STORE_FILE')) {
                storeFile file(MYAPP_UPLOAD_STORE_FILE)
                storePassword MYAPP_UPLOAD_STORE_PASSWORD
                keyAlias MYAPP_UPLOAD_KEY_ALIAS
                keyPassword MYAPP_UPLOAD_KEY_PASSWORD
            }
        }
    }

    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled enableProguardInReleaseBuilds
            proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
        }
    }
}
```

The default template signs release builds with the **debug** key. Make sure `release` points to `signingConfigs.release`, otherwise Play rejects the upload.

`versionCode` must be **higher than every previous upload**, or Play refuses it.

## 4. Build the AAB

```bash
cd android
./gradlew bundleRelease
```

Output: `android/app/build/outputs/bundle/release/app-release.aab`.

The JS is bundled **into** the app (Metro isn't needed), compiled to Hermes bytecode ([Hermes](25-hermes.md)), and `__DEV__` is `false`.

## 5. Test the release build

A release build behaves differently from debug, so **always test it before uploading**:

```bash
# install a release build on a connected device/emulator
npx react-native run-android --mode release
```

To test the **exact AAB** you'll upload, use Google's `bundletool`:

```bash
bundletool build-apks --bundle=app/build/outputs/bundle/release/app-release.aab \
  --output=app.apks --ks=app/my-upload-key.keystore --ks-key-alias=my-key-alias

bundletool install-apks --apks=app.apks
```

Things that often break only in release:

| Problem | Why |
|---|---|
| Requests to `http://` fail | Cleartext traffic is blocked; debug config often allowed it |
| Crash on startup | R8/ProGuard removed or renamed classes a library uses via reflection. Add keep rules from the library's docs |
| Wrong API URL | Environment config still pointing to dev, or `__DEV__` logic |
| Missing assets or fonts | Not bundled correctly |
| Code behind `__DEV__` | Doesn't run in release |
| Google sign-in / App Links / Firebase fail | They check the **signing certificate SHA**: add the upload key's and Play app signing key's fingerprints in Firebase / Google Cloud / `assetlinks.json` |

Watch logs while testing: `adb logcat *:E` or filter by `ReactNativeJS`.

## Rule of thumb

Create an upload key once and keep it out of git, configure the `release` signing config, raise `versionCode` every time, build with `./gradlew bundleRelease`, and install and test the release build on a real device before uploading.
