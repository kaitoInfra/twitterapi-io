# twitterapi.io — Write operations

Any endpoint that **modifies state** on behalf of an X account (posting, liking, following, DM'ing) requires **three** things in the JSON body, plus the standard `x-api-key` header:

1. `login_cookies` (**plural** — not `login_cookie`) — session token from `/twitter/user_login_v2`
2. `proxy` — HTTP/SOCKS proxy URL the server will route the request through. Proxies are configured/managed on the twitterapi.io dashboard; format is typically `http://user:pass@host:port`
3. Action-specific fields — most are **snake_case** (`tweet_id`, `user_id`, `tweet_text`, `text`) not camelCase

Every write endpoint is **POST** (profile updates are **PATCH**). There is no DELETE method — the "reverse" action is a separate endpoint (`unlike_tweet_v2`, `unfollow_user_v2`, `unbookmark_tweet_v2`).

## Step 1 — Log in

```http
POST https://api.twitterapi.io/twitter/user_login_v2
x-api-key: YOUR_KEY
Content-Type: application/json

{
  "user_name":   "elonmusk",
  "email":       "user@example.com",
  "password":    "...",
  "proxy":       "http://user:pass@host:port",
  "totp_secret": "OPTIONAL_BASE32_2FA_SEED"
}
```

All four primary fields (`user_name`, `email`, `password`, `proxy`) are **required**. `totp_secret` is only required if the account has 2FA enabled — pass the **base32 seed**, not a 6-digit code; the server computes the current TOTP.

Response:

```json
{
  "status": "success",
  "login_cookies": "base64-encoded-session-blob",
  "user_id": "44196397",
  "screen_name": "elonmusk"
}
```

**Handling practice:**
- Store `login_cookies` encrypted at rest — it's effectively a session token
- Cookies expire — catch 401 / `cookie_expired` and re-login
- **Pin one proxy per login session** — using the same `login_cookies` from different proxies looks like a compromised account and will trigger X's anti-bot
- Never log or commit cookies or proxies to git

## Step 2 — Call a write endpoint

Every body follows the same shape: `login_cookies` + `proxy` + action fields.

### Create a tweet (text field is `tweet_text`, NOT `text`)

```python
import requests, os

BASE = "https://api.twitterapi.io"
HEADERS = {
    "x-api-key": os.environ["TWITTERAPI_IO_KEY"],
    "Content-Type": "application/json",
}

payload = {
    "login_cookies": cookies,
    "proxy":         "http://user:pass@host:port",
    "tweet_text":    "Hello from twitterapi.io",
}
r = requests.post(f"{BASE}/twitter/create_tweet_v2", json=payload, headers=HEADERS)
print(r.json())
```

### Reply to a tweet

Add `in_reply_to_tweet_id`:

```json
{
  "login_cookies": "...",
  "proxy":         "...",
  "tweet_text":    "Great point!",
  "in_reply_to_tweet_id": "1234567890"
}
```

### Quote tweet

Add `quoted_tweet_id`:

```json
{
  "login_cookies": "...",
  "proxy":         "...",
  "tweet_text":    "worth reading",
  "quoted_tweet_id": "1234567890"
}
```

### Tweet with media

1. Upload each file via `POST /twitter/upload_media_v2` (multipart; `file` field). Returns `media_id`.
2. Pass IDs in the create body:

```json
{ "login_cookies":"...", "proxy":"...", "tweet_text":"photo!", "media_ids":["abc","def"] }
```

### Delete a tweet (`tweet_id`, snake_case)

```http
POST /twitter/delete_tweet_v2
{ "login_cookies":"...", "proxy":"...", "tweet_id":"1234567890" }
```

### Like / unlike (two separate endpoints)

```http
POST /twitter/like_tweet_v2     { "login_cookies":"...", "proxy":"...", "tweet_id":"123" }
POST /twitter/unlike_tweet_v2   { "login_cookies":"...", "proxy":"...", "tweet_id":"123" }
```

### Retweet

```http
POST /twitter/retweet_tweet_v2  { "login_cookies":"...", "proxy":"...", "tweet_id":"123" }
```

### Bookmark / unbookmark

```http
POST /twitter/bookmark_tweet_v2    { "login_cookies":"...", "proxy":"...", "tweet_id":"123" }
POST /twitter/unbookmark_tweet_v2  { "login_cookies":"...", "proxy":"...", "tweet_id":"123" }
```

### List own bookmarks (authenticated read)

```http
POST /twitter/bookmarks_v2
{ "login_cookies":"...", "proxy":"...", "cursor":"" }
```

### Follow / unfollow

```http
POST /twitter/follow_user_v2    { "login_cookies":"...", "proxy":"...", "user_id":"44196397" }
POST /twitter/unfollow_user_v2  { "login_cookies":"...", "proxy":"...", "user_id":"44196397" }
```

### Send a DM (fields: `user_id` + `text`, NOT `message`)

```http
POST /twitter/send_dm_to_user
{
  "login_cookies": "...",
  "proxy":         "...",
  "user_id":       "44196397",
  "text":          "hi"
}
```

### Update profile / avatar / banner (**PATCH**)

```http
PATCH /twitter/update_profile_v2   { "login_cookies":"...", "proxy":"...", "name":"...", "bio":"...", "location":"...", "url":"..." }
PATCH /twitter/update_avatar_v2    { "login_cookies":"...", "proxy":"...", "image":"<base64 or upload_id>" }
PATCH /twitter/update_banner_v2    { "login_cookies":"...", "proxy":"...", "image":"<base64 or upload_id>" }
```

### Communities

```http
POST /twitter/create_community_v2  { "login_cookies":"...", "proxy":"...", "name":"...", "description":"..." }
POST /twitter/delete_community_v2  { "login_cookies":"...", "proxy":"...", "community_id":"..." }
POST /twitter/join_community_v2    { "login_cookies":"...", "proxy":"...", "community_id":"..." }
POST /twitter/leave_community_v2   { "login_cookies":"...", "proxy":"...", "community_id":"..." }
```

## Error-message progression

The API validates required fields one at a time (FastAPI-style). If your call fails with `{"detail":"X is required"}`, add that field and retry — the next missing field will be named. Common progression for writes: `login_cookies` → action-specific id → `proxy` → `text`/`tweet_text`.

## Safety notes for automated writes

- **Rate-limit yourself** — X (not twitterapi.io) will shadowban or suspend accounts that post/follow/like too fast. Conservative daily budget: < 50 follows, < 300 tweets, with seconds of delay between actions.
- **Don't batch writes without confirmation** — if the user asks for "follow everyone in this list", confirm the count and offer a dry-run first.
- **Respect X's terms of service** — automated spam, harassment, or mass-DM can get the underlying account banned.
- **One proxy per cookie** — switching IPs mid-session looks like account takeover.
