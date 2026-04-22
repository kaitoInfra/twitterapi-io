---
name: twitterapi-io
description: Official skill for twitterapi.io — query Twitter/X data (tweets, profiles, followers, advanced search, trends, spaces, communities, lists) and perform authenticated actions (post, reply, like, retweet, follow, DM) via the twitterapi.io REST API using a single `x-api-key` header — no OAuth. Use when the user needs to scrape, analyze, monitor, or automate X/Twitter without going through the official developer portal.
---

# twitterapi.io

**Official skill** maintained by the [twitterapi.io](https://twitterapi.io) team — a pay-per-request REST API that returns Twitter/X data (tweets, users, timelines, search, trends, spaces, communities, lists) and performs write actions on behalf of a logged-in X account.

## When to use this skill

Trigger when the user wants any of:

- Fetch tweets, user profiles, followers/following, trends, replies, quote-tweets, retweeters
- Advanced tweet search (keywords, date ranges, engagement filters)
- Monitor specific users or filter rules in near real time
- Post / delete / like / retweet / bookmark / reply to tweets
- Follow / unfollow / send DM / update profile
- Anything involving `twitterapi.io`, `api.twitterapi.io`, or `x-api-key` in their code or questions

## Core facts (memorize these)

| Thing | Value |
|---|---|
| Base URL | `https://api.twitterapi.io` |
| Path prefix | `/twitter/...` for most endpoints; `/oapi/x_user_stream/...` and `/oapi/tweet_filter/...` for real-time monitoring and webhook rules |
| Auth header | `x-api-key: YOUR_KEY` (case-insensitive) |
| Param casing | **camelCase** — `userName`, `userId`, `tweetId`, `tweetIds`, `pageSize`, `cursor`, `communityId`, `listId` |
| Dashboard (get key) | https://twitterapi.io/dashboard |
| Docs | https://docs.twitterapi.io |
| Auth model | API key alone for **reads**; API key **+ `login_cookie` in body** for **writes** |
| Pagination | `cursor` query param; response has `next_cursor` and `has_next_page` — terminate on `has_next_page === false` |
| Page size | Typically 200 for user lists, 20 for tweet lists; often configurable via `pageSize` (20–200) |
| Rate limit | Up to ~200 req/s per client |
| Pricing | ~$0.15 / 1k tweets, ~$0.18 / 1k profiles, $0.00015 minimum per request |

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
print(r.json())
```

```javascript
const BASE = "https://api.twitterapi.io";
const headers = { "x-api-key": process.env.TWITTERAPI_IO_KEY };

const res = await fetch(`${BASE}/twitter/user/info?userName=elonmusk`, { headers });
if (!res.ok) throw new Error(`${res.status} ${await res.text()}`);
console.log(await res.json());
```

## Endpoint quick reference

Reads (API key only):

| Capability | Method | Path |
|---|---|---|
| User by screen name | GET | `/twitter/user/info?userName=` |
| User about / extended bio | GET | `/twitter/user_about?userName=` |
| Batch users by IDs | GET | `/twitter/user/batch_info_by_ids?userIds=` |
| Search users | GET | `/twitter/user/search?keyword=` |
| User's recent tweets (by name) | GET | `/twitter/user/last_tweets?userName=` |
| User timeline (by ID, sorted) | GET | `/twitter/user/tweet_timeline?userId=` |
| User mentions | GET | `/twitter/user/mentions?userName=` |
| User followers | GET | `/twitter/user/followers?userName=&cursor=&pageSize=` |
| Verified followers only | GET | `/twitter/user/verifiedFollowers?userName=` |
| User following | GET | `/twitter/user/followings?userName=&cursor=` |
| Check follow relationship | GET | `/twitter/user/check_follow_relationship?sourceId=&targetId=` |
| Tweets by IDs | GET | `/twitter/tweets?tweetIds=` |
| Tweet replies | GET | `/twitter/tweet/replies?tweetId=&cursor=` |
| Tweet replies (sortable) | GET | `/twitter/tweet/replies/v2?tweetId=&sort=&cursor=` |
| Quote tweets | GET | `/twitter/tweet/quotes?tweetId=&cursor=` |
| Retweeters | GET | `/twitter/tweet/retweeters?tweetId=&cursor=` |
| Thread context | GET | `/twitter/tweet/thread_context?tweetId=` |
| Article | GET | `/twitter/article?tweetId=` |
| Advanced search | GET | `/twitter/tweet/advanced_search?query=&cursor=` |
| Trends | GET | `/twitter/trends?woeid=` |
| Space detail | GET | `/twitter/spaces/detail?spaceId=` |
| List timeline / members / followers | GET | `/twitter/list/tweets_timeline?listId=` etc. |
| Community info / tweets / members | GET | `/twitter/community/info?communityId=` etc. |

Real-time monitoring (note `/oapi/` prefix):

| Capability | Method | Path |
|---|---|---|
| Start monitoring a user | POST | `/oapi/x_user_stream/add_user_to_monitor_tweet` |
| Stop monitoring | POST | `/oapi/x_user_stream/remove_user_to_monitor_tweet` |
| List monitors | GET | `/oapi/x_user_stream/get_user_to_monitor_tweet` |
| Add webhook filter rule | POST | `/oapi/tweet_filter/add_rule` |
| Get filter rules | GET | `/oapi/tweet_filter/get_rules` |
| Delete filter rule | DELETE | `/oapi/tweet_filter/delete_rule` |

Writes (all POST unless noted; require `login_cookie` in body):

| Capability | Method | Path |
|---|---|---|
| Log in (get cookie) | POST | `/twitter/user_login_v2` |
| Create / reply / quote tweet | POST | `/twitter/create_tweet_v2` |
| Delete tweet | POST | `/twitter/delete_tweet_v2` |
| Like / Unlike (separate endpoints) | POST | `/twitter/like_tweet_v2` · `/twitter/unlike_tweet_v2` |
| Retweet | POST | `/twitter/retweet_tweet_v2` |
| Bookmark / Unbookmark | POST | `/twitter/bookmark_tweet_v2` · `/twitter/unbookmark_tweet_v2` |
| Follow / Unfollow | POST | `/twitter/follow_user_v2` · `/twitter/unfollow_user_v2` |
| Send DM | POST | `/twitter/send_dm_to_user` |
| Upload media | POST | `/twitter/upload_media_v2` |
| Update profile / avatar / banner | **PATCH** | `/twitter/update_profile_v2` etc. |
| Create / join / leave / delete community | POST | `/twitter/{create,join,leave,delete}_community_v2` |

For **every endpoint's params in detail**, see [references/endpoints.md](references/endpoints.md).
For the **login_cookie flow** and full write-endpoint payloads, see [references/write-operations.md](references/write-operations.md).
For **longer runnable examples**, see [references/examples.md](references/examples.md).

## Recurring patterns

### 1. Cursor pagination (use `has_next_page`, not empty-cursor check)

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
        cursor = r.get("next_cursor", "")
```

### 2. Error handling

- `401` → missing/invalid `x-api-key`, or (for writes) expired `login_cookie`
- `402` → account balance too low (top up at the dashboard)
- `429` → rate-limited; back off exponentially
- `5xx` → transient; retry with jitter up to 3 times

Always check `response.status_code` before calling `.json()`, and surface the body's `message` field in errors.

### 3. Cost awareness

Before running a loop that could paginate indefinitely (followers of a mega-account, advanced search over years), **estimate cost up front** using the pricing table and confirm with the user if it could exceed a few dollars. Individual reads are cheap, but a full follower crawl of a 100M-follower account is $15k+.

## Anti-patterns to avoid

- Do **not** use `developer.twitter.com` / official X OAuth endpoints — this API replaces them
- Do **not** put the API key in URL query strings or log it
- Do **not** assume snake_case parameter names — the API uses camelCase (`tweetId`, `userName`, `pageSize`, `userId`)
- Do **not** rely on `next_cursor` being an empty string for termination — use `has_next_page === false`
- Do **not** assume write endpoints accept DELETE — they're all POST to separate `_v2` paths (e.g. `unlike_tweet_v2`, `unfollow_user_v2`)
- Do **not** assume write endpoints work with just the API key — they need a `login_cookie` from `/twitter/user_login_v2` (requires the account's email/username + password + optional 2FA base32 seed)
- Do **not** poll a user's timeline in a hot loop for "real-time" — use `/oapi/tweet_filter/*` filter rules or `/oapi/x_user_stream/*` monitors instead
