# INSTALL — daily-social-auto v3

從零到「每天 08:00 自動發 4 平台 30 天計畫內容」的完整步驟。
預估 30 分鐘安裝 + 10 分鐘對話 onboarding。

## Phase 0：先確認

- [ ] OpenClaw 已安裝、gateway 跑得起來
- [ ] 已裝 git、curl、Python 3.10+、pip
- [ ] IG 帳號（必填）
- [ ] FB 粉專、Threads、X 帳號（選配，越多越好分析）
- [ ] 你滿意的 1-5 張既有設計範本（既有作品或想模仿的別人作品）
- [ ] 一張信用卡

## Phase 1：申請 3 個服務

### 1.1 Apify（資料收集）

1. <https://apify.com> 註冊
2. <https://console.apify.com/account/integrations> 拿 API token
3. 免費 $5/月 額度可以跑完 Stage 1 + 一週 Stage 3，建議 Starter $29/月

### 1.2 Upload-Post（發文管道）

1. <https://upload-post.com> 註冊
2. <https://app.upload-post.com/api-keys> 拿 API key
3. <https://app.upload-post.com/manage-users> 建一個 profile（例：`openclawtw`），串你的 IG / FB / Threads / X 帳號
4. 免費 10 次/月，付費 Starter $16/月 100 次

### 1.3 OpenAI（GPT-image-2 + LLM）

1. <https://platform.openai.com/api-keys> 申請 API key
2. 確認你的 organization 已通過 verification（GPT-image-2 需要）
3. 確認帳戶有儲值（2025 後新帳號沒有免費額度）

**為什麼用 GPT-image-2**：
- 對「文字渲染 + 排版設計」是 SOTA
- 支援 multi-reference image edit（v3 風格學習關鍵）
- 跟 LLM 共用同一把 OpenAI key

**成本參考**（1024×1536 + 3 張 ref + medium quality）：
- 每張 ~$0.07
- 一天 5 張 ~$0.35
- 一個月 ~$10.50

## Phase 2：安裝 Skill

```bash
git clone https://github.com/waylonyang-tech/daily-social-auto.git \
  ~/.openclaw/skills/daily-social-auto

ls ~/.openclaw/skills/daily-social-auto/
# 應該看到：README.md INSTALL.md SKILL.md LICENSE .gitignore references/

openclaw gateway restart
```

## Phase 3：設定環境變數

```bash
cat >> ~/.openclaw/.env <<'EOF'

# === daily-social-auto ===
APIFY_API_TOKEN=apify_api_xxxxxxxxxxxxxxxxx
UPLOAD_POST_API_KEY=up_xxxxxxxxxxxxxxxx
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxx
EOF

openclaw gateway restart
```

驗證：

```bash
openclaw env show | grep -E "APIFY|UPLOAD|OPENAI"
```

## Phase 4：Python 依賴 + 字體

```bash
pip3 install --user Pillow requests pyyaml

mkdir -p ~/.openclaw/skills/daily-social-auto/fonts
cd ~/.openclaw/skills/daily-social-auto/fonts
curl -L -o Inter-Bold.ttf \
  https://github.com/rsms/inter/raw/master/docs/font-files/Inter-Bold.ttf
curl -L -o Inter-Medium.ttf \
  https://github.com/rsms/inter/raw/master/docs/font-files/Inter-Medium.ttf
```

中文字體（選配）：

```bash
curl -L -o NotoSansTC-Bold.otf \
  "https://fonts.gstatic.com/ea/notosanstc/v1/NotoSansTC-Bold.otf"
```

## Phase 5：建工作目錄

```bash
mkdir -p ~/social-content
mkdir -p ~/.openclaw/logs
```

## Phase 6：對話式 Onboarding（10 分鐘）

對 OpenClaw agent 說：

```
我要用 daily-social-auto，幫我設定
```

接下來跟著 agent 的問題走。

### Step 6.1: IG 帳號（30 秒）

