# twitterapi.io — Full endpoint catalog

All endpoints are prefixed with `https://api.twitterapi.io` and require header `x-api-key: YOUR_KEY` (case-insensitive; docs write it as `X-API-Key`). Write endpoints additionally require a `login_cookie` field in the JSON body (see [write-operations.md](write-operations.md)).

> ⚠️ **Verify parameter names per endpoint.** twitterapi.io uses a mix of `camelCase` (`userName`, `tweetId`, `pageSize`, `userId`) and sometimes snake_case — each endpoint's canonical doc page at https://docs.twitterapi.io/api-reference/endpoint/... is authoritative if anything below disagrees with live API behavior.

---

## Authentication

### POST `/twitter/user_login_v2`
Exchange account credentials for a `login_cookie` used by all write endpoints.
- Body: `{ "email_or_username": "...", "password": "...", "totp_secret": "OPTIONAL_2FA_SEED" }`
- Returns: `{ "login_cookie": "...", "user_id": "...", "screen_name": "..." }`

### GET `/oapi/my/info`
Retrieve profile of the API-key owner (the twitterapi.io account, not a Twitter account).

---

## User endpoints (read)

### GET `/twitter/user/info`
Get a user profile by screen name.
- `userName` (required) — e.g. `elonmusk`

### GET `/twitter/user_about`
Extended bio / "about" block (bio, location, URL, join date).
- `userName` (required)

### GET `/twitter/user/batch_info_by_ids`
Fetch multiple users by ID in one call.
- `userIds` (required) — comma-separated or array

### GET `/twitter/user/search`
Search users by keyword.
- `keyword` (required)

### GET `/twitter/user/followers`
Followers of a user. 200 per page, sorted by follow date (most recent first).
- `userName` (required)
- `cursor` (optional) — empty string for first page
- `pageSize` (optional) — default 200, range 20–200
- Returns: `{ followers[], next_cursor, has_next_page }`

### GET `/twitter/user/verifiedFollowers`
Only verified followers. 20 per page.
- `userName` (required)
- `cursor` (optional)

### GET `/twitter/user/followings`
Who a user follows. 200 per page, sorted by follow date.
- `userName` (required)
- `cursor` (optional)
- `pageSize` (optional) — default 200, range 20–200

### GET `/twitter/user/check_follow_relationship`
Does source follow target?
- `sourceId` (required)
- `targetId` (required)

### GET `/twitter/user/mentions`
Tweets that mention a user.
- `userName` (required)
- `cursor` (optional)

---

## Tweet endpoints (read)

### GET `/twitter/tweets`
Fetch multiple tweets by ID.
- `tweetIds` (required) — comma-separated or array

### GET `/twitter/user/last_tweets`
A user's recent tweets (by screen name).
- `userName` (required)
- `cursor` (optional)

### GET `/twitter/user/tweet_timeline`
Timeline by user ID, sorted by `created_at`.
- `userId` (required)
- `cursor` (optional)

### GET `/twitter/tweet/replies`
Replies under a tweet. ~20 per page.
- `tweetId` (required)
- `cursor` (optional)

### GET `/twitter/tweet/replies/v2`
Replies with sort options (e.g. top vs. latest).
- `tweetId` (required)
- `sort` (optional)
- `cursor` (optional)

### GET `/twitter/tweet/quotes`
Quote-tweets of a tweet. 20 per page.
- `tweetId` (required)
- `cursor` (optional)

### GET `/twitter/tweet/retweeters`
Users who retweeted a tweet. ~100 per page.
- `tweetId` (required)
- `cursor` (optional)

### GET `/twitter/tweet/thread_context`
Full thread a tweet belongs to.
- `tweetId` (required)
- `cursor` (optional)

### GET `/twitter/article`
Long-form "Article" content for a tweet (note: ~100 credits per call).
- `tweetId` (required)

### GET `/twitter/tweet/advanced_search`
Full-text search with the same query operators as X's advanced-search UI.
- `query` (required) — e.g. `from:elonmusk since:2025-01-01 until:2025-02-01 min_faves:1000`
- `filters` (optional)
- `cursor` (optional)

---

## Trends & Spaces & Lists

