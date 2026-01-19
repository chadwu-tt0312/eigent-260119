# 前端架構說明

> 本文件詳細說明 Eigent 前端的技術架構、元件組織與狀態管理。

## 技術選型

| 類別 | 技術 | 用途 |
|-----|------|------|
| 框架 | React 18 | UI 元件建構 |
| 語言 | TypeScript | 型別安全 |
| 建置 | Vite 5 | 快速開發與打包 |
| 桌面 | Electron 33 | 跨平台桌面應用 |
| 狀態 | Zustand | 全域狀態管理 |
| 路由 | React Router v7 | 頁面導航 |
| UI | Radix UI | 無障礙基礎元件 |
| 樣式 | Tailwind CSS | 原子化 CSS |
| 動畫 | Framer Motion, GSAP | 互動動畫 |
| 編輯器 | Monaco Editor | 程式碼檢視 |
| 流程圖 | React Flow | 工作流程視覺化 |

---

## 目錄結構

```
src/
├── api/                    # HTTP 請求封裝
│   └── http.ts             # fetchGet/Post/Put/Delete, proxyFetch*
│
├── assets/                 # 靜態資源
│   └── *.svg, *.gif
│
├── components/             # UI 元件
│   ├── ui/                 # 基礎元件 (Radix-based)
│   │   ├── button.tsx
│   │   ├── dialog.tsx
│   │   ├── input.tsx
│   │   └── ...
│   ├── ChatBox/            # 聊天互動區
│   │   ├── index.tsx       # 主容器
│   │   ├── TaskBox/        # 任務卡片
│   │   └── FloatingAction.tsx
│   ├── WorkFlow/           # 工作流程圖
│   │   ├── index.tsx       # React Flow 容器
│   │   └── node.tsx        # 自訂節點
│   ├── Layout/             # 應用外殼
│   │   └── index.tsx       # SideBar + BottomBar
│   └── WindowControls/     # 視窗控制 (最小化/最大化/關閉)
│
├── pages/                  # 頁面元件
│   ├── Home.tsx            # 主工作區
│   ├── Login.tsx           # 登入頁
│   ├── SignUp.tsx          # 註冊頁
│   ├── History.tsx         # 歷史紀錄 + 設定
│   └── NotFound.tsx        # 404 頁面
│
├── store/                  # Zustand 狀態管理
│   ├── authStore.ts        # 認證、用戶偏好
│   ├── chatStore.ts        # 任務、訊息、Agent 狀態
│   ├── projectStore.ts     # 專案管理
│   ├── globalStore.ts      # UI 偏好設定
│   └── sidebarStore.ts     # 側邊欄狀態
│
├── routers/                # 路由設定
│   └── index.tsx           # Routes + ProtectedRoute
│
├── lib/                    # 工具函式
│   ├── utils.ts            # 通用工具
│   ├── oauth.ts            # OAuth 處理
│   └── llm.ts              # LLM 相關工具
│
├── i18n/                   # 國際化
│   ├── index.ts            # i18next 設定
│   └── locales/            # 語言包
│       ├── en-us/
│       ├── zh-Hans/
│       └── ...
│
├── types/                  # TypeScript 型別定義
│   ├── index.ts            # 通用型別
│   ├── chatbox.d.ts        # ChatBox 相關型別
│   └── electron.d.ts       # Electron API 型別
│
├── App.tsx                 # 根元件
├── main.tsx                # 應用入口
└── index.css               # 全域樣式
```

---

## 狀態管理 (Zustand)

### authStore

管理使用者認證與應用偏好。

```typescript
interface AuthState {
  // 認證資訊
  token: string | null;
  email: string | null;
  user_id: number | null;
  
  // 應用設定
  modelType: 'cloud' | 'local' | 'custom';
  cloud_model_type: string;
  initState: 'permissions' | 'carousel' | 'done';
  
  // 方法
  setAuth: (auth: AuthInfo) => void;
  logout: () => void;
}
```

**持久化**：使用 `persist` 中間件存儲於 `localStorage`（key: `auth-storage`）。

### chatStore

管理任務執行狀態，是應用的核心狀態。

