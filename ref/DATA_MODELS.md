# 核心資料模型

> 本文件說明 Eigent 專案中的核心資料結構與型別定義。

## 前端資料模型 (TypeScript)

### AuthState

認證與使用者偏好狀態。

```typescript
// src/store/authStore.ts

interface AuthState {
  // 認證資訊
  token: string | null;
  username: string | null;
  email: string | null;
  user_id: number | null;

  // 應用設定
  appearance: 'light' | 'dark';
  language: 'system' | 'en' | 'zh-cn' | 'zh-tw' | 'ja' | 'ko';
  isFirstLaunch: boolean;
  initState: 'permissions' | 'carousel' | 'done';
  
  // 模型設定
  modelType: 'cloud' | 'local' | 'custom';
  cloud_model_type: CloudModelType;
  
  // 共享相關
  share_token: string | null;
  
  // Worker 清單
  workerListData: { [email: string]: Agent[] };
}

type CloudModelType = 
  | 'gpt-4.1' | 'gpt-4.1-mini' 
  | 'gpt-5' | 'gpt-5.1' | 'gpt-5.2' | 'gpt-5-mini'
  | 'claude-sonnet-4-5' | 'claude-sonnet-4-20250514' | 'claude-3-5-haiku-20241022'
  | 'gemini/gemini-2.5-pro' | 'gemini-2.5-flash' | 'gemini-3-pro-preview';
```

---

### Task

任務狀態（每個對話/任務的完整狀態）。

```typescript
// src/store/chatStore.ts

interface Task {
  // 訊息與內容
  messages: Message[];
  summaryTask: string;
  attaches: File[];
  
  // 任務資訊
  taskInfo: TaskInfo[];           // 原始子任務清單
  taskRunning: TaskInfo[];        // 執行中的任務快照
  taskAssigning: Agent[];         // 已分配的 Agent
  
  // 狀態
  type: string;                   // 'normal' | 'replay' | 'share'
  status: 'running' | 'finished' | 'pending' | 'pause';
  isPending: boolean;
  hasWaitComfirm: boolean;
  hasMessages: boolean;
  isTakeControl: boolean;
  isTaskEdit: boolean;
  isContextExceeded: boolean;
  
  // 工作區
  activeWorkSpace: string | null; // 'workflow' | 'browser_agent' | 'developer_agent' | 'folder'
  activeAgent: string;
  activeAsk: string;
  askList: Message[];
  
  // 瀏覽器相關
  webViewUrls: { url: string; processTaskId: string }[];
  snapshots: any[];
  snapshotsTemp: any[];
  
  // 檔案
  fileList: FileInfo[];
  selectedFile: FileInfo | null;
  nuwFileNum: number;
  
  // 統計
  progressValue: number;
  taskTime: number;
  elapsed: number;
  tokens: number;
  delayTime: number;
  
  // 其他
  cotList: string[];
  hasAddWorker: boolean;
  streamingDecomposeText: string;
}
```

---

### Message

對話訊息。

```typescript
interface Message {
  id: string;
  role: 'user' | 'agent' | 'system';
  content: string;
  step?: string;           // SSE 事件步驟
  isConfirm?: boolean;     // 是否已確認
  taskType?: number;       // 1: 一般, 2: 重播
  showType?: 'list' | 'grid';
  task_id?: string;
  attaches?: File[];
}
```

---

### Agent

Agent 實例。

```typescript
interface Agent {
  agent_id: string;
  name: string;
  type: AgentNameType;
  status?: AgentStatus;
  tasks: TaskInfo[];
  log: AgentMessage[];
  img: any[];
  tools?: string[];
  activeWebviewIds?: { id: string; img: string; processTaskId: string; url: string }[];
  workerInfo?: {
    name: string;
    description: string;
    tools: string[];
    mcp_tools: string[];
  };
}

type AgentNameType = 
  | 'developer_agent' 
  | 'browser_agent' 
  | 'document_agent' 
  | 'multi_modal_agent'
  | 'social_medium_agent';

type AgentStatus = 'running' | 'completed' | 'failed' | 'waiting';
```

---

### TaskInfo

子任務資訊。

```typescript
interface TaskInfo {
  id: string;
  content: string;
  status?: 'waiting' | 'running' | 'completed' | 'failed';
  agent?: Agent;
  report?: string;
  failure_count?: number;
  reAssignTo?: string;
  toolkits?: Toolkit[];
}

interface Toolkit {
  toolkitId: string;
  toolkitName: string;
  toolkitMethods: string;
  message: string;
  toolkitStatus: AgentStatus;
}
```

---

### FileInfo

檔案資訊。

```typescript
interface FileInfo {
  name: string;
  path: string;
  size?: number;
  type?: string;
  modifiedTime?: string;
}
```

---

## 後端資料模型 (Python/Pydantic)

### Chat

聊天請求模型。

