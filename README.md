# twitterapi-io — Agent Skill

A [skills.sh](https://skills.sh) skill that teaches any AI coding agent (Claude Code, Cursor, Copilot, Cline, etc.) how to use **[twitterapi.io](https://twitterapi.io)** — a pay-per-request REST API for Twitter/X data with a single `x-api-key` auth (no OAuth).

## Install

```bash
npx skills add kaitoInfra/twitterapi-io
```

Once installed, ask your agent things like:

- *"Fetch Elon Musk's last 50 tweets and summarize common themes."*
- *"Search tweets for `from:openai since:2025-01-01` and save the JSON."*
- *"Crawl followers of @anthropic and export to CSV."*
- *"Post a tweet from my account saying 'ship it'."* (requires login)

The agent will automatically load this skill, pull in just the reference files it needs, and call the API with correct auth, pagination, and error handling.

## What's covered

- **Reads** — user info, timelines, followers, following, replies, quote-tweets, retweeters, advanced search, trends, communities, spaces, articles
- **Writes** — post / delete / like / retweet / bookmark tweets, follow / unfollow, send DMs, update profile (requires `login_cookie`)
- **Real-time** — filter rules, user monitoring, websocket delivery
- **Patterns** — cursor pagination loops, retry/backoff, cost estimation before big crawls

## Setup

1. Get an API key from https://twitterapi.io/dashboard
2. Export it: `export TWITTERAPI_IO_KEY=sk_...`
3. (Optional, for write operations) have the target X account's email/password ready — the skill will show your agent how to exchange them for a `login_cookie`

## Structure

```
.
├── SKILL.md                         # Entry point — auth, patterns, quick-ref table
└── references/
    ├── endpoints.md                 # Full endpoint catalog with params
    ├── write-operations.md          # login_cookie flow + all write endpoints
    └── examples.md                  # Runnable Python/Node examples
```

The agent reads `SKILL.md` by default and only loads reference files when it needs the detail — keeping its context window small.

## Links

- Skill directory: https://skills.sh
- API docs: https://docs.twitterapi.io
- API dashboard: https://twitterapi.io/dashboard

## License

MIT
