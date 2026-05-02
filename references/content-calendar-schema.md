# content_calendar.yaml Schema

Stage 2 產出、Stage 3 每天讀取的「30 天主題大綱」。

存放位置：`~/social-content/{client}/_config/content_calendar.yaml`

## 完整 schema

```yaml
# 識別
client: string
generated_at: ISO8601                 # Stage 2 產出時間
brand_voice_version: integer          # 對應的 brand_voice.yaml 版本

# 涵蓋區間
period:
  start: YYYY-MM-DD                   # 第一篇日期（通常是「明天」）
  end: YYYY-MM-DD                     # 第 30 篇日期
  total_days: integer                 # 通常 30

# 30 篇主題（每篇一筆）
posts:
  - date: YYYY-MM-DD                  # 此篇執行日期
    day_of_week: string               # mon | tue | wed | thu | fri | sat | sun
    pillar: string                    # 工具實測 | AI新聞 | 教學 | 觀察 | wildcard
    topic_title: string               # 主題標題（之後會被 Stage 3 微調）
    topic_slug: string                # kebab-case, [a-z0-9-], ≤50 字元
    angle: string                     # 切入角度（為什麼這樣講）
    keywords: [string]                # Stage 3 用來抓當天熱門的關鍵字
    cover_hook: string                # IG 封面 hook（之後會被 Stage 3 微調）
    insights: [string]                # 3-5 個 IG 內頁要點
    cta_direction: string             # CTA 方向，不是最終文字
    is_wildcard: boolean              # true = 當天 100% 跟熱門走，忽略 calendar 主題

    # === v3.1 系列模式欄位（series_mode.enabled = true 時填入）===
    series_name: string | null        # 例 "AI 自動化改變我的日常"
    episode_number: integer | null    # 例 3（代表此系列第 3 集）
    total_episodes: integer | null    # 例 7（代表此系列共 7 集）

    # === Stage 3 執行後填入 ===
    status: string                    # planned | skipped | published | failed
    executed_at: ISO8601 | null
    final_topic_title: string | null  # Stage 3 微調後的最終標題
    published_urls:
      instagram: string | null
      facebook: string | null
      threads: string | null
      x: string | null
    error: string | null

# 統計（自動算出）
stats:
  pillar_distribution:
    工具實測: integer
    AI新聞: integer
    教學: integer
    觀察: integer
    wildcard: integer
  wildcard_days: integer
  weekly_recap: integer

# 行為設定
auto_publish: boolean                 # Stage 3 是否自動發布；false = 只生內容、人審
posts_per_day: integer                # 預設 1
```

## 完整範例

```yaml
client: openclawtw
generated_at: 2026-05-02T10:30:00+08:00
brand_voice_version: 1

period:
  start: 2026-05-03
  end: 2026-06-01
  total_days: 30

posts:
  # ===== Week 1 =====
  - date: 2026-05-03
    day_of_week: sat
    pillar: 觀察
    topic_title: "AI 工具訂閱費破萬了？我的省錢取捨"
    topic_slug: ai-subscription-cost-tradeoff
    angle: "從個人開銷角度討論工具堆疊取捨，引發共鳴；週末輕量主題"
    keywords: [AI訂閱, ChatGPT Plus, Claude Pro, 工具堆疊, 訂閱費]
    cover_hook: "這個月 AI 工具帳單 $200，我砍了一半"
    insights:
      - "盤點：你真的每天都用嗎？"
      - "免費替代方案實測：哪些可以取代付費版"
      - "我的最終 stack（保留 + 砍掉）"
    cta_direction: "問讀者他們的 AI 工具 stack"
    is_wildcard: false
    status: planned
    executed_at: null
    final_topic_title: null
    published_urls:
      instagram: null
      facebook: null
      threads: null
      x: null
    error: null

  - date: 2026-05-04
    day_of_week: sun
    pillar: 教學
    topic_title: "5 個 ChatGPT 進階指令大多人沒用過"
    topic_slug: chatgpt-advanced-commands-5
    angle: "週日輕教學，給通勤族存起來週一試用"
    keywords: [ChatGPT, prompt, 進階指令, AI 工具]
    cover_hook: "用 ChatGPT 半年了？這 5 個指令你可能還沒試過"
    insights:
      - "/temperature 控制創意度"
      - "Custom Instructions 永久記憶設定"
      - "Voice mode 多語切換密技"
      - "Code interpreter 處理 Excel"
      - "Memory 關閉 / 開啟時機"
    cta_direction: "請讀者留言他們最常用的 prompt"
    is_wildcard: false
    status: planned
    executed_at: null
    final_topic_title: null
    published_urls: {instagram: null, facebook: null, threads: null, x: null}
    error: null

  - date: 2026-05-05
    day_of_week: mon
    pillar: 教學
    topic_title: "n8n 自動化 email 回信教學"
    topic_slug: n8n-email-auto-reply
    angle: "週一通勤族最愛的實作教學，可立即上手"
    keywords: [n8n, automation, 自動化, email, workflow]
    cover_hook: "我的 email 自己回了一週，靠這個免費工具"
    insights:
      - "n8n 是什麼、為什麼比 Zapier 划算"
      - "實作：抓 Gmail → GPT 草稿 → 等你按確認"
      - "5 個你可以套用的範本"
    cta_direction: "問讀者他們最想自動化哪件事"
    is_wildcard: false
    status: planned

  - date: 2026-05-06
    day_of_week: tue
    pillar: 工具實測
    topic_title: "Cursor 新版 review：值得切過去嗎"
    topic_slug: cursor-ide-review
    angle: "工程師受眾的核心題目，深度測"
    keywords: [Cursor, IDE, AI coding, Copilot, code editor]
    cover_hook: "Cursor 新版有 3 個改變，我已經用 2 週"
    insights:
      - "Composer 模式實測：寫 React 元件"
      - "vs VS Code + Copilot 的成本差距"
      - "我會留下還是換回去？"
    cta_direction: "問讀者用什麼 IDE"
    is_wildcard: false
    status: planned

  - date: 2026-05-07
    day_of_week: wed
    pillar: AI新聞
    topic_title: "本週 AI 大事 #1：Claude Opus 4.7 vs GPT-5"
    topic_slug: claude-vs-gpt5-week-1
    angle: "新聞日，不深測但有觀點"
    keywords: [Claude Opus, GPT-5, Anthropic, OpenAI, 模型比較]
    cover_hook: "兩家模型同週升級，我同樣 prompt 跑一遍"
    insights:
      - "code 任務：誰寫得乾淨"
      - "中文理解：誰更接地氣"
      - "成本與速度"
    cta_direction: "問讀者比較喜歡哪一家"
    is_wildcard: false
    status: planned

  - date: 2026-05-08
    day_of_week: thu
    pillar: wildcard
    topic_title: "[wildcard]"               # Stage 3 當天決定
    topic_slug: wildcard-d8
    angle: "wildcard：當天最熱主題自由發揮"
    keywords: []
    cover_hook: ""
    insights: []
    cta_direction: ""
    is_wildcard: true
    status: planned

  - date: 2026-05-09
    day_of_week: fri
    pillar: AI新聞
    topic_title: "週五 AI 工具榜：本週新出 / 有更新的 5 個"
    topic_slug: weekly-ai-tools-roundup-w1
    angle: "週五輕鬆讀，週末存起來慢慢看"
    keywords: [AI 工具, weekly recap, product hunt]
    cover_hook: "本週我留意到的 5 個 AI 工具"
    insights:
      - "新出：A 工具、B 工具"
      - "有更新：C 工具、D 工具"
      - "下架 / 漲價：E 工具"
    cta_direction: "問讀者本週試了哪個"
    is_wildcard: false
    status: planned

  # ===== Week 2-4 同樣模式 =====
  # ... 共 30 篇

stats:
  pillar_distribution:
    工具實測: 12
    AI新聞: 9
    教學: 6
    觀察: 3
    wildcard: 4              # 第 8、15、22、29 天（每 7 天 1 個）
  wildcard_days: 4
  weekly_recap: 4

auto_publish: false
posts_per_day: 1
```

