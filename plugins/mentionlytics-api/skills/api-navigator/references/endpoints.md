# Endpoint Reference: Mentions, Trackers and Account

Base URL: `https://api.mentionlytics.com`
Every endpoint here is a `GET` and requires the `Authorization: Bearer <token>` header.

> **Parameters are NOT uniform across endpoints.** The `ordering` values, pagination style, defaults, and available filters differ per endpoint. Always check the specific endpoint section below; an unknown parameter or value answers `400`. The `ordering` parameter in particular uses **two different value sets**.

Other references: AI Visibility answers in `answers.md`, writes in `write-endpoints.md`, token refresh and limits in `auth-rate-limits.md`.

## Contents
1. [Shared filter parameters](#shared-filters)
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
13. [Error responses](#errors)

---

<a name="shared-filters"></a>
## Shared filter parameters

These filters are accepted by the mention analytics endpoints: `/v2/mentions`, `/v2/mentions/top/mentioners`, `/v2/mentions/top/keywords`, `/v2/mentions/top/countries`, `/v2/mentions/top/languages`, and `/v2/mentions/stats`. (`/v2/mentions/top/summary/daily` and `/v2/mentions/audience-demographics` accept only a subset; see their sections.)

| Parameter | Type | Values / Format | Default |
| :--- | :--- | :--- | :--- |
| `fields` | string | Comma-separated fields to include. Dotted names select nested fields (`profile.name`, `metrics.engagement`) | all |
| `startDate` | string | `YYYYMMDD`, from `20200101`, at most tomorrow | 7 days ago |
| `endDate` | string | `YYYYMMDD`, at most today | today (UTC), or `startDate` when that is tomorrow |
| `relevance` | string | `all`; `low` = only scores below 0.5 (the mentions `normal` leaves out); `normal` ≥ 0.5; `strict` ≥ 0.75 | `normal` |
| `trackers` | string | `brandmonitoring`, `competitors`, `leads` (Industry & Topics groups), `all`, or comma-separated tracker IDs | `brandmonitoring` |
| `trackerGroups` | string | Comma-separated tracker group IDs | — |
| `preset` | integer | ID of a filter preset saved in the web app (≥ 1) | — |
| `channels` | string | Comma-separated channel IDs (see `channels-categories.md`) | all |
| `categories` | string | Comma-separated category tokens (see `channels-categories.md`) | all |
| `contentType` | string | `image`, `album`, `video`, `audio`, `link`, `webpage`, `text`, `other` | — |
| `sentiment` | string | One of `positive`, `neutral`, `negative` | — |
| `emotion` | string | One of `joy`, `anger`, `sadness`, `fear`, `sarcasm`, `neutral` | — |
| `country` | string | Comma-separated ISO country codes (`US,GB`) | — |
| `language` | string | Comma-separated ISO language codes (`en,fr`) | — |

> **`ordering` is NOT shared**: its allowed values and default depend on the endpoint. See each section.

A field with no value is left out of a response; treat a missing count as 0.

---

<a name="mentions"></a>
## GET /v2/mentions

Fetch individual mentions, one object per mention, with their analysis: sentiment, emotion, entities, video analysis and translations.

**Pagination (cursor-based):**
- `per_page` *(integer, 1–10000, default 50)*: results per page.
- `results_after` *(integer)*: cursor. Pass the `results_after` value of the previous response to get the next page, and stop when a page holds fewer than `per_page` mentions. There is **no `page` parameter** on this endpoint.
- The cursor follows publication time: page through complete exports with the default `ordering=time`, and use the other orderings for a single top-N page.

**Ordering** (`ordering`, default `time`):

| Value | Meaning |
| :--- | :--- |
| `time` | Publication date, newest first |
| `rank` | Reach and rank, most influential first |
| `engagement` | Total engagement (likes, shares, comments, views; excludes X/Twitter views) |
| `engagement_nv` | Engagement without views |

**Additional parameters (beyond the shared set):**
- `mention_ids` *(string)*: comma-separated mention IDs. `startDate` and `endDate` are ignored; the other filters still apply.
- `tracker_totals` *(boolean, default false)*: when `true`, the response adds a `tracker_totals` array with counts per tracker group and tracker. It lists the trackers of the groups the user can see, except Prompt trackers (their answers are counted by `/v2/answers`), with 0 for trackers with no matching mentions.
- `extendedFields` *(boolean, default true; false for Power BI and Excel)*: without `fields`, each mention includes `body`, the full text of a web page. Send `extendedFields=false` to leave it out and make large exports smaller. A later version will default to `false` for every client, so a script that reads `body` should send `extendedFields=true` or list `body` in `fields`.

```bash
curl "https://api.mentionlytics.com/v2/mentions?startDate=20261001&endDate=20261031&trackers=brandmonitoring&relevance=normal&per_page=100&extendedFields=false" \
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
      "trackerGroup": { "id": 123, "name": "My Brand" },
      "pub_date": "2026-10-20",
      "pub_datetime": "2026-10-20T14:30:00Z",
      "title": "Great product review",
      "description": "This is the text of the mention...",
      "link": "https://x.com/johndoe/status/123",
      "relevance": 0.85,
      "sentiment_text": "positive",
      "emotion_text": "joy",
      "entities": [ { "type": "PERSON", "value": "John Doe" } ],
      "country_code": "US", "country_name": "United States",
      "language_code": "en", "language_name": "English",
      "channel_name": "twitter", "channel_id": 2,
      "contentType": "text", "category_id": "tweets", "category_name": "Tweets",
      "profile": {
        "id": 789, "name": "John Doe", "username": "johndoe",
        "link": "https://x.com/johndoe",
        "followers_count": 5000, "verified": false, "type_name": "user"
      },
      "metrics": {
        "engagement": 1500, "engagement_no_views": 150,
        "likes": 100, "shares": 25, "comments": 25, "quotes": 10,
        "views": 1350, "impressions": 2000, "reach": 5000
      },
      "workflow": { "labels": ["Important"], "tags": [ { "name": "launch", "aiTag": true } ] }
    }
  ],
  "totals": [ { "pub_date": "2026-10-20T00:00:00.000Z", "total_count": 42 } ],
  "results_after": 1792506600000
}
```

**Mention fields** (each present when it has a value):

| Field | Notes |
| :--- | :--- |
| `id`, `uu_id` | Mention ID (as accepted by `mention_ids`) and external ID |
| `tracker`, `trackerGroup` | `{id, name}` of the tracker and group that collected it |
| `pub_date`, `pub_datetime`, `discover_at`, `updated_at` | Publication date and time, when it was found, when it was last updated |
| `title`, `description`, `link` | Text and URL. `description` is the mention text |
| `body` | Full text of a web page (see `extendedFields`) |
| `relevance` | 0.25–1.0 |
| `sentiment_text`, `emotion_text` | `positive`/`neutral`/`negative`; `joy`/`anger`/`sadness`/`fear`/`sarcasm`/`neutral` |
| `entities` | `[{type, value}]` named entities found in the text |
| `video_analysis` | `{summary, instances: [{time_seconds, label}]}` for analyzed video and audio mentions (YouTube, TikTok, Instagram, podcasts). Select whole, or `video_analysis.summary` / `video_analysis.instances` |
| `translations` | `[{language_code, language_name, text}]`: translations already made in the web app. Reading them uses no translation quota. Selected whole |
| `country_code`, `country_name`, `language_code`, `language_name` | Detected country and language |
| `channel_id`, `channel_name` | `channel_name` is one of `web`, `twitter`, `facebook`, `youtube`, `instagram`, `reddit`, `reviews`, `other` |
| `category_id`, `category_name`, `contentType`, `subChannel`, `subreddit`, `type_name` | Category token (as in `categories`), content type, sub-channel, subreddit, source type |
| `thumb_url` | Thumbnail. (`image` is deprecated; use `thumb_url`) |
| `profile` | Author: `id`, `uu_id`, `name`, `username`, `description`, `image`, `link`, `country_code`, `country_name`, `followers_count`, `post_count`, `type_name`, `verified`, `geolocation{latitude, longitude, locality, country_name, city_name, region_name}` |
| `metrics` | `engagement`, `engagement_no_views`, `likes`, `shares`, `comments`, `quotes`, `views`, `impressions`, `reach`, `visits`, `users`, `review` (rating), `global_rank`, `country_rank`, `reactions{love, wow, haha, sad, angry, thankful, fire, clap, dislike}` (Facebook, Telegram) |
| `total` | `{pub_date_mentions}`: the total mention count for the mention's publication date |
| `parent` | For comments and replies: `{id, uu_id, description, parent{id, uu_id, description, profile{id, name, image}}}` (up to 2 levels) |
| `workflow` | `assigned_to` (emails), `labels`, `notes`, `tags[{name, aiTag}]` |

When `tracker_totals=true`, a top-level `tracker_totals` array is added: `[{ id, name, type: "trackerGroup", total, trackers: [{ id, name, type: "tracker", total }] }]`.

`totals` has one entry per day of the range (with `mention_ids`, one per day that has one of them); `pub_date` is the day in the account's time zone, given as midnight UTC.

<a name="top-mentioners"></a>
## GET /v2/mentions/top/mentioners

Top profiles/influencers mentioning the tracked terms.

**Pagination (page-based):**
- `per_page` *(integer, 1–100, default 10)*
- `page` *(integer, 1–100, default 1)*

**Ordering** (`ordering`, default `reach`): one of `mentions`, `reach`, `engagement`, `engagement_nv`. (`reach` = most influential, `mentions` = most active.)
- `sortDesc` *(boolean, default true)*: descending order.

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
      "image": "https://example.com/avatar.jpg",
      "link": "https://x.com/johndoe",
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

For web sources, `metrics` carries `users` and `visits` instead of social follower counts.

<a name="top-keywords"></a>
## GET /v2/mentions/top/keywords

Trending keywords and hashtags.

- **Ordering** (`ordering`, default `mentions`): one of `mentions`, `reach`, `engagement`, `engagement_nv`.
- `sortDesc` *(boolean, default true)*
- `limit` *(integer, 1–10000, default 10)*: number of keywords to return.
- `onlyHashtags` *(boolean, default false)*: only hashtags (keywords starting with `#`).
- `adjacentWords` *(integer, one of 0, 1, 2, 3, default 1)*: phrase length; `0` = single words, `1` = 2-word phrases, and so on.
- `ignoreMainKeywords` *(boolean, default true)*: leave out the tracked main keywords.

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

`weight` is the value of the selected `ordering` metric (mentions / engagement / engagement_nv / reach).

<a name="top-countries"></a>
## GET /v2/mentions/top/countries

Geographic mention breakdown by country.

- **Ordering** (`ordering`, default `mentions`): one of `mentions`, `reach`, `engagement`, `engagement_nv`.
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

- **Ordering** (`ordering`, default `mentions`): one of `mentions`, `reach`, `engagement`, `engagement_nv`.
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

AI-generated daily summaries of mention clusters, grouped by theme. They are calculated every morning and cover the past **24 hours**.

**Accepts only this parameter subset** (not the full shared set):
- `fields` *(string)*
- `startDate` / `endDate` *(YYYYMMDD)*
- `trackers` *(default `brandmonitoring`)*: one of `brandmonitoring`, `competitors`, `all`. **`leads` and tracker IDs are NOT valid here.**
- `limit` *(integer, 1–100, default 2)*: number of daily summaries to return.

**Expected response (200):** `{ summary: [DailySummary] }`

```json
{
  "summary": [
    {
      "type": "brandmonitoring",
      "created_at": "2026-10-20 06:00",
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
            { "id": 101, "name": "My Brand", "type": "trackerGroup", "total": 45,
              "trackers": [ { "id": 201, "name": "Brand Name Mentions", "type": "tracker", "total": 25 } ] }
          ]
        }
      ]
    }
  ]
}
```

`type` is `brandmonitoring` or `competitors`; `created_at` is UTC. A cluster's `mention_ids` can be fetched with `/v2/mentions?mention_ids=…`. `tracker_totals` leaves out hidden and deleted tracker groups.

<a name="stats"></a>
## GET /v2/mentions/stats

High-level analytics overview: mention counts (web/social/all), reach, `unique_reach` per channel, engagement / `unique_engagement` / `impression_engagement`, sentiment by channel, EMV (Earned Media Value) and MQI (Mention Quality Index).

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

- `unique_reach` keys: `gg` web, `tw` X/Twitter, `fb` Facebook, `yt` YouTube, `ig` Instagram, `rv` reviews, `rd` Reddit, `tk` TikTok, `ln` LinkedIn, `pn` Pinterest, `tl` Telegram, `th` Threads, `bs` Bluesky, `mn` manual, `ot` other.
- `emv` carries a `currency` code plus per-channel values (`web`, `twitter`, `facebook`, `youtube`, `instagram`, `reviews`, `others`, `total`).
- `mqi.type` is `avg` or `sum`; the per-channel breakdown mirrors `emv`.
- `unique_engagement` and `impression_engagement` share the exact field set of `engagement`.

<a name="audience-demographics"></a>
## GET /v2/mentions/audience-demographics

Age/gender demographics of the audiences of **owned** Facebook and Instagram profiles.

**Accepts only this parameter subset:**
- `fields` *(string)*
- `endDate` *(YYYYMMDD, default today)*: **there is no `startDate` on this endpoint.**
- `trackers`, `trackerGroups`
- `filterGenderByAge` *(string)*: one of `13-17`, `18-24`, `25-34`, `35-44`, `45-54`, `55-64`, `65+`.
- `filterAgeByGender` *(string)*: one of `male`, `female`, `unspecified`.

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

List tracker groups (collections of trackers, by category).

- `fields` *(string)*
- `trackerGroupsCategory` *(string, default `all`)*: comma-separated `all`, `brandmonitoring`, `competitors`, `leads` (Industry & Topics).
- `search` *(string, max 255)*: case-insensitive search in the group name.
- `requireTrackers` *(boolean, default true)*: groups without trackers are left out unless this is `false` (for example a group just created through the API).
- `page` *(integer, ≥ 1, default 1)*
- `limit` *(integer, 1–100, default 10)*

```bash
curl "https://api.mentionlytics.com/v2/tracker-groups/list?trackerGroupsCategory=brandmonitoring,competitors&limit=100" \
  -H "Authorization: Bearer $MENTIONLYTICS_API_TOKEN"
```

**Expected response (200):** `{ trackerGroups: [TrackerGroup] }`

```json
{
  "trackerGroups": [
    {
      "id": 42, "name": "My Brand",
      "excluded_keywords": "spam, irrelevant",
      "created_at": "2023-06-15",
      "type": "trackerGroup",
      "category_id": 0, "category_name": "Brand Monitoring",
      "options": { "ignore_duplicates": true, "color": "#3b82f6" },
      "total": { "keywords": 5, "profiles": 2, "prompts": 0 }
    }
  ]
}
```

- `category_id`/`category_name`: `0` = Brand Monitoring, `1` = Competitors, `2` = Industry & Topics.
- `options.color` is left out when the group has none. `total.prompts` counts AI Visibility prompts.

<a name="trackers"></a>
## GET /v2/trackers/list

List individual trackers. Trackers in groups hidden from the token's user in the web app are not listed.

- `fields` *(string)*
- `category` *(string, default `all`)*: tracker type, comma-separated: `all`, `keyword`, `profile`, `location`, `prompt`.
- `trackerGroups` *(string)*: comma-separated tracker group IDs.
- `trackerGroupsCategory` *(string, default `all`)*: parent group category, comma-separated: `all`, `brandmonitoring`, `competitors`, `leads`.
- `search` *(string, max 255)*: case-insensitive search in the tracker name, its main keyword and its profile name.
- `page` *(integer, ≥ 1, default 1)*
- `limit` *(integer, 1–100, default 10)*

**Expected response (200):** `{ trackers: [Tracker] }`

```json
{
  "trackers": [
    {
      "id": 101, "name": "Brand Keywords",
      "type": "tracker",
      "category_id": 0, "category_name": "Keyword",
      "is_active": 1,
      "created_at": "2023-06-15T10:00:00Z", "updated_at": "2026-01-20T08:30:00Z",
      "tracker_group": { "id": 42, "name": "My Brand", "type": "trackerGroup", "category_id": 0, "category_name": "Brand Monitoring" },
      "keyword": {
        "main_keyword": "mentionlytics",
        "included_keywords": "social media, monitoring",
        "excluded_keywords": "jobs, careers",
        "options": { "include_unknown_countries": true, "sentiment_check": true, "ignore_retweets": true },
        "sources": { "web": true, "twitter": true, "facebook": false, "instagram": false },
        "restrictions": { "language_code": "en" }
      }
    },
    {
      "id": 102, "name": "Brand Instagram",
      "type": "tracker",
      "category_id": 1, "category_name": "Profile",
      "is_active": 1,
      "tracker_group": { "id": 42, "name": "My Brand", "type": "trackerGroup", "category_id": 0, "category_name": "Brand Monitoring" },
      "profile": {
        "id": "1234567890", "type": "instagram", "name": "mybrand",
        "url": "https://www.instagram.com/mybrand",
        "is_authorized": 1, "allow_publishing": true,
        "options": { "sentiment_emotion_analysis": true, "ignore_comments": false }
      }
    },
    {
      "id": 103, "name": "Best CRM tools",
      "type": "tracker",
      "category_id": 3, "category_name": "AI Visibility Prompt",
      "is_active": 1,
      "tracker_group": { "id": 42, "name": "My Brand", "type": "trackerGroup", "category_id": 0, "category_name": "Brand Monitoring" },
      "prompt": {
        "text": "What are the best CRM tools for small businesses?",
        "type": "comparative", "country_code": "US", "runs_per_day": 1,
        "sources": { "chatgpt": true, "gemini": true, "perplexity": true }
      }
    }
  ]
}
```

- `category_id`/`category_name`: `0` = Keyword, `1` = Profile, `2` = Location, `3` = AI Visibility Prompt. Inspect it before parsing.
- `is_active` and `profile.is_authorized` are `1`/`0`.
- `keyword` is present on **Keyword and Location** trackers, `profile` on **Profile and Location** trackers, `prompt` on **AI Visibility Prompt** trackers.
- `keyword` holds `main_keyword` (alternatives separated by `|`; with a `boolean_query`, up to 5 of its terms, used on the channels that don't support boolean queries), `custom_hashtag_search`, `boolean_query`, `included_keywords`, `excluded_keywords`, and:
  - `options`: `included_keywords_or`, `include_unknown_countries`, `sentiment_check`, `video_analysis_check`, `relevance_check_based_on_video_analysis` (while YouTube or TikTok is on), `ignore_comments` (while a source with comments is on), `ignore_retweets`, `exact_match`, `ignore_included_tweets` (X/Twitter only), `targeted_sentiment_emotion` (`Main Keyword`, `Tracker Group`, `Not Targeted`; while `sentiment_check` is on), `daily_mention_limit` and `daily_mention_limit_left` (0 or below: the keyword stops collecting until the next day);
  - `sources`: `web`, `twitter`, `facebook`, `meta_ads`, `youtube`, `instagram`, `reddit`, `tiktok`, `bluesky`, `linkedin`, `pinterest`, `telegram`, `threads`, `podcasts` (booleans);
  - `restrictions`: `site_search` (up to 10 sites), `language_code`, `country_code`, `subreddit` (`r/name`, comma-separated).
- `profile` holds `id` (the platform's ID), `type` (`twitter`, `facebook`, `instagram`, `youtube`, `linkedin`, `tiktok`, `pinterest`, `telegram`, `threads`, `bluesky`, `reddit`, `googleMyBusiness`, `reviewTrackers`, `rss`, `appstore`, `googleplay`), `name`, `url`, `is_authorized`, `allow_publishing`, and `options`: `sentiment_emotion_analysis`, `ignore_comments` (Facebook, Instagram, YouTube, LinkedIn, TikTok, Reddit), `ignore_replies` (X/Twitter, Bluesky), `ignore_tagged_posts_comments` and `track_meta_ads` (Facebook).
- `prompt` holds `text`, `type` (`informational`, `instructional`, `comparative`, `transactional`, `other`), `country_code`, `runs_per_day` (1, 2 or 4) and `sources` (`chatgpt`, `gemini`, `google_ai_overview`, `google_ai_mode`, `copilot`, `perplexity`, `claude`).
- A missing field and an empty string both mean no value.

<a name="account-usage"></a>
## GET /v2/account/usage

The account's plan limits and current usage. No parameters.

```bash
curl "https://api.mentionlytics.com/v2/account/usage" \
  -H "Authorization: Bearer $MENTIONLYTICS_API_TOKEN"
```

**Expected response (200):** `{ usage: AccountUsage }`

```json
{
  "usage": {
    "account": { "id": 12345, "name": "Acme Inc", "subscription_plan": "pro", "state": "active" },
    "mentions": { "total": 50000, "used": 12345, "remaining": 37655, "reset_at": "2026-11-15T00:00:00Z" },
    "translations": { "total": 1000, "used": 120, "remaining": 880, "reset_at": "2026-11-15T00:00:00Z" },
    "keywords": { "total": 20, "used": 12, "remaining": 8 },
    "socialProfiles": { "total": 10, "used": 4, "remaining": 6 },
    "locations": { "total": 5, "used": 1, "remaining": 4 },
    "users": { "total": 5, "used": 3, "remaining": 2 }
  }
}
```

- `mentions` and `translations` reset monthly, at `reset_at` (UTC). The other allowances reflect what is configured on the account now.
- `keywords.remaining` is how many more keyword trackers can be created.
- `brandAccounts` (`total`, `used`, `thisMonthAdded`) and `trialAccounts` (`total`, `used`) are returned only for accounts with Brand Hub.

---

<a name="errors"></a>
## Error responses (all endpoints)

Validate on the `status` + `error` fields.

| HTTP | `error` values | Shape |
| :--- | :--- | :--- |
| `400` | (validation) | `{ "errors": [{ "field": "...", "message": "..." }] }` |
| `401` | `unauthorized`, `invalid_token`, `token_expired`, `invalid_token_options` | `{ "status": 401, "error": "...", "message": "..." }` |
| `403` | `access_denied`, `account_blocked`, `apiV2_not_available`, `account_inactive` | `{ "status": 403, "error": "...", "message": "..." }` |
| `429` | `rate_limit_exceeded` (burst or sustained; the `message` says which) | `{ "status": 429, "error": "...", "message": "..." }` |
| `520` | `internal_error` | `{ "status": 520, "error": "internal_error", "message": "..." }` |

```json
{ "status": 401, "error": "token_expired", "message": "Authentication required" }
```

---

## Data discrepancy notice

Content from X (Twitter), LinkedIn, and Web News may be subject to strict research caps under certain plans. API data might not precisely mirror the main web application for these sources.
