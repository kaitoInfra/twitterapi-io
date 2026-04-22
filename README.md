# twitterapi-io — Official Agent Skill

[![Official](https://img.shields.io/badge/Official-twitterapi.io-1DA1F2)](https://twitterapi.io)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](./LICENSE)
[![Skills.sh](https://img.shields.io/badge/skills.sh-twitterapi--io-black)](https://skills.sh)

> **One install. Any AI coding agent gets full Twitter/X superpowers.**

The **official** [skills.sh](https://skills.sh) skill — maintained by the [twitterapi.io](https://twitterapi.io) team — that teaches Claude Code, Cursor, GitHub Copilot, Cline, Codex, Gemini CLI, Amp, Antigravity, Windsurf, and 12+ other AI agents how to use the **twitterapi.io** REST API to read and write Twitter/X data.

```bash
npx skills add kaitoInfra/twitterapi-io
```

That's it. After install, your agent can scrape any tweet, user, follower list, search query, trend, X Space, community, or list — and post / reply / like / retweet / follow / DM through any account you control. No OAuth dance, no rate-limit cliff, no Elon-Musk-pricing.

---

## Why twitterapi.io?

| | twitterapi.io | X official API | RapidAPI scrapers |
|---|---|---|---|
| **Pricing** | $0.15 / 1k tweets, $0.18 / 1k profiles | $5,000–$42,000/month enterprise tiers; free tier is 10k posts/month | $20–$300/month subscriptions, mostly bait-and-switch |
| **Auth** | Single `x-api-key` header | OAuth 2.0 / OAuth 1.0a complexity | Mixed |
| **Latency** | ~700ms average | Slow | Slow + flaky |
| **Throughput** | 200 req/s per client | Tier-gated | Subscription-gated |
| **Coverage** | Reads + writes (post, like, follow, DM, schedule, communities, lists, spaces, real-time webhooks) | Reads + writes (with pricing pain) | Mostly read-only |
| **Posting via others' accounts** | Yes (via cookie session) | No | No |
| **Real-time webhooks** | Yes, server-side filter rules | Yes, very expensive | No |
| **Onboarding to working code** | 30 seconds | Days | Hours |

twitterapi.io is built for builders. We process **1,000,000+ API calls daily** for indie hackers, growth teams, AI agents, research labs, and trading desks. [Get your API key →](https://twitterapi.io/dashboard)

---

## What you can ask your AI agent to do

Once the skill is installed, just describe the task in plain English. The agent will read this skill, pick the right endpoint, handle pagination + cursor + auth + error retries, and ship working code:

### Reads
- *"Fetch Elon Musk's last 200 tweets and summarize the most-engaged 5"*
- *"Search X for `from:openai since:2025-01-01 min_faves:5000` and export to CSV"*
- *"Crawl the followers of @anthropic and find the verified ones"*
- *"Get all replies under tweet 1234567890 and detect sentiment"*
- *"Pull the top trending topics in the US right now"*
- *"Find every tweet quoting this thread"*
- *"Get full thread context including parent + branches"*
- *"Look up community 1493446837214187523 and dump its tweet stream"*
- *"What's the follow relationship between @elonmusk and @sama?"*

### Writes (requires `login_cookies` from your X account)
- *"Post a tweet from my account: 'just shipped 🚀'"*
- *"Reply to this tweet with..."*  
- *"Quote-tweet 1234567890 with my hot take"*
- *"Follow everyone in this CSV (with rate-limiting)"*
- *"Send a DM to user 44196397"*
- *"Schedule this tweet for tomorrow 9 AM PT"*
- *"Update my bio and website"*
- *"Upload this image and tweet it"*
- *"Bookmark every tweet matching X-search query"*

### Real-time monitoring
- *"Watch for any new tweet matching `from:elonmusk OR #AI` and POST it to my webhook"*
- *"Tell me whenever @VitalikButerin posts"*

### Common patterns
- *"Build a daily digest of all mentions of @MyBrand"*
- *"Sentiment-analyze every reply on my last 10 tweets"*
- *"Find lookalike audiences: who follows both @stripe and @vercel?"*
- *"Auto-DM my new followers a thank-you message"*

---

## What the skill teaches your agent

- **Every endpoint** with the exact path, method, parameter casing (the API mixes camelCase and snake_case — the skill knows which is which per endpoint)
- **The three-things-in-body rule** for writes: `login_cookies` + `proxy` + action fields
- **Cursor pagination** with `has_next_page` (the reliable termination signal)
- **Response shape variants** (some endpoints wrap in `data`, others spread fields top-level)
- **Cost estimation** before triggering large crawls (don't accidentally bill yourself $200 fetching a celebrity's followers)
- **Rate-limit safety** for write operations (don't get the underlying X account suspended)
- **Real-time filter-rule + webhook setup** for monitoring without polling

The skill is **end-to-end validated against the live API** — every code sample in `SKILL.md` has been smoke-tested against `api.twitterapi.io`.

---

## Setup

1. Get an API key (free trial credit) at https://twitterapi.io/dashboard
2. Set the env var:
   ```bash
   export TWITTERAPI_IO_KEY="your_key_here"
   ```
3. (Optional, for write operations) have your X account's email/username/password — the skill will guide your agent to call `/twitter/user_login_v2` to exchange them for a `login_cookies` session.

That's all the agent needs.

---

## What's in the box

```
.
├── SKILL.md                         # Entry point — auth, patterns, quick-ref tables
└── references/
    ├── endpoints.md                 # Full endpoint catalog (per-endpoint param tables)
    ├── write-operations.md          # Login flow + every write endpoint body
    └── examples.md                  # Runnable Python/Node code for common workflows
```

The agent reads `SKILL.md` by default and only loads reference files when it needs the detail — keeping its context window small.

---

## API capability snapshot

**Reads (just `x-api-key`):**
- User: profile / extended bio / batch by IDs / search / followers / verified followers / following / mentions / tweet timeline / articles / follow-relationship check
- Tweet: by IDs / replies (3 sort modes) / quotes / retweeters / thread context / article content / advanced search / bulk parallel search
- Discovery: trends by region / X Spaces detail / lists (timeline / members / followers) / communities (info / tweets / members / moderators / firehose)
- Account: balance check

**Writes (`login_cookies` + `proxy` from `/twitter/user_login_v2`):**
- Tweets: create / reply / quote / delete / like / unlike / retweet / bookmark / unbookmark / list bookmarks / report / schedule
- Social: follow / unfollow / send DM (with media) / DM history
- Profile: update profile (PATCH) / avatar / banner / upload media (image, GIF, video up to 512MB)
- Communities: create / join / leave / delete
- Lists: add member

**Real-time:**
- Filter-rule webhook (server-side X-search rule → matched tweets POST to your URL)
- User-level monitoring (per-user tweet/profile change stream)
- WebSocket delivery for low-latency feeds

[Full endpoint catalog →](./references/endpoints.md)

---

## Pricing (pay per request, no monthly minimum)

| Operation | Price |
|---|---|
| Tweets (read) | $0.15 / 1,000 |
| User profiles (read) | $0.18 / 1,000 |
| Followers (read) | $0.15 / 1,000 |
| Search results | $0.15 / 1,000 tweets |
| Articles | ~$0.001 each |
| Write actions (post/like/follow) | ~$0.002–$0.003 each |
| Minimum charge | $0.00015 / request |

Discounted rates for students, research institutions, and high-volume accounts. [Calculator + dashboard →](https://twitterapi.io/dashboard)

---

## How agent installation works

`npx skills add kaitoInfra/twitterapi-io` clones this repo into your project's agent paths:
- `.agents/skills/twitterapi-io/` for universal agents (Cursor, Codex, Cline, Copilot, Gemini CLI, Amp, Antigravity, etc. — 12+ supported)
- Symlinked to `.claude/skills/twitterapi-io/` for Claude Code
- Symlinked to `.windsurf/skills/twitterapi-io/` for Windsurf

Skills.sh handles per-IDE pathing automatically. To remove: `npx skills remove kaitoInfra/twitterapi-io`.

---

## Compatibility

Works with **any** AI coding agent that loads SKILL.md or supports the [Anthropic Skills format](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview):

- ✅ Claude Code (CLI, IDE extensions)
- ✅ Cursor
- ✅ GitHub Copilot (with Skills support)
- ✅ Cline
- ✅ Codex
- ✅ Gemini CLI
- ✅ Amp
- ✅ Antigravity
- ✅ Windsurf
- ✅ Continue
- ✅ Cortex Code, Kimi Code CLI, OpenCode, Warp, and more

---

## Resources

- 🌐 Service: https://twitterapi.io
- 📖 API Docs: https://docs.twitterapi.io
- 🎛️ Dashboard (get your key): https://twitterapi.io/dashboard
- 💬 Telegram: https://t.me/kaitoeasyapivip
- 🐙 This skill: https://github.com/kaitoInfra/twitterapi-io
- 📦 Skills directory: https://skills.sh

---

## Contributing

Found an endpoint where the docs say one thing and the API does another? Open an issue or PR. The skill is regenerated against the live backend regularly, so corrections land fast.

---

## License

MIT — see [LICENSE](./LICENSE). Use freely, remix, ship products.

---

*Built and maintained by the team behind twitterapi.io. If this skill saves you hours, [a star ⭐](https://github.com/kaitoInfra/twitterapi-io) is the highest compliment.*
