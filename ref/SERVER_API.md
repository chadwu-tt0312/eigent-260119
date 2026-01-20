# Local Server 和 Cloud Service API 參考

> 本文件說明 Local Server（`server/`）和 Cloud Service（`https://dev.eigent.ai`）的 API 端點、請求格式與回應結構。

## API 概述

**Local Server** 和 **Cloud Service** 提供相同的 API 端點，但資料儲存位置不同：

| 項目 | Local Server | Cloud Service |
|-----|-------------|---------------|
| **位置** | `server/` | `https://dev.eigent.ai` |
| **啟動方式** | Docker 或 Source Code | Eigent 雲端服務 |
| **認證方式** | JWT Bearer Token | JWT Bearer Token |
| **資料儲存** | 本地 PostgreSQL | 雲端資料庫 |
| **適用模式** | Local Deployment | Cloud-Connected |

**注意**：部分端點標註為「僅 Cloud-Connected 模式可用」，表示這些功能在 Local Deployment 模式下可能不可用或行為不同。

**相關文件**：
- [嵌入式後端 API](./BACKEND_API.md) - 任務執行相關 API
- [專案架構總覽](./ARCHITECTURE.md)

---

## 認證 API

### POST /api/register

註冊新帳號（僅 Local Deployment 模式）。

**請求**：
```json
{
  "email": "user@example.com",
  "password": "your_password",
  "invite_code": ""
}
```

**回應**：
```json
{
  "status": "success"
}
```

**錯誤碼**：
- `code: 10` - Email 已被註冊

---

### POST /api/login

使用 Email 和密碼登入。

**請求**：
```json
{
  "email": "user@example.com",
  "password": "your_password"
}
```

**回應**：
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "email": "user@example.com"
}
```

**錯誤碼**：
- `code: 10` - 帳號或密碼錯誤

---

### POST /api/login-by_stack

使用 Stack Auth OAuth Token 登入（僅 Cloud-Connected 模式）。

**請求**：
```
POST /api/login-by_stack?token=xxx&type=signup
```

**回應**：
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "email": "user@example.com"
}
```

---

## 用戶管理 API

### GET /api/user

取得當前用戶資訊。

**認證**：需要 JWT Bearer Token

**回應**：
```json
{
  "id": 1,
  "email": "user@example.com",
  "username": "user",
  "nickname": "User",
  "avatar": "https://...",
  "created_at": "2024-01-01T00:00:00Z"
}
```

### PUT /api/user

更新用戶資訊。

### GET /api/user/profile

取得用戶個人資料。

### PUT /api/user/profile

更新用戶個人資料。

### GET /api/user/privacy

取得隱私設定。

### PUT /api/user/privacy

更新隱私設定。

### GET /api/user/current_credits

取得當前額度。

### GET /api/user/stat

取得用戶統計資訊。

---

## 模型供應商 API

### GET /api/providers

取得已設定的模型供應商清單。

**參數**：
- `keyword` - 搜尋關鍵字（可選）
- `prefer=true` - 只返回偏好的供應商（可選）

**認證**：需要 JWT Bearer Token

**回應**：
```json
{
  "items": [
    {
      "id": 1,
      "provider_name": "openai",
      "model_type": "gpt-4.1",
      "api_key": "sk-xxx",
      "endpoint_url": null,
      "prefer": true
    }
  ],
  "total": 1,
  "page": 1,
  "size": 10,
  "pages": 1
}
```

---

### GET /api/provider

取得指定供應商詳情。

**參數**：
- `id` - 供應商 ID

---

### POST /api/provider

新增模型供應商。

**請求**：
```json
{
  "provider_name": "openai",
  "model_type": "gpt-4.1",
  "api_key": "sk-xxx",
  "endpoint_url": null,
  "prefer": true
}
```

---

### PUT /api/provider/{id}

更新模型供應商。

---

### DELETE /api/provider/{id}

刪除模型供應商。

---

### POST /api/provider/prefer

設定首選供應商。

**請求**：
```json
{
  "provider_id": 1
}
```

---

## 設定管理 API

### GET /api/configs

取得設定清單。

**參數**：
- `config_group` - 設定群組（可選）

**認證**：需要 JWT Bearer Token

**回應**：
```json
[
  {
    "id": 1,
    "config_group": "search",
    "config_name": "GOOGLE_API_KEY",
    "config_value": "xxx"
  }
]
```

---

### GET /api/configs/{config_id}

取得指定設定詳情。

---

### POST /api/configs

新增設定。

**請求**：
```json
{
  "config_group": "search",
  "config_name": "GOOGLE_API_KEY",
  "config_value": "xxx"
}
```

---

### PUT /api/configs/{config_id}

更新設定。

---

### DELETE /api/configs/{config_id}

刪除設定。

---

### GET /api/config/info

取得設定資訊（可用設定清單）。

---

## 聊天資料 API

### GET /api/chat/histories

取得聊天歷史清單。

**認證**：需要 JWT Bearer Token

