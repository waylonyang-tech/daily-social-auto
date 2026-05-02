# brand_voice.yaml Schema — v3

Stage 1 產出、Stage 0 補上 `ig_branding.style_dna` 和 `style_refs`、Stage 3 每天讀取的核心檔案。

存放位置：`~/social-content/{client}/_config/brand_voice.yaml`

## 完整 schema

```yaml
# 識別
client: string
generated_at: ISO8601
last_reviewed: ISO8601
version: integer

# 你的社群帳號（Stage 1 抓到後填入）
profiles:
  instagram: string                  # @handle（必填，Stage 0 第一個問題就拿到）
  facebook: string | null            # 粉專 slug，沒有就 null
  threads: string | null
  x: string | null

# 主題範疇
niche: string
keywords: [string]

# 受眾
audience:
  age_range: string
  occupation: [string]
  pain_points: [string]
  language: string
  region: string

# 文字調性
tone:
  primary: string
  secondary: string
  energy_level: integer

# 招牌語言習慣
signature_phrases:
  openings: [string]
  closings: [string]
  emojis: [string]
  self_reference: string
  banned_words: [string]

# 內容比例（Stage 2 用此分配 30 天）
content_pillars:
  - type: string                     # 工具實測 | AI新聞 | 教學 | 觀察
    weight: float
    example: string

# 每平台 hashtag 池
hashtag_pool:
  instagram: [string]
  facebook: [string]
  threads: [string]
  x: [string]

# 發文時間分析
posting_pattern:
  best_times:
    - day: string
      hour: integer
  avg_post_count_per_week: integer
  avg_engagement_rate: float

# 紅線
dont_do: [string]

# === IG 視覺品牌（Stage 0 Step 0.4 從範本提取，永久使用）===
ig_branding:
  handle: string                     # @handle，必填，浮水印用
  brand_color: string                # 主強調色 hex，必填
  secondary_color: string            # 次主色 hex（雙主色品牌時用）
  background_color: string           # 背景色 hex
  font_family: string                # sans-serif | serif | mono（默認 sans-serif）
  watermark_position: string         # 預設 bottom-left
  page_counter_position: string      # 預設 top-right

  # === v3 風格學習 ===
  style_dna: [string]                # LLM 從範本提取的 5-12 條視覺特徵（越具體越好）
  style_refs: [string]               # 範本圖路徑（相對於 ~/social-content/{client}/）
  generation_mode: string            # generate | edit_with_refs
  ref_strength: string               # closely_match | inspired_by | precisely_replicate

  # === v3.1 角色 IP 偵測（範本含人物角色時填入）===
  character_ip:
    has_character: boolean           # true = 偵測到品牌吉祥物
    description: string              # 角色外觀描述（髮型/膚色/服裝/風格）
    must_appear_in_every_slide: bool # 預設 true
    reference_image: string          # 哪張 ref 是純角色定義圖（檔名）
    style_note: string               # 插畫風 / 寫實 / 3D 等

  # === v3.1 系列化偵測（範本有 N/M 頁碼時填入）===
  series_mode:
    enabled: boolean                 # true = 啟動系列規劃模式
    template_length: integer         # 從範本偵測到的分母（例 7、10）
    header_text: string              # 系列頂部 header 小字
    header_color: string             # header 顏色 hex
    page_counter_format: string      # 例 "{n:02d} / {total:02d}"

# 預算
budget:
  daily_image_usd_limit: float       # 預設 1.0
  monthly_openai_usd_limit: float    # 預設 50.0
  daily_apify_usd_limit: float       # 預設 1.0
  monthly_apify_usd_limit: float     # 預設 30.0
  on_exceed: string                  # skip | warn | hard_stop

# Stage 3 行為設定
auto_publish: boolean                # 預設 false
posts_per_day: integer               # 預設 1
min_score_threshold: float           # wildcard 日子用，預設 0.3
```

## 完整範例（科技 / AI / 工具教學頻道，含上傳範本）

