---
name: twitterapi-io
description: Official skill for twitterapi.io — query Twitter/X data (tweets, profiles, followers, advanced search, trends, spaces, communities, lists) and perform authenticated actions (post, reply, like, retweet, follow, DM) via the twitterapi.io REST API using a single `x-api-key` header — no OAuth. Use when the user needs to scrape, analyze, monitor, or automate X/Twitter without going through the official developer portal.
---

# twitterapi.io

**Official skill** maintained by the [twitterapi.io](https://twitterapi.io) team. Paths, parameters, and body fields in this skill are verified against the live backend.

## When to use this skill

Trigger when the user wants any of:

- Fetch tweets, user profiles, followers/following, trends, replies, quote-tweets, retweeters, articles
- Advanced tweet search (X operators, date ranges, engagement filters)
- Monitor users or filter rules in near real time
- Post / delete / like / retweet / bookmark / quote-tweet / reply / follow / DM / schedule tweets
- Update profile / avatar / banner, upload media
- Create/join/leave communities, manage lists, report content
- Anything involving `twitterapi.io`, `api.twitterapi.io`, `x-api-key`, `login_cookies`, or the X API without OAuth

## Core facts

| | |
|---|---|
| Base URL | `https://api.twitterapi.io` |
| Prefix | `/twitter/...` most endpoints; `/oapi/x_user_stream/...` and `/oapi/tweet_filter/...` for real-time monitoring/webhooks; `/oapi/my/info` for balance |
| Auth header | `x-api-key: YOUR_KEY` |
| Dashboard | https://twitterapi.io/dashboard |
| Docs | https://docs.twitterapi.io |
| Rate limit | ~200 req/s per client |
| Pricing | ~$0.15/1k tweets, ~$0.18/1k profiles, $0.00015 minimum |

## ⚠️ Two rules that catch people

### Rule 1 — Parameter naming is per-endpoint

No universal snake_case or camelCase rule. Examples (all correct):
- `/twitter/user/followers?userName=` (camel)
- `/twitter/user/verifiedFollowers?user_id=` (snake)
- `/twitter/user/articles?username=` (all-lowercase)
- `/twitter/tweets?tweet_ids=` (snake) but `/twitter/tweet/replies?tweetId=` (camel)
- `/twitter/list/tweets_timeline?listId=` (camel) but `/twitter/list/members?list_id=` (snake)

**Copy the exact parameter name from [references/endpoints.md](references/endpoints.md). Don't normalize.**

### Rule 2 — Writes need three things in body, not two

Every write endpoint requires:
1. `login_cookies` (**plural**, not `login_cookie`) — base64-encoded JSON from `/twitter/user_login_v2`
2. `proxy` (HTTP/SOCKS proxy URL — configure in dashboard)
3. Action-specific fields, almost always **snake_case** (`tweet_id`, `user_id`)

Plus key field-name traps:
- `create_tweet_v2` text field is `tweet_text` (not `text`); reply field is `reply_to_tweet_id` (not `in_reply_to_tweet_id`)
- `update_profile_v2` uses `description` (not `bio`)
- `bookmarks_v2` uses `count` (not `pageSize`)
- `send_dm_to_user`: `user_id` + `text`; optional `media_id` (singular)
- Monitoring add uses `x_user_name`; remove uses `id_for_user` (different fields per endpoint!)

See [references/write-operations.md](references/write-operations.md) for full body shapes.

### Rule 3 — Response shape varies per endpoint

- `data`-wrapped: `user/info`, `user_about`, `last_tweets`, `tweet_timeline`, `trends`, `check_follow_relationship` → `r["data"]["..."]`
- Flat with envelope: `followers`, `followings`, `replies`, `mentions`, `community/tweets` → `r["followers"]`, `r["tweets"]`, etc.
- Flat without envelope: `advanced_search`, `thread_context`, `user/search`, `get_tweets_from_all_community` → just `{tweets[], has_next_page, next_cursor}`
- Named top-level field: `community/info` → `r["community_info"]`; `batch_info_by_ids` → `r["users"]`
- `oapi/my/info` → `{recharge_credits, total_bonus_credits}` (no status wrapper)

Prefer defensive access: `r.get("tweets", r.get("data", {}).get("tweets", []))`.

## Security

**Never** hardcode the API key. Read from env `TWITTERAPI_IO_KEY`. If missing, ask the user.

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
d = r.json()["data"]
print(f"{d['userName']} — {d['followers']:,} followers")
```

## Endpoint quick reference

Reads (API key only). **Exact param names — copy as shown.**

| Capability | Method | Path & key param |
|---|---|---|
| User by screen name | GET | `/twitter/user/info?userName=` |
| Extended bio | GET | `/twitter/user_about?userName=` |
| Batch users by IDs | GET | `/twitter/user/batch_info_by_ids?userIds=` |
| Search users | GET | `/twitter/user/search?query=` (**not** `keyword`) |
| Recent tweets | GET | `/twitter/user/last_tweets?userName=` (or `userId=`) |
| Timeline by ID | GET | `/twitter/user/tweet_timeline?userId=` |
| User's articles | GET | `/twitter/user/articles?username=` (**all-lowercase**) |
| Mentions of user | GET | `/twitter/user/mentions?userName=` |
| Followers | GET | `/twitter/user/followers?userName=&pageSize=200` |
| Verified followers | GET | `/twitter/user/verifiedFollowers?user_id=` (**snake**) |
| Followings | GET | `/twitter/user/followings?userName=` |
| Check follow | GET | `/twitter/user/check_follow_relationship?source_user_name=&target_user_name=` |
| Tweets by IDs | GET | `/twitter/tweets?tweet_ids=` (**snake**) |
| Tweet replies | GET | `/twitter/tweet/replies?tweetId=` |
| Replies (sortable) | GET | `/twitter/tweet/replies/v2?tweetId=&queryType=Latest` |
| Quote tweets | GET | `/twitter/tweet/quotes?tweetId=` |
| Retweeters | GET | `/twitter/tweet/retweeters?tweetId=` |
| Thread context | GET | `/twitter/tweet/thread_context?tweetId=` |
| Article content | GET | `/twitter/article?tweet_id=` (**snake**) |
| Advanced search | GET | `/twitter/tweet/advanced_search?query=` |
| Bulk advanced search | POST | `/twitter/tweet/bulk_advanced_search` body `{queries:[{query,queryType?,cursor?}]}` |
| Trends | GET | `/twitter/trends?woeid=1&count=30` |
| Space detail | GET | `/twitter/spaces/detail?space_id=` (**snake**) |
| List tweets (with filters) | GET | `/twitter/list/tweets?listId=` |
| List tweets_timeline | GET | `/twitter/list/tweets_timeline?listId=` |
| List members / followers | GET | `/twitter/list/{members,followers}?list_id=` (**snake**) |
| Community info | GET | `/twitter/community/info?community_id=` |
| Community tweets/members/mods | GET | `/twitter/community/{tweets,members,moderators}?community_id=` |
| All-community firehose | GET | `/twitter/community/get_tweets_from_all_community?query=` (required!) |
| Account balance | GET | `/oapi/my/info` |

Real-time monitoring (`/oapi/x_user_stream/*`). **Field names differ per endpoint!**

| Capability | Method | Path & body |
|---|---|---|
| Add monitor | POST | `/oapi/x_user_stream/add_user_to_monitor_tweet` body `{x_user_name}` ← **not `user_id`** |
| List monitors | GET | `/oapi/x_user_stream/get_user_to_monitor_tweet` (+ `query_type` 0/1/2) |
| Remove monitor | POST | `/oapi/x_user_stream/remove_user_to_monitor_tweet` body `{id_for_user}` ← **different field from add** |

Webhook filter rules (`/oapi/tweet_filter/*`):

| Capability | Method | Path & body |
|---|---|---|
| Add rule | POST | `/oapi/tweet_filter/add_rule` body `{tag, value, interval_seconds?=60}` (range 0.05–86400) |
| List rules | GET | `/oapi/tweet_filter/get_rules` |
| Update rule | POST | `/oapi/tweet_filter/update_rule` body `{rule_id, tag, value, interval_seconds?, is_effect?}` |
| Delete rule | DELETE | `/oapi/tweet_filter/delete_rule` body `{rule_id}` (**body JSON, not query**) |

Writes — require `login_cookies` + `proxy` in body. `login_cookies` is **base64(json(cookie_dict))** from `/twitter/user_login_v2`.

| Capability | Method | Path | Key body fields (beyond login_cookies+proxy) |
|---|---|---|---|
| Log in → cookies | POST | `/twitter/user_login_v2` | `{user_name, email, password, totp_secret?, proxy}` |
| Create tweet | POST | `/twitter/create_tweet_v2` | `tweet_text` (req); optional `reply_to_tweet_id`, `quote_tweet_id`, `community_id`, `media_ids[]`, `schedule_for` |
| Delete | POST | `/twitter/delete_tweet_v2` | `tweet_id` |
| Like / Unlike | POST | `/twitter/{like,unlike}_tweet_v2` | `tweet_id` |
| Retweet | POST | `/twitter/retweet_tweet_v2` | `tweet_id` |
| Bookmark / Unbookmark | POST | `/twitter/{bookmark,unbookmark}_tweet_v2` | `tweet_id` |
| List bookmarks | POST | `/twitter/bookmarks_v2` | optional `count` (def 20, **not `pageSize`**), `cursor` |
| Follow / Unfollow | POST | `/twitter/{follow,unfollow}_user_v2` | `user_id` |
| Send DM | POST | `/twitter/send_dm_to_user` | `user_id`, `text`, optional `media_id` (singular), `reply_to_message_id` |
| Report | POST | `/twitter/report_v2` | (`tweet_id` OR `user_id`), `reason` (enum, see endpoints.md) |
| Upload media | POST | `/twitter/upload_media_v2` | **multipart**: `file`, optional `media_category`, `is_long_video` |
| Update profile | PATCH | `/twitter/update_profile_v2` | JSON: at least one of `name`, **`description`** (not `bio`!), `location`, `url` |
| Update avatar/banner | PATCH | `/twitter/update_{avatar,banner}_v2` | **multipart**: `file` |
| Create community | POST | `/twitter/create_community_v2` | `name`, `description` (both required) |
| Join / Leave community | POST | `/twitter/{join,leave}_community_v2` | `community_id` |
| Delete community | POST | `/twitter/delete_community_v2` | `community_id`, **`community_name`** (both required) |
| Add list member | POST | `/twitter/list/add_member_v2` | `list_id`, `user_id` |

For exact param lists, response shapes, and edge cases see [references/endpoints.md](references/endpoints.md).

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

```python
# FastAPI-style 4xx/5xx
{ "detail": "tweet_id is required" }
{ "detail": [{"type":"missing","loc":["body","file"]}] }  # 422 on multipart

# Semantic 200 errors
{ "status": "error", "msg": "No active monitoring subscription" }
```

- `401` → bad `x-api-key` or expired `login_cookies`
- `402` → top up balance at dashboard
- `429` → rate-limited; exponential backoff
- `400/422` → read `detail` — it names the missing/wrong field
- `500` → transient; retry with jitter. If repeated, the param name is wrong

### Cost awareness

Before a loop that could paginate for a long time (followers of a mega-account, years of advanced search), **estimate cost up front** and confirm with the user if it could exceed a few dollars. A 100M-follower crawl is $15k+ at $0.15/1k.

## Anti-patterns

- Do **not** assume a universal param-casing rule — check endpoints.md
- Do **not** use `login_cookie` (singular) — it's `login_cookies` plural
- Do **not** forget `proxy` in write bodies — every write requires it
- Do **not** send `text` for `create_tweet_v2` — use `tweet_text`
- Do **not** use `bio` in `update_profile_v2` — use `description`
- Do **not** use `in_reply_to_tweet_id` for `create_tweet_v2` — it's `reply_to_tweet_id`
- Do **not** send JSON to `upload_media_v2` / `update_avatar_v2` / `update_banner_v2` — they're **multipart/form-data**
- Do **not** use `pageSize` for `bookmarks_v2` — it's `count`
- Do **not** send `user_id` to monitoring add — it's `x_user_name`; remove uses `id_for_user`
- Do **not** log or commit API keys, `login_cookies`, proxies, or `totp_secret`
- Do **not** use `developer.twitter.com` / official X OAuth — this API replaces them
- Do **not** poll `/user/last_tweets` for "real-time" — use `/oapi/tweet_filter/add_rule` + webhook
