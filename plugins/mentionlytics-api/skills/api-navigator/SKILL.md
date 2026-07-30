---
name: api-navigator
description: Query the Mentionlytics API (v2) to fetch, filter, and analyze social listening data - mentions, sentiment, top influencers, trending keywords, geographic and language breakdowns, AI daily summaries, analytics stats (EMV, MQI), audience demographics, trackers, and tracker groups. This skill should be used whenever the user wants to work with the Mentionlytics API, pull brand mentions or monitoring data programmatically, build an integration or script against api.mentionlytics.com, or asks about Mentionlytics endpoints, authentication, tokens, or rate limits.
---

# Mentionlytics API Navigator

Operational guide for interacting with the Mentionlytics API (v2) to fetch, filter, and analyze social media monitoring data.

## Base Configuration

- **Base URL:** `https://api.mentionlytics.com`
- **Headers:**
  ```http
  Authorization: Bearer <token>
  Content-Type: application/json
  ```
- **Token source:** Read the bearer token from the `MENTIONLYTICS_API_TOKEN` environment variable (and `MENTIONLYTICS_REFRESH_TOKEN` for refresh). Never hardcode tokens in scripts, never print them in output, and never expose them in client-side or frontend code.

## Authentication (Dual-Token System)

The API uses a short-lived **Bearer Token (1 hour)** and a long-lived **Refresh Token (30 days)**.

**On any `401 Unauthorized`:** immediately `POST /v2/auth/refresh` with the stored refresh token, then update BOTH tokens before retrying:

```bash
curl -X POST "https://api.mentionlytics.com/v2/auth/refresh" \
  -H "Content-Type: application/json" \
  -d '{"refresh_token": "'"$MENTIONLYTICS_REFRESH_TOKEN"'"}'
```

The old refresh token is invalidated the moment a new one is issued — always persist the new pair.

## Endpoint Routing Matrix

Map user intent to the correct endpoint. Full parameter details for each endpoint are in `references/endpoints.md` — read it before constructing any non-trivial request.

| User Intent | Method | Endpoint |
| :--- | :--- | :--- |
| Refresh expired access token | `POST` | `/v2/auth/refresh` |
| Fetch individual mentions | `GET` | `/v2/mentions` |
| Top profiles / influencers | `GET` | `/v2/mentions/top/mentioners` |
| Trending keywords / hashtags | `GET` | `/v2/mentions/top/keywords` |
| Geographic breakdown | `GET` | `/v2/mentions/top/countries` |
| Language distribution | `GET` | `/v2/mentions/top/languages` |
| AI-generated daily summaries | `GET` | `/v2/mentions/top/summary/daily` |
| High-level analytics (EMV, MQI) | `GET` | `/v2/mentions/stats` |
| FB/IG audience demographics | `GET` | `/v2/mentions/audience-demographics` |
| Tracker groups | `GET` | `/v2/tracker-groups/list` |
| Active trackers (keyword/profile/location) | `GET` | `/v2/trackers/list` |

## Core Parameter Rules

Full per-endpoint tables (params, defaults, response shapes) are in `references/endpoints.md` — read it before any non-trivial request.

- **Dates** (`startDate` / `endDate`): `YYYYMMDD`. Defaults: 7 days ago / today. `audience-demographics` takes only `endDate`.
- **Relevance**: `all` | `low` (≥0.25) | `normal` (≥0.5, default) | `strict` (≥0.75).
- **Trackers**: `brandmonitoring` (default) | `competitors` | `leads` | `all` | comma-separated IDs. Exception: `top/summary/daily` has no `leads`.
- **Other filters** on `/v2/mentions*`: `sentiment`, `emotion`, `contentType`, `country`, `language`, `trackerGroups`, `preset`, `fields`.
- **Ordering** — ⚠️ **values & default differ by endpoint; do NOT assume one set:**
  - `/v2/mentions`: `time` (default) | `rank` | `engagement` | `engagement_nv`
  - top-lists (`mentioners`/`keywords`/`countries`/`languages`): `mentions` | `reach` | `engagement` | `engagement_nv` (default `reach` for mentioners, `mentions` for countries/languages) — plus `sortDesc`.
- **Channels**: comma-separated IDs — see `references/channels-categories.md` when filtering by platform.
- **Pagination differs:** `/v2/mentions` → cursor via `results_after` + `per_page` (no `page`); `top/mentioners` → `page` + `per_page`; other top-lists → `limit`.

## Rate Limits & Caching (must respect)

Two independent throttle layers per API key. Monitor `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset` response headers:

| Layer | Window | Max | On 429 (`error` field) | Strategy |
| :--- | :--- | :--- | :--- | :--- |
| Burst | 1 second | 10 req (default) | `burst_limit_exceeded` | Short pause, retry |
| Sustained | 60 seconds | per `RateLimit-Limit` | `rate_limit_exceeded` | Exponential backoff |

Limits are enforced **per bearer token (account)**. The exact sustained ceiling is plan/config-dependent — read it from the `RateLimit-Limit` header rather than hardcoding a number.

Successful responses are cached for **15 seconds** (per account + method + URL + params + body). Inspect `X-Cache` (`HIT`/`MISS`/`COALESCED`) and `X-Cache-Age`. When firing parallel calls, group identical-parameter requests within ~100ms to benefit from request coalescing.

Full details, retry pseudocode, and error JSON structures: `references/auth-rate-limits.md`.

## Known Data Caveats

Content from X (Twitter), LinkedIn, and Web News may be subject to strict research caps depending on the user's plan — API results might not exactly mirror the Mentionlytics web app for these sources. Mention this proactively if the user compares API output to the dashboard.

## Reference Files

- `references/endpoints.md` — per-endpoint parameters, limits, example calls, and expected `200`/error response shapes. Read before building any request beyond the basics or validating a response.
- `references/channels-categories.md` — channel ID enums and category filter tokens per platform. Read whenever filtering by source/platform.
- `references/auth-rate-limits.md` — token rotation flow, 429 handling, caching and coalescing details. Read when writing retry logic or long-running scripts.
