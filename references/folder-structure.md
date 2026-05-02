# Folder Structure Spec — daily-social-auto v3

## 整體結構

```
~/social-content/
└── {client}/
    ├── _config/                       ← Stage 0/1/2 產出，Stage 3 每天讀
    │   ├── brand_voice.yaml           ← Stage 1：品牌調性 + style_dna + style_refs 路徑
    │   ├── content_calendar.yaml      ← Stage 2：30 天主題大綱
    │   ├── style_refs/                ← Stage 0 上傳的範本圖（永久，每次生圖都附）
    │   │   ├── ref_1.jpg
    │   │   ├── ref_2.jpg
    │   │   └── ref_3.jpg
    │   └── raw/                       ← Stage 1 抓的 4 平台原始資料
    │       ├── ig_profile.json
    │       ├── ig_posts.json
    │       ├── threads_profile.json
    │       ├── threads_posts.json
    │       ├── fb_profile.json
    │       ├── fb_posts.json
    │       ├── x_profile.json
    │       └── x_posts.json
    │
    └── {YYYY-MM-DD}_{slug}/           ← Stage 3 每天產一個（30 天會有 30 個資料夾）
        ├── trend.md                   ← 微調報告：原 calendar 主題 + 今日熱門 angle
        ├── prompts.md                 ← 5+ 張 IG slide 的圖片 prompt
        ├── images/                    ← 5+ 張 1080x1350 IG 輪播圖
        │   ├── slide_1_cover.jpg
        │   ├── slide_2_context.jpg
        │   ├── slide_3_insight1.jpg
        │   ├── slide_4_insight2.jpg
        │   └── slide_5_cta.jpg
        ├── ig/post.md                 ← IG carousel 文案
        ├── fb/post.md                 ← FB 長文文案（同主題不同說法）
        ├── threads/post.md            ← Threads hot-take 文案
        └── x/post.md                  ← X tweet 或 thread 文案
```

## 關鍵設計原則

1. **`_config/` 是長期配置區**：Stage 1/2 產出後永久存在，Stage 3 每天讀取
2. **`style_refs/` 是視覺學習素材**：使用者上傳一次，之後每次生圖都當 input
3. **每日資料夾自包含**：trend、prompts、文案、圖全部在同一個 dated folder，方便審稿與歷史回查
4. **images 只放在每日資料夾根目錄**：FB/Threads/X 透過 Upload-Post API 引用 `images/` 路徑，不重複佔空間
5. **30 天累積後**：每月會有 ~30 個 dated folder + 1 個 `_config`

## 命名規則

### `{client}`
Upload-Post profile 名稱，全小寫無空白：`openclawtw`、`acmecorp`。

### `{YYYY-MM-DD}`
ISO 日期，使用 `Asia/Taipei` 時區的執行當天。

### `{slug}`
從 `content_calendar.yaml` 對應日期的 `topic_slug` 直接取用。kebab-case，最多 50 字元，限 `[a-z0-9-]`。

### style_refs 檔名
固定為 `ref_1.jpg`、`ref_2.jpg`...，依使用者上傳順序。重新上傳會清空舊的、重編號。

## 檔案模板

### `trend.md`（v3 — 微調報告）

```markdown
# Topic Refinement Report — 2026-05-03

## Calendar 原本規劃

- 主題: AI 工具訂閱費破萬了？我的省錢取捨
- Pillar: 觀察
- 原 angle: 從個人開銷角度討論工具堆疊取捨
- 原 cover_hook: 這個月 AI 工具帳單 $200，我砍了一半
- 原 keywords: [AI訂閱, ChatGPT Plus, Claude Pro, 工具堆疊, 訂閱費]

## 今日熱門掃描（4 平台）

### X
- @user1 / 2.3K likes / 6h ago: "..."
- @user2 / 1.1K likes / 8h ago: "..."

### Threads
- 2 posts referencing "ChatGPT Plus 漲價" — 集中在 12-18h 之間

### Instagram
- #AI訂閱 hashtag 24h 內 +18% 貼文量

### Facebook
- inside.com.tw 4h 前發文：「2026 AI 訂閱費全攻略」反應 600+

## 微調決定

**保留主題**：AI 工具訂閱費破萬了？我的省錢取捨

**新 angle**：剛好 ChatGPT Plus 傳出可能再漲，把這個熱門點接到原規劃的「省錢取捨」主軸

**新 cover_hook**：「ChatGPT Plus 又要漲？我這個月帳單破 $200，砍了一半」

**新 insight #1**（替換）：「先盤點：ChatGPT Plus / Claude Pro / Cursor，誰最該砍」

**保留 insight #2、#3**

**source URLs**：
- https://x.com/user1/status/...
- https://www.facebook.com/inside.com.tw/posts/...
```

