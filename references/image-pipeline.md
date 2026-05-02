# Image Pipeline — daily-social-auto v3

完整圖片生成流程：從使用者上傳的範本 → 風格學習 → 每張 IG slide 生成。

## Pipeline 總覽

```
Stage 0：使用者上傳 ref_1.jpg, ref_2.jpg, ref_3.jpg
              ↓
       Pillow 預先處理（縮到 1024px max edge，省 token）
              ↓
       存到 ~/social-content/{client}/_config/style_refs/
              ↓
       LLM (Claude vision / GPT-4o vision) 看範本，
       提取 style_dna 寫進 brand_voice.yaml
              ↓
====================================================
Stage 3：每天 08:00 自動執行
              ↓
       對每張 slide（共 5+ 張）：
         1. 組 prompt（帶入 style_dna + slide_text）
         2. 呼叫 GPT-image-2 /v1/images/edits
            → 附 style_refs 全部當參考圖
         3. 取回 1024×1536 jpeg
         4. Pillow resize+crop 到 1080×1350
         5. Pillow 蓋頁碼（右上 N/T）+ @handle 浮水印（左下）
         6. 存到 {date}_{slug}/images/slide_N_xxx.jpg
              ↓
       FB / Threads 共用整套 IG slides
       X 用 slide_1_cover.jpg 當附圖
```

## Stage 0：範本上傳處理

### 接收上傳

OpenClaw agent 收到使用者拖檔的圖片後：

```python
from pathlib import Path
from PIL import Image
import shutil

def ingest_uploaded_refs(uploaded_files: list, client: str):
    """處理使用者上傳的 1-5 張範本圖"""
    if not (1 <= len(uploaded_files) <= 5):
        raise ValueError("請上傳 1-5 張範本圖")

    style_dir = Path(f"~/social-content/{client}/_config/style_refs").expanduser()
    style_dir.mkdir(parents=True, exist_ok=True)

    # 清空舊範本（如果有）
    for old in style_dir.glob("ref_*.jpg"):
        old.unlink()

    saved = []
    for i, src in enumerate(uploaded_files, start=1):
        dst = style_dir / f"ref_{i}.jpg"
        preprocess_uploaded_ref(src, dst)
        saved.append(dst)

    return saved


def preprocess_uploaded_ref(src: Path, dst: Path):
    """縮到 max edge 1024，存成 jpeg quality 85，省 token 但夠模型理解風格"""
    img = Image.open(src).convert("RGB")
    img.thumbnail((1024, 1024), Image.LANCZOS)
    img.save(dst, "JPEG", quality=85, optimize=True)
```

**為什麼要縮到 1024**：

GPT-image-2 處理所有圖片輸入都是高保真，原始 4K 跟 1K 圖在 model 眼裡細節差不多
（風格資訊不需要那麼高解析度），但 4K 會吃掉 4 倍 input tokens。1024 是品質與成本的甜蜜點。

### LLM 提取 style_dna

讓 vision-capable LLM 看範本圖，產出文字描述塞進 brand_voice.yaml：

```python
def extract_style_dna(ref_paths: list) -> dict:
    """請 LLM 用 vision 看範本，回傳 style_dna 文字描述 + 偵測到的色票"""
    prompt = """
    看這幾張使用者提供的 IG 設計範本，分析它們共同的視覺 DNA。

    任務：
    1. 用 5-8 條精準描述提取「視覺特徵」，例：
       - "極簡布局，大量負空間"
       - "粗黑無襯線標題置左"
       - "亮藍色幾何重點裝飾"
    2. 偵測主色與背景色（hex）

    輸出 JSON：
    {
      "style_dna": ["...", "..."],
      "brand_color": "#RRGGBB",
      "secondary_color": "#RRGGBB"
    }

    描述要具體、可執行（之後會放進 prompt 給 GPT-image-2）。
    不要泛泛而談如「現代」「乾淨」這類沒資訊量的詞。
    """
    # 呼叫 GPT-4o 或 Claude with vision
    # 把每張 ref 圖以 base64 image content block 傳入
    return llm_call_with_vision(prompt, ref_paths)
```

把結果寫進 `brand_voice.yaml` 的 `ig_branding` 區段：

```yaml
ig_branding:
  handle: "@openclawtw"
  brand_color: "#0EA5E9"        # LLM 偵測
  secondary_color: "#FAFAFA"    # LLM 偵測
  style_dna:                    # LLM 寫 5-8 條
    - "極簡布局，大量負空間"
    - "粗黑無襯線標題置左，副標小一號靠下"
    - "亮藍色幾何重點裝飾"
    - "純白或近白背景"
    - "字體層級清晰：標題 / 副標 / body 三層"
  style_refs:                   # 範本圖路徑（永久使用）
    - _config/style_refs/ref_1.jpg
    - _config/style_refs/ref_2.jpg
    - _config/style_refs/ref_3.jpg
  generation_mode: edit_with_refs
```

