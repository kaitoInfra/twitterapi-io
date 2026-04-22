# twitterapi.io — Full endpoint catalog

All endpoints are prefixed with `https://api.twitterapi.io` and require header `x-api-key: YOUR_KEY`.

All paths, parameters, and body fields below were verified against the live backend source (`main.py`). Where the same action has multiple versions (e.g. v2 and v3), both are listed — they coexist and serve different use cases.

---

## Important conventions

> ⚠️ **Parameter naming is per-endpoint.** There is no universal rule. The same concept may be `userName` on one route and `user_id` on another, `tweetId` here and `tweet_id` there. **Copy the exact name from the table below — do not normalize.**

> ⚠️ **Two write-operation patterns** — v2 (cookie-based, client owns the session) and v3 (bot-account, server holds the session). See [write-operations.md](write-operations.md) for the decision matrix. v3 is simpler for most use cases.

> ⚠️ **Response shape varies per endpoint.** Some wrap in `data: {...}`, some spread fields at top level (`tweets[], has_next_page, next_cursor`), some have a named top-level key (`community_info`, `users`). Check the "Response" column or a live response — don't assume a universal envelope.

**Error format**:
- HTTP 400 / 422 / 500: `{"detail": "<reason>"}` (FastAPI style)
- HTTP 200 but semantic failure: `{"status": "error", "msg": "..."}`

---

## Authentication & Account

| Path | Method | Body / Params | Notes |
|---|---|---|---|
| `/twitter/user_login_v2` | POST | body: `user_name`, `email`, `password`, `proxy` (all **required**); optional `totp_secret` | Returns `login_cookies` (base64-encoded JSON of a cookie dict) for v2 writes |
| `/twitter/user_login_v3` | POST | body: `user_name`, `proxy`, plus **either** `cookie` (format `k=v&k=v`) **or** `password` (+ optional `email`, `totp_code`) | Registers a "bot account" stored server-side; later v3 calls reference it by `user_name` |
| `/twitter/login_by_email_or_username` | POST | body: `username_or_email`, `password`, `proxy` | Legacy v1 — returns `auth_session` (older format) |
| `/twitter/login_by_2fa` | POST | 2FA verification flow | |
| `/twitter/logout` | POST | | |
| `/twitter/get_my_x_account_detail_v3` | GET | body: `user_name` (unusual: GET reads body) | For v3 bot accounts |
| `/twitter/delete_my_x_account_v3` | DELETE | body: `user_name` | |
| `/twitter/account_v3` | DELETE | body: `user_name` | Alias for delete_my_x_account_v3 |
| `/oapi/my/info` | GET | — | Returns your API-key account balance: `{recharge_credits, total_bonus_credits}` |

---

## User endpoints (read)

| Path | Method | Exact params | Response top-level |
|---|---|---|---|
| `/twitter/user/info` | GET | `userName` (req) + optional `is_new` | `status, msg, data{id, name, userName, followers, following, favouritesCount, statusesCount, isBlueVerified, createdAt, ...}` |
| `/twitter/user/{username}` | GET | path `{username}` + optional query `is_new` | Same as /user/info |
| `/twitter/user_about` | GET | `userName` | `status, msg, data{...}` |
| `/twitter/user/batch_info_by_ids` | GET | `userIds` (comma-separated, req) | `status, msg, users[]` |
| `/twitter/user/search` | GET | `query` (req, **not** `keyword`) + optional `cursor` | `users[], has_next_page, next_cursor` |
| `/twitter/user/last_tweets` | GET | `userName` **or** `userId` (one req) + optional `cursor`, `includeReplies` (bool, default false) | |
| `/twitter/user/last_tweets/v2` | GET | `userId` (req, must be digits) + optional `cursor` | |
| `/twitter/user/tweet_timeline` | GET | `userId` (req) + optional `cursor`, `includeReplies`, `includeParentTweet` | |
| `/twitter/user/articles` | GET | **`username`** (req, all lowercase!) + optional `cursor` | |
| `/twitter/user/mentions` | GET | `userName` (req) + optional `sinceTime`, `untilTime`, `cursor`, `queryType` (default "Latest") | `tweets[], has_next_page, next_cursor, status, msg` |
| `/twitter/user/followers` | GET | `userName` (req) + optional `cursor`, `pageSize` (default 200, clamped 20–200) | `followers[], has_next_page, next_cursor, status, msg, code` |
| `/twitter/user/verifiedFollowers` | GET | **`user_id`** (snake, req, must be digits) + optional `cursor` | `followers[], has_next_page, next_cursor, status, msg, code` |
| `/twitter/user/followings` | GET | `userName` (req) + optional `cursor`, `pageSize` (20–200) | `followings[], has_next_page, next_cursor` |
| `/twitter/user/check_follow_relationship` | GET | `source_user_name` (snake, req), `target_user_name` (snake, req) | `status, message, data{following, followed_by}` — note **`message`**, not `msg` |

