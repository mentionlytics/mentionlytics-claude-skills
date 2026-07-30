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

```bash
curl "https://api.mentionlytics.com/v2/mentions?startDate=20260701&endDate=20260722&trackers=brandmonitoring&relevance=normal&ordering=time&per_page=50" \
  -H "Authorization: Bearer $MENTIONLYTICS_API_TOKEN"
```

**Expected response (200):** `{ mentions: [Mention], totals: [{pub_date, total_count}], results_after, tracker_totals? }`

```json
{
  "mentions": [
    {
      "id": 123456789,
      "uu_id": "abc123def456",
      "tracker": { "id": 456, "name": "Brand Keywords" },
      "trackerGroup": { "id": 123, "name": "Brand Monitoring" },
      "pub_date": "2024-05-20",
      "pub_datetime": "2024-05-20T14:30:00Z",
      "title": "Great product review",
      "description": "This is the full text of the mention...",
      "link": "https://example.com/post/123",
      "relevance": 0.85,
      "sentiment_text": "positive",
      "emotion_text": "joy",
      "country_code": "US", "country_name": "United States",
      "language_code": "en", "language_name": "English",
      "channel_name": "twitter", "channel_id": 2,
      "contentType": "text", "category_id": "tweets", "category_name": "Tweets",
      "type_name": "news",
      "entities": [ { "type": "PERSON", "value": "John Doe" } ],
      "profile": {
        "id": 789, "name": "John Doe", "username": "johndoe",
        "link": "https://twitter.com/johndoe",
        "followers_count": 5000, "verified": false, "type_name": "user"
      },
      "metrics": {
        "engagement": 1500, "engagement_no_views": 150,
        "likes": 100, "shares": 25, "comments": 25, "quotes": 10,
        "views": 1350, "impressions": 2000, "reach": 5000
      }
    }
  ],
  "totals": [ { "pub_date": "2024-05-20", "total_count": 42 } ],
  "results_after": 1716240000000
}
```

The full `Mention` object also carries (when applicable): `body`, `subreddit`, `subChannel`, `discover_at`, `updated_at`, `image`, `thumb_url`, `metrics.reactions` (Facebook/Telegram: `love`, `wow`, `haha`, `sad`, `angry`, `thankful`, `fire`, `clap`, `dislike`), `metrics.review`, `metrics.global_rank`, `metrics.country_rank`, `parent` (for comments/replies, up to 2 levels), and `workflow` (`assigned_to`, `labels`, `notes`, `tags`). When `tracker_totals=true`, a top-level `tracker_totals` array is added: `[{ id, name, type: "trackerGroup", total, trackers: [{ id, name, type: "tracker", total }] }]`.

<a name="top-mentioners"></a>
## GET /v2/mentions/top/mentioners

Top profiles/influencers mentioning the tracked terms.

**Pagination — page-based:**
- `per_page` *(integer, 1–100, default 10)*
- `page` *(integer, 1–100, default 1)*

**Ordering** — `ordering` *(default `reach`)*: one of `mentions`, `reach`, `engagement`, `engagement_nv`. (`reach` = most influential, `mentions` = most active.)
- `sortDesc` *(boolean, default true)* — descending order.

Supports the shared filter parameters.

```bash
curl "https://api.mentionlytics.com/v2/mentions/top/mentioners?ordering=reach&per_page=20&page=1&trackers=brandmonitoring" \
  -H "Authorization: Bearer $MENTIONLYTICS_API_TOKEN"
```

**Expected response (200):** `{ mentioners: [Mentioner] }`

```json
{
  "mentioners": [
    {
      "id": 789, "uu_id": "abc123def456",
      "channel_id": 2, "channel_name": "twitter",
      "name": "John Doe", "username": "johndoe",
      "description": "Tech enthusiast and blogger",
      "link": "https://twitter.com/johndoe",
      "country": "US", "language": "en",
      "type_name": "user", "verified": true, "verified_badge": "blue",
      "geolocation": { "latitude": 40.7128, "longitude": -74.0060, "city_name": "New York", "country_name": "United States" },
      "metrics": {
        "followers_count": 5000, "following_count": 500, "post_count": 1200,
        "view_count": 100000, "global_rank": 5000, "country_rank": 500
      },
      "total": {
        "engagement": 1500, "reach": 5000, "mentions": 25,
        "positive_mentions": 13, "neutral_mentions": 10, "negative_mentions": 2
      }
    }
  ]
}
```

