# Biometric authentication

Biometric authentication lets the user unlock the app with **Face ID, Touch ID, or a fingerprint** instead of typing a password every time.

Two facts shape everything below:

- **The app never sees the face or the fingerprint.** Matching happens inside secure hardware (the Secure Enclave on iOS, the TEE on Android). The app only learns the answer: matched or not.
- **Biometrics don't replace the first login.** The user signs in with a password once, the app gets tokens, and biometrics then protect access to those tokens on this device.

## Two ways to use it

### 1. A "yes/no" prompt (weak)

Libraries such as `react-native-biometrics` or `expo-local-authentication` can show the system prompt and return a boolean:

```ts
const { success } = await LocalAuthentication.authenticateAsync({
  promptMessage: 'Unlock the app',
});

if (success) {
  showApp();
}
```

The decision is an `if` statement in your JS. On a rooted or jailbroken phone, a tool like Frida can hook the call and make it always return `success: true`. Nothing secret was protected, only a screen.

This is acceptable for low-risk things, for example hiding a screen until the user confirms it's them.

### 2. A secret locked behind biometrics (strong)

Store the refresh token in the Keychain/Keystore with an **access control** that requires biometrics. The OS will only decrypt it after a successful match:

```ts
import * as Keychain from 'react-native-keychain';

await Keychain.setGenericPassword('refresh', refreshToken, {
  service: 'auth-biometric',
  accessControl: Keychain.ACCESS_CONTROL.BIOMETRY_CURRENT_SET,
  accessible: Keychain.ACCESSIBLE.WHEN_PASSCODE_SET_THIS_DEVICE_ONLY,
});
```

Later, reading it shows the system prompt:

```ts
try {
  const result = await Keychain.getGenericPassword({
    service: 'auth-biometric',
    authenticationPrompt: { title: 'Unlock with biometrics' },
  });
  if (result) {
    await refreshSession(result.password);
  }
} catch (e) {
  // user cancelled, too many attempts, or biometrics changed
  showPasswordLogin();
}
```

What happens underneath:

```
getGenericPassword
 -> native module
 -> OS shows Face ID / BiometricPrompt
 -> secure hardware matches the biometric
      match    -> hardware releases the key -> value decrypted -> returned to JS
      no match -> key stays locked          -> error thrown in JS
```

There is no boolean in JS to fake. Without a real match, the key that decrypts the token is never used, so the token cannot be read. This is the approach to use for logging in.

## Access control options

| Option | Meaning |
|---|---|
| `BIOMETRY_CURRENT_SET` | Only the biometrics enrolled **right now**. Adding a new fingerprint or face makes the item unreadable. Safest. |
| `BIOMETRY_ANY` | Any enrolled biometric, including ones added later. |
| `BIOMETRY_ANY_OR_DEVICE_PASSCODE` | Biometrics, or the phone's PIN/passcode as a fallback. |
| `DEVICE_PASSCODE` | Phone PIN/passcode only. |

`BIOMETRY_CURRENT_SET` matters because of a simple attack: someone who knows the phone's PIN adds their own fingerprint, then opens your app. With `CURRENT_SET`, that new fingerprint cannot unlock the old secret.

## Checking support first

Not every device has biometrics, and not every user has enrolled them:

```ts
const type = await Keychain.getSupportedBiometryType();
// FaceID, TouchID, Fingerprint, Face, Iris, or null
```

If it returns `null`, don't offer the option. Use `type` to show the right wording in the UI ("Use Face ID" vs "Use fingerprint").

## Platform setup

- **iOS**: add `NSFaceIDUsageDescription` to `Info.plist`. Without it, the app crashes when Face ID is requested.
- **Android**: the prompt is the system `BiometricPrompt`. Only **Class 3 (strong)** biometrics can unlock Keystore keys. Some face unlock implementations are Class 2 (weak) and won't be offered for this.

## The user flow

```
first login with password
 -> tokens received
 -> "Enable Face ID for next time?"
      yes -> store refresh token with biometric access control
      no  -> store it normally (see 05-token-storage.md)

next launch
 -> biometric prompt
      success                          -> refresh session -> app
      cancel / failure / item invalid  -> password login
```

Things to handle:

- **Always keep a password fallback.** Biometrics can fail: wet fingers, a mask, a lockout after too many attempts, a changed enrollment.
- **Make it opt-in.** Ask after a successful login, and let the user turn it off in settings (which deletes the biometric-protected item).
- **On logout**, delete the biometric item too, otherwise the next launch prompts for an account that is logged out.

## Rule of thumb

A biometric prompt that only returns `true` protects a screen. A secret stored with biometric access control protects the account. For login, use the second one.
