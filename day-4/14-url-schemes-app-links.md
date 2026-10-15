# URL schemes and Android App Links

A **deep link** is a URL that opens the app on a specific screen, instead of only launching it. There are two kinds:

| Kind | Looks like | Platforms |
|---|---|---|
| **Custom URL scheme** | `myapp://orders/812` | iOS and Android |
| **Verified web link** | `https://example.com/orders/812` | **App Links** on Android, **Universal Links** on iOS ([Universal Links](15-universal-links.md)) |

## Custom URL schemes

The app claims a scheme (`myapp`) and the OS sends every `myapp://...` URL to it.

**Android**: an `intent-filter` on the main activity in `AndroidManifest.xml`:

```xml
<activity
  android:name=".MainActivity"
  android:launchMode="singleTask"
  ...>
  <intent-filter>
    <action android:name="android.intent.action.MAIN" />
    <category android:name="android.intent.category.LAUNCHER" />
  </intent-filter>

  <intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="myapp" />
  </intent-filter>
</activity>
```

- `VIEW` + `BROWSABLE`: the link can be opened from a browser or another app.
- `launchMode="singleTask"` (the React Native default): a link opens the **existing** app instead of a second copy of it.

**iOS**: register the scheme in `Info.plist`:

```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array><string>myapp</string></array>
  </dict>
</array>
```

and forward incoming URLs to React Native in `AppDelegate` with `RCTLinkingManager` (the exact code depends on your React Native version, see the React Native `Linking` docs).

### The problems with custom schemes

- **No ownership.** Any app can register `myapp://`. A malicious app can claim the same scheme and receive your links, including links that carry tokens. Never put secrets in a custom-scheme link.
- **No fallback.** If the app isn't installed, `myapp://...` does nothing (or shows an error). You can't send it in an email to someone who may not have the app.
- **Not clickable everywhere.** Many apps (mail clients, chat apps) don't turn `myapp://` into a link.

They're still useful for OAuth redirects, links between your own apps, and testing.

## Android App Links

App Links are normal `https://` links that Android **verifies** belong to your app. If the app is installed, it opens directly; if not, the link opens the website.

**1. Intent filter with `autoVerify`:**

```xml
<intent-filter android:autoVerify="true">
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="https" android:host="example.com" android:pathPrefix="/orders" />
</intent-filter>
```

**2. Prove you own the domain** with a file at `https://example.com/.well-known/assetlinks.json`:

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.myapp",
    "sha256_cert_fingerprints": ["AB:CD:EF:..."]
  }
}]
```

At install time, Android downloads this file and checks that the package name and the app's **signing certificate fingerprint** match. Only then does the link open the app without asking.

The fingerprint must be the one the **installed** app is signed with. With Play App Signing, that's Google's app signing key (Play Console -> App integrity), not your upload key ([Signed AAB](28-signed-aab-release-build.md)). Add the debug key's fingerprint too if you want links to work in debug builds.

If verification fails, the link still works, but Android shows a "Open with..." chooser or opens the browser.

## Comparison

| | Custom scheme | App Links / Universal Links |
|---|---|---|
| URL | `myapp://...` | `https://example.com/...` |
| Setup | Manifest / Info.plist only | Plus a file on your web server |
| Another app can claim it | **Yes** | No, verified with your domain |
| App not installed | Fails | Opens the website |
| Clickable in email/chat | Often not | Yes |

## Testing

```bash
# Android
adb shell am start -W -a android.intent.action.VIEW -d "myapp://orders/812" com.example.myapp
adb shell am start -W -a android.intent.action.VIEW -d "https://example.com/orders/812"

# check App Link verification status
adb shell pm get-app-links com.example.myapp

# iOS Simulator
xcrun simctl openurl booted "myapp://orders/812"

# both, through a helper
npx uri-scheme open "myapp://orders/812" --android
```

## Rule of thumb

Use `https` App Links (and Universal Links on iOS) for links that users share or receive. Use custom schemes for internal cases like OAuth redirects and testing, and never put secrets in them.
