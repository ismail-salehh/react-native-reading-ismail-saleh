# Timeout

A timeout is the longest the app is willing to wait for a response. After that, the request is abandoned and fails with an error.

## Why it matters on mobile

Axios's default timeout is `0`, which means **no timeout**. On a phone that is a real problem:

- A train enters a tunnel and the connection stays "open" but nothing comes back.
- Captive Wi-Fi (hotels, airports) accepts the connection and never answers.
- The server is overloaded and holds the request.

Without a timeout, the spinner spins until the OS gives up, which can take minutes. The user can't tell if the app is broken.

## Setting it

Set a default on the instance, and override it for requests that need longer:

```ts
export const api = axios.create({
  baseURL: 'https://api.example.com/v1',
  timeout: 15_000, // 15 seconds, in milliseconds
});

// a large upload gets more time
await api.post('/uploads', formData, { timeout: 120_000 });

// a quick health check gets less
await api.get('/health', { timeout: 3_000 });
```

Typical values:

| Request | Timeout |
|---|---|
| Normal API call | 10 to 20 s |
| Search-as-you-type | 5 to 10 s |
| File upload / download | Minutes, or none with a progress bar and a cancel button |

## What the timeout measures

In React Native, Axios uses `XMLHttpRequest`, and the timeout covers the **whole request**: connecting, sending, the server working, and downloading the response. A slow download of a large body counts too.

## Detecting a timeout

```ts
try {
  await api.get('/products');
} catch (error) {
  if (axios.isAxiosError(error) && error.code === 'ECONNABORTED') {
    // timed out
  }
}
```

By default a timeout has the code `ECONNABORTED`. With `transitional: { clarifyTimeoutError: true }` in the config, Axios uses `ETIMEDOUT` instead, which is easier to tell apart. The `toApiError` helper in [Error handling](10-error-handling.md) checks for both.

A timeout error has **no `error.response`**, because no response arrived.

## A timeout doesn't mean "nothing happened"

The client stopped waiting. The server may still have received the request and finished it.

```
app:     POST /orders  ---->  (waits 15s)  ---->  timeout, shows "failed"
server:                 receives  ->  creates order  ->  replies (too late)
```

For a `GET` this doesn't matter. For a `POST /orders` or `POST /payments`, blindly [retrying](16-retry-failed-requests.md) can create a **duplicate**. Options:

- Send an `Idempotency-Key` header (a UUID per action). The server returns the original result if it sees the same key twice.
- After a timeout, re-fetch the state (e.g. the order list) before offering to try again.

## Timeout with fetch

`fetch` has no timeout option. You build one with an `AbortController`:

```ts
const controller = new AbortController();
const timer = setTimeout(() => controller.abort(), 15_000);
try {
  const res = await fetch(url, { signal: controller.signal });
} finally {
  clearTimeout(timer);
}
```

This is one of the reasons to use Axios. The same `AbortController` is also how you [cancel requests](12-request-cancellation.md) on purpose.

## Rule of thumb

Always set a timeout on the instance. Keep it short for reads, longer for uploads, and remember that a timed-out write may still have succeeded on the server.
