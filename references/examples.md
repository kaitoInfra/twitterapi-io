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

    def _get(self, path, **params):
        for attempt in range(3):
            r = self.s.get(f"{self.BASE}{path}", params=params, timeout=self.timeout)
            if r.status_code == 429:
                time.sleep(2 ** attempt)
                continue
            r.raise_for_status()
            return r.json()
        r.raise_for_status()

    def _post(self, path, json=None):
        r = self.s.post(f"{self.BASE}{path}", json=json, timeout=self.timeout)
        r.raise_for_status()
        return r.json()

    def user_info(self, user_name):
        return self._get("/twitter/user/info", userName=user_name)

    def user_last_tweets(self, user_name, cursor=""):
        return self._get("/twitter/user/last_tweets", userName=user_name, cursor=cursor)

    def advanced_search(self, query, cursor=""):
        return self._get("/twitter/tweet/advanced_search", query=query, cursor=cursor)

    def iter_followers(self, user_name):
        cursor = ""
        while True:
            r = self._get("/twitter/user/followers", userName=user_name, cursor=cursor)
            for u in r.get("followers", []):
                yield u
            cursor = r.get("next_cursor") or ""
            if not cursor:
                return


if __name__ == "__main__":
    api = TwitterAPIIO()
    print(api.user_info("elonmusk"))
```

## Paginate through every tweet matching a query

```python
api = TwitterAPIIO()
query = 'from:elonmusk since:2025-01-01 until:2025-02-01'
total = 0
cursor = ""
while True:
    page = api.advanced_search(query, cursor=cursor)
    tweets = page.get("tweets", [])
    total += len(tweets)
    for t in tweets:
        print(t["id"], t["text"][:80])
    cursor = page.get("next_cursor") or ""
    if not cursor:
        break
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
    if (cursor) url.searchParams.set("cursor", cursor);
    const res = await fetch(url, { headers });
    if (!res.ok) throw new Error(`${res.status} ${await res.text()}`);
    const data = await res.json();
    for (const u of data.followers ?? []) yield u;
    cursor = data.next_cursor ?? "";
    if (!cursor) return;
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

cookie = login(os.environ["X_EMAIL"], os.environ["X_PASSWORD"], os.environ.get("X_TOTP"))
print(post_tweet(cookie, "Hello from twitterapi.io"))
```

**Persist the cookie** so you don't re-login every run — store it encrypted and re-login only on `401`.

## Real-time monitoring without polling

Instead of hitting `/user/last_tweets` on a timer:

1. `POST /twitter/filter_rules` with a rule like `from:elonmusk OR #AI`
2. Open the websocket endpoint shown in your dashboard
3. Receive matched tweets as they're posted

This is both cheaper and faster than polling, and avoids rate limits.
