---
description: 將 YouTube 影片截圖與重點整理至 Notion 資料庫
argument-hint: <YouTube URL>
---

用戶提供了 YouTube 影片 URL：**$ARGUMENTS**

> 執行前先讀取共用規範：`C:\Users\ella_yang\.claude\commands\notion-spec.md`

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

根據字幕自行整理標題、摘要、重點條列（格式規範見 notion-spec.md）。

### 影片類型判斷

先判斷影片屬於哪種類型，後續重點文字與截圖策略將依此調整：

| 類型 | 特徵 | 例子 |
|------|------|------|
| **操作說明型** | 含安裝步驟、設定流程、功能示範、操作教學 | Chrome MCP 安裝、Claude Code 使用示範 |
| **觀念型** | 概念解說、比較分析、心法分享 | 計劃模式 vs 互動模式、AI 工具選擇 |

### 操作說明型的重點文字規範

每個重點需完整說明「在哪裡操作、做什麼、預期看到什麼結果」，讓讀者不看影片也能跟著執行：

- 字數放寬至 **60～100 字**（觀念型維持 20～60 字）
- 說明操作位置：「從左下角帳號 → Get apps and extensions → Chrome」
- 說明預期結果：「右上角出現 Claude 符號表示安裝成功」
- 各步驟之間有明確的前後順序關係

### 指令與程式碼的呈現

重點文字中若出現以下內容，使用 `` `backtick` `` 標記，對應到 Notion `inline_code` 樣式：
- 斜線指令：`/clear`、`/compact`、`/context`
- 鍵盤快捷鍵：`Escape`、`Ctrl+B`、`Enter`
- CLI 指令、程式語言關鍵字、檔案名稱

例：`任務完成後用 `/clear`，視窗將滿時用 `/compact` 壓縮釋放空間。`

---

### 安裝流程或多步驟設定：逐步拆解（圖＋文交錯）

若重點內容是**安裝說明**或**多步驟設定流程**，不用一個重點對一張截圖，改為：

- **段落標題**：每個獨立流程在第一個步驟前加 `heading_3` 標題，例如「CLI 版安裝 Chrome MCP」、「App 版安裝路徑」
- **每個步驟各一段文字**，格式：`步驟 N：[做什麼、在哪裡] → [預期畫面或結果]`
- **每個步驟各一張截圖**：優先選操作完成後的結果畫面；若該步驟有關鍵操作畫面（按鈕、選單），可額外補一張操作中的截圖
- Notion 中以 `heading_3 → para → image → para → image → ...` 交錯呈現
- **步驟數不設上限**，以「任何人都能依照說明成功完成操作」為目標，每個使用者需執行的動作都獨立成一步
- 步驟編號在同一流程內連貫，不同流程各自從「步驟 1」重新開始

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

取得選項後，依 notion-spec.md 欄位規則，使用 `AskUserQuestion` 詢問使用者。

另外詢問**延伸閱讀**（可選）：
- 是否加入「延伸閱讀」區塊？
- 若是，請使用者提供各條目的**標題**與 **URL**（可多筆），來源不限：
  - 自己的 Notion 頁面（貼上 Notion 頁面 URL）
  - 外部文章或網頁（貼上任意 URL）
- 輸入格式：`標題 | URL`，每條一行；標題應與原文章/頁面標題相同
- 整理後存入 `data['further_reading']`，格式為 `[{'title': '...', 'url': '...'}]`
- 無論來源類型，一律以 `bulleted_list_item` 呈現，文字為標題並附超連結

---

## 步驟 4：擷取影片截圖

根據步驟 2 整理的重點，分析逐字稿估算每個重點對應的影片秒數，作為截圖時間點（TIMESTAMPS）。

- **觀念型**：截圖數量與重點數量相同，不以均勻分散方式截圖
- **操作說明型**：每個主要操作步驟一張，截圖數量可超過重點數（步驟多時可增加）；截圖時間點選在每步操作完成後的靜止畫面

截圖挑選原則（通用）：
- 選取畫面上有視覺內容（圖表、文字投影片、UI 畫面）的時間點，避免純暗色或過渡幀
- 若某張截圖畫面過暗或內容不清晰，可將時間點微調 ±5～10 秒重試
- 若畫面中有文字動畫尚未完成（文字半透明或仍在出現中），將時間點往後推 3～5 秒讓動畫跑完

**操作說明型影片的額外原則：**
- **截圖選在操作完成後的結果畫面**，而非操作進行中（例如：安裝按鈕按下後，選取顯示安裝成功狀態的畫面）
- **截圖內容需與對應文字一致**：文字說「右上角出現 Claude 符號」，截圖就要清楚顯示那個符號；文字說「狀態變為啟用」，截圖就要顯示啟用狀態
- **每個主要操作步驟各一張**，截圖數量可多於觀念型，以確保每步都有視覺對應
- 操作示範段落（demo）的截圖：優先選取操作剛完成、畫面靜止後的那一刻
- 若一個重點描述兩個連續步驟，選最終結果的畫面即可

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
    'keypoints': ['<KP1>', '<KP2>', '...'],  # 數量依影片內容決定，不設上限
    # 延伸閱讀（可選）：由步驟 3 使用者回答填入，空列表則不產生該區塊
    'further_reading': [
        # {'title': '文章標題', 'url': 'https://...'},
    ],
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

def parse_rich_text(text):
    """將含 `code` 標記的文字轉為 Notion rich_text 陣列（inline_code）"""
    import re
    result = []
    for part in re.split(r'(`[^`]+`)', text):
        if part.startswith('`') and part.endswith('`') and len(part) > 2:
            result.append({'type': 'text', 'text': {'content': part[1:-1]},
                           'annotations': {'code': True}})
        elif part:
            result.append({'type': 'text', 'text': {'content': part}})
    return result

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

# 頁面結構、分割線原則、Notion API 限制見 notion-spec.md

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
    content_blocks.append({'object': 'block', 'type': 'paragraph', 'paragraph': {
        'rich_text': parse_rich_text(point)}})
    content_blocks.append(img_block(src))
    content_blocks.append(empty_para())

if data['further_reading']:
    content_blocks += [
        {'object': 'block', 'type': 'heading_2', 'heading_2': {
            'rich_text': [{'type': 'text', 'text': {'content': '延伸閱讀'}}]}},
        {'object': 'block', 'type': 'divider', 'divider': {}},
    ]
    for item in data['further_reading']:
        content_blocks.append({
            'object': 'block', 'type': 'bulleted_list_item',
            'bulleted_list_item': {'rich_text': [{
                'type': 'text',
                'text': {'content': item['title'], 'link': {'url': item['url']}}
            }]}
        })

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
