# AndroidManifest.xml and MainActivity

Every Android app has a **manifest** that tells the OS what the app is, and at least one **activity** (a screen the OS can launch). In a React Native app there is only one activity, `MainActivity`, and React Native draws every screen inside it.

```
android/app/src/main/
 ├─ AndroidManifest.xml          what the app is and what it needs
 ├─ java/com/app/
 │   ├─ MainActivity.kt          the single Android screen that hosts React Native
 │   └─ MainApplication.kt       app-wide setup: native packages, React host
 └─ res/                         icons, app name (values/strings.xml), themes
```

## The manifest

A typical React Native template:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
      android:name=".MainApplication"
      android:label="@string/app_name"
      android:icon="@mipmap/ic_launcher"
      android:allowBackup="false"
      android:theme="@style/AppTheme"
      android:usesCleartextTraffic="${usesCleartextTraffic}"
      android:supportsRtl="true">

      <activity
        android:name=".MainActivity"
        android:label="@string/app_name"
        android:configChanges="keyboard|keyboardHidden|orientation|screenLayout|screenSize|smallestScreenSize|uiMode"
        android:launchMode="singleTask"
        android:windowSoftInputMode="adjustResize"
        android:exported="true">
        <intent-filter>
          <action android:name="android.intent.action.MAIN" />
          <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
      </activity>
    </application>
</manifest>
```

| Attribute | Why it's there |
|---|---|
| `uses-permission` | Permissions the app may use (Day 4: [Android runtime permissions](../day-4/02-android-runtime-permissions.md)) |
| `application android:name` | The `Application` class to create at startup (`MainApplication`) |
| `android:label` / `android:icon` | App name and launcher icon, from `res/` |
| `usesCleartextTraffic` | Allows `http://`. The `${...}` placeholder is filled by Gradle: `true` in debug (Metro is `http`), `false` in release |
| `launchMode="singleTask"` | Links and notification taps reuse the running app instead of opening a second copy |
| `windowSoftInputMode="adjustResize"` | The window shrinks when the keyboard opens, so inputs aren't hidden |
| `configChanges` | Rotation, dark mode, etc. **don't restart** the activity. React Native handles the change itself |
| `exported="true"` | Other apps (the launcher) can start it. Required on Android 12+ for any activity with an intent filter |
| `MAIN` + `LAUNCHER` filter | Makes this activity the icon on the home screen |

Other things you'll add here: deep link intent filters (Day 4: [URL schemes and App Links](../day-4/14-url-schemes-app-links.md)), `<meta-data>` for SDK keys (e.g. Google Maps), and services that libraries need.

**Manifest merging**: the final manifest is your file merged with the manifests of every library, plus `src/debug/` or flavor folders if they have one. Check the result in Android Studio's "Merged Manifest" tab.

## MainActivity

```kotlin
class MainActivity : ReactActivity() {

  // must match AppRegistry.registerComponent('MyApp', ...) in index.js
  override fun getMainComponentName(): String = "MyApp"

  override fun createReactActivityDelegate(): ReactActivityDelegate =
      DefaultReactActivityDelegate(this, mainComponentName, fabricEnabled)
}
```

- `ReactActivity` creates the React Native view and forwards Android events to it (back button, pause/resume, permission results, new intents from links).
- If the names don't match, the app shows *"Invariant Violation: "MyApp" has not been registered"*.
- Libraries sometimes ask you to edit it. For example, `react-native-screens` asks for `super.onCreate(null)` in `onCreate`, so Android doesn't restore old screen fragments after the process is killed.

## MainApplication

Runs once when the process starts, before any activity. It sets up React Native: the list of native packages, Hermes, and the New Architecture.

```kotlin
override fun getPackages(): List<ReactPackage> =
    PackageList(this).packages.apply {
      // autolinked packages come from PackageList
      // add(MyLocalNativePackage())   <- only for native code that isn't autolinked
    }
```

Some SDKs (crash reporters, analytics) ask for code in `MainApplication.onCreate`, so they start as early as possible.

## Rule of thumb

The manifest describes the app to the OS: permissions, the application class, the launcher activity, and links. `MainActivity` is the one Android screen that hosts all of React Native, and its component name must match `index.js`. `MainApplication` is app-wide startup setup.
