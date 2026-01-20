# 常見問題 FAQ

> 本文件收集 Eigent 專案開發和使用過程中的常見問題與解答。

---

## 架構與部署

### Q: 嵌入式後端會將步驟同步到 Cloud Service，如果通訊失敗會有什麼反應？

**答**：通訊失敗**不會影響任務執行**。

**詳細說明**：

1. **非阻塞設計**：
   - 同步使用 `asyncio.create_task` 非同步執行
   - 不會阻塞主任務執行流程
   - 前端 SSE 串流不受影響

2. **錯誤處理**：
   - 通訊失敗時只會記錄錯誤日誌：`Failed to sync step to {url}: {error}`
   - 不會中斷任務執行
   - 不會有重試機制（失敗即放棄，不影響主流程）

3. **影響範圍**：
   - ✅ 任務執行：正常完成
   - ✅ 前端顯示：即時狀態更新正常
   - ❌ 步驟記錄：失敗的步驟不會被記錄到雲端資料庫

**錯誤日誌位置**：嵌入式後端日誌（可透過 Electron DevTools Console 查看）

---

### Q: Local Deployment 模式是否也會做同步行為？

**答**：**預設不會**，但可以手動設定。

**詳細說明**：

1. **預設行為**：
   - Local Deployment 模式下，`SERVER_URL` 通常未設定
   - 嵌入式後端檢查到 `SERVER_URL` 為 `None` 時，會跳過同步步驟
   - 任務步驟不會被同步到任何地方

2. **手動啟用同步**：
   - 可在 `~/.eigent/.env` 設定 `SERVER_URL=http://localhost:3001`
   - 這樣嵌入式後端會將步驟同步到 Local Server
   - 步驟會被記錄到本地 PostgreSQL 資料庫

3. **判斷邏輯**：
   ```python
   server_url = env("SERVER_URL")
   if not server_url:
       # 跳過同步，直接 yield value
       yield value
       continue
   ```

**建議**：
- Local Deployment 模式通常不需要同步步驟（資料已透過 Local Server 的其他 API 管理）
- 如果需要完整的步驟記錄，可以設定 `SERVER_URL` 指向 Local Server

---

## 後端與服務

### Q: 嵌入式後端和 Local Server 的區別是什麼？

**答**：兩者職責不同，在 Local Deployment 模式下會同時運行。

| 項目 | 嵌入式後端（backend/） | Local Server（server/） |
|-----|----------------------|----------------------|
| **啟動方式** | Electron 自動啟動 | 手動啟動（Docker 或 Source Code） |
| **主要功能** | 任務執行引擎（多智能體） | 資料管理（帳號、設定、歷史） |
| **API 端點** | `/chat`、`/task/*`、`/health` | `/api/register`、`/api/login`、`/api/providers` 等 |
| **資料儲存** | 無（僅執行任務） | PostgreSQL（本地） |
| **運行模式** | 兩種模式都會運行 | 僅 Local Deployment 模式需要 |

---

### Q: SERVER_URL 在哪裡設定？如何覆寫？

**答**：主要設定位置和覆寫方式如下。

**設定位置**：
1. **預設值**：`electron/main/init.ts` 第 166 行，硬編碼為 `"https://dev.eigent.ai/api"`
2. **環境變數注入**：Electron 在啟動嵌入式後端時透過 `spawn()` 的 `env` 參數注入
3. **可選覆寫**：可在 `~/.eigent/.env` 設定 `SERVER_URL` 來覆寫預設值

**覆寫方式**：
```bash
# 在 ~/.eigent/.env 中設定
SERVER_URL=http://localhost:3001  # Local Deployment 模式
# 或
SERVER_URL=https://dev.eigent.ai/api  # Cloud-Connected 模式（預設值）
```

**注意**：
- `backend/` 目錄下沒有 `.env` 文件
- 環境變數完全由 Electron 管理
- 覆寫後需要重啟 Electron 應用才會生效

---

## 開發與除錯

### Q: 如何查看嵌入式後端的日誌？

**答**：可透過以下方式查看。

1. **Electron DevTools Console**（即時查看）：
   - 開發模式下自動開啟
   - 或使用快捷鍵：`Ctrl + Shift + I`（Windows/Linux）或 `Cmd + Option + I`（macOS）
   - 日誌會以 `BACKEND:` 前綴顯示