```typescript
interface Task {
  messages: Message[];         // 對話訊息
  taskInfo: TaskInfo[];        // 子任務清單
  taskRunning: TaskInfo[];     // 執行中的任務
  taskAssigning: Agent[];      // 分配的 Agent
  status: 'running' | 'finished' | 'pending' | 'pause';
  webViewUrls: { url: string, processTaskId: string }[];
  snapshots: any[];            // 瀏覽器截圖
}

interface ChatStore {
  tasks: { [taskId: string]: Task };
  activeTaskId: string | null;
  
  // 主要方法
  create: (id?: string) => string;
  startTask: (taskId: string) => Promise<void>;
  stopTask: (taskId: string) => void;
}
```

**特點**：每個專案可有多個 chatStore 實例，透過 `projectStore` 管理。

### projectStore

管理專案與 chatStore 的關聯。

```typescript
interface ProjectStore {
  activeProjectId: string | null;
  chatStores: Map<string, VanillaChatStore>;
  
  appendInitChatStore: (projectId: string) => { taskId: string, chatStore };
}
```

---

## 路由結構

```typescript
// src/routers/index.tsx

<Routes>
  {/* 公開路由 */}
  <Route path="/login" element={<Login />} />
  <Route path="/signup" element={<Signup />} />
  
  {/* 受保護路由 */}
  <Route element={<ProtectedRoute />}>
    <Route element={<Layout />}>
      <Route path="/" element={<Home />} />
      <Route path="/history" element={<History />} />
    </Route>
  </Route>
  
  <Route path="*" element={<NotFound />} />
</Routes>
```

**ProtectedRoute**：檢查 `authStore.token` 是否存在，若無則導向 `/login`。

---

## Electron 整合

### IPC 通訊

前端透過 `window.ipcRenderer` 與 Electron 主程序通訊：

```typescript
// 調用主程序方法
const port = await window.ipcRenderer.invoke('get-backend-port');

// 監聽主程序事件
window.ipcRenderer.on('auth-code-received', (event, code) => {
  // 處理 OAuth 回調
});
```

### electronAPI

Preload Script 暴露的高階 API：

```typescript
window.electronAPI.closeWindow(forceQuit);
window.electronAPI.selectFile(options);
window.electronAPI.getPlatform();
```

---

## API 層

### http.ts

封裝兩種 HTTP 請求模式：

| 函式 | 目標 | 用途 |
|------|------|------|
| `fetchGet/Post/Put/Delete` | `localhost:{port}` | 嵌入式後端 (backend/) |
| `proxyFetchGet/Post/Put/Delete` | `VITE_PROXY_URL` | 雲端/本地部署後端 |

```typescript
// 嵌入式後端請求
const baseURL = await getBaseURL(); // http://localhost:5001
await fetchPost('/chat', data);

// 代理請求 (雲端或本地部署)
const proxyURL = await getProxyBaseURL(); // https://dev.eigent.ai 或 http://localhost:3001
await proxyFetchPost('/api/login', data);
```

---

## 主要頁面

### Home.tsx

主工作區，包含：

1. **ChatBox**：左側對話區，顯示任務訊息
2. **動態工作區**：右側根據當前 Agent 切換顯示
   - `workflow`：React Flow 工作流程圖
   - `browser_agent`：WebView 瀏覽器畫面
   - `developer_agent`：終端機輸出
   - `folder`：檔案管理

### History.tsx

歷史紀錄與設定頁面，包含：

- 任務歷史列表（支援 grid/list/table 視圖）
- 設定頁籤（Models, MCP, API, Privacy, General）

---

## 即時通訊 (SSE)

任務執行時使用 Server-Sent Events 接收後端推播：

```typescript
// src/store/chatStore.ts

fetchEventSource(`${baseURL}/chat`, {
  method: 'POST',
  body: JSON.stringify({ ... }),
  
  onmessage(event) {
    const data = JSON.parse(event.data);
    
    switch (data.step) {
      case 'to_sub_tasks':
        // 更新子任務清單
        break;
      case 'assign_task':
        // Agent 被分配任務
        break;
      case 'activate_toolkit':
        // 工具被調用
        break;
      case 'task_state':
        // 任務狀態變更
        break;
    }
  }
});
```

---

## 相關文件

- [專案架構總覽](./ARCHITECTURE.md)
- [後端 API 參考](./BACKEND_API.md)
- [Electron IPC 通道](./ELECTRON_IPC.md)
- [資料模型](./DATA_MODELS.md)
