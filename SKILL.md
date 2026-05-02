---
name: daily-social-auto
description: Conversational social media skill that starts by asking for an Instagram handle, analyzes the user's profile and brand voice across IG/FB/Threads/X, ingests user-uploaded image samples to learn visual style, plans a 30-day content calendar, and then runs daily automated execution at 08:00 Asia/Taipei. Each daily run reads the day's planned topic, fine-tunes it against current trends, generates IG-first content with carousel images via GPT-image-2 using the uploaded samples as reference (1080x1350, 5+ slides, handle watermark + page counter), and publishes IG carousel + same-topic FB long-form + Threads hot-take + X tweet/thread via Upload-Post API.
---

# daily-social-auto v3: 對話式 Onboarding × 30 天規劃 × 風格學習

四階段架構：

| 階段 | 何時跑 | 做什麼 |
|---|---|---|
| **Stage 0 — Onboarding** | 第一次安裝 | 對話式：問 IG 帳號 → 分析 → 上傳範本圖 → 規劃 30 天 |
| **Stage 1 — Profile Intelligence** | Stage 0 中段 | Apify 抓 4 平台 profile + 近期貼文 → LLM 產出 brand_voice.yaml |
| **Stage 2 — 30-Day Calendar** | Stage 0 末段 | 規劃 30 天主題大綱 → 產出 content_calendar.yaml |
| **Stage 3 — Daily Execution** | 每天 08:00 | 讀今天 calendar 主題 + 抓今天熱門微調 → 生內容 → IG 為主、FB/Threads/X 同主題另生 → 發布 |

## Prerequisites

- **OpenAI API key** (`OPENAI_API_KEY`) — GPT-image-2 + LLM 文案
- **Apify API token** (`APIFY_API_TOKEN`) — IG/FB/Threads/X 資料收集
- **Upload-Post API key** (`UPLOAD_POST_API_KEY`) — 4 平台發文
- **OpenClaw scheduler** — Stage 3 排程
- Python 3.10+ 與 Pillow

不需要：登入任何社群帳號、cookie、session token。

---

# Operating Modes（運作模式）

這個 Skill 設計成「**漸進降級**」，沒接齊全部 API 的使用者也能用到 70% 的功能。

| 模式 | 需要的 API | 能做到 | 不能做 |
|---|---|---|---|
| **A. Full Auto** | Apify + Upload-Post + OpenAI | 全自動、抓真實熱門、生圖、自動發 | — |
| **B. Plan + Generate** | OpenAI（含 gpt-image-2）| 規劃 30 天 + 生 4 平台文案 + 生 IG 圖；手動發 | 不抓即時熱門、不自動發 |
| **C. Plan + Text Only** | 任一個 LLM API（OpenAI 或 Claude）| 規劃 30 天 + 生 4 平台文案 + 圖片 prompt 文件；手動發 | 不抓熱門、不生圖、不自動發 |
| **D. Pure Planning** | 無（agent 直接用 LLM 做視覺推論）| 從範本+帳號名推測 brand_voice + 規劃 30 天大綱 | 其他全部 |

**判斷邏輯**：Stage 0 啟動時 agent 自動偵測環境變數，挑能執行的最高模式。**未接 Apify** 走視覺推論（看範本 + handle 推 brand_voice）；**未接 Upload-Post** 把 status 停在 `draft` 等使用者手動發；**未接 OpenAI** 不生圖、只產文案 + prompt 文件。

**降級時 agent 必須明確告知使用者**：

```
⚠️ 偵測到目前沒接 Apify。

Skill 會自動降級到「視覺推論模式」：
- ✅ 我會從你上傳的範本 + handle 推測 brand_voice
- ✅ 30 天 calendar 規劃照常
- ✅ 4 平台文案照常
- ❌ 不會抓你帳號真實近 30 篇貼文（推論可能不準）
- ❌ Stage 3 每天執行不會抓即時熱門（會直接用 calendar 主題）

要繼續嗎？或先去設定 APIFY_API_TOKEN 再回來？
```

依使用者選擇繼續或暫停。

---

# Stage 0 — Conversational Onboarding（一次性，對話式）

**觸發**：使用者第一次跟 agent 說「我要用 daily-social-auto」、「幫我設定」、「開始用」。

**核心原則**：**不要一次問完所有問題**，分多輪對話、邊做邊問，每一步都讓使用者看到產出再進下一步。

## Step 0.1: 問 IG 帳號（第一個問題、唯一必填）

Agent 第一句話：