Agent：「告訴我你的 IG 帳號名稱：」
你：「@openclawtw」

### Step 6.2: 跑 Stage 1 IG 分析（3 分鐘）

Agent 跑 Apify + LLM 分析，給你看摘要：

```
✅ 分析完成。我看到的你是這樣：

📌 主題：tech / AI / 工具教學
🎯 受眾：25-40 工程師、PM、自由工作者
🗣️ 調性：工程師口吻、簡潔、有點嘲諷
✍️ 你常用的開頭：「今天看到一個有意思的」「我幫你測過了」
🚫 紅線：不碰政治、不業配不標、不誇大
📊 內容比例：工具實測 40% / AI 新聞 30% / 教學 20% / 觀察 10%

這個分析準確嗎？
```

**重點檢查**：
- 主題對嗎？
- 調性像不像你？
- 紅線完整嗎？

不準的地方告訴 agent，它會修。準了再進下一步。

### Step 6.3: 補抓 FB / Threads / X（2 分鐘）

```
Agent：要不要把 FB / Threads / X 也加進分析？
你：FB 粉專是 openclawtw，其他都沒有
```

或：

```
你：都一樣，handle 都是 @openclawtw
```

或：

```
你：跳過，我只做 IG
```

### Step 6.4: 上傳範本圖（2 分鐘）★ 關鍵

Agent：

```
請上傳 1-5 張代表性範本圖：
- 你既有作品中你最滿意的
- 或別人作品中你想模仿的
- 建議：1 張 cover 範本 + 1-2 張內頁範本
- JPG/PNG 都可以，>800px

📤 直接拖檔案進來
```

你拖 3 張圖進 Telegram/Discord/Web。

Agent 會：
1. 下載到 `~/social-content/openclawtw/_config/style_refs/`
2. 用 vision LLM 提取 style DNA
3. 顯示給你看：

```
✅ 我看了你上傳的 3 張圖：

🎨 配色：亮藍 #0EA5E9 + 近白 #FAFAFA 為主
📐 版型：極簡，大量負空間，標題置左
🔤 字體：粗黑無襯線標題 + 細無襯線 body
✨ 裝飾：少量幾何亮藍重點

這 3 張圖會被永久附在每次生圖當參考。
要進下一步（規劃 30 天）嗎？
```

如果你覺得 LLM 描述不準，告訴 agent「再加一條：標題一定要置中」之類，它會補進 yaml。

### Step 6.5: 規劃 30 天 Calendar（1 分鐘）

Agent 跑 Stage 2，給你看第一週主題：

```
✅ 30 天大綱完成。

第 1 週：
- 5/3 (六)：AI 工具訂閱費破萬了？我的省錢取捨 [觀察]
- 5/4 (日)：5 個 ChatGPT 進階指令大多人沒用過 [教學]
- 5/5 (一)：n8n 自動化 email 回信教學 [教學]
- 5/6 (二)：Cursor 新版 review：值得切過去嗎 [實測]
- 5/7 (三)：本週 AI 大事 #1：Claude 4.7 vs GPT-5 [新聞]
- 5/8 (四)：[wildcard - 當天決定]
- 5/9 (五)：週五 AI 工具榜：本週 5 個 [新聞]

完整 calendar：~/social-content/openclawtw/_config/content_calendar.yaml

最後問：
- 「auto_publish: true」→ 全自動發
- 「auto_publish: false」→ 我只生內容、你看了再決定（建議前 7 天用這個）
```

你回「auto_publish: false」（推薦）或「auto_publish: true」。

### Step 6.6: 排程啟動

Agent 自動寫進 `~/.openclaw/schedules.yaml`：

```yaml
- skill: daily-social-auto
  cron: "0 8 * * *"
  timezone: "Asia/Taipei"
  args:
    client: openclawtw
    stage: 3
```

並執行：

```bash
openclaw scheduler reload
```

完成。

## Phase 7：第一天驗收

明天早上 8:00 後，agent 會通知你：

