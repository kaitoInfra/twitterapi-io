# twitterapi.io — Full endpoint catalog

All endpoints are prefixed with `https://api.twitterapi.io` and require header `x-api-key: YOUR_KEY`. Write endpoints additionally require `login_cookies` **and** `proxy` fields in the JSON body (see [write-operations.md](write-operations.md)).

## Important conventions

> ⚠️ **Parameter naming is inconsistent across endpoints.** There is no universal camelCase-or-snake_case rule. Some endpoints use `tweetId`, others `tweet_id`; some use `userName`, others `user_id`. **The tables below reflect what the live API actually accepts** — copy the exact parameter name for the endpoint you're calling. Do not guess or apply a style convention.

> ⚠️ **Response shape is also endpoint-specific.** Some responses wrap payload in `data: {...}`, others spread fields at the top level, others return `{items[], has_next_page, next_cursor, status, msg}`. Check the "Response top-level keys" column or inspect a live response before parsing.

**Error format** (HTTP 400 / 422 / 500): `{"detail": "<human-readable reason>"}` or `{"detail": [{"type":"missing","loc":[...],"msg":"..."}]}` (FastAPI style).
**Semantic errors** (HTTP 200 but failed): `{"status":"error","msg":"..."}`.

---

## Authentication & account

| Path | Method | Params (exact) | Notes |
|---|---|---|---|
| `/twitter/user_login_v2` | POST | body: `user_name`, `email`, `password`, `proxy`, optional `totp_secret` | All snake_case. See write-operations.md |
| `/oapi/my/info` | GET | none | Returns API-key owner's balance: `{recharge_credits, total_bonus_credits}` |

---

## User endpoints (read)

| Path | Method | Exact param | Response top-level keys |
|---|---|---|---|
| `/twitter/user/info` | GET | `userName` | `status, msg, data{id,name,userName,followers,following,favouritesCount,statusesCount,isBlueVerified,...}` |
| `/twitter/user_about` | GET | `userName` | `status, msg, data{...extended bio}` |
| `/twitter/user/batch_info_by_ids` | GET | `userIds` (comma-separated) | `status, msg, users[]` |
| `/twitter/user/search` | GET | **`query`** (not `keyword`) | `users[]` |
| `/twitter/user/last_tweets` | GET | `userName`, optional `cursor` | `status, msg, data{tweets[], pin_tweet}` |
| `/twitter/user/tweet_timeline` | GET | `userId`, optional `cursor` | `status, msg, data{tweets[]}` |
| `/twitter/user/mentions` | GET | `userName`, optional `cursor` | `tweets[], has_next_page, next_cursor` |
| `/twitter/user/followers` | GET | `userName`, optional `cursor`, `pageSize` (default 200, range 20–200) | `followers[], has_next_page, next_cursor, status, msg, code` |
| `/twitter/user/verifiedFollowers` | GET | **`user_id`** (not `userName`!), optional `cursor` | `followers[]` |
| `/twitter/user/followings` | GET | `userName`, optional `cursor`, `pageSize` | `followings[], has_next_page, next_cursor` |
| `/twitter/user/check_follow_relationship` | GET | `source_user_name`, `target_user_name` | `status, message, data{following, followed_by}` |

Each follower/following object: `{id, name, screen_name, userName, location, url, description, followers_count, following_count, verified, ...}`.

---

## Tweet endpoints (read)

| Path | Method | Exact param | Response top-level keys |
|---|---|---|---|
| `/twitter/tweets` | GET | **`tweet_ids`** (comma-separated, snake_case) | `tweets[]` |
| `/twitter/tweet/replies` | GET | `tweetId`, optional `cursor` | `tweets[], has_next_page, next_cursor` |
| `/twitter/tweet/replies/v2` | GET | `tweetId`, `sort` (`Latest` / `Top`), optional `cursor` | `tweets[]` |
| `/twitter/tweet/quotes` | GET | `tweetId`, optional `cursor` | `tweets[]` |
| `/twitter/tweet/retweeters` | GET | `tweetId`, optional `cursor` | `users[]` (NOT `retweeters`) |
| `/twitter/tweet/thread_context` | GET | `tweetId`, optional `cursor` | `tweets[]` |
| `/twitter/article` | GET | **`tweet_id`** (snake_case) | `{article_content,...}` |
| `/twitter/tweet/advanced_search` | GET | `query` (X advanced-search operators), optional `cursor` | `tweets[], has_next_page, next_cursor` |

Each tweet object: `{type, id, url, twitterUrl, text, source, retweetCount, replyCount, likeCount, quoteCount, viewCount, createdAt, author{...}}`.

---

## Trends / Spaces / Lists

| Path | Method | Exact param | Notes |
|---|---|---|---|
| `/twitter/trends` | GET | `woeid` | 1=worldwide, 23424977=US. Returns `status, msg, trends[{trend:{name,target,rank}}]` |
| `/twitter/spaces/detail` | GET | **`space_id`** (snake) | X Space metadata |
| `/twitter/list/tweets_timeline` | GET | **`listId`** (camel) | `tweets[], has_next_page, next_cursor, status, msg` |
| `/twitter/list/members` | GET | **`list_id`** (snake) | `members[]` — note different casing from tweets_timeline! |
| `/twitter/list/followers` | GET | **`list_id`** (snake) | `followers[]` |

