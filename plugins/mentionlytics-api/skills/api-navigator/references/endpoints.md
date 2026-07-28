# Endpoint Reference

Base URL: `https://api.mentionlytics.com`
All endpoints require the `Authorization: Bearer <token>` header.

> **Parameters are NOT uniform across endpoints.** The `ordering` values, pagination style, defaults, and available filters differ per endpoint. Always check the specific endpoint section below — do not assume a parameter behaves the same everywhere. The `ordering` parameter in particular uses **two different value sets** (see below).

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

---

## Shared filter parameters

These filters are accepted by the mention analytics endpoints — `/v2/mentions`, `/v2/mentions/top/mentioners`, `/v2/mentions/top/keywords`, `/v2/mentions/top/countries`, `/v2/mentions/top/languages`, and `/v2/mentions/stats`. (`/v2/mentions/top/summary/daily` and `/v2/mentions/audience-demographics` accept only a subset — see their sections.)

| Parameter | Type | Values / Format | Default |
| :--- | :--- | :--- | :--- |
| `fields` | string | Comma-separated list of fields to include in the response | all |
| `startDate` | string | `YYYYMMDD` | 7 days ago |
| `endDate` | string | `YYYYMMDD` | today |
| `relevance` | string | `all`, `low` (≥0.25), `normal` (≥0.5), `strict` (≥0.75) | `normal` |
| `trackers` | string | `brandmonitoring`, `competitors`, `leads`, `all`, or comma-separated tracker IDs | `brandmonitoring` |
| `trackerGroups` | string | Comma-separated tracker group IDs | — |
| `preset` | integer | ID of a saved filter preset (≥1) | — |
| `channels` | string | Comma-separated channel IDs (see channels-categories.md) | all |
| `categories` | string | Comma-separated category tokens (see channels-categories.md) | all |
| `contentType` | string | `image`, `album`, `video`, `audio`, `link`, `webpage`, `text`, `other` | — |
| `sentiment` | string | `positive`, `neutral`, `negative` | — |
| `emotion` | string | `joy`, `anger`, `sadness`, `fear`, `sarcasm`, `neutral` | — |
| `country` | string | Comma-separated ISO country codes | — |
| `language` | string | Comma-separated ISO language codes | — |

> **`ordering` is NOT shared** — its allowed values and default depend on the endpoint. See each section.

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

**Response (200):**
```json
{
  "bearer": "<new bearer token, valid 1 hour>",
  "refresh_token": "<new refresh token, replaces the old one>"
}
```

The old refresh token is invalidated immediately — persist BOTH new values. On a failed refresh the `error` field is one of `invalid_refresh_token`, `token_expired`, or `token_revoked`; in that case the user must re-authenticate (do not loop).

<a name="mentions"></a>
## GET /v2/mentions

Fetch individual social/web mentions, one object per mention.

**Pagination — cursor-based:**
- `per_page` *(integer, 1–10000, default 50)* — results per page.
- `results_after` *(integer)* — timestamp cursor. Pass the `results_after` value returned by the previous response to fetch the next page. There is **no `page` parameter** on this endpoint.

**Ordering** — `ordering` *(default `time`)*:

| Value | Meaning |
| :--- | :--- |
| `time` | Publication date, newest first |
| `rank` | Reach and rank, most influential first |
| `engagement` | Total engagement (likes, shares, comments, views — excludes X/Twitter views) |
| `engagement_nv` | Engagement without views |

**Additional filters (beyond the shared set):**
- `mention_ids` *(string)* — comma-separated mention IDs to fetch specific mentions.
- `tracker_totals` *(boolean, default false)* — when `true`, response includes a `tracker_totals` array with counts grouped by tracker group and tracker.

**Response shape:** `{ mentions: [...], totals: [{pub_date, total_count}], results_after, tracker_totals? }`.

```bash
curl "https://api.mentionlytics.com/v2/mentions?startDate=20260701&endDate=20260722&trackers=brandmonitoring&relevance=normal&ordering=time&per_page=50" \
  -H "Authorization: Bearer $MENTIONLYTICS_API_TOKEN"
```

<a name="top-mentioners"></a>
## GET /v2/mentions/top/mentioners

Top profiles/influencers mentioning the tracked terms.

**Pagination — page-based:**
- `per_page` *(integer, 1–100, default 10)*
- `page` *(integer, 1–100, default 1)*

**Ordering** — `ordering` *(default `reach`)*: one of `mentions`, `reach`, `engagement`, `engagement_nv`. (`reach` = most influential, `mentions` = most active.)
- `sortDesc` *(boolean, default true)* — descending order.

Supports the shared filter parameters. Response: `{ mentioners: [...] }`.

```bash
curl "https://api.mentionlytics.com/v2/mentions/top/mentioners?ordering=reach&per_page=20&page=1&trackers=brandmonitoring" \
  -H "Authorization: Bearer $MENTIONLYTICS_API_TOKEN"
```

<a name="top-keywords"></a>
## GET /v2/mentions/top/keywords

Trending keywords and hashtags.

