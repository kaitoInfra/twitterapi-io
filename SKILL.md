---
name: twitterapi-io
description: Query Twitter/X data (tweets, profiles, followers, advanced search, trends, spaces, communities) and perform authenticated actions (post, reply, like, retweet, follow, DM) via the twitterapi.io REST API using a single `x-api-key` header — no OAuth. Use when the user needs to scrape, analyze, monitor, or automate X/Twitter without going through the official developer portal.
---

# twitterapi.io

A skill for working with [twitterapi.io](https://twitterapi.io) — a pay-per-request REST API that returns Twitter/X data (tweets, users, timelines, search, trends, spaces, communities) and performs write actions on behalf of a logged-in account.

## When to use this skill

Trigger when the user wants any of:

- Fetch tweets, user profiles, followers/following, trends, replies, quote-tweets, retweeters
- Advanced tweet search (keywords, date ranges, filters)
- Monitor specific users or filter rules in real time
- Post / delete / like / retweet / bookmark / reply to tweets
- Follow / unfollow / send DM / update profile
- Anything involving `twitterapi.io`, `api.twitterapi.io`, or `x-api-key` in their code or questions

## Core facts (memorize these)

| Thing | Value |
|---|---|
| Base URL | `https://api.twitterapi.io` |
| Path prefix | `/twitter/...` |
| Auth header | `x-api-key: YOUR_KEY` |
| Dashboard (get key) | https://twitterapi.io/dashboard |
| Docs | https://docs.twitterapi.io |
| Auth model | API key for **reads**; API key **+ `login_cookie`** for **writes** |
| Pagination | Cursor-based via `cursor` query param; response includes `next_cursor` |
| Rate limit | Up to ~200 req/s per client |
| Pricing | ~$0.15 / 1k tweets, ~$0.18 / 1k profiles, $0.00015 minimum per request |

**Never** hardcode the API key. Read from `TWITTERAPI_IO_KEY` env var. If missing, ask the user or prompt them to set it.

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
| User by username | GET | `/twitter/user/info?userName=` |
| Batch users by IDs | POST | `/twitter/user/batch_info_by_ids` |
| Search users | GET | `/twitter/user/search?keyword=` |
| User's recent tweets | GET | `/twitter/user/last_tweets?userName=` |
| User followers | GET | `/twitter/user/followers?userName=&cursor=` |
| User following | GET | `/twitter/user/followings?userName=&cursor=` |
| Tweets by IDs | GET | `/twitter/tweets?tweet_ids=` |
| Tweet replies | GET | `/twitter/tweet/replies?tweet_id=&cursor=` |
| Quote tweets | GET | `/twitter/tweet/quotations?tweet_id=&cursor=` |
| Retweeters | GET | `/twitter/tweet/retweeters?tweet_id=&cursor=` |
| Thread context | GET | `/twitter/tweet/thread_context?tweet_id=` |
| Advanced search | GET | `/twitter/tweet/advanced_search?query=&cursor=` |
| Trends | GET | `/twitter/trends?woeid=` |
| Community info | GET | `/twitter/community/info?community_id=` |
| Space detail | GET | `/twitter/space/info?space_id=` |

Writes (require `login_cookie` obtained via `/twitter/user_login_v2`):

| Capability | Method | Path |
|---|---|---|
| Create tweet / reply | POST | `/twitter/create_tweet_v2` |
| Delete tweet | DELETE | `/twitter/delete_tweet` |
| Like / unlike | POST/DELETE | `/twitter/like_tweet` |
| Retweet | POST | `/twitter/retweet_tweet` |
| Bookmark / unbookmark | POST/DELETE | `/twitter/bookmark_tweet` |
| Follow / unfollow | POST/DELETE | `/twitter/follow_user` |
| Send DM | POST | `/twitter/send_dm_v2` |
| Update profile/avatar/banner | PUT/POST | `/twitter/profile/*` |

For the **full endpoint catalog** with every parameter, see [references/endpoints.md](references/endpoints.md).
For the **login_cookie flow** and write-action details, see [references/write-operations.md](references/write-operations.md).
For **longer runnable examples** (pagination loop, advanced search, real-time monitor), see [references/examples.md](references/examples.md).

## Recurring patterns

### 1. Cursor pagination

Every list endpoint returns `next_cursor` (empty string or absent when done). Pass it back as `cursor` to get the next page.

```python
def iter_followers(user_name):
    cursor = ""
    while True:
        r = requests.get(f"{BASE}/twitter/user/followers",
                         headers=HEADERS,
                         params={"userName": user_name, "cursor": cursor},
                         timeout=30).json()
        yield from r.get("followers", [])
        cursor = r.get("next_cursor") or ""
        if not cursor:
            break
```

### 2. Error handling

- `401` → missing/invalid `x-api-key`
- `402` → account balance too low (top up at dashboard)
- `429` → rate-limited; back off exponentially
- `5xx` → transient; retry with jitter up to 3 times

Always check `response.status_code` before calling `.json()`, and surface the response body in the error message — twitterapi.io returns a JSON `msg` field explaining the failure.

### 3. Cost awareness

Before running a loop that could paginate indefinitely (followers of a mega-account, advanced search over years), **estimate cost up front** using the pricing table and confirm with the user if it could exceed a few dollars. Reads are cheap individually but a full follower crawl of 100M-follower accounts is $15k+.

## Anti-patterns to avoid

- Do **not** use the official Twitter OAuth or `developer.twitter.com` endpoints — this API is an alternative that replaces them
- Do **not** put the API key in URL query strings or log it
- Do **not** forget to handle `next_cursor` being an empty string vs. absent — both mean "end of results"
- Do **not** assume write endpoints work with just the API key; they require a `login_cookie` obtained through `/twitter/user_login_v2`, which needs the account's email + password + optional 2FA secret
- Do **not** poll a user's timeline in a hot loop for "real-time" updates — use the **filter rules / websocket** endpoints instead (see references/endpoints.md)
