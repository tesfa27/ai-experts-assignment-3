# Bug Fix Explanation

## What was the bug?

The token refresh logic failed when `oauth2_token` was a plain dict. No refresh occurred and no Authorization header was set.

## Why did it happen?

Original condition:
```python
if not self.oauth2_token or (isinstance(self.oauth2_token, OAuth2Token) and self.oauth2_token.expired)
```

When a dict was assigned:
- `not self.oauth2_token` → False (dict is truthy)
- `isinstance(..., OAuth2Token)` → False (short-circuits)

Both failed, so refresh never triggered.

## Why does your fix solve it?

Fixed condition:
```python
if not self.oauth2_token or not isinstance(self.oauth2_token, OAuth2Token) or self.oauth2_token.expired
```

Now refreshes if: token missing, OR not an OAuth2Token instance, OR expired.

## One edge case not covered

**Expired OAuth2Token instance** — No test verifies that a valid `OAuth2Token` with past `expires_at` triggers refresh.

```python
def test_api_request_refreshes_when_token_is_expired():
    c = Client()
    c.oauth2_token = OAuth2Token(access_token="stale", expires_at=int(time.time()) - 3600)
    resp = c.request("GET", "/me", api=True)
    assert resp["headers"].get("Authorization") == "Bearer fresh-token"
```

**Expected:** Pass. Adding this test prevents regressions.
