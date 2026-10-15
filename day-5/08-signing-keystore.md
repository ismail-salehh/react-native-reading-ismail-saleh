# Signing and keystore

Every Android app must be **signed** before it can be installed. The signature proves **who** built the app, and that it wasn't changed afterwards. Android only installs an **update** if it's signed with the **same key** as the installed version.

The step-by-step setup (creating the upload key, `gradle.properties`, `signingConfigs`) is in Day 4: [Generate a signed AAB](../day-4/28-signed-aab-release-build.md). This file explains the concepts around it.

## Keystore, key, alias

```
my-upload-key.keystore          a file (password protected)
 └─ alias "my-key-alias"        one key inside it (its own password)
     ├─ private key             signs the app. Never share it
     └─ certificate             public part, embedded in the app
```

- A **keystore** is a password-protected file that can hold several keys.
- Each key has an **alias** (its name) and its own password.
- Build tools need four things: the keystore file, its password, the alias, and the key password.

## The keys you'll meet

| Key | Where | Used for |
|---|---|---|
| **Debug key** | `android/app/debug.keystore` (in the template, password `android`) | Debug builds. Public, everyone has the same one |
| **Upload key** | Created by you, kept secret | Signing AABs you upload to Google Play |
| **App signing key** | Kept by Google (Play App Signing) | Signing the APKs users actually install |

```
you  ─sign with upload key──►  Google Play  ─re-sign with app signing key──►  users
```

Because users get the app signed by **Google's** key, anything that checks "is this my app?" in production must know Google's key, not only yours.

## Fingerprints (SHA-1 / SHA-256)

A **fingerprint** is a short hash of a key's certificate. Services use it to recognize your app:

| Service | Needs fingerprints for |
|---|---|
| Firebase (Google sign-in, phone auth, App Check) | SHA-1 and SHA-256 ([Firebase per environment](06-firebase-per-environment.md)) |
| Google Maps / Google Cloud API key restrictions | SHA-1 |
| Android App Links `assetlinks.json` | SHA-256 (Day 4: [App Links](../day-4/14-url-schemes-app-links.md)) |

Getting them:

```bash
# all variants of the project
cd android && ./gradlew signingReport

# one keystore
keytool -list -v -keystore my-upload-key.keystore -alias my-key-alias
```

The **app signing key** fingerprint is in Play Console -> App integrity.

Register **all three** (debug, upload, app signing). A classic bug: Google sign-in works in debug and in your local release build, but fails for users who installed from Play, because only Google's key fingerprint was missing.

## Keeping keys safe

- **Never commit** the keystore or passwords. Add `*.keystore` (except `debug.keystore`) to `.gitignore`.
- Store passwords in `~/.gradle/gradle.properties` locally, and in CI secrets for automated builds. A common CI trick: store the keystore as a base64 string secret and decode it during the build.
- **Back it up** (password manager, company vault). With Play App Signing a lost upload key can be reset by Google, but it takes time and blocks releases until then.

## "Signatures don't match"

Installing a build signed with a different key over an existing app fails:

```
INSTALL_FAILED_UPDATE_INCOMPATIBLE: Package com.app signatures do not match
```

This happens when you install a release build over a debug build (or the Play version over a local one). Uninstall first: `adb uninstall com.app`. Different app IDs per environment avoid most of these clashes ([Different bundle IDs](05-bundle-ids-per-environment.md)).

## iOS

iOS also signs every build, but with **certificates and provisioning profiles** managed through Apple's developer portal instead of a keystore file. See [iOS signing, Archive and TestFlight](12-ios-signing-archive-testflight.md).

## Rule of thumb

A keystore is a password-protected file of keys; the upload key signs what you send to Play, and Google re-signs with the app signing key. Register the SHA fingerprints of the debug, upload, and app signing keys wherever Google services check them, keep keystores out of git, and back them up.
