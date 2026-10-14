# Access and refresh tokens

Most mobile APIs use **token-based authentication**. The user logs in once with a password, and from then on the app proves who it is by sending a token. Using two tokens instead of one is a trade-off between security and convenience.

Where to store them on the device is covered in [Token storage](05-token-storage.md). This file covers what they are and how they are used.

## Why two tokens

One long-lived token would be convenient, but if it leaks (logs, a proxy, a compromised library), the attacker has access for weeks. One short-lived token would be safe, but the user would have to log in every 15 minutes.

Two tokens split those jobs:

| | Access token | Refresh token |
|---|---|---|
| Purpose | Prove identity on API calls | Get a new access token |
| Lifetime | Short: 5 to 60 minutes | Long: days to months |
| Sent | On **every** request, `Authorization: Bearer ...` | Only to `/auth/refresh` |
| Exposure | High (many requests, many logs) | Low (one endpoint, rarely) |
| If stolen | Useful until it expires, soon | Dangerous, can mint access tokens |
| Server checks it by | Usually its signature only (fast, no DB lookup) | Looking it up in the database |

The access token is exposed often but expires fast. The refresh token lives long but is rarely sent and can be revoked.

## The lifecycle

```
login (email + password)
 -> server returns { accessToken, refreshToken }

GET /products   Authorization: Bearer <access>   -> 200
GET /orders     Authorization: Bearer <access>   -> 200
... 15 minutes later ...
GET /orders     Authorization: Bearer <access>   -> 401 (expired)

POST /auth/refresh { refreshToken }
 -> 200 { accessToken: new, refreshToken: new }
 -> repeat GET /orders with the new token        -> 200

... refresh token expired or revoked ...
POST /auth/refresh -> 401 -> log the user out
```

Doing this refresh step automatically, without the user noticing, is [Automatic token refresh](15-automatic-token-refresh.md).

## JWTs

Access tokens are often **JWTs** (JSON Web Tokens). A JWT has three base64url parts separated by dots:

```
eyJhbGciOiJIUzI1NiJ9 . eyJzdWIiOiI0MiIsImV4cCI6MTcwMDAwMDkwMH0 . SflKxwRJSMeKKF2QT4fwpM...
      header                          payload                           signature
```

The payload decodes to something like:

```json
{ "sub": "42", "role": "customer", "exp": 1700000900 }
```

- `sub`: the user id. `exp`: expiry time, in **seconds** since 1970.
- The signature lets the server check the token wasn't changed. Only the server has the key.
- A JWT is **signed, not encrypted**. Anyone can decode the payload. Never put secrets in it.

The app may decode `exp` to refresh slightly *before* expiry, but it must never use the payload for security decisions (like "is this user an admin?") that the server doesn't also check.

Refresh tokens are usually **opaque**: a long random string that means nothing outside the server's database.

## Refresh token rotation

Good backends issue a **new refresh token on every refresh** and invalidate the old one. This allows **reuse detection**:

```
attacker steals refresh token R1
user's app refreshes with R1  -> gets R2, R1 is now used
attacker tries R1             -> server sees a used token
                              -> revokes the whole session (R2 too)
```

For the app this means: **always save the new refresh token** returned by `/auth/refresh`. Keeping the old one logs the user out on the next refresh.

It also means two parallel refreshes with the same token can break the session: the second one looks like reuse. The app must make sure only **one** refresh runs at a time (shown in the next file).

## Logout and revocation

An access token can't be "deleted" from the server: it is valid until `exp`. That is why it is short. On logout:

1. Call `POST /auth/logout` with the refresh token so the server revokes it.
2. Clear both tokens on the device (`clearTokens()`).
3. Clear cached user data (`queryClient.clear()`).

## Rule of thumb

The access token is the short-lived key you show at every door. The refresh token is the long-lived key you keep in the safe and only use to get a new access token. Save the new refresh token every time, and never refresh twice in parallel.
