# Endpoint Reference

Base URL: `https://api.mentionlytics.com` · all endpoints require `Authorization: Bearer <token>`.

> **Parameters are NOT uniform.** `ordering`, pagination, defaults, and filters differ per endpoint — check the specific section. `ordering` uses **two different value sets** (see below).

## Shared filter parameters

Accepted by `/v2/mentions`, `top/mentioners`, `top/keywords`, `top/countries`, `top/languages`, `stats`. (`top/summary/daily` and `audience-demographics` take only a subset — see their rows.)

| Param | Values / Format | Default |
| :--- | :--- | :--- |
| `fields` | comma-separated fields to include | all |
| `startDate` / `endDate` | `YYYYMMDD` | 7 days ago / today |
| `relevance` | `all`, `low` (≥0.25), `normal` (≥0.5), `strict` (≥0.75) | `normal` |
| `trackers` | `brandmonitoring`, `competitors`, `leads`, `all`, or comma-separated IDs | `brandmonitoring` |
| `trackerGroups` | comma-separated group IDs | — |
| `preset` | saved filter preset ID (≥1) | — |
| `channels` | comma-separated channel IDs (see channels-categories.md) | all |
| `categories` | comma-separated category tokens (see channels-categories.md) | all |
| `contentType` | `image`, `album`, `video`, `audio`, `link`, `webpage`, `text`, `other` | — |
| `sentiment` | `positive`, `neutral`, `negative` | — |
| `emotion` | `joy`, `anger`, `sadness`, `fear`, `sarcasm`, `neutral` | — |
| `country` / `language` | comma-separated ISO codes | — |

**`ordering` — two families (values + default differ):**
- **`/v2/mentions`:** `time` (default) · `rank` · `engagement` · `engagement_nv`
- **top-lists** (`mentioners`, `keywords`, `countries`, `languages`): `mentions` · `reach` · `engagement` · `engagement_nv`. Default: `reach` (mentioners), `mentions` (countries/languages). These also accept `sortDesc` (bool, default true).

`engagement` = likes + shares + comments + views (excl. X/Twitter views); `engagement_nv` = engagement without views.

---

