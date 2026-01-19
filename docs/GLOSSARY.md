# 專案術語表

> 本文件列出 Eigent 專案中常用的專有名詞與技術術語。

---

## A

### Agent

智能代理，能夠自主執行任務的 AI 實體。Eigent 中包含多種預定義 Agent：

| Agent | 說明 |
|-------|------|
| **Developer Agent** | 負責程式開發、終端機操作、檔案處理 |
| **Browser Agent** | 負責網頁瀏覽、資訊擷取、表單填寫 |
| **Document Agent** | 負責文件生成（PPT、Excel、Word） |
| **Multi-Modal Agent** | 負責圖片/影片處理、多模態任務 |
| **Social Media Agent** | 負責社群媒體操作 |

---

## C

### CAMEL

[CAMEL-AI](https://github.com/camel-ai/camel) 是 Eigent 使用的多智能體協作框架。全名為 **C**ommunicative **A**gents for **M**ind **E**xploration of **L**arge Language Model Society。

### ChatStore

Zustand Store，管理單一任務的完整狀態（訊息、Agent、進度等）。每個專案可有多個 ChatStore 實例。

### Cloud-Connected Mode

雲端連線模式。前端連接 Eigent Cloud 服務進行認證，多智能體執行仍在本地進行。

---

## E

### Electron

跨平台桌面應用框架。Eigent 使用 Electron 將 React 應用包裝為桌面應用，並提供原生系統整合能力。

---

## H

### Human-in-the-Loop

人類參與迴圈。當 Agent 遇到需要人類決策的情況時，會暫停並等待使用者輸入。

---

## I

### IPC (Inter-Process Communication)

程序間通訊。Electron 中 Main Process 與 Renderer Process 之間的通訊機制。

---

## L

### ListenChatAgent

Eigent 自訂的 CAMEL ChatAgent 子類別。每當執行步驟或調用工具時，會自動將狀態推送到 SSE 佇列，實現即時監控。

### Local Deployment Mode

本地部署模式。完全在本地運行，包含獨立的 PostgreSQL 資料庫，不依賴雲端服務。

---

## M

### Main Process

Electron 主程序。負責視窗管理、系統 API 存取、IPC 處理、後端生命週期管理。

### MCP (Model Context Protocol)

[模型上下文協議](https://modelcontextprotocol.io/)。由 Anthropic 提出的標準化協議，用於 LLM 與外部工具/服務的整合。Eigent 支援安裝 MCP 伺服器來擴展 Agent 能力。

---

## P

### Preload Script

Electron 預載腳本。在 Renderer Process 載入網頁前執行，透過 `contextBridge` 安全地暴露 Main Process 的 API。

### ProjectStore

Zustand Store，管理專案層級的狀態，包含多個 ChatStore 實例的映射關係。

---

## R

### Renderer Process

Electron 渲染程序。運行 React 應用的程序，相當於一個 Chromium 瀏覽器頁籤。

---

## S

### SSE (Server-Sent Events)

伺服器推送事件。單向的即時通訊協議，後端透過 SSE 將 Agent 的執行狀態即時推送給前端。

### Stack Auth

Eigent Cloud 使用的第三方認證服務，支援 OAuth（Google、GitHub）登入。

### step_solve

`backend/app/service/chat_service.py` 中的核心函式。是一個大型異步迴圈，負責：
1. 接收使用者請求
2. 協調 Workforce 進行任務分解
3. 監控 Agent 執行狀態
4. 透過 SSE 推送即時更新
5. 生成任務總結

---

## T

### Task

任務。使用者發起的一個工作請求，可被分解為多個子任務。

### TaskInfo

子任務資訊。包含任務 ID、內容、狀態、分配的 Agent 等。

### TaskLock

任務鎖。`backend/app/model/chat.py` 中的核心類別，管理單一專案的任務執行狀態，包含：
- SSE 訊息佇列 (Queue)
- 對話歷史 (conversation_history)
- 背景任務集合 (background_tasks)
- CAMEL Workforce 實例

### Toolkit

工具包。Agent 可調用的工具集合，如 Terminal Toolkit、Browser Toolkit、File Toolkit 等。

---

## U

### uv

[uv](https://github.com/astral-sh/uv) 是 Rust 編寫的超快速 Python 套件管理器。Eigent 使用 uv 管理 Python 虛擬環境和依賴。

---

## W

### WebView

Electron 中的嵌入式瀏覽器視圖。Eigent 使用多個 WebView 實例來支援 Browser Agent 的網頁自動化操作。

### Workforce

CAMEL 框架中的工作隊伍概念。由多個 Agent 組成，能夠協作完成複雜任務。Eigent 擴展了原生 Workforce，實現了：
- 自訂任務分解邏輯 (eigent_make_sub_tasks)
- 異步啟動機制 (eigent_start)
- SSE 即時監控整合

---

## Z

### Zustand

輕量級 React 狀態管理庫。Eigent 使用 Zustand 管理全域狀態，配合 `persist` 中間件實現狀態持久化。

---

## 縮寫對照表

| 縮寫 | 全名 | 說明 |
|------|------|------|
| API | Application Programming Interface | 應用程式介面 |
| CDP | Chrome DevTools Protocol | Chrome 開發者工具協議 |
| CORS | Cross-Origin Resource Sharing | 跨來源資源共享 |
| IPC | Inter-Process Communication | 程序間通訊 |
| JWT | JSON Web Token | JSON 網路令牌 |
| LLM | Large Language Model | 大型語言模型 |
| MCP | Model Context Protocol | 模型上下文協議 |
| OAuth | Open Authorization | 開放授權 |
| SSE | Server-Sent Events | 伺服器推送事件 |
| UI | User Interface | 使用者介面 |
| UX | User Experience | 使用者體驗 |

---

## 相關文件

- [專案架構總覽](./ARCHITECTURE.md)
- [前端架構說明](./FRONTEND.md)
- [後端 API 參考](./BACKEND_API.md)
- [資料模型](./DATA_MODELS.md)
