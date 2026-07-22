# Authentication, Rate Limits & Caching

## Dual-token system

| Token | Lifetime | Purpose |
| :--- | :--- | :--- |
| Bearer token | 1 hour | Sent as `Authorization: Bearer <token>` on every request |
| Refresh token | 30 days | Exchanged at `POST /v2/auth/refresh` for a new pair |

### 401 handling flow

1. Request fails with `401 Unauthorized`.
2. Intercept the response; do NOT retry with the same token.
3. `POST /v2/auth/refresh` with the stored `refresh_token`.
4. Persist BOTH the new bearer token and the new refresh token.
5. Retry the original request once with the new bearer token.

The old refresh token is invalidated the instant a new one is issued (rotation prevents reuse attacks). If the refresh call itself fails, the stored refresh token is stale or revoked — the user must re-authenticate; do not loop.

### Security guardrails

- Read tokens from environment variables (`MENTIONLYTICS_API_TOKEN`, `MENTIONLYTICS_REFRESH_TOKEN`).
- Never hardcode tokens in scripts, commit them to git, log them, or expose them in client-side/frontend code or generated output.

## Rate limiting (two independent layers per API key)

| Layer | Window | Max requests | 429 `error` value | Strategy |
| :--- | :--- | :--- | :--- | :--- |
| Burst protection | 1 second | 20 | `burst_limit_exceeded` | Short pause (~250–500ms), immediate retry |
| Sustained limit | 60 seconds | 100 | `rate_limit_exceeded` | Exponential backoff |

**429 error JSON shape:**

```json
{"status": 429, "error": "rate_limit_exceeded", "message": "..."}
```

**Response headers to monitor on every request:**

- `RateLimit-Limit` — max requests in the current window
- `RateLimit-Remaining` — requests left in the current window
- `RateLimit-Reset` — Unix timestamp when the window resets

### Retry pseudocode

```python
import time, random

def request_with_backoff(call, max_retries=5):
    for attempt in range(max_retries):
        resp = call()
        if resp.status_code != 429:
            return resp
        body = resp.json()
        if body.get("error") == "burst_limit_exceeded":
            time.sleep(0.5)                      # burst: brief pause
        else:
            time.sleep(min(2 ** attempt + random.random(), 60))  # sustained: backoff
    raise RuntimeError("Rate limit retries exhausted")
```

## Caching & request coalescing

- Successful responses (2xx) are cached for **15 seconds**.
- Cache key: Account ID + HTTP method + URL + query parameters + request body.
- Headers: `X-Cache` returns `HIT`, `MISS`, or `COALESCED`; `X-Cache-Age` gives seconds since caching.
- **Coalescing:** identical simultaneous requests arriving within ~100ms execute only once on the backend and share the result. When making parallel data calls, deliberately group identical-parameter requests to exploit this.
- Practical implication: polling the same endpoint more often than every 15s returns cached data — don't poll faster than the cache TTL.
