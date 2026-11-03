# Answers Reference: AI Visibility (beta)

The answers endpoints read what AI engines answer to the account's AI Visibility prompts: the answers themselves, how they present your brands and your competitors, and the pages they cite.

- All three are `GET`, need `Authorization: Bearer <token>`, and are **beta**.
- They need AI Visibility on the account. Without it they answer `403` with `"error": "ai_visibility_not_available"`.
- Answers are not mentions: the mention endpoints never return them, and these endpoints read only Prompt trackers.
- The prompts themselves (text, type, country, engines, runs per day) are listed by `GET /v2/trackers/list?category=prompt` (see `endpoints.md`). They can't be created, changed or deleted through the API.
- "Your brands" are the brands set up as your own in AI Visibility; the other brands set up there are your competitors.

## Contents
1. [Engines](#engines)
2. [Shared answer filters](#answer-filters)
3. [GET /v2/answers](#answers)
4. [GET /v2/answers/brand-overview](#brand-overview)
5. [GET /v2/answers/citation-analysis/{id}](#citation-analysis)
6. [Error responses](#errors)

---

<a name="engines"></a>
## Engines

| `engines` value | Engine |
| :--- | :--- |
| `chatgpt` | ChatGPT |
| `gemini` | Gemini |
| `google_ai_mode` | Google AI Mode |
| `google_ai_overview` | Google AI Overview |
| `copilot` | Copilot |
| `perplexity` | Perplexity |
| `claude` | Claude |
| `grok` | Grok |

<a name="answer-filters"></a>
## Shared answer filters

Accepted by `/v2/answers` and `/v2/answers/brand-overview`. These differ from the mention filters: note the `relevance` meaning and the `trackers` default.

| Parameter | Type | Values / Format | Default |
| :--- | :--- | :--- | :--- |
| `engines` | string | Comma-separated engine values (`chatgpt,perplexity`) | all |
| `relevance` | string | `all`; `normal` or `strict` = answers that name at least one of your brands (`relevance` 1); `low` = answers that name none (`relevance` 0.25) | `all` |
| `sentiment` | string | One of `positive`, `neutral`, `negative` | — |
| `emotion` | string | One of `joy`, `anger`, `sadness`, `fear`, `sarcasm`, `neutral` | — |
| `startDate` | string | `YYYYMMDD` | 7 days ago |
| `endDate` | string | `YYYYMMDD` | today (UTC), or `startDate` when that is tomorrow |
| `trackers` | string | `all` (every Prompt tracker); `brandmonitoring`, `competitors` or `leads` (the Prompt trackers in Brand Monitoring, Competitors or Industry & Topics groups); or comma-separated tracker IDs. Trackers that aren't Prompt trackers are ignored | `all` |
| `trackerGroups` | string | Comma-separated tracker group IDs | — |

There are no `channels`, `categories`, `country` or `language` filters on answers; filter by `engines`, trackers and groups.

---

<a name="answers"></a>
## GET /v2/answers

The answers, **newest first**, with the analysis the web app shows in each answer's details. One answer is one engine's response to one prompt on one run, so a prompt that runs daily on four engines adds four answers a day.

**Parameters** (beyond the shared answer filters):
- `fields` *(string)*: comma-separated answer fields. Lists and objects such as `brands`, `citations`, `domains` and `entities` are selected whole: `brands`, not `brands.name`.
- `per_page` *(integer, 1–1000, default 50)*
- `results_after` *(integer)*: cursor. Pass the previous response's `results_after`, and stop when a page holds fewer than `per_page` answers.
- `answer_ids` *(string)*: comma-separated answer IDs. `startDate` and `endDate` are ignored; the other filters still apply.
- `tracker_totals` *(boolean, default false)*: adds the number of matching answers of each Prompt tracker, grouped by tracker group, with 0 for trackers with none.

```bash
curl "https://api.mentionlytics.com/v2/answers?engines=chatgpt,perplexity&startDate=20261001&endDate=20261031&per_page=100&fields=id,pub_date,engine,prompt,relevance,sentiment_text,brands,citations" \
  -H "Authorization: Bearer $MENTIONLYTICS_API_TOKEN"
```

**Expected response (200):** `{ answers: [Answer], totals: [{pub_date, total_count}], results_after, tracker_totals? }`

```json
{
  "answers": [
    {
      "id": 987654321,
      "pub_date": "2026-10-20",
      "pub_datetime": "2026-10-20T09:15:00.000Z",
      "engine": { "id": "chatgpt", "name": "ChatGPT" },
      "model": "gpt-5",
      "prompt": "What are the best social listening tools?",
      "answer": "Here are some of the best social listening tools:\n\n1. **Acme Listen** ...",
      "relevance": 1,
      "sentiment_text": "positive",
      "sentiment_score": 0.62,
      "emotion_text": "joy",
      "executive_summary": "Acme Listen is ranked first and praised for its pricing...",
      "quality": { "bias_detected": false },
      "keyword": { "count": 3, "density_percentage": 0.85 },
      "brands": [
        { "name": "Acme Listen", "rank": 1, "appearances": 3, "sentiment_score": 0.7, "sentiment_text": "positive", "found_in_link": true, "is_active": true },
        { "name": "Globex", "rank": 2, "appearances": 1, "sentiment_score": 0.1, "sentiment_text": "neutral", "found_in_link": false, "is_active": true }
      ],
      "citations": [
        { "position": 1, "title": "Top 10 social listening tools", "url": "https://example.com/top-tools", "domain": "example.com", "description": "We compared ...", "is_citation": true }
      ],
      "domains": ["example.com"],
      "entities": { "companies": ["Acme", "Globex"], "people": [], "products": ["Acme Listen"] },
      "tracker": { "id": 501, "name": "Best social listening tools" },
      "trackerGroup": { "id": 42, "name": "My Brand" },
      "workflow": { "labels": ["Important"] }
    }
  ],
  "totals": [ { "pub_date": "2026-10-20T00:00:00.000Z", "total_count": 12 } ],
  "results_after": 1792476900000
}
```

**Answer fields** (a field with no value is left out):

| Field | Notes |
| :--- | :--- |
| `id` | Answer ID, as accepted by `answer_ids` and `/v2/answers/citation-analysis/{id}` |
| `pub_date`, `pub_datetime` | When the answer was collected, in the account's time zone |
| `engine` | `{id, name}`; `id` is an `engines` value |
| `model` | The model that generated the answer, as the engine reports it |
| `prompt` | The prompt the engine answered |
| `answer` | The answer as the engine gave it, usually Markdown |
| `relevance` | `1` when the answer names at least one of your brands, `0.25` when it names none |
| `sentiment_text`, `sentiment_score` | Overall sentiment; the score runs from -1 to 1, and the label is `negative` at -0.33 or below and `positive` at 0.33 or above |
| `emotion_text` | `joy`, `anger`, `sadness`, `fear`, `sarcasm` or `neutral` |
| `executive_summary` | Short Markdown summary of what the answer says about your brands. An answer that names only other brands reads "No brand mentions were found in the answer" |
| `quality` | `bias_detected` (boolean: one-sided or favoring a view); `relevance_score` (1–10, only answers analyzed between mid-May and early June 2026); `relevance_explanation` (only answers analyzed before late September 2026) |
| `keyword` | `count`: times your brands appear in the answer, links left out (the tracker group's name when no brands are set up); `density_percentage`: those occurrences as a percentage of the answer's words |
| `brands` | The AI Visibility brands the answer names, yours and competitors', ordered by `rank` (order of first appearance): `name`, `rank`, `appearances` (including aliases), `sentiment_score`, `sentiment_text`, `found_in_link`, `is_active` (active when the answer was analyzed) |
| `citations` | The pages the engine read, in its order: `position`, `title`, `url`, `domain` (without www), `description` (the engine's snippet), `is_citation` (`true` when the engine cites the page among the links it shows with the answer, rather than only reading it) |
| `domains` | Domains of the citations |
| `entities` | `{companies, people, products}` named in the answer |
| `tracker`, `trackerGroup` | `{id, name}` of the Prompt tracker and its group |
| `workflow` | `assigned_to` (emails), `labels` (such as `FYI`, `Reply`, `Important`, `Done`, `Good`, `Bad`), `notes`, `tags[{name, aiTag}]` |

`totals` has one entry per day of the range (with `answer_ids`, one per day that has one of them); `pub_date` is the day in the account's time zone, given as midnight UTC. With `tracker_totals=true`, `tracker_totals` is `[{ id, name, type: "trackerGroup", total, trackers: [{ id, name, type: "tracker", total }] }]`.

<a name="brand-overview"></a>
## GET /v2/answers/brand-overview

How the matching answers present the brands set up in AI Visibility, yours and your competitors': how many answers name each brand, its share of all brand mentions, and its average sentiment, position and number of mentions. These are the numbers of the web app's **LLM Brand Visibility** table.

**Parameters:** the shared answer filters, plus `fields` *(string)*: the brand fields to include (all by default; `total_answers` and `counted_answers` are always included). There is no `per_page`, `results_after` or `answer_ids`; sending them answers `400`.

- It counts the **newest 10,000** matching answers. `total_answers` is how many answers match, `counted_answers` how many were counted.
- A brand counts in an answer when the answer's `brands` list names it.
- Only brands that are active in AI Visibility are listed, and only when at least one counted answer names them.
- Answers that name none of the brands, including answers not analyzed yet, count as answers without them.

```bash
curl "https://api.mentionlytics.com/v2/answers/brand-overview?startDate=20261001&endDate=20261031&engines=chatgpt,gemini" \
  -H "Authorization: Bearer $MENTIONLYTICS_API_TOKEN"
```

**Expected response (200):** `{ brands: [BrandOverview], total_answers, counted_answers }`

```json
{
  "brands": [
    { "name": "Acme Listen", "is_own_brand": true, "visibility": 62.5, "share_of_voice": 58.8, "average_sentiment": 0.42, "average_rank": 1.4, "average_appearances": 2.35, "total_appearances": 470, "answers": 200 },
    { "name": "Globex", "is_own_brand": false, "visibility": 46.9, "share_of_voice": 41.3, "average_sentiment": 0.18, "average_rank": 1.9, "average_appearances": 2.2, "total_appearances": 330, "answers": 150 }
  ],
  "total_answers": 320,
  "counted_answers": 320
}
```

| Field | Notes |
| :--- | :--- |
| `name`, `is_own_brand` | Brand name; `true` for your brands, `false` for competitors |
| `visibility` | Percentage of the counted answers that name the brand, to 1 decimal |
| `share_of_voice` | The brand's percentage of all the times the counted answers name any listed brand, to 1 decimal |
| `average_sentiment` | Average sentiment of the answers towards the brand, -1 to 1, to 2 decimals |
| `average_rank` | Average position among the brands an answer names (1 = first), to 2 decimals |
| `average_appearances` | Average times an answer that names the brand names it or an alias, to 2 decimals |
| `total_appearances` | Times the counted answers name the brand or an alias |
| `answers` | Counted answers that name the brand |

Brands are sorted by `answers` (most first), then by `total_appearances`. `brands` is empty when the counted answers name none.

<a name="citation-analysis"></a>
## GET /v2/answers/citation-analysis/{id}

The citations of one answer, in the order `/v2/answers` returns them, each with the AI analysis of the page it links to: its channel and language, whether and where it names your brand and with what sentiment, who published it and what kind of content it is, and how many of your competitors it names.

- `{id}` is the answer's `id`. The endpoint takes **no query parameters**; sending one answers `400`.
- `404` with `"error": "answer_not_found"` when the answer doesn't exist or isn't accessible.
- The pages are analyzed **10 minutes or more** after the answer is collected, so a new answer's citations have no `analysis` yet. Poll again later rather than in a tight loop.
- A page whose text could not be read is not analyzed.
- When an answer cites a page an earlier answer cited, it gets the earlier analysis, unless the page was updated since, when it is analyzed again. Posts on Instagram, X, Facebook, YouTube, Reddit and TikTok are never analyzed again.
- Answers collected before citations were analyzed have none.
- An analysis that matches none of the answer's citations is listed after them, with the `title`, `url` and `is_citation` of the citation it was made for.

```bash
curl "https://api.mentionlytics.com/v2/answers/citation-analysis/987654321" \
  -H "Authorization: Bearer $MENTIONLYTICS_API_TOKEN"
```

**Expected response (200):** `{ answer_id, citations: [Citation + analysis] }`

```json
{
  "answer_id": 987654321,
  "citations": [
    {
      "position": 1,
      "title": "Top 10 social listening tools",
      "url": "https://example.com/top-tools",
      "domain": "example.com",
      "description": "We compared ...",
      "is_citation": true,
      "analysis": {
        "channel": "web",
        "language": "en",
        "brand": { "in_link": "found_and_linked", "position": "top", "sentiment_score": 0.6, "sentiment_text": "positive" },
        "content": { "creator_type": "corporate", "media_type": "web", "genre": "listicle", "ai_content": "human_generated", "ai_content_score": 20 },
        "competitors": { "mentioned": 4, "linked": 2 }
      }
    }
  ]
}
```

**Analysis fields** (a field is left out when the analysis has no value for it):

| Field | Values |
| :--- | :--- |
| `channel` | The page's channel, such as `web`, `youtube`, `reddit` or `linkedin`. X is `twitter` |
| `language` | Language code of the page |
| `brand.in_link` | `not_found`, `found_but_not_linked`, `found_and_linked` (named in the text of a link) |
| `brand.position` | `top` (headline, opening, early overview), `middle` (body, middle of a list), `bottom` (conclusion, footer, end), `not_found` |
| `brand.sentiment_score`, `brand.sentiment_text` | Sentiment of the page towards your brand, -1 to 1, with the same label limits as answers. Left out when `in_link` is `not_found` |
| `content.creator_type` | `owned` (your brand's site or profiles), `competitor`, `ugc` (forums, individuals' posts), `corporate` (another company or a partner), `news`. A web page that is not user-generated can have its site's type instead, such as `blog` or `community` |
| `content.media_type` | `web`, `document` (such as a PDF), `video`, `image`, `social_post` |
| `content.genre` | Long-form: `comparison`, `listicle`, `eshop`, `review`, `how_to`, `case_study`, `research`, `directory`, `documentation`, `press_release`, `pricing_page`, `opinion_editorial`, `home_page`. Conversational and social: `question`, `recommendation_request`, `troubleshooting`, `complaint`, `praise`, `announcement`, `promo`, `poll`, `meme`, `news_share` |
| `content.ai_content` | Band of `ai_content_score`: `human_generated` (0–50), `human_generated_ai_assisted` (51–59), `ai_generated_human_edited` (60–89), `ai_generated` (90–100) |
| `content.ai_content_score` | 0 (written by a human) to 100 (generated by AI). An analysis made before 2026-09-23 holds a 0/1 flag instead, which comes through as 0 or 1 |
| `competitors.mentioned`, `competitors.linked` | How many of your competitors the page names, and names in the text of a link |

---

<a name="errors"></a>
## Error responses

| HTTP | `error` values | Notes |
| :--- | :--- | :--- |
| `400` | (validation) | `{ "errors": [{ "field": "...", "message": "..." }] }`: an unknown parameter or value, such as `per_page` on brand-overview |
| `401` | `unauthorized`, `invalid_token`, `token_expired`, `invalid_token_options` | Refresh the token once, then retry |
| `403` | `ai_visibility_not_available`, `access_denied`, `account_blocked`, `apiV2_not_available`, `account_inactive` | Don't retry |
| `404` | `answer_not_found` | Citation analysis only |
| `429` | `rate_limit_exceeded` | See `auth-rate-limits.md` |
| `520` | `internal_error` | Retry with backoff |