For web sources, `metrics` also carries `users` and `visits` instead of social follower counts.

<a name="top-keywords"></a>
## GET /v2/mentions/top/keywords

Trending keywords and hashtags.

- **Ordering** — `ordering`: one of `mentions`, `reach`, `engagement`, `engagement_nv` (no fixed default — server default applies).
- `sortDesc` *(boolean, default true)*
- `limit` *(integer, 1–10000, default 10)* — number of keywords to return.
- `onlyHashtags` *(boolean, default false)* — restrict results to hashtags (keywords starting with `#`).
- `adjacentWords` *(integer, one of 0,1,2,3, default 1)* — phrase length: `0` = single words, `1` = 2-word phrases, etc.
- `ignoreMainKeywords` *(boolean, default true)* — exclude your main tracked keywords from results.

Supports the shared filter parameters.

**Expected response (200):** `{ keywords: [Keyword] }`

```json
{
  "keywords": [
    {
      "keyword": "customer service",
      "weight": 150,
      "adjacentWords": 2,
      "total": {
        "mentions": 50, "engagement": 3000, "engagement_nv": 1000, "reach": 15000,
        "positive_mentions": 25, "neutral_mentions": 20, "negative_mentions": 5
      }
    }
  ]
}
```

`weight` is the count for the selected `ordering` metric (mentions / engagement / engagement_nv / reach).

<a name="top-countries"></a>
## GET /v2/mentions/top/countries

Geographic mention breakdown by country.

- **Ordering** — `ordering` *(default `mentions`)*: one of `mentions`, `reach`, `engagement`, `engagement_nv`.
- `sortDesc` *(boolean, default true)*
- `limit` *(integer, 1–10000, default 10)*

Supports the shared filter parameters.

**Expected response (200):** `{ countries: [Country] }`

```json
{
  "countries": [
    {
      "country_code": "US", "country_name": "United States",
      "total": { "mentions": 150, "engagement": 5000, "engagement_nv": 500, "reach": 25000 }
    }
  ]
}
```

<a name="top-languages"></a>
## GET /v2/mentions/top/languages

Language distribution of mentions.

- **Ordering** — `ordering` *(default `mentions`)*: one of `mentions`, `reach`, `engagement`, `engagement_nv`.
- `sortDesc` *(boolean, default true)*
- `limit` *(integer, 1–10000, default 10)*

Supports the shared filter parameters.

**Expected response (200):** `{ languages: [Language] }`

```json
{
  "languages": [
    {
      "language_code": "en", "language_name": "English",
      "total": { "mentions": 200, "engagement": 8000, "engagement_nv": 800, "reach": 40000 }
    }
  ]
}
```

<a name="daily-summary"></a>
## GET /v2/mentions/top/summary/daily

AI-generated daily summaries of mention clusters. Auto-calculated every morning covering the past **24 hours**.

**Accepts only this parameter subset** (not the full shared set):
- `fields` *(string)*
- `startDate` / `endDate` *(YYYYMMDD)*
- `trackers` *(default `brandmonitoring`)* — one of `brandmonitoring`, `competitors`, `all`. **Note: `leads` is NOT valid here.**
- `limit` *(integer, 1–100, default 2)* — number of daily summaries to return.

**Expected response (200):** `{ summary: [DailySummary] }`

```json
{
  "summary": [
    {
      "type": "brandmonitoring",
      "created_at": "2026-02-11 14:00",
      "description": "Daily brand monitoring summary",
      "clusters": [
        {
          "description": ["Great product review!", "Amazing customer service"],
          "mention_ids": [123456, 123457, 123458],
          "total": {
            "mentions": 45, "reach": 125000, "engagement": 5600,
            "positive_mentions": 30, "negative_mentions": 5, "neutral_mentions": 10,
            "web_mentions": 15, "twitter_mentions": 12, "retweet_mentions": 8,
            "facebook_mentions": 5, "youtube_mentions": 2, "instagram_mentions": 3, "others_mentions": 0
          },
          "tracker_totals": [
            { "id": 101, "name": "Brand Monitoring", "type": "trackerGroup", "total": 45,
              "trackers": [ { "id": 201, "name": "Brand Name Mentions", "type": "tracker", "total": 25 } ] }
          ]
        }
      ]
    }
  ]
}
```

