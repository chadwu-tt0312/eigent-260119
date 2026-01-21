# Slack 訊息傳遞流程分析

## 概述

本文件說明專案中如何將訊息傳遞到 Slack 頻道的完整流程，包含 OAuth 認證、Token 管理、Toolkit 啟用與實際訊息發送機制。

## 架構流程圖

```mermaid
sequenceDiagram
    participant User as 使用者
    participant Frontend as 前端 (React)
    participant Server as 後端 Server
    participant Backend as 後端 Backend
    participant Slack as Slack API
    participant Agent as Social Medium Agent

    User->>Frontend: 點擊 Slack 整合
    Frontend->>Server: GET /api/oauth/slack/login
    Server->>Slack: 重導向至 OAuth 授權頁面
    Slack->>User: 顯示授權頁面
    User->>Slack: 同意授權
    Slack->>Server: GET /api/oauth/slack/callback?code=xxx
    Server->>Frontend: 回傳 eigent://callback/oauth
    Frontend->>Server: POST /api/oauth/slack/token
    Server->>Slack: 交換 access_token
    Slack->>Server: 回傳 access_token
    Server->>Frontend: 回傳 token 資料
    Frontend->>Server: POST /api/configs (保存 SLACK_BOT_TOKEN)
    Frontend->>Backend: 寫入環境變數檔案

    User->>Agent: 要求發送 Slack 訊息
    Agent->>Backend: 檢查 SLACK_BOT_TOKEN
    Backend->>SlackToolkit: 啟用工具 (如果 token 存在)
    Agent->>SlackToolkit: 呼叫 send_message 方法
    SlackToolkit->>Slack: 透過 slack-sdk 發送訊息
    Slack->>SlackToolkit: 回傳結果
    SlackToolkit->>Agent: 回傳成功/失敗
```

## 詳細流程說明

### 1. OAuth 認證流程

#### 1.1 前端啟動 OAuth

**檔案位置：** `src/pages/Setting/MCP.tsx`, `src/components/IntegrationList/index.tsx`

```typescript
// 使用者點擊 Slack 整合按鈕
const url = `${baseURL}/api/oauth/${key.toLowerCase()}/login`;
window.location.href = url;
```

#### 1.2 後端生成授權 URL

**檔案位置：** `server/app/controller/oauth/oauth_controller.py`

```python
@router.get("/{app}/login")
def oauth_login(app: str, request: Request, state: Optional[str] = None):
    callback_url = str(request.url_for("OAuth Callback", app=app))
    adapter = get_oauth_adapter(app, callback_url)
    url = adapter.get_authorize_url(state)
    return RedirectResponse(str(url))
```

**檔案位置：** `server/app/component/oauth_adapter.py`

```python
class SlackOAuthAdapter(OAuthAdapter):
    def get_authorize_url(self, state: Optional[str] = None) -> Optional[str]:
        url = (
            f"https://slack.com/oauth/v2/authorize?client_id={self.client_id}"
            f"&scope={self.scope}"  # chat:write,channels:read,channels:join,groups:read,im:write
            f"&redirect_uri={self.redirect_uri}"
        )
        if state:
            url += f"&state={state}"
        return url
```

**環境變數需求：**
- `SLACK_CLIENT_ID`: Slack App 的 Client ID
- `SLACK_CLIENT_SECRET`: Slack App 的 Client Secret
- `SLACK_REDIRECT_URI`: OAuth 回調 URL (預設: `https://localhost/api/oauth/slack/callback`)
- `SLACK_SCOPE`: 授權範圍 (預設: `chat:write,channels:read,channels:join,groups:read,im:write`)

#### 1.3 Slack 回調處理

**檔案位置：** `server/app/controller/oauth/oauth_controller.py`

```python
@router.get("/{app}/callback")
def oauth_callback(app: str, request: Request, code: Optional[str] = None):
    # 重導向到前端應用程式
    redirect_url = f"eigent://callback/oauth?provider={app}&code={code}&state={state}"
    return HTMLResponse(content=html_content)
```

#### 1.4 Token 交換

**檔案位置：** `server/app/controller/oauth/oauth_controller.py`

```python
@router.post("/{app}/token")
def fetch_token(app: str, request: Request, data: OauthCallbackPayload):
    adapter = get_oauth_adapter(app, callback_url)
    token_data = adapter.fetch_token(data.code)
    return JSONResponse(token_data)
```

**檔案位置：** `server/app/component/oauth_adapter.py`

```python
def fetch_token(self, code: Optional[str]) -> Optional[Dict[str, Any]]:
    token_url = "https://slack.com/api/oauth.v2.access"
    data = {
        "client_id": self.client_id,
        "client_secret": self.client_secret,
        "code": code,
        "redirect_uri": self.redirect_uri,
    }
    with httpx.Client() as client:
        resp = client.post(token_url, data=data)
        return resp.json()  # 包含 access_token
```

