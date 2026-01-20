# 開發環境設置指南

> 本文件說明如何設置 Eigent 的本地開發環境。

## 系統需求

| 項目 | 版本要求 |
|-----|---------|
| Node.js | 18.x - 22.x |
| Python | 3.12.x |
| Docker Desktop | 最新版 (僅本地部署需要) |
| 作業系統 | Windows 10+, macOS 10.15+, Linux |

---

## 快速開始

### 1. Cloud-Connected 模式（推薦新手）

此模式連接 Eigent 雲端服務，無需設定資料庫。

```bash
# 複製專案
git clone https://github.com/eigent-ai/eigent.git
cd eigent

# 安裝依賴
npm install

# 啟動開發伺服器
npm run dev
```

**帳號取得**：前往 [eigent.ai/signup](https://www.eigent.ai/signup) 註冊。

**後端說明**：
- **嵌入式後端**（`backend/`）：由 Electron 自動啟動，負責多智能體任務執行
- **Cloud Service**：提供帳號認證、設定管理、資料儲存等功能
- 無需手動啟動任何後端服務

---

### 2. Local Deployment 模式（完全離線）

此模式完全在本地運行，資料不會上傳雲端。

**後端說明**：
- **嵌入式後端**（`backend/`）：由 Electron 自動啟動，負責多智能體任務執行
- **Local Server**（`server/`）：需要手動啟動，提供帳號管理、設定管理、資料持久化等功能

#### 步驟 1：啟動 Local Server

**方式 A：Docker 啟動（推薦）**

```bash
cd server

# 複製環境變數範本
cp .env.example .env

# 使用 Docker 啟動服務
docker compose up -d
```

**方式 B：Source Code 啟動（開發調試）**

```bash
cd server

# 複製環境變數範本
cp .env.example .env

# 編輯 .env 設定資料庫連線
# database_url=postgresql://postgres:123456@localhost:5432/eigent

# 執行資料庫遷移
uv run alembic upgrade head

# 啟動服務（Windows PowerShell）
.\start_server.ps1

# 或手動啟動
uv run uvicorn main:api --reload --port 3001 --host 0.0.0.0
```

服務啟動後：
- Local Server API：<http://localhost:3001>
- API 文件：<http://localhost:3001/docs>
- PostgreSQL：localhost:5432

#### 步驟 2：修改前端環境變數

編輯 `.env.development`：

```bash
VITE_BASE_URL=/api
VITE_PROXY_URL=http://localhost:3001
VITE_USE_LOCAL_PROXY=true
```

#### 步驟 3：啟動前端

```bash
npm install
npm run dev
```

#### 步驟 4：註冊帳號

1. 應用啟動後，點擊「Sign Up」
2. 填寫 Email 和密碼（至少 8 字元）
3. 完成註冊後登入

**註冊 API**：`POST /api/register`，目標為 Local Server（`http://localhost:3001`）

**注意**：Local Deployment 模式下，嵌入式後端（`backend/`）會由 Electron 自動啟動，無需手動操作。

---

## 登入流程說明

### 支援的登入方式

| 模式 | Email + 密碼 | Google OAuth | GitHub OAuth |
|-----|:-----------:|:------------:|:------------:|
| Cloud-Connected | ✅ | ✅ | ✅ |
| Local Deployment | ✅ | ❌ | ❌ |

### 程式碼位置

| 功能 | 檔案路徑 |
|-----|---------|
| 登入頁面 | `src/pages/Login.tsx` |
| 註冊頁面 | `src/pages/SignUp.tsx` |
| HTTP 請求 | `src/api/http.ts` |
| 認證狀態 | `src/store/authStore.ts` |
| 路由保護 | `src/routers/index.tsx` |
| 本地登入 API | `server/app/controller/user/login_controller.py` |

### 登入 API 端點

```
POST /api/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "your_password"
}

Response:
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "email": "user@example.com"
}
```

---

## 開發模式切換

### 切換到 Cloud 模式

```bash
# .env.development
VITE_USE_LOCAL_PROXY=false
VITE_PROXY_URL=https://dev.eigent.ai
```

### 切換到 Local 模式

```bash
# .env.development
VITE_USE_LOCAL_PROXY=true
VITE_PROXY_URL=http://localhost:3001
```

> **注意**：切換模式後需要重新登入，因為 Token 不互通。

---

## 常見問題

### Q: 啟動時出現「Backend is not ready」錯誤

**原因**：嵌入式後端（`backend/`）尚未啟動完成。

**解決方案**：
1. 確認 Electron 已正確啟動（嵌入式後端由 Electron 自動管理）
2. 檢查是否有其他程序佔用 5001 埠（或動態分配的埠）
3. 查看 Electron 開發者工具的 Console 輸出
4. 確認 `uv` 已正確安裝，Electron 會自動執行 `uv sync` 安裝依賴

**注意**：嵌入式後端不需要手動啟動，由 Electron 的 `startBackend()` 函數自動管理。

### Q: Local 模式下無法註冊

**原因**：Local Server 未啟動或資料庫連線失敗。

**解決方案**：
```bash
# 檢查 Docker 容器狀態（Docker 啟動方式）
docker ps

# 查看容器日誌
docker logs eigent_api
docker logs eigent_postgres

# 檢查 Local Server 是否正常運行
curl http://localhost:3001/health

# 確認資料庫連線設定（Source Code 啟動方式）
# 檢查 server/.env 中的 database_url 設定
```

### Q: OAuth 登入在 Local 模式下不可用

**這是預期行為**。OAuth (Google/GitHub) 需要與 Eigent Cloud 服務整合，Local Deployment 模式僅支援 Email + 密碼登入。

### Q: SERVER_URL 在哪裡設定？

**嵌入式後端的 SERVER_URL**：
- **預設值**：`https://dev.eigent.ai/api`（在 `electron/main/init.ts` 中硬編碼）
- **可選覆寫**：可在 `~/.eigent/.env` 設定 `SERVER_URL` 來覆寫預設值
- **作用**：在 Cloud-Connected 模式下，嵌入式後端會將任務步驟同步到 Cloud Service
- **Local Deployment 模式**：通常不需要設定，或可設定為 Local Server 的 URL（`http://localhost:3001`）

### Q: 任務步驟同步失敗會影響任務執行嗎？

**不會**。任務步驟同步是**非阻塞**的：

- 同步使用非同步任務（`asyncio.create_task`）執行，不會阻塞主流程
- 如果同步失敗（網路錯誤、服務不可用等），只會記錄錯誤日誌
- 任務執行不受影響，會正常完成
- 前端 SSE 串流不受影響，使用者仍可看到即時狀態更新
- 唯一的影響是：步驟不會被記錄到雲端資料庫（Cloud-Connected 模式）或本地資料庫（Local Deployment 模式，如果設定了 `SERVER_URL`）

**錯誤日誌位置**：嵌入式後端日誌（可透過 Electron DevTools Console 或 Electron 日誌檔案查看）。

### Q: 如何查看運行紀錄？日誌檔案在哪裡？

**答**：不同組件的日誌位置不同。

1. **Electron 主程序日誌**（有檔案記錄）：
   - **Windows**：`%APPDATA%\eigent\logs\main.log`
   - **macOS**：`~/Library/Logs/eigent/main.log`
   - **Linux**：`~/.config/eigent/logs/main.log`
   - **查看方式**：直接開啟檔案，或透過應用「Bug Report」功能匯出

2. **嵌入式後端日誌**（無檔案記錄）：
   - 輸出到 stdout/stderr，由 Electron 捕獲
   - 會出現在 Electron 日誌檔案中（以 `BACKEND:` 前綴）
   - **即時查看**：Electron DevTools Console（`Ctrl + Shift + I`）

3. **任務執行日誌**（CAMEL）：
   - 位置：`~/.eigent/{email}/project_{project_id}/task_{task_id}/camel_logs/`

4. **Local Server 日誌**（Docker）：
   - 查看：`docker logs -f eigent_api`
   - 容器內檔案：`/app/runtime/log/app.log`

詳細說明請參考 [常見問題 FAQ](./FAQ.md)。

---

## 下一步

- [專案架構總覽](./ARCHITECTURE.md)
- [前端架構說明](./FRONTEND.md)
- [嵌入式後端 API](./BACKEND_API.md)
- [Local Server 和 Cloud Service API](./SERVER_API.md)
- [開發指令速查](./COMMANDS.md)
- [常見問題 FAQ](./FAQ.md)
