# feedback-widget 規格與策略

> 狀態：草案 v0.2（2026-11-27）。尚未動工，本文件是開發前的共識紀錄。
> v0.2：圖片改為直接上傳成 GitHub 附件，移除圖床 driver 與 GitHub App 驗證（見 `docs/adr/0001-images-as-github-attachments.md`）。
> 佔位網域：`feedback.example-studio.tw`、宿主站 `www.example-studio.tw`，部署時再替換。

## 1. 目標與非目標

### 目標

讓有管理權限的成員在任何頁面「看到問題 → 截圖標註 → 一鍵開 GitHub issue」：

1. 管理員登入後，頁面右下角出現浮動小圓按鈕。
2. 按下後擷取目前頁面畫面，開啟光箱。
3. 在擷圖上畫線、螢光筆、箭頭、文字、框線、塗黑，並填寫標題、描述、類型。
4. 送出後在指定 repo 開一張 GitHub issue，內含標註後圖片、問題內容、回報人、頁面網址、瀏覽器／OS 等環境資料。

設計原則：

- **與宿主低耦合**：widget 不認識 Laravel／Drupal／WordPress／Rust；宿主只負責「判斷誰能用」和「提供 endpoint 與驗證方式」。
- **不重造輪子**：擷圖用 snapdom、標註用 marker.js 3、GitHub 用各語言成熟 client。
- **金鑰不進瀏覽器**：GitHub token 只存在後端。
- **同一份規格、多套實作**：Rust 中繼服務與各語言 framework package 以 `spec/` 的 HTTP 規格與 conformance 測試資料為準。

### 非目標（本版不做）

- 自動偵測並遮蔽個資（先訂約定與 selector 設定，見 §9；實作在 M4）。
- session replay、網路請求紀錄、cookie／localStorage 收集。
- 非管理員（一般訪客）回報。
- 在 issue 上指派 assignee（回報人多半沒有 GitHub 帳號，寫在本文）。
- npm／Packagist 正式發佈（初期以 GitHub release、VCS repository 引用）。
- GitHub Enterprise Server（沒有附件上傳端點，見 §11）。

## 2. 架構總覽

兩種部署模式，**widget 打的 HTTP 規格完全相同**，只差 endpoint 與驗證方式。

```
中繼模式（relay mode）

 宿主站（任意框架）                         feedback.example-studio.tw（Rust, Docker）
 ┌─────────────────────────┐                ┌──────────────────────────────────┐
 │ framework package       │  簽短效 token   │ relay-server (axum)              │
 │  - 判斷管理員            │ ─────────────▶ │  - 驗 token / CORS / 限流         │
 │  - 輸出 <script>         │   (由 widget   │  - 圖片上傳為 GitHub 附件         │
 │  - 簽 HS256 token        │    帶過去)     │  - GitHub client → 開 issue       │
 └─────────────────────────┘                │  - SQLite/Postgres 送出紀錄       │
          │ 載入 widget.js（由 relay 提供）  │  - 提供 /widget.js                │
          ▼                                 └──────────────────────────────────┘
 ┌─────────────────────────┐   POST /v1/feedback (Bearer token)   ▲
 │ widget（瀏覽器）         │ ─────────────────────────────────────┘
 └─────────────────────────┘

內嵌模式（embedded mode）

 宿主站
 ┌──────────────────────────────────────────────┐
 │ framework package（例如 php/laravel）          │
 │  - route + middleware：session 判斷管理員+CSRF │
 │  - php/core：驗證輸入、上傳附件、GitHub 開單   │
 │  - 宿主資料庫存送出紀錄                         │
 └──────────────────────────────────────────────┘
          ▲  POST {host}/_feedback/v1/feedback（session cookie + X-CSRF-TOKEN）
 ┌────────┴────────────────┐
 │ widget（瀏覽器）         │
 └─────────────────────────┘
```

選擇指引：

| 情境 | 建議模式 |
| --- | --- |
| 多個站共用、想集中管理 GitHub 憑證 | 中繼模式 |
| 單一站、不想多維護一個服務 | 內嵌模式 |

