# Explanation

## What was the bug?

For API requests, the client only refreshed the OAuth2 token when it was missing or when it was an `OAuth2Token` instance that was expired. When `oauth2Token` was a **plain object** (e.g. `{ accessToken: "stale", expiresAt: 0 }` from JSON or a bad deserialization), the code did not refresh. It also did not set the `Authorization` header, because only `OAuth2Token` instances have `asHeader()`. So the request went out with no auth and a stale/invalid token left in state.

## Why did it happen?

`TokenState` allows `OAuth2Token | Record<string, unknown> | null`. The refresh condition was:

- `!this.oauth2Token` (missing) → refresh
- `this.oauth2Token instanceof OAuth2Token && this.oauth2Token.expired` → refresh

A plain object is truthy and is not an instance of `OAuth2Token`, so both parts were false and the token was never refreshed. The code never handled the “token present but not a valid `OAuth2Token`” case.

## Why does your fix solve it?

The fix adds a check so we also refresh when the current token is **not** an `OAuth2Token` instance: `!(this.oauth2Token instanceof OAuth2Token)`. So we refresh when: token is null, token is a plain object (or any non-instance), or token is an expired `OAuth2Token`. After that, we only set `Authorization` when the token is an `OAuth2Token`, so we always use a valid instance for the header.

## One realistic case / edge case your tests still don't cover

**Invalid `expiresAt` (e.g. `NaN`).** If an `OAuth2Token` is constructed with `expiresAt` that is `NaN` (e.g. from bad JSON or an API response), then `now >= NaN` is false, so the `expired` getter returns false and the token is never treated as expired. The client would keep using it and never refresh. A test that sets `expiresAt: NaN` and expects a refresh would catch this; the current tests do not.
