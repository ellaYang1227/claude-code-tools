# claude-code-tools

以 Claude Code 為核心打造的個人工作流工具集。每個工具都是一個 Claude Code 自訂指令（slash command），透過自然語言驅動，讓 Claude 完成整個流程。

## 工具列表

| 工具 | 指令 | 說明 |
|------|------|------|
| [youtube-to-notion](commands/youtube-to-notion.md) | `/youtube-to-notion <URL>` | 整理 YouTube 影片重點摘要，自動建立 Notion 筆記頁面 |
| [save-to-notion](commands/save-to-notion.md) | `/save-to-notion` | 整理對話技術重點，寫入 Notion 技術筆記資料庫 |
| [figma-to-scss](commands/figma-to-scss.md) | `/figma-to-scss <URL>` | 從 Figma 自動讀取 design token，輸出 SCSS / CSS / JSON 等格式 |

## 安裝方式

1. 將 `commands/` 內的 `.md` 檔案複製到 `~/.claude/commands/`（Windows：`C:\Users\YOUR_USER\.claude\commands\`）
2. 打開檔案，依說明修改路徑設定
3. 在 Claude Code CLI 中輸入對應的 `/指令名稱` 即可使用

## 前置需求

各工具的安裝需求請見各自的 `.md` 檔案底部「前置需求」章節。

---

> 所有工具皆由 [Claude Code](https://claude.ai/code) 協助設計與實作。