```python
# backend/app/model/chat.py

class Chat(BaseModel):
    project_id: str
    task_id: str
    question: str
    model_platform: str       # 'openai', 'anthropic', 'gemini'
    model_type: str           # 'gpt-4.1', 'claude-sonnet-4-5'
    api_key: str
    api_url: str | None = None
    email: str
    language: str = "en"
    browser_port: int = 9222
    allow_local_system: bool = True
    attaches: list[str] = []
    installed_mcp: dict = {}
    new_agents: list[dict] = []
    env_path: str = ""
    search_config: dict = {}
    extra_params: dict = {}
```

---

### TaskLock

任務狀態管理器（核心狀態類）。

```python
# backend/app/model/chat.py

class TaskLock:
    """
    管理單一專案的任務執行狀態。
    每個 project_id 對應一個 TaskLock 實例。
    """
    
    def __init__(self, project_id: str):
        self.project_id = project_id
        self.queue: asyncio.Queue = asyncio.Queue()  # SSE 訊息佇列
        self.conversation_history: list = []          # 對話歷史
        self.background_tasks: set = set()            # 背景任務集合
        self.is_paused: bool = False                  # 是否暫停
        self.current_task_id: str | None = None       # 當前任務 ID
        self.workforce: Workforce | None = None       # CAMEL Workforce 實例
```

---

### ActionData

SSE 事件資料結構。

```python
# backend/app/model/chat.py

class ActionData(BaseModel):
    """
    後端推送給前端的動作資料。
    """
    step: str                     # 事件類型
    data: dict                    # 事件資料
    
    # step 可能的值:
    # - 'decompose_text': 任務分解串流文字
    # - 'to_sub_tasks': 子任務清單
    # - 'confirmed': 使用者確認
    # - 'create_agent': Agent 建立
    # - 'assign_task': 任務分配
    # - 'activate_agent': Agent 開始工作
    # - 'deactivate_agent': Agent 結束工作
    # - 'activate_toolkit': 工具調用
    # - 'task_state': 任務狀態變更
    # - 'wait_confirm': 等待人類確認
    # - 'end': 任務結束
```

---

### LoginResponse

登入回應。

```python
# server/app/model/user/user.py

class LoginResponse(BaseModel):
    token: str
    email: str
    username: str | None = None
    user_id: int | None = None
```

---

### User

使用者模型（資料庫）。

```python
# server/app/model/user/user.py

class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    email: str = Field(unique=True, index=True)
    username: str | None = None
    nickname: str | None = None
    password: str | None = None      # bcrypt 雜湊
    avatar: str | None = None
    stack_id: str | None = None      # Stack Auth ID
    status: Status = Status.Normal
    created_at: datetime = Field(default_factory=datetime.now)
    updated_at: datetime = Field(default_factory=datetime.now)

class Status(str, Enum):
    Normal = "normal"
    Block = "block"
```

---

### Provider

模型供應商設定。

```python
# server/app/model/provider/provider.py

class Provider(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    user_id: int
    provider_name: str             # 'openai', 'anthropic', 'gemini'
    model_type: str                # 'gpt-4.1'
    api_key: str                   # 加密儲存
    endpoint_url: str | None = None
    prefer: bool = False           # 是否為偏好
    encrypted_config: dict = {}    # 額外加密設定
```

---

### ChatHistory

聊天歷史紀錄。

```python
# server/app/model/chat/chat_history.py

class ChatHistory(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    project_id: str
    task_id: str
    user_id: int
    question: str
    project_name: str | None = None
    summary: str | None = None
    model_platform: str
    model_type: str
    api_url: str
    language: str = "en"
    status: int = 1
    tokens: int = 0
    created_at: datetime = Field(default_factory=datetime.now)
    updated_at: datetime = Field(default_factory=datetime.now)
```

---

## 資料流關係圖

```
┌─────────────────────────────────────────────────────────────┐
│                        Frontend                              │
├─────────────────────────────────────────────────────────────┤
│  AuthState ──────────► ChatStore ──────────► ProjectStore   │
│      │                     │                      │         │
│      │                     ▼                      │         │
│      │              ┌─────────────┐               │         │
│      │              │    Task     │               │         │
│      │              │  ┌───────┐  │               │         │
│      │              │  │Message│  │               │         │
│      │              │  │Agent  │  │               │         │
│      │              │  │TaskInfo│ │               │         │
│      │              │  └───────┘  │               │         │
│      │              └─────────────┘               │         │
└──────┼────────────────────┼───────────────────────┼─────────┘
       │                    │ SSE                   │
       │                    ▼                       │
┌──────┼────────────────────┼───────────────────────┼─────────┐
│      │              ┌─────────────┐               │         │
│      │              │  TaskLock   │               │         │
│      │              │  ┌───────┐  │               │         │
│      │              │  │Queue  │  │               │         │
│      │              │  │History│  │               │         │
│      │              │  │Workfrc│  │               │         │
│      │              │  └───────┘  │               │         │
│      │              └─────────────┘               │         │
│      │                    │                       │         │
│      ▼                    ▼                       ▼         │
│   User ◄────────────► ChatHistory ◄──────────► Provider    │
│                                                             │
│                        Backend                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 相關文件

- [專案架構總覽](./ARCHITECTURE.md)
- [前端架構說明](./FRONTEND.md)
- [後端 API 參考](./BACKEND_API.md)
- [術語表](./GLOSSARY.md)
