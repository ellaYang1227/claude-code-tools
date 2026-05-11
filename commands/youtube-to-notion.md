---
description: 將 YouTube 影片截圖與重點整理至 Notion 資料庫
argument-hint: <YouTube URL>
---

用戶提供了 YouTube 影片 URL：**$ARGUMENTS**

請依序執行以下步驟，過程中主動說明進度。

---

## ⚙️ 路徑設定（使用前必填）

將以下三條路徑改為你電腦上的實際安裝位置後再使用。
**以下所有腳本中的路徑皆應使用此設定區的值，無需個別調整。**

```python
PYTHON = r'C:\Users\YOUR_USER\AppData\Local\Programs\Python\Python312\python.exe'
YTDLP  = r'C:\Users\YOUR_USER\AppData\Local\Programs\Python\Python312\Scripts\yt-dlp.exe'
FFMPEG = r'C:\Users\YOUR_USER\AppData\Local\Microsoft\WinGet\Packages\Gyan.FFmpeg_Microsoft.Winget.Source_8wekyb3d8bbwe\ffmpeg-8.1.1-full_build\bin\ffmpeg.exe'
# FFmpeg 路徑依 winget 安裝版本而異，可用 where.exe ffmpeg 確認
```

---

## 環境說明（Windows 路徑）

在 Windows 上，`python` 指令可能被 Microsoft Store 假捷徑攔截，請使用上方設定區的完整路徑。

所有 Python 腳本請存成暫存檔再執行（避免 PowerShell inline `-c` 引號問題）：
```powershell
$script | Set-Content "$env:TEMP\yt_xxx.py" -Encoding utf8
& $PYTHON "$env:TEMP\yt_xxx.py"
```

---

## 步驟 1：取得字幕與影片資訊

將以下腳本存成 `$env:TEMP\yt_fetch.py` 後執行：

```python
import json, subprocess, re, sys, io
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')
from youtube_transcript_api import YouTubeTranscriptApi
from youtube_transcript_api._errors import NoTranscriptFound

YTDLP = r'C:\Users\YOUR_USER\AppData\Local\Programs\Python\Python312\Scripts\yt-dlp.exe'
url = '$ARGUMENTS'
vid = re.search(r'(?:v=|youtu\.be/)([A-Za-z0-9_-]{11})', url).group(1)

info = json.loads(subprocess.run(
    [YTDLP, '--dump-json', '--no-download', '--quiet', url],
    capture_output=True, text=True, encoding='utf-8').stdout)
print('VIDEO_ID=' + vid)
print('DURATION=' + str(info.get('duration', 600)))
print('TITLE=' + info.get('title', ''))

try:
    api = YouTubeTranscriptApi()
    tlist = api.list(vid)
    for lang in ['zh-TW', 'zh-Hant', 'zh', 'zh-CN', 'en']:
        try:
            entries = tlist.find_transcript([lang]).fetch()
            print('LANG=' + lang)
            print('TRANSCRIPT_START')
            print(' '.join(e.text for e in entries)[:8000])
            print('TRANSCRIPT_END')
            break
        except NoTranscriptFound:
            continue
except Exception:
    print('FALLBACK=' + info.get('description', '')[:3000])
```

---

## 步驟 2：整理重點（Claude 直接完成）

根據字幕自行整理：
- **繁體中文標題**（20 字以內；系列文章需加 EP 前綴，格式：`EP.01 標題`、`EP.02 標題`，依步驟 6 是否為系列文章而定）
- **摘要**（3～5 句，說明核心內容；開頭不加「本片由 XXX 解說，」等描述語）
- **重點條列**（每點 20～60 字；不加 ▶ 等前置圖示）
  - 數量**不設上限**，以影片實際重要概念數量為準
  - 原則：每個重點都應是獨立且值得記錄的概念，不因湊數而拆細，也不因壓縮而遺漏重要觀點
  - 短片（< 5 分鐘）通常 3～5 點，長片（> 15 分鐘）可達 8～10 點

