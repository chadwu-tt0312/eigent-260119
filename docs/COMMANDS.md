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

### 本地開發

```bash
# 進入 backend 目錄
cd backend

# 啟動後端服務 (使用 uv)
uv run uvicorn main:api --port 5001 --reload

# 或指定資料庫連線
export database_url=postgresql://postgres:123456@localhost:5432/eigent
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

### 後端 (server/.env)

| 變數名稱 | 說明 | 範例值 |
|---------|------|--------|
| `database_url` | PostgreSQL 連線字串 | `postgresql://user:pass@host:5432/db` |
| `url_prefix` | API 路由前綴 | `/api` |

---

## 除錯技巧

### 開啟 Electron DevTools

開發模式下自動開啟，或使用快捷鍵：
- Windows/Linux: `Ctrl + Shift + I`
- macOS: `Cmd + Option + I`

### 檢查後端健康狀態

```bash
curl http://localhost:5001/health
```

### 查看 Swagger API 文件

開啟瀏覽器訪問：
- 嵌入式後端：`http://localhost:5001/docs`
- 本地部署後端：`http://localhost:3001/docs`

---

## 相關文件

- [專案架構總覽](./ARCHITECTURE.md)
- [開發環境設置](./GETTING_STARTED.md)
- [後端 API 參考](./BACKEND_API.md)
