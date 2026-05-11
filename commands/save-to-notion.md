請將我們這次對話中最重要的技術內容整理成筆記，
儲存到 Notion「技術筆記」資料庫（database ID: c8b2b191308c4366884fa11ed8bdd3d8），
並填入以下欄位：

- 名字：根據內容自動命名
- 分類：根據內容判斷
- 程式語言：根據內容判斷
- 文章分類：根據內容判斷
- 狀態：已完成
- 來源：Claude Code CLI

---

## 步驟 1：整理技術內容（Claude 直接完成）

從對話中提取：
- **標題**（20 字以內，精準描述核心主題）
- **摘要**（3～5 句，說明核心內容；開頭不加「本文由 XXX 介紹」等描述語）
- **技術重點**（每點 20～60 字；不加 ▶ 等前置圖示）
  - 數量以對話中實際重要概念為準，不湊數也不遺漏
  - 可依主題分組，形成多個 H2 章節

---

## 步驟 2：詢問使用者

使用 `AskUserQuestion` 工具詢問：

- `紀錄起源`（**必問**）：上班 / 進修 / 教練 / 面試

以下欄位**固定值，不詢問**：
- `來源` → 固定填「**Claude Code CLI**」
- `狀態` → 固定填「**已完成**」
- `建立者` → Notion 自動填入

以下欄位由 Claude 根據內容自行判斷：
- `分類`、`程式語言`、`文章分類`

---

## 步驟 3：建立 Notion 頁面

使用 notion MCP 工具（`notion-create-pages`）建立頁面。

**頁面 Icon（固定）：**
`https://notion-emojis.s3-us-west-2.amazonaws.com/prod/svg-twitter/1f916.svg`

### 頁面結構

```
1.  [callout 📝]  摘要          blue_background
2.  [divider]
3.  [callout 📍]  頁面目錄      gray_background，文字 orange bold
      └─ [table_of_contents]   ← 先建立 callout 取得 ID 再 append
4.  [H2]  章節標題（依內容決定，可多個）
5.  [divider]                  ← 每個 H2 下方直接加
6.  段落 / bullet / code block
    ...（各重點之間用 1 行空行，不加分割線）
7.  [H2]  程式碼範例（若有）
8.  [divider]
9.  code block(s)
10. [H2]  注意事項（若有）
11. [divider]
12. 注意事項條列
```

### 分割線原則（同 youtube-to-notion）
- 摘要 callout 後加一條（結束段落）
- 每個 H2 標題文字**下方直接**加一條（標題 → 分割線 → 內容）
- 各重點 / 小段落之間用 **1 行空行**隔開，不加分割線
- 不在最末尾段落後加分割線

### Notion API 注意事項
- Callout 的 children（TOC）**不可在建立時直接帶入**：先 append callout 取得 ID，再對該 ID append table_of_contents
- 若需更新 image block，必須先 delete 再用 `after` 參數 append，不可直接更新