整理完成後，根據逐字稿分析各重點在影片中出現的大約時間點（秒），用於步驟 4 截圖。

---

## 步驟 3：查詢資料庫欄位並詢問使用者

執行以下腳本取得資料庫 schema，**對每個 select / multi_select 空白欄位詢問使用者**如何填寫，狀態統一設為「已完成」：

```python
import os, sys, io, requests
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')

NOTION_TOKEN = os.environ['NOTION_TOKEN']
DB_ID = os.environ['NOTION_DATABASE_ID']
h = {'Authorization': f'Bearer {NOTION_TOKEN}', 'Notion-Version': '2022-06-28'}

r = requests.get(f'https://api.notion.com/v1/databases/{DB_ID}', headers=h)
props = r.json().get('properties', {})
for name, prop in props.items():
    ptype = prop['type']
    if ptype == 'select':
        opts = [o['name'] for o in prop['select'].get('options', [])]
        print(f"[select] '{name}': {opts}")
    elif ptype == 'multi_select':
        opts = [o['name'] for o in prop['multi_select'].get('options', [])]
        print(f"[multi_select] '{name}': {opts}")
    elif ptype == 'status':
        opts = [o['name'] for o in prop['status'].get('options', [])]
        print(f"[status] '{name}': {opts}  → 固定填「已完成」")
    else:
        print(f"[{ptype}] '{name}'")
```

取得選項後，使用 `AskUserQuestion` 工具詢問使用者以下欄位：
- `紀錄起源` (select: 上班 / 進修 / 教練 / 面試) → **詢問使用者**
- `分類`、`程式語言`、`文章分類` → **詢問使用者**

以下欄位**固定值，不詢問**：
- `來源` → 固定填「**Claude Code CLI**」
- `狀態` → 固定填「**已完成**」
- `建立者` → Notion 自動填入，不需設定

---

## 步驟 4：擷取影片截圖

根據步驟 2 整理的重點，分析逐字稿估算每個重點對應的影片秒數，作為截圖時間點（TIMESTAMPS）。
截圖數量與重點數量相同（不以均勻分散方式截圖）。

截圖挑選原則：
- 選取畫面上有視覺內容（圖表、文字投影片）的時間點，避免純暗色或過渡幀
- 避開示範操作段落（demo 區間通常在影片後半段），除非重點本身涉及 demo
- 若某張截圖畫面過暗或內容不清晰，可將時間點微調 ±5～10 秒重試
- 若畫面中有文字動畫尚未完成（文字半透明或仍在出現中），將時間點往後推 3～5 秒讓動畫跑完，可保留相同畫面構成

```python
import subprocess, os, tempfile

YTDLP  = r'C:\Users\YOUR_USER\AppData\Local\Programs\Python\Python312\Scripts\yt-dlp.exe'
FFMPEG = r'C:\Users\YOUR_USER\AppData\Local\Microsoft\WinGet\Packages\Gyan.FFmpeg_Microsoft.Winget.Source_8wekyb3d8bbwe\ffmpeg-8.1.1-full_build\bin\ffmpeg.exe'

url = '$ARGUMENTS'
# 根據逐字稿分析填入各重點對應的時間點（秒）
TIMESTAMPS = [<T1>, <T2>, <T3>, <T4>, <T5>]

tmpdir     = tempfile.mkdtemp(prefix='yt_frames_')
video_path = os.path.join(tmpdir, 'video.mp4')

subprocess.run([YTDLP, '-f', 'bestvideo[height<=720][ext=mp4]/best[height<=720]',
    '-o', video_path, '--no-playlist', '--quiet', url], check=True)

frames = []
for i, ts in enumerate(TIMESTAMPS):
    frame = os.path.join(tmpdir, f'frame_{i:02d}.jpg')
    subprocess.run([FFMPEG, '-ss', str(ts), '-i', video_path,
        '-vframes', '1', '-q:v', '2',
        '-vf', 'unsharp=3:3:0.8:3:3:0.0',   # 銳化濾鏡
        '-y', frame], check=True, capture_output=True)
    frames.append(frame)
    print(f'FRAME_{i}={frame}  @{ts}s')
print('TMPDIR=' + tmpdir)
```

