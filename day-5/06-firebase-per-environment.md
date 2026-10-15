# Firebase configuration per environment

Day 4 set up Firebase for one app ([Firebase project setup](../day-4/08-firebase-setup.md)). With three environments and three app IDs ([Different bundle IDs](05-bundle-ids-per-environment.md)), each build needs **its own** Firebase config file. If dev builds use the production config, test events land in production analytics, test crashes in production Crashlytics, and pushes can go to the wrong app.

## One project per environment

```
Firebase project myapp-dev     → Android app com.app.dev    + iOS app com.app.dev
Firebase project myapp-stage   → Android app com.app.stage  + iOS app com.app.stage
Firebase project myapp-prod    → Android app com.app        + iOS app com.app
```

Separate projects keep data, users, and permissions fully apart (developers can have full access to dev, limited access to prod). Putting all apps in **one** project also works, but analytics, Crashlytics, and the user list are then shared.

Each project gives you one `google-services.json` and one `GoogleService-Info.plist` per app.

## Android: one file per flavor folder

```
android/app/src/
 ├─ dev/google-services.json      (package_name: com.app.dev)
 ├─ stage/google-services.json    (package_name: com.app.stage)
 └─ prod/google-services.json     (package_name: com.app)
```

The Google Services Gradle plugin looks in the **variant's** folders first, so `stageRelease` uses the stage file. No script needed. Remove the old `android/app/google-services.json` so it's never picked by mistake.

If the build fails with *"No matching client found for package name 'com.app.dev'"*, the file in that folder belongs to another app.

## iOS: copy the right plist at build time

Xcode has no flavor folders, so a **Run Script build phase** copies the right file into the built app.

1. Store the files outside the target (don't add them to "Copy Bundle Resources"):

```
ios/Firebase/Dev/GoogleService-Info.plist
ios/Firebase/Stage/GoogleService-Info.plist
ios/Firebase/Prod/GoogleService-Info.plist
```

2. Target -> Build Phases -> + New Run Script Phase:

```bash
case "${CONFIGURATION}" in
  *Dev*)   ENV=Dev ;;
  *Stage*) ENV=Stage ;;
  *)       ENV=Prod ;;
esac

cp "${PROJECT_DIR}/Firebase/${ENV}/GoogleService-Info.plist" \
   "${BUILT_PRODUCTS_DIR}/${PRODUCT_NAME}.app/GoogleService-Info.plist"
```

`CONFIGURATION` is the build configuration the scheme uses (`Debug-Dev`, `Release-Stage`, `Release`, ...).

## Per-project setup that's easy to forget

Every Firebase project is configured separately:

| Setting | Why |
|---|---|
| APNs key uploaded (Cloud Messaging) | Otherwise iOS pushes don't arrive for that environment (Day 4: [APNs](../day-4/13-apns.md)) |
| SHA-1 / SHA-256 fingerprints of the Android app | Needed for Google sign-in, phone auth, App Check. Add the debug key, upload key, and Play app signing key ([Signing and keystore](08-signing-keystore.md)) |
| Auth providers, authorized domains | Enabled per project |
| Backend service account | The server sending pushes must use the **matching** project's credentials |

The backend of each environment must send pushes through **its** Firebase project. Device tokens from `myapp-dev` don't work with `myapp-prod` credentials.

## Checking which project a build uses

- Log the project ID once at startup in non-prod builds:

```ts
import { getApp } from '@react-native-firebase/app';

if (env.name !== 'prod') console.log('Firebase project:', getApp().options.projectId);
```

- Send a test push from the right project's console to that build's token.
- Check that the event or crash appears in the expected project.

## Rule of thumb

One Firebase project per environment, one config file per app. On Android put each `google-services.json` in its flavor folder; on iOS copy the right `GoogleService-Info.plist` with a build script based on the configuration. Then repeat the per-project setup (APNs key, SHA fingerprints, server credentials) for every environment.
