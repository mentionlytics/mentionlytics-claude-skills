# Mentionlytics API Plugin

Teach Claude how to work with the [Mentionlytics API](https://www.mentionlytics.com) (v2) — fetch mentions, analyze sentiment, find top influencers, extract trending keywords, and pull analytics like EMV and MQI, with correct authentication rotation and rate-limit handling out of the box.

## What's included

| Skill | Purpose |
| :--- | :--- |
| `api-navigator` | Full operational guide for the Mentionlytics API: endpoint routing, parameters, channel/category enums, auth token rotation, rate limits, and caching behavior. |

## Setup

1. Get your API tokens from your Mentionlytics account (API access required on your plan).
2. Export them as environment variables:

```bash
export MENTIONLYTICS_API_TOKEN="your-bearer-token"
export MENTIONLYTICS_REFRESH_TOKEN="your-refresh-token"
```

The skill never hardcodes or prints tokens.

## Example prompts

- "Pull the top 20 influencers mentioning my brand this month from the Mentionlytics API"
- "Write a Python script that fetches yesterday's mentions and groups them by sentiment"
- "What's the trending hashtag breakdown for my competitors tracker over the last 7 days?"
- "Break down my brand mentions by country and language for last month"

## Skill structure

```
skills/api-navigator/
├── SKILL.md                        # Core workflow, routing matrix, key rules
└── references/
    ├── endpoints.md                # Per-endpoint parameters and examples
    ├── channels-categories.md      # Channel ID enums and category tokens
    └── auth-rate-limits.md         # Token rotation, 429 handling, caching
```

## License

MIT
