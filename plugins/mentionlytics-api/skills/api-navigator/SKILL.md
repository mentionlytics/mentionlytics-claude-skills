---
name: api-navigator
description: Query and manage a Mentionlytics account through the Mentionlytics API (v2) - mentions with sentiment, emotion, entities, translations and video analysis; top mentioners, keywords, countries and languages; AI daily summaries; analytics stats (EMV, MQI); audience demographics; AI Visibility answers, brand overview and citation analysis; tracker and tracker group lists; account usage; and creating, updating, pausing and deleting trackers and tracker groups. This skill should be used whenever the user wants to work with the Mentionlytics API, pull brand monitoring or AI visibility data programmatically, manage trackers through the API, build a script or integration against api.mentionlytics.com, or asks about Mentionlytics endpoints, tokens, rate limits or errors.
---

# Mentionlytics API Navigator

Operational guide for the Mentionlytics API v2 (documentation version 2.1.17): reading mentions, analytics and AI Visibility answers, and managing trackers and tracker groups.

## Base Configuration

- **Base URL:** `https://api.mentionlytics.com`
- **Headers:**
  ```http
  Authorization: Bearer <token>
  Content-Type: application/json
  ```
  `Authorization` goes on every request except `POST /v2/auth/refresh`. `Content-Type` is needed on requests with a body.
- **Token source:** read the bearer token from `MENTIONLYTICS_API_TOKEN` and the refresh token from `MENTIONLYTICS_REFRESH_TOKEN`. Users copy both from the web app: **Settings → Api/Access Tokens → User API Tokens**. Never hardcode tokens in scripts, print them, commit them, or put them in client-side or frontend code.
- **Access:** the account's plan must include API access; otherwise every call answers `403 apiV2_not_available`. The answers endpoints also need AI Visibility, and the write endpoints need write access (see [Writing](#writing-beta)).

## Authentication (Dual-Token System)

The API uses a short-lived **bearer token (1 hour)** and a long-lived **refresh token (30 days)**.

**On any `401`:** `POST /v2/auth/refresh` with the stored refresh token, store BOTH returned tokens, then retry the original request once:

```bash
curl -X POST "https://api.mentionlytics.com/v2/auth/refresh" \
  -H "Content-Type: application/json" \
  -d '{"refresh_token": "'"$MENTIONLYTICS_REFRESH_TOKEN"'"}'
```

The response is `{"bearer": "...", "refresh_token": "..."}`. The old refresh token stops working the moment it is used, so refresh from one place only: two processes refreshing with the same token lock one of them out (`401 token_revoked`). If the refresh itself fails with `401` (`invalid_refresh_token`, `token_expired`, `token_revoked`), stop and ask the user for new tokens from the web app. Never loop.

## Endpoint Routing Matrix

Map user intent to the endpoint, then read the reference file before building any non-trivial request.

| User intent | Method | Endpoint | Reference |
| :--- | :--- | :--- | :--- |
| Renew an expired bearer token | `POST` | `/v2/auth/refresh` | `auth-rate-limits.md` |
| Individual mentions (posts, articles, comments, reviews) | `GET` | `/v2/mentions` | `endpoints.md` |
| Top profiles / influencers | `GET` | `/v2/mentions/top/mentioners` | `endpoints.md` |
| Trending keywords / hashtags | `GET` | `/v2/mentions/top/keywords` | `endpoints.md` |
| Mentions by country | `GET` | `/v2/mentions/top/countries` | `endpoints.md` |
| Mentions by language | `GET` | `/v2/mentions/top/languages` | `endpoints.md` |
| AI daily summaries of mention clusters | `GET` | `/v2/mentions/top/summary/daily` | `endpoints.md` |
| Totals: mentions, reach, engagement, sentiment, EMV, MQI | `GET` | `/v2/mentions/stats` | `endpoints.md` |
| Age/gender of owned Facebook/Instagram audiences | `GET` | `/v2/mentions/audience-demographics` | `endpoints.md` |
| AI Visibility answers (ChatGPT, Gemini, Perplexity, …) | `GET` | `/v2/answers` | `answers.md` |
| Brand visibility and share of voice in AI answers | `GET` | `/v2/answers/brand-overview` | `answers.md` |
| Analysis of the pages an AI answer cites | `GET` | `/v2/answers/citation-analysis/{id}` | `answers.md` |
| List tracker groups | `GET` | `/v2/tracker-groups/list` | `endpoints.md` |
| List trackers (keyword, profile, location, prompt) | `GET` | `/v2/trackers/list` | `endpoints.md` |
| Plan limits and current usage | `GET` | `/v2/account/usage` | `endpoints.md` |
| Create a tracker group | `POST` | `/v2/tracker-groups/create` | `write-endpoints.md` |
| Rename, recolor, pause or resume a tracker group | `PUT` | `/v2/tracker-groups/update/{id}` | `write-endpoints.md` |
| Delete a tracker group and all its trackers | `DELETE` | `/v2/tracker-groups/delete/{id}` | `write-endpoints.md` |
| Create a keyword tracker | `POST` | `/v2/trackers/create` | `write-endpoints.md` |
| Change, move, pause or resume a tracker | `PUT` | `/v2/trackers/update/{id}` | `write-endpoints.md` |
| Delete a tracker | `DELETE` | `/v2/trackers/delete/{id}` | `write-endpoints.md` |