```
我會幫你建立一個全自動的 IG 內容 pipeline。

開始之前，告訴我你的 IG 帳號名稱（例：@openclawtw 或 openclawtw）：
```

等使用者回答。**不要**同時問其他平台、niche、調性等等——這些 Stage 1 會自動分析出來。

收到 IG handle 後，agent 自動推測同名其他平台帳號（IG / Threads / FB / X 通常用同一個 handle），但**只先用 IG 跑分析**，避免一次太多問題：

```
收到 @openclawtw。

我準備分析你的 IG 帳號，這會：
- 用 Apify 抓你的 bio + 近 30 篇貼文
- LLM 分析調性、慣用詞、受眾、發文比例
- 產出 brand_voice.yaml

整個過程約 2-3 分鐘，不需登入。要繼續嗎？
```

確認後直接跑 Stage 1（IG-only 模式）。

## Step 0.2: 跑 Stage 1 IG 分析、回報結果

跑 Stage 1 IG-only（細節見下方 Stage 1 章節）。完成後 agent 給使用者**摘要**，不是把整份 yaml 貼出來：

```
✅ 分析完成。我看到的你是這樣：

📌 主題：tech / AI / 工具教學
🎯 受眾：25-40 工程師、PM、自由工作者
🗣️ 調性：工程師口吻、簡潔、有點嘲諷
✍️ 你常用的開頭：「今天看到一個有意思的」「我幫你測過了」
🚫 紅線：不碰政治、不業配不標、不誇大
📊 內容比例：工具實測 40% / AI 新聞 30% / 教學 20% / 觀察 10%

完整檔案存在：~/social-content/openclawtw/_config/brand_voice.yaml

這個分析準確嗎？
- 「準」→ 進下一步（FB/Threads/X 帳號 + 圖片範本）
- 「不準」→ 告訴我哪裡不對，我修
```

只在使用者確認後才進下一步。**brand_voice 不準的話後面 30 天規劃都會歪掉，這一步寧願多花 10 分鐘**。

## Step 0.3: 補抓 FB / Threads / X（如果有）

```
要不要把 FB / Threads / X 也加進分析？

這會讓我每天規劃 IG 主題時，同時幫你產出 FB 長文 + Threads hot-take + X 串文，全部對齊同一主題、不同口吻。

如果有，請給我帳號（同一個 handle 的話直接說「都一樣」）：
- FB 粉專名（粉專，不是個人帳號）：
- Threads handle：
- X handle：

如果沒有，回「跳過」，我只做 IG。
```

對使用者給的每個額外平台跑 Apify scraper、把資料 merge 進 brand_voice.yaml 的 `profiles` 區段。

## Step 0.4: 上傳圖片範本（風格學習的核心）

這是 v3 的關鍵新增：

```
接下來最關鍵的一步。

我用 GPT-image-2 生 IG 輪播圖，預設它什麼風格都會。但如果你已經有自己的視覺品牌（顏色、版型、字體感），請上傳 1-5 張代表性的範本圖。

範本圖會被「永久附在每次生圖請求」當參考，這樣 30 天裡每張圖都會像你既有的風格。

請上傳：
- 至少 1 張、最多 5 張
- 你既有作品中你最滿意的、或別人作品中你想模仿的
- 建議放至少一張「cover 範本」+ 一張「內頁範本」
- JPG/PNG 都可以、解析度不用太高（>800px 即可）

📤 直接拖檔案到對話框，或回「沒範本」我用預設風格生
```

**處理上傳（v3.1 加強版）**：

