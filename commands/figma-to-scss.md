---
description: 從 Figma 檔案自動讀取 design token，輸出 SCSS / CSS / JSON 等格式
argument-hint: <Figma URL 或 File ID>
---

用戶提供了 Figma 檔案：**$ARGUMENTS**

請依序執行以下步驟，過程中主動說明進度。

---

## 步驟 1：解析 Figma File ID

從 `$ARGUMENTS` 解析 File ID：
- URL 格式（`https://www.figma.com/design/XXXXX/...` 或 `/file/XXXXX/...`）→ 取 `/design/` 或 `/file/` 後第一段
- 純 ID 格式（如 `5XwpV4fOuLWzKQqpti1LKN`）→ 直接使用

---

## 步驟 2：取得 Figma API Token

依序嘗試：

```powershell
$env:FIGMA_TOKEN = [System.Environment]::GetEnvironmentVariable("FIGMA_TOKEN", "User")
```

1. 若 `$env:FIGMA_TOKEN` 有值 → 直接使用
2. 若無 → 使用 `AskUserQuestion` 詢問使用者輸入

**Token 所需 scope（建立時需勾選）：**
- `File content` → Read only
- `Variables` → Read only
- `Library content` → Read only

---

## 步驟 3：詢問使用者

使用 `AskUserQuestion` 工具詢問以下兩項：

**① 輸出格式（multiSelect: true）：**
- `SCSS 變數` → `_tokens.scss`
- `CSS 變數` → `_variables.css`
- `JSON token` → `tokens.json`（W3C Design Tokens 格式）
- `Bootstrap 5 專用` → `_bootstrap-overrides.scss`
- `其他` → 詢問自訂說明後再決定格式

**② 輸出目錄：**
- 預設為目前工作目錄下的 `design-system/`
- 詢問使用者是否自訂路徑

---

## 步驟 4：呼叫 Figma API 讀取樣式

```powershell
$headers = @{ "X-Figma-Token" = $token }

# 1. Styles 列表（需 library_content:read）
$styles = Invoke-RestMethod `
    -Uri "https://api.figma.com/v1/files/$fileId/styles" `
    -Headers $headers

# 2. 節點詳細值（fills / effects / style）
$nodeIds = ($styles.meta.styles | ForEach-Object { $_.node_id }) -join ','
$nodes = Invoke-RestMethod `
    -Uri "https://api.figma.com/v1/files/$fileId/nodes?ids=$nodeIds" `
    -Headers $headers

# 3. Figma Variables（需 file_variables:read，失敗則跳過）
try {
    $vars = Invoke-RestMethod `
        -Uri "https://api.figma.com/v1/files/$fileId/variables/local" `
        -Headers $headers
} catch { $vars = $null }
```

---

## 步驟 5：轉換 token 值

Claude 從 API 回傳資料中萃取各類 token。

### 色彩（FILL style）

RGB 0\~1 浮點數 → HEX，透明度 < 1 時以 rgba() 表示：

```powershell
function ToHex($r, $g, $b) {
    $ri = [int][Math]::Round($r * 255)
    $gi = [int][Math]::Round($g * 255)
    $bi = [int][Math]::Round($b * 255)
    return '#' + $ri.ToString('X2') + $gi.ToString('X2') + $bi.ToString('X2')
}
# 注意：必須先 [int] 轉型，否則 .ToString('X2') 會拋出 FormatException
```

### 陰影（DROP_SHADOW effect）

```
box-shadow: {offset.x}px {offset.y}px {radius}px rgba(r*255, g*255, b*255, a)
```

### 字型（TEXT style）

取 `fontFamily`、`fontWeight`、`fontSize`、`lineHeightPx / fontSize`（計算比例）。

### 命名規則

Figma 樣式名稱轉為 kebab-case，特殊字元（空格、括號）替換為 `-`：
- `primary-2` → `primary-2`
- `bg - blur(30px)` → `bg-blur`
- `shadow-lg` → `shadow-lg`

---

## 步驟 6：生成各格式內容

### SCSS 變數（`_tokens.scss`）

```scss
// ================================
// Design Tokens — {Figma 檔案名稱}
// 來源：Figma ({fileId})
// ================================

// Color
$color-{name}: {hex};                    // 實色
$color-{name}: rgba({hex}, {opacity});   // 透明度變體

// Shadow
$shadow-{name}: {box-shadow};

// Typography
$font-family-base:   '{fontFamily}', sans-serif;
$font-size-base:     {fontSize}px;
$font-weight-base:   {fontWeight};
$line-height-base:   {lineHeight};
```

### CSS 變數（`_variables.css`）

```css
/* Design Tokens — {Figma 檔案名稱} */
:root {
  /* Color */
  --color-{name}: {hex};

  /* Shadow */
  --shadow-{name}: {box-shadow};

  /* Typography */
  --font-family-base: '{fontFamily}', sans-serif;
  --font-size-base:   {fontSize}px;
  --font-weight-base: {fontWeight};
  --line-height-base: {lineHeight};
}
```

### JSON token（`tokens.json`，W3C Design Tokens 格式）

```json
{
  "color": {
    "{name}": { "$value": "{hex}", "$type": "color" }
  },
  "shadow": {
    "{name}": { "$value": "{box-shadow}", "$type": "shadow" }
  },
  "typography": {
    "fontFamily": { "$value": "{fontFamily}", "$type": "fontFamily" },
    "fontSize":   { "$value": "{fontSize}px", "$type": "dimension" },
    "fontWeight": { "$value": {fontWeight}, "$type": "fontWeight" },
    "lineHeight": { "$value": {lineHeight}, "$type": "number" }
  }
}
```

### Bootstrap 5 專用（`_bootstrap-overrides.scss`）

語意色對應規則（依 token 名稱判斷）：

| Figma token 名稱含 | Bootstrap 變數 |
|---------------------|----------------|
| `primary`（不含數字） | `$primary` |
| `secondary` | `$secondary` |
| `warning` | `$warning` |
| `danger` / `error` | `$danger` |
| `success` | `$success` |
| `info` | `$info` |
| `white` | `$white` |
| `dark` | `$dark` |
| `light` | `$light` |
| 無對應 | CSS 自訂屬性，加 `// 無 Bootstrap 對應` 備註 |

```scss
// ================================
// Bootstrap 5 SCSS 變數覆寫
// 來源：Figma ({fileId})
// 使用方式：必須在 @import 'bootstrap/scss/bootstrap' 之前引入
// ================================

$primary:  {hex};
$warning:  {hex};
// ... 其他語意色

$box-shadow:    {shadow};
$box-shadow-lg: {shadow-lg};
$box-shadow-sm: {shadow-sm};

$font-family-base:       '{fontFamily}', system-ui, -apple-system, sans-serif;
$font-family-sans-serif: $font-family-base;
$font-size-base:         {fontSize}rem;
$font-weight-base:       {fontWeight};
$line-height-base:       {lineHeight};

// 無 Bootstrap 對應的 token（建議加入 :root）
// --color-{name}: {hex};
```

---

## 步驟 7：寫出檔案

1. 確認輸出目錄存在，不存在則建立
2. 依使用者選擇寫出對應檔案
3. 完成後列出所有產出路徑與 token 統計（色彩 N 個、陰影 N 個、字型 N 個）