```yaml
client: openclawtw
generated_at: 2026-05-02T10:00:00+08:00
last_reviewed: 2026-05-02T10:30:00+08:00
version: 1

profiles:
  instagram: "@openclawtw"
  facebook: "openclawtw"
  threads: "@openclawtw"
  x: "@openclawtw"

niche: tech-ai-tools
keywords:
  - AI
  - ChatGPT
  - Claude
  - LLM
  - 自動化
  - 工具教學
  - prompt
  - n8n
  - Cursor
  - workflow
  - MCP
  - agent

audience:
  age_range: "25-40"
  occupation:
    - 工程師
    - 產品經理
    - 數位行銷
    - 自由工作者
    - 新創團隊
  pain_points:
    - 想用 AI 但不知道從哪開始
    - 工具太多挑不出來
    - 想提升生產力但怕被取代
    - 看不懂英文技術文章
  language: zh-TW
  region: taiwan

tone:
  primary: 工程師口吻、簡潔、有點嘲諷、講重點不囉嗦
  secondary: 教學情境轉成耐心拆步驟、用具體範例
  energy_level: 6

signature_phrases:
  openings:
    - "今天看到一個有意思的"
    - "工程師都在用的"
    - "我幫你測過了"
    - "30 秒看懂"
  closings:
    - "你怎麼看？"
    - "留言告訴我你的工作流"
  emojis: ["🤖", "⚡", "🧠", "🔧", "👀", "📌"]
  self_reference: "我"
  banned_words:
    - 神器
    - 絕對
    - 最強
    - 無腦
    - 必看
    - 震驚
    - 流量密碼

content_pillars:
  - type: 工具實測
    weight: 0.4
    example: "我把 Cursor 跟 Claude Code 拿來蓋同一個 React 專案，結論是..."
  - type: AI新聞
    weight: 0.3
    example: "Claude Opus 4.7 出了，比 4.6 強在哪？三個重點"
  - type: 教學
    weight: 0.2
    example: "5 步驟用 n8n 串 GPT 自動回 email"
  - type: 觀察
    weight: 0.1
    example: "AI 工具訂閱費一個月加起來 $200 了，該怎麼省？"

hashtag_pool:
  instagram:
    - "#AI工具"
    - "#ChatGPT"
    - "#Claude"
    - "#工具教學"
    - "#生產力"
    - "#AInews"
    - "#prompt"
    - "#自動化"
    - "#工程師日常"
    - "#AI應用"
  facebook:
    - "#AI"
    - "#人工智慧"
    - "#工具教學"
    - "#自動化"
    - "#生產力"
  threads:
    - "#AI"
  x:
    - "#AI"
    - "#LLM"

posting_pattern:
  best_times:
    - day: weekday
      hour: 8
    - day: weekday
      hour: 21
    - day: weekend
      hour: 10
  avg_post_count_per_week: 5
  avg_engagement_rate: 4.2

dont_do:
  - 政治評論（左右派、選舉、政黨）
  - 業配但不標註
  - 過度誇大（用 banned_words 自動擋）
  - 沒實測過就推薦工具
  - 抄襲他人原 po 不註明來源
  - 個人生活流水帳

ig_branding:
  handle: "@openclawtw"
  brand_color: "#0EA5E9"
  secondary_color: "#FAFAFA"
  background_color: "#FAFAFA"
  font_family: sans-serif
  watermark_position: bottom-left
  page_counter_position: top-right

  # === v3：使用者上傳範本後自動填入 ===
  style_dna:
    - "極簡布局，大量負空間"
    - "粗黑無襯線標題置左，副標小一號靠下"
    - "亮藍色幾何重點裝飾（圓圈、線條）"
    - "純白或近白背景 #FAFAFA"
    - "字體層級清晰：標題 / 副標 / body 三層"
    - "標題佔畫面上方 1/3，視覺呼吸感強"
    - "重點數字或關鍵字用亮藍色強調"
  style_refs:
    - _config/style_refs/ref_1.jpg
    - _config/style_refs/ref_2.jpg
    - _config/style_refs/ref_3.jpg
  generation_mode: edit_with_refs
  ref_strength: closely_match

  # === v3.1 角色 IP（範本未含角色時 has_character: false 即可，其他欄位省略）===
  character_ip:
    has_character: false

  # === v3.1 系列模式（範本沒 N/M 頁碼時 enabled: false 即可）===
  series_mode:
    enabled: false

budget:
  daily_image_usd_limit: 1.0
  monthly_openai_usd_limit: 50.0
  daily_apify_usd_limit: 1.0
  monthly_apify_usd_limit: 30.0
  on_exceed: skip

auto_publish: false   # 前 7 天手動審
posts_per_day: 1
min_score_threshold: 0.3
```

## Validation 規則

Stage 3 啟動時 agent 會檢查：

- [ ] `client` 不為空
- [ ] `profiles.instagram` 以 `@` 開頭
- [ ] `ig_branding.handle` 以 `@` 開頭
- [ ] `ig_branding.brand_color` 是合法 hex
- [ ] `keywords` 至少 5 個
- [ ] `content_pillars` 各 weight 加總在 0.95–1.05 之間
- [ ] `dont_do` 至少 3 條
- [ ] **若 `generation_mode: edit_with_refs`**：
  - [ ] `style_refs` 至少 1 個
  - [ ] `style_refs` 對應的檔案實際存在於 `_config/style_refs/`
  - [ ] `style_dna` 至少 3 條
- [ ] `budget.daily_image_usd_limit` > 0

任一不過 → Stage 3 abort + 通知使用者。

## 修改流程

### 改文字內容
直接 vim 編輯 yaml，agent 不會自動覆寫，除非使用者明確要求重跑 Stage 1。

### 改 style_refs

**清空重來**：

```
agent，清空 style_refs，我要重傳
```

→ agent 刪掉 `style_refs/` 內所有 jpg + 移除 yaml 裡的 `style_refs` / `style_dna` / 把 `generation_mode` 改回 `generate`

**再加幾張**：

```
agent，加幾張範本進去
```

→ agent 等使用者上傳 → 接續編號 ref_4.jpg, ref_5.jpg → 重跑 LLM style_dna 提取

### 改 ref 強度

```yaml
ig_branding:
  ref_strength: inspired_by   # 改成 inspired_by 讓 GPT-image-2 自由度更高
```

