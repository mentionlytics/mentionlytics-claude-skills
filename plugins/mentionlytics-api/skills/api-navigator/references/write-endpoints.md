# Write Reference: Trackers and Tracker Groups (beta)

Six endpoints create, change, pause, resume and delete the account's tracker groups and trackers.

| Method | Endpoint | What it does |
| :--- | :--- | :--- |
| `POST` | `/v2/tracker-groups/create` | Create a tracker group |
| `PUT` | `/v2/tracker-groups/update/{id}` | Rename a group, change its category or options, or pause or resume all its trackers |
| `DELETE` | `/v2/tracker-groups/delete/{id}` | Delete a group and all of its trackers |
| `POST` | `/v2/trackers/create` | Create a Keyword tracker |
| `PUT` | `/v2/trackers/update/{id}` | Change, rename, move, pause or resume a tracker |
| `DELETE` | `/v2/trackers/delete/{id}` | Delete a tracker |

## Contents
1. [Before you write](#before)
2. [Request rules](#rules)
3. [POST /v2/tracker-groups/create](#group-create)
4. [PUT /v2/tracker-groups/update/{id}](#group-update)
5. [DELETE /v2/tracker-groups/delete/{id}](#group-delete)
6. [POST /v2/trackers/create](#tracker-create)
7. [PUT /v2/trackers/update/{id}](#tracker-update)
8. [DELETE /v2/trackers/delete/{id}](#tracker-delete)
9. [Keyword settings](#keyword)
10. [Profile settings](#profile)
11. [Errors and retries](#errors)

---

<a name="before"></a>
## Before you write

- **Beta, available upon review.** Write access is enabled for an account on request, after review, for specific use cases. Until then every write answers `403 apiV2Write_not_available`.
- **Admin role.** The token's user must be an admin on the account, otherwise `403 write_not_allowed`.
- **Token scope.** A token limited to specific trackers cannot create trackers, or create, change or delete tracker groups: `403 token_scope_restricted`.
- **Plan.** A setting that the account's plan doesn't include answers `403` with a code naming it, such as `source_not_available` or `boolean_query_not_available` (see [Errors and retries](#errors)).
- **Keywords.** Each Keyword tracker uses one keyword of the plan. `keywords.remaining` in `GET /v2/account/usage` says how many more can be created; with none left, a create answers `403 keyword_limit_reached`.
- **Write limits.** Writes have their own, lower limits than reads: 2 per 5 seconds, 10 per 60 seconds and 100 per day for a plan that allows up to 25 trackers, proportionally more for a larger plan (see `auth-rate-limits.md`). Pace bulk jobs, such as creating many trackers, rather than sending the requests in a loop.
- **What the API can't create.** Profile trackers, Location trackers and AI Visibility prompts are created in the web app. Profile and Location trackers can then be updated and deleted here.
- **AI Visibility prompts are read-only.** A prompt, or a group that contains one, can be read through the API but changed, paused or deleted in the web app only: `422 ai_visibility_prompt_read_only`.

<a name="rules"></a>
## Request rules

- Send JSON with `Content-Type: application/json`.
- A body takes the writable fields in this file only. Any other key is a `400`, including the read-only fields a read returns, such as `id`, `type`, `created_at` or a profile's `name`, and a mistyped option name. The exception is `keyword.options.daily_mention_limit_left`, which is read-only but accepted and ignored.
- **Updates are partial.** Only the fields sent change; in `keyword.sources`, only the channels sent change. `null` clears a nullable field.
- **An update must change something.** A body is a `400`, as an empty body is, when all it holds is:
  - an empty object, such as `"keyword": {}` or `"options": {}`;
  - `null` on a field that can't be cleared, such as `is_active` or `options.ignore_duplicates`;
  - a read-only field;
  - `existing_mentions`, which applies only to a keyword change sent with it.
- `is_active` takes `true`/`false`, or `1`/`0` as the list endpoints return it.
- Create and update take a `fields` query parameter: comma-separated fields to include in the response, as on the list endpoints.
- Responses are the group or tracker in the same shape as `GET /v2/tracker-groups/list` and `GET /v2/trackers/list` (see `endpoints.md`).
- A `201` whose body holds only `{ "id": ... }` means the group or tracker was created but could not be read back. It exists, so don't create it again; read it with the list endpoints.
- Changes to what a tracker collects apply to new mentions. Mentions already collected are kept unless a tracker update sends `"existing_mentions": "delete"`.

---

<a name="group-create"></a>
## POST /v2/tracker-groups/create

| Field | Type | Rules |
| :--- | :--- | :--- |
| `name` | string, **required** | 1–255 characters. Can't be `null` |
| `category_id` | integer, **required** | `0` Brand Monitoring, `1` Competitors, `2` Industry & Topics. `403 tracker_group_category_not_available` when the plan doesn't include Competitors or Industry & Topics |
| `excluded_keywords` | string or null | Comma-separated keywords that disqualify a mention for every tracker in the group, up to 4000 characters. `403 excluded_keywords_not_available` when the plan doesn't include group excluded keywords |
| `options.ignore_duplicates` | boolean | Whether duplicate mentions are ignored for this group. Default `true` |
| `options.color` | string or null | Display colour as a hex value: `#RGB`, `#RGBA`, `#RRGGBB` or `#RRGGBBAA`. `null` or white clears it; a colour name such as `red` is a `400` |

```bash
curl -X POST "https://api.mentionlytics.com/v2/tracker-groups/create" \
  -H "Authorization: Bearer $MENTIONLYTICS_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Competitor Watch", "category_id": 1, "excluded_keywords": "spam, irrelevant", "options": {"ignore_duplicates": true, "color": "#3b82f6"}}'
```

**Expected response (201):** the tracker group.

```json
{
  "id": 57, "name": "Competitor Watch",
  "excluded_keywords": "spam, irrelevant",
  "created_at": "2026-11-03",
  "type": "trackerGroup",
  "category_id": 1, "category_name": "Competitors",
  "options": { "ignore_duplicates": true, "color": "#3b82f6" },
  "total": { "keywords": 0, "profiles": 0, "prompts": 0 }
}
```

`GET /v2/tracker-groups/list` lists a group without trackers only with `requireTrackers=false`.

**Errors:** `400`, `401`, `403`, `429`, `520`.

<a name="group-update"></a>
## PUT /v2/tracker-groups/update/{id}

Takes any of the create fields, all optional, plus `is_active`.

- `null` clears `excluded_keywords` or `options.color`; `name` can't be cleared.
- On any plan, `excluded_keywords` can be set to `null` or to the list already set.
- `is_active` pauses (`false`) or resumes (`true`) every tracker in the group at once:
  - Trackers already in that state are left as they are.
  - The response doesn't show the trackers' state; `GET /v2/trackers/list?trackerGroups={id}` does.
  - Resuming takes a few seconds for each Location tracker in the group. A Location tracker that was switched off with the account's location subscription stays paused.
  - A group that contains an AI Visibility prompt can be paused or resumed in the web app only: `422 ai_visibility_prompt_read_only`.

```bash
curl -X PUT "https://api.mentionlytics.com/v2/tracker-groups/update/42" \
  -H "Authorization: Bearer $MENTIONLYTICS_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"is_active": false}'
```

Other bodies:

```json
{ "name": "My Brand (EMEA)" }
```
```json
{ "options": { "color": null } }
```

**Expected response (200):** the tracker group, as on create.

**Errors:** `400`, `401`, `403`, `404 tracker_group_not_found`, `409 location_changed`, `422 ai_visibility_prompt_read_only`, `429`, `520` (`internal_error`, `location_release_failed`, `location_register_failed`, `update_incomplete`).

<a name="group-delete"></a>
## DELETE /v2/tracker-groups/delete/{id}

Deletes the group **and all of its trackers**. They stop collecting, and they and their mentions are no longer returned by the API. **This cannot be undone.**

```bash
curl -X DELETE "https://api.mentionlytics.com/v2/tracker-groups/delete/42" \
  -H "Authorization: Bearer $MENTIONLYTICS_API_TOKEN"
```

**Expected response (200):**

```json
{ "id": 42, "type": "trackerGroup", "deleted": true, "deleted_trackers": [101, 102] }
```

- A group that contains an AI Visibility prompt can be deleted in the web app only: `422 ai_visibility_prompt_read_only`.
- If the delete fails part-way, the group is left in place, and retrying the request finishes it.

**Errors:** `400`, `401`, `403`, `404 tracker_group_not_found`, `409 location_changed`, `422 ai_visibility_prompt_read_only`, `429`, `520` (`internal_error`, `location_release_failed`, `delete_incomplete`).

---

<a name="tracker-create"></a>
## POST /v2/trackers/create

Creates a **Keyword** tracker.

| Field | Type | Rules |
| :--- | :--- | :--- |
| `tracker_group_id` | integer ≥ 1 | The existing group to add the tracker to. `404 tracker_group_not_found` |
| `tracker_group` | object | A new group to create together with the tracker, with the group create fields (`name` and `category_id` required). `422 keyword_not_allowed` for a name that can't be used |
| `name` | string or null | Tracker name, up to 255 characters. Defaults to a name derived from the keyword. `422 keyword_not_allowed` for a name that can't be used |
| `is_active` | boolean | Default `true`: collection starts as soon as the tracker is created |
| `keyword` | object, **required** | See [Keyword settings](#keyword). `keyword.main_keyword` is required, and at least one `keyword.sources` channel must be `true` |

Send exactly one of `tracker_group_id` and `tracker_group`; both or neither is a `400`.

```bash
curl -X POST "https://api.mentionlytics.com/v2/trackers/create" \
  -H "Authorization: Bearer $MENTIONLYTICS_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "tracker_group_id": 42,
    "name": "ACME in the UK press",
    "keyword": {
      "main_keyword": "acme corp | acme ltd",
      "boolean_query": "(\"acme corp\" OR \"acme ltd\") AND (launch OR release)",
      "sources": { "web": true },
      "restrictions": { "language_code": "en", "country_code": "GB" },
      "options": { "sentiment_check": true }
    }
  }'
```

Other bodies:

```json
{ "tracker_group_id": 42, "keyword": { "main_keyword": "mentionlytics", "sources": { "web": true, "twitter": true } } }
```
```json
{
  "tracker_group": { "name": "Competitor Watch", "category_id": 1 },
  "keyword": { "main_keyword": "acme corp", "excluded_keywords": "jobs, careers", "sources": { "web": true } }
}
```

**Expected response (201):** the tracker, in the shape of `GET /v2/trackers/list` (abridged here).

```json
{
  "id": 205, "name": "ACME in the UK press",
  "type": "tracker",
  "category_id": 0, "category_name": "Keyword",
  "is_active": 1,
  "tracker_group": { "id": 42, "name": "My Brand", "type": "trackerGroup", "category_id": 0, "category_name": "Brand Monitoring" },
  "keyword": {
    "main_keyword": "acme corp | acme ltd",
    "boolean_query": "(\"acme corp\" OR \"acme ltd\") AND (launch OR release)",
    "options": { "include_unknown_countries": true, "sentiment_check": true },
    "sources": { "web": true },
    "restrictions": { "language_code": "en", "country_code": "GB" }
  }
}
```

With `tracker_group`, the new group's ID is the response's `tracker_group.id`.

**Errors:** `400`, `401`, `403` (write access, plan, `keyword_limit_reached`), `404 tracker_group_not_found`, `422` (keyword and source codes), `429`, `503 boolean_query_check_unavailable`, `520`.

<a name="tracker-update"></a>
## PUT /v2/trackers/update/{id}

All fields are optional, and at least one must change something.

| Field | Rules |
| :--- | :--- |
| `tracker_group_id` | Moves the tracker to this group. `404 tracker_group_not_found` |
| `name` | `null` restores the name derived from the keyword. `422 keyword_not_allowed` |
| `is_active` | `false` pauses collection, `true` resumes it. Resuming a Location tracker can take a few seconds. `422 location_deactivated`: the Location tracker was switched off with the account's location subscription and can't be resumed here |
| `keyword` | For **Keyword and Location** trackers: any [keyword setting](#keyword); `null` clears one. `422 keyword_not_applicable` on a Profile tracker |
| `profile` | For **Profile and Location** trackers: `profile.options` (see [Profile settings](#profile)). `422 profile_not_applicable` on a Keyword tracker |
| `existing_mentions` | `keep` (default) or `delete`: what happens to the mentions already collected when this request changes the keyword |

Send `keyword` or `profile`, not both (`400`).

**`existing_mentions`**
- `keep`: the mentions already collected stay, and the new settings apply from now on.
- `delete`: every mention collected so far is deleted, and the tracker starts over. **This cannot be undone.**
  - It needs a change in the same request to `keyword.main_keyword`, `keyword.boolean_query`, `keyword.included_keywords`, `keyword.excluded_keywords` or `keyword.options.included_keywords_or`, otherwise `422 existing_mentions_not_applicable`. A boolean query that differs only in spacing, or a keyword list that differs only in the spaces around its commas, is not a change.
  - It can be used a limited number of times per tracker per day: `429 keyword_edit_limit_reached`. Try again tomorrow, or keep the mentions.
  - `422 keyword_edit_allowance_missing`: the tracker's mentions can't be reset right now; contact support.

```bash
curl -X PUT "https://api.mentionlytics.com/v2/trackers/update/101" \
  -H "Authorization: Bearer $MENTIONLYTICS_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"keyword": {"excluded_keywords": "jobs, careers", "sources": {"reddit": true}}}'
```

Other bodies:

| Goal | Body |
| :--- | :--- |
| Rename | `{"name": "Brand Keywords (EMEA)"}` |
| Pause | `{"is_active": false}` |
| Move to another group | `{"tracker_group_id": 57}` |
| Change what is collected | `{"keyword": {"main_keyword": "mentionlytics", "included_keywords": "monitoring, listening", "sources": {"web": true, "twitter": true, "reddit": false}}}` |
| Change the keyword and discard its mentions | `{"keyword": {"main_keyword": "mentionlytics"}, "existing_mentions": "delete"}` |
| Turn off a restriction | `{"keyword": {"restrictions": {"country_code": null}}}` |
| Profile settings | `{"profile": {"options": {"sentiment_emotion_analysis": true}}}` |

**Expected response (200):** the tracker, as on create.

**Errors:** `400`, `401`, `403`, `404` (`tracker_not_found`, or `tracker_group_not_found` for the group to move it to; the code says which), `409 location_changed`, `422`, `429` (including `keyword_edit_limit_reached`), `503 boolean_query_check_unavailable`, `520` (`internal_error`, `location_release_failed`, `location_register_failed`, `update_incomplete`).

<a name="tracker-delete"></a>
## DELETE /v2/trackers/delete/{id}

The tracker stops collecting, and it and its mentions are no longer returned by the API. Its group is kept, even when no trackers are left in it. **This cannot be undone through the API.**

```bash
curl -X DELETE "https://api.mentionlytics.com/v2/trackers/delete/101" \
  -H "Authorization: Bearer $MENTIONLYTICS_API_TOKEN"
```

**Expected response (200):**

```json
{ "id": 101, "type": "tracker", "deleted": true, "tracker_group_id": 42 }
```

If the delete of a Location tracker fails part-way, the tracker is left in place and paused, and retrying the request finishes the delete.

**Errors:** `400`, `401`, `403`, `404 tracker_not_found`, `409 location_changed`, `422 ai_visibility_prompt_read_only`, `429`, `520` (`internal_error`, `location_release_failed`, `delete_incomplete`).

---

<a name="keyword"></a>
## Keyword settings (`keyword`)

For Keyword and Location trackers. On create, `main_keyword` and one `sources` channel are required; on update, every field is optional and `null` clears it.

| Field | Type | Rules and errors |
| :--- | :--- | :--- |
| `main_keyword` | string | 3–99 characters (`422 keyword_too_short` / `keyword_too_long`). Separate alternatives with `\|` to track them in one tracker, such as `acme \| acme corp` (`403 main_keyword_or_not_available` when the plan doesn't include it). `422 keyword_not_allowed`: this keyword can't be tracked; contact support |
| `boolean_query` | string, up to 4000 | See below |
| `custom_hashtag_search` | string | One hashtag, such as `#acme`, searched for on hashtag-based sources instead of the main keyword. Anything else is a `400` |
| `included_keywords` | string, up to 4000 | Comma-separated keywords that must also appear: all of them, unless `options.included_keywords_or` is `true`. `422 keyword_not_allowed` |
| `excluded_keywords` | string, up to 4000 | Comma-separated keywords that disqualify a mention |
| `options` | object | See below |
| `sources` | object | See below |
| `restrictions` | object | See below |

**Boolean queries**
- Use `AND`, `OR`, `NOT`, quoted phrases and parentheses. Put phrases and terms with punctuation in quotes (`"acme corp"`, `"coca-cola"`), and exclude a term with `AND NOT`: `apple AND NOT pie`.
- With a `boolean_query`, `main_keyword` lists up to 5 of the query's terms, separated by `|`. They are used on the channels that don't support boolean queries.
- `null` clears the query, and the tracker collects on `main_keyword` only.
- Errors:
  - `403 boolean_query_not_available`: the plan doesn't include boolean queries.
  - `422 boolean_query_invalid`: the query isn't valid; the message says why.
  - `422 boolean_query_too_broad`: the query would match too many mentions; narrow it with `AND` or more specific phrases.
  - `422 boolean_main_keywords_required`: the tracker has a boolean query but no main keyword was given.
  - `422 boolean_main_keywords_limit`: more than 5 main keyword terms.
  - `422 boolean_main_keywords_unknown`: a main keyword term isn't part of the query; the message lists the ones that are.
  - `422 main_keyword_required`: `main_keyword` or `boolean_query` was cleared, which would leave the tracker with neither.
  - `422 keyword_not_allowed`: the query contains a keyword that can't be tracked; contact support.
  - `503 boolean_query_check_unavailable`: the query couldn't be checked right now, so nothing was saved; retry the request.

**`keyword.options`**

| Option | Default on create | Notes |
| :--- | :--- | :--- |
| `included_keywords_or` | — | `true`: `included_keywords` match with OR instead of AND. `403 included_keywords_or_not_available` |
| `include_unknown_countries` | `true` | Include mentions whose country couldn't be determined |
| `sentiment_check` | `true` | Run sentiment analysis |
| `video_analysis_check` | `false` | Run video analysis. `403 video_analysis_not_available` |
| `relevance_check_based_on_video_analysis` | `false` | Change relevance and sentiment based on the video and audio analysis. YouTube and TikTok |
| `ignore_comments` | — | Ignore comment-type mentions |
| `ignore_retweets` | `true` | X only: ignore retweets |
| `ignore_included_tweets` | — | X only: don't bring the original tweets of quotes and replies when the keyword isn't in them |
| `exact_match` | — | X only: lower the relevance of a tweet where the keyword isn't an exact whole word, special characters such as `_` included |
| `targeted_sentiment_emotion` | — | `Main Keyword`, `Tracker Group` or `Not Targeted`: what targeted sentiment and emotion analysis is aimed at |
| `daily_mention_limit` | off | The most mentions the keyword collects in a day; once reached, it stops until the next day. A multiple of 500 from 500 to 1,000,000; `null` or `0` turns it off. `422 daily_mention_limit_invalid` |
| `daily_mention_limit_left` | — | Read-only: how much of the limit is left today. Ignored when sent |

Reads return the channel-specific options only while their channels are on, such as `ignore_retweets` while `sources.twitter` is `true`.

**`keyword.sources`** (booleans; at least one `true` on create, otherwise `400`)

| Source | Notes |
| :--- | :--- |
| `web` | Web, news and blogs |
| `twitter` | X. `403 source_not_available` when the plan doesn't include it |
| `facebook` | |
| `meta_ads` | Meta ads (Facebook and Instagram), for Keyword trackers. `403 source_not_available`. A Facebook profile tracker uses `profile.options.track_meta_ads` instead |
| `youtube` | |
| `instagram` | Needs a connected Instagram account: `422 source_not_authorized` |
| `reddit`, `tiktok`, `bluesky`, `linkedin`, `pinterest` | |
| `telegram` | Needs Telegram monitoring in the plan (`403 source_not_available`) and a monitored Telegram channel (`422 source_not_authorized`) |
| `threads` | Needs a connected Threads account: `422 source_not_authorized` |
| `podcasts` | `403 source_not_available` when the plan doesn't include podcast monitoring |

**`keyword.restrictions`**

| Field | Rules |
| :--- | :--- |
| `site_search` | Up to 10 sites, comma-separated: `example.com,anotherexample.com`. `null` or `""` clears it. `400` for more than 10, or a site that isn't valid |
| `language_code` | ISO 639-1 code (two lowercase letters), or one of `zh-CN`, `zh-TW`, `iw`, `fil`, `yue`, `ckb`, `war`, `n/a` (unknown). `null` removes the restriction, so the tracker collects in every language |
| `country_code` | ISO 3166-1 alpha-2 code (two uppercase letters), or `XK` (Kosovo). `null` removes the restriction, so the tracker collects from every country |
| `subreddit` | Up to 5 subreddits, comma-separated, as `r/name` or just the name. `null` or `""` clears it. `400` for a name that isn't valid; `422 subreddit_limit` for more than 5 |

To collect in every language or from every country, leave `language_code` or `country_code` out on create, or send `null` on update.

<a name="profile"></a>
## Profile settings (`profile.options`)

For Profile and Location trackers. The profile itself, your own or a public one such as a competitor's, is set up in the web app.

| Option | Profiles | Notes |
| :--- | :--- | :--- |
| `sentiment_emotion_analysis` | All | Run sentiment and emotion analysis |
| `ignore_comments` | Facebook, Instagram, YouTube, LinkedIn, TikTok, Reddit | Skip comments and replies on the profile's own posts |
| `ignore_replies` | X, Bluesky | Skip replies to the profile's own posts |
| `ignore_tagged_posts_comments` | Facebook | Skip comments on posts the profile is tagged in |
| `track_meta_ads` | Facebook | Collect the profile's Meta ads. `403 meta_ads_not_available`; `422 meta_ads_not_applicable` on another profile type |

An option sent for a profile it doesn't apply to answers `422 profile_option_not_applicable`.

---

<a name="errors"></a>
## Errors and retries

Every error is `{ "status": ..., "error": "...", "message": "..." }`, except a `400`: `{ "errors": [{ "field": "body.category_id", "message": "category_id invalid value: 7. Allowed: 0, 1, 2" }] }`.

| HTTP | `error` | Meaning | What to do |
| :--- | :--- | :--- | :--- |
| `400` | (validation) | A field is invalid or unknown, the body changes nothing, both or neither group fields were sent on create, or `keyword` and `profile` were sent together | Fix the request |
| `401` | `unauthorized`, `invalid_token`, `token_expired`, `invalid_token_options` | The bearer token is missing, invalid or expired | Refresh the token once, then retry |
| `403` | `write_not_allowed` | The user isn't an admin on the account | Don't retry |
| `403` | `apiV2Write_not_available` | Write access isn't enabled for the account | Don't retry; it is enabled on request, after review |
| `403` | `token_scope_restricted` | The token is limited to specific trackers | Use a token that isn't limited to specific trackers |
| `403` | `keyword_limit_reached` | All the plan's keywords are in use | Don't retry; `keywords.remaining` in `GET /v2/account/usage` shows when there is room again |
| `403` | `source_not_available`, `boolean_query_not_available`, `main_keyword_or_not_available`, `included_keywords_or_not_available`, `tracker_group_category_not_available`, `excluded_keywords_not_available`, `meta_ads_not_available`, `video_analysis_not_available` | The plan doesn't include that setting | Leave the setting out |
| `403` | `access_denied`, `account_blocked`, `apiV2_not_available`, `account_inactive` | The account or user can't use the API | Don't retry |
| `404` | `tracker_not_found`, `tracker_group_not_found` | It doesn't exist or isn't accessible | Check the ID with the list endpoints |
| `409` | `location_changed` | A Location tracker was changed by another request at the same time, so the rest of this request wasn't saved | Retry the same request to finish it |
| `422` | `keyword_too_short`, `keyword_too_long`, `keyword_not_allowed`, `main_keyword_required`, `boolean_query_invalid`, `boolean_query_too_broad`, `boolean_main_keywords_required`, `boolean_main_keywords_limit`, `boolean_main_keywords_unknown`, `daily_mention_limit_invalid`, `subreddit_limit`, `source_not_authorized` | A keyword setting can't be applied (see [Keyword settings](#keyword)) | Fix the setting; for `keyword_not_allowed`, contact support |
| `422` | `keyword_not_applicable`, `profile_not_applicable`, `profile_option_not_applicable`, `meta_ads_not_applicable` | The setting doesn't apply to this kind of tracker or profile | Send the settings of the tracker's kind |
| `422` | `existing_mentions_not_applicable`, `keyword_edit_allowance_missing` | `existing_mentions: delete` without a keyword change, or the mentions can't be reset right now | Send a keyword change with it, or keep the mentions; contact support for the second |
| `422` | `location_deactivated` | The Location tracker was switched off with the account's location subscription | Don't retry; it can't be resumed through the API |
| `422` | `ai_visibility_prompt_read_only` | A prompt, or a group that contains one | Use the web app |
| `429` | `rate_limit_exceeded` | Burst or sustained write limit | Wait `Retry-After` seconds, then retry |
| `429` | `daily_write_limit_exceeded` | The day's writes are used up | Stop; it resets after `Retry-After` seconds, at the time in the message |
| `429` | `keyword_edit_limit_reached` | `existing_mentions: delete` was used on this tracker too many times today | Try tomorrow, or resend without it to keep the mentions |
| `503` | `boolean_query_check_unavailable` | The boolean query couldn't be checked, so nothing was saved | Retry the same request |
| `520` | `internal_error` | Something went wrong; on update and delete, nothing was saved | Retry with backoff; after a failed create, check the list before retrying |
| `520` | `location_release_failed`, `location_register_failed` | A Location tracker couldn't be paused, resumed or released, so nothing was changed | Retry the same request |
| `520` | `update_incomplete` | The pause or resume went through, but the rest of the update wasn't saved | Retry the same request to finish it |
| `520` | `delete_incomplete` | The delete didn't finish; the tracker or group is left in place | Retry the same request to finish it |

Retrying a write is safe in the cases above: a retry of a partly applied update or delete finishes it. Don't retry a create that answered `201`, even with only `{ "id": ... }` in the body.