### 2. Token 保存機制

#### 2.1 前端處理 OAuth 回調

**檔案位置：** `src/hooks/useIntegrationManagement.ts`

```typescript
const processOauth = useCallback(async (data: { provider: string; code: string }) => {
    // 1. 交換 authorization code 取得 access_token
    const tokenResult = await proxyFetchPost(
        `/api/oauth/${provider}/token`,
        { code: data.code }
    );
    
    // 2. 保存 token 到環境變數和資料庫
    if (provider === "slack") {
        if (tokenResult.access_token && currentItem?.env_vars?.length > 0) {
            const envVarKey = currentItem.env_vars[0]; // "SLACK_BOT_TOKEN"
            await saveEnvAndConfig(
                provider,
                envVarKey,
                tokenResult.access_token
            );
        }
    }
}, [items, saveEnvAndConfig]);
```

#### 2.2 保存到資料庫和環境變數

**檔案位置：** `src/hooks/useIntegrationManagement.ts`

```typescript
const saveEnvAndConfig = useCallback(async (
    provider: string,
    envVarKey: string,  // "SLACK_BOT_TOKEN"
    value: string        // access_token
) => {
    // 1. 保存到資料庫 (Config 表)
    const configPayload = {
        config_group: provider,      // "Slack"
        config_name: envVarKey,      // "SLACK_BOT_TOKEN"
        config_value: value,         // access_token
    };
    
    // POST /api/configs 或 PUT /api/configs/{id}
    await proxyFetchPost("/api/configs", configPayload);
    
    // 2. 寫入環境變數檔案
    if (window.electronAPI?.envWrite) {
        await window.electronAPI.envWrite(email, { 
            key: envVarKey, 
            value 
        });
    }
}, [configs, email]);
```

**資料庫模型：** `server/app/model/config/config.py`

```python
class Config(AbstractModel, DefaultTimes, table=True):
    user_id: int
    config_name: str      # "SLACK_BOT_TOKEN"
    config_value: str      # access_token 值
    config_group: str      # "Slack"
```

**環境變數檔案位置：**
- 預設路徑：`~/.eigent/.env`
- 使用者特定路徑：由 `set_user_env_path()` 設定

### 3. SlackToolkit 啟用機制

#### 3.1 Toolkit 類別定義

**檔案位置：** `backend/app/utils/toolkit/slack_toolkit.py`

```python
from camel.toolkits import SlackToolkit as BaseSlackToolkit

@auto_listen_toolkit(BaseSlackToolkit)
class SlackToolkit(BaseSlackToolkit, AbstractToolkit):
    agent_name: str = Agents.social_medium_agent
    
    def __init__(self, api_task_id: str, timeout: float | None = None):
        super().__init__(timeout)
        self.api_task_id = api_task_id
    
    @classmethod
    def get_can_use_tools(cls, api_task_id: str) -> list[FunctionTool]:
        # 檢查環境變數是否存在
        if env("SLACK_BOT_TOKEN") or env("SLACK_USER_TOKEN"):
            return SlackToolkit(api_task_id).get_tools()
        else:
            return []  # 沒有 token 則不啟用工具
```

**關鍵點：**
- 繼承自 `camel.toolkits.SlackToolkit`，實際的 Slack API 呼叫由基礎類別處理
- 使用 `@auto_listen_toolkit` 裝飾器，自動監聽工具呼叫
- 只有在 `SLACK_BOT_TOKEN` 或 `SLACK_USER_TOKEN` 存在時才啟用

#### 3.2 環境變數讀取

**檔案位置：** `backend/app/component/environment.py`

```python
def env(key: str, default=None):
    """
    讀取環境變數的優先順序：
    1. 使用者特定的 .env 檔案 (~/.eigent/.env 或自訂路徑)
    2. 全域環境變數
    """
    # 檢查使用者特定的環境變數檔案
    if hasattr(_thread_local, 'env_path') and os.path.exists(_thread_local.env_path):
        from dotenv import dotenv_values
        user_env_values = dotenv_values(_thread_local.env_path)
        if key in user_env_values:
            return user_env_values[key] or default
    
    # 回退到全域環境變數
    return os.getenv(key, default)
```

#### 3.3 Agent 整合

**檔案位置：** `backend/app/utils/agent.py`

```python
@traceroot.trace()
async def social_medium_agent(options: Chat):
    """
    Social Medium Agent 負責處理社交媒體相關任務
    包含：WhatsApp, Twitter, LinkedIn, Reddit, Notion, Slack 等
    """
    tools = [
        *WhatsAppToolkit.get_can_use_tools(options.project_id),
        *TwitterToolkit.get_can_use_tools(options.project_id),
        # *SlackToolkit.get_can_use_tools(options.project_id),  # 目前被註解
        # ... 其他工具
    ]
    
    # 建立 ChatAgent 並提供工具
    return ListenChatAgent(
        system_message=system_message,
        options=options,
        tools=tools,
    )
```