## 3. Repo 結構（monorepo）

```
widget/            TypeScript：loader（浮動按鈕）與 editor（擷圖＋標註＋表單）
relay/core/        Rust lib：輸入驗證、token 驗簽、GitHub 附件上傳與開單、issue 格式
relay/server/      Rust bin：axum HTTP、DB（SQLite／Postgres）、限流、靜態提供 widget、圖片備份檔與重試、Dockerfile
php/core/          框架無關 PHP 核心（PSR-7／PSR-17／PSR-18）：與 relay/core 同職責
php/laravel/       Laravel adapter：ServiceProvider、route、middleware、Blade directive、migration
spec/              openapi.yaml、token.md、db-schema.md、masking.md、conformance/
examples/          純 HTML 範例、mock 後端、兩種模式的 docker-compose
SPEC.md            本文件
```

Rust 宿主（例如自己的 Rust 系統）不需另外的 package，直接依賴 `relay/core` crate，在自己的 axum／actix route 中呼叫。

WordPress plugin、Drupal module 之後以 `php/core` 為基礎各做一層薄 adapter（§13 只定介面）。

## 4. 元件職責

### 4.1 widget（瀏覽器）

- **loader**（目標 < 5 KB gzip）：建立 Web Component `<feedback-widget>`（Shadow DOM，樣式隔離），顯示右下角浮動按鈕。按下時才動態 `import()` editor。
- **editor**（數百 KB，延遲載入）：
  1. 依遮蔽 selector 處理後，用 snapdom 擷取 `document.documentElement` 可視區（失敗退回 html-to-image）。
  2. 開光箱，載入 marker.js 3 編輯器（線、手繪、螢光筆、箭頭、文字、框線、塗黑矩形）。
  3. 表單：標題（必填）、描述（選填，多行）、類型（選項由站台設定提供）、「我已確認圖片中沒有未遮蔽的個資」勾選（必勾）。
  4. 若站台設定 `allowUpload: true`，提供「改用上傳圖片」備援（擷圖失敗或要附其他圖時）。
  5. 收集環境資料（§7.3）與最近 20 筆 console error／未處理例外（loader 載入時就開始收集）。
  6. 送出前呼叫 `getToken()`（若有）取得最新 token；送出中按鈕停用防連點；失敗時保留表單與標註結果，可重試。
- 不讀 cookie、localStorage、表單內容；不認識任何宿主框架。
- UI 語系：`zh-TW`、`en`，由 `locale` 決定，預設讀 `<html lang>`。

### 4.2 relay（Rust 中繼服務）

- 驗 token（§6）、CORS（只允許站台設定的 origin）、請求大小與圖片格式檢查。
- 把圖片上傳成目標 repo 的 GitHub 附件，取得附件網址（§11）。
- 組 issue 本文（§8），以 PAT 開 issue、加 label。
- 寫入送出紀錄與圖片備份檔（§10），限流、失敗重試。
- 提供 `/widget.js`（loader）、`/widget/editor.<hash>.js`、`/healthz`。

### 4.3 framework package（以 php 為例）

- 中繼模式：提供「簽 token」與「輸出 script」的 helper（例如 Laravel 的 `@feedbackWidget` Blade directive），不碰 GitHub。
- 內嵌模式：提供與 relay 相同的 `/v1/feedback`、`/v1/config` 端點，用宿主 session 與 CSRF 驗證；內部呼叫 `php/core` 做上傳與開單。
- 「誰算管理員」由宿主以 callback／gate 決定，package 不預設任何角色模型。

## 5. HTTP 規格

完整定義放 `spec/openapi.yaml`，本節為摘要。base path 為 `{endpoint}`，例如中繼模式 `https://feedback.example-studio.tw`，內嵌模式 `https://www.example-studio.tw/_feedback`。

### 5.1 驗證

| 模式 | 方式 |
| --- | --- |
| 中繼 | `Authorization: Bearer <token>`（§6），不帶 cookie（`credentials: 'omit'`） |
| 內嵌 | 宿主 session cookie（同源，`credentials: 'same-origin'`）＋ widget `init({ headers })` 給的 header，例如 `X-CSRF-TOKEN` |

