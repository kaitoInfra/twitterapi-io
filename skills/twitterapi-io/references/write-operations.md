# twitterapi.io — Write operations

Any endpoint that **modifies state** on behalf of an X account requires three things in the JSON body, plus the standard `x-api-key` header:

1. `login_cookies` (**plural** — not `login_cookie`) — session token from `/twitter/user_login_v2`. It's a **base64-encoded JSON** of a cookie dict; pass it back verbatim.
2. `proxy` — HTTP/SOCKS proxy URL (e.g. `http://user:pass@host:port`); proxies are configured/managed on the twitterapi.io dashboard.
3. Action-specific fields — most are **snake_case** (`tweet_id`, `user_id`, `tweet_text`).

Almost every write endpoint is **POST**. Profile updates are **PATCH**. There is no DELETE method — the "reverse" action is a separate endpoint (`unlike_tweet_v2`, `unfollow_user_v2`, `unbookmark_tweet_v2`).

## Step 1 — Log in

```http
POST https://api.twitterapi.io/twitter/user_login_v2
x-api-key: YOUR_KEY
Content-Type: application/json

{
  "user_name": "your_x_handle",
  "email":     "user@example.com",
  "password":  "...",
  "proxy":     "http://user:pass@host:port",
  "totp_secret": "OPTIONAL_BASE32_2FA_SEED"
}
```

Required: `user_name`, `email`, `password`, `proxy`. Optional: `totp_secret` (base32 seed, not a 6-digit code) — strongly recommended; without 2FA the cookie can be flagged.

Returns `login_cookies` — a base64-encoded JSON cookie dict. Pass it back verbatim to every subsequent write call.

## Step 2 — Every write body = `login_cookies` + `proxy` + action fields

```python
import os, requests

BASE = "https://api.twitterapi.io"
HEADERS = {
    "x-api-key": os.environ["TWITTERAPI_IO_KEY"],
    "Content-Type": "application/json",
}

# Create tweet — text field is tweet_text, reply field is reply_to_tweet_id
body = {
    "login_cookies": cookies,
    "proxy":         "http://user:pass@host:port",
    "tweet_text":    "Hello from twitterapi.io",
    # optional: reply_to_tweet_id, quote_tweet_id, community_id,
    # media_ids=["id1","id2"], attachment_url, is_note_tweet,
    # schedule_for="2026-01-20T10:00:00.000Z"
}
r = requests.post(f"{BASE}/twitter/create_tweet_v2", json=body, headers=HEADERS)
```

```python
# Like — POST to a separate endpoint, not DELETE
def like(cookies, proxy, tweet_id):
    return requests.post(f"{BASE}/twitter/like_tweet_v2",
                         json={"login_cookies": cookies, "proxy": proxy, "tweet_id": tweet_id},
                         headers=HEADERS).json()

def unlike(cookies, proxy, tweet_id):
    return requests.post(f"{BASE}/twitter/unlike_tweet_v2",  # separate endpoint
                         json={"login_cookies": cookies, "proxy": proxy, "tweet_id": tweet_id},
                         headers=HEADERS).json()
```

```python
# Send DM — fields are user_id + text
body = {
    "login_cookies": cookies,
    "proxy":         proxy,
    "user_id":       "44196397",
    "text":          "hi",
    # optional: media_id (singular), reply_to_message_id
}
r = requests.post(f"{BASE}/twitter/send_dm_to_user", json=body, headers=HEADERS)
```

```python
# Upload media — multipart, not JSON
files = {"file": ("photo.jpg", open("photo.jpg","rb"), "image/jpeg")}
data = {
    "login_cookies": cookies,
    "proxy":         proxy,
    # optional: media_category, is_long_video
}
r = requests.post(f"{BASE}/twitter/upload_media_v2", files=files, data=data,
                  headers={"x-api-key": os.environ["TWITTERAPI_IO_KEY"]})  # let requests set multipart Content-Type
media_id = r.json()["media_id"]
```

```python
# Update profile — PATCH, description (not bio), at least one field
body = {
    "login_cookies": cookies,
    "proxy":         proxy,
    "name":          "My new name",      # ≤50
    "description":   "My new bio",        # ≤160  (note: description, NOT bio)
    "location":      "San Francisco",     # ≤30
    "url":           "https://example.com",
}
r = requests.patch(f"{BASE}/twitter/update_profile_v2", json=body, headers=HEADERS)
```