---

## Community endpoints

| Path | Method | Exact param | Response |
|---|---|---|---|
| `/twitter/community/info` | GET | `community_id` | `{community_info{id,name,description,question,...}}` |
| `/twitter/community/tweets` | GET | `community_id`, optional `cursor` | `tweets[]` |
| `/twitter/community/get_tweets_from_all_community` | GET | **`query` (required)**, optional `cursor` | `tweets[]` |
| `/twitter/community/members` | GET | `community_id`, optional `cursor` | `members[]` |
| `/twitter/community/moderators` | GET | `community_id`, optional `cursor` | `moderators[]` |

---

## Real-time monitoring — `/oapi/x_user_stream/*`

Requires an **active monitoring subscription** on your account; returns `{status:"error","msg":"No active monitoring subscription"}` if not.

| Path | Method | Body (JSON) | Notes |
|---|---|---|---|
| `/oapi/x_user_stream/add_user_to_monitor_tweet` | POST | `{"user_id":"..."}` | Body JSON, not query string |
| `/oapi/x_user_stream/get_user_to_monitor_tweet` | GET | — | Returns `data[{id_for_user, x_user_id, x_user_screen_name, is_monitor_tweet, is_monitor_profile}]` |
| `/oapi/x_user_stream/remove_user_to_monitor_tweet` | POST | `{"user_id":"..."}` | |

---

## Webhook filter rules — `/oapi/tweet_filter/*`

Define server-side rules; matching tweets stream to your webhook / websocket (configured on the dashboard).

| Path | Method | Body | Notes |
|---|---|---|---|
| `/oapi/tweet_filter/add_rule` | POST | `{"tag":"...","value":"<x-search-query>","interval_seconds":60,"is_effect":1}` | Fields: `tag`, `value`, `interval_seconds`, `is_effect`. Returns `rule_id` |
| `/oapi/tweet_filter/get_rules` | GET | — | Returns `rules[{rule_id, user_id, tag, value, is_delete, created_at,...}]` |
| `/oapi/tweet_filter/update_rule` | POST | `{"rule_id":"...", "tag":"...", "value":"...", ...}` | |
| `/oapi/tweet_filter/delete_rule` | DELETE | `{"rule_id":"..."}` | **Body JSON**, NOT query string (query string returns 500) |

---

## Write endpoints — see [write-operations.md](write-operations.md) for bodies

Every write requires **three** things in the body beyond the action-specific fields:
1. `login_cookies` (plural, from `/twitter/user_login_v2`)
2. `proxy` (proxy URL — configure in dashboard)
3. action-specific fields, typically **snake_case** (`tweet_id`, `user_id`)

| Capability | Method | Path |
|---|---|---|
| Create / reply / quote tweet | POST | `/twitter/create_tweet_v2` |
| Delete tweet | POST | `/twitter/delete_tweet_v2` |
| Like | POST | `/twitter/like_tweet_v2` |
| Unlike | POST | `/twitter/unlike_tweet_v2` |
| Retweet | POST | `/twitter/retweet_tweet_v2` |
| Bookmark | POST | `/twitter/bookmark_tweet_v2` |
| Unbookmark | POST | `/twitter/unbookmark_tweet_v2` |
| List own bookmarks | POST | `/twitter/bookmarks_v2` |
| Follow | POST | `/twitter/follow_user_v2` |
| Unfollow | POST | `/twitter/unfollow_user_v2` |
| Send DM | POST | `/twitter/send_dm_to_user` |
| Upload media | POST | `/twitter/upload_media_v2` (multipart, `file` field) |
| Update profile | PATCH | `/twitter/update_profile_v2` |
| Update avatar | PATCH | `/twitter/update_avatar_v2` |
| Update banner | PATCH | `/twitter/update_banner_v2` |
| Create community | POST | `/twitter/create_community_v2` |
| Delete community | POST | `/twitter/delete_community_v2` |
| Join community | POST | `/twitter/join_community_v2` |
| Leave community | POST | `/twitter/leave_community_v2` |

---

## Pagination

Both `next_cursor` and `has_next_page` appear in most list responses. **Terminate on `has_next_page === false`** — more reliable than the cursor string, which may be empty string, `null`, or the last valid value depending on endpoint.

```python
cursor = ""
while True:
    r = requests.get(url, params={..., "cursor": cursor}, headers=HEADERS).json()
    yield from r.get("items", [])
    if not r.get("has_next_page"):
        break
    cursor = r.get("next_cursor") or ""
```

## Response envelopes — three common shapes you'll see

```json
// 1. data-wrapped (user/info, user_about, last_tweets, tweet_timeline, trends, check_follow)
{ "status": "success", "msg": "success", "data": { ... } }

// 2. flat list (followers, followings, replies, quotes, advanced_search, mentions, community/tweets)
{ "tweets": [...], "has_next_page": true, "next_cursor": "..." }
// (may or may not also carry "status":"success","msg":"success","code":0)

// 3. named top-level field (community/info, batch_info_by_ids)
{ "status": "success", "msg": "success", "community_info": {...} }  // or "users": [...]
```

When writing parsing code, prefer `r.get("tweets", r.get("data", {}).get("tweets", []))` over hard-coding one path.