### 5.2 `GET /v1/config`

widget 開啟光箱時取得站台設定。

```json
{
  "site": "example-studio",
  "types": [
    { "id": "bug", "label": { "zh-TW": "錯誤", "en": "Bug" } },
    { "id": "change", "label": { "zh-TW": "修改需求", "en": "Change request" } },
    { "id": "other", "label": { "zh-TW": "其他", "en": "Other" } }
  ],
  "allowUpload": true,
  "maxImageBytes": 5242880,
  "maskSelectors": ["[data-feedback-mask]", ".rr-mask"],
  "ignoreSelectors": ["[data-feedback-ignore]", ".rr-block"]
}
```

### 5.3 `POST /v1/feedback`

`multipart/form-data`：

| part | 型別 | 說明 |
| --- | --- | --- |
| `image` | `image/png`／`image/jpeg`／`image/webp` | 標註後的圖片（只送標註後版本，不送原圖，避免未遮蔽版本外流） |
| `payload` | `application/json` | 見下 |

```json
{
  "title": "報名表的日期欄位跑版",
  "description": "手機寬度下日期欄位擠在一起……",
  "type": "bug",
  "consent": true,
  "source": "capture",
  "page": { "url": "https://www.example-studio.tw/admin/orders?id=12", "title": "訂單管理", "referrer": "" },
  "env": {
    "userAgent": "Mozilla/5.0 …",
    "browser": { "name": "Chrome", "version": "131" },
    "os": { "name": "macOS", "version": "15.1" },
    "viewport": { "width": 1440, "height": 900 },
    "screen": { "width": 2560, "height": 1440 },
    "devicePixelRatio": 2,
    "language": "zh-TW",
    "timezone": "Asia/Taipei",
    "clientTime": "2026-11-23T10:15:00+08:00"
  },
  "release": "v2026.11.0",
  "consoleErrors": [
    { "time": "2026-11-23T10:14:51+08:00", "message": "TypeError: …", "stack": "…" }
  ],
  "widgetVersion": "0.1.0"
}
```

- `source`：`capture`（擷圖）或 `upload`（使用者上傳）。
- 回報人資訊**不由 payload 提供**：中繼模式取自 token，內嵌模式取自宿主 session，避免前端偽造。
- `release` 由宿主 `init({ release })` 提供，選填。

成功回應 `201`：

```json
{ "id": "01JD…", "issue": { "number": 321, "url": "https://github.com/owner/repo/issues/321" } }
```

錯誤回應一律：

```json
{ "error": { "code": "token_expired", "message": "…" } }
```

| HTTP | code（範例） | widget 行為 |
| --- | --- | --- |
| 400 | `invalid_payload` | 顯示訊息 |
| 401 | `token_missing`／`token_invalid`／`token_expired` | 若有 `getToken` 重取一次後重送 |
| 403 | `origin_not_allowed`／`forbidden` | 顯示訊息 |
| 413 | `image_too_large` | 提示壓縮或改 JPEG |
| 422 | `consent_required`／`unsupported_image` | 顯示訊息 |
| 429 | `rate_limited`（帶 `Retry-After`） | 顯示等待秒數 |
| 502 | `upload_failed`／`github_failed` | 保留表單，可重試；紀錄已存，後端也會重試 |

### 5.4 靜態資源（僅中繼模式）

- `GET /widget.js`：loader，`Cache-Control: max-age=300`。
- `GET /widget/editor.<hash>.js`：editor，immutable 長快取。
- `GET /healthz`。

## 6. Token（中繼模式）

- 格式：JWT，`HS256`，每個站台一把 secret（≥ 32 bytes 隨機）。
- header 加 `kid` = 站台代號，relay 依此找 secret。