1. 把使用者上傳的檔案複製到 `~/social-content/{client}/_config/style_refs/`
2. 重新命名成 `ref_1.png` / `ref_1.jpg` 等（保留原副檔名）
3. 用 vision-capable LLM（GPT-4o vision 或 Claude vision）做**深度分析**，提取以下 7 類元素：

   **A. 配色系統**
   - `brand_color`：偵測主強調色（CTA / 標題第二行常用色）
   - `secondary_color`：偵測標題第一行 / 主視覺穩重色
   - `background_color`：偵測畫面整體背景色
   - **重要**：如果偵測到 ≥ 2 個強烈主色（雙主色品牌），都要記錄

   **B. Style DNA（5–12 條視覺特徵）**
   越具體越好。爛例：「現代乾淨」。好例：「奶油米色背景 #F5EFE6」「標題粗黑無襯線中文」「Q 版美少女金髮捲髮」。

   **C. 角色 IP 偵測（Character IP Detection）**
   如果範本圖含**重複出現的人物角色**（不是 stock photo 的人臉，是有設計感的插畫角色或特定真人代言人），把它標記為**品牌吉祥物**。這是 v3.1 新增的關鍵能力。

   ```yaml
   ig_branding:
     character_ip:
       has_character: true
       description: "金色長捲髮東方臉孔 Q 版插畫女性角色，常穿白色西裝外套 + 白色上衣 + 金色項鍊耳環"
       must_appear_in_every_slide: true   # 預設 true 若偵測到角色 IP
       reference_image: ref_3.png         # 哪張 ref 是純角色定義圖
       style_note: "插畫風（非寫實、非 3D），溫暖手繪感"
   ```

   生圖時 prompt 會強制要求「角色臉部、髮型、服裝必須延續 ref 的設計」，這樣每張圖都是「同一個人」。

   **D. 系列化偵測（Series Detection）**
   如果範本上有「N / M」格式的頁碼（例 `01 / 07`、`02 / 07`），代表這個帳號**用 N 集小系列在經營**。Agent 自動：

   - 從範本提取「分母」（例 07），記錄為 `series_template_length: 7`
   - Stage 2 規劃 30 天時改用「**系列模式**」：把 30 天切成 4 個 7 集小系列 + 月底 2 篇單篇 + wildcard
   - 每集自動編號 `01/07`、`02/07`...，cover 上方加系列名 header

   ```yaml
   ig_branding:
     series_mode:
       enabled: true
       template_length: 7                          # 從範本偵測
       header_text: "AI 自動化改變我的日常"        # 範本頂部那行小字
       header_color: "#E85A2B"                     # header 用的顏色
       page_counter_format: "{n} / {total}"        # "01 / 07" 而非 "1/5"
   ```

   如果**沒偵測到** N/M 格式，使用預設模式（30 天獨立題目 + wildcard）。

   **E. 文字排版規律**
   - 標題在畫面哪個位置（top / center / bottom）
   - 是否雙行雙色（例：上行深藍 + 下行橘紅）
   - 副標題顏色與字級
   - body 字級與顏色

   **F. 裝飾元素**
   - 幾何形狀（圓形 / 矩形 / 線條）
   - 圖示風格（線框 / 實心 / 漸層）
   - 點點 / 紋理 / 留白比例

   **G. CTA 與識別元素**
   - 底部是否有 CTA 按鈕（顏色、形狀、emoji）
   - 浮水印位置（左下 / 右下 / 不顯示）
   - 頭像是否與 handle 並排

把以上全部寫進 `brand_voice.yaml.ig_branding`：

```yaml
ig_branding:
  handle: "@shinya_shen"
  brand_color: "#E85A2B"
  secondary_color: "#1E3A5F"
  background_color: "#F5EFE6"
  font_family: sans-serif

  style_dna:
    - "奶油米色背景 #F5EFE6 為主"
    - "雙主色：深海軍藍 + 橘紅交錯使用"
    - "標題粗黑無襯線，雙行時上深藍下橘紅"
    - "Q 版美少女插畫主角"
    - "幾何抽象畫掛飾"
    - "頁碼右上 'N / M' 淺灰格式"
    - "左下圓形頭像 + @handle"
    - "底部橘紅膠囊 CTA + emoji"

  character_ip:
    has_character: true
    description: "金色長捲髮東方臉孔 Q 版插畫女性，白色西裝外套 + 白色上衣 + 金色配件"
    must_appear_in_every_slide: true
    reference_image: ref_3.png
    style_note: "插畫風（非寫實非 3D），溫暖手繪感"

  series_mode:
    enabled: true
    template_length: 7
    header_text: "AI 自動化改變我的日常"
    header_color: "#E85A2B"
    page_counter_format: "{n:02d} / {total:02d}"

  style_refs:
    - _config/style_refs/ref_1.png
    - _config/style_refs/ref_2.png
    - _config/style_refs/ref_3.png

  generation_mode: edit_with_refs
  ref_strength: closely_match
```

**為什麼兩種方法都做**：
- `style_dna`（文字）：給 prompt 工程用，讓 GPT-image-2 知道「目標風格」是什麼
- `style_refs`（圖）：每次生圖都附過去，讓模型「看著做」
- `character_ip`：強制每張圖都有同一個角色，這是品牌識別的核心
- `series_mode`：影響 Stage 2 規劃方式

四者並用最穩。詳細生圖流程見 Stage 3。

回報給使用者（依偵測結果動態調整）：

