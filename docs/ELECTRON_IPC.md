# Electron IPC 通道說明

> 本文件說明 Eigent 中 Electron 主程序與渲染程序之間的 IPC 通訊通道。

## IPC 架構概述

```
┌─────────────────────────────────────────────────────────────┐
│                    Renderer Process                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 React Application                    │    │
│  │                                                      │    │
│  │  window.ipcRenderer.invoke('channel', args)         │    │
│  │  window.ipcRenderer.on('event', callback)           │    │
│  │  window.electronAPI.methodName(args)                │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                    contextBridge (Preload)
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     Main Process                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              ipcMain.handle / ipcMain.on             │    │
│  │                                                      │    │
│  │  - 視窗管理          - 檔案系統                      │    │
│  │  - 後端生命週期      - MCP 管理                      │    │
│  │  - WebView 控制      - 環境變數                      │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## 系統資訊通道

### get-app-version

取得應用程式版本。

```typescript
const version = await window.ipcRenderer.invoke('get-app-version');
// 回傳: "1.2.0"
```

### get-backend-port

取得嵌入式後端服務的埠號。

```typescript
const port = await window.ipcRenderer.invoke('get-backend-port');
// 回傳: 5001
```

### get-browser-port

取得 Chrome DevTools Protocol 的埠號（用於 Browser Agent）。

```typescript
const port = await window.ipcRenderer.invoke('get-browser-port');
// 回傳: 9222
```

### get-system-language

取得系統語言。

```typescript
const lang = await window.ipcRenderer.invoke('get-system-language');
// 回傳: "zh-cn" 或 "en"
```

### get-home-dir

取得使用者家目錄路徑。

```typescript
const homeDir = await window.ipcRenderer.invoke('get-home-dir');
// 回傳: "C:\\Users\\username" 或 "/Users/username"
```

### is-fullscreen

檢查視窗是否全螢幕。

```typescript
const isFullscreen = await window.ipcRenderer.invoke('is-fullscreen');
// 回傳: true 或 false
```

---

## 視窗控制通道

### window-close

關閉視窗。

```typescript
window.ipcRenderer.send('window-close', { isForceQuit: false });
```

### window-minimize

最小化視窗。

```typescript
window.ipcRenderer.send('window-minimize');
```

### window-toggle-maximize

切換視窗最大化狀態。

```typescript
window.ipcRenderer.send('window-toggle-maximize');
```

---

## 應用程式生命週期

### restart-app

重新啟動應用程式。

```typescript
await window.ipcRenderer.invoke('restart-app');
```

### restart-backend

重新啟動 Python 後端服務。

```typescript
const result = await window.ipcRenderer.invoke('restart-backend');
// 回傳: { success: true } 或 { success: false, error: "..." }
```

### install-dependencies

安裝/重試安裝依賴套件（uv, bun, Python venv）。

```typescript
const result = await window.ipcRenderer.invoke('install-dependencies');
// 回傳: { success: true, isInstalled: true }
```

### check-tool-installed

檢查必要工具是否已安裝。

```typescript
const result = await window.ipcRenderer.invoke('check-tool-installed');
// 回傳: { success: true, isInstalled: true }
```

### get-installation-status

取得安裝狀態。

```typescript
const status = await window.ipcRenderer.invoke('get-installation-status');
// 回傳: { success: true, isInstalling: false, hasLockFile: true, timestamp: 1234567890 }
```

---

## 檔案操作通道

### select-file

開啟檔案選擇對話框。

```typescript
const result = await window.ipcRenderer.invoke('select-file', {
  properties: ['openFile', 'multiSelections'],
  filters: [{ name: 'Documents', extensions: ['pdf', 'docx'] }]
});
// 回傳: { success: true, files: [{ filePath: "...", fileName: "..." }], fileCount: 1 }
```

### read-file

讀取檔案內容（Buffer）。

```typescript
const result = await window.ipcRenderer.invoke('read-file', '/path/to/file');
// 回傳: { success: true, data: Buffer, size: 1234 }
```

### read-file-dataurl

讀取檔案並轉換為 Data URL（用於圖片預覽）。

```typescript
const dataUrl = await window.ipcRenderer.invoke('read-file-dataurl', '/path/to/image.png');
// 回傳: "data:image/png;base64,iVBORw0KGgo..."
```

### reveal-in-folder

在檔案總管中顯示檔案。

```typescript
await window.ipcRenderer.invoke('reveal-in-folder', '/path/to/file');
```

### download-file

下載網路檔案到本地。

```typescript
const result = await window.ipcRenderer.invoke('download-file', 'https://example.com/file.pdf');
// 回傳: { success: true, path: "C:\\Users\\...\\Downloads\\file.pdf" }
```

### delete-folder

刪除使用者的 MCP 設定資料夾。

```typescript
const result = await window.ipcRenderer.invoke('delete-folder', 'user@example.com');
// 回傳: { success: true, message: "Folder deleted successfully" }
```

---

## MCP 管理通道

### mcp-install

安裝 MCP 伺服器工具。

```typescript
await window.ipcRenderer.invoke('mcp-install', 'notion', {
  command: 'npx',
  args: ['-y', '@modelcontextprotocol/server-notion'],
  env: { NOTION_TOKEN: 'xxx' }
});
// 回傳: { success: true }
```

### mcp-remove

移除 MCP 伺服器工具。

```typescript
await window.ipcRenderer.invoke('mcp-remove', 'notion');
// 回傳: { success: true }
```

### mcp-update

更新 MCP 伺服器設定。

```typescript
await window.ipcRenderer.invoke('mcp-update', 'notion', {
  command: 'npx',
  args: ['-y', '@modelcontextprotocol/server-notion@latest'],
  env: { NOTION_TOKEN: 'new_token' }
});
// 回傳: { success: true }
```

### mcp-list

取得已安裝的 MCP 伺服器清單。

```typescript
const mcpConfig = await window.ipcRenderer.invoke('mcp-list');
// 回傳: { mcpServers: { notion: { command: "npx", args: [...] } } }
```

---

## 環境變數通道

### get-env-path

取得使用者的 .env 檔案路徑。

```typescript
const envPath = await window.ipcRenderer.invoke('get-env-path', 'user@example.com');
// 回傳: "C:\\Users\\.../.eigent/user@example.com/.env"
```

### env-write

寫入環境變數。

```typescript
await window.ipcRenderer.invoke('env-write', 'user@example.com', {
  key: 'OPENAI_API_KEY',
  value: 'sk-xxx'
});
// 回傳: { success: true }
```

### env-remove

移除環境變數。

```typescript
await window.ipcRenderer.invoke('env-remove', 'user@example.com', 'OPENAI_API_KEY');
// 回傳: { success: true }
```

### get-env-has-key

檢查環境變數是否存在。

```typescript
const result = await window.ipcRenderer.invoke('get-env-has-key', 'user@example.com', 'OPENAI_API_KEY');
// 回傳: { success: true }  // 表示存在
```

---

## WebView 控制通道

### create-webview

建立新的 WebView 實例。

```typescript
await window.ipcRenderer.invoke('create-webview', 'webview_id');
```

### show-webview

顯示 WebView。

```typescript
await window.ipcRenderer.invoke('show-webview', 'webview_id');
```

### hide-webview

隱藏 WebView。

```typescript
await window.ipcRenderer.invoke('hide-webview', 'webview_id');
```

### hide-all-webview

隱藏所有 WebView。

```typescript
await window.ipcRenderer.invoke('hide-all-webview');
```

### capture-webview

擷取 WebView 截圖。

```typescript
const screenshot = await window.ipcRenderer.invoke('capture-webview', 'webview_id');
// 回傳: Base64 編碼的圖片資料
```

### change-view-size

調整 WebView 尺寸。

```typescript
await window.ipcRenderer.invoke('change-view-size', { width: 800, height: 600 });
```

### webview-destroy

銷毀 WebView 實例。

```typescript
await window.ipcRenderer.invoke('webview-destroy', 'webview_id');
```

---

## 專案/任務檔案通道

### get-file-list

取得任務產生的檔案清單。

```typescript
const files = await window.ipcRenderer.invoke('get-file-list', 'user@example.com', 'task_123', 'project_456');
// 回傳: [{ name: "output.txt", path: "...", size: 1234 }]
```

### delete-task-files

刪除任務相關檔案。

```typescript
await window.ipcRenderer.invoke('delete-task-files', 'user@example.com', 'task_123', 'project_456');
```

### open-file

開啟檔案進行檢視（支援 docx, xlsx, pptx, pdf, csv 等）。

```typescript
const content = await window.ipcRenderer.invoke('open-file', 'docx', '/path/to/file.docx', false);
```

### get-log-folder

取得日誌資料夾路徑。

```typescript
const logFolder = await window.ipcRenderer.invoke('get-log-folder', 'user@example.com');
```

---

## 日誌相關通道

### export-log

匯出應用程式日誌。

```typescript
const result = await window.ipcRenderer.invoke('export-log');
// 回傳: { success: true, savedPath: "C:\\...\\eigent-1.2.0-win32-x64-1234567890.log" }
```

### upload-log

上傳任務日誌到雲端。

```typescript
const result = await window.ipcRenderer.invoke('upload-log', 
  'user@example.com', 
  'task_123', 
  'https://api.eigent.ai', 
  'bearer_token'
);
// 回傳: { success: true, data: {...} }
```

---

## 事件監聽（從 Main 到 Renderer）

### auth-code-received

OAuth 授權碼回調。

```typescript
window.ipcRenderer.on('auth-code-received', (event, code) => {
  // 處理 OAuth 授權碼
});
```

### oauth-authorized

OAuth 授權完成。

```typescript
window.ipcRenderer.on('oauth-authorized', (event, { provider, code }) => {
  // 處理 OAuth 結果
});
```

### install-dependencies-complete

依賴安裝完成。

```typescript
window.ipcRenderer.on('install-dependencies-complete', (event, { success, code }) => {
  if (success) {
    // 安裝成功，可以啟動後端
  }
});
```

### backend-started

後端服務已啟動。

```typescript
window.ipcRenderer.on('backend-started', (event, { success, port, error }) => {
  if (success) {
    console.log(`Backend running on port ${port}`);
  }
});
```

### before-close

視窗即將關閉（用於確認對話框）。

```typescript
window.ipcRenderer.on('before-close', () => {
  // 顯示確認對話框或直接關閉
  window.electronAPI.closeWindow(true);
});
```

---

## electronAPI 高階封裝

Preload Script 提供的便捷 API：

```typescript
window.electronAPI = {
  closeWindow: (forceQuit: boolean) => void,
  minimizeWindow: () => void,
  toggleMaximize: () => void,
  selectFile: (options) => Promise<Result>,
  getPlatform: () => 'darwin' | 'win32' | 'linux',
  webviewDestroy: (id: string) => void,
  // ...更多方法
}
```

---

## 相關文件

- [專案架構總覽](./ARCHITECTURE.md)
- [前端架構說明](./FRONTEND.md)
- [後端 API 參考](./BACKEND_API.md)