---

## 步驟 5：上傳截圖並建立 Notion 頁面

將步驟 2～4 的內容代入後執行：

```python
import os, sys, io, requests, shutil
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')
from notion_client import Client as NotionClient

NOTION_TOKEN = os.environ['NOTION_TOKEN']
DB_ID = os.environ['NOTION_DATABASE_ID']
VIDEO_ID = '<VIDEO_ID>'
VIDEO_URL = '$ARGUMENTS'
TMPDIR = '<TMPDIR>'

data = {
    'title': '<TITLE>',
    'summary': '<SUMMARY>',
    'keypoints': ['<KP1>', '<KP2>', '...']  # 數量依影片內容決定，不設上限
}

# 欄位值（來源、狀態固定；紀錄起源及其他依步驟 3 使用者回答填入）
user_properties = {
    '名字':     {'title': [{'type': 'text', 'text': {'content': data['title']}}]},
    '來源':     {'select': {'name': 'Claude Code CLI'}},   # 固定
    '狀態':     {'status': {'name': '已完成'}},             # 固定
    '紀錄起源': {'select': {'name': '<使用者選擇: 上班/進修/教練/面試>'}},
    # 以下依使用者回答填入：
    '<multi_select欄位名>': {'multi_select': [{'name': '<選項1>'}, {'name': '<選項2>'}]},
    '<select欄位名>': {'select': {'name': '<使用者選擇>'}},
}

h = {'Authorization': f'Bearer {NOTION_TOKEN}', 'Notion-Version': '2022-06-28'}
thumbnail = f'https://img.youtube.com/vi/{VIDEO_ID}/maxresdefault.jpg'

def upload_image(path):
    """上傳圖片至 Notion：POST /file_uploads → POST /send（multipart）
    注意：不可將 Notion 回傳的 file URL 當 external URL 重用，那是有時效的 S3 簽名 URL。
    每次重建頁面都必須重新上傳，才能保證圖片持續顯示。"""
    try:
        fname = os.path.basename(path)
        r = requests.post('https://api.notion.com/v1/file_uploads',
            headers={**h, 'Content-Type': 'application/json'},
            json={'name': fname, 'content_type': 'image/jpeg'}, timeout=30)
        if r.status_code not in (200, 201):
            return None
        d = r.json()
        with open(path, 'rb') as f:
            r2 = requests.post(d['upload_url'], headers=h,
                files={'file': (fname, f, 'image/jpeg')}, timeout=120)
        if r2.status_code not in (200, 201):
            return None
        return {'type': 'file_upload', 'file_upload': {'id': d['id']}}
    except Exception as e:
        print(f'  上傳失敗: {e}')
        return None

def img_block(src):
    if src:
        return {'object': 'block', 'type': 'image', 'image': src}
    return {'object': 'block', 'type': 'image',
            'image': {'type': 'external', 'external': {'url': thumbnail}}}

def empty_para():
    return {'object': 'block', 'type': 'paragraph', 'paragraph': {'rich_text': []}}

n_kp = len(data['keypoints'])
print('上傳截圖...')
sources = []
for i in range(n_kp):
    path = os.path.join(TMPDIR, f'frame_{i:02d}.jpg')
    src = upload_image(path)
    sources.append(src)
    print(f'  frame_{i:02d}: {"✅" if src else "⚠️ 改用縮圖"}')
shutil.rmtree(TMPDIR, ignore_errors=True)

# 頁面結構：
# 1.  [image]      YouTube thumbnail（放在摘要前）
# 2.  [callout 📝] 摘要（開頭不加「本片由 XXX 解說，」）
# 3.  [divider]                  ← 摘要段落結束
# 4.  [callout 📍] 頁面目錄      color=gray_background，文字 color=orange bold
#       └─ [table_of_contents]   ← TOC 內嵌於 callout，形成統一背景框
# 5.  [H2] 重點整理
# 6.  [divider]                  ← H2 標題文字下方（標題 → 分割線 → 內容）
# 7.  × N：[ 文字（無▶圖示）| 截圖 | 空行 ]  ← 小段落間 1 行，不加分割線
# 8.  [H2] 資料來源
# 9.  [divider]                  ← H2 標題文字下方（標題 → 分割線 → 內容）
# 10. [bookmark]
# 11. [divider]                  ← 資料來源後（系列導覽前）
# 12. [paragraph] ← 上一篇：...  ← page mention 或純文字「（目前為最早一篇）」
# 13. [paragraph] → 下一篇：...  ← page mention 或純文字「（目前為最新一篇）」
#
# 分割線原則：
#   - 摘要 callout 後加一條（結束段落）
#   - 每個 H2 標題文字下方直接加一條（標題 → 分割線 → 內容）
#   - 小段落（各重點）之間用 1 行空行隔開，不加分割線
#   - 不在段落末尾/大段落之間加分割線
#
# 注意：Notion API blocks.children.append 不支援 block 陣列直接帶 children，
#       必須先 append callout，取得 ID 後再 append TOC 進去。
# 注意：更新現有 image block 不可用 file_upload type，需先 delete 再用
#       after 參數 append 到對應 paragraph 後面。

notion = NotionClient(auth=NOTION_TOKEN)

# ── 建立頁面（帶前三個 blocks）──────────────────────────────────
page = notion.pages.create(
    parent={'database_id': DB_ID},
    icon={'type': 'emoji', 'emoji': '🤖'},
    properties=user_properties,
    children=[
        # 1. YouTube 封面
        {'object': 'block', 'type': 'image',
         'image': {'type': 'external', 'external': {'url': thumbnail}}},
        # 2. 摘要
        {'object': 'block', 'type': 'callout', 'callout': {
            'rich_text': [{'type': 'text', 'text': {'content': data['summary']}}],
            'icon': {'type': 'emoji', 'emoji': '📝'}, 'color': 'blue_background'}},
        # 3. 分割線
        {'object': 'block', 'type': 'divider', 'divider': {}},
    ]
)
page_id = page['id']

# ── 📍 頁面目錄 callout → 內嵌 TOC ─────────────────────────────
toc_resp = notion.blocks.children.append(
    block_id=page_id,
    children=[{'object': 'block', 'type': 'callout', 'callout': {
        'rich_text': [{'type': 'text', 'text': {'content': '頁面目錄'},
                       'annotations': {'bold': True, 'color': 'orange'}}],
        'icon': {'type': 'emoji', 'emoji': '📍'}, 'color': 'gray_background'}}]
)
toc_callout_id = toc_resp['results'][0]['id']
notion.blocks.children.append(
    block_id=toc_callout_id,
    children=[{'object': 'block', 'type': 'table_of_contents',
               'table_of_contents': {'color': 'default'}}]
)

# ── 重點整理（H2 → divider → KPs）+ 資料來源（H2 → divider → bookmark）
content_blocks = [
    {'object': 'block', 'type': 'heading_2', 'heading_2': {
        'rich_text': [{'type': 'text', 'text': {'content': '重點整理'}}]}},
    {'object': 'block', 'type': 'divider', 'divider': {}},
]
for i, point in enumerate(data['keypoints']):
    src = sources[i] if i < len(sources) else None
    content_blocks.append({'object': 'block', 'type': 'paragraph', 'paragraph': {'rich_text': [
        {'type': 'text', 'text': {'content': point}}]}})
    content_blocks.append(img_block(src))
    content_blocks.append(empty_para())

content_blocks += [
    {'object': 'block', 'type': 'heading_2', 'heading_2': {
        'rich_text': [{'type': 'text', 'text': {'content': '資料來源'}}]}},
    {'object': 'block', 'type': 'divider', 'divider': {}},
    {'object': 'block', 'type': 'bookmark', 'bookmark': {'url': VIDEO_URL}},
]
notion.blocks.children.append(block_id=page_id, children=content_blocks)

print('✅ Notion 頁面：https://notion.so/' + page_id.replace('-', ''))
```

