# Bug Fix Explanation

## What was the bug?

For API requests (`api=True`), when `oauth2_token` was a **dict** (e.g. raw JSON from an OAuth response that was never converted to `OAuth2Token`), the client did not refresh the token and did not set the `Authorization` header. The code only refreshed when the token was `None` or when it was an `OAuth2Token` that was expired. A dict is truthy and not an `OAuth2Token`, so neither branch ran; then `isinstance(self.oauth2_token, OAuth2Token)` was false, so the header was never set. The request was sent without auth.

## Why did it happen?

The type of `oauth2_token` is `Union[OAuth2Token, Dict[str, Any], None]`, but the refresh condition only handled `None` and `OAuth2Token` (with expiry). The case "token is present but not an `OAuth2Token`" (e.g. a dict) was not treated as "needs refresh," so the client assumed it could use the token. Only `OAuth2Token` instances get passed to `as_header()`, so the dict was never used and no header was added.

## Why does your fix actually solve it?

The fix adds an explicit check: refresh when the token is **not** an `OAuth2Token` instance. So for `api=True` we now refresh when: (1) token is missing, (2) token is not an `OAuth2Token` (e.g. dict), or (3) token is an expired `OAuth2Token`. After refresh, `oauth2_token` is always an `OAuth2Token`, so `as_header()` runs and the `Authorization` header is set correctly.

## What's one realistic case / edge case your tests still don't cover?

**Empty dict `{}` as token.** If `oauth2_token = {}`, it is truthy and not an `OAuth2Token`, so the fix refreshes and sets the header. So behavior is correct, but there is no test that explicitly sets `oauth2_token = {}` and asserts we get a fresh token and a valid header. Adding that test would make the "non-OAuth2Token but truthy" branch more explicit.
