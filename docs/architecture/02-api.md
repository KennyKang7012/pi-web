# 02 · API 路由目錄

`app/api/**/route.ts` 是 Next App Router 的 handler。所有路由都經過
[`proxy.ts`](01-runtime.md#21-中介層-proxyts) 的主機/來源/密碼檢查；
下表只描述各路由自己的邏輯。

## 共通慣例

- **動態資料**標 `export const dynamic = "force-dynamic"`，回應帶
  `Cache-Control: no-store`。少數刻意長快取的例外會標明。
- **請求格式**：需要 JSON 主體的路由先檢查 `hasJsonContentType()`
  （`application/json` 或 `application/*+json`），不合回 415。
- **安全檢查順序**（寫入類路由幾乎一致）：
  1. `isApiRequestAllowed()` → 403 `request-denied`
  2. Content-Type → 415
  3. 主體形狀 → 400 `invalid-request`
  4. `cwd` 驗證（存在、是目錄、在允許根目錄內）→ `cwd-invalid` / `cwd-not-directory` / `cwd-denied`
  5. 功能層檢查（專案信任、MCP 是否停用…）→ 各自的 `reason`
- **錯誤形狀**：`{ error: string }`，需要機器判讀時再帶
  `reason`（英文代碼，前端對應 i18n）或 `code`。部分路由回傳
  `{ error, reason, ...diagnostics }`。
- **長流程**（MCP Test、sign-in、catalog refresh）有自己的逾時與 deadline，
  見 [05-config-and-mcp.md](05-config-and-mcp.md)。

---

## Agent（執行 agent 回合）

|路由|方法|說明|
| --- | --- | --- |
|`/api/agent/new`|POST|建立全新 session 並（可選）立即送出第一個命令。主體：`{ cwd, type: "prompt" \| "ensure_session", message?, toolNames?, provider?, modelId?, thinkingLevel? }`。`ensure_session` 只建立 runtime（讓前端查命令），不送 prompt。回傳 pi 的真實 session id 與啟動時的 model / thinking。`cwd` 不存在回 400；prompt 被拒時帶 `code: "prompt_rejected", accepted: false`。|
|`/api/agent/[id]`|POST|對既有 session 送任何命令（見下方命令表）。`set_tools` 走 `setRpcSessionTools()`，可能重建 wrapper（跨越 Chat only 邊界或清掉 pin 時）。未執行中的 session 會先載入檔案再啟動。|
|`/api/agent/[id]`|GET|`{ running, state }`；未執行中回 `{ running: false }`。|
|`/api/agent/[id]/events`|GET|SSE 事件串流（`text/event-stream`）。詳見 [01-runtime.md §4](01-runtime.md#4-sse-串流libagent-event-streamts)。|
|`/api/agent/[id]/lease`|POST|續租選取中 session 的 liveness lease，回 `{ renewed }`。|
|`/api/agent/[id]/bash-output`|GET|讀取「此 session 引用過的」bash 暫存輸出檔。`?path=<abs>&download=1` 可串流下載（不進記憶體）。inline 顯示有大小上限。|
|`/api/agent/running`|GET|輪詢快照：清單世代、running / awaitingInput / suppressed 的 session id、ui-state 修訂號。|

### `POST /api/agent/[id]` 的命令

分派在 `lib/rpc-manager.ts` 的 `AgentSessionWrapper.send()`（見
[01-runtime.md §3.3](01-runtime.md#33-sendcommand命令分派)）：

|類別|命令|備註|
| --- | --- | --- |
|回合|`prompt`、`steer`、`follow_up`、`abort`、`clear_queue`|`prompt` 需要准入與 MCP 準備；`steer` / `follow_up` 進佇列|
|模型|`set_model`、`set_thinking_level`|session 級別，不寫 `settings.json`；thinking 可在串流中改，下一個請求生效|
|分支|`fork`、`fork_branch`、`clone`、`navigate_tree`|`fork`/`fork_branch` 另開 `SessionManager`，不碰執行中的 runtime；`clone` 拒絕 running|
|壓縮|`compact`、`abort_compaction`|manual compact 是阻塞式 POST|
|工具|`get_tools`、`set_tools`、`reload`|`reload` 拒絕於正在跑的 session|
|資訊|`get_state`、`get_commands`、`get_session_stats`、`get_last_assistant_text`|`get_state` 帶 exact system prompt 等|
|設定|`set_session_name`、`set_auto_compaction`、`set_auto_retry`||
|Shell|`bash`、`abort_bash`|project command environment（已消毒）|
|Extension UI|`extension_ui_response`、`extension_ui_input`|回答 extension 的對話框 / 自訂面板|

---

## Sessions（清單、內容、分支、匯出）

|路由|方法|說明|
| --- | --- | --- |
|`/api/sessions`|GET|清單。`?force=1` 強制重掃；`?summary=1` 只回 header/stat 級別的資料，讓側邊欄先畫（首屏的檔案 mtime 也在這批）。回傳 sessions 與 running / awaitingInput 等 id。|
|`/api/sessions/[id]`|GET|單一 session（執行中 wrapper 的記憶體狀態，否則讀檔）。含 `toolNames` 等供前端顯示目前工具選擇。|
|`/api/sessions/[id]`|PATCH|改名（寫 `session_info`）。|
|`/api/sessions/[id]`|DELETE|刪除一個 session（含 subagent，並把留下的子孫重指到最近的存活祖先）。執行中的保留。|
|`/api/sessions/delete`|POST|`{ ids }` 批次刪除（專案「Delete all sessions」）。語意同 DELETE。|
|`/api/sessions/[id]/context`|GET|一個 leaf 的上下文分頁。`?leafId=&tail=50&before=` —— 預設取最後 50，`before` 往上翻頁。每頁帶 `turnsBefore` 讓縮圖編號與伺服器一致。|
|`/api/sessions/[id]/state`|GET|執行中 wrapper 的即時狀態（重新載入後接回執行中的回合）。|
|`/api/sessions/[id]/fork`|POST|**在磁碟上**把該 session 目前分支複製成新 `.jsonl`（`forkSessionBranch()`）。不建立 AgentSession、不啟動、不關閉 wrapper，所以執行中也能 fork。回傳複製品的清單列 `relation: { kind: "fork", originSessionId }`。拒絕原因：`not_found`、`unsaved`、`empty`、`subagent`。|
|`/api/sessions/[id]/export`|GET|匯出 HTML。委派 pi 的 export helper 後，把它產生的遞迴樹輔助函式改成迭代版，避免極深的線性 session 在瀏覽器爆 call stack。|
|`/api/sessions/[id]/auto-name`|POST|產生 session 標題（用 pi 的 session title 產生器）。|
|`/api/sessions/[id]/entries/[entryId]/tool-result-image`|GET|`?blockIndex=` 取出某個 tool result 區塊的影像。回應 `private, max-age=31536000, immutable`（entry id + block index 永久指向同一份位元組）。錯誤回應不帶快取標頭。|
|`/api/sessions/[id]/entries/[entryId]/thinking`|GET|`?blockIndex=` 取出單一 assistant 訊息的 thinking 區塊文字。|
|`/api/sessions/search`|GET|搜尋 session（索引使用者與 assistant 訊息）。|
|`/api/sessions/ui-state`|GET / POST|Pi Web 的側邊欄狀態（釘選、封存、專案順序與名稱）。詳見 [03-frontend.md](03-frontend.md#41-sidebar-的伺服器端狀態釘選--封存--專案順序)。|

---

## 檔案、目錄與 Git

|路由|方法|說明|
| --- | --- | --- |
|`/api/files/[...path]`|GET|`?type=list \| read \| download \| meta \| preview \| watch`。讀取受允許根目錄限制；`?sessionId=` 可讀一個在該 session 出現過、但位於根目錄之外的確切路徑。|
|`/api/files/[...path]`|POST|`?type=upload \| upload-check \| allow-link`。上傳單檔上限 25 MB、總量 100 MB（`MAX_UPLOAD_FILE_BYTES` / `MAX_UPLOAD_TOTAL_BYTES`）；`upload-check` 讓 Explorer 先問要覆蓋或跳過；`allow-link` 授權一個指向根目錄之外的目錄連結。|
|`/api/cwd/validate`|POST|驗證/選取一個 cwd，通過則 `allowFileRoot()`。|
|`/api/cwd/browse`|GET / POST|目錄瀏覽（cwd 選擇器）；POST 建立子目錄。|
|`/api/default-cwd`|POST|建立 `~/pi-cwd/YYYYMMDD`（本地日期）並回傳路徑；前端再走 `/api/cwd/validate` 選取它。|
|`/api/home`|GET|使用者家目錄。|
|`/api/file-index`|GET|`@` 提及用的檔案清單（模糊比對在 `lib/file-fuzzy.ts`）。|
|`/api/git/status`|GET|某 cwd 的變更檔清單。|
|`/api/git/diff`|GET|單一變更檔的 diff。|
|`/api/worktrees`|GET / POST / DELETE|列出 / 建立 / 移除 git worktree。新 worktree 在 `<repoRoot>-worktrees/<sanitized-branch>`；移除 dirty worktree 回 `409 { dirty: true }`，前端確認後帶 `force` 重試。與 `/api/files` 同一套允許根目錄規則。|
|`/api/open-in-explorer`|GET / POST|GET 回報是否支援；POST 在作業系統檔案總管開啟一個 cwd（**僅限 loopback**）。|

## 終端機

|路由|方法|說明|
| --- | --- | --- |
|`/api/terminal`|POST|建立終端機 session（帶隨機 id 與 cwd；同 id + cwd 是 idempotent）。|
|`/api/terminal/[id]`|GET|`{ id, cwd }`；關閉後 404。|
|`/api/terminal/[id]`|POST|`input` / `resize`。輸入與 resize 都會序列化；相鄰輸入會批次，避免每個按鍵一次往返。|
|`/api/terminal/[id]`|DELETE|終止（先等建立與在途輸入，再刪 PTY）。|
|`/api/terminal/[id]/events`|GET|SSE 輸出。事件帶單調遞增的 UTF-16 offset 作為 SSE `id`；重連用 `Last-Event-ID` 只補缺口。伺服器最多保留 128 KiB；游標太舊則重置並有界重播。|

終端機的租約、native 模組與修復方式見 [`docs/terminal.md`](../terminal.md)。

---

## 模型與供應商認證

|路由|方法|說明|
| --- | --- | --- |
|`/api/models`|GET|`?cwd=` 回 `{ models, modelList, defaultModel, thinkingLevelPins, modelScopeWarnings, … }`。cwd 已被刪除時會解析回主要專案（並重新對允許根目錄確認）。|
|`/api/models/default`|PUT|儲存預設模型或 reasoning level（等同 pi 的 Ctrl+S）。只接受在解析後 `enabledModels` 範圍內的模型；專案 `.pi/settings.json` 覆寫該鍵時回 `409 { reason: "project-scope", settingsPath }`。|
|`/api/models/enabled`|GET / PUT|`enabledModels` 的開關。每個切換都是**最小編輯**；不能關掉最後一個已啟用模型（`409 { reason: "last-model" }`）。|
|`/api/models/refresh`|POST|**只在使用者按下時**執行：向 pi.dev 抓取 catalog 並寫入 `models-store.json`，之後 `invalidateModelsCache()`，由面板重新載入。不排程、不順手觸發。|
|`/api/models-config`|GET / PUT|`~/.pi/agent/models.json` 的讀寫（JSONC）。|
|`/api/models-config/catalog`|GET|models.dev 的定價預設值。|
|`/api/models-config/discover`|POST|抓取某個已設定供應商的上游模型清單。|
|`/api/models-config/test`|POST|測試一個設定的模型/供應商（注意：`/api/models/test/` **不存在**）。|
|`/api/auth/providers`|GET|OAuth 與 API-key 供應商清單（能力驅動，非 id 驅動）。|
|`/api/auth/login/[provider]`|GET / POST|GET 是 OAuth / device-code / 手動碼的 SSE 串流；POST 回傳手動碼（短命 token 放在 `globalThis.__piLoginCallbacks`）。|
|`/api/auth/api-key/[provider]`|POST / DELETE|儲存 / 移除供應商 API key。|
|`/api/auth/logout/[provider]`|POST|OAuth 登出。|
|`/api/provider-usage/query`|POST|查詢供應商配額用量。|

## 設定（MCP、Skills、Plugins、Subagents、Tools）

|路由|方法|說明|
| --- | --- | --- |
|`/api/mcp`|GET|Settings › MCP 總覽。**只讀檔，不啟動任何伺服器、不解析任何值。**|
|`/api/mcp`|POST|`enable` / `disable` / `remove` / `set-enabled` / `set-exposure` / `set-in-project` / `sign-out` / `undo` / `add`。每個寫入都經 `lib/mcp-config-file.ts`。|
|`/api/mcp/test`|POST|`{ scope, name, cwd? }` 測試單一伺服器（有界連線、遮罩輸出）。|
|`/api/mcp/sign-in`|POST|開始或加入一個 OAuth 登入流程。|
|`/api/mcp/sign-in/[flowId]`|GET / POST / DELETE|輪詢流程狀態；POST 送出重導向網址或授權碼；DELETE 取消。|
|`/api/skills`|GET / PATCH|列出已載入 skills（透過 SDK 的 `DefaultResourceLoader`）；PATCH 切換 `disable-model-invocation`（只改 frontmatter 的那個鍵）。|
|`/api/skills/content`|GET|讀取單一 skill 的 Markdown 來源（唯讀、上限 1 MiB；只允許 runtime loader 回傳過的路徑）。|
|`/api/skills/search`|POST|在 skills.sh 搜尋。|
|`/api/skills/install`|POST|以 `npx skills add … --agent pi` 安裝。|
|`/api/skills/check`|POST|檢查 skill 更新。|
|`/api/skills/update`|POST|更新 skill（`runNpx`）。|
|`/api/plugins`|GET / POST|套件外掛的安裝、移除、更新、啟用、停用（走 pi 的 `SettingsManager` + `DefaultPackageManager`）。|
|`/api/plugins/check`|POST|檢查套件外掛是否有新版（`npm view`）。|
|`/api/subagents/settings`|GET / PUT|內建 subagent 開關與 `maxConcurrent`。|
|`/api/subagents/profiles`|GET / PUT / PATCH / DELETE|subagent profile 清單與編輯；`PATCH` 帶 `scope: "builtin"` 可切換內建 profile。|
|`/api/subagents/[id]`|GET / POST|讀取一個 subagent run 的狀態；POST `action: "steer"` / `abort`。|
|`/api/tools/settings`|GET / PUT|`defaultTools` 開關（PowerShell、Code mode automatic/always）與 `codemode.mode`、`codemode.inlineBudget`。是這些鍵唯一的寫入者。|
|`/api/project-trust`|GET / POST|專案信任狀態 + 該專案的 `.pi/mcp.json` 伺服器清單（只讀檔）。POST 建立信任並重建該 cwd 的 wrapper。|
|`/api/web-auth`|GET / POST / DELETE|瀏覽器密碼登入狀態 / 登入 / 登出。|

## 其他

|路由|方法|說明|
| --- | --- | --- |
|`/api/app-update`|GET|目前版本 vs 已發佈的最新版本。|
|`/api/push/config`|GET|VAPID 公鑰（私鑰不離開伺服器）。|
|`/api/push/subscribe`|POST|註冊瀏覽器推播訂閱（依 endpoint upsert）。|

---

## 型別

- `lib/api-types.ts`（904 行）是各路由回應形狀的集中定義。
- `lib/types.ts` 是 session 檔與 extension UI 的領域型別。
- `lib/pi-types.ts` 是 pi SDK 物件的**本地結構型別**（因為 SDK 沒有匯出所有型別）。
- `lib/agent-client.ts` 是前端呼叫 `/api/agent` 的型別化 fetch 輔助。
