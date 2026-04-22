---
name: twitterapi-io
description: Official skill for twitterapi.io — query Twitter/X data (tweets, profiles, followers, advanced search, trends, spaces, communities, lists) and perform authenticated actions (post, reply, like, retweet, follow, DM) via the twitterapi.io REST API using a single `x-api-key` header — no OAuth. Use when the user needs to scrape, analyze, monitor, or automate X/Twitter without going through the official developer portal.
---

# twitterapi.io

**Official skill** maintained by the [twitterapi.io](https://twitterapi.io) team — a pay-per-request REST API that returns Twitter/X data (tweets, users, timelines, search, trends, spaces, communities, lists) and performs write actions on behalf of a logged-in X account. All endpoint paths, parameters, and body fields in this skill were validated against the live API.

## When to use this skill

Trigger when the user wants any of:

- Fetch tweets, user profiles, followers/following, trends, replies, quote-tweets, retweeters
- Advanced tweet search (keywords, date ranges, engagement filters)
- Monitor users or filter rules in near real time
- Post / delete / like / retweet / bookmark / reply to tweets
- Follow / unfollow / send DM / update profile
- Anything involving `twitterapi.io`, `api.twitterapi.io`, or `x-api-key` in their code or questions

## Core facts (memorize these)

| Thing | Value |
|---|---|
| Base URL | `https://api.twitterapi.io` |
| Path prefix | `/twitter/...` for most endpoints; `/oapi/x_user_stream/...` and `/oapi/tweet_filter/...` for real-time monitoring and webhook rules; `/oapi/my/info` for account balance |
| Auth header | `x-api-key: YOUR_KEY` |
| Dashboard | https://twitterapi.io/dashboard |
| Docs | https://docs.twitterapi.io |
| Auth model | API key alone for **reads**; API key **+ `login_cookies` + `proxy`** in JSON body for **writes** |
| Rate limit | Up to ~200 req/s per client |
| Pricing | ~$0.15 / 1k tweets, ~$0.18 / 1k profiles, $0.00015 minimum per request |

## ⚠️ Two rules that catch people

### Rule 1 — Parameter naming is **inconsistent across endpoints**

There is NO universal snake_case or camelCase rule. Every endpoint has its own convention, e.g.:
- `/twitter/user/followers` takes `userName` (camel)
- `/twitter/user/verifiedFollowers` takes `user_id` (snake)
- `/twitter/list/tweets_timeline` takes `listId` (camel)
- `/twitter/list/members` takes `list_id` (snake)
- `/twitter/tweets` takes `tweet_ids` (snake), but `/twitter/tweet/replies` takes `tweetId` (camel)

**Always look up the exact parameter name in [references/endpoints.md](references/endpoints.md) — do not guess the casing.**

### Rule 2 — Writes need **three** things, not two

Every write endpoint requires in the JSON body:
1. `login_cookies` (**plural**, not `login_cookie`)
2. `proxy` (HTTP/SOCKS proxy URL — configure in dashboard)
3. Action-specific fields (almost always **snake_case** like `tweet_id`, `user_id`)

Plus the text field for `create_tweet_v2` is `tweet_text` (not `text`). And for `send_dm_to_user` it's `text` (not `message`). See [references/write-operations.md](references/write-operations.md) for exact body shapes per endpoint.

## Security

**Never** hardcode the API key. Read from env `TWITTERAPI_IO_KEY`. If missing, ask the user or prompt them to set it.

## Minimal working example

```bash
curl -s "https://api.twitterapi.io/twitter/user/info?userName=elonmusk" \
  -H "x-api-key: $TWITTERAPI_IO_KEY"
```

```python
import os, requests

BASE = "https://api.twitterapi.io"
HEADERS = {"x-api-key": os.environ["TWITTERAPI_IO_KEY"]}

r = requests.get(f"{BASE}/twitter/user/info",
                 headers=HEADERS,
                 params={"userName": "elonmusk"},
                 timeout=30)
r.raise_for_status()
data = r.json()["data"]
print(data["userName"], "—", f'{data["followers"]:,} followers')
```

## Endpoint quick reference

Read endpoints (API key only). **Param casing is exactly what's shown — do not change it.**

| Capability | Method | Path & required param |
|---|---|---|
| User by screen name | GET | `/twitter/user/info?userName=` |
| User extended bio | GET | `/twitter/user_about?userName=` |
| Batch users by IDs | GET | `/twitter/user/batch_info_by_ids?userIds=` |
| Search users | GET | `/twitter/user/search?query=` ← **not** `keyword` |
| User's recent tweets | GET | `/twitter/user/last_tweets?userName=` |
| User timeline by ID | GET | `/twitter/user/tweet_timeline?userId=` |
| User mentions | GET | `/twitter/user/mentions?userName=` |
| Followers | GET | `/twitter/user/followers?userName=&pageSize=200` |
| Verified followers | GET | `/twitter/user/verifiedFollowers?user_id=` ← **snake_case** |
| Following | GET | `/twitter/user/followings?userName=` |
| Check follow | GET | `/twitter/user/check_follow_relationship?source_user_name=&target_user_name=` |
| Tweets by IDs | GET | `/twitter/tweets?tweet_ids=` ← **snake_case** |
| Tweet replies | GET | `/twitter/tweet/replies?tweetId=` |
| Tweet replies (sorted) | GET | `/twitter/tweet/replies/v2?tweetId=&sort=Latest` |
| Quote tweets | GET | `/twitter/tweet/quotes?tweetId=` |
| Retweeters | GET | `/twitter/tweet/retweeters?tweetId=` |
| Thread context | GET | `/twitter/tweet/thread_context?tweetId=` |
| Article | GET | `/twitter/article?tweet_id=` ← **snake_case** |
| Advanced search | GET | `/twitter/tweet/advanced_search?query=` |
| Trends | GET | `/twitter/trends?woeid=1` |
| Space detail | GET | `/twitter/spaces/detail?space_id=` |
| List timeline | GET | `/twitter/list/tweets_timeline?listId=` |
| List members / followers | GET | `/twitter/list/{members,followers}?list_id=` ← **snake** |
| Community info | GET | `/twitter/community/info?community_id=` |
| Community tweets/members/mods | GET | `/twitter/community/{tweets,members,moderators}?community_id=` |
| All-community firehose | GET | `/twitter/community/get_tweets_from_all_community?query=` |
| Account balance | GET | `/oapi/my/info` |

Real-time monitoring (`/oapi/x_user_stream/*`, body JSON not query string):

| Capability | Method | Path |
|---|---|---|
| Monitor a user | POST | `/oapi/x_user_stream/add_user_to_monitor_tweet` body `{user_id}` |
| List monitors | GET | `/oapi/x_user_stream/get_user_to_monitor_tweet` |
| Stop monitoring | POST | `/oapi/x_user_stream/remove_user_to_monitor_tweet` body `{user_id}` |

Webhook filter rules (`/oapi/tweet_filter/*`):

| Capability | Method | Path |
|---|---|---|
| Add rule | POST | `/oapi/tweet_filter/add_rule` body `{tag, value, interval_seconds, is_effect}` |
| List rules | GET | `/oapi/tweet_filter/get_rules` |
| Update rule | POST | `/oapi/tweet_filter/update_rule` body `{rule_id, tag, value, ...}` |
| Delete rule | DELETE | `/oapi/tweet_filter/delete_rule` body `{rule_id}` ← **body, not query** |

Writes (POST unless noted; require `login_cookies` + `proxy` in body). See [references/write-operations.md](references/write-operations.md) for exact body shapes.

| Capability | Method | Path |
|---|---|---|
| Log in → `login_cookies` | POST | `/twitter/user_login_v2` |
| Create / reply / quote tweet | POST | `/twitter/create_tweet_v2` (text field: `tweet_text`) |
| Delete tweet | POST | `/twitter/delete_tweet_v2` |
| Like / Unlike | POST | `/twitter/like_tweet_v2` · `/twitter/unlike_tweet_v2` |
| Retweet | POST | `/twitter/retweet_tweet_v2` |
| Bookmark / Unbookmark | POST | `/twitter/bookmark_tweet_v2` · `/twitter/unbookmark_tweet_v2` |
| Follow / Unfollow | POST | `/twitter/follow_user_v2` · `/twitter/unfollow_user_v2` |
| Send DM | POST | `/twitter/send_dm_to_user` (fields: `user_id`, `text`) |
| Upload media | POST | `/twitter/upload_media_v2` (multipart, `file` field) |
| Update profile / avatar / banner | **PATCH** | `/twitter/{update_profile_v2,update_avatar_v2,update_banner_v2}` |
| Communities | POST | `/twitter/{create,join,leave,delete}_community_v2` |

## Recurring patterns

### Cursor pagination — use `has_next_page`

```python
def iter_followers(user_name):
    cursor = ""
    while True:
        r = requests.get(f"{BASE}/twitter/user/followers",
                         headers=HEADERS,
                         params={"userName": user_name, "cursor": cursor, "pageSize": 200},
                         timeout=30).json()
        yield from r.get("followers", [])
        if not r.get("has_next_page"):
            break
        cursor = r.get("next_cursor") or ""
```

### Error handling

Two error shapes to handle:

```python
# FastAPI-style 400/422/500
{ "detail": "tweet_id is required" }
{ "detail": [{"type":"missing","loc":["body","file"],"msg":"Field required"}] }

# Semantic 200 errors
{ "status": "error", "msg": "No active monitoring subscription" }
```

Codes:
- `401` → missing/invalid `x-api-key`, or (for writes) expired `login_cookies`
- `402` → account balance too low (top up at dashboard)
- `429` → rate-limited; back off exponentially
- `400` / `422` → parameter missing or wrong name — read the `detail` and add/rename it
- `500` → transient; retry with jitter. If persistent, the parameter name is likely wrong

Surface `response.json().get("detail") or response.json().get("msg")` in error messages.

### Cost awareness

Before a loop that could paginate indefinitely (followers of a mega-account, years-long advanced search), **estimate cost up front** and confirm with the user if it could exceed a few dollars. Individual reads are cheap, but a full follower crawl of a 100M-follower account is $15k+.

### Response shape varies per endpoint

- **data-wrapped**: `user/info`, `user_about`, `last_tweets`, `tweet_timeline`, `trends`, `check_follow_relationship` → `r["data"]["..."]`
- **flat list**: `followers`, `followings`, `replies`, `quotes`, `advanced_search`, `mentions`, `community/tweets` → `r["followers"]`, `r["tweets"]`, etc. directly
- **named top-level field**: `community/info` → `r["community_info"]`, `batch_info_by_ids` → `r["users"]`

Prefer `r.get("tweets", r.get("data", {}).get("tweets", []))` over hardcoding a path.

## Anti-patterns to avoid

- Do **not** assume a universal casing rule for parameters — check endpoints.md
- Do **not** use `login_cookie` (singular) — the field is `login_cookies` (plural)
- Do **not** forget `proxy` in write bodies — every write requires it
- Do **not** use `text` as the field name for create_tweet — it's `tweet_text`
- Do **not** put the API key in URL query strings or log it
- Do **not** use `developer.twitter.com` / official X OAuth — this API replaces them
- Do **not** rely on `next_cursor` being empty for termination — use `has_next_page === false`
- Do **not** assume write endpoints accept DELETE — they're all POST to separate `_v2` paths (e.g. `unlike_tweet_v2`)
- Do **not** poll a user's timeline in a hot loop for "real-time" — use `/oapi/tweet_filter/add_rule` or `/oapi/x_user_stream/add_user_to_monitor_tweet` instead
