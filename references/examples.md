# twitterapi.io — Runnable examples

All examples assume `TWITTERAPI_IO_KEY` is set in the environment.

## Python — reusable client (reads)

```python
import os, time, requests

class TwitterAPIIO:
    BASE = "https://api.twitterapi.io"

    def __init__(self, api_key=None, timeout=30):
        self.key = api_key or os.environ["TWITTERAPI_IO_KEY"]
        self.timeout = timeout
        self.s = requests.Session()
        self.s.headers.update({"x-api-key": self.key})

    def _request(self, method, path, *, params=None, json=None, files=None, data=None):
        for attempt in range(3):
            kwargs = {"timeout": self.timeout}
            if params is not None: kwargs["params"] = params
            if json is not None:   kwargs["json"]   = json
            if files is not None:  kwargs["files"]  = files
            if data is not None:   kwargs["data"]   = data
            r = self.s.request(method, f"{self.BASE}{path}", **kwargs)
            if r.status_code in (429,) or r.status_code >= 500:
                time.sleep(2 ** attempt); continue
            if not r.ok:
                body = r.json() if r.headers.get("content-type","").startswith("application/json") else {}
                detail = body.get("detail") or body.get("msg") or r.text
                raise requests.HTTPError(f"{r.status_code}: {detail}", response=r)
            return r.json()
        r.raise_for_status()

    def get(self, path, **params):       return self._request("GET",   path, params=params)
    def post(self, path, body=None):     return self._request("POST",  path, json=body)
    def patch(self, path, body=None):    return self._request("PATCH", path, json=body)
    def put(self, path, body=None):      return self._request("PUT",   path, json=body)
    def delete(self, path, body=None):   return self._request("DELETE", path, json=body)

    # Reads
    def user_info(self, user_name):        return self.get("/twitter/user/info", userName=user_name)
    def user_about(self, user_name):       return self.get("/twitter/user_about", userName=user_name)
    def last_tweets(self, user_name, cursor=""): return self.get("/twitter/user/last_tweets", userName=user_name, cursor=cursor)
    def search_users(self, query):         return self.get("/twitter/user/search", query=query)       # query, not keyword
    def tweets_by_ids(self, ids):
        joined = ",".join(ids) if isinstance(ids, (list, tuple)) else ids
        return self.get("/twitter/tweets", tweet_ids=joined)                                           # snake!
    def advanced_search(self, query, cursor=""): return self.get("/twitter/tweet/advanced_search", query=query, cursor=cursor)
    def trends(self, woeid=1, count=30):   return self.get("/twitter/trends", woeid=woeid, count=count)
    def article(self, tweet_id):           return self.get("/twitter/article", tweet_id=tweet_id)       # snake!
    def space_detail(self, space_id):      return self.get("/twitter/spaces/detail", space_id=space_id) # snake!
    def check_follow(self, source_user_name, target_user_name):
        return self.get("/twitter/user/check_follow_relationship",
                        source_user_name=source_user_name, target_user_name=target_user_name)
    def balance(self):                     return self.get("/oapi/my/info")

    def iter_followers(self, user_name, page_size=200):
        cursor = ""
        while True:
            page = self.get("/twitter/user/followers",
                            userName=user_name, cursor=cursor, pageSize=page_size)
            for u in page.get("followers", []):
                yield u
            if not page.get("has_next_page"):
                return
            cursor = page.get("next_cursor") or ""


if __name__ == "__main__":
    api = TwitterAPIIO()
    d = api.user_info("elonmusk")["data"]
    print(f"{d['userName']} — {d['followers']:,} followers")
    print(f"Balance: {api.balance()['recharge_credits']:,} credits")
```

## Paginate every tweet matching a query

```python
api = TwitterAPIIO()
query = 'from:elonmusk since:2025-01-01 until:2025-02-01 min_faves:1000'

total, cursor = 0, ""
while True:
    page = api.advanced_search(query, cursor=cursor)
    tweets = page.get("tweets", [])
    total += len(tweets)
    for t in tweets:
        print(t["id"], t["createdAt"], t["text"][:80])
    if not page.get("has_next_page"):
        break
    cursor = page.get("next_cursor") or ""
print(f"\n{total} tweets")
```

## Cost guard before a big crawl

```python
def estimate_follower_cost(api, user_name):
    info = api.user_info(user_name)["data"]
    count = info["followers"]
    cost_usd = count / 1000 * 0.15
    print(f"{user_name}: {count:,} followers ~= ${cost_usd:,.2f}")
    return cost_usd

if estimate_follower_cost(api, "elonmusk") > 5.00:
    raise SystemExit("Confirm with user before proceeding")
```

## Node.js (fetch, Node 20+)

