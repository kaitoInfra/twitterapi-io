---
name: twitterapi-io-read
description: Read-only skill for twitterapi.io — look up public X/Twitter data (tweets, user profiles, followers and followings, advanced search, replies, quotes, trends, lists, communities) through the twitterapi.io REST API with a single x-api-key header. Use when the user wants to search, analyze or monitor public posts and accounts. Read-only: no posting, no account login, no credentials beyond the API key.
---

# twitterapi.io (read-only)

Read-only skill maintained by the [twitterapi.io](https://twitterapi.io) team. It covers **public X/Twitter data only**: it never logs in to an X account, never handles cookies, passwords or proxies, and never posts, likes, follows or sends messages.

twitterapi.io is an independent third-party REST API for public X data, not affiliated with X Corp. If you need X's own API, the official one is at https://developer.x.com.

## When to use this skill

- Search posts with X search operators (`from:`, `since_time:`, `min_faves:`, `lang:` …)
- Look up user profiles, followers, followings
- Get a user's recent posts, replies, quote posts, retweeters, thread context
- Trends, lists, communities
- Build dashboards, research datasets or monitoring jobs on public data

## Core facts

| | |
|---|---|
| Base URL | `https://api.twitterapi.io` |
| Auth header | `x-api-key: YOUR_KEY` (from https://twitterapi.io/dashboard) |
| Docs | https://docs.twitterapi.io |
| Pricing | pay per call, see https://twitterapi.io/pricing |
| Hosted MCP server | `https://mcp.twitterapi.io/mcp` (12 read-only tools) |

## Security

Read the API key from the environment variable `TWITTERAPI_IO_KEY`. Never hardcode it, print it or commit it. If it is missing, ask the user to create one on the dashboard.

## Minimal example

```bash
curl -s "https://api.twitterapi.io/twitter/user/info?userName=jack" \
  -H "x-api-key: $TWITTERAPI_IO_KEY"
```

```python
import os, requests

BASE = "https://api.twitterapi.io"
HEADERS = {"x-api-key": os.environ["TWITTERAPI_IO_KEY"]}

r = requests.get(f"{BASE}/twitter/tweet/advanced_search",
                 headers=HEADERS,
                 params={"query": "from:jack", "queryType": "Latest"},
                 timeout=30)
r.raise_for_status()
for t in r.json().get("tweets", []):
    print(t["createdAt"], t["text"][:80])
```

## Read endpoints

Parameter names differ per endpoint (some camelCase, some snake_case). **Copy them exactly as shown.**

| Capability | Method | Path & key param |
|---|---|---|
| User by screen name | GET | `/twitter/user/info?userName=` |
| Extended profile | GET | `/twitter/user_about?userName=` |
| Users by IDs | GET | `/twitter/user/batch_info_by_ids?userIds=` |
| Search users | GET | `/twitter/user/search?query=` |
| Recent posts | GET | `/twitter/user/last_tweets?userName=` (or `userId=`) |
| Mentions | GET | `/twitter/user/mentions?userName=` |
| Followers | GET | `/twitter/user/followers?userName=&pageSize=200` |
| Followings | GET | `/twitter/user/followings?userName=` |
| Posts by IDs | GET | `/twitter/tweets?tweet_ids=` |
| Replies | GET | `/twitter/tweet/replies?tweetId=` |
| Quote posts | GET | `/twitter/tweet/quotes?tweetId=` |
| Retweeters | GET | `/twitter/tweet/retweeters?tweetId=` |
| Thread context | GET | `/twitter/tweet/thread_context?tweetId=` |
| Advanced search | GET | `/twitter/tweet/advanced_search?query=&queryType=Latest` |
| Trends | GET | `/twitter/trends?woeid=1` |
| List posts | GET | `/twitter/list/tweets?listId=` |
| List members | GET | `/twitter/list/members?list_id=` |
| Community posts | GET | `/twitter/community/tweets?community_id=` |
| Account balance | GET | `/oapi/my/info` |

## Patterns

### Pagination

List endpoints return `has_next_page` and `next_cursor`. Pass `next_cursor` back as `cursor`; stop when `has_next_page` is false. Always set a page limit so a bug can't page forever.

### Errors

- `400` → a required parameter is missing or the search query is invalid (the response `detail` says what to fix)
- `401` / `403` → missing or wrong `x-api-key`
- `402` → out of credits; stop and tell the user to top up (don't retry)
- `429` → rate-limited; retry with backoff
- `5xx` → transient; retry with backoff

### Cost awareness

Each call is billed. Before a loop that could run long (all followers of a large account, months of search), estimate the number of calls and confirm with the user.

## Anti-patterns

- Don't normalise parameter names; copy them from the table
- Don't retry on `402`
- Don't log or commit the API key