```
✅ 我看了你上傳的 3 張圖，提取的設計系統：

🎨 配色：橘紅 #E85A2B + 深藍 #1E3A5F + 米背景 #F5EFE6（雙主色品牌）
📐 版型：標題雙行雙色、頁碼右上、@handle 配頭像左下
👤 角色 IP：偵測到「金髮捲髮 Q 版美少女」是你的品牌吉祥物
        → 我會強制每張圖都有同一個角色（臉、髮、衣著一致）
📺 系列模式：偵測到 "02 / 07" 頁碼格式
        → 啟動「7 集小系列」規劃模式
        → 30 天 = 4 個 7 集系列 + 月底 2 篇單篇 + wildcard
🏷️ 系列頂部 header：「AI 自動化改變我的日常」（橘紅小字）

完整 style DNA 12 條已存入 brand_voice.yaml。

要進下一步（規劃 30 天）嗎？
```

要進下一步（規劃 30 天主題）嗎？
```

## Step 0.5: 規劃 30 天 Calendar

```
最後一步：規劃 30 天主題大綱。

我會根據你的：
- niche（tech/AI 工具教學）
- content_pillars 比例（實測 40% / 新聞 30% / 教學 20% / 觀察 10%）
- 你常發文的時段（已分析）

幫你排 30 天的主題、每天的角度、要查的關鍵字。
之後每天執行時，會用當天熱門微調這個大綱。

要開始嗎？預估 30 秒：
```

跑 Stage 2（細節見下方 Stage 2 章節）。完成後給摘要：

```
✅ 30 天大綱完成。

第 1 週主題（範例）：
- 5/3 (六)：Claude Opus 4.7 vs GPT-5 實測對比
- 5/4 (日)：5 個 ChatGPT 你沒用過的進階指令
- 5/5 (一)：n8n 自動化 email 回信教學
- 5/6 (二)：AI 工具訂閱費太貴？我這樣省一半
- 5/7 (三)：Cursor 新版 review：值得切過去嗎
- 5/8 (四)：[預留：抓當天最熱]
- 5/9 (五)：每週 AI 新聞精選 #1

完整 calendar：~/social-content/openclawtw/_config/content_calendar.yaml

排程已設定：每天 08:00（Asia/Taipei）自動執行第 N 天主題。

最後問：
- 「auto_publish: true」→ 全自動發布
- 「auto_publish: false」→ 我只生內容，你看了再決定發不發（建議前 7 天用這個）

你選哪個？
```

依使用者回答更新 `content_calendar.yaml` 並啟動排程：

```yaml
# ~/.openclaw/schedules.yaml 自動加入：
- skill: daily-social-auto
  cron: "0 8 * * *"
  timezone: "Asia/Taipei"
  args:
    client: openclawtw
    stage: 3
```

Stage 0 完成。後面使用者可以放著不管。

---

# Stage 1 — Profile Intelligence

**觸發**：Stage 0 Step 0.2、Step 0.3 自動呼叫，或使用者明確要求重跑。

**輸入**：1–4 個社群帳號 username
**輸出**：`~/social-content/{client}/_config/brand_voice.yaml`

## Apify scrapers（依平台）

對每個平台呼叫對應 Actor。**不需登入目標帳號**。完整 Actor ID、輸入 schema、成本見 `references/apify-actors.md`。

| 平台 | Actor ID | 抓什麼 |
|---|---|---|
| IG | `apify/instagram-profile-scraper` | bio + 近 30 篇貼文 + hashtags + likes/comments |
| Threads | `automation-lab/threads-scraper` (mode=posts) | bio + 近 30 篇 + replies/reposts |
| FB | `apify/facebook-pages-scraper` | 粉專 intro + 近 30 篇 + reactions |
| X | `apidojo/twitter-scraper-lite` | bio + 近 30 篇 tweet + engagement |

寫到 `~/social-content/{client}/_config/raw/`。

## LLM 分析

把所有 raw json 丟給 LLM，prompt：

```
分析以下 1-4 個平台的 profile + 近期貼文，產出 brand_voice.yaml。

任務：
1. niche — 主題範疇 + 5-15 個核心關鍵字
2. audience — 受眾畫像
3. tone — 文字調性
4. signature_phrases — 慣用開頭、結尾、emoji、自稱、禁用字
5. content_pillars — 通常發哪幾類內容比例
6. hashtag_pool — 每平台常用 hashtag
7. posting_pattern — 發文時段、互動最高時段
8. dont_do — 從歷史看絕對不碰的主題或語氣
9. ig_branding.brand_color — 從 IG 圖片色票偵測主色（若無 IG 跳過）

