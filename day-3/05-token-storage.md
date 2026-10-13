# Access and refresh tokens: where to store them

After login, the server usually gives the app two tokens:

- **Access token**: short-lived (minutes). Sent with every API request in the `Authorization` header.
- **Refresh token**: long-lived (days or weeks). Used only to get a new access token when the old one expires.

How the refresh flow works is covered in [Access and refresh tokens](14-access-and-refresh-tokens.md) and [Automatic token refresh](15-automatic-token-refresh.md). This file answers one question: **where on the device should each token live?**

## The options

| Place | Survives app restart | Encrypted at rest | Readable by |
|---|---|---|---|
| JS memory (a variable, a store) | No | Not on disk at all | Only this JS runtime while it runs |
| AsyncStorage | Yes | **No** | Anyone who can read the app's files |
| MMKV / redux-persist / SQLite (default) | Yes | **No** (unless configured) | Anyone who can read the app's files |
| Keychain (iOS) / Keystore (Android) via `react-native-keychain` | Yes | **Yes**, with hardware-backed keys | Only this app, after the OS checks it |

## The recommended setup

```
refresh token  ->  Keychain / Keystore      (persistent, encrypted)
access token   ->  JS memory                 (fast, gone when the app is killed)
                   + optionally Keychain     (to skip one refresh call on cold start)
```

Why this split:

- The **refresh token is the valuable one**. Whoever has it can mint new access tokens for weeks. It must be encrypted at rest, so it goes in secure storage.
- The **access token is used on every request**. Reading secure storage is an async call across the bridge into native code (and on Android, a decryption with a Keystore key). Doing that before every request is slow. So you read it once and keep it in memory.
- If the access token is lost when the app is killed, nothing bad happens. On the next launch you read the refresh token from secure storage and get a fresh access token.

## In code

A small module owns the tokens. The rest of the app never touches storage directly.

```ts
import * as Keychain from 'react-native-keychain';

const SERVICE = 'auth';

let accessToken: string | null = null;

export function getAccessToken() {
  return accessToken;
}

export async function saveTokens(access: string, refresh: string) {
  accessToken = access;
  await Keychain.setGenericPassword('refresh', refresh, {
    service: SERVICE,
    accessible: Keychain.ACCESSIBLE.AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY,
  });
}

export async function getRefreshToken() {
  const result = await Keychain.getGenericPassword({ service: SERVICE });
  return result ? result.password : null;
}

export async function clearTokens() {
  accessToken = null;
  await Keychain.resetGenericPassword({ service: SERVICE });
}
```

What each part does:

- `accessToken` is a module-level variable. It lives on the JS heap and disappears when the process dies.
- `setGenericPassword(username, password)` stores one username/password pair per `service`. We don't have a username, so we put a label there and the refresh token in the password slot.
- `AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY` means the item can be read once the user has unlocked the phone at least once since boot (so a background refresh still works while the screen is locked), and it is **never copied** to backups or to a new phone.
- `getGenericPassword` returns `false` when nothing is stored, so the caller can tell "logged out" apart from an error.

## App start

```
launch
 -> getRefreshToken()
      null      -> show the login screen
      present   -> call /auth/refresh
                     success -> saveTokens(...) -> show the app
                     401     -> clearTokens()  -> show the login screen
```

This is the same check that drives the auth navigation from Day 1: the result decides which navigator is rendered.

## Logout

Logging out must clear **every** copy:

1. `clearTokens()` (memory + Keychain).
2. Clear cached server data, for example `queryClient.clear()`, so the next user doesn't see the previous user's data.
3. Optionally tell the server to revoke the refresh token, so a stolen copy stops working too.

## iOS gotcha: the Keychain survives uninstall

On iOS, Keychain items are **not** deleted when the app is uninstalled. A user who deletes and reinstalls the app can find themselves already logged in. The usual fix uses AsyncStorage as a "first launch" marker, because AsyncStorage *is* deleted with the app:

```ts
const launched = await AsyncStorage.getItem('hasLaunched');
if (!launched) {
  await clearTokens();
  await AsyncStorage.setItem('hasLaunched', 'true');
}
```

This is a good use of AsyncStorage: the flag is not a secret.

## Things to avoid

- Putting tokens in AsyncStorage, MMKV, or a persisted Redux/Zustand store. These are plain files. See [Why AsyncStorage is not for secrets](06-asyncstorage-not-for-secrets.md).
- Logging tokens with `console.log`. Logs end up in Metro, `adb logcat`, crash reporters and screen recordings.
- Putting tokens in navigation params or URLs. They get logged, persisted in navigation state, and passed to deep link handlers.
- Hard-coding API secrets in the JS bundle. The bundle ships inside the app and can be unpacked by anyone. Secure storage protects values the app **receives at runtime**, not values compiled into it.

## Rule of thumb

The refresh token goes in the Keychain/Keystore. The access token lives in memory. Everything that is not a secret (theme, language, "has launched") can stay in AsyncStorage.
