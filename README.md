# daily-social-auto v3.1

> OpenClaw Skill：對話式 onboarding（問 IG → 分析 → 上傳範本 → 規劃 30 天）→ 每天 08:00 自動執行當日主題、生成多平台貼文、自動發布

## 一句話說明

你只要回答一個問題（IG 帳號是什麼）+ 上傳 1-5 張你滿意的圖片範本，
之後 **30 天的 IG / FB / Threads / X 貼文都會自動產生並發布**，每張圖都像你既有風格。

## ⚡ 不用接齊全部 API 也能用

Skill 有 4 種運作模式，自動偵測你的環境變數降級：

| 模式 | 需要的 API | 能做到 |
|---|---|---|
| **A. Full Auto** | Apify + Upload-Post + OpenAI | 全自動、抓真實熱門、生圖、自動發 |
| **B. Plan + Generate** | OpenAI 即可 | 規劃 30 天 + 生 4 平台文案 + 生 IG 圖；手動發 |
| **C. Plan + Text Only** | 任一 LLM | 規劃 30 天 + 生 4 平台文案 + 圖片 prompt 文件 |
| **D. Pure Planning** | 無 | 視覺推論（從範本+handle 推測）+ 規劃 30 天大綱 |

第一次用建議 D → C → B → A 漸進升級，每步都看得到產出再加錢接下一個 API。

## 四階段流程

### Stage 0 — 對話式 Onboarding（一次性，~10 分鐘）

像跟編輯團隊開會：

1. Agent 問：「你的 IG 帳號是？」 → 你回答
2. Agent 跑分析，給你看「我看到的你」摘要 → 你確認
3. Agent 問：「FB / Threads / X 也要一起嗎？」（選配）
4. **Agent 問：「上傳 1-5 張你滿意的設計範本」** ← 風格學習關鍵
5. Agent 規劃 30 天主題大綱 → 你確認
6. 啟動排程，搞定

### Stage 1 — Profile Intelligence

Apify 抓你 4 個社群帳號的 bio + 近 30 篇貼文 → LLM 分析 → 產出 `brand_voice.yaml`（你的調性、慣用詞、受眾、紅線、發文比例）

### Stage 2 — 30-Day Content Calendar

依據 `brand_voice.yaml` 產出 30 天主題大綱，按你的內容比例分配（例：12 天工具實測、9 天 AI 新聞、6 天教學、3 天觀察、4 天 wildcard），週一-五排重點、週末輕量、週五排週報。

### Stage 3 — 每日 08:00 全自動執行

OpenClaw scheduler 觸發，**完全不用你介入**：

1. 讀今天 calendar 的主題
2. Apify 抓今天 4 平台熱門
3. 用今日熱門「微調」原規劃（不取代主題，只調 angle 和 cover hook）
4. 生 4 平台文案（IG 為主，FB/Threads/X 同主題另生說法）
5. **GPT-image-2 生 5 張 IG 輪播圖，每張都附你的範本當參考** ← 風格一致性關鍵
6. Pillow 蓋頁碼 + @handle 浮水印
7. Upload-Post API 發 4 平台
8. 通知你：「今天發了什麼、明天要發什麼」

## v3 vs v2 vs seo-to-social

| | seo-to-social | v2 | **v3** |
|---|---|---|---|
| 內容來源 | SEO 文章 | 每日熱門 | **30 天 calendar + 每日熱門微調** |
| Onboarding | 手動填表 | 一次性指令 | **對話式、邊做邊問** |
| 個人化 | 模板 | brand_voice 分析 | **brand_voice + 上傳範本學習** |
| 圖片風格 | 隨機 AI | 固定 5 種模板 | **GPT-image-2 看範本生圖，每張都像你** |
| 觸發 | 手動 | 每日 cron | **每日 cron + 28 天前自動延期** |
| 平台範圍 | 10 個 | 4 個 | **IG 主 + FB/Threads/X 同步** |

## 三個關鍵能力（這是跟其他 social skill 最大差別）

### 1. 風格學習（v3）

**你上傳一次範本** → 永久存在 `_config/style_refs/` →
**未來 30 天每張生圖請求都會把範本附過去** →
GPT-image-2 看著範本生新內容，每張圖視覺都像你既有的設計系統。

支援：
- 1–5 張範本（推薦 2–3 張）
- 自動縮到 1024px max edge（省 input token，不影響風格學習）
- LLM 看範本同時提取「style DNA」文字描述塞進 prompt
- 三段強度可調：`closely_match` / `inspired_by` / `precisely_replicate`