### GET `/twitter/trends`
Trending topics by region.
- `woeid` (required) — Yahoo "Where On Earth" ID (1 = worldwide, 23424977 = US)

### GET `/twitter/spaces/detail`
X Space (audio room) metadata.
- `spaceId` (required)

### GET `/twitter/list/tweets_timeline`
Tweets from a curated list.
- `listId` (required)
- `cursor` (optional)

### GET `/twitter/list/members`
- `listId` (required)
- `cursor` (optional)

### GET `/twitter/list/followers`
- `listId` (required)
- `cursor` (optional)

---

## Community endpoints (read)

### GET `/twitter/community/info`
- `communityId` (required)

### GET `/twitter/community/tweets`
- `communityId` (required)
- `cursor` (optional)

### GET `/twitter/community/get_tweets_from_all_community`
All-communities firehose. 20 per page.
- `cursor` (optional)

### GET `/twitter/community/members`
- `communityId` (required)
- `cursor` (optional)

### GET `/twitter/community/moderators`
- `communityId` (required)
- `cursor` (optional)

---

## Real-time monitoring — note the `/oapi/x_user_stream/` prefix (not `/twitter/`)

### POST `/oapi/x_user_stream/add_user_to_monitor_tweet`
Track a user's new tweets in near real-time.
- Body: `{ "userId": "..." }`

### GET `/oapi/x_user_stream/get_user_to_monitor_tweet`
List currently monitored accounts.

### POST `/oapi/x_user_stream/remove_user_to_monitor_tweet`
- Body: `{ "userId": "..." }`

---

## Webhook filter rules — note the `/oapi/tweet_filter/` prefix

Define server-side rules; matching tweets are delivered to your configured webhook / websocket.

### POST `/oapi/tweet_filter/add_rule`
- Body: `{ "rule_text": "from:elonmusk OR #AI", "active": true }`

### GET `/oapi/tweet_filter/get_rules`
List all rules.

### POST `/oapi/tweet_filter/update_rule`
- Body: `{ "ruleId": "...", "rule_text": "...", "active": true }`

### DELETE `/oapi/tweet_filter/delete_rule`
- Body / param: `{ "ruleId": "..." }`

---

## Write endpoints (all POST unless noted; all require `login_cookie` in body)

See [write-operations.md](write-operations.md) for the login flow and example bodies.

| Capability | Method | Path |
|---|---|---|
| Create / reply / quote tweet | POST | `/twitter/create_tweet_v2` |
| Delete tweet | POST | `/twitter/delete_tweet_v2` |
| Like | POST | `/twitter/like_tweet_v2` |
| Unlike | POST | `/twitter/unlike_tweet_v2` |
| Retweet | POST | `/twitter/retweet_tweet_v2` |
| Bookmark | POST | `/twitter/bookmark_tweet_v2` |
| Unbookmark | POST | `/twitter/unbookmark_tweet_v2` |
| Get bookmarks | POST | `/twitter/bookmarks_v2` |
| Follow | POST | `/twitter/follow_user_v2` |
| Unfollow | POST | `/twitter/unfollow_user_v2` |
| Send DM | POST | `/twitter/send_dm_to_user` |
| Upload media | POST | `/twitter/upload_media_v2` |
| Update profile text | PATCH | `/twitter/update_profile_v2` |
| Update avatar | PATCH | `/twitter/update_avatar_v2` |
| Update banner | PATCH | `/twitter/update_banner_v2` |
| Create community | POST | `/twitter/create_community_v2` |
| Delete community | POST | `/twitter/delete_community_v2` |
| Join community | POST | `/twitter/join_community_v2` |
| Leave community | POST | `/twitter/leave_community_v2` |

---

## Response shape

Most endpoints return:

```json
{
  "status": "success",
  "code": 0,
  "message": "",
  "data": { ... },
  "next_cursor": "abc123",
  "has_next_page": true
}
```

**Terminate pagination on `has_next_page === false`** — it's more reliable than checking `next_cursor` for empty string.

Errors:

```json
{ "status": "error", "code": 401, "message": "invalid api key" }
```

Always check HTTP status *and* the body's `status` / `code`.
