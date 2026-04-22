# twitterapi.io — Write operations

The API offers **two parallel write patterns**. Pick one and stick with it for a given account.

## Decision matrix: v2 vs v3

| | **v2 (cookie-based)** | **v3 (bot-account)** |
|---|---|---|
| Pattern | Client holds the session; every call sends `login_cookies` + `proxy` + action fields | Client registers the account once (`user_login_v3`); server stores the session. Subsequent calls send only `user_name` + action fields |
| Initial auth | `/twitter/user_login_v2` → returns `login_cookies` (base64 JSON) | `/twitter/user_login_v3` → registers bot server-side |
| Proxy | Provided on every request | Stored server-side; stable |
| Text field | `tweet_text` | `text` |
| Reply field | `reply_to_tweet_id` | `reply_to_tweet_id` |
| Profile "about" field | `description` (max 160) | `bio` |
| Media for tweets | Upload first (`/twitter/upload_media_v2`, multipart) → pass `media_ids[]` | Base64 inline (`media_data_base64` + `media_type`) |
| Best for | Full account control, integrations where cookies already exist | Simplicity; long-lived automation; when you don't want to manage cookies client-side |

**Recommendation for most users: use v3.** Cleaner, fewer footguns, no per-request proxy management. Only use v2 if you already own the cookies (e.g. from a prior OAuth exchange) or need features v3 doesn't expose (scheduled tweets, `report_v2`, list-management `_v2`, communities-create, media upload with arbitrary files).

---

## v2 — cookie-based

### Step 1: Log in

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

Required: `user_name`, `email`, `password`, `proxy`. Optional: `totp_secret` (base32 seed, not a 6-digit code).

Returns `login_cookies` — a **base64-encoded JSON** of a cookie dict. Pass it back verbatim to every v2 write call; the server base64-decodes and parses it on each call.

### Step 2: Every v2 write body = `login_cookies` + `proxy` + action fields

Most action fields are **snake_case** (`tweet_id`, `user_id`, `community_id`). Media/avatar/banner use `multipart/form-data` (not JSON). Profile updates use PATCH.

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
    "tweet_text":    "Hello from v2",
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
    "description":   "My new bio",        # ≤160  (note: description, NOT bio, in v2)
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

### v2 body field cheat sheet

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

---

## v3 — bot-account API (recommended for most)

### Step 1: Register a bot account (once)

```http
POST https://api.twitterapi.io/twitter/user_login_v3
x-api-key: YOUR_KEY

{
  "user_name": "your_x_handle",
  "proxy":     "http://user:pass@host:port",
  "password":  "...",
  "email":     "user@example.com",
  "totp_code": "OPTIONAL_BASE32_2FA_SEED"
}
```

Required: `user_name`, `proxy`, and **either** `cookie` (format `k=v&k=v`) **or** `password`. Optional: `email`, `totp_code`. On success the server stores the session under your API-key account.

### Step 2: Every v3 call references the bot by `user_name`

```python
BASE = "https://api.twitterapi.io"
HEADERS = {"x-api-key": os.environ["TWITTERAPI_IO_KEY"],
           "Content-Type": "application/json"}

# Send a tweet (text field is `text`, not `tweet_text`)
requests.post(f"{BASE}/twitter/send_tweet_v3",
              json={"user_name": "my_bot", "text": "Hello from v3"},
              headers=HEADERS)

# Reply
requests.post(f"{BASE}/twitter/reply_tweet_v3",
              json={"user_name": "my_bot", "text": "good point", "reply_to_tweet_id": "123"},
              headers=HEADERS)

# Quote (note: tweet_username is the AUTHOR of the quoted tweet, required)
requests.post(f"{BASE}/twitter/quote_tweet_v3",
              json={"user_name": "my_bot", "text": "worth reading",
                    "tweet_id": "123", "tweet_username": "elonmusk"},
              headers=HEADERS)

# Like / retweet / follow
requests.post(f"{BASE}/twitter/like_tweet_v3",
              json={"user_name": "my_bot", "tweet_id": "123"},
              headers=HEADERS)

requests.post(f"{BASE}/twitter/retweet_v3",          # note: retweet_v3, not retweet_tweet_v3
              json={"user_name": "my_bot", "tweet_id": "123"},
              headers=HEADERS)

requests.post(f"{BASE}/twitter/follow_v3",
              json={"user_name": "my_bot", "target_user_name": "elonmusk"},  # or target_user_id
              headers=HEADERS)

# Update profile (PUT, `bio` here — NOT `description` like v2)
requests.put(f"{BASE}/twitter/update_profile_v3",
             json={"user_name": "my_bot",
                   "name": "New Name",
                   "bio": "New bio",
                   "location": "...",
                   "website": "https://...",
                   "avatar": base64_image,
                   "banner": base64_image},
             headers=HEADERS)

# Get bot account status
requests.get(f"{BASE}/twitter/get_my_x_account_detail_v3",
             json={"user_name": "my_bot"},  # GET with body — unusual
             headers=HEADERS)

# Delete the bot from your account
requests.delete(f"{BASE}/twitter/delete_my_x_account_v3",
                json={"user_name": "my_bot"}, headers=HEADERS)
```

### What v3 doesn't have (fall back to v2)
- `schedule_for` / scheduled tweets
- Report (`report_v2`)
- Communities create/join/leave/delete
- List `add_member_v2`
- File-upload media (v3 takes base64 inline; fine for small images but not large video)

---

## Safety notes for automated writes

- **Rate-limit yourself**. X (not twitterapi.io) will shadowban / suspend accounts that post/follow/like too fast. Conservative daily budget: <50 follows, <300 tweets, seconds of delay between actions.
- **Don't batch writes without confirmation**. "Follow everyone in this list" → confirm the count first, offer a dry-run.
- **Respect X's Terms of Service**. Automated spam / harassment / mass-DM can get the underlying account banned.
- **Pin one proxy per session**. Using the same cookies/bot from many IPs looks like account takeover and trips X's anti-bot.
- **Handle cookie expiry**. 401 / `cookie_expired` → re-run login and retry once.
