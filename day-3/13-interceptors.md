# Interceptors

An interceptor is a function that Axios runs on **every** request before it is sent, or on **every** response before your code gets it. It is the place for logic that would otherwise be copied into every API call.

```
your code -> request interceptors -> network -> response interceptors -> your code
```

## Request interceptors

They receive the request config and must **return** it (changed or not):

```ts
import { api } from './client';
import { getAccessToken } from '../auth/tokens';

api.interceptors.request.use((config) => {
  const token = getAccessToken();
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```

Now no screen has to remember to attach the token. `getAccessToken` is the in-memory token from [Token storage](05-token-storage.md). It is a sync read, which is why the access token is kept in memory and not read from the Keychain each time.

Other common uses:

- Add `Accept-Language` from the current locale (Day 2: localization).
- Add an app version or device id header.
- Start a timer for logging.

## Response interceptors

They take **two** functions: one for success (2xx) and one for errors:

```ts
api.interceptors.response.use(
  (response) => response,          // 2xx: must return the response
  (error) => Promise.reject(error), // non-2xx / network: must reject (or recover)
);
```

The error handler has three choices:

| Return | Effect |
|---|---|
| `Promise.reject(error)` (or `throw`) | The caller's `catch` runs |
| A new request, e.g. `return api(error.config)` | The caller gets **that** result instead, as if nothing failed |
| A value | The caller's `await` resolves with it. Rarely what you want |

The second option is what makes [automatic token refresh](15-automatic-token-refresh.md) and [retries](16-retry-failed-requests.md) possible: the screen never knows the first attempt failed.

## A logging interceptor for development

```ts
if (__DEV__) {
  api.interceptors.request.use((config) => {
    console.log('→', config.method?.toUpperCase(), config.url);
    return config;
  });
  api.interceptors.response.use(
    (res) => {
      console.log('←', res.status, res.config.url);
      return res;
    },
    (error) => {
      console.log('✕', error.response?.status ?? error.code, error.config?.url);
      return Promise.reject(error);
    },
  );
}
```

Never log headers or bodies in release builds: they contain tokens and personal data.

## Order

When you register several interceptors:

- **Request** interceptors run in **reverse** order of registration (last added runs first).
- **Response** interceptors run in **the same** order as registration.

```
register: request A, request B, response C, response D

B -> A -> network -> C -> D
```

This matters when one interceptor depends on another, for example a retry interceptor and a token-refresh interceptor. Register them in one file so the order is obvious.

## Removing an interceptor

`use` returns an id:

```ts
const id = api.interceptors.request.use(addAuthHeader);
api.interceptors.request.eject(id);
```

Useful in tests, or when an interceptor depends on something that can be torn down.

## Mistakes to watch for

- **Forgetting to return.** A request interceptor that doesn't return `config`, or a response interceptor that doesn't return `response`, breaks every request.
- **Swallowing errors.** An error handler that doesn't rethrow makes every failure look like a success with `undefined` data.
- **Infinite loops.** An interceptor that re-sends a request which fails the same way again, forever. Always mark retried requests (e.g. `config._retry = true`) and check the mark.
- **Interceptors on the global `axios`.** Add them to your instance (`api`), not the default `axios` export, so third-party code and the refresh call are not affected.

## Rule of thumb

If every request needs it, it goes in an interceptor. Request interceptors add things. Response interceptors classify, recover, or rethrow, and they always return or reject.
