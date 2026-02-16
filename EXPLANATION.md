# Bug Fix Explanation

## What was the bug?

When `api=True` and `oauth2_token` was a dict (e.g. raw JSON that was never converted to `OAuth2Token`), the client didn't refresh and didn't set the `Authorization` header. The code only refreshed when the token was `None` or when it was an expired `OAuth2Token`. A dict is truthy and not an `OAuth2Token`, so the refresh condition never ran. Then the header-setting branch only runs for `OAuth2Token` instances, so the dict was skipped and the request went out without auth.

## Why did it happen?

`oauth2_token` can be `None`, an `OAuth2Token`, or a dict. The refresh condition only checked for `None` and for "is it an OAuth2Token and expired?" So when the token was a dict, we never refreshed. And since only `OAuth2Token` instances get passed to `as_header()`, the dict was never used and no header was added.

## Why does your fix actually solve it?

The fix is to also refresh when the token is not an `OAuth2Token`. So we refresh when: token is missing, or it's not an OAuth2Token (e.g. a dict), or it's an expired OAuth2Token. After that, we always have an `OAuth2Token`, so `as_header()` runs and the header gets set.

## What's one realistic case / edge case your tests still don't cover?

An empty dict `{}` as the token. The fix handles it (we refresh and set the header), but we don't have a test that sets `oauth2_token = {}` and checks we get a fresh token. Adding that would make the "non-OAuth2Token but truthy" case explicit.
