# 開發指令速查表

> 本文件列出 Eigent 專案常用的開發指令。

## 前端指令 (npm)

### 開發與建置

| 指令 | 說明 |
|-----|------|
| `npm install` | 安裝依賴套件 |
| `npm run dev` | 啟動開發伺服器 (含 Electron) |
| `npm run build` | 建置生產版本並打包 |
| `npm run build:win` | 建置 Windows 安裝檔 |
| `npm run build:mac` | 建置 macOS 安裝檔 |
| `npm run preview` | 預覽建置結果 |

### 測試

| 指令 | 說明 |
|-----|------|
| `npm run test` | 執行單元測試 |
| `npm run test:watch` | 監聽模式執行測試 |
| `npm run test:e2e` | 執行端對端測試 |
| `npm run test:coverage` | 產生測試覆蓋率報告 |

### 程式碼品質

| 指令 | 說明 |
|-----|------|
| `npm run type-check` | TypeScript 型別檢查 |

---

## 後端指令 (Python/uv)

### 嵌入式後端（backend/）

**注意**：嵌入式後端通常由 Electron 自動啟動，不需要手動執行。以下指令僅供開發調試使用。

```bash
# 進入 backend 目錄
cd backend

# 啟動嵌入式後端服務（開發模式）
uv run uvicorn main:api --port 5001 --reload

# 設定環境變數（可選）
# SERVER_URL 用於同步任務步驟到雲端（Cloud-Connected 模式）
export SERVER_URL=https://dev.eigent.ai/api
```

**啟動說明**：
- Electron 會自動執行 `uv sync --no-dev` 安裝依賴
- 使用 uv 管理的虛擬環境（位於 `{userData}/venv/{version}/`）
- 工作目錄為 `backend/` 目錄
- 環境變數由 Electron 注入（包括 `SERVER_URL`、`PYTHONIOENCODING` 等）

### Local Server（server/）

#### 方式 A：使用 .env 文件（推薦）

```bash
# 進入 server 目錄
cd server

# 複製環境變數範本
cp .env.example .env

# 編輯 .env 設定資料庫連線
# database_url=postgresql://postgres:123456@localhost:5432/eigent

# 啟動服務（Windows PowerShell）
.\start_server.ps1

# 或手動啟動
uv run uvicorn main:api --reload --port 3001 --host 0.0.0.0
```

#### 方式 B：使用環境變數

**Linux/macOS**：
```bash
export database_url=postgresql://postgres:123456@localhost:5432/eigent
uv run uvicorn main:api --reload --port 3001 --host 0.0.0.0
```

**Windows PowerShell**：
```powershell
$env:database_url="postgresql://postgres:123456@localhost:5432/eigent"
uv run uvicorn main:api --reload --port 3001 --host 0.0.0.0
```

**Windows CMD**：
```cmd
set database_url=postgresql://postgres:123456@localhost:5432/eigent
uv run uvicorn main:api --reload --port 3001 --host 0.0.0.0
```

### 資料庫遷移

```bash
# 執行遷移
uv run alembic upgrade head

# 產生新遷移
uv run alembic revision --autogenerate -m "描述"

# 回滾遷移
uv run alembic downgrade -1
```

### 國際化 (i18n)

```bash
# 提取翻譯字串
uv run pybabel extract -F babel.cfg -o messages.pot .

# 初始化語言包
uv run pybabel init -i messages.pot -d lang -l zh_CN

# 編譯語言包
uv run pybabel compile -d lang -l zh_CN

# 更新語言包
uv run pybabel update -i messages.pot -d lang
```

---

## Docker 指令 (本地部署)

```bash
# 進入 server 目錄
cd server

# 啟動所有服務 (後台)
docker compose up -d

# 查看容器狀態
docker ps

# 查看 API 日誌
docker logs -f eigent_api

# 查看資料庫日誌
docker logs -f eigent_postgres

# 停止服務
docker compose stop

# 重啟服務
docker compose restart

# 停止並移除容器
docker compose down

# 停止並移除容器 + 資料卷 (慎用！會刪除資料)
docker compose down -v
```

---

## Git 工作流程

### 常用指令

```bash
# 查看狀態
git status

# 查看變更
git diff

# 提交變更
git add .
git commit -m "feat: 新增功能描述"

# 推送到遠端
git push origin feature-branch
```

### Commit 訊息規範

| 類型 | 說明 |
|-----|------|
| `feat:` | 新功能 |
| `fix:` | 錯誤修復 |
| `docs:` | 文件更新 |
| `style:` | 程式碼格式調整 |
| `refactor:` | 重構 |
| `test:` | 測試相關 |
| `chore:` | 建置/工具相關 |

---

## 環境變數

### 前端 (.env.development)

