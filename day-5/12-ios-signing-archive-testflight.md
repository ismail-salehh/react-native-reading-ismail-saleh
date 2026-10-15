# Certificates, provisioning profiles, signing, entitlements, Archive and TestFlight (reading only)

iOS only runs apps that Apple can trace back to a known developer. Every build that runs on a real device is **signed**, and the signature must match a **provisioning profile** that allows that app, on that device, with those capabilities. It's the iOS version of [Signing and keystore](08-signing-keystore.md), with more pieces.

A **paid Apple Developer account** is needed for distribution, push notifications, and most capabilities.

## The pieces

```
Certificate            WHO signed it          (your team, with a private key in your Mac's Keychain)
App ID                 WHICH app              (com.app + its capabilities)
Devices                WHERE it may run       (only for development / ad hoc)
Entitlements           WHAT it may do         (push, associated domains, ...)
        │
        └──► Provisioning profile = certificate + App ID + (devices) + entitlements
```

| Piece | What it is |
|---|---|
| **Certificate** | Proves the build comes from your team. **Development** for running on your devices, **Distribution** for TestFlight/App Store. The private key lives in the Keychain of the Mac that created it |
| **App ID** | The bundle ID registered in the developer portal, with the capabilities it's allowed to use |
| **Devices** | Registered device UDIDs. Development and ad hoc builds run only on these |
| **Provisioning profile** | A file that ties it all together. Embedded in the app; iOS checks it at install and launch |
| **Entitlements** | The capabilities the build claims, in `MyApp.entitlements` |

## Provisioning profile types

| Type | Runs on | Used for |
|---|---|---|
| **Development** | Registered devices | Running from Xcode on your phone |
| **Ad Hoc** | Up to 100 registered devices per type | Sharing a build outside the store (Firebase App Distribution) |
| **App Store** | Any device, through Apple | TestFlight and App Store |
| Enterprise | Company devices | In-house apps, special program |

## Automatic vs manual signing

- **Automatic** (Signing & Capabilities -> "Automatically manage signing"): Xcode creates certificates, App IDs, and profiles for you. Best for most developers.
- **Manual**: you pick each profile. Common on CI and in teams that share certificates with tools like **fastlane match** (certificates and profiles stored encrypted in a git repo, installed on every machine).

## Entitlements

Turning on a capability in Xcode (Signing & Capabilities -> + Capability) writes it to the `.entitlements` file:

| Capability | Entitlement | Used for |
|---|---|---|
| Push Notifications | `aps-environment` | Push (Day 4: [APNs](../day-4/13-apns.md)) |
| Associated Domains | `com.apple.developer.associated-domains` | Universal Links (Day 4: [Universal Links](../day-4/15-universal-links.md)) |
| Keychain Sharing | `keychain-access-groups` | Sharing Keychain items between your apps |
| App Groups | `com.apple.security.application-groups` | Sharing data with extensions |
| Sign in with Apple | `com.apple.developer.applesignin` | Apple login |

The capability must be enabled for the **App ID** and included in the **profile**. If not, the build fails with *"Provisioning profile doesn't include the X entitlement"*. With automatic signing Xcode fixes this for you; with manual signing, regenerate the profile.

## Version and build number

| Setting | Info.plist key | Example | Rule |
|---|---|---|---|
| Version (`MARKETING_VERSION`) | `CFBundleShortVersionString` | `1.4.0` | Shown to users |
| Build (`CURRENT_PROJECT_VERSION`) | `CFBundleVersion` | `52` | Must increase for every upload with the same version |

Like Android's `versionName` / `versionCode` (Day 4: [Signed AAB](../day-4/28-signed-aab-release-build.md)).

## Archive and upload

An **archive** is a release build packaged for distribution.

```
1. Select the scheme (e.g. MyApp) and the destination "Any iOS Device (arm64)"
2. Product -> Archive                (uses the scheme's Archive configuration: Release)
3. Organizer opens -> Distribute App -> App Store Connect -> Upload
4. Xcode re-signs with a distribution certificate and App Store profile, and uploads
5. App Store Connect processes the build (minutes to an hour)
```

The app record must exist in **App Store Connect** first, with the same bundle ID. On CI the same steps run with `xcodebuild archive` / `-exportArchive` or fastlane.

Common upload blockers: build number not increased, missing icon sizes, a usage description missing for an API a library links, and the export compliance question (set `ITSAppUsesNonExemptEncryption` to `false` in Info.plist if you only use standard HTTPS).

## TestFlight

TestFlight is Apple's beta testing service. Testers install the **TestFlight** app and get your builds there.

| | Internal testers | External testers |
|---|---|---|
| Who | Members of your App Store Connect team | Anyone with an email or a public link |
| Limit | 100 | 10,000 |
| Review | None, available after processing | **Beta App Review** for the first build of a version |
| Typical use | Team, QA | Clients, beta users |

- Builds expire after **90 days**.
- Push notifications in TestFlight use the **production** APNs environment, unlike builds run from Xcode.
- It's the iOS equivalent of Play internal/closed testing (Day 4: [Play Store internal testing](../day-4/29-play-store-internal-testing.md)).

## Rule of thumb

A certificate says who signed, the App ID says which app, entitlements say what it may do, and the provisioning profile ties them together (plus devices for development). Let Xcode manage signing unless your team shares certificates. To ship: raise the build number, Archive with the Release configuration, upload to App Store Connect, and test through TestFlight.
