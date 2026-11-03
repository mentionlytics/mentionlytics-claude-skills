# Mentionlytics API Plugin

Teach Claude how to work with the [Mentionlytics API](https://www.mentionlytics.com) (v2). It covers reading mentions, sentiment and analytics such as EMV and MQI, reading what AI engines answer about your brand, and creating, updating, pausing and deleting trackers and tracker groups. Token refresh, rate limits and caching are handled correctly out of the box.

## What's included

| Skill | Purpose |
| :--- | :--- |
| `api-navigator` | Operational guide for the Mentionlytics API v2 (documentation version 2.1.17): endpoint routing, parameters and response fields, AI Visibility answers, tracker and tracker group writes, channel and category tokens, token refresh, rate limits, caching and errors. |

## Setup

1. Get your API tokens in the Mentionlytics web app under **Settings → Api/Access Tokens → User API Tokens**. Your plan must include API access.
2. Export them as environment variables:

```bash
export MENTIONLYTICS_API_TOKEN="your-bearer-token"
export MENTIONLYTICS_REFRESH_TOKEN="your-refresh-token"
```

The skill never hardcodes or prints tokens.

Two parts of the API are in beta:
- **AI Visibility answers** need AI Visibility on your account.
- **Writes** (creating, changing and deleting trackers and tracker groups) are enabled for an account on request, after review, and the token's user must be an admin on the account.

## Example prompts

- "Pull the top 20 influencers mentioning my brand this month from the Mentionlytics API"
- "Write a Python script that fetches yesterday's mentions and groups them by sentiment"
- "What's the trending hashtag breakdown for my competitors tracker over the last 7 days?"
- "Break down my brand mentions by country and language for last month"
- "How often do ChatGPT and Perplexity name my brand compared with my competitors this month?"
- "Which pages did AI answers cite about my brand last week, and how do those pages present us?"
- "Create a tracker group for my competitors with a keyword tracker for each of them"
- "Pause every tracker in my Competitors group"
- "How many more keyword trackers can I create on my plan?"

## Skill structure

```
skills/api-navigator/
├── SKILL.md                        # Core workflow, routing matrix, key rules
└── references/
    ├── endpoints.md                # Mention, analytics, list and usage endpoints
    ├── answers.md                  # AI Visibility answers, brand overview, citations
    ├── write-endpoints.md          # Creating, updating and deleting trackers and groups
    ├── channels-categories.md      # Channel IDs and category tokens
    └── auth-rate-limits.md         # Token refresh, rate limits, retries, caching
```

## What's new in 2.0.0

- Covers API v2 documentation version 2.1.17.
- New: AI Visibility answers, brand overview and citation analysis (`answers.md`).
- New: creating, updating, pausing, resuming and deleting trackers and tracker groups, with their own write limits (`write-endpoints.md`).
- Updated: parameters and response fields of every read endpoint, the tracker and group lists and account usage; token refresh errors and a retry helper.
- Corrected: read limits (5 per second and 100 per minute, per account, both answering `rate_limit_exceeded`), the `X-RateLimit-*` header names, and the cache (30 seconds, `GET` responses only).
- New: the podcast categories `podcast` and `spotify` (channel 99).
- Removed: the web category `comments`, which the API doesn't accept.

## License

MIT
