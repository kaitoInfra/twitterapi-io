# twitterapi.io — Full endpoint catalog

All endpoints are prefixed with `https://api.twitterapi.io` and require header `x-api-key: YOUR_KEY`. Endpoints in the **Write** sections additionally require a `login_cookie` (see [write-operations.md](write-operations.md)).

Exact paths and parameters may evolve — when in doubt, check the authoritative reference at https://docs.twitterapi.io/api-reference.

---

## User endpoints

### GET `/twitter/user/info`
Get a user profile by screen name.
- `userName` (required) — e.g. `elonmusk`

### GET `/twitter/user/profile/about`
Extended profile "about" block (bio, location, URL, joined date).
- `userName` (required)

### POST `/twitter/user/batch_info_by_ids`
Fetch up to N users in one call.
- Body: `{ "userIds": ["44196397", "..."] }`

### GET `/twitter/user/search`
Search users by keyword.
- `keyword` (required)
- `cursor` (optional)

### GET `/twitter/user/followers`
- `userName` (required)
- `cursor` (optional)
- Returns: `followers[]`, `next_cursor`

### GET `/twitter/user/followings`
- `userName` (required)
- `cursor` (optional)

### GET `/twitter/user/check_follow`
Check whether user A follows user B.
- `user_id`, `target_user_id` (both required)

### GET `/twitter/user/mentions`
Tweets mentioning a user.
- `userName` (required)
- `cursor` (optional)

---

## Tweet endpoints (read)

### GET `/twitter/tweets`
Fetch multiple tweets by ID.
- `tweet_ids` (required) — comma-separated

### GET `/twitter/user/last_tweets`
Recent tweets from a user's timeline.
- `userName` (required)
- `cursor` (optional)

### GET `/twitter/tweet/replies`
Replies under a tweet.
- `tweet_id` (required)
- `cursor` (optional)

### GET `/twitter/tweet/quotations`
Quote-tweets of a tweet.
- `tweet_id` (required)
- `cursor` (optional)

### GET `/twitter/tweet/retweeters`
Users who retweeted a tweet.
- `tweet_id` (required)
- `cursor` (optional)

### GET `/twitter/tweet/thread_context`
Full thread a tweet belongs to.
- `tweet_id` (required)

### GET `/twitter/tweet/advanced_search`
Full-text search with operators (same query syntax as X's advanced search UI).
- `query` (required) — e.g. `from:elonmusk since:2025-01-01 until:2025-02-01`
- `cursor` (optional)

### GET `/twitter/trends`
Trending topics for a region.
- `woeid` (required) — Yahoo "Where On Earth" ID (1 = worldwide, 23424977 = US)

### GET `/twitter/article`
Long-form "Article" content (note: ~100 credits each).
- `tweet_id` (required)

### GET `/twitter/space/info`
X Space (audio room) metadata.
- `space_id` (required)

---

## Community endpoints

### GET `/twitter/community/info`
- `community_id` (required)

### GET `/twitter/community/tweets`
- `community_id` (required)
- `cursor` (optional)

### GET `/twitter/community/members`
- `community_id` (required)
- `cursor` (optional)

### GET `/twitter/community/moderators`
- `community_id` (required)

---

## Real-time monitoring

Poll-free alternative to repeatedly hitting a user's timeline.

### POST `/twitter/monitor/add_user`
Start tracking a user.
- Body: `{ "userId": "..." }`

### DELETE `/twitter/monitor/remove_user`
- Body: `{ "userId": "..." }`

### GET `/twitter/monitor/list`
Currently tracked users.

---

## Webhook / websocket filter rules

Define server-side rules; matching tweets are delivered to your webhook or websocket.

### POST `/twitter/filter_rules`
- Body: `{ "rule": "from:elonmusk OR #AI", "tag": "my-rule" }`

### PUT `/twitter/filter_rules/{id}`

### DELETE `/twitter/filter_rules/{id}`

### GET `/twitter/filter_rules`
List all active rules.

---

## Write endpoints (require login_cookie)

See [write-operations.md](write-operations.md) for the login flow and request bodies.

| Capability | Method | Path |
|---|---|---|
| Log in | POST | `/twitter/user_login_v2` |
| Create / reply tweet | POST | `/twitter/create_tweet_v2` |
| Delete tweet | DELETE | `/twitter/delete_tweet` |
| Like | POST | `/twitter/like_tweet` |
| Unlike | DELETE | `/twitter/like_tweet` |
| Retweet | POST | `/twitter/retweet_tweet` |
| Bookmark | POST | `/twitter/bookmark_tweet` |
| Unbookmark | DELETE | `/twitter/bookmark_tweet` |
| Follow | POST | `/twitter/follow_user` |
| Unfollow | DELETE | `/twitter/follow_user` |
| Send DM | POST | `/twitter/send_dm_v2` |
| Create community | POST | `/twitter/community/create_v2` |
| Delete community | DELETE | `/twitter/community/delete_v2` |
| Join community | POST | `/twitter/community/join_v2` |
| Leave community | DELETE | `/twitter/community/leave_v2` |
| Update profile | PUT | `/twitter/profile/update` |
| Update avatar | POST | `/twitter/profile/avatar` |
| Update banner | POST | `/twitter/profile/banner` |
| Upload media | POST | `/twitter/media/upload` |
| List bookmarks | GET | `/twitter/bookmarks` |

---

## Response shape (common fields)

Most list endpoints return:

```json
{
  "status": "success",
  "code": 0,
  "msg": "",
  "data": { ... },
  "next_cursor": "abc123"
}
```

Error responses:

```json
{
  "status": "error",
  "code": 401,
  "msg": "invalid api key"
}
```

Always check `response.status_code` (HTTP) *and* the body's `status`/`code` field.