---

## 步驟 6：新增上下篇導覽（系列文章）

使用 `AskUserQuestion` 詢問使用者：
- 此影片是否屬於系列文章？
- 上一篇的 Notion 頁面 ID（從 URL 取最後 32 碼，無則跳過）

取得資訊後執行以下腳本，在當前頁面與上一篇**雙向**加入導覽段落：

```python
import os, sys, io
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')
from notion_client import Client as NotionClient

notion = NotionClient(auth=os.environ['NOTION_TOKEN'])

CURRENT_PAGE_ID = '<PAGE_ID>'   # 剛建立的頁面 ID
PREV_PAGE_ID    = None           # 上一篇頁面 ID（無則 None）
# 下一篇尚未建立時固定為 None，未來新增下一篇時由下一篇腳本負責更新本頁

def mention_block(label, page_id):
    """只顯示存在的方向，無箭頭，格式：「上一篇：[page mention]」"""
    return {'object': 'block', 'type': 'paragraph', 'paragraph': {'rich_text': [
        {'type': 'text', 'text': {'content': f'{label}：'}},
        {'type': 'mention', 'mention': {'type': 'page', 'page': {'id': page_id}}}
    ]}}

# 當前頁面導覽
nav = [{'object': 'block', 'type': 'divider', 'divider': {}}]
if PREV_PAGE_ID:
    nav.append(mention_block('上一篇', PREV_PAGE_ID))
# 下一篇尚未建立時不加此行；待下一篇建立後由下一篇腳本負責更新本頁

notion.blocks.children.append(block_id=CURRENT_PAGE_ID, children=nav)
print('當前頁面導覽加入完成')

# 上一篇：刪除舊「下一篇」段落（若存在），再補上指向當前頁的「下一篇」
# 注意：先在 Notion 確認上一篇最後是否已有導覽段落，若有需手動刪除後再執行
if PREV_PAGE_ID:
    notion.blocks.children.append(
        block_id=PREV_PAGE_ID,
        children=[mention_block('下一篇', CURRENT_PAGE_ID)]
    )
    print('上一篇導覽更新完成')

print('✅ 導覽設定完成')
```

