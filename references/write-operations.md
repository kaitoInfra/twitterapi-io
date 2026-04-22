# twitterapi.io — Write operations (login_cookie flow)

Any endpoint that **modifies state** on behalf of an X account (posting, liking, following, DM'ing) requires two credentials:

1. `x-api-key` header — your twitterapi.io API key
2. `login_cookie` field in the JSON body — a session token obtained by logging in the target X account

The `login_cookie` is **per-X-account**: each account you want to control needs its own login.

All write endpoints in this file are **POST** (except profile updates, which are **PATCH**) and take a JSON body. None of them accept DELETE — the "reverse" operation is a separate endpoint (e.g. `unlike_tweet_v2`, `unfollow_user_v2`).

## Step 1 — Log in

```http
POST https://api.twitterapi.io/twitter/user_login_v2
x-api-key: YOUR_KEY
Content-Type: application/json

{
  "email_or_username": "user@example.com",
  "password": "...",
  "totp_secret": "OPTIONAL_BASE32_2FA_SEED"
}
```

Response:

```json
{
  "status": "success",
  "login_cookie": "base64-encoded-session-blob",
  "user_id": "44196397",
  "screen_name": "elonmusk"
}
```

**Handling practice:**
- Store `login_cookie` encrypted at rest — it's effectively a session token
- Cookies expire — catch `401` / `cookie_expired` and re-login
- Never log or commit cookies to git; never pass via URL
- If 2FA is enabled, supply `totp_secret` (the **base32 seed**, not a 6-digit code) — the server computes the current TOTP

## Step 2 — Call a write endpoint

Every write endpoint follows the same pattern: include `login_cookie` in the JSON body alongside the action-specific fields.

### Create a tweet

```python
import requests, os

BASE = "https://api.twitterapi.io"
HEADERS = {
    "x-api-key": os.environ["TWITTERAPI_IO_KEY"],
    "Content-Type": "application/json",
}

payload = {
    "login_cookie": cookie,
    "text": "Hello from twitterapi.io",
}
r = requests.post(f"{BASE}/twitter/create_tweet_v2", json=payload, headers=HEADERS)
print(r.json())
```

### Reply to a tweet

Add `in_reply_to_tweet_id`:

```json
{
  "login_cookie": "...",
  "text": "Great point!",
  "in_reply_to_tweet_id": "1234567890"
}
```

### Quote tweet

Add `quoted_tweet_id`:

```json
{ "login_cookie": "...", "text": "worth reading", "quoted_tweet_id": "1234567890" }
```

### Tweet with media

1. Upload each file via `POST /twitter/upload_media_v2` → returns `media_id`
2. Pass IDs in the create body:

```json
{ "login_cookie": "...", "text": "photo!", "media_ids": ["abc", "def"] }
```

### Delete a tweet

```http
POST /twitter/delete_tweet_v2
{ "login_cookie": "...", "tweetId": "1234567890" }
```

### Like / unlike (two separate endpoints)

```http
POST /twitter/like_tweet_v2     { "login_cookie": "...", "tweetId": "123" }
POST /twitter/unlike_tweet_v2   { "login_cookie": "...", "tweetId": "123" }
```

### Retweet

```http
POST /twitter/retweet_tweet_v2  { "login_cookie": "...", "tweetId": "123" }
```

### Bookmark / unbookmark

```http
POST /twitter/bookmark_tweet_v2    { "login_cookie": "...", "tweetId": "123" }
POST /twitter/unbookmark_tweet_v2  { "login_cookie": "...", "tweetId": "123" }
```

### List own bookmarks (authenticated read, `login_cookie` required)

```http
POST /twitter/bookmarks_v2
{ "login_cookie": "...", "cursor": "" }
```

### Follow / unfollow (two separate endpoints)

```http
POST /twitter/follow_user_v2    { "login_cookie": "...", "userId": "44196397" }
POST /twitter/unfollow_user_v2  { "login_cookie": "...", "userId": "44196397" }
```

### Send a DM

```http
POST /twitter/send_dm_to_user
{
  "login_cookie": "...",
  "userId": "44196397",
  "message": "hi"
}
```

### Update profile / avatar / banner (note: **PATCH**)

```http
PATCH /twitter/update_profile_v2   { "login_cookie": "...", "name": "...", "bio": "...", "location": "...", "url": "..." }
PATCH /twitter/update_avatar_v2    { "login_cookie": "...", "image": "<base64 or upload_id>" }
PATCH /twitter/update_banner_v2    { "login_cookie": "...", "image": "<base64 or upload_id>" }
```

### Communities

```http
POST /twitter/create_community_v2  { "login_cookie": "...", "name": "...", "description": "..." }
POST /twitter/delete_community_v2  { "login_cookie": "...", "communityId": "..." }
POST /twitter/join_community_v2    { "login_cookie": "...", "communityId": "..." }
POST /twitter/leave_community_v2   { "login_cookie": "...", "communityId": "..." }
```

## Safety notes for automated writes

- **Rate-limit yourself** — X (not twitterapi.io) will shadowban or suspend accounts that post/follow/like too fast. Conservative daily budget: < 50 follows, < 300 tweets, with seconds of delay between actions.
- **Don't batch writes without confirmation** — if the user asks for "follow everyone in this list", confirm the count first and offer a dry-run.
- **Respect X's terms of service** — automated spam, harassment, or mass-DM can get the underlying account banned.
- **One cookie, one machine** — using the same `login_cookie` from many IPs simultaneously looks like a compromised account and will get flagged.