The three `/v2/answers*` endpoints and the six write endpoints are **beta**.

## Core Parameter Rules

- **Parameters are NOT uniform.** `ordering` values, paging, defaults and accepted filters differ per endpoint. An unknown parameter or value answers `400`.
- **Dates** (`startDate` / `endDate`): `YYYYMMDD`, from `20200101`; `startDate` at most tomorrow, `endDate` at most today. Default range: the last 7 days up to today (UTC). `/v2/mentions/audience-demographics` takes only `endDate`. `mention_ids` and `answer_ids` ignore the dates.
- **Trackers** (`trackers`): `brandmonitoring` | `competitors` | `leads` (Industry & Topics groups) | `all` | comma-separated tracker IDs. Defaults: `brandmonitoring` on mention endpoints, `all` Prompt trackers on answers. `/v2/mentions/top/summary/daily` takes only `brandmonitoring` | `competitors` | `all`. `trackerGroups` takes comma-separated group IDs. Get IDs from the two list endpoints.
- **Relevance** (`relevance`) means different things per family:
  - mentions: `normal` (default, score ≥ 0.5) | `strict` (≥ 0.75) | `low` (only below 0.5, the mentions `normal` leaves out) | `all`;
  - answers: `all` (default) | `normal` or `strict` (the answer names at least one of your brands) | `low` (it names none).
- **Filters:** `sentiment` (`positive` | `neutral` | `negative`) and `emotion` (`joy` | `anger` | `sadness` | `fear` | `sarcasm` | `neutral`) take one value. `channels`, `categories`, `country`, `language` (ISO codes), `engines` and ID lists are comma-separated. `contentType`: `image` | `album` | `video` | `audio` | `link` | `webpage` | `text` | `other`. `preset` is the ID of a filter saved in the web app.
- **Ordering** (`ordering`): ⚠️ two value sets, never mix them.
  - `/v2/mentions`: `time` (default) | `rank` | `engagement` | `engagement_nv`.
  - Top lists (`top/mentioners`, `top/keywords`, `top/countries`, `top/languages`): `mentions` | `reach` | `engagement` | `engagement_nv`, plus `sortDesc` (default `true`). Default `reach` for mentioners, `mentions` for the others.

  `engagement` counts likes, shares, comments and views, except X/Twitter views; `engagement_nv` leaves views out.
- **Channels and categories:** look up IDs and tokens in `references/channels-categories.md` whenever the user filters by platform. AI Visibility answers never come back from the mention endpoints; use `/v2/answers`.
- **Fields** (`fields`, comma-separated) trims any response. Dotted names select nested fields (`profile.name`, `metrics.engagement`); lists such as `translations`, `brands` and `citations` are selected whole. On `/v2/mentions` without `fields`, send `extendedFields=false` to leave out the full text of web pages (`body`) on large exports, or `extendedFields=true` if the script needs `body`.
- **Missing values:** a field with no value is left out of the response; treat a missing count as 0.

## Paging

| Endpoint | Style | Parameters |
| :--- | :--- | :--- |
| `/v2/mentions` | cursor | `per_page` 1–10000 (default 50), `results_after` |
| `/v2/answers` | cursor | `per_page` 1–1000 (default 50), `results_after` |
| `top/mentioners` | page | `page` 1–100 (default 1), `per_page` 1–100 (default 10) |
| `top/keywords`, `top/countries`, `top/languages` | one list | `limit` 1–10000 (default 10) |
| `top/summary/daily` | one list | `limit` 1–100 (default 2) |
| `tracker-groups/list`, `trackers/list` | page | `page` ≥ 1 (default 1), `limit` 1–100 (default 10) |

Cursor paging: send each response's `results_after` back as `results_after`, and stop when a page holds fewer items than `per_page`. The cursor follows publication time, so page through complete exports with the default `ordering=time`; use the other orderings for a single top-N page. `brand-overview`, `stats`, `audience-demographics`, `citation-analysis` and `account/usage` are not paged.

## Writing (beta)

