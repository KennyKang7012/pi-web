# 00 · 本機執行手冊

> 這份是**操作手冊**，不是架構說明。要理解系統怎麼運作，看
> [README.md](README.md) 的閱讀順序；要把它跑起來、停掉、排錯，看這裡。

## 前提

| 項目 | 值 |
| --- | --- |
| 專案位置 | `C:\Users\Elan\workspace\pi-web` |
| Node.js | **>= 22.19.0**（`package.json` 的 `engines`）；本機實測 **24.15.0** 可用 |
| 相依套件 | 已 `npm install`（775 套件）。不存在時見 [第 1 節方案 C](#c-第一次node_modules-不見了) |
| 預設位址 | <http://127.0.0.1:30141>（只綁 loopback） |

確認 Node 版本：

```bat
node --version
```

---

## 1. 把服務跑起來

### A. 從這個 clone 跑（開發模式，日常用這個）

```bat
cd C:\Users\Elan\workspace\pi-web
npm run dev
```

看到這行就是成功：

```
▲ Next.js 16.3.8 (Turbopack)
- Local:         http://127.0.0.1:30141
✓ Ready in 23.6s
```

然後用瀏覽器開 👉 **<http://127.0.0.1:30141>**

### B. 只是想用，不管這個 clone（最簡單）

```bat
npx @agegr/pi-web@latest
```

官方的一行啟動方式，會自動開瀏覽器。用的是 npm 上的發佈版，跟本機的 clone 無關。

### C. 第一次／`node_modules` 不見了

```bat
cd C:\Users\Elan\workspace\pi-web
npm install
npm run dev
```

`npm install` 約需 **2 分鐘**（會跑 `postinstall` → `bin/prepare-terminal.js` 修補 node-pty）。

### 所有可用的 npm 腳本

| 指令 | 實際執行 | 用途 |
| --- | --- | --- |
| `npm run dev` | `next dev -H 127.0.0.1 -p 30141` | 開發模式（Turbopack），**日常用這個** |
| `npm run dev:lan` | `next dev -H 0.0.0.0 -p 30141` | 讓同網段其他裝置連得到（見 [第 5 節](#5-讓其他裝置連進來選用)） |
| `npm run build` | `next build --webpack` | 正式建置。⚠️ **dev 還開著時不要跑** |
| `npm start` | `next start -H 127.0.0.1 -p 30141` | 跑建置產物（需先 `build`） |
| `npm run lint` | `eslint .` | 靜態檢查 |
| `npm test` | `node --test`（全部測試檔） | 測試（見 [第 7 節](#7-測試與檢查)） |
| `npm run test:e2e` | `node e2e/run.mjs` | Playwright 端到端測試 |

---

## 2. 確認它真的起來了

伺服器啟動後，開**另一個終端機**：

```bat
curl http://127.0.0.1:30141/api/home
```

看到這樣就正常：

```json
{"home":"C:\\Users\\Elan"}
```

想更完整確認，這幾個端點都應該回 200：

```bat
curl -o NUL -w "%%{http_code}\n" http://127.0.0.1:30141/
curl -o NUL -w "%%{http_code}\n" http://127.0.0.1:30141/api/agent/running
curl -o NUL -w "%%{http_code}\n" http://127.0.0.1:30141/api/sessions
curl -o NUL -w "%%{http_code}\n" http://127.0.0.1:30141/api/models
```

> PowerShell 請把 `%%{http_code}` 寫成 `%{http_code}`，或用 `curl.exe`。

---

## 3. 停止服務

在跑 `npm run dev` 的那個終端機視窗按 **`Ctrl + C`**，需要確認時按 `Y`。

如果那個視窗已經關掉、但埠 30141 還被佔用（幂等，沒在監聽時不會報錯）：

```powershell
$conn = Get-NetTCPConnection -LocalPort 30141 -State Listen -ErrorAction SilentlyContinue
if ($conn) {
  $conn | ForEach-Object { Stop-Process -Id $_.OwningProcess -Force }
  "已終止佔用 30141 的行程"
} else {
  "30141 沒有在監聽（不需處理）"
}
```

確認已釋放（**要一起篩 `LISTENING`**）：

```bat
netstat -ano | findstr "LISTENING" | findstr ":30141"
```

（沒有輸出 = 已釋放）

> ⚠️ **不要只跑 `netstat -ano | findstr :30141`。** 剛關掉伺服器時，瀏覽器那些已經斷開的連線會留下
> `TIME_WAIT` 的殘影，看起來很像「還在跑」：
>
> ```
> TCP  127.0.0.1:50669   127.0.0.1:30141   TIME_WAIT  0
>              ↑ 本機 5xxxx      ↑ 遠端才是 30141     ↑ PID 0
> ```
>
> 那是**客戶端**的 socket（local port 是 5xxxx，remote 才是 30141，擁有者 PID 0），不佔用 30141，
> 幾分鐘內自己消失。同理，`Get-NetTCPConnection` 在找不到符合物件時是**丟例外**
> （`找不到相符的 MSFT_NetTCPConnection 物件`），不是回空值——那個「錯誤」等於「沒東西可殺」。
>
> 想知道埠到底空不空，最直接的方式是試綁定：
>
> ```bat
> node -e "const n=require('net'),s=n.createServer();s.once('error',e=>{console.log('佔用中：',e.code);process.exit(1)});s.listen(30141,'127.0.0.1',()=>{console.log('可以綁定，埠是空閒的');s.close()})"
> ```

---

## 4. 環境變數

設定方式（PowerShell，只在該視窗有效）：

```powershell
$env:PI_WEB_IDLE_TIMEOUT_MS = "0"
npm run dev
```

`cmd` 的話是 `set PI_WEB_IDLE_TIMEOUT_MS=0`。常用項目：

| 變數 | 作用 | 預設 |
| --- | --- | --- |
| `PI_WEB_IDLE_TIMEOUT_MS` | session 閒置回收毫秒數，`0` 停用 | `600000`（10 分） |
| `PI_CODING_AGENT_DIR` | pi agent 目錄（session、設定、憑證都在裡面） | `~/.pi/agent` |
| `PI_WEB_PASSWORD` | 啟用瀏覽器密碼登入 | 關閉 |
| `PI_WEB_MCP_IDLE_MS` | MCP 連線保留時間，`0` 停用 | `600000`（10 分） |
| `PI_WEB_NO_OPEN` | 不要自動開瀏覽器（`npx` 啟動時用） | 未設 |
| `PI_WEB_SHUTDOWN_DEADLINE_MS` | 給 extension 處理 `session_shutdown` 的期限 | `5000` |
| `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` | 伺服器端模型請求的代理 | 未設 |

完整清單見 [README.md 的環境變數一覽](01-runtime.md#環境變數一覽) 與上游 `README.zh-CN.md`。

> **`PI_CODING_AGENT_DIR` 的實際值**請自己確認目前是 `C:\Users\Elan\.pi\agent`（預設）。
> 改動它等於換一套 session 與憑證，兩邊不會互通。

---

## 5. 讓其他裝置連進來（選用）

```bat
npm run dev:lan
```

⚠️ 這會綁到 `0.0.0.0`，等於把一個**能執行高權限動作的 agent** 曝露在網路上。務必同時設密碼：

```powershell
$env:PI_WEB_PASSWORD = "一組很長的隨機密碼"
npm run dev:lan
```

- 純 HTTP **不加密**，不要在不受信任的網路上用。要外網請走 HTTPS 反向代理或 VPN。
- 反向代理送外部主機名時，把該名稱加進 `PI_WEB_ALLOWED_HOSTS`（逗號分隔）。

---

## 6. 常見狀況

| 症狀 | 原因與處理 |
| --- | --- |
| **第一次開頁面很慢（約 50 秒）** | 正常。`next dev` 要即時編譯整個應用（Next 49s + 應用程式碼 1.8s）。之後同一頁重載約 **0.1 秒** |
| `Error: Port 30141 is already in use` | 上一個 dev server 還在跑。用 [第 3 節](#3-停止服務) 的指令清掉，或直接沿用那個視窗 |
| `Get-NetTCPConnection … 找不到相符的 MSFT_NetTCPConnection 物件` | **不是錯誤**：代表 30141 沒有在監聽，也就沒有東西可殺（該 cmdlet 找不到時是丟例外而非回空值）。詳見 [第 3 節](#3-停止服務) |
| `netstat` 還看得到 `:30141` | 若狀態是 `TIME_WAIT` 且 local port 是 5xxxx、PID 0，那是瀏覽器端的殘影，不是伺服器；看 `LISTENING` 才是判準 |
| 頁面出現 `Module ... factory is not available` 覆蓋層 | 通常是**那個分頁**的 HMR 圖過期，不是原始碼壞掉。先按 `Ctrl + Shift + R` 重新載入，再比對伺服器 log 與直接打 API |
| `⚠️ next build` 把 dev 弄壞 | **不要在 `npm run dev` 還開著時跑 `npm run build`**，它會改寫 `.next/`。這是專案 `AGENTS.md` 明確禁止的 |
| `next dev` 一直往 `AGENTS.md` 附加區塊 | Next 16 的工具輸出（`BEGIN:nextjs-agent-rules`）。`git status` 會看到它被修改，**不要**把它混進無關的 commit；`git checkout -- AGENTS.md` 可還原，下次跑 dev 會再出現 |
| 模型清單是空的 | 還沒設定 provider。開 **Settings › Models** 登入或填 API key |

---

## 7. 測試與檢查

```bat
npm run lint
node_modules\.bin\tsc --noEmit
npm test
```

### ⚠️ Windows 上 `npm test` 有一個已知失敗

`components/ChatInput.test.mjs` 的 `renders image warnings for known text-only defaults…`
在 Windows 上會失敗，**在 Linux 上通過**（上游 CI 只跑 ubuntu）。已確認這是**測試載具**
的問題，不影響應用本身 —— 詳見
[issue #1211](https://github.com/agegr/pi-web/issues/1211)。

三個選擇：

1. **不管它** —— 其餘 42 個在同檔內都通過，應用執行也完全正常。
2. **在 WSL 裡跑**（要全綠就這個）：
   ```bash
   wsl -e bash -lc 'cd /mnt/c/Users/Elan/workspace/pi-web && \
     ~/.nvm/versions/node/v24.15.0/bin/node --experimental-strip-types --test components/ChatInput.test.mjs'
   ```
3. **降併發、分區跑**（避免 Windows 上一次跑 307 個測試檔互相搶資源）：
   ```bat
   node --experimental-strip-types --test --test-concurrency=2 "lib/*.test.mjs"
   ```

> **不要無參數直接跑 `npm test` 然後等它結束。** 它會以 CPU 數（12）併發啟動 307 個
> 測試檔，前 50 秒沒有任何輸出，而且在中途會因為資源耗盡而停止推進；中止後還會留下
> 孤兒化的 fixture 行程與暫存檔。

---

## 8. WSL（選用）

只有在「要跟上游 CI 一致的測試環境」或「要完整的 Linux 開發體驗」時才需要。

```bash
# 1) 專案要放在 Linux 檔案系統，不要在 /mnt/c 上跑 npm install
mkdir -p ~/workspace && cd ~/workspace
git clone https://github.com/KennyKang7012/pi-web.git
cd pi-web && npm ci
npm run dev
```

三個務必注意：

1. **絕對不要在 `/mnt/c/.../pi-web` 裡跑 `npm install`／`npm ci`。** 那會把 Linux 版二進位
   （`@next/swc-linux-*`、node-pty 的 linux prebuild）寫進 Windows 的 `node_modules`，
   讓 Windows 的 `npm run dev` 壞掉。
2. **`~` 會變成 `/home/elan`**，pi 資料就對不上了。要跟 Windows 共用同一份 agent 目錄：
   ```bash
   PI_CODING_AGENT_DIR=/mnt/c/Users/Elan/.pi/agent npm run dev
   ```
   共用很方便，但 `/mnt/c` 上的檔案鎖與 atomic rename 走 9p 較慢，**兩邊不要同時在跑**。
3. WSL2 的 `127.0.0.1:30141` 通常可直接從 Windows 瀏覽器開啟（localhost 轉發）。

---

## 9. 這台機器的實測紀錄（2026-10）

| 項目 | 結果 |
| --- | --- |
| `npm install` | ✅ 775 套件、2 分鐘、`postinstall` 正常 |
| `npm run dev`（Windows, Node 24.15.0） | ✅ `Ready in 15–24s` |
| `GET /` 首次／暖機 | ✅ 200（首次編譯約 50s，之後 0.10s） |
| 12 個 API 端點 | ✅ 全部 200（`/api/home`、`/api/sessions`、`/api/models`、`/api/mcp`、`/api/tools/settings`、`/api/cwd/browse` …） |
| `tsc --noEmit` | ✅ 零錯誤（約 1 分 13 秒） |
| `npm run lint` | ✅ 零錯誤（約 4 分鐘） |
| `npm test` | ⚠️ Linux 全綠；Windows 有 1 個已知失敗（見 [第 7 節](#7-測試與檢查)） |
| 草稿還原（切換專案後返回） | ✅ 在 Windows 真實瀏覽器上正常 |

---

## 10. 相關文件

| 文件 | 內容 |
| --- | --- |
| [README.md](README.md) | 架構文件索引與閱讀順序 |
| [01-runtime.md](01-runtime.md) | 啟動流程、AgentSession 生命週期、SSE、session 檔 |
| [AGENTS.md](../../AGENTS.md) | 開發注意事項與檔案地圖（上游維護者寫的） |
| [docs/terminal.md](../terminal.md) | 終端機面板與 node-pty 修復 |
| [docs/worktrees.md](../worktrees.md) | git worktree 的使用與行為 |
| [docs/release.md](../release.md) | 發佈流程（本機開發不需要） |
