# Retry failed requests

Some failures are temporary: a dropped packet, a server restarting, a moment of overload. Retrying the same request a bit later often succeeds. Other failures will fail the same way every time, and retrying them only wastes battery and server capacity.

## What to retry

| Failure | Retry? | Why |
|---|---|---|
| Network error (no response) | Yes | Connection may come back |
| [Timeout](11-timeout.md) | Yes, for safe methods | Could be a slow moment |
| 408 Request Timeout | Yes | Same |
| 429 Too Many Requests | Yes, after `Retry-After` | The server told you to wait |
| 502, 503, 504 | Yes | Gateway or server is temporarily down |
| 500 | Maybe once | Often a real bug that won't go away |
| 400, 403, 404, 409, 422 | **No** | The request itself is wrong |
| 401 | **No**, refresh instead | Handled by [token refresh](15-automatic-token-refresh.md) |
| Cancelled | **No** | The app asked for it |

## Which methods are safe to retry

A request is **idempotent** if sending it twice has the same effect as sending it once.

| Method | Idempotent | Retry automatically |
|---|---|---|
| `GET`, `HEAD`, `OPTIONS` | Yes | Yes |
| `PUT`, `DELETE` | Yes (by HTTP rules) | Yes, if your backend respects that |
| `POST`, `PATCH` | No | Only with an `Idempotency-Key` |

If a `POST /orders` timed out, the order may already exist. Retrying it can create a second order. If the backend supports it, send a unique `Idempotency-Key` header per user action and reuse it on retries; the server then returns the first result instead of creating a duplicate.

## Exponential backoff with jitter

Don't retry immediately and don't retry at a fixed interval. Wait longer after each attempt, and add randomness:

```
attempt 1 fails -> wait ~0-1 s
attempt 2 fails -> wait ~0-2 s
attempt 3 fails -> wait ~0-4 s
attempt 4 fails -> give up, show the error
```

- **Exponential**: a struggling server gets more breathing room each time.
- **Jitter** (randomness): if a server restart makes 100,000 phones fail at the same second, fixed delays make them all retry at the same second too, and knock it down again. Random delays spread them out.

```ts
function backoffDelay(attempt: number) {
  const max = Math.min(30_000, 1_000 * 2 ** attempt);
  return Math.random() * max; // "full jitter"
}
```

## As an interceptor

```ts
// src/api/retry.ts
import axios, { AxiosError, InternalAxiosRequestConfig } from 'axios';
import { api } from './client';

const MAX_RETRIES = 3;
const RETRY_STATUS = [408, 429, 502, 503, 504];
const SAFE_METHODS = ['get', 'head', 'options', 'put', 'delete'];

type RetryConfig = InternalAxiosRequestConfig & { _retryCount?: number };

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

api.interceptors.response.use(undefined, async (error: AxiosError) => {
  const config = error.config as RetryConfig | undefined;
  if (!config || axios.isCancel(error)) throw error;

  const status = error.response?.status;
  const retryable = status === undefined || RETRY_STATUS.includes(status);
  const safe = SAFE_METHODS.includes((config.method ?? 'get').toLowerCase());
  const attempt = config._retryCount ?? 0;

  if (!retryable || !safe || attempt >= MAX_RETRIES) throw error;

  config._retryCount = attempt + 1;

  const retryAfter = Number(error.response?.headers['retry-after']);
  await sleep(retryAfter > 0 ? retryAfter * 1000 : backoffDelay(attempt));

  if (config.signal?.aborted) throw error; // cancelled while waiting
  return api(config);
});
```

The pieces match the tables above: no response or a temporary status, a safe method, and a limit on attempts. `Retry-After` is honoured when it is given in seconds.

## Or use a library

`axios-retry` does the same with less code:

```ts
import axiosRetry from 'axios-retry';

axiosRetry(api, {
  retries: 3,
  retryDelay: axiosRetry.exponentialDelay,
  // default condition: network errors + 5xx on idempotent methods
});
```

## Don't stack retries

Retries can be configured in several layers. If each one retries 3 times, one tap can become many requests:

```
TanStack Query retry: 3   x   Axios retry: 3   =  up to 16 attempts
```

Pick **one** layer. If you use [TanStack Query](18-tanstack-query.md), its built-in retry (3 times with backoff, for queries) is often enough, and the Axios retry interceptor can be dropped or limited to network errors.

## When retries run out

Show an error state with a **Retry** button (see [Loading, error, and empty states](19-loading-error-empty-states.md)). If the device is offline, waiting for [NetInfo](17-netinfo.md) to report the connection is back and retrying then is better than retrying on a timer.

## Rule of thumb

Retry only temporary failures, only on idempotent requests, a few times, with exponential backoff and jitter. Everything else goes straight to the user.
