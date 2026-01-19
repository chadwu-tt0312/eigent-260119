# 後端 API 參考

> 本文件說明 Eigent 後端 API 的端點、請求格式與回應結構。

## API 概述

Eigent 有兩套後端：

| 後端 | 位置 | 用途 | 認證方式 |
|-----|------|------|---------|
| **嵌入式後端** | `backend/` | 多智能體執行引擎 | 無（本地通訊） |
| **本地部署後端** | `server/` | 完整 API（含用戶管理） | JWT Bearer Token |

---

## 認證 API

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
  "email": "user@example.com",
  "username": "user",
  "user_id": 1
}
```

**錯誤碼**：
- `code: 10` - 帳號或密碼錯誤

---

### POST /api/register

註冊新帳號（僅本地部署模式）。

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

### POST /api/login-by_stack

使用 Stack Auth OAuth Token 登入。

**請求**：
```
POST /api/login-by_stack?token=xxx
```

**回應**：
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "email": "user@example.com"
}
```

---

## 聊天/任務 API

### POST /chat

啟動新任務（SSE 串流）。

**請求**：
```json
{
  "project_id": "proj_123",
  "task_id": "task_456",
  "question": "幫我寫一個 Python 爬蟲",
  "model_platform": "openai",
  "model_type": "gpt-4.1",
  "api_key": "sk-xxx",
  "email": "user@example.com",
  "language": "zh-cn",
  "browser_port": 9222
}
```

**回應 (SSE)**：
```
event: message
data: {"step": "to_sub_tasks", "data": {"sub_tasks": [...]}}

event: message
data: {"step": "create_agent", "data": {"agent_name": "developer_agent"}}

event: message
data: {"step": "assign_task", "data": {"task_id": "sub_1", "assignee_id": "agent_1"}}

event: message
data: {"step": "activate_toolkit", "data": {"toolkit_name": "Terminal", "method_name": "run"}}

event: message
data: {"step": "task_state", "data": {"task_id": "sub_1", "state": "DONE"}}

event: message
data: {"step": "end", "data": {"summary": "任務完成"}}
```

---

### POST /chat/{id}

改進/追問任務（保留上下文）。

**請求**：
```json
{
  "question": "請修改一下輸出格式",
  "project_id": "proj_123"
}
```

---

### DELETE /chat/{id}

強制停止任務。

---

### POST /chat/{id}/human-reply

Human-in-the-Loop 回覆。

**請求**：
```json
{
  "content": "使用者的回覆內容"
}
```

---

## 任務管理 API

### POST /task/{id}/start

開始執行已分解的任務。

---

### PUT /task/{id}

更新子任務清單。

**請求**：
```json
{
  "sub_tasks": [
    {"id": "sub_1", "content": "修改後的任務描述"}
  ]
}
```

---

### PUT /task/{id}/take-control

暫停或恢復任務。

**請求**：
```json
{
  "action": "pause"  // 或 "resume"
}
```

---

## 模型 API

### GET /api/providers

取得已設定的模型供應商清單。

**參數**：
- `prefer=true` - 只返回偏好的供應商

**回應**：
```json
{
  "items": [
    {
      "id": 1,
      "provider_name": "openai",
      "model_type": "gpt-4.1",
      "api_key": "sk-xxx",
      "endpoint_url": null
    }
  ]
}
```

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

## 設定 API

### GET /api/configs

取得設定清單。

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

## MCP 管理 API

### GET /api/mcps

取得已安裝的 MCP 工具清單。

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

## SSE 事件類型

| 事件 | 說明 |
|------|------|
| `decompose_text` | 任務分解過程中的串流文字 |
| `to_sub_tasks` | 子任務清單已生成 |
| `confirmed` | 使用者確認任務分解 |
| `create_agent` | 新 Agent 被建立 |
| `assign_task` | 任務被分配給 Agent |
| `activate_agent` | Agent 開始工作 |
| `deactivate_agent` | Agent 完成工作 |
| `activate_toolkit` | 工具被調用 |
| `task_state` | 子任務狀態變更 (DONE/FAILED) |
| `wait_confirm` | 需要人類確認 |
| `end` | 任務完成 |

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

- [專案架構總覽](./ARCHITECTURE.md)
- [前端架構說明](./FRONTEND.md)
- [Electron IPC 通道](./ELECTRON_IPC.md)
- [資料模型](./DATA_MODELS.md)
