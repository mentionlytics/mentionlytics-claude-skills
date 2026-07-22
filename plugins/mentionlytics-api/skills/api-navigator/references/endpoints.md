# Endpoint Reference

Base URL: `https://api.mentionlytics.com`
All endpoints require `Authorization: Bearer <token>` header unless noted.

## Contents
1. [POST /v2/auth/refresh](#auth-refresh)
2. [GET /v2/mentions](#mentions)
3. [GET /v2/mentions/top/mentioners](#top-mentioners)
4. [GET /v2/mentions/top/keywords](#top-keywords)
5. [GET /v2/mentions/top/countries](#top-countries)
6. [GET /v2/mentions/top/languages](#top-languages)
7. [GET /v2/mentions/top/summary/daily](#daily-summary)
8. [GET /v2/mentions/stats](#stats)
9. [GET /v2/mentions/audience-demographics](#audience-demographics)
10. [GET /v2/tracker-groups/list](#tracker-groups)
11. [GET /v2/trackers/list](#trackers)
12. [GET /v2/account/usage](#account-usage)

## Common query parameters

These apply to most `/v2/mentions*` endpoints:

| Parameter | Type | Values / Format | Default |
| :--- | :--- | :--- | :--- |
| `startDate` | string | `YYYYMMDD` | 7 days ago |
| `endDate` | string | `YYYYMMDD` | today |
| `relevance` | string | `all`, `low` (≥0.25), `normal` (≥0.5), `strict` (≥0.75) | — |
| `trackers` | string | `brandmonitoring`, `competitors`, `leads`, `all`, or comma-separated tracker IDs | — |
| `channels` | string | comma-separated channel IDs (see channels-categories.md) | all |
| `categories` | string | comma-separated category tokens (see channels-categories.md) | all |
| `ordering` | string | `time`, `rank`, `engagement`, `engagement_nv` | — |

---

<a name="auth-refresh"></a>
## POST /v2/auth/refresh

Exchange a long-lived refresh token for a fresh bearer token. Call automatically on any 401.

**Payload (application/json):**
- `refresh_token` *(string, required)* — the refresh token issued during initial authentication.

```bash
curl -X POST "https://api.mentionlytics.com/v2/auth/refresh" \
  -H "Content-Type: application/json" \
  -d '{"refresh_token": "'"$MENTIONLYTICS_REFRESH_TOKEN"'"}'
```

Response contains a new bearer token AND a new refresh token. The old refresh token is invalidated immediately — persist both new values.

<a name="mentions"></a>
## GET /v2/mentions

Fetch individual social media mentions.

- **Pagination:** timestamp-based cursor via `results_after` (pass the timestamp of the last mention received to get the next batch). Do not use `page`/`per_page` here.
- Supports all common parameters above.

```bash
curl "https://api.mentionlytics.com/v2/mentions?startDate=20260701&endDate=20260722&trackers=brandmonitoring&relevance=normal&ordering=engagement" \
  -H "Authorization: Bearer $MENTIONLYTICS_API_TOKEN"
```

<a name="top-mentioners"></a>
## GET /v2/mentions/top/mentioners

Top profiles/influencers mentioning the tracked terms.

- **Pagination:** `page`, `per_page` (max 100).
- Supports common parameters.

<a name="top-keywords"></a>
## GET /v2/mentions/top/keywords

Trending keywords and hashtags.

- `onlyHashtags` — restrict results to hashtags.
- `adjacentWords` — integer 0 to 3, number of adjacent words to include.
- Supports common parameters.

<a name="top-countries"></a>
## GET /v2/mentions/top/countries

Geographic mention breakdown, volume sorted by metric. Max `limit` 10000.

<a name="top-languages"></a>
## GET /v2/mentions/top/languages

Language distribution of mentions. Max `limit` 10000.

<a name="daily-summary"></a>
## GET /v2/mentions/top/summary/daily

AI-generated daily summaries, auto-generated every morning covering the past 24 hours.

<a name="stats"></a>
## GET /v2/mentions/stats

High-level analytics overview, including EMV (Earned Media Value) and MQI (Mention Quality Index).

<a name="audience-demographics"></a>
## GET /v2/mentions/audience-demographics

Age/gender demographics for owned Facebook and Instagram media channels.

<a name="tracker-groups"></a>
## GET /v2/tracker-groups/list

Structured groups of trackers. Filter via `trackerGroupsCategory`.

<a name="trackers"></a>
## GET /v2/trackers/list

Active keyword/profile queries. Response contains distinct structures for `keyword.*` and `profile.*` fields — inspect the type before parsing.

<a name="account-usage"></a>
## GET /v2/account/usage

Subscription quotas and usage: allowances, reset cycles, and Brand Hub states. Check this before large batch pulls.

---

## Data discrepancy notice

Content from X (Twitter), LinkedIn, and Web News may experience strict research caps under certain plans. API data might not precisely mirror the main web application interface for these sources.
