# .env and different API URLs

Each environment talks to its own backend: dev builds to `https://dev.api.example.com`, staging to `https://stage.api.example.com`, production to `https://api.example.com`. The URL (and similar settings) should come from the **build**, not be edited by hand before each release.

## Why not `__DEV__`?

```ts
const API_URL = __DEV__ ? 'https://dev.api.example.com' : 'https://api.example.com';
```

`__DEV__` only knows debug vs release. It can't express "a **release** build pointing to **staging**", which is exactly what QA needs. Environments and build types are different things ([Build variants](03-build-variants-product-flavors.md)).

## The .env files

One file per environment, in the project root:

```
.env.dev
.env.stage
.env.prod
```

```bash
# .env.stage
API_URL=https://stage.api.example.com
ENV_NAME=stage
SENTRY_ENVIRONMENT=staging
```

The common library is **`react-native-config`**. It reads the chosen file **at build time** and exposes the values to JS and to native code.

```ts
import Config from 'react-native-config';

export const api = axios.create({ baseURL: Config.API_URL });
```

## Choosing the file per build

**Android**: map each variant to a file in `android/app/build.gradle`, **before** the `apply from` line:

```gradle
project.ext.envConfigFiles = [
    devdebug: ".env.dev",
    devrelease: ".env.dev",
    stagedebug: ".env.stage",
    stagerelease: ".env.stage",
    proddebug: ".env.prod",
    prodrelease: ".env.prod",
]
apply from: project(':react-native-config').projectDir.getPath() + "/dotenv.gradle"
```

Keys are the variant names in lowercase. Because the app ID differs from the code's package (`com.app.dev` vs `com.app`), also add this in `defaultConfig`:

```gradle
resValue "string", "build_config_package", "com.app"
```

**iOS**: each scheme copies its file before building (scheme -> Edit Scheme -> Build -> Pre-actions):

```bash
cp "${PROJECT_DIR}/../.env.stage" "${PROJECT_DIR}/../.env"
```

Or from the command line: `ENVFILE=.env.stage npx react-native run-ios`.

Check the library's README for the current setup. It changes between versions.

## Typing and validating

Every value is a **string** (`'true'`, not `true`), and a missing key is `undefined`. Read them in one file, check them once, and import from there:

```ts
// src/config.ts
import Config from 'react-native-config';

function required(name: string): string {
  const value = Config[name];
  if (!value) throw new Error(`Missing env var ${name}`);
  return value;
}

export const env = {
  apiUrl: required('API_URL'),
  name: required('ENV_NAME') as 'dev' | 'stage' | 'prod',
  analyticsEnabled: Config.ANALYTICS_ENABLED === 'true',
};
```

A build with a missing URL then fails loudly at startup instead of sending requests to `undefined/login`.

## .env is not secret

Everything in `.env` is **compiled into the app**. Anyone can unzip an APK and read it. API URLs, public keys, and feature flags are fine. Server secrets (database passwords, private API keys, the Firebase service account) must **never** be in it. Those stay on the server (Day 3: [Token storage](../day-3/05-token-storage.md)).

- Commit a `.env.example` with every key and fake values, so new developers know what's needed.
- If your `.env` files contain anything you don't want public in git, ignore them and inject them in CI.

## Changes need a rebuild

The values are read at build time. After editing a `.env` file, **rebuild** the app; a Metro reload isn't enough.

## Show the environment in the app

Put the environment name on the login screen or in a settings "About" row (`Stage · 1.4.0 (52)`) in non-prod builds. Bug reports then say which backend they happened on.

## Rule of thumb

One `.env` file per environment, chosen by the build variant (Android) or scheme (iOS), read through one typed config module. Use it for URLs and public settings, never for secrets, and rebuild after changing it.