| 變數名稱 | 說明 | 範例值 |
|---------|------|--------|
| `VITE_BASE_URL` | API 基礎路徑 | `/api` |
| `VITE_PROXY_URL` | 代理目標 URL | `https://dev.eigent.ai` |
| `VITE_USE_LOCAL_PROXY` | 是否使用本地代理 | `true` / `false` |

### 後端環境變數

#### 嵌入式後端（backend/）

**注意**：嵌入式後端的環境變數由 Electron 自動注入，通常不需要手動設定。

| 變數名稱 | 說明 | 預設值 | 可選覆寫位置 |
|---------|------|--------|------------|
| `SERVER_URL` | 雲端服務 URL（用於同步任務步驟） | `https://dev.eigent.ai/api` | `~/.eigent/.env` |
| `PYTHONIOENCODING` | Python I/O 編碼 | `utf-8` | - |
| `PYTHONUNBUFFERED` | Python 輸出緩衝 | `1` | - |

#### Local Server（server/.env）

| 變數名稱 | 說明 | 範例值 |
|---------|------|--------|
| `database_url` | PostgreSQL 連線字串 | `postgresql://user:pass@host:5432/db` |
| `url_prefix` | API 路由前綴 | `/api` |

**設定方式**：
1. **.env 文件**（推薦）：在 `server/` 目錄下建立 `.env` 文件
2. **環境變數**：在啟動前設定環境變數（見上方「方式 B」）

---

## 除錯技巧

### 開啟 Electron DevTools

開發模式下自動開啟，或使用快捷鍵：
- Windows/Linux: `Ctrl + Shift + I`
- macOS: `Cmd + Option + I`

### 檢查後端健康狀態

```bash
# 檢查嵌入式後端（backend/）
# 注意：埠號可能動態分配，可透過 Electron IPC 取得
curl http://localhost:5001/health

# 檢查 Local Server（server/）
curl http://localhost:3001/health
```

### 查看 Swagger API 文件

開啟瀏覽器訪問：

- **嵌入式後端**（`backend/`）：
  - URL：`http://localhost:<port>/docs`
  - 注意：埠號由 Electron 動態分配（預設從 5001 起），可透過 Electron IPC 取得實際埠號
  - 主要端點：`/chat`（SSE 任務執行）、`/health`

- **Local Server**（`server/`）：
  - URL：`http://localhost:3001/docs`
  - 主要端點：`/api/register`、`/api/login`、`/api/providers`、`/api/configs`、`/api/mcps`、`/api/chat/history`、`/api/chat/steps`、`/api/proxy/google` 等

---

## 日誌查看

### Electron 主程序日誌

**日誌檔案位置**：
- **Windows**：`%APPDATA%\eigent\logs\main.log`
- **macOS**：`~/Library/Logs/eigent/main.log`
- **Linux**：`~/.config/eigent/logs/main.log`

**查看指令**：

**Windows PowerShell**：
```powershell
# 查看最新 100 行
Get-Content "$env:APPDATA\eigent\logs\main.log" -Tail 100

# 即時監控（類似 tail -f）
Get-Content "$env:APPDATA\eigent\logs\main.log" -Wait -Tail 50
```

**macOS/Linux**：
```bash
# 查看最新 100 行
tail -n 100 ~/Library/Logs/eigent/main.log  # macOS
tail -n 100 ~/.config/eigent/logs/main.log  # Linux

# 即時監控
tail -f ~/Library/Logs/eigent/main.log  # macOS
tail -f ~/.config/eigent/logs/main.log  # Linux
```

**透過應用匯出**：
- 在應用中點擊「Bug Report」按鈕，會自動匯出日誌檔案

### 嵌入式後端日誌

**記錄方式**：
- 日誌輸出到 stdout/stderr，由 Electron 捕獲
- 會出現在 Electron 日誌檔案中（以 `BACKEND:` 前綴）
- **不會單獨寫入檔案**

**查看方式**：
- **Electron DevTools Console**：即時查看（`Ctrl + Shift + I` 或 `Cmd + Option + I`）
- **Electron 日誌檔案**：包含嵌入式後端的輸出

### Local Server 日誌（Docker）

**查看容器日誌**：
```bash
# 即時查看
docker logs -f eigent_api

# 查看最新 100 行
docker logs --tail 100 eigent_api

# 查看容器內日誌檔案（如果存在）
docker exec eigent_api cat /app/runtime/log/app.log
```

### 任務執行日誌（CAMEL）

**日誌位置**：
```
~/.eigent/{email}/project_{project_id}/task_{task_id}/camel_logs/
```

**查看方式**：
- 直接瀏覽上述目錄
- 透過應用介面查看任務相關檔案

---

## 相關文件

- [專案架構總覽](./ARCHITECTURE.md)
- [開發環境設置](./GETTING_STARTED.md)
- [嵌入式後端 API](./BACKEND_API.md)
- [Local Server 和 Cloud Service API](./SERVER_API.md)