- **Ordering** — `ordering`: one of `mentions`, `reach`, `engagement`, `engagement_nv` (no fixed default — server default applies).
- `sortDesc` *(boolean, default true)*
- `limit` *(integer, 1–10000, default 10)* — number of keywords to return.
- `onlyHashtags` *(boolean, default false)* — restrict results to hashtags (keywords starting with `#`).
- `adjacentWords` *(integer, one of 0,1,2,3, default 1)* — phrase length: `0` = single words, `1` = 2-word phrases, etc.
- `ignoreMainKeywords` *(boolean, default true)* — exclude your main tracked keywords from results.

Supports the shared filter parameters. Response: `{ keywords: [{keyword, weight, adjacentWords, total}] }`.

<a name="top-countries"></a>
## GET /v2/mentions/top/countries

Geographic mention breakdown by country.

- **Ordering** — `ordering` *(default `mentions`)*: one of `mentions`, `reach`, `engagement`, `engagement_nv`.
- `sortDesc` *(boolean, default true)*
- `limit` *(integer, 1–10000, default 10)*

Supports the shared filter parameters. Response: `{ countries: [{country_code, country_name, total}] }`.

<a name="top-languages"></a>
## GET /v2/mentions/top/languages

Language distribution of mentions.

- **Ordering** — `ordering` *(default `mentions`)*: one of `mentions`, `reach`, `engagement`, `engagement_nv`.
- `sortDesc` *(boolean, default true)*
- `limit` *(integer, 1–10000, default 10)*

Supports the shared filter parameters. Response: `{ languages: [{language_code, language_name, total}] }`.

<a name="daily-summary"></a>
## GET /v2/mentions/top/summary/daily

AI-generated daily summaries of mention clusters. Auto-calculated every morning covering the past **24 hours**.

**Accepts only this parameter subset** (not the full shared set):
- `fields` *(string)*
- `startDate` / `endDate` *(YYYYMMDD)*
- `trackers` *(default `brandmonitoring`)* — one of `brandmonitoring`, `competitors`, `all`. **Note: `leads` is NOT valid here.**
- `limit` *(integer, 1–100, default 2)* — number of daily summaries to return.

Response: `{ summary: [DailySummary] }`, where each summary has `type`, `created_at`, `description`, and `clusters[]` (each cluster carries sample `description[]`, `mention_ids[]`, `total` metrics, and `tracker_totals[]`).

<a name="stats"></a>
## GET /v2/mentions/stats

High-level analytics overview: mention counts (web/social/all), reach, `unique_reach` per channel, engagement / `unique_engagement` / `impression_engagement`, sentiment breakdown by channel, EMV (Earned Media Value) and MQI (Mention Quality Index).

Accepts the shared filter parameters. **No pagination, ordering, or limit.** Response: `{ stats: MentionStats }`.

- `emv` carries a `currency` code plus per-channel values (`web`, `twitter`, `facebook`, `youtube`, `instagram`, `reviews`, `others`, `total`).
- `mqi` carries `type` (`avg` or `sum`) plus the same per-channel breakdown.

<a name="audience-demographics"></a>
## GET /v2/mentions/audience-demographics

Age/gender demographics for **owned** Facebook and Instagram media channels.

**Accepts only this parameter subset:**
- `fields` *(string)*
- `endDate` *(YYYYMMDD, default today)* — **there is no `startDate` on this endpoint.**
- `trackers`, `trackerGroups`
- `filterGenderByAge` *(string)* — one of `13-17`, `18-24`, `25-34`, `35-44`, `45-54`, `55-64`, `65+`.
- `filterAgeByGender` *(string)* — one of `male`, `female`, `unspecified`.

Response: `{ demographics: { gender: {male, female, unspecified}, ages: {"13-17", ...} } }`.

<a name="tracker-groups"></a>
## GET /v2/tracker-groups/list

List tracker groups.

- `fields` *(string)*
- `trackerGroupsCategory` *(default `all`)* — one of `all`, `brandmonitoring`, `competitors`, `leads`.
- `page` *(integer, default 1)*
- `limit` *(integer, default 10)*

Each group carries `id`, `name`, `type` (`trackerGroup`), `category_id` (0,1,2), and `category_name` (`Brand Monitoring`, `Competitors`, `Marketing Leads`).

<a name="trackers"></a>
## GET /v2/trackers/list

List individual trackers (keyword / profile / location queries).

- `fields` *(string)*
- `category` *(default `all`)* — one of `all`, `keyword`, `profile`, `location`. (This is the tracker-type filter.)
- `trackerGroups` *(string)* — comma-separated tracker group IDs.
- `trackerGroupsCategory` *(default `all`)* — filter by parent group category: `all`, `brandmonitoring`, `competitors`, `leads`.
- `page` *(integer, default 1)*
- `limit` *(integer, default 10)*

Each tracker carries `id`, `name`, `type` (`tracker`), `category_id` (0,1,2), and `category_name` (`Keyword`, `Profile`, `Location`). Profile trackers additionally expose profile fields (platform identifier, username/page name, link) — inspect `category_name` before parsing.

---

## Data discrepancy notice

Content from X (Twitter), LinkedIn, and Web News may be subject to strict research caps under certain plans. API data might not precisely mirror the main web application for these sources.
