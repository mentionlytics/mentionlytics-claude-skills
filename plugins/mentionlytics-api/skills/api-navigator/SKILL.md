---
name: api-navigator
description: Query the Mentionlytics API (v2) to fetch, filter, and analyze social listening data - mentions, sentiment, top influencers, trending keywords, geographic and language breakdowns, AI daily summaries, analytics stats (EMV, MQI), audience demographics, trackers, and account usage. This skill should be used whenever the user wants to work with the Mentionlytics API, pull brand mentions or monitoring data programmatically, build an integration or script against api.mentionlytics.com, or asks about Mentionlytics endpoints, authentication, tokens, or rate limits.
---

# Mentionlytics API Navigator

Operational guide for interacting with the Mentionlytics API (v2.0.8) to fetch, filter, and analyze social media monitoring data.

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
| Active trackers (keyword/profile) | `GET` | `/v2/trackers/list` |
| Subscription quota and usage | `GET` | `/v2/account/usage` |

## Core Parameter Rules

- **Dates** (`startDate` / `endDate`): format `YYYYMMDD` (e.g. `20240520`). Defaults: startDate = 7 days ago, endDate = today.
- **Relevance** (`relevance`): `all` | `low` (≥0.25) | `normal` (≥0.5) | `strict` (≥0.75).
- **Trackers** (`trackers`): `brandmonitoring` | `competitors` | `leads` | `all` | comma-separated tracker IDs.
- **Ordering** (`ordering`): `time` | `rank` | `engagement` | `engagement_nv` (without views).
- **Channels** (`channels`): comma-separated integers. Look up the channel ID and category token mappings in `references/channels-categories.md` whenever the user filters by platform (e.g. "only Instagram", "Reddit posts", "reviews").
- `/v2/mentions` uses **timestamp-based cursor pagination** via `results_after`; top-lists use `page` / `per_page` (max 100).

## Rate Limits & Caching (must respect)

Two independent throttle layers per API key. Monitor `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset` response headers:

| Layer | Window | Max | On 429 (`error` field) | Strategy |
| :--- | :--- | :--- | :--- | :--- |
| Burst | 1 second | 20 req | `burst_limit_exceeded` | Short pause, retry |
| Sustained | 60 seconds | 100 req | `rate_limit_exceeded` | Exponential backoff |

Successful responses are cached for **15 seconds** (per account + method + URL + params + body). Inspect `X-Cache` (`HIT`/`MISS`/`COALESCED`) and `X-Cache-Age`. When firing parallel calls, group identical-parameter requests within ~100ms to benefit from request coalescing.

Full details, retry pseudocode, and error JSON structures: `references/auth-rate-limits.md`.

## Known Data Caveats

Content from X (Twitter), LinkedIn, and Web News may be subject to strict research caps depending on the user's plan — API results might not exactly mirror the Mentionlytics web app for these sources. Mention this proactively if the user compares API output to the dashboard.

## Reference Files

- `references/endpoints.md` — per-endpoint parameters, limits, and example calls. Read before building any request beyond the basics.
- `references/channels-categories.md` — channel ID enums and category filter tokens per platform. Read whenever filtering by source/platform.
- `references/auth-rate-limits.md` — token rotation flow, 429 handling, caching and coalescing details. Read when writing retry logic or long-running scripts.
