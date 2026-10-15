# Universal Links (reading only)

Universal Links are iOS's version of Android App Links ([URL schemes and Android App Links](14-url-schemes-app-links.md)): normal `https://` URLs that open your app if it's installed, and your website if it isn't. Apple verifies that the domain and the app belong to the same owner, so no other app can take them over.

## How it works

```
app installed
 -> iOS reads the app's Associated Domains: applinks:example.com
 -> iOS downloads https://example.com/.well-known/apple-app-site-association
 -> the file lists your app ID and which paths it handles
 -> verified: iOS remembers "example.com/orders/* -> this app"

user taps https://example.com/orders/812
 -> app installed and path matches -> app opens with the URL
 -> otherwise                      -> Safari opens the page
```

## The two pieces

**1. The Associated Domains capability** in Xcode (Signing & Capabilities -> + Capability -> Associated Domains):

```
applinks:example.com
applinks:www.example.com
```

Each subdomain is listed separately. This ends up in the app's `.entitlements` file and must be enabled for the App ID in the Apple Developer portal.

**2. The apple-app-site-association (AASA) file**, served at `https://example.com/.well-known/apple-app-site-association`:

```json
{
  "applinks": {
    "details": [
      {
        "appIDs": ["ABCDE12345.com.example.myapp"],
        "components": [
          { "/": "/orders/*" },
          { "/": "/reset-password", "?": { "token": "?*" } },
          { "/": "/admin/*", "exclude": true }
        ]
      }
    ]
  }
}
```

- `appIDs`: your **Team ID** + bundle ID.
- `components`: which paths open the app. Paths not listed stay on the website.
- Rules for the file: **no `.json` extension**, served over **HTTPS** with a valid certificate, `Content-Type: application/json`, **no redirects**.

**3. Forward the link to React Native** in `AppDelegate` with `RCTLinkingManager` (the `continueUserActivity` method). The React Navigation and React Native `Linking` docs have the exact code for your React Native version. After that, the URL arrives in JS like any other link.

## Things that surprise people

- **Apple caches the AASA file** through its own CDN. Changes can take time to reach devices. During development, you can add `?mode=developer` to the domain entry (`applinks:example.com?mode=developer`) and enable Associated Domains Development in the device's developer settings to fetch it directly.
- **Typing or pasting the URL in Safari's address bar doesn't open the app.** Universal Links trigger when the user taps a link (in Notes, Mail, Messages, a web page on another domain).
- **Links on the same domain** (a link on `example.com` to `example.com/orders`) usually stay in Safari.
- **The user can turn it off.** If they choose "Open in Safari" from the banner, iOS keeps opening that domain in Safari until they long-press a link and choose "Open in MyApp".
- **In-app browsers** (some social and chat apps) may not trigger Universal Links at all.

## Debugging checklist

1. Does `https://example.com/.well-known/apple-app-site-association` load in a browser, without a redirect, as JSON?
2. Does `appIDs` match `TEAMID.bundle.id` exactly?
3. Is the Associated Domains capability on, with `applinks:` and the right host?
4. Is the path covered by `components`?
5. Reinstall the app (iOS fetches the file at install time).
6. Test by tapping a link in Notes or Messages, not by typing it in Safari.

## Rule of thumb

Universal Links need both sides: the Associated Domains capability in the app, and a valid AASA file on the domain. Once both are right, a `https://` link opens the app when it's installed and the website when it isn't.