- **Access:** write access is enabled per account on request, after review (`403 apiV2Write_not_available` otherwise), and the token's user must be an admin (`403 write_not_allowed`). A token limited to specific trackers cannot create trackers or create, change or delete tracker groups (`403 token_scope_restricted`).
- **Scope:** only **Keyword** trackers can be created. Profile and Location trackers can be updated and deleted. AI Visibility Prompt trackers, and tracker groups that contain them, can be changed in the web app only.
- **Partial updates:** only the fields sent change, and `null` clears a nullable field. Unknown or read-only keys answer `400`.
- **Confirm with the user before** deleting a tracker or group, or sending `existing_mentions: "delete"`: none of these can be undone through the API.
- **Plan allowance:** each tracker uses one keyword of the plan (`403 keyword_limit_reached`). Check `/v2/account/usage` before creating many.
- **Do not repeat a create that may have succeeded.** List the trackers first. A `201` whose body holds only `{id}` means the tracker exists.
- **Retries:** `409 location_changed`, `503` and the `520` codes of updates and deletes are safe to retry; a retry finishes an incomplete update or delete.

Bodies, field rules, examples and every error code: `references/write-endpoints.md`.

## Rate Limits & Caching (must respect)

Limits are **per account**: every token of the account shares the same counters.

| Requests | Burst | Sustained | Daily |
| :--- | :--- | :--- | :--- |
| Reads (`GET`) | 5 per second | 100 per 60 seconds | — |
| Writes (`POST`, `PUT`, `DELETE`) | 2 per 5 seconds | 10 per 60 seconds | 100 per day |

The write limits above are those of a plan that allows up to 25 trackers; a larger plan gets proportionally higher write limits, up to 20 times. They follow the plan, not the trackers in use. Read the headers rather than hardcoding numbers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (Unix seconds). They report the sustained limit, except on a burst `429`.

| `429` `error` | Meaning | Strategy |
| :--- | :--- | :--- |
| `rate_limit_exceeded` | Burst ("Burst limit exceeded…") or sustained ("Rate limit exceeded…") limit | Wait `Retry-After` seconds or until `X-RateLimit-Reset`, then retry with backoff |
| `daily_write_limit_exceeded` | The day's writes are used up | Stop writing; `Retry-After` gives the seconds until the reset |
| `keyword_edit_limit_reached` | `existing_mentions: "delete"` used too often on this tracker today | Retry tomorrow, or keep the existing mentions |

Successful `GET` responses are cached for **30 seconds** per user and identical request (method, URL and query). `Cache-Control: max-age=N` marks a cached response, `no-cache` a fresh one, and `X-Cache: COALESCED` a response shared with an identical read that was already running. Cached responses still count toward the limits, so don't poll the same request more often than every 30 seconds. Writes are never cached.

Full details, retry code and error shapes: `references/auth-rate-limits.md`.

## Errors

Errors are JSON `{"status": 403, "error": "<code>", "message": "..."}`, except validation errors: `400 {"errors": [{"field": "...", "message": "..."}]}`. Branch on `error`, not on `message`.

| HTTP | Meaning | Action |
| :--- | :--- | :--- |
| `400` | Invalid parameter, value or body | Fix the request; don't retry it unchanged |
| `401` | `unauthorized`, `invalid_token`, `token_expired`, `invalid_token_options` | Refresh once, then retry |
| `403` | `access_denied`, `account_blocked`, `apiV2_not_available`, `account_inactive`; `ai_visibility_not_available` on answers; plan and permission codes on writes | Tell the user; don't retry |
| `404` | Tracker, tracker group or answer not found | Check the ID |
| `409` | A Location tracker was changed by another request at the same time | Retry |
| `422` | The write can't apply to this tracker (code in `error`) | Fix the request |
| `429` | Rate or daily limit | See above |
| `503` | A boolean query couldn't be checked; nothing was saved | Retry |
| `520` | `internal_error`, or an update or delete that didn't finish | Retry with backoff |

## Known Data Caveats

- Content from X (Twitter), LinkedIn and Web News may be subject to strict research caps depending on the plan, so API results might not exactly match the Mentionlytics web app for these sources. Mention this proactively if the user compares API output with the dashboard.
- The pages an answer cites are analyzed 10 minutes or more after the answer is collected, so a new answer's citations have no `analysis` yet.
- Days are the account's time zone: `totals[].pub_date` is that day given as midnight UTC.

## Reference Files

- `references/endpoints.md`: mention and analytics endpoints, tracker and tracker group lists, account usage. Parameters, examples and response shapes.
- `references/answers.md`: AI Visibility answers, brand overview and citation analysis.
- `references/write-endpoints.md`: creating, updating, pausing, moving and deleting trackers and tracker groups. Bodies, field rules and error codes.
- `references/channels-categories.md`: channel IDs and category tokens per platform. Read whenever filtering by source.
- `references/auth-rate-limits.md`: token rotation, rate limits, retry code, caching and coalescing. Read when writing retry logic or long-running scripts.