```
📤 daily-social-auto 已執行：

主題：AI 工具訂閱費破萬了？我的省錢取捨
✅ 內容已生成，等你審核（auto_publish=false）

審核：
- 文案：~/social-content/openclawtw/2026-05-03_.../*/post.md
- 圖片：~/social-content/openclawtw/2026-05-03_.../images/

審核完回我：「發出去」或「明天重來」
```

打開圖看：

```bash
DATE=$(date +%Y-%m-%d)
open ~/social-content/openclawtw/${DATE}*/images/slide_1_cover.jpg
```

**驗收清單**：

- [ ] 5 張圖都是 1080×1350
- [ ] 風格跟你上傳的範本「一眼看像同一套設計」
- [ ] 每張右上角有 `1/5`、`2/5` 頁碼
- [ ] 每張左下角有 `@openclawtw` 浮水印
- [ ] 4 平台文案不一樣（不是同一份貼 4 次）
- [ ] 文案像你（用了你的開頭、結尾、emoji）
- [ ] 沒有用 banned_words

任何一項不過 → 不要按「發出去」，回去調 brand_voice 或 style_refs。

## Phase 8：第一週微調

連續看 7 天，每天可能會調：

- 文案不夠像你 → 改 `brand_voice.yaml.signature_phrases`
- 圖片風格漂走 → 加更具體的 `style_dna` 條目，或重傳 ref
- 某類主題不喜歡 → 改 `content_calendar.yaml` 的對應日子
- 某天主題剛好爆熱、想多發一篇 → agent，「明天加一篇 XXX 主題」

## Phase 9：開全自動

第 7 天滿意後：

```yaml
# ~/social-content/openclawtw/_config/brand_voice.yaml
auto_publish: true
```

之後就放著它跑。第 28 天 agent 會自動規劃下個 30 天，你只要審核。

## 常見錯誤排除

| 錯誤 | 解法 |
|---|---|
| `OPENAI_API_KEY not set` | 重啟 gateway，確認 .env 有讀到 |
| `Organization not verified for gpt-image-2` | <https://platform.openai.com/settings/organization/general> 跑 verification |
| `style_refs 為空` | 重新進入 Stage 0 Step 6.4 上傳範本 |
| `brand_voice.yaml not found` | 還沒跑 Stage 1，重啟 onboarding 對話 |
| 生圖太像範本（沒新意）| 改 `ref_strength: inspired_by` |
| 生圖不像範本（沒學到）| 改 `ref_strength: precisely_replicate` 或加更具體 style_dna |
| 一直 429 rate limit | OpenAI tier 太低，升級 tier 或加 backoff |
| Apify 月底超 budget | 把每天 hashtag 範圍縮小，或升級 Apify 方案 |

## 多客戶設定

每個客戶獨立跑 Stage 0：

```
我要為新客戶 acmecorp 跑 daily-social-auto setup
```

Agent 會建 `~/social-content/acmecorp/_config/`，跟 openclawtw 完全隔離。

排程錯開：

```yaml
# ~/.openclaw/schedules.yaml
- skill: daily-social-auto
  cron: "0 8 * * *"
  timezone: "Asia/Taipei"
  args: { client: openclawtw }

- skill: daily-social-auto
  cron: "5 8 * * *"
  timezone: "Asia/Taipei"
  args: { client: acmecorp }

- skill: daily-social-auto
  cron: "10 8 * * *"
  timezone: "Asia/Taipei"
  args: { client: client3 }
```

## 完成

到這邊你有：
- ✅ 一份你親自審過的 brand_voice.yaml
- ✅ 1-5 張範本，每天每張生圖都會像你
- ✅ 30 天主題大綱，週末 / 平日 / 週報節奏均衡
- ✅ 每天 08:00 自動執行，自動 28 天前延期
- ✅ 完整紀錄與審核機制

放它跑兩週，需要時微調 brand_voice、style_refs、calendar。
三週後你應該已經完全信任它，每天只看通知就好。