## Stage 3：每張 slide 生成（核心）

### 完整函式

```python
import os
import base64
import requests
from pathlib import Path
from PIL import Image, ImageDraw, ImageFont

OPENAI_API_KEY = os.environ["OPENAI_API_KEY"]
FONT_BOLD = "~/.openclaw/skills/daily-social-auto/fonts/Inter-Bold.ttf"


def make_ig_slide(slide_num: int, total: int, slide_text: str,
                  brand: dict, output_dir: Path, slide_name: str,
                  quality: str = "medium") -> Path:
    """
    完整生 1 張 IG slide：
      1. GPT-image-2 /v1/images/edits + style_refs
      2. Pillow resize+crop 到 1080x1350
      3. Pillow 蓋頁碼 + 浮水印
    """
    output_path = output_dir / f"slide_{slide_num}_{slide_name}.jpg"

    # 1. 組 prompt
    prompt = build_slide_prompt(slide_num, total, slide_text, brand)

    # 2. 呼叫 GPT-image-2 edits endpoint，附範本
    generate_with_style_refs(prompt, brand, output_path, quality)

    # 3. resize 到 IG 標準尺寸
    resize_to_ig_carousel(output_path)

    # 4. 蓋頁碼與浮水印
    stamp_overlays(
        output_path,
        slide_num=slide_num,
        total=total,
        ig_handle=brand["ig_branding"]["handle"]
    )

    return output_path


def build_slide_prompt(slide_num: int, total: int, slide_text: str, brand: dict) -> str:
    style_dna_lines = "\n".join(f"- {line}" for line in brand["ig_branding"]["style_dna"])
    ref_count = len(brand["ig_branding"]["style_refs"])
    ref_descriptions = "\n".join(
        f"- Image {i+1}: brand reference (study layout, color, typography)"
        for i in range(ref_count)
    )

    return f"""You are creating Slide {slide_num} of {total} for an Instagram carousel.

REFERENCE IMAGES (study these for visual style, layout, color, typography):
{ref_descriptions}

YOUR TASK:
Create a NEW slide that closely matches the visual style, color palette,
typography hierarchy, layout grid, and decorative elements of the reference
images. Do NOT copy reference text — replace with the new text below.

NEW TEXT FOR THIS SLIDE (render exactly as written, no paraphrasing):
\"\"\"
{slide_text}
\"\"\"

STYLE DNA TO FOLLOW (extracted from refs):
{style_dna_lines}

LAYOUT CONSTRAINTS:
- Portrait 2:3 aspect ratio (will be cropped to 4:5 in post-processing)
- Text is the visual hero — large, immediately readable
- Maximum 3 lines of text, max 15 words per line
- Reserve top-right region (200x100px) clear of detail (page counter overlay)
- Reserve bottom-left region (300x100px) clear of detail (handle watermark)
- Brand color {brand["ig_branding"]["brand_color"]} as primary accent
- Background tone close to {brand["ig_branding"]["secondary_color"]}

CRITICAL CONSTRAINTS:
- Match references' typography weight, alignment, and hierarchy
- Match references' use of whitespace and decorative elements
- Do NOT include: people, hands, faces, generic AI humans
- Do NOT include: any text other than what's specified above
- Do NOT include: watermarks, page numbers, signatures (added in post-processing)

The output should look like it came from the same designer who made the references."""


def generate_with_style_refs(prompt: str, brand: dict, output_path: Path,
                              quality: str = "medium"):
    """呼叫 GPT-image-2 /v1/images/edits，附使用者範本當參考"""
    url = "https://api.openai.com/v1/images/edits"
    headers = {"Authorization": f"Bearer {OPENAI_API_KEY}"}

    client_dir = output_path.parent.parent.parent
    ref_paths = [client_dir / "_config" / Path(p).name
                 for p in brand["ig_branding"]["style_refs"]]

    files = []
    for ref in ref_paths:
        files.append(("image[]", (ref.name, open(ref, "rb"), "image/jpeg")))

    data = {
        "model": "gpt-image-2",
        "prompt": prompt,
        "size": "1024x1536",
        "quality": quality,
        "output_format": "jpeg",
        "output_compression": "92",
        "n": "1"
    }

    r = requests.post(url, headers=headers, data=data, files=files, timeout=180)
    # 關閉檔案 handle
    for _, (_, f, _) in files:
        f.close()

    r.raise_for_status()
    img_b64 = r.json()["data"][0]["b64_json"]
    output_path.write_bytes(base64.b64decode(img_b64))


def resize_to_ig_carousel(img_path: Path):
    """1024x1536 (2:3) → 1080x1350 (4:5)，等比放大寬度後居中裁高度"""
    img = Image.open(img_path)
    target_w, target_h = 1080, 1350

    src_w, src_h = img.size
    scale = target_w / src_w
    new_w = target_w
    new_h = int(src_h * scale)

    img = img.resize((new_w, new_h), Image.LANCZOS)

    top = (new_h - target_h) // 2
    img = img.crop((0, top, target_w, top + target_h))
    img.save(img_path, "JPEG", quality=92)


def stamp_overlays(img_path: Path, slide_num: int, total: int,
                   ig_handle: str, brand_color: str = "#FFFFFF"):
    """蓋頁碼（右上）+ @handle 浮水印（左下）"""
    img = Image.open(img_path).convert("RGBA")
    W, H = img.size  # 1080, 1350

    overlay = Image.new("RGBA", (W, H), (0, 0, 0, 0))
    draw = ImageDraw.Draw(overlay)

    # 1. 頁碼右上
    counter = f"{slide_num}/{total}"
    counter_font = ImageFont.truetype(FONT_BOLD, 32)
    bbox = draw.textbbox((0, 0), counter, font=counter_font)
    cw = bbox[2] - bbox[0]
    draw.text((W - cw - 50, 50), counter, fill=brand_color, font=counter_font)

    # 2. handle 左下，半透明
    handle_font = ImageFont.truetype(FONT_BOLD, 28)
    draw.text((50, H - 70), ig_handle, fill=(255, 255, 255, 200), font=handle_font)

    img = Image.alpha_composite(img, overlay)
    img.convert("RGB").save(img_path, "JPEG", quality=92)
```

