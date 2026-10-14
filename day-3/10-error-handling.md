# Error handling

A request can fail in very different ways, and each one needs a different message and a different reaction. Good error handling starts with telling them apart.

## The four ways a request fails

```
api.get('/products')
 ├─ server replied with non-2xx      -> error.response exists     (404, 500, ...)
 ├─ request sent, no reply           -> error.request exists      (offline, timeout, DNS)
 ├─ request cancelled by the app     -> axios.isCancel(error)     (screen closed)
 └─ our own code threw               -> not an AxiosError at all  (bug)
```

In code:

```ts
import axios from 'axios';

try {
  const { data } = await api.get('/products');
} catch (error) {
  if (axios.isCancel(error)) {
    // we cancelled it on purpose, ignore
  } else if (axios.isAxiosError(error)) {
    if (error.response) {
      // server answered: error.response.status, error.response.data
    } else if (error.request) {
      // no answer: error.code is 'ERR_NETWORK', 'ECONNABORTED', ...
    }
  } else {
    // a bug in our code (TypeError, bad JSON handling, ...)
  }
}
```

`axios.isAxiosError` is also a TypeScript type guard, so after it `error.response` is typed. In a `catch` block, `error` is `unknown` and must be narrowed first.

## Normalize once

Writing that `if` chain on every screen is noisy and inconsistent. Convert every error into one app-level type, in one place:

```ts
// src/api/errors.ts
import axios from 'axios';

export type ApiError =
  | { kind: 'cancelled' }
  | { kind: 'offline' }
  | { kind: 'timeout' }
  | { kind: 'http'; status: number; message?: string; fieldErrors?: Record<string, string> }
  | { kind: 'unknown'; cause: unknown };

export function toApiError(error: unknown): ApiError {
  if (axios.isCancel(error)) return { kind: 'cancelled' };

  if (axios.isAxiosError(error)) {
    if (error.response) {
      const body = error.response.data as { message?: string; errors?: Record<string, string> };
      return {
        kind: 'http',
        status: error.response.status,
        message: body?.message,
        fieldErrors: body?.errors,
      };
    }
    if (error.code === 'ECONNABORTED' || error.code === 'ETIMEDOUT') return { kind: 'timeout' };
    if (error.code === 'ERR_NETWORK') return { kind: 'offline' };
  }

  return { kind: 'unknown', cause: error };
}
```

The shape of `body` depends on your backend. Agree on one error format with the backend team (for example `{ message, errors }`) so this function stays small.

Then turn it into text for the user:

```ts
export function errorMessage(e: ApiError): string {
  switch (e.kind) {
    case 'offline': return 'You are offline. Check your connection and try again.';
    case 'timeout': return 'The server is taking too long. Please try again.';
    case 'http':
      if (e.status === 404) return 'We could not find that.';
      if (e.status === 403) return "You don't have access to this.";
      if (e.status >= 500) return 'Something went wrong on our side. Please try again.';
      return e.message ?? 'Something went wrong.';
    default: return 'Something went wrong.';
  }
}
```

## Where to handle what

| Layer | Handles |
|---|---|
| [Interceptor](13-interceptors.md) | Things every request shares: 401 + [token refresh](15-automatic-token-refresh.md), [retries](16-retry-failed-requests.md), logging |
| API function (`getProducts`) | Usually nothing. Let errors pass through |
| Screen / hook | What the user sees: an error state, a toast, field errors on a form |

A screen should only special-case what is meaningful for it, like `409` on "create account" meaning "email already used". Everything else falls back to `errorMessage`.

## Form errors (422)

```ts
try {
  await api.post('/signup', values);
} catch (err) {
  const e = toApiError(err);
  if (e.kind === 'http' && e.status === 422 && e.fieldErrors) {
    Object.entries(e.fieldErrors).forEach(([field, msg]) => setError(field, { message: msg }));
    return;
  }
  showToast(errorMessage(e));
}
```

`setError` here is from `react-hook-form`. The idea is the same with any form library: map server field errors onto the fields.

## Things to avoid

- **Swallowing errors.** `catch (e) {}` turns a failure into a screen that never finishes loading.
- **Showing raw server messages or stack traces.** They can leak internals and are rarely written for users.
- **Showing an error for a cancelled request.** The user left the screen on purpose.
- **Expecting an Error Boundary to catch request errors.** Error boundaries only catch errors thrown **while rendering**. A rejected promise in `useEffect` or an `onPress` handler is not caught by them.
- **Forgetting unhandled rejections.** A promise without `.catch` or `await` inside `try` fails silently in release builds. Report unexpected errors (`kind: 'unknown'`) to a crash reporter such as Sentry.

## Rule of thumb

Classify every error once with `toApiError`, handle shared cases in interceptors, and let screens only decide what the user sees.