## 排程規則（Stage 2 LLM 必須遵守）

1. **總天數 = 30**：從 `period.start` 起連續 30 天，不跳過任何日期
2. **pillar 比例**：對齊 brand_voice.content_pillars 的權重 × 30
   - 例：工具實測 0.4 → 12 篇；AI 新聞 0.3 → 9 篇
3. **wildcard 規則**：每 7 天插 1 個 wildcard，共 4 個（第 8、15、22、29 天）
4. **週末輕量**：週六、週日放 pillar=觀察 / 教學
5. **週五週報**：每週五排「週報」類（pillar=AI 新聞 / 工具實測 視當週主題定）
6. **不重複**：同一個工具或主題在 14 天內不能再出現
7. **日子 vs 主題深度匹配**：
   - 週一-三：核心實測 / 教學（讀者最有時間看）
   - 週四：可以排稍輕的觀察或 wildcard
   - 週五：週報
   - 週末：輕量、可儲存型

## Stage 3 執行邏輯

每天 08:00 執行：

```python
today = date.today().isoformat()
post = next(p for p in cal["posts"] if p["date"] == today)

if post["status"] != "planned":
    # 已經執行過、被跳過、或失敗——不再跑
    log(f"Today already {post['status']}, skipping.")
    exit(0)

if post["is_wildcard"]:
    # 100% 跟熱門走，忽略 topic_title
    final_post = run_wildcard_flow(brand)
else:
    # 用 calendar 主題 + 今日熱門微調
    trending = fetch_trending(post["keywords"], brand)
    final_post = refine_with_trending(post, trending, brand)

# 生內容、發布、更新 status
generate_and_publish(final_post, brand)
post["status"] = "published"
post["executed_at"] = datetime.now().isoformat()
post["final_topic_title"] = final_post["topic_title"]
save_yaml(cal, calendar_path)
```

## 30 天結束後

第 28 天 Stage 3 跑完後，自動觸發一次 Stage 2 延期：

```python
if date.today() >= cal["period"]["end"] - timedelta(days=2):
    # 還剩 2 天就結束 — 規劃下一個 30 天
    new_cal = run_stage_2(brand, start_date=cal["period"]["end"] + timedelta(days=1))
    save_yaml(new_cal, calendar_path)
    notify_user("已自動規劃下一個 30 天，請審核：~/social-content/.../content_calendar.yaml")
```

如果 `auto_publish: false`，第二個 30 天規劃完會等使用者確認；`auto_publish: true` 直接接續執行。

## 使用者修改慣例

- **改主題**：直接 `vim content_calendar.yaml`，改 `topic_title` / `keywords` / `cover_hook`，存檔即可
- **跳過某天**：把該日 `status` 從 `planned` 改成 `skipped`，Stage 3 會略過
- **替換主題**：刪掉一筆、加一筆新的（保持 date 順序）
- **延長 calendar**：對 agent 說「規劃下個月」→ 觸發 Stage 2 延期
- **完全重排**：對 agent 說「重新規劃 30 天」→ 舊版備份成 `content_calendar.{timestamp}.bak.yaml`，重跑 Stage 2