`type` is `brandmonitoring` or `competitors`; `created_at` is UTC.

<a name="stats"></a>
## GET /v2/mentions/stats

High-level analytics overview: mention counts (web/social/all), reach, `unique_reach` per channel, engagement / `unique_engagement` / `impression_engagement`, sentiment breakdown by channel, EMV (Earned Media Value) and MQI (Mention Quality Index).

Accepts the shared filter parameters. **No pagination, ordering, or limit.**

**Expected response (200):** `{ stats: MentionStats }`

```json
{
  "stats": {
    "mentions": {
      "web": { "total": 150, "sites": 45, "unique": 120 },
      "social": { "total": 500, "users": 200, "unique": 450 },
      "all": { "period": 650, "total": 650, "all": 1500, "unique": 570 }
    },
    "reach": { "total": 250000 },
    "unique_reach": {
      "gg": 50000, "tw": 75000, "fb": 60000, "yt": 30000, "ig": 25000, "rv": 5000,
      "rd": 10000, "tk": 15000, "ln": 8000, "pn": 3000, "tl": 2000, "th": 5000,
      "bs": 1000, "mn": 500, "ot": 2500, "total": 292000
    },
    "engagement": {
      "likes": 5000, "comments": 1200, "shares": 1500, "views": 75000,
      "retweets": 800, "retweets_unique": 750, "tweet_views": 50000,
      "reviews": 86.0, "reviews_count": 20, "reviews_average": 4.3,
      "impressions": 100000, "reach": 250000,
      "tiktok_views": 20000, "tiktok_shares": 300, "taps_forward": 500, "taps_back": 150
    },
    "unique_engagement": { "…": "same shape as engagement, deduplicated" },
    "impression_engagement": { "…": "same shape as engagement, mentions with impression data only" },
    "sentiment": {
      "positive": { "web": 80, "twitter": 150, "facebook": 120, "youtube": 50, "instagram": 90, "reviews": 45, "others": 30, "total": 565 },
      "negative": { "…": "same per-channel shape", "total": 113 },
      "neutral":  { "…": "same per-channel shape", "total": 252 }
    },
    "emv": { "currency": "usd", "total": 125430.50, "web": 45230.25, "twitter": 32150.75, "facebook": 28540.00, "youtube": 10320.50, "instagram": 7189.00, "reviews": 1500.00, "others": 500.00 },
    "mqi": { "type": "avg", "total": 75.50, "web": 82.30, "twitter": 68.75, "facebook": 71.20, "youtube": 79.40, "instagram": 73.85, "reviews": 88.60, "others": 65.90 }
  }
}
```

- `emv` carries a `currency` code plus per-channel values (`web`, `twitter`, `facebook`, `youtube`, `instagram`, `reviews`, `others`, `total`).
- `mqi.type` is `avg` or `sum`; the per-channel breakdown mirrors `emv`.
- `unique_engagement` and `impression_engagement` share the exact field set of `engagement`.

<a name="audience-demographics"></a>
## GET /v2/mentions/audience-demographics

Age/gender demographics for **owned** Facebook and Instagram media channels.

**Accepts only this parameter subset:**
- `fields` *(string)*
- `endDate` *(YYYYMMDD, default today)* — **there is no `startDate` on this endpoint.**
- `trackers`, `trackerGroups`
- `filterGenderByAge` *(string)* — one of `13-17`, `18-24`, `25-34`, `35-44`, `45-54`, `55-64`, `65+`.
- `filterAgeByGender` *(string)* — one of `male`, `female`, `unspecified`.

**Expected response (200):** `{ demographics: Demographics }`

```json
{
  "demographics": {
    "gender": { "male": 15000, "female": 18000, "unspecified": 2000 },
    "ages": { "13-17": 2000, "18-24": 8000, "25-34": 12000, "35-44": 7000, "45-54": 4000, "55-64": 2500, "65+": 1500 }
  }
}
```

<a name="tracker-groups"></a>
## GET /v2/tracker-groups/list

List tracker groups.

- `fields` *(string)*
- `trackerGroupsCategory` *(default `all`)* — one of `all`, `brandmonitoring`, `competitors`, `leads`.
- `page` *(integer, default 1)*
- `limit` *(integer, default 10)*

