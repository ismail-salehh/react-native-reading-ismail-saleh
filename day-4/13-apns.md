# APNs (reading only)

**APNs (Apple Push Notification service)** is Apple's system for delivering pushes to iPhones and iPads. **Every** push to an iOS device goes through APNs, no matter what you use on the server. When you use FCM for iOS, FCM is a middleman that forwards your message to APNs.

```
your backend -> FCM -> APNs -> iPhone
your backend -> APNs  ------> iPhone   (sending directly, without FCM)
```

## Why it matters if you use FCM

FCM can only forward to APNs if Apple trusts it to send on behalf of your app. That trust is set up by you, once, by uploading an **APNs key** to Firebase. If this step is missing, Android pushes work and iOS pushes silently never arrive.

## The setup, step by step

**1. Apple Developer account (paid)** is required. Push doesn't work with a free account.

**2. Create an APNs Authentication Key** (Certificates, Identifiers & Profiles -> Keys -> "Apple Push Notifications service"):

- You download a `.p8` file **once**. Store it safely; Apple won't let you download it again.
- Note the **Key ID** and your **Team ID**.
- One key works for all your apps and for both environments.

The older alternative is a per-app `.p12` **certificate**, which expires every year. The `.p8` key doesn't expire and is the recommended option.

**3. Upload the key to Firebase**: Project settings -> Cloud Messaging -> Apple app configuration -> APNs Authentication Key (with Key ID and Team ID).

**4. Enable capabilities in Xcode** (app target -> Signing & Capabilities):

- **Push Notifications**
- **Background Modes** -> "Remote notifications" (needed for background/data messages)

These go into the app's entitlements and provisioning profile.

**5. Rebuild and run on a device.**

## The two tokens on iOS

```
iPhone registers with APNs  -> APNs device token   (hex string, Apple's address)
Firebase SDK sends it to FCM -> FCM registration token (what your backend uses)
```

With FCM you normally only use the FCM token ([FCM and the device token](09-fcm-device-token.md)). The APNs token is used under the hood. If `getToken()` fails on iOS with an error about the APNs token, APNs registration didn't work: check the capability, the provisioning profile, and that you're on a device.

## Sandbox vs production

APNs has two environments:

| Environment | Used by |
|---|---|
| **Sandbox (development)** | Builds run from Xcode with a development profile |
| **Production** | TestFlight and App Store builds |

A token from one environment doesn't work in the other. FCM handles this for you based on the build. When sending to APNs directly, the server must use the right one, which is a common reason pushes work in development and not in TestFlight (or the opposite).

## Payload basics

APNs payloads have an `aps` dictionary that iOS understands. Through FCM you put it under `apns.payload`:

```json
{
  "apns": {
    "payload": {
      "aps": {
        "alert": { "title": "Order shipped", "body": "Order #812 is on its way" },
        "badge": 3,
        "sound": "default",
        "content-available": 1
      }
    }
  }
}
```

- `alert`: the visible text. `badge`: the number on the app icon. `sound`: plays a sound.
- `content-available: 1`: wake the app in the background (silent push). iOS throttles these.
- `mutable-content: 1`: lets a Notification Service Extension modify the notification (add images, decrypt).
- Max payload size is 4 KB.

## Testing

- Real devices always work. Simulators on Apple-silicon Macs (Xcode 14+, iOS 16+) can receive remote pushes too.
- You can also drag an `.apns` JSON file onto the Simulator, or use `xcrun simctl push`, to test how the app *handles* a notification without a server.

## Rule of thumb

Every iOS push goes through APNs. With FCM, you set it up once: a `.p8` key uploaded to Firebase, plus the Push Notifications and Background Modes capabilities in Xcode. If iOS pushes don't arrive but Android ones do, check this setup first.