| claim | 必填 | 說明 |
| --- | --- | --- |
| `iss` | ✓ | 站台代號，須等於 `kid` |
| `aud` | ✓ | 固定 `feedback-relay` |
| `sub` | ✓ | 宿主的使用者 id（字串） |
| `name` | ✓ | 顯示名稱 |
| `email` |  | 選填 |
| `roles` |  | 字串陣列，僅供記錄 |
| `iat`、`exp` | ✓ | `exp - iat` ≤ 900 秒，relay 容許 60 秒時鐘誤差 |
| `jti` | ✓ | 隨機 id，relay 在 `exp` 前拒絕重複使用（防重放） |

token 只代表「這個人此刻是此站管理員」。是否為管理員由宿主判斷，relay 不知道宿主的角色模型。

widget 取得 token 的兩種方式（都支援，`getToken` 優先）：

```js
FeedbackWidget.init({ endpoint, token: '<jwt>' })
FeedbackWidget.init({ endpoint, getToken: async () => (await fetch('/api/me').then(r => r.json())).feedbackToken })
```

各語言簽 token 的範例片段放 `spec/token.md`：PHP（firebase/php-jwt）、WordPress（`wp_get_current_user` + `current_user_can`）、Drupal（`\Drupal::currentUser()`）、Rust（jsonwebtoken）、Node。

## 7. widget 初始化

### 7.1 JS API（主要）

```js
FeedbackWidget.init({
  endpoint: 'https://feedback.example-studio.tw', // 必填
  getToken: async () => '…',                     // 中繼模式擇一
  token: '…',                                    // 中繼模式擇一
  headers: { 'X-CSRF-TOKEN': '…' },               // 內嵌模式
  credentials: 'omit',                            // 中繼 omit；內嵌 same-origin（預設依是否同源自動判斷）
  site: 'example-studio',                         // 選填，僅顯示用；實際以 token／後端為準
  release: 'v2026.11.0',
  locale: 'zh-TW',
  position: 'bottom-right',                       // bottom-right | bottom-left
  zIndex: 2147483000,
  maskSelectors: [],                              // 附加到站台預設清單
  ignoreSelectors: [],
  onSubmitted: (result) => {},
})
FeedbackWidget.destroy()
FeedbackWidget.open()                             // 宿主可自訂觸發點
```

### 7.2 script 標籤簡寫（給 WordPress／Drupal 塞一行）

```html
<script src="https://feedback.example-studio.tw/widget.js"
        data-endpoint="https://feedback.example-studio.tw"
        data-token="<jwt>"
        data-locale="zh-TW"
        data-auto-init defer></script>
```

`data-*` 對應 JS API 的同名選項（kebab-case）；需要 `getToken` 或 callback 時改用 JS API。

### 7.3 收集的環境資料

收集：頁面網址與標題、referrer、User-Agent 原文與解析後瀏覽器／OS（用輕量 UA parser 或 `navigator.userAgentData`）、viewport、screen、devicePixelRatio、語言、時區、客戶端時間、宿主提供的 release、widget 版本、最近 20 筆 console error 與 `error`／`unhandledrejection`。

不收集：cookie、localStorage／sessionStorage、表單輸入值、網路請求、DOM 原始碼。

## 8. GitHub issue 格式

以 `spec/conformance/` 的測試資料為唯一標準，Rust 與 PHP 產出必須逐字一致。

- **標題**：`[{type label}] {title}`，例如 `[錯誤] 報名表的日期欄位跑版`。前綴可在站台設定關閉。
- **label**：站台設定的固定 label（例如 `feedback`）＋類型對應的 label。
- **本文**：

````markdown
{description}

![screenshot]({image url})