---

## Tweet endpoints (read)

| Path | Method | Exact params | Response |
|---|---|---|---|
| `/twitter/tweets` | GET | **`tweet_ids`** (snake, comma-separated) + optional `use_new` | `tweets[], status, msg, code` |
| `/twitter/tweet/replies` | GET | `tweetId` (req) + optional `sinceTime`, `untilTime`, `cursor`, `queryType` (default "Latest") | `tweets[], has_next_page, next_cursor, status, msg` |
| `/twitter/tweet/replies/v1` | GET | `tweetId` + `cursor` | Alternate backend; same shape |
| `/twitter/tweet/replies/v2` | GET | `tweetId` (req) + optional `cursor`, **`queryType`** (must be `Relevance` / `Latest` / `Likes`) | |
| `/twitter/tweet/quotes` | GET | `tweetId` (req) + optional `cursor` | `tweets[], has_next_page, next_cursor, status, msg` |
| `/twitter/tweet/retweeters` | GET | `tweetId` (req) + optional `cursor` | `users[], has_next_page, next_cursor` (no status wrapper) |
| `/twitter/tweet/thread_context` | GET | `tweetId` (req) + optional `cursor` | `tweets[], has_next_page, next_cursor` |
| `/twitter/article` | GET | **`tweet_id`** (snake, req) | `{article, status, msg}` |
| `/twitter/tweet/advanced_search` | GET | `query` (req) + optional `queryType` (default "Latest"), `cursor`, `provider` | `{tweets[], has_next_page, next_cursor}` (no status wrapper) |
| `/twitter/tweet/bulk_advanced_search` | POST | body: `{queries: [{query, queryType?, cursor?}]}` | `{results: {query_0: {...}, query_1: {...}}, status, msg}` — parallel dispatch |

---

## Trends / Spaces / Lists

| Path | Method | Exact params | Notes |
|---|---|---|---|
| `/twitter/trends` | GET | `woeid` (req, digits) + optional `count` (default 30) | 1=worldwide, 23424977=US |
| `/twitter/spaces/detail` | GET | **`space_id`** (snake, req) | |
| `/twitter/list/tweets` | GET | `listId` (req) + optional `sinceTime`, `untilTime`, `includeReplies` (default "true"), `cursor`, `queryType` | |
| `/twitter/list/tweets_timeline` | GET | **`listId`** (camel, req) + optional `cursor` | |
| `/twitter/list/members` | GET | **`list_id`** (snake, req) + optional `cursor` | Returns `members[]` |
| `/twitter/list/followers` | GET | **`list_id`** (snake, req) + optional `cursor` | Returns `followers[]` |

---

## Community endpoints (read)

| Path | Method | Exact params | Response |
|---|---|---|---|
| `/twitter/community/info` | GET | `community_id` (snake, req) | `{community_info: {...}, status, msg}` |
| `/twitter/community/tweets` | GET | `community_id` (req) + optional `cursor` | `{tweets, has_next_page, next_cursor}` |
| `/twitter/community/members` | GET | `community_id` (req) + optional `cursor` | `members[]` (cursor is base64-encoded) |
| `/twitter/community/moderators` | GET | `community_id` (req) + optional `cursor` | `moderators[]` (cursor is base64-encoded) |
| `/twitter/community/get_tweets_from_all_community` | GET | **`query` (required!)** + optional `queryType`, `cursor` | `{tweets, has_next_page, next_cursor}` |

---

## Real-time user monitoring — `/oapi/x_user_stream/*`

Requires an **active monitoring subscription**; returns `{status: "error", msg: "No active monitoring subscription"}` if not.

**Note the unusual field names** — each endpoint uses a different name for the same concept:

| Path | Method | Body / Params | Notes |
|---|---|---|---|
| `/oapi/x_user_stream/add_user_to_monitor_tweet` | POST | body: `{x_user_name}` | **Field is `x_user_name`**, NOT `user_id`. Leading `@` stripped. |
| `/oapi/x_user_stream/get_user_to_monitor_tweet` | GET | query: optional `query_type` (default 1; 0=all, 1=tweet, 2=profile) | Returns `{status, msg, data: [{id_for_user, x_user_id, x_user_name, x_user_screen_name, is_monitor_tweet, is_monitor_profile}]}` |
| `/oapi/x_user_stream/remove_user_to_monitor_tweet` | POST | body: `{id_for_user}` | **Field is `id_for_user`** (from the list response), NOT `user_id` or `x_user_name` |
| `/oapi/x_user_stream/get_user_monitor_account` | GET | query: optional `query_type` | |