2. **日誌級別**：
   - `ERROR`：錯誤訊息（包含同步失敗）
   - `INFO`：一般資訊
   - `DEBUG`：除錯資訊

3. **同步失敗日誌範例**：
   ```
   BACKEND: ERROR: Failed to sync step to https://dev.eigent.ai/api/chat/steps: ConnectionError: ...
   ```

**注意**：嵌入式後端的日誌**不會寫入檔案**，只輸出到 stdout/stderr，由 Electron 捕獲並顯示在 Console。如果需要保存日誌，請使用 Electron 的日誌匯出功能。

---

### Q: 有記錄 log 到檔案中嗎？如果沒有，要如何查看運行紀錄？

**答**：不同組件的日誌記錄方式不同。

#### 1. Electron 主程序日誌（有檔案記錄）

**日誌檔案位置**：
- **Windows**：`%USERPROFILE%\AppData\Roaming\eigent\logs\main.log`
  - 完整路徑範例：`C:\Users\{username}\AppData\Roaming\eigent\logs\main.log`
- **macOS**：`~/Library/Logs/eigent/main.log`
- **Linux**：`~/.config/eigent/logs/main.log`

**查看方式**：
1. **直接開啟檔案**：使用文字編輯器開啟上述路徑的 `main.log` 檔案
2. **透過應用匯出**：在應用中點擊「Bug Report」按鈕，會自動匯出日誌檔案
3. **程式化取得路徑**：
   ```typescript
   import log from 'electron-log';
   const logPath = log.transports.file.getFile().path;
   ```

**日誌內容**：
- Electron 主程序的日誌（視窗管理、IPC、後端啟動等）
- 嵌入式後端的 stdout/stderr 輸出（以 `BACKEND:` 前綴顯示）

#### 2. 嵌入式後端日誌（無檔案記錄）

**記錄方式**：
- 使用 `traceroot` 或 Python `logging`（fallback）
- 日誌輸出到 **stdout/stderr**，由 Electron 捕獲
- **不會寫入檔案**（除非 traceroot 有特別配置）

**查看方式**：
- **Electron DevTools Console**：即時查看，日誌以 `BACKEND:` 前綴顯示
- **Electron 日誌檔案**：會包含嵌入式後端的輸出（因為 Electron 捕獲了 stdout/stderr）

#### 3. 任務執行日誌（CAMEL 日誌）

**日誌檔案位置**：
```
~/.eigent/{email}/project_{project_id}/task_{task_id}/camel_logs/
```

**查看方式**：
- 直接瀏覽上述目錄
- 透過應用介面查看任務相關檔案

#### 4. Browser Toolkit 日誌（可選）

**日誌檔案位置**（如果啟用 `browser_log_to_file=True`）：
```
browser_log/hybrid_browser_toolkit_{timestamp}_{session_id}.log
```

**查看方式**：
- 直接開啟檔案（位於工作目錄下的 `browser_log/` 資料夾）

#### 5. Local Server 日誌（Docker 模式）

**日誌位置**：
- **容器內**：`/app/runtime/log/app.log`（見 `server/README_EN.md`）
- **查看方式**：
  ```bash
  docker logs -f eigent_api
  ```

---

### 快速查看日誌指令

**Windows PowerShell**：
```powershell
# 查看 Electron 日誌
Get-Content "$env:APPDATA\eigent\logs\main.log" -Tail 100

# 查看最新日誌（即時監控）
Get-Content "$env:APPDATA\eigent\logs\main.log" -Wait -Tail 50
```

**macOS/Linux**：
```bash
# 查看 Electron 日誌
tail -f ~/Library/Logs/eigent/main.log  # macOS
tail -f ~/.config/eigent/logs/main.log  # Linux

# 查看最新 100 行
tail -n 100 ~/Library/Logs/eigent/main.log  # macOS
```

**查看 Docker 容器日誌**（Local Server）：
```bash
# 查看即時日誌
docker logs -f eigent_api

# 查看最新 100 行
docker logs --tail 100 eigent_api
```

---

## 相關文件

- [專案架構總覽](./ARCHITECTURE.md)
- [開發環境設置](./GETTING_STARTED.md)
- [開發指令速查](./COMMANDS.md)
- [嵌入式後端 API](./BACKEND_API.md)
- [Local Server 和 Cloud Service API](./SERVER_API.md)