---

### GET /api/chat/histories/grouped

取得分組的聊天歷史。

**參數**：
- `include_tasks` - 是否包含任務（可選）

---

### GET /api/chat/history/{history_id}

取得指定聊天歷史詳情。

---

### PUT /api/chat/history/{history_id}

更新聊天歷史。

---

### POST /api/chat/history

建立新的聊天歷史。

---

### GET /api/chat/snapshots

取得快照清單。

---

### POST /api/chat/snapshots

建立新快照。

---

### GET /api/chat/share/{share_token}

取得分享的聊天內容。

---

## 任務步驟 API

### POST /api/chat/steps

建立任務步驟（用於接收嵌入式後端同步的步驟）。

**注意**：此端點主要用於 Cloud-Connected 模式，嵌入式後端會將任務步驟同步到此端點。

**請求**：
```json
{
  "task_id": "task_123",
  "step": "to_sub_tasks",
  "data": {
    "sub_tasks": [...]
  },
  "timestamp": 1234567890.123
}
```

**同步機制說明**：

1. **觸發條件**：
   - 僅當嵌入式後端設定了 `SERVER_URL` 環境變數時才會進行同步
   - Cloud-Connected 模式：`SERVER_URL` 預設為 `https://dev.eigent.ai/api`（由 Electron 設定）
   - Local Deployment 模式：通常 `SERVER_URL` 未設定，因此**不會進行同步**

2. **同步行為**：
   - 同步是**非阻塞**的：使用 `asyncio.create_task` 非同步執行，不會影響任務執行
   - 同步失敗處理：如果通訊失敗（網路錯誤、服務不可用等），只會記錄錯誤日誌，**不會中斷任務執行**
   - 任務仍會正常完成，只是步驟不會被記錄到雲端資料庫

3. **錯誤處理**：
   - 通訊失敗時會在嵌入式後端日誌中記錄錯誤：`Failed to sync step to {url}: {error}`
   - 不會影響前端 SSE 串流，任務執行不受影響
   - 不會重試機制：失敗即放棄，不影響主流程

---

### GET /api/chat/steps

列出任務步驟。

**參數**：
- `task_id` - 任務 ID（必填）
- `step` - 步驟類型（可選）

---

### GET /api/chat/steps/{step_id}

取得指定步驟詳情。

---

### GET /api/chat/steps/playback/{task_id}

透過 SSE 播放任務步驟（重播功能）。

**參數**：
- `task_id` - 任務 ID
- `delay_time` - 延遲時間（秒，可選，最大 5 秒）

---

### PUT /api/chat/steps/{step_id}

更新任務步驟。

---

### DELETE /api/chat/steps/{step_id}

刪除任務步驟。

---

## MCP 管理 API

### GET /api/mcps

取得已安裝的 MCP 工具清單。

**認證**：需要 JWT Bearer Token

---

### POST /api/mcp/install

安裝 MCP 工具。

**請求**：
```json
{
  "name": "notion",
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-notion"],
  "env": {
    "NOTION_TOKEN": "xxx"
  }
}
```

---

### POST /api/mcp/import/Local

匯入本地 MCP 伺服器。

---

### POST /api/mcp/import/Remote

匯入遠端 MCP 伺服器。

---

### GET /api/mcp/categories

取得 MCP 分類清單。

---

### GET /api/mcp/user

取得用戶的 MCP 設定。

---

## 代理服務 API

### GET /api/proxy/google

Google 搜尋代理（僅 Cloud-Connected 模式可用）。

**注意**：當用戶沒有配置自己的 Google Search API key 時，嵌入式後端的 SearchToolkit 會透過此端點代理搜尋請求。

**參數**：
- `query` - 搜尋查詢
- `search_type` - 搜尋類型（`web` 或 `image`，預設 `web`）
- `key` - API Key（用於認證）

**認證**：需要 API Key（透過 `key` 參數或 Header）

**回應**：
```json
[
  {
    "result_id": 1,
    "title": "搜尋結果標題",
    "description": "搜尋結果描述",
    "url": "https://..."
  }
]
```

---

### POST /api/proxy/exa

Exa 搜尋代理（僅 Cloud-Connected 模式可用）。

**認證**：需要 API Key

---

## 健康檢查

### GET /health

檢查服務是否正常運行。

**回應**：
```json
{
  "status": "healthy"
}
```

---

## 錯誤碼

| Code | 說明 |
|------|------|
| 0 | 成功 |
| 1 | 參數驗證失敗 |
| 10 | 認證錯誤 |
| 13 | Token 過期 |
| 20 | 額度不足 |
| 21 | 儲存空間不足 |
| 100 | 欄位驗證錯誤 |
| 300 | 一般業務錯誤 |

---

## 相關文件

- [嵌入式後端 API](./BACKEND_API.md)
- [專案架構總覽](./ARCHITECTURE.md)
- [開發環境設置](./GETTING_STARTED.md)
- [開發指令速查](./COMMANDS.md)