或讓使用者直接對 agent 說「讓圖更自由發揮一點」、「讓圖更貼近範本」，agent 改 yaml。

### 重跑 Stage 1

```bash
openclaw run-skill daily-social-auto --client openclawtw --stage 1 --force
```

會覆蓋 brand_voice.yaml，但 `ig_branding.style_dna` / `style_refs` / `generation_mode` 會保留（這些是 Stage 0 後續加的）。

---

## 進階範例：角色 IP + 系列模式（@shinya_shen 真實案例）

當使用者上傳的範本含有重複出現的插畫角色 + N/M 頁碼格式，Stage 0 會自動偵測並填入完整 schema：

```yaml
client: shinya_shen
generated_at: 2026-05-02T12:00:00+08:00
version: 1

profiles:
  instagram: "@shinya_shen"

niche: ai-automation-productivity
keywords:
  - AI 自動化
  - n8n
  - ChatGPT
  - 工具教學
  - 生產力
  - workflow

audience:
  age_range: "25-35"
  occupation: [上班族, 行銷人員, 新創團隊, 自由工作者]
  pain_points:
    - 工作太多時間不夠用
    - 想學 AI 但怕技術門檻
  language: zh-TW
  region: taiwan

tone:
  primary: 溫柔親切、第一人稱分享、強調「親身實測」
  secondary: 教學情境拆解清楚、用具體數字佐證
  energy_level: 5

signature_phrases:
  openings: ["我用 AI", "我幫你測過了", "從早晨到晚上", "再也不用"]
  closings: ["開始你的效率革命", "你也可以做到"]
  emojis: ["🚀", "✨", "💡", "☕"]
  self_reference: "我"
  banned_words: [神器, 絕對, 最強, 無腦, 必看, 暴賺]

content_pillars:
  - { type: AI 自動化系列, weight: 0.5 }
  - { type: 工具教學, weight: 0.3 }
  - { type: 個人心得, weight: 0.15 }
  - { type: 數據佐證, weight: 0.05 }

ig_branding:
  handle: "@shinya_shen"
  brand_color: "#E85A2B"          # 橘紅（CTA 與強調）
  secondary_color: "#1E3A5F"      # 深海軍藍（標題穩重）
  background_color: "#F5EFE6"     # 奶油米
  font_family: sans-serif
  watermark_position: bottom-left
  page_counter_position: top-right

  style_dna:
    - "奶油米色背景 #F5EFE6 為主"
    - "雙主色：深海軍藍 + 橘紅交錯使用"
    - "標題粗黑無襯線中文，雙行時上深藍下橘紅"
    - "Q 版美少女插畫主角為核心識別"
    - "幾何抽象畫掛飾（深藍方塊 + 橘紅圓 + 米色色塊）"
    - "頁碼右上 'NN / NN' 淺灰細字"
    - "左下圓形頭像 + @handle 並排"
    - "底部 CTA 橘紅膠囊 + 火箭 emoji + 白色粗字"
    - "資訊圖示：線框矩形深藍邊 + 橘紅內字"
    - "標頭橘紅小字：系列名"
    - "副標題深灰細體，置左對齊"
    - "整體插畫風（非寫實非 3D），溫暖手繪感"
  style_refs:
    - _config/style_refs/ref_1.png
    - _config/style_refs/ref_2.png
    - _config/style_refs/ref_3.png
  generation_mode: edit_with_refs
  ref_strength: closely_match

  # === 角色 IP（v3.1 關鍵能力）===
  character_ip:
    has_character: true
    description: "金色長捲髮東方臉孔 Q 版插畫女性角色，常穿白色西裝外套 + 白色上衣 + 金色項鍊耳環"
    must_appear_in_every_slide: true
    reference_image: ref_3.png      # 純角色定義圖
    style_note: "插畫風（非寫實非 3D），溫暖手繪感，臉部表情溫柔自信"

  # === 系列模式（v3.1 關鍵能力）===
  series_mode:
    enabled: true
    template_length: 7              # 偵測到 "01 / 07" 頁碼
    header_text: "AI 自動化改變我的日常"
    header_color: "#E85A2B"
    page_counter_format: "{n:02d} / {total:02d}"

dont_do:
  - 政治評論
  - 業配但不標註
  - 過度誇大
  - 沒實測就推薦工具
  - 純技術硬內容（要有「我」的角度）

budget:
  daily_image_usd_limit: 1.0
  monthly_openai_usd_limit: 50.0
  on_exceed: skip

auto_publish: false
posts_per_day: 1
```

這個案例展示 v3.1 兩大新能力如何同時運作：

1. **`character_ip.must_appear_in_every_slide: true`** → Stage 3 每張 slide 的 prompt 都加入「角色臉、髮、衣著必須延續 ref_3」的硬約束；agent 為每張 slide 設計不同 pose（坐辦公桌、看手機、拿咖啡、比手勢...）但保持同一角色

2. **`series_mode.enabled: true`** → Stage 2 規劃 30 天時改走系列模式：4 個 7 集系列 + 月底 2 篇單篇 + 4 wildcard。每集自動編號、cover 上方加系列 header。