**Expected response (200):** `{ trackerGroups: [TrackerGroup] }`

```json
{
  "trackerGroups": [
    {
      "id": 42, "name": "My Brand",
      "excluded_keywords": "spam irrelevant",
      "created_at": "2023-06-15",
      "type": "trackerGroup",
      "category_id": 0, "category_name": "Brand Monitoring",
      "options": { "ignore_duplicates": 0, "color": "#3b82f6" },
      "total": { "keywords": 5, "profiles": 2 }
    }
  ]
}
```

`category_id`/`category_name`: `0`=Brand Monitoring, `1`=Competitors, `2`=Marketing Leads.

<a name="trackers"></a>
## GET /v2/trackers/list

List individual trackers (keyword / profile / location queries).

- `fields` *(string)*
- `category` *(default `all`)* — one of `all`, `keyword`, `profile`, `location`. (This is the tracker-type filter.)
- `trackerGroups` *(string)* — comma-separated tracker group IDs.
- `trackerGroupsCategory` *(default `all`)* — filter by parent group category: `all`, `brandmonitoring`, `competitors`, `leads`.
- `page` *(integer, default 1)*
- `limit` *(integer, default 10)*

**Expected response (200):** `{ trackers: [Tracker] }`

```json
{
  "trackers": [
    {
      "id": 101, "name": "Brand Keywords",
      "type": "tracker",
      "category_id": 0, "category_name": "Keyword",
      "is_active": true,
      "created_at": "2023-06-15T10:00:00Z", "updated_at": "2024-01-20T08:30:00Z",
      "tracker_group": { "id": 42, "name": "My Brand", "type": "trackerGroup", "category_id": 0, "category_name": "Brand Monitoring" },
      "keyword": {
        "main_keyword": "mentionlytics",
        "included_keywords": "social media, monitoring",
        "excluded_keywords": "spam, irrelevant",
        "sources": { "web": true, "twitter": true, "facebook": false, "instagram": false }
      }
    },
    {
      "id": 102, "name": "Brand Instagram",
      "type": "tracker",
      "category_id": 1, "category_name": "Profile",
      "is_active": true,
      "tracker_group": { "id": 42, "name": "My Brand", "type": "trackerGroup", "category_id": 0, "category_name": "Brand Monitoring" },
      "profile": {
        "id": "1234567890", "type": "instagram", "name": "mybrand",
        "url": "https://www.instagram.com/mybrand",
        "trackposts": true, "is_authorized": true, "allow_publishing": true
      }
    }
  ]
}
```

- `category_id`/`category_name`: `0`=Keyword, `1`=Profile, `2`=Location.
- The `keyword` object is present **only on Keyword trackers**; the `profile` object is present **only on Profile and Location trackers**. Inspect `category_name` before parsing. Keyword trackers additionally expose `keyword.options`, `keyword.sources` (per-channel booleans), and `keyword.restrictions` (`site_search`, `language_code`, `country_code`, `subreddit`).

---

## Error responses (all endpoints)

Every endpoint returns these error shapes. Validate on the `status` + `error` fields.

| HTTP | `error` values | Shape |
| :--- | :--- | :--- |
| `400` | (validation) | `{ "errors": [{ "field": "...", "message": "..." }] }` |
| `401` | `unauthorized`, `invalid_token`, `token_expired`, `invalid_token_options` | `{ "status": 401, "error": "...", "message": "..." }` |
| `403` | `access_denied`, `account_blocked`, `apiV2_not_available`, `account_inactive` | `{ "status": 403, "error": "...", "message": "..." }` |
| `429` | `rate_limit_exceeded` (sustained) **or** `burst_limit_exceeded` (burst) | `{ "status": 429, "error": "...", "message": "..." }` |
| `520` | `internal_error` | `{ "status": 520, "error": "internal_error", "message": "..." }` |

`POST /v2/auth/refresh` additionally returns `400` with `error: "bad_request"` (e.g. `message: "missing_refresh_token"`) and `401` with `error` one of `invalid_refresh_token`, `token_expired`, `token_revoked`.

```json
{ "status": 401, "error": "token_expired", "message": "Authentication required" }
```

---

## Data discrepancy notice

Content from X (Twitter), LinkedIn, and Web News may be subject to strict research caps under certain plans. API data might not precisely mirror the main web application for these sources.