輸出嚴格 YAML 格式。
```

完整 schema 見 `references/brand-voice-schema.md`。

## 輸出 brand_voice.yaml

寫到 `~/social-content/{client}/_config/brand_voice.yaml`。Stage 0 Step 0.4 會繼續補上 `ig_branding.style_dna` 和 `style_refs`。

---

# Stage 2 — 30-Day Calendar Planning

**觸發**：Stage 0 Step 0.5 自動呼叫，或使用者要求「重排 30 天」。

**輸入**：`brand_voice.yaml`
**輸出**：`~/social-content/{client}/_config/content_calendar.yaml`

## 規劃邏輯

**先檢查 `brand_voice.ig_branding.series_mode.enabled`**，依此分流到兩種模式：

### Mode A：系列化規劃（series_mode.enabled = true）

當 Stage 0 從範本偵測到 N/M 集數標記，或使用者明確說「我要做系列」。

LLM prompt：

```
依據 brand_voice.yaml，幫使用者規劃未來 30 天 IG 內容。
這個帳號用「{template_length} 集小系列」格式經營。

任務：
1. 從明天起連續 30 天
2. 把 30 天切成 4 個「{template_length} 集小系列」+ 月底 2 篇單篇 + 4 wildcard
   - 每個系列要有清楚主題（例：「AI 自動化改變我的日常」、「AI 工具大評比」、「我的 AI 工作術」）
   - 每集 1 個主題，編號 01/{template_length}、02/{template_length}...
   - 系列 1 接續使用者既有編號（若 brand_voice 顯示 "已發到 02/07"，新規劃從 03/07 開始）
3. wildcard 安排在每個系列的「第 7 天位」之後（讓系列保持連貫）
4. 月底最後 2 篇做「月度復盤」+「下個月預告」
5. 每集要有：cover_hook、3 個 insight、CTA 方向、keywords
6. 主題對齊 content_pillars 比例（系列佔 70%、wildcard 13%、週報 / 月底復盤 17%）
7. 不觸碰 dont_do
8. 重要：每集標記 series_name + episode_number / total_episodes

輸出嚴格 YAML 格式。
```

### Mode B：傳統獨立題目（series_mode.enabled = false）

預設模式。每天一個獨立題目。

LLM prompt：

```
依據以下 brand_voice.yaml，幫使用者規劃未來 30 天的 IG 主軸內容。

規則：
1. 從明天開始（YYYY-MM-DD），連續 30 天
2. 每天一個主題；對齊 content_pillars 的比例分配
3. 主題要足夠具體可執行，例：「Claude Opus 4.7 實測對比 GPT-5」而非「AI 模型新聞」
4. 主題不能觸碰 dont_do
5. 每篇預留 IG cover hook、3 個 insight 點、CTA 方向
6. 每篇標記 keywords，給 Stage 3 用來抓當天熱門微調
7. 週末（六、日）放比較輕的主題，週一-五放重點實測 / 教學
8. 每隔 7 天留 1 天為 "wildcard"
9. 同一主題或工具不在 14 天內重複出現
10. 每週五排 "週報" 類型

輸出嚴格 YAML 格式（schema 見 references/content-calendar-schema.md）。
```

兩種模式輸出的 yaml schema 相同，差別只在 Mode A 多填 `series_name` / `episode_number` / `total_episodes` 三個欄位（見 `references/content-calendar-schema.md`）。

## 輸出 content_calendar.yaml

```yaml
client: openclawtw
generated_at: 2026-05-02T10:30:00+08:00
brand_voice_version: 1
period:
  start: 2026-05-03
  end: 2026-06-01
  total_days: 30

posts:
  - date: 2026-05-03
    day_of_week: sat
    pillar: 觀察               # 工具實測 / AI新聞 / 教學 / 觀察 / wildcard
    topic_title: "AI 工具訂閱費破萬了？我的省錢取捨"
    topic_slug: ai-subscription-cost-tradeoff
    angle: "從個人開銷角度討論工具堆疊取捨，引發共鳴"
    keywords: [AI訂閱, ChatGPT Plus, Claude Pro, 工具堆疊]
    cover_hook: "這個月 AI 工具帳單 $200，我砍了一半"
    insights:
      - "盤點：你真的每天都用嗎？"
      - "免費替代方案實測：哪些可以取代付費版"
      - "我的最終 stack（保留 + 砍掉）"
    cta_direction: "你的 AI 工具 stack 是什麼？留言告訴我"
    is_wildcard: false

  - date: 2026-05-04
    day_of_week: sun
    pillar: 教學
    topic_title: "5 個 ChatGPT 進階指令大多人沒用過"
    # ... 同樣結構

  # ... 共 30 篇