## POST /v2/auth/refresh
Body `{ "refresh_token": "..." }`. Returns `{ "bearer": "...", "refresh_token": "..." }` — persist BOTH; the old refresh token is invalidated. Failure `error`: `invalid_refresh_token` | `token_expired` | `token_revoked` → re-authenticate (don't loop).

## GET /v2/mentions
Individual mentions. **Cursor pagination:** `per_page` (1–10000, default 50) + `results_after` (timestamp from previous response). No `page`. `ordering` default `time`. Extra filters: `mention_ids` (comma-sep), `tracker_totals` (bool, default false).

Response `{ mentions[], totals[], results_after, tracker_totals? }`:
```json
{
  "mentions": [{
    "id": 123456789, "uu_id": "abc123", "pub_date": "2024-05-20", "pub_datetime": "2024-05-20T14:30:00Z",
    "title": "Great review", "link": "https://example.com/123", "relevance": 0.85,
    "sentiment_text": "positive", "emotion_text": "joy",
    "country_code": "US", "language_code": "en", "channel_id": 2, "channel_name": "twitter",
    "contentType": "text", "category_id": "tweets", "type_name": "news",
    "tracker": {"id": 456, "name": "Brand Keywords"}, "trackerGroup": {"id": 123, "name": "Brand Monitoring"},
    "profile": {"id": 789, "username": "johndoe", "followers_count": 5000, "verified": false},
    "metrics": {"engagement": 1500, "engagement_no_views": 150, "likes": 100, "shares": 25, "comments": 25, "views": 1350, "reach": 5000}
  }],
  "totals": [{"pub_date": "2024-05-20", "total_count": 42}],
  "results_after": 1716240000000
}
```
`Mention` may also include `body`, `subreddit`, `subChannel`, `image`, `thumb_url`, `metrics.reactions` (love/wow/haha/…), `metrics.review`, `parent` (comments/replies), `workflow` (assigned_to/labels/notes/tags). With `tracker_totals=true`: `tracker_totals: [{id, name, type:"trackerGroup", total, trackers:[{id, name, type:"tracker", total}]}]`.

## GET /v2/mentions/top/mentioners
Top profiles. **Page pagination:** `page` (1–100, default 1) + `per_page` (1–100, default 10). `ordering` default `reach`, `sortDesc`.
```json
{ "mentioners": [{
  "id": 789, "channel_id": 2, "channel_name": "twitter", "username": "johndoe",
  "country": "US", "verified": true,
  "metrics": {"followers_count": 5000, "following_count": 500, "post_count": 1200},
  "total": {"mentions": 25, "reach": 5000, "engagement": 1500, "positive_mentions": 13, "neutral_mentions": 10, "negative_mentions": 2}
}] }
```
Web sources return `metrics.users`/`visits`/`global_rank`/`country_rank` instead of follower counts.

## GET /v2/mentions/top/keywords
`ordering` (`mentions`/`reach`/`engagement`/`engagement_nv`), `sortDesc`, `limit` (1–10000, default 10), `onlyHashtags` (bool, default false), `adjacentWords` (0–3, default 1: 0=single word, 1=2-word phrase…), `ignoreMainKeywords` (bool, default true).
```json
{ "keywords": [{
  "keyword": "customer service", "weight": 150, "adjacentWords": 2,
  "total": {"mentions": 50, "engagement": 3000, "engagement_nv": 1000, "reach": 15000, "positive_mentions": 25, "neutral_mentions": 20, "negative_mentions": 5}
}] }
```
`weight` = count for the selected `ordering` metric.

## GET /v2/mentions/top/countries · /languages
`ordering` default `mentions`, `sortDesc`, `limit` (1–10000, default 10).
```json
{ "countries": [{"country_code": "US", "country_name": "United States",
  "total": {"mentions": 150, "engagement": 5000, "engagement_nv": 500, "reach": 25000}}] }
```
`/languages` is identical with `languages[]` and `language_code`/`language_name`.

## GET /v2/mentions/top/summary/daily
AI daily summaries (past 24h, generated each morning). Params: `fields`, `startDate`/`endDate`, `trackers` (`brandmonitoring` default | `competitors` | `all` — **no `leads`**), `limit` (1–100, default 2).
```json
{ "summary": [{
  "type": "brandmonitoring", "created_at": "2026-02-11 14:00", "description": "Daily brand monitoring summary",
  "clusters": [{
    "description": ["Great product review!", "Amazing service"], "mention_ids": [123456, 123457],
    "total": {"mentions": 45, "reach": 125000, "engagement": 5600, "positive_mentions": 30, "negative_mentions": 5, "neutral_mentions": 10, "web_mentions": 15, "twitter_mentions": 12, "retweet_mentions": 8, "facebook_mentions": 5, "youtube_mentions": 2, "instagram_mentions": 3, "others_mentions": 0},
    "tracker_totals": [{"id": 101, "name": "Brand Monitoring", "type": "trackerGroup", "total": 45, "trackers": [{"id": 201, "name": "Brand Mentions", "type": "tracker", "total": 25}]}]
  }]
}] }
```

## GET /v2/mentions/stats
Analytics overview. Shared filters only — **no pagination/ordering/limit.** Returns `{ stats }`:
```json
{ "stats": {
  "mentions": {"web": {"total": 150, "sites": 45, "unique": 120}, "social": {"total": 500, "users": 200, "unique": 450}, "all": {"period": 650, "total": 650, "all": 1500, "unique": 570}},
  "reach": {"total": 250000},
  "unique_reach": {"gg": 50000, "tw": 75000, "fb": 60000, "yt": 30000, "ig": 25000, "rv": 5000, "rd": 10000, "tk": 15000, "ln": 8000, "pn": 3000, "tl": 2000, "th": 5000, "bs": 1000, "mn": 500, "ot": 2500, "total": 292000},
  "engagement": {"likes": 5000, "comments": 1200, "shares": 1500, "views": 75000, "retweets": 800, "tweet_views": 50000, "reviews": 86.0, "reviews_count": 20, "reviews_average": 4.3, "impressions": 100000, "reach": 250000, "tiktok_views": 20000, "tiktok_shares": 300, "taps_forward": 500, "taps_back": 150},
  "sentiment": {"positive": {"web": 80, "twitter": 150, "facebook": 120, "youtube": 50, "instagram": 90, "reviews": 45, "others": 30, "total": 565}, "negative": {"total": 113}, "neutral": {"total": 252}},
  "emv": {"currency": "usd", "total": 125430.5, "web": 45230.25, "twitter": 32150.75, "facebook": 28540.0, "youtube": 10320.5, "instagram": 7189.0, "reviews": 1500.0, "others": 500.0},
  "mqi": {"type": "avg", "total": 75.5, "web": 82.3, "twitter": 68.75, "facebook": 71.2, "youtube": 79.4, "instagram": 73.85, "reviews": 88.6, "others": 65.9}
}}
```
`unique_engagement` and `impression_engagement` mirror `engagement`'s fields. `sentiment.negative`/`neutral` mirror `positive`'s per-channel shape. `mqi.type` is `avg` or `sum`.

## GET /v2/mentions/audience-demographics
Owned FB/IG audience. Params: `fields`, `endDate` (default today — **no `startDate`**), `trackers`, `trackerGroups`, `filterGenderByAge` (`13-17`…`65+`), `filterAgeByGender` (`male`/`female`/`unspecified`).
```json
{ "demographics": {"gender": {"male": 15000, "female": 18000, "unspecified": 2000},
  "ages": {"13-17": 2000, "18-24": 8000, "25-34": 12000, "35-44": 7000, "45-54": 4000, "55-64": 2500, "65+": 1500}} }
```

## GET /v2/tracker-groups/list
Params: `fields`, `trackerGroupsCategory` (`all` default/`brandmonitoring`/`competitors`/`leads`), `page` (default 1), `limit` (default 10).
```json
{ "trackerGroups": [{"id": 42, "name": "My Brand", "type": "trackerGroup", "category_id": 0, "category_name": "Brand Monitoring", "created_at": "2023-06-15", "total": {"keywords": 5, "profiles": 2}}] }
```
`category_id`: 0=Brand Monitoring, 1=Competitors, 2=Marketing Leads.

## GET /v2/trackers/list
Params: `fields`, `category` (`all` default/`keyword`/`profile`/`location`), `trackerGroups`, `trackerGroupsCategory` (`all` default/`brandmonitoring`/`competitors`/`leads`), `page` (default 1), `limit` (default 10).
```json
{ "trackers": [
  {"id": 101, "name": "Brand Keywords", "type": "tracker", "category_id": 0, "category_name": "Keyword", "is_active": true,
   "tracker_group": {"id": 42, "name": "My Brand", "category_name": "Brand Monitoring"},
   "keyword": {"main_keyword": "mentionlytics", "included_keywords": "monitoring", "excluded_keywords": "spam", "sources": {"web": true, "twitter": true}}},
  {"id": 102, "name": "Brand IG", "type": "tracker", "category_id": 1, "category_name": "Profile",
   "profile": {"id": "1234567890", "type": "instagram", "name": "mybrand", "url": "https://instagram.com/mybrand", "is_authorized": true}}
]}
```
`category_id`: 0=Keyword, 1=Profile, 2=Location. `keyword` present only on Keyword trackers; `profile` only on Profile/Location trackers — check `category_name` before parsing.

---

## Error responses (all endpoints)

Validate on `status` + `error`:

| HTTP | `error` values |
| :--- | :--- |
| `400` | `{ errors: [{field, message}] }` (validation) |
| `401` | `unauthorized`, `invalid_token`, `token_expired`, `invalid_token_options` |
| `403` | `access_denied`, `account_blocked`, `apiV2_not_available`, `account_inactive` |
| `429` | `rate_limit_exceeded` (sustained) or `burst_limit_exceeded` (burst) |
| `520` | `internal_error` |

Non-400 shape: `{ "status": 401, "error": "token_expired", "message": "..." }`.

## Data discrepancy notice
X (Twitter), LinkedIn, and Web News may be subject to strict research caps under certain plans — API data may not exactly mirror the web app for these sources.
