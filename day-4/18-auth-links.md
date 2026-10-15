# Handling authentication links

Two problems come up when deep links meet authentication:

1. **A link to a protected screen while the user is logged out.** `myapp://orders/812` arrives, but the order screen isn't available until login.
2. **Links that *are* authentication**: password reset, email verification, magic login links, OAuth redirects. They carry a token in the URL.

## 1. Protected links while logged out

The usual auth setup renders different screens depending on the session (Day 1: [Auth navigation](../day-1/27-auth-navigation.md)):

```tsx
<Stack.Navigator>
  {isSignedIn ? (
    <>
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen name="OrderDetails" component={OrderDetailsScreen} />
    </>
  ) : (
    <Stack.Screen name="SignIn" component={SignInScreen} />
  )}
</Stack.Navigator>
```

When logged out, `OrderDetails` **doesn't exist** in the navigator, so a link to it has nowhere to go. Without extra work, the link is lost: after login the user lands on `Home` and has to find the order themselves.

### Remember the link, open it after login

```ts
// src/linking/pendingLink.ts
let pendingUrl: string | null = null;

export const setPendingUrl = (url: string | null) => { pendingUrl = url; };
export const takePendingUrl = () => {
  const url = pendingUrl;
  pendingUrl = null;
  return url;
};
```

In the linking config, keep links that arrive while logged out:

```ts
const linking: LinkingOptions<RootStackParamList> = {
  prefixes: ['myapp://', 'https://example.com'],
  config,
  async getInitialURL() {
    const url = await Linking.getInitialURL();
    if (url && !useAuthStore.getState().isSignedIn && isProtected(url)) {
      setPendingUrl(url);
      return null; // start on the sign-in screen
    }
    return url;
  },
  subscribe(listener) {
    const sub = Linking.addEventListener('url', ({ url }) => {
      if (!useAuthStore.getState().isSignedIn && isProtected(url)) {
        setPendingUrl(url);
        return;
      }
      listener(url);
    });
    return () => sub.remove();
  },
};
```

After a successful login, once the signed-in screens are rendered:

```ts
const url = takePendingUrl();
if (url) Linking.openURL(url); // goes through the linking config again
```

Notes:

- **Session restore takes time.** On a cold start, `isSignedIn` may not be known yet while the refresh token is read from the Keychain ([Token storage](../day-3/05-token-storage.md)). Keep the splash screen (and don't render the navigator) until the auth state is known, otherwise every link looks like "logged out".
- Check the React Navigation docs for your version: newer versions have built-in options for links to screens that become available later, which can replace this manual code.

## 2. Links that carry a token

Examples:

```
https://example.com/reset-password?token=8f3a...
https://example.com/verify-email?token=c71d...
https://example.com/login/magic?token=19be...
```

The flow:

```
link opens app -> ResetPassword screen with { token }
 -> POST /auth/reset-password { token, newPassword }
     200 -> logged in, or "password changed, please sign in"
     400 / 410 -> "This link has expired. Send a new one."
```

Rules for these links:

- **Use https Universal Links / App Links, not a custom scheme.** Any app can register `myapp://` and receive the token ([URL schemes and App Links](14-url-schemes-app-links.md)). Verified links can only open your app.
- **The token is short-lived and single-use.** The server enforces that. The app must handle "expired" and "already used" with a clear message and a way to request a new link.
- **Exchange it immediately and forget it.** Send it to the API, then navigate with `reset` so the token isn't kept in the navigation state (which may be persisted) or in the back stack.
- **Don't log the URL.** Analytics and crash reporters often record screen params and URLs.
- **The website must handle the same link** for users who open it on a computer or without the app installed.

## 3. OAuth / social login redirects

"Sign in with Google" style flows open a browser, and the provider redirects back to the app with a code:

```
app -> system browser: https://accounts.example.com/authorize?...&code_challenge=...
user signs in
provider -> redirect: myapp://oauth/callback?code=abc
app -> exchanges code + code_verifier for tokens
```

- Use the **system browser** (`ASWebAuthenticationSession` / Custom Tabs), not a WebView. Libraries like `react-native-app-auth` do this.
- Use **PKCE** (`code_challenge` / `code_verifier`). It makes a stolen redirect `code` useless, which matters because the redirect often uses a custom scheme.
- The redirect is handled by the library, so **exclude it from your React Navigation linking config**.

## Rule of thumb

If the user isn't signed in, save the protected link and open it after login. Links that carry tokens use verified https links, are exchanged right away and then forgotten, and handle expired tokens clearly. OAuth uses the system browser with PKCE.