---

## Webhook filter rules — `/oapi/tweet_filter/*`

| Path | Method | Body | Notes |
|---|---|---|---|
| `/oapi/tweet_filter/add_rule` | POST | `{tag, value, interval_seconds?=60}` | `tag` ≤255 chars, `value` ≤255 chars, `interval_seconds` range 0.05–86400. Returns `rule_id` |
| `/oapi/tweet_filter/get_rules` | GET | — | Returns `{rules: [{rule_id, user_id, tag, value, interval_seconds, is_effect, is_delete, created_at}]}` |
| `/oapi/tweet_filter/update_rule` | POST | `{rule_id, tag, value, interval_seconds?, is_effect?}` | Only update takes `is_effect` (0 or 1) |
| `/oapi/tweet_filter/delete_rule` | DELETE | **body**: `{rule_id}` | Body JSON, NOT query string |

---

## v2 write endpoints — cookie-based (require `login_cookies` + `proxy` in body)

`login_cookies` is returned from `/twitter/user_login_v2` as a base64-encoded JSON string of the cookie dict. Pass it back verbatim; the server base64-decodes and parses it.

| Capability | Method | Path | Body fields |
|---|---|---|---|
| Create / reply / quote tweet | POST | `/twitter/create_tweet_v2` | `login_cookies`, `proxy`, `tweet_text` (req); optional `reply_to_tweet_id`, `quote_tweet_id`, `community_id`, `media_ids[]`, `attachment_url`, `is_note_tweet`, `schedule_for` (format `2026-01-20T10:00:00.000Z`) |
| Delete tweet | POST | `/twitter/delete_tweet_v2` | `login_cookies`, `proxy`, `tweet_id` |
| Like | POST | `/twitter/like_tweet_v2` | `login_cookies`, `proxy`, `tweet_id` |
| Unlike | POST | `/twitter/unlike_tweet_v2` | `login_cookies`, `proxy`, `tweet_id` |
| Retweet | POST | `/twitter/retweet_tweet_v2` | `login_cookies`, `proxy`, `tweet_id` |
| Bookmark | POST | `/twitter/bookmark_tweet_v2` | `login_cookies`, `proxy`, `tweet_id` |
| Unbookmark | POST | `/twitter/unbookmark_tweet_v2` | `login_cookies`, `proxy`, `tweet_id` |
| List own bookmarks | POST | `/twitter/bookmarks_v2` | `login_cookies`, `proxy`; optional **`count`** (default 20, **not `pageSize`**), `cursor` |
| Follow | POST | `/twitter/follow_user_v2` | `login_cookies`, `proxy`, `user_id` |
| Unfollow | POST | `/twitter/unfollow_user_v2` | `login_cookies`, `proxy`, `user_id` |
| Report tweet / user | POST | `/twitter/report_v2` | `login_cookies`, `proxy`, `tweet_id` **or** `user_id`, `reason` (enum of 12 — see below) |
| Send DM | POST | `/twitter/send_dm_to_user` | `login_cookies`, `proxy`, `user_id`, `text`; optional `media_id` (singular), `reply_to_message_id` |
| DM history | GET | `/twitter/get_dm_history_by_user_id` | query params `login_cookies`, `user_id`, `proxy` (note: credentials in query string, unusual) |
| Upload media | POST | `/twitter/upload_media_v2` | **multipart/form-data**: `file` (required), `login_cookies`, `proxy`; optional `media_category` (auto-set to `tweet_video` for video), `is_long_video` |
| Update profile | **PATCH** | `/twitter/update_profile_v2` | JSON: `login_cookies`, `proxy`; at least one of `name` (≤50), **`description`** (≤160, **not `bio`**), `location` (≤30), `url` |
| Update avatar | **PATCH** | `/twitter/update_avatar_v2` | **multipart**: `file` (JPG/PNG, ≤700KB, rec. 400×400), `login_cookies`, `proxy` |
| Update banner | **PATCH** | `/twitter/update_banner_v2` | **multipart**: `file` (JPG/PNG, ≤2MB, rec. 1500×500), `login_cookies`, `proxy` |
| Create community | POST | `/twitter/create_community_v2` | `login_cookies`, `proxy`, `name`, `description` (**required**) |
| Join community | POST | `/twitter/join_community_v2` | `login_cookies`, `proxy`, `community_id` |
| Leave community | POST | `/twitter/leave_community_v2` | `login_cookies`, `proxy`, `community_id` |
| Delete community | POST | `/twitter/delete_community_v2` | `login_cookies`, `proxy`, `community_id`, **`community_name`** (also required) |
| Add list member | POST | `/twitter/list/add_member_v2` | `login_cookies`, `proxy`, `list_id`, `user_id` |

