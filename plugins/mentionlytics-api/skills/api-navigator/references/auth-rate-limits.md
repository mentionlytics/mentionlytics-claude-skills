# Authentication, Rate Limits & Caching

## Contents
1. [Tokens](#tokens)
2. [Refreshing tokens](#refresh)
3. [Rate limits](#rate-limits)
4. [Retry helper](#retry)
5. [Caching and coalescing](#caching)

---

<a name="tokens"></a>
## Tokens

| Token | Lifetime | Purpose |
| :--- | :--- | :--- |
| Bearer token | 1 hour | Sent as `Authorization: Bearer <token>` on every request |
| Refresh token | 30 days | Exchanged at `POST /v2/auth/refresh` for a new pair |

- Both are in the web app under **Settings → Api/Access Tokens → User API Tokens**.
- A token acts as the user it was issued for: reads see what that user sees, and writes need that user to be an admin on the account (see `write-endpoints.md`).
- The account's plan must include API access; otherwise every request answers `403 apiV2_not_available`.

**Security guardrails**
- Read tokens from environment variables (`MENTIONLYTICS_API_TOKEN`, `MENTIONLYTICS_REFRESH_TOKEN`) or a secret store.
- Never hardcode tokens in scripts, commit them to git, log them, print them in output, or put them in frontend code.
- In examples and commands shown to the user, write `$MENTIONLYTICS_API_TOKEN` rather than the token's value.
- Use HTTPS for every request.

<a name="refresh"></a>
## Refreshing tokens

`POST /v2/auth/refresh` takes the refresh token in a JSON body and needs no `Authorization` header.

```bash
curl -X POST "https://api.mentionlytics.com/v2/auth/refresh" \
  -H "Content-Type: application/json" \
  -d "{\"refresh_token\": \"$MENTIONLYTICS_REFRESH_TOKEN\"}"
```

**Expected response (200):**

```json
{
  "bearer": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "12346-xyz789abc012def345ghi678jkl901"
}
```

- **Rotation:** the old refresh token stops working as soon as the new pair is issued. Persist **both** new tokens before using them. A lost new refresh token means new tokens have to be copied from the web app.
- **One refresher:** two processes refreshing with the same token race, and the loser gets `401 token_revoked`. Let one process, or a lock, do the refreshing, and share the new bearer token with the others.
- **Timeouts:** give the refresh call a generous timeout. A refresh the client gives up on may still have rotated the tokens, which leaves the stored refresh token used up.
- **Proactive refresh:** refreshing shortly before the bearer token's hour is up avoids a failed request; refreshing on a `401` works too.

| HTTP | `error` | Meaning |
| :--- | :--- | :--- |
| `400` | `bad_request` (message `missing_refresh_token`) | No refresh token in the body |
| `401` | `invalid_refresh_token` | The refresh token is invalid |
| `401` | `token_expired` | The refresh token is older than 30 days |
| `401` | `token_revoked` | The refresh token was already used |
| `429` | `rate_limit_exceeded` | Too many refreshes; wait before trying again |
| `520` | `internal_error` | Server error; retry later |

A `401` from the refresh endpoint means the stored refresh token can't be used again: new tokens have to come from the web app. Don't retry it in a loop.

### Handling a 401 on a request

1. A request fails with `401` (`token_expired`, `invalid_token`, `unauthorized` or `invalid_token_options`).
2. Don't retry with the same bearer token.
3. `POST /v2/auth/refresh` with the stored refresh token.
4. Persist both new tokens.
5. Retry the original request once with the new bearer token.
6. If the refresh itself fails with `401`, stop and ask the user for new tokens.

---

<a name="rate-limits"></a>
## Rate limits

Limits apply **per account**: every token issued for the account draws on the same counters.

### Reads (`GET`)

| Layer | Window | Requests |
| :--- | :--- | :--- |
| Burst | 1 second | 5 |
| Sustained | 60 seconds | 100 |

Both answer `429` with `"error": "rate_limit_exceeded"`, and the message tells them apart:

```json
{ "status": 429, "error": "rate_limit_exceeded", "message": "Burst limit exceeded. Please slow down your requests." }
```
```json
{ "status": 429, "error": "rate_limit_exceeded", "message": "Rate limit exceeded. Please slow down your requests." }
```

### Writes (`POST`, `PUT`, `DELETE` on trackers and tracker groups)

Writes have limits of their own, separate from the read limits:

| Layer | Window | Writes, plan of up to 25 trackers |
| :--- | :--- | :--- |
| Burst | 5 seconds | 2 |
| Sustained | 60 seconds | 10 |
| Daily | 1 day | 100 |

- A larger plan gets higher write limits, in proportion to the trackers it allows (keyword trackers, social profiles, Locations and AI Visibility prompts, add-ons included), up to 20 times these: a plan that allows 100 trackers gets 8, 40 and 400. The limits follow the plan, not the trackers in use, so creating or deleting trackers doesn't change them. The `X-RateLimit-*` headers of a write report the account's own limits.
- `429 rate_limit_exceeded`: the burst or sustained write limit. It clears after a short wait.
- `429 daily_write_limit_exceeded`: the day's writes are used up. `Retry-After` gives the seconds until the reset, and the message gives the time, such as `Daily write limit exceeded. It resets at 2026-09-28T09:00:00.000Z.` Writes that fail with an error don't count toward the daily limit.
- `429 keyword_edit_limit_reached` (tracker updates only): `existing_mentions: "delete"` was used on this tracker too many times today. It isn't a pace limit: try again tomorrow, or resend without it to keep the mentions.
- Pace bulk jobs, such as creating many trackers, rather than sending the requests in a loop.

### Headers

| Header | Meaning |
| :--- | :--- |
| `X-RateLimit-Limit` | Requests allowed in the window |
| `X-RateLimit-Remaining` | Requests left in the window |
| `X-RateLimit-Reset` | When the window resets, as a Unix timestamp in seconds |
| `Retry-After` | On a `429`: seconds to wait before retrying |

- On reads, the `X-RateLimit-*` headers describe the sustained limit, except on a `429` from the burst limit, which reports the burst limit. `X-RateLimit-Remaining` therefore doesn't warn of a burst `429`: keep reads to about 5 a second.
- A read `429`'s `Retry-After` is the length of the window. Waiting that long always clears the limit, and it may clear sooner: see `X-RateLimit-Reset`.

### What to do on a 429

| `error` | Applies to | What to do |
| :--- | :--- | :--- |
| `rate_limit_exceeded` | Reads and writes | Wait until `X-RateLimit-Reset` or `Retry-After`, whichever comes first, then retry; back off further if it repeats |
| `daily_write_limit_exceeded` | Writes | Stop writing until the reset; report the reset time to the user |
| `keyword_edit_limit_reached` | Tracker updates | Don't retry today |

---

<a name="retry"></a>
## Retry helper

A Python client that refreshes once on a `401`, waits out pace limits, and stops on the limits that a retry can't clear:

```python
import os
import random
import time

import requests

BASE = "https://api.mentionlytics.com"
NO_RETRY_429 = {"daily_write_limit_exceeded", "keyword_edit_limit_reached"}


def error_code(resp):
    try:
        return resp.json().get("error")
    except ValueError:
        return None


def retry_delay(resp, attempt):
    delays = []
    if "Retry-After" in resp.headers:
        delays.append(float(resp.headers["Retry-After"]))
    if "X-RateLimit-Reset" in resp.headers:
        delays.append(float(resp.headers["X-RateLimit-Reset"]) - time.time())
    delays = [d for d in delays if d > 0]
    return min(delays) if delays else min(2 ** attempt, 60)


class MentionlyticsClient:
    def __init__(self, bearer, refresh_token, save_tokens):
        self.bearer = bearer
        self.refresh_token = refresh_token
        self.save_tokens = save_tokens  # persists both tokens, e.g. to a secret store

    def refresh(self):
        resp = requests.post(f"{BASE}/v2/auth/refresh", json={"refresh_token": self.refresh_token}, timeout=60)
        resp.raise_for_status()  # a 401 here means new tokens are needed from the web app
        tokens = resp.json()
        self.bearer, self.refresh_token = tokens["bearer"], tokens["refresh_token"]
        self.save_tokens(self.bearer, self.refresh_token)

    def request(self, method, path, max_retries=5, **kwargs):
        refreshed = False
        for attempt in range(max_retries + 1):
            headers = {"Authorization": f"Bearer {self.bearer}"}
            resp = requests.request(method, BASE + path, headers=headers, timeout=120, **kwargs)
            if resp.status_code == 401 and not refreshed:
                self.refresh()
                refreshed = True
                continue
            if resp.status_code != 429 or error_code(resp) in NO_RETRY_429 or attempt == max_retries:
                return resp
            time.sleep(retry_delay(resp, attempt) + random.random())
        return resp


client = MentionlyticsClient(
    os.environ["MENTIONLYTICS_API_TOKEN"],
    os.environ["MENTIONLYTICS_REFRESH_TOKEN"],
    save_tokens=lambda bearer, refresh_token: None,  # replace with real persistence
)
resp = client.request("GET", "/v2/mentions", params={"startDate": "20261001", "per_page": 100})
```

It retries only `401` (once, after a refresh) and `429`. A request that times out isn't retried, so a create is never sent twice.

---

<a name="caching"></a>
## Caching and coalescing

- **What is cached:** read (`GET`) responses with status `200`, for **30 seconds**. Writes are never cached or coalesced.
- **Cache key:** the account and user, the method, and the URL with its query parameters. Any change to the query is a new request.
- **Headers:** `Cache-Control: max-age=N` means the response came from the cache, with `N` seconds left; `Cache-Control: no-cache` means it was freshly generated.
- **Coalescing:** identical reads that arrive while the first one is still running wait for it and share its response, marked `X-Cache: COALESCED`. If the first one fails, each of them runs on its own.
- Cached and coalesced responses still count toward the rate limits.

In practice:
- Polling the same URL more often than every 30 seconds returns the same data.
- A write doesn't clear the cache. A read identical to one made in the last 30 seconds can still show the state before the write, so use the write's response, or read again after 30 seconds.
- Parallel calls with identical parameters run once on the server, so there is no need to deduplicate them first.
