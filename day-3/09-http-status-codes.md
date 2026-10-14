# HTTP status codes

Every HTTP response starts with a three-digit status code. The first digit tells you the category, and that is often all the client needs to decide what to do.

| Range | Meaning | Whose problem |
|---|---|---|
| 1xx | Informational | Handled by the network layer, you won't see these |
| 2xx | Success | Nobody's |
| 3xx | Redirect | Followed automatically by the native HTTP stack |
| 4xx | The **request** was wrong | The client (app or user) |
| 5xx | The **server** failed | The server |

Axios resolves the promise for 2xx and **rejects** for everything else (see [Error handling](10-error-handling.md)).

## The codes you will actually handle

### 2xx

| Code | Name | When |
|---|---|---|
| 200 | OK | Normal success with a body |
| 201 | Created | A `POST` created something. Often returns the new item |
| 204 | No Content | Success with **no body**, common for `DELETE`. `res.data` is empty, don't parse it |

### 4xx

| Code | Name | Meaning | What the app does |
|---|---|---|---|
| 400 | Bad Request | Malformed request | Bug in the app, log it |
| 401 | Unauthorized | Not authenticated: token missing, invalid, or expired | [Refresh the token](15-automatic-token-refresh.md), then log out if that fails |
| 403 | Forbidden | Authenticated, but **not allowed** | Show "no access". Refreshing won't help |
| 404 | Not Found | Resource doesn't exist | Show a "not found" state |
| 408 | Request Timeout | Server gave up waiting for the request | Safe to [retry](16-retry-failed-requests.md) |
| 409 | Conflict | Clashes with current state (duplicate email, edited by someone else) | Show a specific message |
| 422 | Unprocessable Entity | Validation failed | Show field errors on the form |
| 429 | Too Many Requests | Rate limited | Wait (`Retry-After` header), then retry |

The names of 401 and 403 are confusing. Remember them as: **401 = "who are you?"**, **403 = "I know who you are, and no."**

### 5xx

| Code | Name | Meaning | What the app does |
|---|---|---|---|
| 500 | Internal Server Error | Unhandled server bug | Generic error, maybe retry once |
| 502 | Bad Gateway | A proxy got a bad answer from the server behind it | Retry with backoff |
| 503 | Service Unavailable | Overloaded or in maintenance | Retry with backoff, respect `Retry-After` |
| 504 | Gateway Timeout | A proxy waited too long for the server | Retry with backoff (careful with `POST`) |

## No status code at all

Some failures never reach the server, so there is no status:

- no internet connection
- DNS failure, TLS/certificate error
- the request [timed out](11-timeout.md) on the client
- the request was [cancelled](12-request-cancellation.md)

In Axios these errors have `error.response === undefined`. They need different handling from a 4xx or 5xx, and they are covered in [Error handling](10-error-handling.md).

## Reading the code in Axios

```ts
try {
  await api.delete(`/orders/${id}`);
} catch (error) {
  if (axios.isAxiosError(error) && error.response) {
    switch (error.response.status) {
      case 403: return showMessage("You can't cancel this order.");
      case 404: return showMessage('This order no longer exists.');
      case 409: return showMessage('This order has already shipped.');
    }
  }
  throw error;
}
```

Only handle the codes that mean something on **this screen**. Let the rest go up to the shared handler.

## Rule of thumb

4xx means "change the request", so retrying the same request is pointless (except 408 and 429). 5xx and "no response" mean "try again later", so retrying can help. That one split drives most of the decisions in the next files.
