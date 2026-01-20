# 嵌入式後端 API 參考

> 本文件說明 Eigent 嵌入式後端（`backend/`）的 API 端點、請求格式與回應結構。

## API 概述

**嵌入式後端**（Embedded Backend）是 Eigent 的多智能體任務執行引擎，由 Electron 自動啟動。

| 項目 | 說明 |
|-----|------|
| **位置** | `backend/` |
| **啟動方式** | 由 Electron 自動啟動（動態埠，預設從 5001 起） |
| **認證方式** | 無（本地通訊，透過 Electron IPC 取得埠號） |
| **主要功能** | 多智能體任務執行、CAMEL Workforce 協調 |
| **運行模式** | Cloud-Connected 和 Local Deployment 兩種模式下都會運行 |

**相關文件**：
- [Local Server 和 Cloud Service API](./SERVER_API.md) - 用戶管理、設定管理、資料持久化
- [專案架構總覽](./ARCHITECTURE.md)

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

## 模型驗證 API

### POST /model/validate

驗證模型配置和工具呼叫支援。

**請求**：
```json
{
  "model_platform": "OPENAI",
  "model_type": "GPT_4O_MINI",
  "api_key": "sk-xxx",
  "url": null,
  "model_config_dict": null,
  "extra_params": null
}
```

**回應**：
```json
{
  "is_valid": true,
  "is_tool_calls": true,
  "error_code": null,
  "error": null,
  "message": "Validation Success"
}
```

---

## 工具管理 API

### GET /tools/available

列出所有可安裝的 MCP 工具。

**回應**：
```json
{
  "tools": [
    {
      "name": "notion",
      "display_name": "Notion MCP",
      "description": "Notion workspace integration",
      "toolkit_class": "NotionMCPToolkit",
      "requires_auth": true
    }
  ]
}
```

### POST /install/tool/{tool}

安裝指定的工具。

### DELETE /uninstall/tool/{tool}

卸載指定的工具。

### GET /oauth/status/{provider}

取得 OAuth 狀態。

### POST /oauth/cancel/{provider}

取消 OAuth 授權。

### POST /browser/login

開啟瀏覽器進行登入。

### GET /browser/cookies

列出所有 Cookie 網域。

### GET /browser/cookies/{domain}

取得指定網域的 Cookie。

### DELETE /browser/cookies/{domain}

刪除指定網域的 Cookie。

### DELETE /browser/cookies

刪除所有 Cookie。

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

- [Local Server 和 Cloud Service API](./SERVER_API.md)
- [專案架構總覽](./ARCHITECTURE.md)
- [前端架構說明](./FRONTEND.md)
- [Electron IPC 通道](./ELECTRON_IPC.md)
- [資料模型](./DATA_MODELS.md)
