# twitterapi.io — Write operations (login_cookie flow)

Any endpoint that **modifies state** on behalf of an X account (posting, liking, following, DM'ing) requires two credentials:

1. `x-api-key` header — your twitterapi.io API key
2. `login_cookie` field in the request body — a session token obtained by logging in the target X account

The `login_cookie` is **per-account**: each X account you want to control needs its own login.

## Step 1 — Log in

```http
POST https://api.twitterapi.io/twitter/user_login_v2
x-api-key: YOUR_KEY
Content-Type: application/json

{
  "email_or_username": "user@example.com",
  "password": "...",
  "totp_secret": "OPTIONAL_2FA_SECRET"
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
- Store `login_cookie` encrypted at rest (it's effectively a session token)
- Cookies expire — catch `401` / `cookie_expired` errors and re-login
- Never log or commit cookies to git
- If the account has 2FA enabled, you must supply `totp_secret` (the base32 seed, not a 6-digit code); the server computes the current code

## Step 2 — Call a write endpoint

Every write endpoint follows the same pattern — include `login_cookie` in the JSON body alongside the action-specific fields.

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

1. Upload each media file via `POST /twitter/media/upload` → returns `media_id`
2. Pass IDs in create_tweet:

```json
{ "login_cookie": "...", "text": "photo!", "media_ids": ["abc", "def"] }
```

### Like / unlike

```http
POST   /twitter/like_tweet        { "login_cookie": "...", "tweet_id": "123" }
DELETE /twitter/like_tweet        { "login_cookie": "...", "tweet_id": "123" }
```

### Retweet

```http
POST /twitter/retweet_tweet       { "login_cookie": "...", "tweet_id": "123" }
```

### Follow / unfollow

```http
POST   /twitter/follow_user       { "login_cookie": "...", "user_id": "44196397" }
DELETE /twitter/follow_user       { "login_cookie": "...", "user_id": "44196397" }
```

### Send a DM

```http
POST /twitter/send_dm_v2
{
  "login_cookie": "...",
  "recipient_user_id": "44196397",
  "text": "hi"
}
```

## Safety notes for automated writes

- **Rate-limit yourself** — X (not twitterapi.io) will shadowban or suspend accounts that post/follow/like too fast. Conservative budget: < 50 follows/day, < 300 tweets/day, seconds of delay between actions.
- **Don't batch writes without confirmation** — when the user asks for "follow everyone in this list", confirm the count before executing and offer a dry-run first.
- **Respect X's terms of service** — automated spam, harassment, or mass-DM sending is against platform rules and can get the underlying account banned.
- **One cookie, one machine** — using the same `login_cookie` from many IPs simultaneously looks like a compromised account and will get flagged.