stats:
  pillar_distribution:
    工具實測: 12
    AI新聞: 9
    教學: 6
    觀察: 3
  wildcard_days: 4   # 第 8、15、22、29 天
  weekly_recap: 4    # 每週五

auto_publish: false   # Stage 0 Step 0.5 使用者選的
```

完整 schema 見 `references/content-calendar-schema.md`。

---

# Stage 3 — Daily Execution

**觸發**：OpenClaw scheduler 每天 08:00（Asia/Taipei）。**完全不需要使用者介入。**

## Step 3.1: 載入今天的計畫

```python
import yaml, datetime
today = datetime.date.today().isoformat()  # "2026-05-03"

with open(f"~/social-content/{client}/_config/content_calendar.yaml") as f:
    cal = yaml.safe_load(f)

today_post = next((p for p in cal["posts"] if p["date"] == today), None)

if not today_post:
    log("No plan for today — skipping. Run Stage 2 to extend calendar.")
    exit(0)

with open(f"~/social-content/{client}/_config/brand_voice.yaml") as f:
    brand = yaml.safe_load(f)
```

## Step 3.2: 抓當天熱門（微調用）

從 `today_post.keywords` 出發，用 Apify 抓今天 4 平台熱門。流程同 v2：

- IG：`apify/instagram-hashtag-scraper`，hashtags = today_post.keywords
- Threads：`automation-lab/threads-scraper` mode=search
- X：`apidojo/twitter-scraper-lite` searchTerms=today_post.keywords
- FB：`apify/facebook-pages-scraper`（使用者自訂的科技粉專清單）

挑選 1-3 篇相關度最高的當「今日最新 angle」。

## Step 3.3: 微調主題（不取代）

LLM prompt：

```
今天計畫主題：{today_post.topic_title}
原本 angle：{today_post.angle}
原本 cover hook：{today_post.cover_hook}
原本 insights：{today_post.insights}

今天剛抓到的相關熱門：
{recent_trending_posts}

任務：
- 主題 NOT change（除非完全脫鉤）
- 但用今天的熱門找一個「今天為什麼要講這個」的 timely angle
- 改寫 cover_hook，融入今日時事感
- insights 可以微調 1-2 點變更鮮明

輸出 final_post（同 today_post schema 但欄位被微調）。
```

特例：`is_wildcard: true` 的日子完全跑 v2 流程（不看 calendar，純抓熱門挑題）。

## Step 3.4: 生成 4 平台文案

IG 為主軸。FB/Threads/X 是「同主題不同說法」。Prompt 共用前綴帶入 brand_voice，後段每平台不同（細節見 v2 SKILL.md Step 2.4）。

關鍵差異 vs v2：

```
你正在執行 30 天 calendar 第 {day_n} 天的主題。
今天的 final_post（已微調過）：{final_post}

請產出 4 平台版本，IG 為主：
- IG carousel：5+ 張，cover hook 用 final_post.cover_hook
- FB：300-500 字長文，把同主題用 narrative 重講
- Threads：50-150 字 hot-take
- X：280 字 single tweet 或 3-5 串

4 個版本同主題、不同口吻、不能複製貼上。
全部對齊 brand_voice.tone。
```

## Step 3.5: 生成 IG 輪播圖（核心 — 風格學習）

**這是 v3 與 v2 最大的不同**。

### 3.5.1 GPT-image-2 multi-reference image edit

不用 `/v1/images/generations`，改用 `/v1/images/edits`，**把使用者上傳的範本圖每次都附過去**。

```python
import base64
import requests
from pathlib import Path

OPENAI_API_KEY = os.environ["OPENAI_API_KEY"]
CLIENT_DIR = Path(f"~/social-content/{client}").expanduser()

def load_style_refs() -> list:
    """讀 _config/style_refs/ 裡的 1-5 張範本圖，回傳 base64 list"""
    ref_dir = CLIENT_DIR / "_config" / "style_refs"
    refs = []
    for f in sorted(ref_dir.glob("ref_*.jpg")):
        refs.append(("image[]", (f.name, f.open("rb"), "image/jpeg")))
    return refs

