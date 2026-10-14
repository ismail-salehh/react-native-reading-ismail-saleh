# Automatic token refresh

When the access token expires, the next request gets a `401`. Automatic refresh means the app quietly gets a new access token and re-sends the request, so the screen only ever sees the successful result.

The concepts are in [Access and refresh tokens](14-access-and-refresh-tokens.md). This file is the implementation, using a [response interceptor](13-interceptors.md).

## The flow

```
request -> 401
 -> refresh already running?
      yes -> wait for it
      no  -> start POST /auth/refresh
 -> refresh succeeded -> set new token -> re-send original request -> return its result
 -> refresh failed (401) -> clear tokens -> go to login -> reject
```

## The code

```ts
// src/api/refresh.ts
import axios, { AxiosError, InternalAxiosRequestConfig } from 'axios';
import { api } from './client';
import { getRefreshToken, saveTokens, clearTokens } from '../auth/tokens';
import { useAuthStore } from '../auth/store';

// A separate client WITHOUT interceptors, so the refresh call
// can never trigger another refresh.
const authClient = axios.create({ baseURL: api.defaults.baseURL, timeout: 15_000 });

type RetryableConfig = InternalAxiosRequestConfig & { _retry?: boolean };

let refreshPromise: Promise<string> | null = null;

async function refreshAccessToken(): Promise<string> {
  const refreshToken = await getRefreshToken();
  if (!refreshToken) throw new Error('No refresh token');

  const { data } = await authClient.post('/auth/refresh', { refreshToken });
  await saveTokens(data.accessToken, data.refreshToken); // save the rotated refresh token too
  return data.accessToken;
}

// Every caller shares the same in-flight refresh.
function getFreshAccessToken(): Promise<string> {
  if (!refreshPromise) {
    refreshPromise = refreshAccessToken().finally(() => {
      refreshPromise = null;
    });
  }
  return refreshPromise;
}

api.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const original = error.config as RetryableConfig | undefined;

    if (error.response?.status !== 401 || !original || original._retry) {
      return Promise.reject(error);
    }
    original._retry = true;

    try {
      const token = await getFreshAccessToken();
      original.headers.Authorization = `Bearer ${token}`;
      return api(original);
    } catch (refreshError) {
      if (axios.isAxiosError(refreshError) && !refreshError.response) {
        // offline or timeout during refresh: the session may still be valid
        return Promise.reject(error);
      }
      await clearTokens();
      useAuthStore.getState().setSignedOut();
      return Promise.reject(refreshError);
    }
  },
);
```

Import this file once at app start (e.g. in `App.tsx`) so the interceptor is registered.

## Why each piece is there

**`authClient` without interceptors.** If the refresh call went through `api` and got a `401`, the interceptor would try to refresh again, which calls refresh again, and so on. A separate client breaks the loop.

**`_retry` flag.** If the re-sent request *also* gets a `401` (for example the user's account was disabled), we must not refresh again. The flag allows exactly one retry per request.

**One shared `refreshPromise` (single-flight).** A screen often fires several requests at once. If the token has expired, they **all** get `401` at about the same time:

```
without single-flight                with single-flight
GET /a -> 401 -> refresh(R1)         GET /a -> 401 -> refresh(R1) ─┐
GET /b -> 401 -> refresh(R1)  ✕      GET /b -> 401 -> wait ─────────┤
GET /c -> 401 -> refresh(R1)  ✕      GET /c -> 401 -> wait ─────────┤
                                                    new token ◄─────┘
                                     all three re-sent with it
```

With refresh token rotation, the second and third refresh would send an already-used token, and the server would revoke the session. Sharing one promise means one refresh, and every waiting request gets the same new token. `.finally` resets it so the next expiry starts a fresh refresh.

**Offline is not "logged out".** If the refresh fails because there is no network, the refresh token is probably still fine. Logging the user out on a train tunnel is a bad bug. Only clear tokens when the server actually rejects the refresh.

**`setSignedOut()`** flips the auth state in the store. The root navigator (Day 1: auth navigation) renders the login stack when that state changes, so the interceptor doesn't need access to navigation.

## Proactive refresh (optional)

Instead of waiting for a `401`, a request interceptor can check the JWT's `exp` and refresh shortly before it expires. This saves one failed request but adds clock-skew problems (the phone's clock can be wrong). Keep the `401` interceptor either way, as the safety net: the server can revoke a token at any time.

## Testing it

- Set the access token lifetime on a dev server to ~30 seconds.
- Open a screen that fires several requests at once after it expires: there should be **one** `/auth/refresh` call.
- Revoke the refresh token on the server: the app should land on the login screen.
- Turn on airplane mode while expired: the app should show an offline error, not log out.

## Rule of thumb

Refresh on `401` in a response interceptor, with a separate client for the refresh call, a `_retry` flag per request, and one shared refresh promise. Log out only when the server says the refresh token is bad.