```javascript
const BASE = "https://api.twitterapi.io";
const headers = { "x-api-key": process.env.TWITTERAPI_IO_KEY };

async function call(path, params = {}) {
  const url = new URL(`${BASE}${path}`);
  for (const [k, v] of Object.entries(params))
    if (v !== undefined && v !== "") url.searchParams.set(k, v);
  const res = await fetch(url, { headers });
  const body = await res.json();
  if (!res.ok) throw new Error(`${res.status}: ${body.detail ?? body.msg ?? "unknown"}`);
  return body;
}

async function* iterFollowers(userName) {
  let cursor = "";
  while (true) {
    const data = await call("/twitter/user/followers", { userName, cursor, pageSize: 200 });
    for (const u of data.followers ?? []) yield u;
    if (!data.has_next_page) return;
    cursor = data.next_cursor ?? "";
  }
}

const info = (await call("/twitter/user/info", { userName: "elonmusk" })).data;
console.log(`${info.userName}: ${info.followers.toLocaleString()} followers`);
```

---

## v3 writes — bot-account API (recommended)

One-time registration, then `user_name` is the handle for every action.

```python
import os, requests

BASE = "https://api.twitterapi.io"
H = {"x-api-key": os.environ["TWITTERAPI_IO_KEY"], "Content-Type": "application/json"}

def register_bot(user_name, proxy, password=None, cookie=None, email=None, totp=None):
    body = {"user_name": user_name, "proxy": proxy}
    if password: body["password"] = password
    if cookie:   body["cookie"] = cookie              # format: k=v&k=v
    if email:    body["email"] = email
    if totp:     body["totp_code"] = totp
    r = requests.post(f"{BASE}/twitter/user_login_v3", json=body, headers=H)
    r.raise_for_status(); return r.json()

def send_tweet(user_name, text, community_id=None):
    body = {"user_name": user_name, "text": text}   # NOTE: text in v3, not tweet_text
    if community_id: body["community_id"] = community_id
    return requests.post(f"{BASE}/twitter/send_tweet_v3", json=body, headers=H).json()

def reply(user_name, text, reply_to_tweet_id):
    body = {"user_name": user_name, "text": text, "reply_to_tweet_id": reply_to_tweet_id}
    return requests.post(f"{BASE}/twitter/reply_tweet_v3", json=body, headers=H).json()

def quote(user_name, text, tweet_id, tweet_username):
    # tweet_username = author of the tweet being quoted (required in v3)
    body = {"user_name": user_name, "text": text, "tweet_id": tweet_id, "tweet_username": tweet_username}
    return requests.post(f"{BASE}/twitter/quote_tweet_v3", json=body, headers=H).json()

def like(user_name, tweet_id):
    return requests.post(f"{BASE}/twitter/like_tweet_v3",
                         json={"user_name": user_name, "tweet_id": tweet_id}, headers=H).json()

def retweet(user_name, tweet_id):
    return requests.post(f"{BASE}/twitter/retweet_v3",   # not retweet_tweet_v3!
                         json={"user_name": user_name, "tweet_id": tweet_id}, headers=H).json()

def follow(user_name, target_user_name=None, target_user_id=None):
    body = {"user_name": user_name}
    if target_user_name: body["target_user_name"] = target_user_name
    if target_user_id:   body["target_user_id"] = target_user_id
    return requests.post(f"{BASE}/twitter/follow_v3", json=body, headers=H).json()

def update_profile(user_name, name=None, bio=None, location=None, website=None, avatar=None, banner=None):
    body = {"user_name": user_name}
    for k, v in [("name",name),("bio",bio),("location",location),("website",website),
                 ("avatar",avatar),("banner",banner)]:
        if v is not None: body[k] = v
    return requests.put(f"{BASE}/twitter/update_profile_v3", json=body, headers=H).json()


# One-time setup
register_bot("my_bot_handle",
             proxy=os.environ["X_PROXY"],
             password=os.environ["X_PASSWORD"],
             email=os.environ["X_EMAIL"])

# Ongoing use
send_tweet("my_bot_handle", "Hello from v3!")
like("my_bot_handle", "1234567890")
follow("my_bot_handle", target_user_name="elonmusk")
```

---

## v2 writes — cookie-based (full-featured)

Use when you need scheduling, reports, list management, communities, or file upload.