```python
# Update avatar / banner — PATCH + multipart
files = {"file": ("avatar.jpg", open("avatar.jpg","rb"), "image/jpeg")}
data = {"login_cookies": cookies, "proxy": proxy}
r = requests.patch(f"{BASE}/twitter/update_avatar_v2", files=files, data=data,
                   headers={"x-api-key": os.environ["TWITTERAPI_IO_KEY"]})
```

```python
# List bookmarks — uses count (not pageSize)
body = {"login_cookies": cookies, "proxy": proxy, "count": 20, "cursor": ""}
r = requests.post(f"{BASE}/twitter/bookmarks_v2", json=body, headers=HEADERS)
```

```python
# Report — tweet_id OR user_id + reason from 12-value enum
body = {
    "login_cookies": cookies,
    "proxy":         proxy,
    "tweet_id":      "1234567890",        # or user_id instead
    "reason":        "SpamSimpleOption",  # one of: SpamSimpleOption, HateOrAbuseSimpleOption,
                                           # ChildSafetySimpleOption, ViolentSpeechSimpleOption,
                                           # ViolentMediaSimpleOption, IRBSimpleOption,
                                           # ImpersonationSimpleOption, AdultContentSimpleOption,
                                           # PrivateContentSimpleOption, SuicideSelfHarmSimpleOption,
                                           # TerrorismSimpleOption, CivicIntegritySimpleOption
}
```

```python
# Delete community — note BOTH id AND name are required
body = {
    "login_cookies":  cookies,
    "proxy":          proxy,
    "community_id":   "1493446837214187523",
    "community_name": "Build in Public",
}
```

## Body field cheat sheet

| Endpoint | Required body fields |
|---|---|
| `create_tweet_v2` | `login_cookies`, `proxy`, `tweet_text` |
| `delete_tweet_v2` / `like_tweet_v2` / `unlike_tweet_v2` / `retweet_tweet_v2` / `bookmark_tweet_v2` / `unbookmark_tweet_v2` | `login_cookies`, `proxy`, `tweet_id` |
| `follow_user_v2` / `unfollow_user_v2` | `login_cookies`, `proxy`, `user_id` |
| `bookmarks_v2` | `login_cookies`, `proxy`; opt. `count` (def 20), `cursor` |
| `send_dm_to_user` | `login_cookies`, `proxy`, `user_id`, `text` |
| `report_v2` | `login_cookies`, `proxy`, (`tweet_id` OR `user_id`), `reason` |
| `create_community_v2` | `login_cookies`, `proxy`, `name`, `description` |
| `join_community_v2` / `leave_community_v2` | `login_cookies`, `proxy`, `community_id` |
| `delete_community_v2` | `login_cookies`, `proxy`, `community_id`, `community_name` |
| `list/add_member_v2` | `login_cookies`, `proxy`, `list_id`, `user_id` |
| `update_profile_v2` (PATCH) | `login_cookies`, `proxy` + at least one of `name`, `description`, `location`, `url` |
| `update_avatar_v2` / `update_banner_v2` (PATCH, multipart) | `file`, `login_cookies`, `proxy` |
| `upload_media_v2` (multipart) | `file`, `login_cookies`, `proxy` |

## Cookie / session handling

- Store `login_cookies` encrypted at rest — it's effectively a session token
- Cookies expire — catch 401 / `cookie_expired` and re-login
- **Pin one proxy per login session** — using the same `login_cookies` from different proxies looks like a compromised account and trips X's anti-bot
- Never log or commit cookies / proxies / `totp_secret`

## Safety notes for automated writes

- **Rate-limit yourself** — X (not twitterapi.io) will shadowban or suspend accounts that post/follow/like too fast. Conservative daily budget: <50 follows, <300 tweets, with seconds of delay between actions.
- **Don't batch writes without confirmation** — if the user asks for "follow everyone in this list", confirm the count and offer a dry-run.
- **Respect X's terms of service** — automated spam, harassment, or mass-DM can get the underlying account banned.