def generate_slide_with_refs(prompt: str, output_path: Path, quality="medium"):
    """用 /v1/images/edits 生圖，附使用者範本當參考"""
    url = "https://api.openai.com/v1/images/edits"
    headers = {"Authorization": f"Bearer {OPENAI_API_KEY}"}

    # multipart/form-data
    files = load_style_refs()           # image[] 是 list of files
    data = {
        "model": "gpt-image-2",
        "prompt": prompt,
        "size": "1024x1536",            # portrait, 最接近 IG 4:5
        "quality": quality,             # medium 推薦
        "output_format": "jpeg",
        "output_compression": "92",
        "n": "1"
    }

    r = requests.post(url, headers=headers, data=data, files=files, timeout=180)
    r.raise_for_status()
    img_b64 = r.json()["data"][0]["b64_json"]
    output_path.write_bytes(base64.b64decode(img_b64))
```

### 3.5.2 Prompt 模板（附範本版）

對 GPT-image-2 明確標號參考圖：

```
You are creating Slide [N] of [TOTAL] for an Instagram carousel.

REFERENCE IMAGES (study these for visual style, layout, color, typography):
- Image 1: cover/headline reference from this brand
- Image 2: insight slide reference from this brand
- Image 3: (etc., depending on count — character IP reference is usually last)

YOUR TASK:
Create a NEW slide that closely matches the visual style, color palette,
typography hierarchy, layout grid, and decorative elements of the reference
images. Do NOT copy reference text — replace with the new text below.

NEW TEXT FOR THIS SLIDE (render exactly as written, no paraphrasing):
"""
{slide_text}
"""

STYLE DNA TO FOLLOW (extracted from refs):
{brand.ig_branding.style_dna joined with newlines}

{IF brand.ig_branding.character_ip.has_character == true:}
CHARACTER IP — CRITICAL (must appear in this slide):
Description: {brand.ig_branding.character_ip.description}
Style: {brand.ig_branding.character_ip.style_note}
Pose for this slide: {pose_for_this_slide}

The character's face, hair color/style, clothing, and overall design
MUST closely match Image {ref_index_of_character}. This is the brand's
mascot — consistency across all slides is non-negotiable.
{END IF}

{IF brand.ig_branding.series_mode.enabled == true:}
SERIES HEADER (top of slide, small text):
"{brand.ig_branding.series_mode.header_text}"
Color: {brand.ig_branding.series_mode.header_color}
Position: top-left, small (around 24-28pt)
{END IF}

LAYOUT CONSTRAINTS:
- Portrait 2:3, will be cropped to 4:5 later
- Text is the visual hero — large, immediately readable
- Maximum 3 lines of main text, max 15 words/line
- Reserve top-right 200x100px region clear of detail (page counter overlay)
- Reserve bottom-left 300x100px region clear of detail (handle watermark)
- Brand colors: primary {brand.ig_branding.brand_color}, secondary {brand.ig_branding.secondary_color}
- Background: {brand.ig_branding.background_color}

CRITICAL CONSTRAINTS:
- Match references' typography weight, alignment, and hierarchy
- Match references' use of whitespace and decorative elements
{IF NOT character_ip:}
- Do NOT include: people / hands / faces / generic AI humans
{END IF}
- Do NOT include: any text other than what's specified above
- Do NOT include: watermarks, page numbers, signatures (added in post-processing)

The output should look like it came from the same designer who made the references.
```

**Pose 規劃**：

LLM 在生成每張 slide 內容時，對含角色 IP 的帳號，**也要為每張 slide 設計一個合適的角色 pose**，例：

| Slide | Pose 範例 |
|---|---|
| 1 (Cover) | 角色坐辦公桌、看鏡頭、自信微笑、手邊有筆電 |
| 2 (Problem) | 角色看手機、皺眉、訊息泡泡圍繞 |
| 3 (Insight 1) | 角色站著拿咖啡、手指向資訊圖示 |
| 4 (Insight 2) | 角色坐沙發、放鬆、筆電在腿上 |
| 5 (CTA) | 角色看鏡頭、雙手比 OK 或火箭手勢 |

Pose 必須**動作不重複**、**情境符合該 slide 內容**、**穿著與表情風格延續 ref**。

### 3.5.3 後處理（resize + 浮水印）

跟 v2 完全一樣：
1. Pillow 等比 resize + 居中裁切：1024×1536 → 1080×1350
2. 蓋頁碼（右上 N/T）+ 帳號浮水印（左下 @handle）

完整 Pillow 程式碼見 `references/image-pipeline.md`。

### 3.5.4 預算與成本

GPT-image-2 用 `images.edits` + 多張參考圖時，**每次都會把全部 input 圖也算成 image input tokens**。
參考圖會增加 input tokens，建議先把高解析度圖縮小到任務需要的尺寸。

**因此**：使用者上傳的範本，agent **預先處理**過再存：

```python
def preprocess_uploaded_ref(uploaded_path, save_path):
    img = Image.open(uploaded_path).convert("RGB")
    # 縮到最長邊 1024，足夠模型理解風格、不浪費 token
    img.thumbnail((1024, 1024), Image.LANCZOS)
    img.save(save_path, "JPEG", quality=85)