### 2. 角色 IP 偵測（v3.1 新增）

如果你的範本有**重複出現的人物角色**（插畫吉祥物或定裝設計），Stage 0 自動偵測為「品牌吉祥物」並標記 `must_appear_in_every_slide: true`。

之後：
- 每張 slide 的 prompt 都強制要求「臉、髮、衣著延續 ref」
- LLM 為每張 slide 設計不同 pose（坐辦公桌 / 看手機 / 拿咖啡 / 比手勢）但保持同一角色
- 30 天 150 張圖看起來像同一個人在演出，不是 stock photo 拼貼

### 3. 系列化規劃（v3.1 新增）

如果你的範本上有 `01 / 07`、`02 / 07` 這種 N/M 頁碼格式，Stage 0 自動偵測為「系列經營者」，把 30 天規劃**從每天獨立題目** 切換到 **4 個 7 集小系列 + 月底復盤**：

```
週 1：「AI 自動化改變我的日常」01-07/07
週 2：「AI 工具大評比」01-07/07
週 3+4：「我的 AI 工作術」01-07/07 + 月底復盤
```

每集自動編號、cover 上方加系列名 header，配合你已開始的編號接續推進（例：你已發到 02/07，新規劃從 03/07 開始）。

詳情見 `references/image-pipeline.md` 與 `references/brand-voice-schema.md`。

## 安裝（簡略版）

完整步驟見 `INSTALL.md`。

```bash
# 1. clone
git clone https://github.com/waylonyang-tech/daily-social-auto.git \
  ~/.openclaw/skills/daily-social-auto

# 2. 申請 3 個服務 + 設環境變數
cat >> ~/.openclaw/.env <<'EOF'
APIFY_API_TOKEN=apify_api_xxxxx
UPLOAD_POST_API_KEY=up_xxxxx
OPENAI_API_KEY=sk-xxxxx
EOF

# 3. Python 依賴與字體
pip3 install --user Pillow requests pyyaml
mkdir -p ~/.openclaw/skills/daily-social-auto/fonts
curl -L -o ~/.openclaw/skills/daily-social-auto/fonts/Inter-Bold.ttf \
  https://github.com/rsms/inter/raw/master/docs/font-files/Inter-Bold.ttf

# 4. 重啟 OpenClaw
openclaw gateway restart
```

## 第一次使用

對 OpenClaw agent（Telegram/Discord/Web）說：

```
我要用 daily-social-auto，幫我設定
```

Agent 會自動進入 Stage 0 對話流程。**你不需要記任何指令、不需要看 SKILL.md**，跟著 agent 的問題回答就好。

整個過程 ~10 分鐘：
- 問 IG 帳號（30 秒）
- 分析 + 你確認（3 分鐘）
- 上傳範本（2 分鐘）
- 規劃 30 天（1 分鐘）
- 你掃過 calendar 確認（3 分鐘）

之後就每天等通知就好。

## 檔案結構

```
daily-social-auto/
├── README.md                              ← 本文件
├── INSTALL.md                             ← 完整安裝指南
├── SKILL.md                               ← OpenClaw 主指令（4 階段邏輯）
├── LICENSE                                ← MIT
├── .gitignore
└── references/
    ├── apify-actors.md                    ← 4 平台 Apify Actor 清單
    ├── brand-voice-schema.md              ← brand_voice.yaml schema
    ├── content-calendar-schema.md         ← content_calendar.yaml schema
    ├── image-pipeline.md                  ← GPT-image-2 + style refs 完整流程
    └── folder-structure.md                ← 產出資料夾結構
```

## 預估月成本

| 項目 | 月成本 |
|---|---|
| Apify Starter | $29（免費 $5 也行但會撞限）|
| Upload-Post Starter（100 次/月）| $16 |
| GPT-image-2（每天 5 張 × 1024×1536 medium + 3 張 ref）| ~$10.50 |
| LLM 文案生成（GPT-4o 或 Claude）| ~$5 |
| **總計** | **約 $60/月** |

一個 client 每天發一次的成本。多 client 看比例增加（Apify / Upload-Post 額度可共用）。

## 維護成本

設定完之後幾乎為零：

- 每天早上看一下通知（10 秒）
- 每週檢查一次 calendar（5 分鐘）
- 第 28 天 agent 自動規劃下個 30 天，你只要審核（5 分鐘）
- 想換視覺風格 → 重傳 style_refs（1 分鐘）

## License

MIT
