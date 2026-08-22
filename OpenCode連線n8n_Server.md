# OpenCode 連線 n8n Server

本文件整理如何讓 OpenCode 透過 MCP（Model Context Protocol）連線至 n8n Server，以及連線後的使用方式與注意事項。

## 一、架構概念

```text
OpenCode
   │ MCP over HTTP
   ▼
n8n MCP Server
   │
   ▼
n8n 工作流、執行紀錄與管理功能
```

OpenCode 不是直接讀取 n8n 的資料庫，而是透過 n8n 提供的 MCP Server API 操作工作流。n8n Server 必須能被 OpenCode 所在的電腦連線到，若 n8n 在本機，通常需要使用 ngrok 等工具建立公開 HTTPS 連線。

## 二、必要條件

- 已啟動 n8n Server。
- n8n 已啟用 MCP Server 功能。
- 已取得 MCP Server 的 HTTP endpoint。
- OpenCode 已安裝並能讀取專案內的 `opencode.json`。
- n8n MCP endpoint 對外提供 HTTPS 連線。

## 三、OpenCode 設定檔

在專案根目錄建立 `opencode.json`：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "n8n": {
      "type": "remote",
      "url": "https://你的公開網域/mcp-server/http",
      "enabled": true,
      "headers": {
        "Accept-Encoding": "identity"
      }
    }
  }
}
```

### 設定欄位說明

| 欄位 | 說明 |
| --- | --- |
| `mcp.n8n` | MCP Server 的名稱，可自行命名。 |
| `type` | `remote` 表示連線到遠端 MCP Server。 |
| `url` | n8n MCP Server 的 HTTP endpoint。 |
| `enabled` | `true` 表示啟用此 MCP 連線。 |
| `headers` | HTTP 標頭設定；`Accept-Encoding: identity` 可避免部分壓縮回應造成的相容性問題。 |

實際使用時，請將 `url` 改成目前有效的 n8n MCP endpoint。例如：

```text
https://example.ngrok-free.dev/mcp-server/http
```

不要把 API Key、密碼或其他秘密值直接提交到 GitHub。若 MCP Server 需要驗證，應使用環境變數或安全的憑證管理方式，並依照 OpenCode 與 n8n 的版本文件設定。

## 四、啟動與確認連線

1. 啟動 n8n Server。
2. 啟動 ngrok 或其他反向代理，取得公開 HTTPS 網址。
3. 確認 MCP endpoint 路徑正確，通常是：

   ```text
   /mcp-server/http
   ```

4. 將 endpoint 填入 `opencode.json`。
5. 重新啟動 OpenCode，讓設定重新載入。
6. 在 OpenCode 中要求列出 n8n 工作流，確認可以取得工作流清單。

成功連線後，OpenCode 通常可以依照目前 n8n 帳號的權限使用工作流相關工具。

## 五、連線後可以使用的方式

### 1. 查詢工作流

可以要求 OpenCode：

```text
列出目前 n8n 中的所有工作流，並標示哪些已啟用。
```

也可以用名稱或關鍵字搜尋特定工作流。

### 2. 讀取工作流內容

```text
請讀取「下載新北市youbike及時資料」工作流，說明它的觸發器、主要節點與資料流程。
```

讀取前，應先確認工作流名稱或 ID，避免操作到相似名稱的工作流。

### 3. 執行工作流

```text
請先檢查「下載新北市youbike及時資料」的輸入格式，再以測試模式執行它。
```

執行涉及外部服務、寫入資料或傳送通知時，應先確認：

- 執行的是測試模式還是正式模式。
- 是否會修改資料或呼叫付費服務。
- 工作流所需的輸入資料是否正確。

### 4. 修改工作流

```text
請在「下載新北市youbike及時資料」完成後，增加錯誤處理，先說明你準備修改哪些節點。
```

修改前建議要求 OpenCode 先讀取工作流、說明變更內容，再進行更新。重要工作流應先匯出或保留版本紀錄，並在測試通過後才發佈。

### 5. 啟用或停用工作流

```text
請確認「工作流名稱」目前是否啟用；不要直接修改，先回報狀態。
```

啟用工作流可能會讓排程、Webhook 或其他觸發器立即開始運作，必須確認後再執行。

## 六、Expose 與權限的差異

這是最容易混淆的部分。

### Expose

Expose 指工作流是否設定為可提供給 MCP 使用，常見標記是：

```text
availableInMCP: true
```

目前的情況是：**只有 1 個工作流 expose 給 MCP**。這代表被設計為可由 MCP 使用的工作流數量是 1 個。

### n8n 連線權限

OpenCode 透過 n8n MCP Server 連線時，實際能搜尋、讀取、修改或執行哪些工作流，還取決於目前 n8n 帳號與 MCP 連線授予的權限。

因此可能出現以下情況：

- expose 給 MCP 的工作流只有 1 個。
- OpenCode 仍能看見或管理更多工作流，因為連線本身擁有工作流管理權限。
- 「已 expose」不等於「OpenCode 完全只能操作該工作流」。

本次環境查詢到的工作流總數是 7 個，其中 1 個已啟用，1 個標示為 `availableInMCP: true`。實際數量會隨 n8n Server 的設定與工作流狀態變更。

## 七、安全重點

- 不要將 n8n API Key、帳號密碼或私密 webhook URL 提交到公開 GitHub。
- ngrok 免費網址可能會變動；網址變動後，必須同步更新 `opencode.json`。
- 公開 MCP endpoint 應設定驗證、存取限制或只在課堂測試期間開啟。
- 不要讓不需要的人擁有 `workflow:update`、`workflow:execute`、`workflow:publish` 等高權限。
- 執行正式工作流前，明確區分 manual/test 與 production 執行模式。
- 修改或發佈工作流前，先確認工作流名稱、ID、輸入資料與影響範圍。
- 測試完成後關閉 ngrok 或停用不需要的 MCP endpoint。

## 八、常見問題排查

### OpenCode 找不到 n8n

- 確認 n8n Server 正在執行。
- 確認 ngrok 或反向代理仍在執行。
- 確認 `url` 使用 `https://`。
- 確認 endpoint 結尾是 `/mcp-server/http`。
- 重新啟動 OpenCode。

### 連線成功但看不到工作流

- 確認目前使用的 n8n 帳號。
- 確認該帳號對專案或工作流有讀取權限。
- 確認工作流沒有被封存或移至其他專案。
- 確認 MCP expose 設定與工作流分享設定。

### 執行工作流失敗

- 先讀取工作流描述與輸入格式。
- 先使用測試模式，不要直接使用 production 模式。
- 檢查工作流節點所需的憑證是否有效。
- 檢查外部 API、Webhook 或資料表是否可連線。
- 讀取 n8n execution details，找出實際失敗的節點。

## 九、推薦操作流程

```text
確認工作流名稱或 ID
        ↓
讀取工作流與輸入格式
        ↓
說明預計執行或修改的內容
        ↓
使用測試模式驗證
        ↓
檢查執行結果與錯誤
        ↓
確認後才發佈或正式執行
```

核心原則是：**先讀取、再確認、後執行；先測試、再發佈。**
