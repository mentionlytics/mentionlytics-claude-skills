# Mentionlytics Skills for Claude

Official [Mentionlytics](https://www.mentionlytics.com) skills marketplace for Claude Code and Claude apps. Give Claude deep knowledge of the Mentionlytics API for social listening, brand monitoring, and AI visibility workflows.

## Installation (Claude Code)

Add this marketplace once:

```
/plugin marketplace add mentionlytics/mentionlytics-claude-skills
```

Then install the plugin:

```
/plugin install mentionlytics-api@mentionlytics-skills
```

Or browse via `/plugin` → Discover.

## Available plugins

| Plugin | Description |
| :--- | :--- |
| `mentionlytics-api` | Query the Mentionlytics API (v2): mentions, top influencers, trending keywords, geo/language breakdowns, AI daily summaries, analytics (EMV, MQI), trackers, and account usage — with auth rotation and rate-limit handling built in. |

## Setup

Skills read your credentials from environment variables — no tokens are ever stored in the skill files:

```bash
export MENTIONLYTICS_API_TOKEN="your-bearer-token"
export MENTIONLYTICS_REFRESH_TOKEN="your-refresh-token"
```

## Using with Claude.ai / Claude API

The skills follow the portable Agent Skills format (SKILL.md with `name` + `description` frontmatter and plain Markdown), so individual skill folders can also be uploaded as custom skills in Claude.ai or used via the Skills API.

## Also available: Mentionlytics MCP Connector

Prefer live data over API scripting? The [Mentionlytics MCP connector](https://www.mentionlytics.com) connects your Mentionlytics account directly to Claude, ChatGPT, and Copilot — no code required. This skill and the MCP connector complement each other: use the MCP for interactive analysis, use this skill when building custom integrations and scripts.

## Contributing

Issues and PRs welcome. For questions about the Mentionlytics API itself, see the official API documentation or contact support.

## License

MIT — see [LICENSE](./LICENSE).