```python
import os, base64, json, requests

BASE = "https://api.twitterapi.io"
H = {"x-api-key": os.environ["TWITTERAPI_IO_KEY"], "Content-Type": "application/json"}

def login_v2(user_name, email, password, proxy, totp_secret=None):
    body = {"user_name": user_name, "email": email, "password": password, "proxy": proxy}
    if totp_secret: body["totp_secret"] = totp_secret
    r = requests.post(f"{BASE}/twitter/user_login_v2", json=body, headers=H)
    r.raise_for_status()
    # login_cookies is base64-encoded JSON — pass back verbatim
    return r.json()["login_cookies"]

def create_tweet(cookies, proxy, text, *, reply_to=None, quote_id=None,
                 community_id=None, media_ids=None, schedule_for=None):
    body = {
        "login_cookies": cookies,
        "proxy":         proxy,
        "tweet_text":    text,                  # NOTE: tweet_text in v2 (not text)
    }
    if reply_to:     body["reply_to_tweet_id"] = reply_to   # NOT in_reply_to_tweet_id
    if quote_id:     body["quote_tweet_id"] = quote_id
    if community_id: body["community_id"] = community_id
    if media_ids:    body["media_ids"] = media_ids          # list of strings
    if schedule_for: body["schedule_for"] = schedule_for    # "2026-01-20T10:00:00.000Z"
    r = requests.post(f"{BASE}/twitter/create_tweet_v2", json=body, headers=H)
    r.raise_for_status(); return r.json()

def delete_tweet(cookies, proxy, tweet_id):
    return requests.post(f"{BASE}/twitter/delete_tweet_v2",
                         json={"login_cookies": cookies, "proxy": proxy, "tweet_id": tweet_id},
                         headers=H).json()

def send_dm(cookies, proxy, user_id, text, media_id=None, reply_to_message_id=None):
    body = {"login_cookies": cookies, "proxy": proxy, "user_id": user_id, "text": text}
    if media_id:            body["media_id"] = media_id           # singular
    if reply_to_message_id: body["reply_to_message_id"] = reply_to_message_id
    return requests.post(f"{BASE}/twitter/send_dm_to_user", json=body, headers=H).json()

def upload_media(cookies, proxy, path, media_category=None, is_long_video=False):
    # multipart, NOT JSON
    mime = "video/mp4" if path.lower().endswith(".mp4") else "image/jpeg"
    with open(path, "rb") as f:
        files = {"file": (os.path.basename(path), f, mime)}
        data  = {"login_cookies": cookies, "proxy": proxy, "is_long_video": str(is_long_video).lower()}
        if media_category: data["media_category"] = media_category
        # requests will set multipart Content-Type — don't add it manually
        r = requests.post(f"{BASE}/twitter/upload_media_v2",
                          files=files, data=data,
                          headers={"x-api-key": os.environ["TWITTERAPI_IO_KEY"]})
    r.raise_for_status(); return r.json()

def update_profile(cookies, proxy, *, name=None, description=None, location=None, url=None):
    # PATCH with JSON. Note: description (v2), NOT bio
    body = {"login_cookies": cookies, "proxy": proxy}
    for k, v in [("name",name),("description",description),("location",location),("url",url)]:
        if v is not None: body[k] = v
    r = requests.patch(f"{BASE}/twitter/update_profile_v2", json=body, headers=H)
    r.raise_for_status(); return r.json()

def update_avatar(cookies, proxy, path):
    # PATCH + multipart
    with open(path, "rb") as f:
        files = {"file": (os.path.basename(path), f, "image/jpeg")}
        data  = {"login_cookies": cookies, "proxy": proxy}
        r = requests.patch(f"{BASE}/twitter/update_avatar_v2",
                           files=files, data=data,
                           headers={"x-api-key": os.environ["TWITTERAPI_IO_KEY"]})
    r.raise_for_status(); return r.json()

def list_bookmarks(cookies, proxy, count=20, cursor=""):
    # count, NOT pageSize
    body = {"login_cookies": cookies, "proxy": proxy, "count": count, "cursor": cursor}
    return requests.post(f"{BASE}/twitter/bookmarks_v2", json=body, headers=H).json()


# Usage
cookies = login_v2(
    user_name=os.environ["X_USER"],
    email=os.environ["X_EMAIL"],
    password=os.environ["X_PASSWORD"],
    proxy=os.environ["X_PROXY"],
    totp_secret=os.environ.get("X_TOTP"),
)
proxy = os.environ["X_PROXY"]

# Upload a photo, then tweet with it
media_id = upload_media(cookies, proxy, "photo.jpg")["media_id"]
print(create_tweet(cookies, proxy, "Check this out", media_ids=[media_id]))
```

## Real-time monitoring via filter rules

Instead of polling `/user/last_tweets` on a timer:

```python
# 1. Configure a webhook URL on your dashboard (one-time)
# 2. Create a filter rule:
r = requests.post(
    f"{BASE}/oapi/tweet_filter/add_rule",
    json={
        "tag":              "ai-watchlist",
        "value":            "from:elonmusk OR #AI OR @OpenAI",
        "interval_seconds": 60,
    },
    headers=H,
).json()
print(r)                                   # {status, msg, rule_id}

# 3. Matched tweets stream to your webhook as they're posted.

# Delete it later
requests.delete(
    f"{BASE}/oapi/tweet_filter/delete_rule",
    json={"rule_id": r["rule_id"]}, headers=H,   # body JSON, not query
)
```

## User-level monitoring (requires subscription)

```python
# Add (field is x_user_name — NOT user_id)
requests.post(f"{BASE}/oapi/x_user_stream/add_user_to_monitor_tweet",
              json={"x_user_name": "elonmusk"}, headers=H)

# List what's being monitored
data = requests.get(f"{BASE}/oapi/x_user_stream/get_user_to_monitor_tweet",
                    headers=H).json()
# Each row carries an `id_for_user` used for removal
for row in data.get("data", []):
    print(row["x_user_screen_name"], row["id_for_user"])

# Remove (field is id_for_user — different from add!)
requests.post(f"{BASE}/oapi/x_user_stream/remove_user_to_monitor_tweet",
              json={"id_for_user": "abc123..."}, headers=H)
```
