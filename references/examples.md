# twitterapi.io — Runnable examples

Every example assumes `TWITTERAPI_IO_KEY` is set in the environment.

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

    def _request(self, method, path, *, params=None, json=None):
        for attempt in range(3):
            r = self.s.request(method, f"{self.BASE}{path}",
                               params=params, json=json, timeout=self.timeout)
            if r.status_code == 429 or r.status_code >= 500:
                time.sleep(2 ** attempt)
                continue
            if not r.ok:
                body = r.json() if r.headers.get("content-type","").startswith("application/json") else {}
                detail = body.get("detail") or body.get("msg") or r.text
                raise requests.HTTPError(f"{r.status_code}: {detail}", response=r)
            return r.json()
        r.raise_for_status()

    def get(self, path, **params):
        return self._request("GET", path, params=params)

    def post(self, path, body=None):
        return self._request("POST", path, json=body)

    # ---- reads ----

    def user_info(self, user_name):
        # response: { status, msg, data: { id, userName, followers, following, ... } }
        return self.get("/twitter/user/info", userName=user_name)

    def user_last_tweets(self, user_name, cursor=""):
        return self.get("/twitter/user/last_tweets", userName=user_name, cursor=cursor)

    def advanced_search(self, query, cursor=""):
        # response: { tweets: [...], has_next_page, next_cursor }  (no data wrapper)
        return self.get("/twitter/tweet/advanced_search", query=query, cursor=cursor)

    def search_users(self, query):
        # NOTE: param is `query`, not `keyword`
        return self.get("/twitter/user/search", query=query)

    def tweets_by_ids(self, tweet_ids):
        # NOTE: param is snake_case `tweet_ids`, unlike /tweet/* subpaths
        ids = ",".join(tweet_ids) if isinstance(tweet_ids, (list, tuple)) else tweet_ids
        return self.get("/twitter/tweets", tweet_ids=ids)

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

    def balance(self):
        return self.get("/oapi/my/info")   # { recharge_credits, total_bonus_credits }


if __name__ == "__main__":
    api = TwitterAPIIO()
    info = api.user_info("elonmusk")["data"]
    print(f'{info["userName"]}: {info["followers"]:,} followers')
    print(f'Account balance: {api.balance()["recharge_credits"]}')
```

## Paginate every tweet matching a query

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
    cost_usd = count / 1000 * 0.15   # $0.15 per 1k followers
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

async function userInfo(userName) {
  return call("/twitter/user/info", { userName });
}

async function* iterFollowers(userName) {
  let cursor = "";
  while (true) {
    const data = await call("/twitter/user/followers", {
      userName, cursor, pageSize: 200,
    });
    for (const u of data.followers ?? []) yield u;
    if (!data.has_next_page) return;
    cursor = data.next_cursor ?? "";
  }
}

const info = (await userInfo("elonmusk")).data;
console.log(`${info.userName}: ${info.followers.toLocaleString()} followers`);
```

## Post a tweet — full login + write flow

```python
import os, requests, json

BASE = "https://api.twitterapi.io"
HEADERS = {
    "x-api-key": os.environ["TWITTERAPI_IO_KEY"],
    "Content-Type": "application/json",
}

def login(user_name, email, password, proxy, totp=None):
    body = {
        "user_name": user_name,
        "email":     email,
        "password":  password,
        "proxy":     proxy,
    }
    if totp:
        body["totp_secret"] = totp
    r = requests.post(f"{BASE}/twitter/user_login_v2", json=body, headers=HEADERS)
    r.raise_for_status()
    return r.json()["login_cookies"]

def post_tweet(cookies, proxy, text, reply_to=None, quoted=None, media_ids=None):
    body = {
        "login_cookies": cookies,
        "proxy":         proxy,
        "tweet_text":    text,            # NOTE: tweet_text, not text
    }
    if reply_to:  body["in_reply_to_tweet_id"] = reply_to
    if quoted:    body["quoted_tweet_id"] = quoted
    if media_ids: body["media_ids"] = media_ids
    r = requests.post(f"{BASE}/twitter/create_tweet_v2", json=body, headers=HEADERS)
    r.raise_for_status()
    return r.json()

def like_tweet(cookies, proxy, tweet_id):
    r = requests.post(f"{BASE}/twitter/like_tweet_v2",
                      json={"login_cookies": cookies, "proxy": proxy, "tweet_id": tweet_id},
                      headers=HEADERS)
    r.raise_for_status()
    return r.json()

def follow(cookies, proxy, user_id):
    r = requests.post(f"{BASE}/twitter/follow_user_v2",
                      json={"login_cookies": cookies, "proxy": proxy, "user_id": user_id},
                      headers=HEADERS)
    r.raise_for_status()
    return r.json()

def send_dm(cookies, proxy, user_id, text):
    r = requests.post(f"{BASE}/twitter/send_dm_to_user",
                      json={"login_cookies": cookies, "proxy": proxy,
                            "user_id": user_id, "text": text},   # text, not message
                      headers=HEADERS)
    r.raise_for_status()
    return r.json()


cookies = login(
    user_name = os.environ["X_USER"],
    email     = os.environ["X_EMAIL"],
    password  = os.environ["X_PASSWORD"],
    proxy     = os.environ["X_PROXY"],
    totp      = os.environ.get("X_TOTP"),
)
print(post_tweet(cookies, os.environ["X_PROXY"], "Hello from twitterapi.io"))
```

**Persist the cookie** so you don't re-login every run — store encrypted, refresh only on 401. **Keep using the same proxy as the one you logged in through** — switching proxies mid-session trips X's anti-bot.

## Real-time monitoring without polling

Instead of hitting `/user/last_tweets` on a timer:

1. Configure a webhook URL on your twitterapi.io dashboard
2. Create a server-side filter rule:

```python
r = requests.post(
    f"{BASE}/oapi/tweet_filter/add_rule",
    json={
        "tag": "ai-watchlist",
        "value": "from:elonmusk OR #AI OR @OpenAI",
        "interval_seconds": 60,
        "is_effect": 1,
    },
    headers=HEADERS,
)
print(r.json())  # { status, msg, rule_id }
```

3. Matched tweets stream to your webhook.

## User-level monitoring (alternative)

For "tell me when @someone tweets" (requires active monitoring subscription):

```python
# add
requests.post(f"{BASE}/oapi/x_user_stream/add_user_to_monitor_tweet",
              json={"user_id": "44196397"}, headers=HEADERS)

# list what's being monitored
requests.get(f"{BASE}/oapi/x_user_stream/get_user_to_monitor_tweet",
             headers=HEADERS).json()

# stop
requests.post(f"{BASE}/oapi/x_user_stream/remove_user_to_monitor_tweet",
              json={"user_id": "44196397"}, headers=HEADERS)
```
