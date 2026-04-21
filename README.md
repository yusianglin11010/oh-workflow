# oh-workflow

[OpenHands](https://github.com/All-Hands-AI/OpenHands) + [n8n](https://n8n.io) AI Agent 自動化工作流：由 GitLab webhook 觸發，在隔離 sandbox 中自動完成 Code Review、Bug Fix、Issue 處理。

> 2026/04/21 技術分享 demo。投影片：[slides/slides.md](slides/slides.md)

---

## Quick Start

### 1. Build custom sandbox image(optional)

- Just change it if needed

```bash
docker build -f oh/Dockerfile.sandbox-dotnet -t openhands-dotnet:latest oh/
```

### 2. 準備環境變數

```bash
cp .env.example .env
```

編輯 `.env` 填入所需設定。

### 3. 啟動服務

```bash
docker-compose up -d
```

- OpenHands Web UI：<http://localhost:3000>
- n8n Web UI：<http://localhost:5678>

---

## 目錄結構

| 路徑                                                         | 說明                                    |
| ------------------------------------------------------------ | --------------------------------------- |
| [docker-compose.yaml](docker-compose.yaml)                   | OpenHands + n8n 的主要 compose 檔       |
| [.env.example](.env.example)                                 | 環境變數範本                            |
| [oh/Dockerfile.sandbox-dotnet](oh/Dockerfile.sandbox-dotnet) | 自訂 sandbox 映像檔（以 .NET 8.0 為例） |
| [n8n/workflow.json](n8n/workflow.json)                       | n8n workflow 匯出檔，可直接 import      |
| [CLIProxyApi/](CLIProxyApi/)                                 | 選用：訂閱方案轉接 proxy                |
| [slides/slides.md](slides/slides.md)                         | 技術分享投影片                          |

---

## Import n8n Workflow

1. 開啟 n8n Web UI
2. 右上角 `Import from File` → 選擇 [n8n/workflow.json](n8n/workflow.json)
3. 依需要調整節點參數：
   - **Webhook** 節點：設定 path
   - **HTTP Request (OpenHands)**：改成自己的 OpenHands URL 與目標 repo
   - **Send Notification (Mattermost)**：填入 Mattermost token 與 channel id

---

## 設定 GitLab Webhook

在 GitLab project → **Settings → Webhooks**：

- URL：`https://your-n8n-domain.example.com/webhook/oh`（對應 n8n webhook path）
- Trigger：勾選 **Issues events**（或 **Merge request events**）
- 測試：建一個 issue，貼上 `agent-trigger` label，n8n 與 OpenHands 應會依序啟動

---

## CLIProxyAPI(optional)

如果你只有 Claude Pro / Gemini 訂閱、沒有獨立 API key，可以用 [CLIProxyApi/](CLIProxyApi/) 目錄裡的設定把 **OAuth 登入**轉成相容 API endpoint。

```bash
cd CLIProxyApi
docker compose up -d

# 以 Claude 為例，OAuth 登入一次
docker exec -it cli-proxy-api ./CLIProxyAPI -claude-login
```

然後在 OpenHands 的設定把 LLM 指向 proxy：

```yaml
LLM_API_KEY: your-proxy-key # 對應 CLIProxyApi/config.yaml 的 api-keys
LLM_BASE_URL: http://host.docker.internal:8317
```

上游 repo：<https://github.com/router-for-me/CLIProxyAPI>

---

## 相關資源

- OpenHands：<https://github.com/All-Hands-AI/OpenHands>
- n8n：<https://n8n.io>
- CLIProxyAPI：<https://github.com/router-for-me/CLIProxyAPI>
- 投影片：[slides/slides.md](slides/slides.md)