### `ig/post.md`

```markdown
---
client: openclawtw
platform: instagram
post_type: carousel
slide_count: 5
date: 2026-05-03
slug: ai-subscription-cost-tradeoff
calendar_day: 1                     # 30 天裡的第幾天
pillar: 觀察
brand_voice_version: 1
calendar_planned_title: "AI 工具訂閱費破萬了？我的省錢取捨"
final_topic_title: "ChatGPT Plus 要漲了？我這個月 AI 帳單破 $200 怎麼砍"
trend_score: 0.78
source_urls:
  - https://x.com/...
  - https://www.facebook.com/inside.com.tw/posts/...
status: draft
published_at: null
published_url: null
error: null
---

# Caption

[實際的 IG caption 文字]

# Hashtags

#AI訂閱 #ChatGPT #生產力 #工具堆疊 #AInews
```

### `fb/post.md`、`threads/post.md`、`x/post.md`

同 v2 結構，新增 `calendar_day`、`calendar_planned_title`、`final_topic_title` 三欄。

## Status 生命週期

```
planned     ← Stage 2 規劃時寫進 content_calendar.yaml（不在 post.md）
  ↓
draft       ← Stage 3 寫每日 post.md 當下
  ↓
approved    ← 使用者審核通過（auto_publish=false 才會經過此步）
  ↓
published   ← Upload-Post 回 200，補上 published_url
```

或：

```
draft → failed   ← 任一步驟失敗，error 欄位記錄詳情
planned → skipped ← 使用者手動把該日 calendar 設為 skipped
```

## Shell 工具

### 看今天有沒有跑

```bash
ls ~/social-content/openclawtw/$(date +%Y-%m-%d)*
```

### 看本週發了幾篇

```bash
for d in $(seq 0 6); do
  date_str=$(date -d "$d days ago" +%Y-%m-%d)
  count=$(grep -l "status: published" \
    ~/social-content/openclawtw/${date_str}*/*/post.md 2>/dev/null | wc -l)
  echo "$date_str: $count published"
done
```

### 看 calendar 進度

```bash
cat ~/social-content/openclawtw/_config/content_calendar.yaml | \
  yq '.posts[] | "\(.date) [\(.status)] \(.topic_title)"'
```

### 重新檢視 style_refs

```bash
ls ~/social-content/openclawtw/_config/style_refs/
open ~/social-content/openclawtw/_config/style_refs/
```

### 清理 90 天前的每日資料夾（保留 _config）

```bash
find ~/social-content/openclawtw/ -maxdepth 1 -type d -name "20*" -mtime +90 -exec rm -rf {} \;
```

注意：`_config/` 不會被刪到（不符 `20*` pattern）。

## 多客戶情境

```
~/social-content/
├── openclawtw/
│   ├── _config/
│   │   ├── brand_voice.yaml
│   │   ├── content_calendar.yaml
│   │   └── style_refs/
│   └── 2026-05-03_.../
├── acmecorp/
│   ├── _config/
│   │   ├── brand_voice.yaml
│   │   ├── content_calendar.yaml
│   │   └── style_refs/
│   └── 2026-05-03_.../
```

每個 client 獨立的 `_config/` 包含自己的 brand_voice、calendar、style_refs。
排程錯開幾分鐘避免 Apify / OpenAI 撞牆。
