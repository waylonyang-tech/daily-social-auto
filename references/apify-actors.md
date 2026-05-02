# Apify Actors Reference — daily-social-auto v3

完整列出 Skill 用到的 Apify Actor、輸入 schema、典型成本，方便 agent 直接呼叫。

> 這個 Skill 故意只用 Apify 做資料收集，因為：
> - 不需登入任何社群帳號
> - 不需維護 cookie / session / fingerprint
> - 一個 API token 解決 IG/FB/Threads/X 全部
> - 每月 $5 免費額度，付費 $29/月 Starter 通常用不完

## Actor 總覽

| 用途 | Actor ID | 哪個 Stage 用 | 典型成本 |
|---|---|---|---|
| IG profile + 近期貼文 | `apify/instagram-profile-scraper` | Stage 1 | $0.0023/profile + $0.0005/post |
| IG 熱門 hashtag | `apify/instagram-hashtag-scraper` | Stage 3 | ~$0.005/hashtag |
| Threads profile + posts | `automation-lab/threads-scraper` | Stage 1, 3 | $0.01/run + $0.003/post |
| Threads search | `automation-lab/threads-scraper` mode=search | Stage 3 | ~$0.075/搜尋 |
| FB 粉專 + 貼文 | `apify/facebook-pages-scraper` | Stage 1, 3 | ~$0.005/post |
| X profile + posts | `apidojo/twitter-scraper-lite` | Stage 1 | ~$0.001/post |
| X 關鍵字搜尋 | `apidojo/twitter-scraper-lite` searchTerms | Stage 3 | ~$0.001/post |

每天 Stage 3 跑一次的總成本估算：4 平台 × 30 篇候選 ≈ **$0.10–0.30 / 天**。
加 Stage 1 一次性：~$1。
30 天總計約 **$5–10**（實際看 client 數量）。

---

## Stage 1 用（一次性，分析使用者自己的帳號）

### IG Profile Scraper

```bash
curl -X POST "https://api.apify.com/v2/acts/apify~instagram-profile-scraper/run-sync-get-dataset-items" \
  -H "Authorization: Bearer $APIFY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "usernames": ["openclawtw"],
    "resultsLimit": 30,
    "addParentData": false
  }'
```

抽取欄位：`username`, `fullName`, `biography`, `followersCount`, `followsCount`, `postsCount`, `externalUrl`, `latestPosts[].caption`, `latestPosts[].hashtags`, `latestPosts[].likesCount`, `latestPosts[].commentsCount`, `latestPosts[].timestamp`.

### Threads Scraper

```bash
curl -X POST "https://api.apify.com/v2/acts/automation-lab~threads-scraper/run-sync-get-dataset-items" \
  -H "Authorization: Bearer $APIFY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "usernames": ["openclawtw"],
    "mode": "posts",
    "maxPosts": 30
  }'
```

### Facebook Pages Scraper（限公開粉專）

```bash
curl -X POST "https://api.apify.com/v2/acts/apify~facebook-pages-scraper/run-sync-get-dataset-items" \
  -H "Authorization: Bearer $APIFY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "startUrls": [{"url": "https://www.facebook.com/openclawtw"}],
    "resultsLimit": 30
  }'
```

> 注意：個人 FB 帳號（非粉專）不能抓。沒粉專的話 Stage 0 跳過 FB 分析。

### X Scraper Lite

```bash
curl -X POST "https://api.apify.com/v2/acts/apidojo~twitter-scraper-lite/run-sync-get-dataset-items" \
  -H "Authorization: Bearer $APIFY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "twitterHandles": ["openclawtw"],
    "maxItems": 30
  }'
```

---

## Stage 3 用（每天，抓今日熱門做主題微調）

只在 calendar 該日**不是 wildcard** 時跑。
wildcard 日子直接走 v2 純熱門挑題流程。

### IG Hashtag Trending

```bash
curl -X POST "https://api.apify.com/v2/acts/apify~instagram-hashtag-scraper/run-sync-get-dataset-items" \
  -H "Authorization: Bearer $APIFY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "hashtags": ["AI訂閱", "ChatGPT Plus", "Claude Pro"],
    "resultsLimit": 30
  }'
```

`hashtags` 從 `content_calendar.posts[today].keywords` 動態帶入。

### Threads Search

```bash
curl -X POST "https://api.apify.com/v2/acts/automation-lab~threads-scraper/run-sync-get-dataset-items" \
  -H "Authorization: Bearer $APIFY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "searchKeywords": ["AI 訂閱費", "ChatGPT Plus"],
    "mode": "search",
    "maxPosts": 30
  }'
```

### FB 科技粉專最近 24h

```bash
curl -X POST "https://api.apify.com/v2/acts/apify~facebook-pages-scraper/run-sync-get-dataset-items" \
  -H "Authorization: Bearer $APIFY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "startUrls": [
      {"url": "https://www.facebook.com/TechCrunch"},
      {"url": "https://www.facebook.com/inside.com.tw"},
      {"url": "https://www.facebook.com/iThome.NEWS"}
    ],
    "resultsLimit": 10
  }'
```

> FB 粉專清單放在 `brand_voice.yaml.fb_monitored_pages`（選填）。沒有就跳過 FB 來源。

### X Keyword Search

```bash
curl -X POST "https://api.apify.com/v2/acts/apidojo~twitter-scraper-lite/run-sync-get-dataset-items" \
  -H "Authorization: Bearer $APIFY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "searchTerms": ["AI subscription", "ChatGPT Plus pricing"],
    "maxItems": 30,
    "minRetweets": 50,
    "since": "2026-05-02"
  }'
```

---

## Python 呼叫範本

```python
import requests
import os

APIFY_TOKEN = os.environ["APIFY_API_TOKEN"]

def run_apify_actor(actor_id: str, input_data: dict, timeout: int = 120) -> list:
    url = f"https://api.apify.com/v2/acts/{actor_id}/run-sync-get-dataset-items"
    headers = {
        "Authorization": f"Bearer {APIFY_TOKEN}",
        "Content-Type": "application/json"
    }
    r = requests.post(url, json=input_data, headers=headers, timeout=timeout)
    r.raise_for_status()
    return r.json()

# Stage 1: 分析使用者自己 IG
my_ig = run_apify_actor(
    "apify~instagram-profile-scraper",
    {"usernames": ["openclawtw"], "resultsLimit": 30}
)

# Stage 3: 抓今天熱門 hashtag
trending = run_apify_actor(
    "apify~instagram-hashtag-scraper",
    {"hashtags": ["AI訂閱", "ChatGPT Plus"], "resultsLimit": 30}
)
```

> Actor ID 在 URL 用 `~` 取代 `/`。

## 預算保護

`brand_voice.yaml.budget`：

```yaml
budget:
  daily_apify_usd_limit: 1.0
  monthly_apify_usd_limit: 30.0
```

每天 Stage 3 開始前先檢查月度 Apify 用量：

```python
def check_apify_budget(monthly_limit_usd=30.0) -> bool:
    r = requests.get(
        "https://api.apify.com/v2/users/me/usage/monthly",
        headers={"Authorization": f"Bearer {APIFY_TOKEN}"}
    )
    used = r.json()["data"]["usageCycleUsdSpend"]
    return used < monthly_limit_usd
```

超過 → abort 當天並通知。