## 範本生效程度的調控

如果使用者覺得生圖太像範本（沒新意）或太不像（沒學到），調這幾個地方：

### 1. 修改 prompt 強度

在 `build_slide_prompt` 裡：

- 太像範本 → 把 `closely matches` 改成 `inspired by`
- 太不像 → 把 `closely matches` 改成 `precisely replicates`

### 2. 調 style_refs 數量

- 1 張：模型自由度最高，可能風格漂移
- 2-3 張：甜蜜點（推薦）
- 4-5 張：最一致但最貴（每張都當 input 算 token）

### 3. 用 style_dna 詞語

LLM 提取的 style_dna 太抽象 → agent 可問使用者「再給我幾條形容詞」直接編輯 yaml。
具體比抽象重要：「粗黑 Inter Bold」勝過「現代字體」。

## 成本對照（1024×1536 medium）

| 設定 | 每張 | 5 張一組 | 30 天 |
|---|---|---|---|
| 純 generation（無 ref）| ~$0.041 | ~$0.20 | ~$6.20 |
| edits + 1 ref | ~$0.055 | ~$0.28 | ~$8.30 |
| **edits + 3 refs（推薦）** | **~$0.07** | **~$0.35** | **~$10.50** |
| edits + 5 refs | ~$0.085 | ~$0.43 | ~$12.80 |

實際成本取決於 prompt 長度、ref 圖實際 token 數，估算僅供參考。
正式上線後第一週每天記錄實際 OpenAI billing dashboard 的消耗，再調整 quality / ref 數量。

## 預算保護

`brand_voice.yaml.budget`:

```yaml
budget:
  daily_image_usd_limit: 1.0      # 一天 image 用量上限
  monthly_openai_usd_limit: 50.0  # 月 OpenAI 總額（含 LLM）
  on_exceed: skip                 # skip | warn | hard_stop
```

每天 Stage 3 開始前先 query OpenAI usage API：

```python
def check_image_budget(daily_limit_usd=1.0) -> bool:
    today = date.today().isoformat()
    r = requests.get(
        f"https://api.openai.com/v1/usage?date={today}",
        headers={"Authorization": f"Bearer {OPENAI_API_KEY}"}
    )
    today_image_cost = sum(
        item["cost_usd"] for item in r.json()["data"]
        if "image" in item.get("model", "")
    )
    return today_image_cost < daily_limit_usd
```

超過 → `skip` 模式直接 abort 當天並通知；`warn` 繼續但發提醒；`hard_stop` 暫停 cron。

## Troubleshooting

| 症狀 | 可能原因 | 解法 |
|---|---|---|
| 生圖風格漂走，每張都不一樣 | style_refs 不夠或太雜 | 換成 2-3 張同設計師同系列的範本 |
| 每張都跟某張範本一模一樣 | prompt 措辭太強 | 把 `closely matches` 改 `inspired by` |
| 文字渲染拼錯 | slide_text 太長或含罕用字 | 縮到 ≤15 字/行；非常重要的字用 Pillow 文字渲染另外蓋上 |
| 圖片有 watermark / 頁碼但變糊 | overlay 顏色與背景色衝突 | 在 stamp_overlays 加半透明遮罩 |
| Cost 比預估高很多 | quality 是 high 或 ref 圖太大 | 降為 medium、確認 ref 都已縮到 1024px |
| 偶爾回傳 429 | concurrent rate limit | 加 exponential backoff with jitter，2 秒 → 4 秒 → 8 秒 |
