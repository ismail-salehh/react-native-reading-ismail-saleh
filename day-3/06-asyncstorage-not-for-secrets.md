# Why AsyncStorage is not for secrets

AsyncStorage is a good place for preferences, cached data, and flags. It is the wrong place for passwords, tokens, API keys, or personal data. The reason is simple: **it writes plain text to a file**, and several kinds of attacker can read that file.

## Where the data actually goes

AsyncStorage has no storage of its own. It is a JS API over a native module that writes to normal app files:

```
AsyncStorage.setItem('token', 'eyJhbGci...')
 -> native module
      iOS:     JSON manifest file (and one file per large value)
               in the app's Application Support directory
      Android: SQLite database (RKStorage)
               in /data/data/<package>/databases/
```

Nothing in this path encrypts the value. Open the file and the token is right there, exactly as you wrote it.

## Who can read it

The app sandbox stops **other apps** from reading these files on a normal, locked-down phone. That is the only protection. It does not help in these cases:

- **Rooted or jailbroken devices.** Root ignores the sandbox. Any process running as root, including malware, can read every app's files.
- **Physical access.** Forensic tools and debugging tools can pull app data from a phone someone else has in their hands.
- **Backups.** Depending on configuration, app files can be included in iCloud/iTunes backups or Android Auto Backup. The backup then sits on another machine or a cloud account.
- **Debuggable builds.** On Android, `adb shell run-as <package>` gives access to a debuggable app's files. A debug or staging build that leaks to testers exposes everything inside it.
- **Other JS in your own app.** Every library in your bundle can call `AsyncStorage.getAllKeys()` and `multiGet()`. A compromised or malicious npm package needs no special permission to read your tokens.

## What secure storage does differently

The Keychain (iOS) and Keystore (Android) don't just "hide" the file. They change **who holds the key**:

| | AsyncStorage | Keychain / Keystore |
|---|---|---|
| Value on disk | Plain text | Encrypted |
| Encryption key | None | Held by the OS, often in secure hardware (Secure Enclave, TEE, StrongBox) |
| Key can be copied out | n/a | No, it never leaves the secure hardware |
| Access check | File permissions only | OS checks the app's identity, and optionally device lock or biometrics |
| Included in backups | Possibly | Controlled with flags such as `THIS_DEVICE_ONLY` |

So even if an attacker copies the encrypted file, they have bytes they cannot decrypt without the device's hardware-held key.

Secure storage doesn't make a rooted phone safe. Malware with root can still hook into the running app and read secrets from memory after they are decrypted. It does raise the cost: a copied file or a backup is no longer enough.

## A quick test

Ask: **"If this value showed up in a public paste, would it hurt?"**

| Value | Where it goes |
|---|---|
| Theme, language, onboarding seen, last selected tab | AsyncStorage |
| Cached list of public products | AsyncStorage (or the query cache) |
| Refresh token, access token | Keychain / Keystore |
| Password, PIN | Don't store it. Store the token you got by using it. |
| Personal data (ID numbers, health data) | Keychain / Keystore, or better, don't keep it on the device |

## The same rule for similar libraries

The problem is not the AsyncStorage brand. It is any storage that writes plain text:

- **MMKV** is faster, but unencrypted by default. It supports an `encryptionKey`, but then you have to store that key somewhere safe, which brings you back to the Keychain.
- **redux-persist** and Zustand's `persist` middleware usually write to AsyncStorage. A token in your persisted store is a token in AsyncStorage.
- **SQLite** files are plain unless you use an encrypted variant such as SQLCipher.

## Rule of thumb

AsyncStorage is for data you would be fine with someone reading. Anything that grants access or identifies a person goes in the Keychain/Keystore through [react-native-keychain](04-react-native-keychain.md).