**注意：** 目前 `SlackToolkit.get_can_use_tools()` 在 `social_medium_agent` 中被註解掉（第 1601 行），但 Toolkit 本身已實作完成。

### 4. 訊息發送流程

#### 4.1 工具呼叫流程

當 Agent 需要發送 Slack 訊息時：

1. **Agent 決策：** `social_medium_agent` 根據使用者需求決定使用 Slack 工具
2. **工具檢查：** `SlackToolkit.get_can_use_tools()` 檢查環境變數
3. **工具啟用：** 如果 token 存在，返回可用的工具列表
4. **API 呼叫：** 基礎類別 `BaseSlackToolkit` 使用 `slack-sdk` 套件發送訊息

#### 4.2 基礎類別功能

`camel.toolkits.SlackToolkit` 提供以下功能（由 `slack-sdk` 實作）：

- `create_channel()`: 建立新頻道（公開或私有）
- `join_channel()`: 加入現有頻道
- `leave_channel()`: 離開頻道
- `send_message()`: 在頻道中發送訊息
- `delete_message()`: 刪除訊息
- `get_channel_info()`: 取得頻道資訊
- `get_channel_history()`: 取得頻道訊息歷史

#### 4.3 監聽機制

**檔案位置：** `backend/app/utils/listen/toolkit_listen.py`

`@auto_listen_toolkit` 裝飾器會：
- 自動監聽工具呼叫
- 記錄工具輸入和輸出
- 透過 SSE (Server-Sent Events) 將狀態推送到前端

## 配置需求

### 環境變數設定

#### 後端 Server (OAuth 認證)

```bash
SLACK_CLIENT_ID=your_slack_client_id
SLACK_CLIENT_SECRET=your_slack_client_secret
SLACK_REDIRECT_URI=https://your-domain.com/api/oauth/slack/callback
SLACK_SCOPE=chat:write,channels:read,channels:join,groups:read,im:write
```

#### 後端 Backend (訊息發送)

```bash
SLACK_BOT_TOKEN=xoxb-your-bot-token  # 或
SLACK_USER_TOKEN=xoxp-your-user-token
```

### 資料庫配置

**檔案位置：** `server/app/model/config/config.py`

```python
ConfigGroup.SLACK.value: {
    "env_vars": ["SLACK_BOT_TOKEN"],
    "toolkit": "slack_toolkit",
}
```

## 依賴套件

### Python 套件

- `slack-sdk`: Slack API 客戶端（由 `camel.toolkits` 使用）
- `httpx`: OAuth token 交換時的 HTTP 客戶端
- `python-dotenv`: 環境變數檔案讀取

### 前端套件

- React Router: 路由處理
- Electron IPC: 環境變數檔案寫入

## 注意事項

1. **目前狀態：** `SlackToolkit` 在 `social_medium_agent` 中被註解掉，需要取消註解才能使用
2. **Token 管理：** Token 同時保存在資料庫和環境變數檔案中，確保一致性很重要
3. **環境變數優先順序：** 使用者特定的 `.env` 檔案優先於全域環境變數
4. **OAuth Scope：** 確保 Slack App 已申請所需的 OAuth Scopes
5. **安全性：** Token 應妥善保管，避免洩露

## 啟用 SlackToolkit

若要啟用 Slack 功能，需要：

1. **取消註解：** 在 `backend/app/utils/agent.py` 第 1601 行取消註解
   ```python
   *SlackToolkit.get_can_use_tools(options.project_id),
   ```

2. **設定環境變數：** 確保 `SLACK_BOT_TOKEN` 或 `SLACK_USER_TOKEN` 已設定

3. **測試：** 透過 Agent 發送測試訊息到 Slack 頻道

## 相關檔案清單

### 後端檔案

- `backend/app/utils/toolkit/slack_toolkit.py` - SlackToolkit 實作
- `backend/app/utils/agent.py` - Agent 定義與工具整合
- `backend/app/component/environment.py` - 環境變數管理
- `server/app/controller/oauth/oauth_controller.py` - OAuth 控制器
- `server/app/component/oauth_adapter.py` - OAuth 適配器
- `server/app/model/config/config.py` - 配置模型

### 前端檔案

- `src/hooks/useIntegrationManagement.ts` - OAuth 流程處理
- `src/pages/Setting/MCP.tsx` - 整合設定頁面
- `src/components/IntegrationList/index.tsx` - 整合列表元件

## 參考資料

- [Slack API 文件](https://api.slack.com/)
- [Slack OAuth 流程](https://api.slack.com/authentication/oauth-v2)
- [camel.toolkits 文件](https://github.com/camel-ai/camel)
