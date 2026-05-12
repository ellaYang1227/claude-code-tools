# Notion 撰寫規範（共用）

> 本檔案管理 `save-to-notion` 與 `youtube-to-notion` 的重複規範，兩個 skill 各自的獨有規則仍在原檔案。
> **兩個 skill 執行前須先讀取此檔案。**

---

## 資料庫欄位規則

固定值（不詢問）：
- `來源` → 網頁版 Claude Code 填「**Claude Chat**」；終端機 Claude Code 填「**Claude Code CLI**」；無法判斷時詢問使用者
- `狀態` → `已完成`

用 `AskUserQuestion` 詢問使用者：
- `紀錄起源`（**必問**）：上班 / 進修 / 教練 / 面試
- `分類`、`程式語言`、`文章分類`：**先依內容自行提供建議值，再與使用者確認**

`名字`：系列文章加 EP 前綴（格式：`EP.01 標題`、`EP.02 標題`）

---

## 頁面 Icon

固定 🤖：`{'type': 'emoji', 'emoji': '🤖'}`

---

## 內容格式規範

- **標題**：20 字以內；系列文章加 EP 前綴
- **摘要**：3～5 句，說明核心內容；開頭不加「本片由 XXX 解說，」等描述語
- **重點條列**：每點 20～60 字；不加 ▶ 等前置圖示；數量不設上限，以實際重要概念為準
  - 短片（< 5 分鐘）通常 3～5 點，長片（> 15 分鐘）可達 8～10 點

---

## Callout 頁首結構

```
[callout 📝]  摘要      blue_background
[divider]                ← 摘要段落結束
[callout 📍]  頁面目錄  gray_background，文字 color=orange bold
    └─ [table_of_contents]
```

---

## 分割線原則

- 摘要 callout 後加一條（結束段落）
- 每個 H2 標題文字**下方直接**加一條（標題 → 分割線 → 內容）
- 小段落（各重點）之間用 **1 行空行**隔開，不加分割線
- 不在段落末尾 / 大段落之間加分割線

---

## Notion API 限制

- **Callout children**：`blocks.children.append` 不支援直接帶 children，必須先 append callout 取得 ID，再對該 ID append `table_of_contents`
- **Image block 更新**：不可直接更新，需先 `blocks.delete` 再用 `after` 參數 append
