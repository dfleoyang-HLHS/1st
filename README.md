# Vibe Portal — 花蓮高中 校園連結入口網站

給花蓮高中「教學」與「行政」用途連結的入口網站。前端是純靜態 HTML，連結資料存在 GitHub repo 的 `db.json`，管理員登入後可以新增／刪除／拖曳排序連結，並透過 Cloudflare Worker 把異動寫回 GitHub。

## 架構

```
瀏覽器 (index.html)
   │  GET  /api/db   (公開，讀取連結清單)
   │  PUT  /api/db   (需要管理密碼，寫入連結清單)
   ▼
Cloudflare Worker (work.js)
   │  帶著 GITHUB_TOKEN 呼叫 GitHub Contents API
   ▼
GitHub repo 的 db.json
```

- 前端只認識 Cloudflare Worker 的網址，不直接碰 GitHub，也不儲存任何密碼或 Token。
- 管理密碼只存在 Cloudflare Worker 的環境變數，前端每次登入都是把密碼送給 Worker 驗證。
- GitHub Token 只存在 Worker，用來代替管理員寫入 `db.json`。

## 檔案說明

| 檔案 | 說明 |
|---|---|
| `index.html` | 靜態前端頁面：連結目錄展示、深／淺色主題、管理員登入、新增／刪除／拖曳排序、匯入匯出備份 |
| `db.json` | 連結資料的本機備份格式範例（`exportJSON()` 下載下來的格式），**不是**網站實際讀取的資料來源——網站一律即時向 Cloudflare Worker 拿最新資料 |
| `cloudflare/work.js` | Cloudflare Worker 原始碼：提供 `/api/db` 的 GET（公開讀取）與 PUT（管理員寫入），負責跟 GitHub API 溝通 |

> 目前資料夾內有 `01/`、`02/` 兩份幾乎相同的副本（差異只在 `db.json` 的資料筆數），請依你實際部署／使用的那一份為準；建議之後整理成單一版本並用 Git 做版本控管，避免混淆。

## 部署方式

### 1. 前端（GitHub Pages）
把 `index.html` 放進要公開的 GitHub repo（例如 `dfleoyang-hlhs.github.io` 或子 repo），啟用 GitHub Pages 即可。

### 2. 資料儲存（GitHub repo）
另外準備一個 repo（或同一個也可以）放 `db.json`，Worker 會用 GitHub Contents API 讀寫這個檔案。

### 3. Cloudflare Worker
1. 到 Cloudflare Dashboard 建立一個 Worker，把 `cloudflare/work.js` 的內容貼進去部署。
2. 在 Worker 的 **Settings → Variables and Secrets** 設定以下環境變數：

| 變數 | 說明 | 範例 |
|---|---|---|
| `GITHUB_TOKEN` | GitHub Personal Access Token，需要 `Contents: Read and write` 權限（建議用 Fine-grained token，只授權給存放 `db.json` 的 repo） | — |
| `ADMIN_PASSWORD` | 管理員登入密碼 | — |
| `GITHUB_OWNER` | GitHub 帳號／組織名稱 | `dfleoyang-hlhs` |
| `GITHUB_REPO` | 存放 `db.json` 的 repo 名稱 | `vibe-portal` |
| `GITHUB_PATH` | `db.json` 在 repo 裡的路徑（預設 `db.json`） | `db.json` |
| `GITHUB_BRANCH` | 分支（預設 `main`） | `main` |
| `ALLOWED_ORIGIN` | 允許呼叫 API 的前端網域（可用逗號分隔多個） | `https://dfleoyang-hlhs.github.io` |

3. 部署後把 Worker 網址填回 `index.html` 裡的 `workerUrl` 變數（目前寫死在 `<script>` 開頭）。

## 功能

- 教學／行政兩欄式連結目錄，深色／淺色主題切換（記在瀏覽器 localStorage）
- 管理員登入後可：新增連結、刪除連結（需二次確認）、拖曳排序
- 修改後需手動按「儲存至 GitHub」才會真的寫回去（未儲存時畫面上方會出現提示橫幅）
- 支援匯出 JSON 備份、匯入 JSON 還原（匯入時會做欄位格式驗證）

## 已知限制 / 待辦

- `ADMIN_PASSWORD` 目前沒有登入失敗次數限制，建議在 Cloudflare 加上 Rate Limiting Rule，或用 KV 實作節流。
- `01/`、`02/` 重複資料夾建議整理成單一版本，避免之後改錯地方。
- Worker 的 GitHub Token 若過期／被撤銷，前端會顯示「無法讀取資料」，需回 Cloudflare Dashboard 更新 `GITHUB_TOKEN`。

## 安全性修正紀錄

- 登入流程會先重新抓取最新遠端資料，避免用空白的本機狀態覆寫 GitHub 上的資料。
- 連結的 `id` 欄位在渲染前一律經過 HTML escape，並改用事件委派取代內嵌 `onclick`，避免被竄改過的資料造成儲存型 XSS。
- 匯入 JSON 與 Worker 的 PUT API 都會驗證每筆連結的欄位型別，避免壞資料寫入或造成畫面崩潰。
- Worker 密碼比對改為常數時間比較，降低 timing attack 風險。