> 注意：
> - 導覽段落無箭頭（← →），格式為「上一篇：[頁面 mention]」、「下一篇：[頁面 mention]」
> - 第一篇只有「下一篇」，最後一篇只有「上一篇」，不顯示不存在的方向
> - 上一篇若已有導覽段落，請先在 Notion 手動刪除後再執行，避免重複

---

## 完成

輸出 Notion 頁面連結，並列出整理好的重點摘要。

---

## 前置需求（每台電腦只需設定一次）

```powershell
winget install Python.Python.3.12
# 重新開啟終端機後：
pip install youtube-transcript-api notion-client requests yt-dlp
winget install FFmpeg
```

環境變數（以系統環境變數設定，重開終端機生效）：
```powershell
[System.Environment]::SetEnvironmentVariable("NOTION_TOKEN", "ntn_xxx", "User")
[System.Environment]::SetEnvironmentVariable("NOTION_DATABASE_ID", "32碼ID", "User")
```

在 PowerShell session 中載入：
```powershell
$env:NOTION_TOKEN = [System.Environment]::GetEnvironmentVariable("NOTION_TOKEN", "User")
$env:NOTION_DATABASE_ID = [System.Environment]::GetEnvironmentVariable("NOTION_DATABASE_ID", "User")
```

- `NOTION_TOKEN` — Notion Integration Token（`ntn_...`）
- `NOTION_DATABASE_ID` — 目標資料庫 ID（32 碼，從 URL 取得）

> **注意**：需在 Notion 資料庫頁面 → 右上角「⋯」→「Connect to」，加入你的 Integration 才能存取。
