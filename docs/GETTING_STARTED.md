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

---

### 2. Local Deployment 模式（完全離線）

此模式完全在本地運行，資料不會上傳雲端。

#### 步驟 1：啟動本地後端

```bash
cd server

# 複製環境變數範本
cp .env.example .env

# 使用 Docker 啟動後端服務
docker compose up -d
```

服務啟動後：
- API 服務：http://localhost:3001
- API 文件：http://localhost:3001/docs
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

**原因**：Python 後端尚未啟動完成。

**解決方案**：
1. 確認 `backend/` 目錄下的 Python 環境已正確安裝
2. 檢查是否有其他程序佔用 5001 埠
3. 查看 Electron 開發者工具的 Console 輸出

### Q: Local 模式下無法註冊

**原因**：Docker 服務未啟動或資料庫連線失敗。

**解決方案**：
```bash
# 檢查 Docker 容器狀態
docker ps

# 查看容器日誌
docker logs eigent_api
docker logs eigent_postgres
```

### Q: OAuth 登入在 Local 模式下不可用

**這是預期行為**。OAuth (Google/GitHub) 需要與 Eigent Cloud 服務整合，本地模式僅支援 Email + 密碼登入。

---

## 下一步

- [專案架構總覽](./ARCHITECTURE.md)
- [前端架構說明](./FRONTEND.md)
- [後端 API 參考](./BACKEND_API.md)
- [開發指令速查](./COMMANDS.md)
