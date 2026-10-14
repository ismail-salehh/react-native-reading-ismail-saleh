# Axios

Axios is a library for making HTTP requests. React Native already has `fetch` built in, so Axios is not required. Teams pick it because it takes care of several things you would otherwise write yourself on top of `fetch`.

In React Native, Axios sends requests through `XMLHttpRequest`, which React Native implements natively. No extra native setup is needed.

```bash
npm install axios
```

## Axios vs fetch

| | `fetch` | Axios |
|---|---|---|
| Rejects on 4xx / 5xx | **No**, you must check `res.ok` | **Yes**, any non-2xx status throws |
| JSON body | `JSON.stringify` + set `Content-Type` yourself | Pass an object, it does both |
| JSON response | `await res.json()` | Already parsed in `res.data` |
| Timeout | None, build it with `AbortController` + `setTimeout` | `timeout` option |
| Cancellation | `AbortController` | `AbortController` (same `signal`) |
| Shared config (base URL, headers) | Write a wrapper | `axios.create()` |
| Interceptors | Write a wrapper | Built in |

The first row is the one that causes bugs. With `fetch`, a `500` response *resolves* the promise, so code that only uses `try/catch` treats a server error as success.

## A basic request

```ts
import axios from 'axios';

const res = await axios.get('https://api.example.com/products', {
  params: { page: 2, q: 'shoes' }, // -> /products?page=2&q=shoes
});

console.log(res.data);    // parsed JSON body
console.log(res.status);  // 200
console.log(res.headers); // response headers
```

Every method returns the same response object:

| Field | Contains |
|---|---|
| `data` | The response body, already parsed |
| `status` | HTTP status code |
| `headers` | Response headers (lower-case keys) |
| `config` | The config that was used to send the request |
| `request` | The underlying `XMLHttpRequest` |

Sending data:

```ts
await axios.post('/orders', { productId: 42, quantity: 1 });
await axios.patch('/users/me', { name: 'Sara' });
await axios.delete('/orders/7');
```

## One instance for the whole app

Don't call `axios.get` with a full URL all over the app. Create one instance with shared config and import it everywhere:

```ts
// src/api/client.ts
import axios from 'axios';

export const api = axios.create({
  baseURL: 'https://api.example.com/v1',
  timeout: 15_000,
  headers: { Accept: 'application/json' },
});
```

```ts
// src/api/products.ts
import { api } from './client';

export type Product = { id: number; name: string; price: number };

export async function getProducts(page: number) {
  const { data } = await api.get<Product[]>('/products', { params: { page } });
  return data; // typed as Product[]
}
```

Why this layout:

- Screens call `getProducts(1)`, not `api.get('/products')`. If the endpoint changes, only one file changes.
- The base URL, [timeout](11-timeout.md), and [interceptors](13-interceptors.md) live in one place.
- The generic `api.get<Product[]>` types `data`. It is a **promise, not a check**: Axios doesn't validate the shape. If you need that guarantee, validate with a library such as Zod.

The later files build on this `api` instance.

## Local development gotchas

- **Android emulator**: `localhost` is the emulator itself. Use `10.0.2.2` to reach your computer. On a real device, use your computer's LAN IP.
- **Plain `http://`**: Android (API 28+) blocks cleartext traffic and iOS App Transport Security blocks non-HTTPS by default. Use HTTPS, or allow it for development builds only.
- **Base URL per environment**: read it from config (for example `react-native-config` or Expo's `extra`), not a hard-coded string.

## Rule of thumb

Create one Axios instance, wrap each endpoint in a small typed function, and let screens call those functions. Everything else in this day (errors, timeouts, tokens, retries) plugs into that one instance.
