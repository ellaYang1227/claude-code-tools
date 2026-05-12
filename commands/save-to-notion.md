> 執行前先讀取共用規範：`C:\Users\ella_yang\.claude\commands\notion-spec.md`

請將我們這次對話中最重要的技術內容整理成筆記，
儲存到 Notion「技術筆記」資料庫（database ID: c8b2b191308c4366884fa11ed8bdd3d8），
並填入以下欄位：

- 名字：根據內容自動命名
- 分類：根據內容判斷
- 程式語言：根據內容判斷
- 文章分類：根據內容判斷
- 狀態：已完成
- 來源：依 notion-spec.md 規則判斷

---

## 步驟 1：整理技術內容（Claude 直接完成）

從對話中提取（格式規範見 notion-spec.md）：
- **標題**：精準描述核心主題
- **摘要**：說明核心內容
- **技術重點**：數量以對話中實際重要概念為準，不湊數也不遺漏；可依主題分組，形成多個 H2 章節

---

## 步驟 2：詢問使用者

依 notion-spec.md 欄位規則，使用 `AskUserQuestion` 工具詢問。

---

## 步驟 3：建立 Notion 頁面

使用 notion MCP 工具（`notion-create-pages`）建立頁面。

Icon、Callout 頁首結構、分割線原則、Notion API 限制見 notion-spec.md。

### 頁面結構

```
[Callout 📝 + divider + Callout 📍 + TOC]（見 notion-spec.md）
4.  [H2]  章節標題（依內容決定，可多個）
5.  [divider]
6.  段落 / bullet / code block
    ...（各重點之間 1 行空行）
7.  [H2]  程式碼範例（若有）
8.  [divider]
9.  code block(s)
10. [H2]  注意事項（若有）
11. [divider]
12. 注意事項條列
```