**回報人**：王小明（id: 42）
**頁面**：[訂單管理](https://www.example-studio.tw/admin/orders?id=12)

<details>
<summary>環境資料</summary>

| 項目 | 值 |
| --- | --- |
| 瀏覽器 | Chrome 131 |
| 作業系統 | macOS 15.1 |
| Viewport | 1440×900 @2x |
| 螢幕 | 2560×1440 |
| 語言／時區 | zh-TW／Asia/Taipei |
| 回報時間 | 2026-11-23 10:15 (+08:00) |
| 版本 | v2026.11.0 |
| User-Agent | `Mozilla/5.0 …` |
| 來源 | 擷圖（widget 0.1.0） |

</details>

<details>
<summary>Console errors（3）</summary>

```text
…
```

</details>

<!-- feedback-widget: {"id":"01JD…","site":"example-studio"} -->
````

跳脫規則：

- 使用者輸入的 `@xxx` 轉成 `@​xxx`，避免通知不相干的 GitHub 使用者。
- 表格內的值跳脫 `|` 與換行，HTML 實體化 `<`、`>`。
- code fence 長度取「內容中最長連續反引號 + 1」（至少 3）。
- 圖片網址只接受 GitHub 附件端點回傳的 `https://` 網址。
- 尾端 HTML 註解帶送出紀錄 id，方便日後從 issue 反查。email 不寫入 issue（只存送出紀錄），除非站台設定 `showReporterEmail: true`。

## 9. 個資遮蔽約定

業界沒有 W3C 標準，各家 session replay／analytics 工具各自訂約定。本專案採用「自有屬性為主，預設也認得常見工具的屬性」：

| 行為 | 預設 selector |
| --- | --- |
| **mask**（擷圖時以實心色塊覆蓋，保留版面） | `[data-feedback-mask]`、`.rr-mask`、`[data-private]`、`[data-sentry-mask]`、`.sentry-mask`、`[data-clarity-mask="true"]`、`[data-hj-suppress]`、`input[type=password]` |
| **ignore**（擷圖時整塊移除或留白） | `[data-feedback-ignore]`、`.rr-block`、`[data-sentry-block]`、`.ph-no-capture`，以及 widget 自身 |

- 來源：rrweb（`rr-block`／`rr-mask`，Sentry、PostHog replay 底層）、Sentry、LogRocket（`data-private`）、Microsoft Clarity、Hotjar、PostHog。
- 清單可由站台設定（§5.2）與 `init()` 增補，也可關閉預設清單。
- 實作上不自寫遮蔽邏輯：mask／ignore 交給 snapdom 的 exclude／filter 選項，mask 再加一層 CSS 色塊（API 細節見 §15 待查證）。
- 時程：M1 先完成 ignore（移除 widget 自身與 ignore 清單）與手動塗黑工具，並在文件中寫明約定；mask 的自動覆蓋與自動偵測身分證／電話等，在 M4 做。
- 在自動遮蔽完成前，送出前必須勾選「我已確認圖片中沒有未遮蔽的個資」。

## 10. 送出紀錄（資料表）

完整 schema 放 `spec/db-schema.md`。Rust 用 sqlx，migration 只寫 SQLite 與 Postgres 共通的 SQL，以 `DATABASE_URL` 切換；PHP 內嵌模式用宿主資料庫（Laravel 一個 migration＋model）。

`feedback_submissions`

| 欄位 | 型別 | 說明 |
| --- | --- | --- |
| `id` | text（ULID） | 主鍵 |
| `site` | text | 站台代號 |
| `reporter_id`／`reporter_name`／`reporter_email` | text | 來自 token 或 session |
| `title`、`type` | text |  |
| `page_url` | text |  |
| `payload_json` | text | 完整 payload（不含圖片） |
| `image_url` | text | GitHub 附件網址，上傳成功後才有值 |
| `image_backup_path` | text | 圖片備份檔路徑；開單成功或備份到期刪除後清空 |
| `status` | text | `pending`／`created`／`failed` |
| `issue_number`、`issue_url` | int／text |  |
| `attempts`、`last_error` | int／text | 重試用 |
| `created_at`、`updated_at` | timestamp |  |

另有 `token_jti`（防重放，可定期清除過期資料）。限流用記憶體計數器（單機）即可，不寫資料庫。

流程：收件 → 圖片存成備份檔並寫 `pending` → 上傳成 GitHub 附件（成功就記下 `image_url`）→ 開 issue → 更新為 `created` 並立刻刪除備份檔。

- 上傳附件或開 issue 失敗時記為 `failed` 並回 502；背景工作以指數退避重試 `failed`（最多 5 次），成功時不會重複開單（以紀錄 id 判斷）。
- 紀錄已有 `image_url` 時只重試開單，不重傳圖片：附件上傳後無法刪除，重傳會留下沒人引用的檔案。
- 5 次都失敗後不再自動重試，備份檔留到收件滿 7 天（`backup_retention_days`，可設定）才刪除，供人工補救；紀錄維持 `failed`。
- 備份檔位置：relay 放 `{data_dir}/pending/`；內嵌模式放宿主的私有磁碟（例如 Laravel `local` disk，不是 `public`）。一律不對外提供，檔名為 `{ULID}.{ext}`。
- 系統裡因此只會留著「還沒成功送出」的圖片。

## 11. 圖片附件（GitHub attachments）

圖片直接上傳成目標 repo 的 GitHub 附件，不經任何圖床；issue 本文引用回傳的附件網址。決策背景與取捨見 `docs/adr/0001-images-as-github-attachments.md`。

```
POST https://uploads.github.com/user-attachments/assets
       ?name={ULID}.{ext}&content_type={mime}&repository_id={repo 的數字 id}
Authorization: token {PAT}
Content-Type: application/octet-stream
Content-Length: {bytes}
Accept: application/vnd.github+json

{圖片原始位元組}

→ 201 { "url": "https://github.com/user-attachments/assets/{uuid}" }
```

- **來源**：與 `gh issue create --attach`（gh v2.99.0，2026-09-01 起）是同一個端點。它不在 GitHub 官方 REST API 文件中，以 cli/cli 的 `internal/attachments/client.go` 為參考實作。
- **不依賴 `gh` 執行檔**：Rust（reqwest）與 PHP（PSR-18）各自直接打這個端點；開 issue 走正式 REST API `POST /repos/{owner}/{repo}/issues`。
- **`repository_id`**：啟動時以 `GET /repos/{owner}/{repo}` 取得並快取。
- **憑證**：只接受使用者 token（classic 或 fine-grained PAT），且該帳號對 repo 要有 write 權限。GitHub App installation token 會得到 404；權限不足也是 404，不是 403。
- **檔名**：由核心產生 `{ULID}.{ext}`，不含使用者可控字串。
- **限制**：GitHub 端圖片上限 10 MB（本專案預設 5 MB）；不支援 GitHub Enterprise Server；被限流時回 429 並帶 `Retry-After`。
- **可見度**：附件跟隨 repo 的可見度（尚未實測，見 §17）；目標 repo 若是公開的，擷圖就是公開的。relay 啟動時偵測到公開 repo 會印警告。
- **無法刪除**：GitHub 沒有刪除附件的 API，網站上也沒有刪除按鈕。誤傳時的處理見 §15。
- **端點變動的偵測**：conformance 之外另有一個對真實測試 repo 的上傳檢查，端點行為改變時能第一時間發現。

## 12. 設定檔（relay）

TOML，機密以 `${ENV}` 引用環境變數，啟動時展開，缺少就啟動失敗。

```toml
[server]
listen = "0.0.0.0:8080"
public_base_url = "https://feedback.example-studio.tw"
database_url = "${DATABASE_URL}"        # sqlite:///data/feedback.db 或 postgres://…
max_image_bytes = 5242880
data_dir = "/data"                      # 圖片備份檔放 {data_dir}/pending/
backup_retention_days = 7               # 沒送成功的備份檔保留天數

[sites.example-studio]
secret = "${SITE_EXAMPLE_STUDIO_SECRET}"
origins = ["https://www.example-studio.tw", "https://admin.example-studio.tw"]
repo = "owner/repo"
labels = ["feedback"]
title_prefix = true
show_reporter_email = false
allow_upload = true
rate_limit_per_minute = 5
types = [
  { id = "bug",    label = { "zh-TW" = "錯誤",     en = "Bug" },            github_label = "bug" },
  { id = "change", label = { "zh-TW" = "修改需求", en = "Change request" }, github_label = "enhancement" },
  { id = "other",  label = { "zh-TW" = "其他",     en = "Other" } },
]
mask_selectors = []
ignore_selectors = []

[sites.example-studio.github]
token = "${GITHUB_TOKEN}"               # 專用機器帳號的 PAT，限定單一 repo，需 write 權限
```

`php/laravel` 以 `config/feedback-widget.php` 提供同義設定（一個站台，不需要 `sites` 層）。

## 13. 宿主整合指南（摘要）

宿主要做的事，不論哪種框架都一樣：

1. 判斷目前使用者是否可回報（管理員）；不是就什麼都不輸出。
2. 中繼模式：簽 token（§6），輸出 `<script src="{relay}/widget.js">`，並呼叫 `FeedbackWidget.init({ endpoint, getToken })`。
3. 內嵌模式：掛上 package 的 route，輸出 widget script，並呼叫 `init({ endpoint: '/_feedback', headers: { 'X-CSRF-TOKEN': … } })`。
4. 想從 widget 擷圖中排除的區塊，加上 `data-feedback-mask`／`data-feedback-ignore`。
5. 靜態頁（沒有伺服器 template）：由既有的「取得登入狀態 API」回傳是否可回報與 token，前端依此動態載入 widget。

各框架 adapter 的介面：

| 框架 | 中繼模式 | 內嵌模式 | 時程 |
| --- | --- | --- | --- |
| Laravel | `@feedbackWidget` directive、`FeedbackWidget::token($user)` | `Route::feedbackWidget()`、middleware、migration | M1／M2 |
| WordPress | plugin：`wp_footer` 輸出、`current_user_can` 判斷、設定頁 | REST route `feedback/v1` | M4 |
| Drupal | module：`hook_page_attachments`、permission `submit feedback` | route + controller | M4 |
| Rust | 依賴 `relay/core` 的 `sign_token` | 依賴 `relay/core` 的 handler，掛進自己的 router | M2 |

## 14. 選型與替代方案

| 項目 | 採用 | 理由 | 考慮過的替代方案 |
| --- | --- | --- | --- |
| 買或做 | 自己做，用現成套件組合 | 跨多種框架、圖片與憑證自己掌握、沒有月費 | Marker.io、BugHerd、Usersnap、Jam（商業、月費、圖存在對方）；Sentry User Feedback（需要 Sentry；擷圖用螢幕分享 API） |
| 擷圖 | snapdom，失敗時退回 html-to-image | 不跳權限視窗、可在擷圖前排除和遮蔽 DOM | `getDisplayMedia`（像素精準，但每次要選分頁、只能事後遮蔽）、html2canvas（較舊、較慢） |
| 標註 | marker.js 3 | 不依賴框架，工具最齊全；linkware 授權（免費版有小 logo，可付費拿掉），已確認可接受 | Filerobot（React、bundle 大）、tui-image-editor（停止維護）、fabric.js／Konva（要自己做 UI） |
| 擴充方式 | Web Component＋Shadow DOM | 和宿主的 CSS、JS 隔離 | iframe（擷不到宿主頁面）、直接插入 DOM（樣式互相干擾） |
| relay 語言 | Rust：axum、tokio、octocrab、jsonwebtoken、sqlx、reqwest | 單一執行檔、省資源；作者想用 Rust 練功 | TypeScript＋Hono（可共用型別，但不符合目標） |
| GitHub 驗證 | PAT：專用機器帳號的 fine-grained PAT，限定單一 repo | 附件上傳端點只接受使用者 token；專用帳號讓 issue 有固定的 bot 身分 | GitHub App（installation token 傳不了附件，GitHub 開放後再評估） |
| 圖片 | 上傳成 GitHub 附件 | 不必架圖床、不必公開檔案、少一套金鑰；圖片跟隨 repo 可見度 | 可抽換的圖床 driver（local、S3、imgur、通用 HTTP：圖片必須是公開網址，已否決，見 ADR 0001）；把圖 commit 進 repo（和程式碼耦合，已否決） |
| 資料庫 | sqlx，SQLite 起步，保留 Postgres 相容 | 不用另外架資料庫，之後可擴充 | 只用 SQLite |
| PHP 核心 | PSR-7／PSR-18 框架無關核心＋薄 adapter | Laravel、WordPress、Drupal 共用同一份邏輯 | 每個框架各寫一份 |

## 15. 安全與個資

- GitHub token 只存在後端；widget 原始碼與回應都不得包含它們。
- 中繼模式：檢查 token 簽章、`aud`、`exp` 與 `jti` 防重放；CORS 只允許站台列出的 origin，並同時檢查 `Origin` header。
- 內嵌模式：用宿主 session 加 CSRF 驗證，並在伺服器端再判斷一次權限，不信任前端。
- 上傳限制：預設 5 MB；以檔頭 magic bytes 驗證是 PNG、JPEG 或 WebP，不採信宿主宣告的 MIME；尺寸上限 8192×8192。
- 限流：每位使用者每分鐘 5 次（可設定），超過時回 429。
- 圖片是 GitHub 附件，上傳後無法自行刪除：在遮蔽功能完成前靠送出前的勾選確認。誤傳個資時，先編輯 issue 移除圖片那一行，再向 GitHub 客服申請刪除該附件網址。
- 目標 repo 建議設為私有；公開 repo 的擷圖等於永久公開，relay 啟動時會印警告。
- 圖片備份檔只在開單成功前存在，放在不對外的目錄，最長保留 7 天（§10）。
- PAT 用專用機器帳號產生，限定單一 repo、設定到期日；到期前要換新並更新環境變數。
- issue 本文的跳脫規則見 §8。
- 送出紀錄含 email，保存期限由站台設定（預設 365 天），到期時清空 `reporter_email` 與 `payload_json`。

## 16. 里程碑

| 里程碑 | 內容 | 完成標準 |
| --- | --- | --- |
| **M0** | 本規格、`spec/openapi.yaml`、`spec/token.md`、`spec/db-schema.md`、`spec/masking.md`、conformance 測試資料初版 | 經作者 review |
| **M1** | widget（loader＋editor：snapdom＋marker.js＋表單、ignore 清單、手動塗黑、上傳備援）；relay（token、CORS、GitHub 附件上傳、PAT、圖片備份檔、SQLite、issue 格式、限流）；`examples/` 純 HTML＋docker-compose | 在範例頁送出一張含圖的 issue 到測試 repo；Rust 通過 conformance |
| **M2** | `php/core`＋`php/laravel`（兩種模式）；`relay/core` 給 Rust 宿主內嵌使用；Postgres 實測 | Laravel 範例站兩種模式都能開單；PHP 通過 conformance |
| **M3** | 失敗重試背景工作；備份檔到期清除；公開 repo 啟動警告；對真實測試 repo 的附件上傳檢查 | 以 mock server 模擬上傳失敗與開單失敗，重試與清除都有整合測試 |
| **M4** | 自動 mask 覆蓋與個資偵測；WordPress plugin、Drupal module | 依各自的 issue 驗收 |

第一個宿主（作者的 Laravel 站）的整合，另外在該站 repo 開 issue 處理，不寫進本專案。

## 17. 待查證（動工前確認）

- marker.js 3：vanilla 用法、工具清單（螢光筆、塗黑矩形、文字）、匯出 PNG 的 API、Shadow DOM 內能否正常運作、linkware 授權條款原文。
- snapdom：`exclude`／`filter` 選項的實際名稱與行為、能否只擷取可視區、跨網域圖片與字型的處理、失敗時的錯誤型態。
- html-to-image：作為退路時的 `filter` 行為。
- GitHub 附件端點：fine-grained PAT 需要勾選哪些 repo 權限才能上傳；`Authorization` 用 `token` 或 `Bearer` 是否都可以。
- GitHub 附件可見度：私有 repo 的附件網址，未登入或沒有 repo 權限的人是否打不開（實測）。
- GitHub 附件刪除：向 GitHub 客服申請刪除的實際流程與所需時間。
- UA 解析：選用 `ua-parser-js`（注意 v2 改為 AGPL 授權）或 `bowser`，或只用 `navigator.userAgentData`；先確認授權。