```

成本估算（1024×1536 medium quality + 3 張 1024px 範本）：
- 每張 ~$0.07（比純生成的 $0.04 多 ~$0.03 給 input tokens）
- 一天 5 張 ~$0.35
- 一個月 ~$10.50

預算保護：每天 image 預算 $1，超過中止。

## Step 3.6: 存檔結構

```
~/social-content/{client}/
├── _config/
│   ├── brand_voice.yaml
│   ├── content_calendar.yaml
│   ├── style_refs/                    ← Stage 0 上傳的範本（永久）
│   │   ├── ref_1.jpg
│   │   ├── ref_2.jpg
│   │   └── ref_3.jpg
│   └── raw/...
└── 2026-05-03_ai-subscription-cost-tradeoff/
    ├── trend.md
    ├── prompts.md
    ├── images/
    │   ├── slide_1_cover.jpg
    │   ├── slide_2_context.jpg
    │   ├── slide_3_insight1.jpg
    │   ├── slide_4_insight2.jpg
    │   └── slide_5_cta.jpg
    ├── ig/post.md
    ├── fb/post.md
    ├── threads/post.md
    └── x/post.md
```

完整 schema 見 `references/folder-structure.md`。

## Step 3.7: 發布

依序呼叫 Upload-Post API：IG carousel → FB carousel → Threads carousel → X tweet/thread。
細節同 v2 SKILL.md Step 2.7。

## Step 3.8: 更新 calendar 與通知

```python
# 更新 content_calendar.yaml 的對應日期
today_post["status"] = "published"  # or "failed"
today_post["published_at"] = datetime.now().isoformat()
today_post["published_urls"] = {...}
save_yaml(cal, calendar_path)
```

並透過 OpenClaw 通知 channel（Telegram/Discord/email）發訊息給使用者：

```
📤 daily-social-auto 已執行：

主題：AI 工具訂閱費破萬了？我的省錢取捨
✅ IG 已發：https://www.instagram.com/p/...
✅ FB 已發：https://www.facebook.com/...
✅ Threads 已發：https://www.threads.net/...
✅ X 已發：https://x.com/...

明天主題：5 個 ChatGPT 進階指令大多人沒用過

完整檔案：~/social-content/openclawtw/2026-05-03_.../
```

---

## Calendar 維護

### 30 天結束後自動延期

排程在第 28 天自動跑一次「規劃下一個 30 天」（需要時擴 calendar），避免斷檔。

### 使用者主動修改

使用者可以隨時：
- 編輯 `content_calendar.yaml` 改主題、改順序、刪除日子
- 對 agent 說「明天的主題改成 XXX」→ agent 直接修改 yaml
- 對 agent 說「下週都跳過」→ agent 把 1-7 天 status 設為 `skipped`

### 重排規劃

「重新規劃 30 天」→ 重跑 Stage 2，舊 calendar 備份成 `content_calendar.{date}.bak.yaml`。

---

## Safety Rules

- **Stage 0 必須對話式進行**：絕對不能一次問完所有問題，每步等使用者確認
- **brand_voice.yaml 第一次必須使用者明確確認**：是後續一切的根
- **style_refs 是強制使用**：一旦上傳就每次生圖都附；使用者要清空或更換要明確下指令
- **content_calendar.yaml 使用者可隨時改**：agent 不偷偷改，除非使用者要求
- **Stage 3 全自動但有護欄**：
  - calendar 沒有今日條目 → skip + log
  - brand_voice / style_refs 缺失 → abort + 通知
  - 觸碰 dont_do 的 topic（即使在 calendar 裡）→ abort + 通知
  - GPT-image-2 連續失敗 3 次 → abort 該天，標 failed
- **永遠不偽造引言、統計、技術聲稱**
- **預算硬限**：每天 image 用量 ≤$1，每月 Apify 用量 ≤$30，超過 abort
- **wildcard 日子的 abort 後處理**：那天可以無聲 skip（沉默勝過垃圾）