**Report reasons enum** (for `/twitter/report_v2`):
`SpamSimpleOption`, `HateOrAbuseSimpleOption`, `ChildSafetySimpleOption`, `ViolentSpeechSimpleOption`, `ViolentMediaSimpleOption`, `IRBSimpleOption`, `ImpersonationSimpleOption`, `AdultContentSimpleOption`, `PrivateContentSimpleOption`, `SuicideSelfHarmSimpleOption`, `TerrorismSimpleOption`, `CivicIntegritySimpleOption`.

---

## v3 write endpoints — bot-account API (simpler, no per-call cookies/proxy)

After registering a bot via `/twitter/user_login_v3`, subsequent calls only need the `user_name` of the bot account. Server handles cookies/proxy.

| Capability | Method | Path | Body fields |
|---|---|---|---|
| Send tweet | POST | `/twitter/send_tweet_v3` | `user_name` (bot), `text`; optional `community_id`, `media_data_base64`, `media_type` |
| Reply | POST | `/twitter/reply_tweet_v3` | `user_name`, `text`, `reply_to_tweet_id` |
| Quote | POST | `/twitter/quote_tweet_v3` | `user_name`, `text`, `tweet_id`, `tweet_username` (author of the quoted tweet, required) |
| Like | POST | `/twitter/like_tweet_v3` | `user_name`, `tweet_id` |
| Retweet | POST | `/twitter/retweet_v3` | `user_name`, `tweet_id` |
| Follow | POST | `/twitter/follow_v3` | `user_name`, **either** `target_user_id` **or** `target_user_name` |
| Update profile | **PUT** | `/twitter/update_profile_v3` | `user_name`; optional `name`, **`bio`** (note v3 uses `bio`, v2 uses `description`), `location`, `website`, `avatar` (base64), `banner` (base64) |

v3 write endpoints return the underlying action result (shape depends on the bot-layer implementation).

---

## v1 legacy write endpoints (use `auth_session` — prefer v2 or v3 instead)

| Path | Method | Body |
|---|---|---|
| `/twitter/create_tweet` | POST | `auth_session`, `proxy`, `tweet_text`; optional `quote_tweet_id`, `in_reply_to_tweet_id`, `media_id` |
| `/twitter/like_tweet` | POST | `auth_session`, `proxy`, `tweet_id` |
| `/twitter/retweet_tweet` | POST | `auth_session`, `proxy`, `tweet_id` |
| `/twitter/list/create` | POST | `auth_session`, `proxy`, `name`; optional `description`, `private` |
| `/twitter/list/add_member` | POST | `auth_session`, `proxy`, `list_id`; `user_id` **or** `user_name` |
| `/twitter/list/remove_member` | POST | same as add_member |
| `/twitter/upload_image` | POST | | 
| `/twitter/upload_video` | POST | | 

---

## Pagination

List endpoints return both `next_cursor` (string) and `has_next_page` (bool). **Terminate the loop on `has_next_page === false`** — more reliable than checking for empty-string / null cursor.

```python
cursor = ""
while True:
    r = requests.get(url, params={..., "cursor": cursor}, headers=HEADERS).json()
    yield from r.get("items", [])
    if not r.get("has_next_page"):
        break
    cursor = r.get("next_cursor") or ""
```

## Response envelope variants

Don't hardcode one path. Prefer `r.get("tweets", r.get("data", {}).get("tweets", []))`.

```json
// 1. data-wrapped — user/info, user_about, last_tweets, tweet_timeline, trends, check_follow, last_tweets/v2
{ "status": "success", "msg": "success", "data": { ... } }

// 2. flat-list with envelope — followers, followings, replies, mentions, list/tweets_timeline, community/tweets
{ "tweets": [...], "has_next_page": true, "next_cursor": "...", "status": "success", "msg": "success" }

// 3. flat-list WITHOUT envelope — advanced_search, thread_context, user/search, get_tweets_from_all_community
{ "tweets": [...], "has_next_page": true, "next_cursor": "..." }

// 4. named top-level field — community/info, batch_info_by_ids, oapi/my/info
{ "status": "success", "msg": "success", "community_info": {...} }
{ "recharge_credits": 1000, "total_bonus_credits": 0 }
```

`check_follow_relationship` uniquely uses `message` instead of `msg`. Everything else uses `msg`.
