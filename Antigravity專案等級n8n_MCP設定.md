# Antigravity 專案等級 n8n MCP 設定教學講義

本講義說明如何在 **Google Antigravity** 環境中，為單一專案（Project-level / Workspace-level）設定 **n8n MCP（Model Context Protocol）** 連線。

> [!IMPORTANT]
> **認證重要須知**：
> **Antigravity 目前不支援互動式 OAuth 授權流程**。如果 n8n MCP Server 啟用認證，**必須使用 Access Token（Bearer Token）**，透過設定檔的 `headers` 帶入 `Authorization: Bearer <YOUR_ACCESS_TOKEN>` 進行連線授權。

---

## 一、核心觀念與架構

### 1. 什麼是專案等級（Project-level）設定？
- **全域設定（Global Level）**：位於系統目錄（如 `~/.gemini/`），適用於本機所有的專案與對話。
- **專案等級設定（Project/Workspace Level）**：位於專案根目錄下的 `.agents/` 目錄中，**僅對當前專案目錄生效**，設定可隨著 Git 專案版本控制分享給協同作業團隊。

### 2. 連線與認證架構
```text
┌────────────────────────────────────────────────────────┐
│                   Antigravity IDE                      │
│                                                        │
│  讀取專案設定: .agents/mcp_config.json                 │
│  發送請求帶有 Header: Authorization: Bearer <TOKEN>   │
└──────────────────────────┬─────────────────────────────┘
                           │
                           │ (SSE / HTTP 協定)
                           ▼
              ┌─────────────────────────┐
              │   ngrok / 反向代理轉發   │
              └────────────┬────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │     n8n MCP Server      │
              │   (/mcp-server/http)    │
              │                         │
              │  ✔ 驗證 Bearer Token    │
              │  ✔ 提供工作流/節點工具   │
              └────────────┬────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │      n8n 自動化引擎      │
              │ (工作流、節點、執行紀錄) │
              └─────────────────────────┘
```

---

## 二、專案目錄結構

在你的專案根目錄建立 `.agents` 目錄，結構如下：

```text
專案根目錄/
├── .agents/
│   ├── mcp_config.json                 # ★ 專案主要 MCP 設定檔
│   └── plugins/
│       └── n8n/
│           ├── plugin.json             # Plugin 宣告檔
│           └── mcp_config.json         # Plugin 專屬 MCP 設定檔
├── opencode.json                       # (可選) 其他 Client 如 OpenCode 設定檔
└── ... (專案其他程式碼)
```

---

## 三、步驟教學與設定檔範例

### 步驟 1：取得 n8n MCP Access Token
由於 Antigravity 不支援瀏覽器彈跳 OAuth 驗證，請先從 n8n 系統取得授權用的 Access Token / JWT Token。

---

### 步驟 2：設定專案 MCP 設定檔 (`.agents/mcp_config.json`)

在專案根目錄下的 `.agents/mcp_config.json` 填入遠端伺服器網址與 Bearer Token：

```json
{
  "mcpServers": {
    "n8n": {
      "serverUrl": "https://<你的ngrok網址或公開伺服器網址>/mcp-server/http",
      "headers": {
        "Authorization": "Bearer <你的_n8n_ACCESS_TOKEN>",
        "Accept-Encoding": "identity"
      }
    }
  }
}
```

> [!TIP]
> - `serverUrl`：請務必指向 n8n MCP HTTP 端點（通常結尾為 `/mcp-server/http`）。
> - `headers.Authorization`：固定格式為 `Bearer ` 加上你的 Access Token。
> - `headers.Accept-Encoding`：設為 `"identity"` 可避免壓縮導致 MCP 通訊協定解析異常。

---

### 步驟 3：(建議) 建立 Plugin 設定以確保完整相容

建立 `.agents/plugins/n8n/plugin.json`：
```json
{
  "name": "n8n"
}
```

建立 `.agents/plugins/n8n/mcp_config.json`：
```json
{
  "mcpServers": {
    "n8n": {
      "serverUrl": "https://<你的ngrok網址或公開伺服器網址>/mcp-server/http",
      "headers": {
        "Authorization": "Bearer <你的_n8n_ACCESS_TOKEN>",
        "Accept-Encoding": "identity"
      }
    }
  }
}
```

---

## 四、設定欄位說明

| 欄位路徑 | 型別 | 必填 | 說明 |
| :--- | :--- | :---: | :--- |
| `mcpServers.n8n` | 物件 | 是 | MCP Server 的名稱識別碼（此處定義為 `n8n`）。 |
| `serverUrl` | 字串 | 是 | 遠端 n8n MCP Server 的 HTTP/SSE Endpoint 完整網址。 |
| `headers` | 物件 | 是 | 傳送給 MCP Server 的 HTTP 標頭。 |
| `headers.Authorization` | 字串 | 是 | **存取認證**：格式為 `Bearer <TOKEN>`。 |
| `headers.Accept-Encoding` | 字串 | 建議 | 傳輸編碼：設為 `"identity"`。 |

---

## 五、連線驗證與測試

1. **檢查 n8n 與 ngrok 狀態**：
   - 確認本機或遠端的 n8n 已啟動，且 MCP Server 功能已啟用。
   - 確認 ngrok 反向代理正在運行，且 URL 與設定檔完全相符。
2. **開啟新對話（New Chat）**：
   - 修改 `.agents/mcp_config.json` 後，請在 Antigravity 中開啟**新對話**，讓 Agent 重新載入專案等級的 MCP 工具。
3. **向 Agent 發送測試提問**：
   - 輸入：`「請列出目前 n8n 上的所有工作流」` 或 `「我有 n8n mcp server 嗎？」`
   - 若能順利列出 n8n 上的工作流程名稱與狀態，即代表連線與 Bearer 認證成功！

---

## 六、常見問題排查（Troubleshooting）

### Q1: 為什麼看到 `401 Unauthorized` 錯誤？
- **原因**：
  1. 未設定 `headers.Authorization`。
  2. Token 前方缺少 `Bearer ` 前綴。
  3. Access Token 已過期或無效。
  4. 誤以為 Antigravity 會跳出瀏覽器進行 OAuth 登入（**Antigravity 不支援 OAuth 互動流程，請務必手動填寫 Bearer Token**）。

### Q2: 出現連線失敗（Connection Refused / 404 / 502）？
- **原因**：
  1. ngrok 重啟後網址改變，但設定檔未同步更新。
  2. `serverUrl` 結尾遺漏了 `/mcp-server/http`。

### Q3: Agent 回應「未找到 n8n 工具」？
- **原因**：設定檔剛建立或變更，當前對話 Session 尚未更新工具清單。
- **解法**：在 Antigravity 開啟新對話（New Chat）即可自動讀取。

---

## 七、安全性注意事項

> [!CAUTION]
> - `.agents/mcp_config.json` 包含敏感的 `Bearer Token`，若專案為公開公開庫（Public Repository），**請勿直接將含真實 Token 的設定檔 commit 上傳至 GitHub**。
> - 建議可將敏感設定檔加入 `.gitignore`，或提供範本檔（如 `mcp_config.json.example`）供使用者複製填寫。
