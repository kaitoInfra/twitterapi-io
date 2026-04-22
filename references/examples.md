# twitterapi.io — Runnable examples

Every example assumes `TWITTERAPI_IO_KEY` is set in the environment.

## Python — reusable client

```python
import os, time, requests

class TwitterAPIIO:
    BASE = "https://api.twitterapi.io"

    def __init__(self, api_key=None, timeout=30):
        self.key = api_key or os.environ["TWITTERAPI_IO_KEY"]
        self.timeout = timeout
        self.s = requests.Session()
        self.s.headers.update({"x-api-key": self.key})

    def _request(self, method, path, *, params=None, json=None):
        for attempt in range(3):
            r = self.s.request(method, f"{self.BASE}{path}",
                               params=params, json=json, timeout=self.timeout)
            if r.status_code == 429:
                time.sleep(2 ** attempt)
                continue
            if r.status_code >= 500:
                time.sleep(2 ** attempt)
                continue
            r.raise_for_status()
            return r.json()
        r.raise_for_status()

    def get(self, path, **params):
        return self._request("GET", path, params=params)

    def post(self, path, json=None):
        return self._request("POST", path, json=json)

    # ---- convenience wrappers ----

    def user_info(self, user_name):
        return self.get("/twitter/user/info", userName=user_name)

    def user_last_tweets(self, user_name, cursor=""):
        return self.get("/twitter/user/last_tweets", userName=user_name, cursor=cursor)

    def advanced_search(self, query, cursor=""):
        return self.get("/twitter/tweet/advanced_search", query=query, cursor=cursor)

    def iter_followers(self, user_name, page_size=200):
        cursor = ""
        while True:
            page = self.get("/twitter/user/followers",
                            userName=user_name, cursor=cursor, pageSize=page_size)
            for u in page.get("followers", []):
                yield u
            if not page.get("has_next_page"):
                return
            cursor = page.get("next_cursor", "")


if __name__ == "__main__":
    api = TwitterAPIIO()
    print(api.user_info("elonmusk"))
```

## Paginate through every tweet matching a query

```python
api = TwitterAPIIO()
query = 'from:elonmusk since:2025-01-01 until:2025-02-01 min_faves:1000'
total = 0
cursor = ""
while True:
    page = api.advanced_search(query, cursor=cursor)
    tweets = page.get("tweets", [])
    total += len(tweets)
    for t in tweets:
        print(t["id"], t["text"][:80])
    if not page.get("has_next_page"):
        break
    cursor = page.get("next_cursor", "")
print(f"\n{total} tweets")
```

## Cost guard before a big crawl

```python
def estimate_follower_cost(api, user_name):
    info = api.user_info(user_name)
    count = info["user"]["followers_count"]
    # $0.15 per 1,000 followers
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

async function userInfo(userName) {
  const url = new URL(`${BASE}/twitter/user/info`);
  url.searchParams.set("userName", userName);
  const res = await fetch(url, { headers });
  if (!res.ok) throw new Error(`${res.status} ${await res.text()}`);
  return res.json();
}

async function* iterFollowers(userName) {
  let cursor = "";
  while (true) {
    const url = new URL(`${BASE}/twitter/user/followers`);
    url.searchParams.set("userName", userName);
    url.searchParams.set("pageSize", "200");
    if (cursor) url.searchParams.set("cursor", cursor);
    const res = await fetch(url, { headers });
    if (!res.ok) throw new Error(`${res.status} ${await res.text()}`);
    const data = await res.json();
    for (const u of data.followers ?? []) yield u;
    if (!data.has_next_page) return;
    cursor = data.next_cursor ?? "";
  }
}

console.log(await userInfo("elonmusk"));
```

## Post a tweet (full login + write flow)

```python
import os, requests

BASE = "https://api.twitterapi.io"
HEADERS = {
    "x-api-key": os.environ["TWITTERAPI_IO_KEY"],
    "Content-Type": "application/json",
}

def login(email_or_user, password, totp=None):
    body = {"email_or_username": email_or_user, "password": password}
    if totp:
        body["totp_secret"] = totp
    r = requests.post(f"{BASE}/twitter/user_login_v2", json=body, headers=HEADERS)
    r.raise_for_status()
    return r.json()["login_cookie"]

def post_tweet(cookie, text, reply_to=None):
    body = {"login_cookie": cookie, "text": text}
    if reply_to:
        body["in_reply_to_tweet_id"] = reply_to
    r = requests.post(f"{BASE}/twitter/create_tweet_v2", json=body, headers=HEADERS)
    r.raise_for_status()
    return r.json()

def like_tweet(cookie, tweet_id):
    body = {"login_cookie": cookie, "tweetId": tweet_id}
    r = requests.post(f"{BASE}/twitter/like_tweet_v2", json=body, headers=HEADERS)
    r.raise_for_status()
    return r.json()

def follow(cookie, user_id):
    body = {"login_cookie": cookie, "userId": user_id}
    r = requests.post(f"{BASE}/twitter/follow_user_v2", json=body, headers=HEADERS)
    r.raise_for_status()
    return r.json()

cookie = login(os.environ["X_EMAIL"], os.environ["X_PASSWORD"], os.environ.get("X_TOTP"))
print(post_tweet(cookie, "Hello from twitterapi.io"))
```

**Persist the cookie** so you don't re-login every run — store it encrypted, re-login only on 401.

## Real-time monitoring without polling

Instead of hitting `/user/last_tweets` on a timer:

1. `POST /oapi/tweet_filter/add_rule` with a rule body like `{ "rule_text": "from:elonmusk OR #AI", "active": true }`
2. Open the websocket endpoint shown in your dashboard, or configure a webhook URL
3. Matched tweets stream to you as they're posted

This is both cheaper and faster than polling and avoids rate limits.

## User-level monitoring (alternative)

For just "tell me when @someone tweets":

```python
# add
requests.post(f"{BASE}/oapi/x_user_stream/add_user_to_monitor_tweet",
              json={"userId": "44196397"}, headers=HEADERS)

# list what's being monitored
requests.get(f"{BASE}/oapi/x_user_stream/get_user_to_monitor_tweet", headers=HEADERS).json()

# stop
requests.post(f"{BASE}/oapi/x_user_stream/remove_user_to_monitor_tweet",
              json={"userId": "44196397"}, headers=HEADERS)
```
