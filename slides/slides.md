---
type: slide
theme: black
transition: slide
slideOptions:
  spotlight:
    enabled: true
  allottedMinutes: 60
---

# 用 OpenHands 打造 AI Agent 自動化工作流

**自動化 Code Review、Bug fix，以及更多**

---

Devin Lin | 2026 年 4 月 21 日

---

## Source Code

- [ ] 待補 Source code link

---

## 議程

| 段落 | 主題                      | 時間   |
| ---- | ------------------------- | ------ |
| 1    | 環境配置與核心簡介        | 15 min |
| 2    | 自動化程式碼審查          | 10 min |
| 3    | 高效開發與 bug 修正       | 10 min |
| 4    | 探索 open source codebase | 10 min |
| 5    | Use Case & Demo           | 12 min |
| 6    | Q&A                       | 3 min+ |

---

<!-- .slide: data-background="#1a1a2e" -->

# 第一部分：環境配置與核心簡介

---

## 目前的痛點

**這些手動作業經常打斷開發者的專注流程：**

- 審查 MR (Merge Request)
- 調查 Bug
- 熟悉不熟的 codebase

---

## BizForm 現況

**Mattermost `create-issue` 指令：**

- 能從討論串建立 GitLab issue
- 但它**只負責建立** issue
- 後續追蹤仍然靠人工處理

---

**Create Issue 示範**

- [ ] 放截圖

---

### 現在流程

- User -> Prompt AI check issue(skills or CLI) -> Implement

---

## 如果可以...

> _「如果我們被動能直接讓 AI 偵測 Issue 並直接處理 Issue 呢」_

---

## Chatbot vs. Agent

---

### Web Chatbot

**回答問題、產生文字**

```
Input → LLM → Output（一次性回應）
```

_例：「幫我解釋這段 code」_

---

- [ ] 放 Web 聊天截圖

---

### CLI Agent

**採取行動來達成目標**

```
Input → LLM → Action → Observe → LLM → Action → … → Done
```

_例：「修好這個 bug 然後送 PR」_

---

- [ ] 放 CLI 聊天截圖

---

### 關鍵比較

| 面向     | Chatbot        | Agent        |
| -------- | -------------- | ------------ |
| 互動方式 | 單次回應       | 多輪迭代     |
| 執行動作 | 無（只有文字） | 使用工具執行 |
| 自主性   | 無             | 目標導向     |
| 適用場景 | 問答、解釋     | 完成任務     |

---

## Sandbox vs. Git Worktree

**兩種隔離方式：**

| 面向       | Sandbox (Docker) | Git Worktree |
| ---------- | ---------------- | ------------ |
| 隔離程度   | 完整 OS 層級     | 僅檔案系統   |
| Submodules | 正常運作         | 可能有問題   |
| 資源消耗   | 較高             | 較低         |
| 安全性     | 強               | 有限         |

---

### 核心觀念

Agent 需要**自主性**和**工具使用能力**

Sandbox 提供**安全的執行環境**

---

<!-- .slide: data-background="#16213e" -->

# 第二部分：自動化程式碼審查

---

## 通用化的工作流架構

**先理解模式，再看具體工具！**

```
┌─────────────────────────────────────────────────┐
│        通用 AI Agent 工作流                      │
│                                                 │
│  Event         Workflow        AI Agent         │
│  Source   →  Orchestrator  →  (Sandbox)         │
│  (webhook)       │              │               │
│                  │←─────────────┘               │
│                  ↓ Results                      │
│             Notification                        │
└─────────────────────────────────────────────────┘
```

---

### 模式拆解

1. **Webhook** 接收事件（MR、issue、tag）
   - 我們需要一個發送端
2. **觸發** AI agent 並帶入上下文
   - 我們需要一個接收端
3. **接收** 通知與結果

---

## 技術選型

---

### 事件來源

- GitLab / GitHub webhooks
- Slack / Mattermost 指令
  - 也許可以銜接現有的 `create-issue` 指令？
- 排程觸發 (cron)
  - 定時掃描 issue board
  - 善用 AI 訂閱額度...？

---

### 工作流調度器

| 工具           | 優點                 | 缺點        |
| -------------- | -------------------- | ----------- |
| **n8n**        | 視覺化、可自建、免費 | 有學習曲線  |
| Temporal       | 企業級               | 設定複雜    |
| GitHub Actions | 原生整合 GitHub      | 限定 GitHub |
| 自寫腳本       | 完全掌控             | 維護成本    |

---

### AI Agent 框架

| 工具          | Sandbox | 多 Runtime      | Agent | 授權   |
| ------------- | ------- | --------------- | ----- | ------ |
| **OpenHands** | Docker  | 單一映像檔      | 有    | MIT    |
| Daytona       | 有      | 每 sandbox 獨立 | 無    | AGPL   |
| SWE-agent     | 有限    | 無              | 有    | MIT    |
| Aider         | 無      | 不適用          | 有    | Apache |

---

## 我們選的組合：n8n + OpenHands

---

### 為什麼選 n8n？

- 視覺化的 workflow 編輯器
- 可自建（資料留在內部）
- 輕鬆處理 webhook
- 良好的 GitLab 整合

---

### 為什麼選 OpenHands？

- 開源 (MIT)
- 內建 sandbox
- 提供 REST API 可程式化呼叫
- 社群活躍

---

### 架構總覽

```
GitLab Event
     ↓
  Webhook
     ↓
    n8n
     ↓
OpenHands REST API
     ↓
  Sandbox
     ↓
  Results
     ↓
  GitLab
```

---

<!-- .slide: data-background="#1a1a2e" -->

# 第三部分：高效開發與 bug 修正

---

## 核心概念

---

### 元件

- **OpenHands App**：LLM 調度、計畫、工具分配
- **Agent Server**：在 sandbox 內執行的 runtime
- **Sandbox**：隔離的 Docker container
- **REST API**：程式化存取介面，適合自動化整合

---

### 使用模式

| 模式         | 用途                         |
| ------------ | ---------------------------- |
| **GUI**      | 互動式操作                   |
| **CLI**      | 本機使用、手動觸發           |
| **REST API** | 自動化整合（我們採用的方式） |

---

## OpenHands 內部運作原理

---

### Agent Loop

```python
while not done:
    # 1. 取得目前狀態
    state = get_observation()

    # 2. 向 LLM 詢問下一步動作
    action = agent.step(state)

    # 3. 在 sandbox 裡執行動作
    result = runtime.execute(action)

    # 4. 檢查是否完成
    done = is_finished(result)
```

---

### 原始碼參考

- `openhands/agenthub/codeact_agent/` — agent 實作
- `openhands/runtime/` — sandbox 執行
- Prompt template 和 tool 定義

_（有興趣的話可以一起深入看）_

---

## UI 概覽

_（展示 OpenHands 的 web 介面）_

- 聊天介面
- 檔案瀏覽器
- Terminal 檢視
- Agent 狀態 / 日誌

---

### 聊天介面

- [ ] 放截圖 + live demo

---

### Terminal 檢視

- [ ] 放截圖 + live demo

---

### 檔案瀏覽器

- [ ] 放截圖 + live demo

---

## 可設定的部分

---

### 主要環境變數

```yaml
# Sandbox 設定
SANDBOX_BASE_CONTAINER_IMAGE: openhands-custom:latest
SANDBOX_CONTAINER_URL_PATTERN: http://host:port/agentproxy/{port}
SANDBOX_STARTUP_GRACE_SECONDS: 120

# LLM 設定
LLM_MODEL: anthropic/claude-3-5-sonnet
LLM_API_KEY: sk-xxx

# Agent 設定
AGENT_NAME: CodeActAgent
```

---

### 可自訂的項目

- Base container image（執行環境）
- LLM provider 和 model
- Agent 類型和行為
- Timeout 和資源限制

---

## 沒有 API Key？用 CLIProxyAPI

---

### 問題

OpenHands 需要 `LLM_API_KEY` 來呼叫 LLM provider。

但很多開發者只有**訂閱方案**（Claude Pro、Gemini 等）— 沒有獨立的 API key。

---

### 解法：CLIProxyAPI

一個自建的 proxy，把 **OAuth/CLI 登入**轉成**相容 API 的 endpoint**。

```
┌──────────┐     ┌──────────────┐     ┌─────────────┐
│ OpenHands│────►│ CLIProxyAPI  │────►│ Claude/     │
│          │     │ (Docker)     │     │ Gemini/     │
│          │     │              │     │ Codex       │
└──────────┘     └──────────────┘     └─────────────┘
   LLM_API_KEY      OAuth 登入          訂閱方案
   + BASE_URL       （一次性）
```

---

### 設定方式

**1. 用 Docker Compose 啟動 CLIProxyAPI：**

```bash
cd CLIProxyAPI && docker compose up -d
```

**2. 登入一次（以 Claude 為例）：**

```bash
docker exec -it cli-proxy-api ./CLIProxyAPI -claude-login
```

**3. 把 OpenHands 指向 proxy：**

```yaml
LLM_API_KEY: your-proxy-key # CLIProxyAPI config 裡的 key
LLM_BASE_URL: http://host:8317 # CLIProxyAPI 位址
```

---

### 支援的 Provider

| Provider | 登入指令                      |
| -------- | ----------------------------- |
| Claude   | `./CLIProxyAPI -claude-login` |
| Codex    | `./CLIProxyAPI -codex-login`  |
| Gemini   | `./CLIProxyAPI -login`        |

_詳細設定請參考共享的 setup repo_

---

## 自訂映像檔

---

### 為什麼需要自訂？

- 預設映像檔以 Python 為主
- 需要特定 runtime（.NET、Node.js、Go）
- 專案特有的相依性

---

### 範例：.NET Dockerfile

```dockerfile
FROM ghcr.io/openhands/agent-server:1.12.0-python

USER root

# 安裝 .NET 8.0
RUN apt-get update && apt-get install -y curl libicu-dev \
    && curl -fsSL https://dot.net/v1/dotnet-install.sh \
       | bash -s -- --channel 8.0 --install-dir /usr/local/dotnet

ENV DOTNET_ROOT=/usr/local/dotnet
ENV PATH=$PATH:/usr/local/dotnet
```

---

### 關於網路

- 需要存取外部網路的 LLM API
- 存取公司 mlpub 需要 VPN
- 設定方式：讓 container 使用 host network

---

### 範例設定檔

- [ ] 提供 docker-compose.yml 範例
- [ ] 提供 nginx.conf 範例
- [ ] 提供 .env 範例

---

### 設定檔

- [ ] 待補

---

<!-- .slide: data-background="#16213e" -->

# 第四部分：探索 open source codebase

---

## Demo 總覽

**我們將一步步建構完整的自動化流程：**

1. **Step A**：在 n8n 接收 webhook
2. **Step B**：透過 REST API 觸發 OpenHands
3. **Step C**：n8n 串接 HTTP Request
4. **Step D**：完整的工作流整合
5. **Step E**：端對端 demo

---

## Demo 前置準備

**需要準備的東西：**

- OpenHands 啟動且可存取
- 已建置好的 kitchen sink image（.NET + Node.js）
- n8n 運行中
- 測試用的 GitLab 專案

---

<!-- .slide: data-background="#0d1117" -->

## Step A：在 n8n 接收 Webhook

---

### A.1 建立 Webhook 節點

**設定 n8n 接收 GitLab 事件**

- 建立新的 workflow
- 新增「Webhook」節點
- 設定 HTTP method (POST)
- 複製 webhook URL

- [ ] 放 n8n webhook 節點截圖

---

### A.2 設定 GitLab Webhook

**把 GitLab 接到 n8n**

- 到 GitLab project → Settings → Webhooks
- 貼上 n8n 的 webhook URL
- 選擇觸發事件（Issue events、Label events）
- 測試 webhook

- [ ] 放 GitLab webhook 設定截圖

---

### A.3 測試 Webhook 接收

**確認 n8n 有收到資料**

- 在 GitLab 觸發事件（替 issue 加標籤）
- 檢查 n8n 執行日誌
- 查看收到的 payload

- [ ] 放 n8n 執行日誌截圖

---

<!-- .slide: data-background="#0d1117" -->

## Step B：透過 REST API 觸發 OpenHands

---

### B.1 OpenHands REST API 基礎

**用 HTTP 呼叫來啟動 agent**

```bash
# 基本 REST API 呼叫
curl -X POST http://localhost:3000/api/v1/app-conversations/stream-start \
  -H "Content-Type: application/json" \
  -d '{
    "initial_message": {
      "role": "user",
      "content": [{"text": "Review this code for bugs"}],
      "run": true
    }
  }'
```

---

### B.2 帶入 Repository 上下文

**指定 Git provider 和 repo**

```bash
curl -X POST http://localhost:3000/api/v1/app-conversations/stream-start \
  -H "Content-Type: application/json" \
  -d '{
    "initial_message": {
      "role": "user",
      "content": [{"text": "Review the code and fix any bugs"}],
      "run": true
    },
    "git_provider": "gitlab",
    "selected_repository": "mygroup/myproject",
    "selected_branch": "main"
  }'
```

---

### B.3 API 回應格式

**Streaming JSON 狀態更新**

```json
{"id":"...","status":"WORKING","sandbox_id":null,...}
{"id":"...","status":"WAITING_FOR_SANDBOX",...}
{"id":"...","status":"PREPARING_REPOSITORY",...}
{"id":"...","status":"STARTING_CONVERSATION",...}
{"id":"...","status":"READY","app_conversation_id":"...",...}
```

---

### CLI vs REST API 比較

| 特性            | CLI (`openhands -t`) | REST API               |
| --------------- | -------------------- | ---------------------- |
| Sandbox 隔離    | 無（在 host 上執行） | 有（Docker container） |
| Repository 複製 | 手動                 | 自動                   |
| n8n 整合方式    | Shell command        | HTTP Request           |
| 隔離性          | 無                   | Docker container       |

**結論：自動化整合建議用 REST API**

---

<!-- .slide: data-background="#0d1117" -->

## Step C：n8n 串接 HTTP Request

---

### C.1 新增 HTTP Request 節點

**在 n8n 裡呼叫 OpenHands API**

- 新增「HTTP Request」節點
- 設定 Method: POST
- URL: `http://localhost:3000/api/v1/app-conversations/stream-start`
- 設定 JSON body

- [ ] 放 n8n HTTP Request 節點截圖

---

### C.2 動態組合 Request Body

**把 webhook 資料帶入 API 呼叫**

```json
{
  "initial_message": {
    "role": "user",
    "content": [
      {
        "text": "Review issue #{{ $json.object_attributes.iid }}: {{ $json.object_attributes.title }}"
      }
    ],
    "run": true
  },
  "git_provider": "gitlab",
  "selected_repository": "{{ $json.project.path_with_namespace }}",
  "selected_branch": "{{ $json.object_attributes.source_branch || 'main' }}"
}
```

- [ ] 放 n8n expression 範例截圖

---

### C.3 處理回應

**解析 API 回應結果**

- 擷取 `app_conversation_id`
- 處理 streaming JSON
- 錯誤處理

- [ ] 放 output 處理截圖

---

<!-- .slide: data-background="#0d1117" -->

## Step D：完整的工作流整合

---

### D.1 連接所有節點

**組合完整的 pipeline**

```
Webhook → IF（檢查標籤）→ HTTP Request（OpenHands API）→ HTTP Request（GitLab API）
```

- [ ] 放完整 workflow 截圖

---

### D.2 加入條件判斷

**依標籤或事件類型過濾**

- 新增「IF」節點
- 檢查標籤是否為 `ai-review`
- 只在條件成立時繼續

- [ ] 放 IF 節點設定截圖

---

### D.3 回寫結果到 GitLab

**閉合整個迴圈**

- 新增「HTTP Request」節點
- 設定 GitLab API endpoint
- 在 issue/MR 下方留言
- 附上 OpenHands 的分析結果

- [ ] 放 GitLab API 節點截圖

---

### D.4 完整工作流圖

```
┌──────────┐    ┌─────────┐    ┌─────────────┐    ┌──────────┐
│ Webhook  │───►│ IF Node │───►│ HTTP Request│───►│ HTTP Req │
│ (GitLab) │    │(label?) │    │ (OpenHands) │    │(GitLab)  │
└──────────┘    └─────────┘    └─────────────┘    └──────────┘
```

- [ ] 放最終 workflow 總覽截圖

---

<!-- .slide: data-background="#0d1117" -->

## Step E：端對端 Demo

---

### E.1 準備測試 Issue

**在 GitLab 建立 issue**

- 建立帶有描述的 issue
- 包含程式碼上下文或 repo 參考

- [ ] 放測試 issue 截圖

---

### E.2 觸發工作流

**加上觸發標籤**

- 替 issue 加上 `ai-review` 標籤
- 觀察 GitLab 發送 webhook

---

### E.3 觀察 n8n 執行

**即時觀看 workflow 運作**

- 看到 webhook 被接收
- 觀察 IF 節點判斷
- 看到 HTTP Request 送出

- [ ] 放 n8n 即時執行截圖

---

### E.4 觀察 OpenHands 工作

**Agent 實際運作中**

- 觀察 agent 在 sandbox 內啟動
- 即時查看 agent 的動作
- 任務完成

- [ ] 放 OpenHands 工作截圖

---

### E.5 結果回寫到 GitLab

**最終輸出**

- 留言出現在 issue 上
- 可以看到 agent 的分析/審查結果
- 工作流完成！

- [ ] 放 GitLab 留言結果截圖

---

## 原始碼 Walkthrough

**帶走這些設定檔，自己跑看看：**

- `docker-compose.yml`
- `nginx.conf`
- `Dockerfile.sandbox`
- n8n workflow 匯出檔 (JSON)

- [ ] 準備原始碼包
- [ ] 建立 GitHub/GitLab repo 分享設定檔

---

<!-- .slide: data-background="#1a1a2e" -->

# 第五部分：Use Case & Demo

---

## 只能用一個自訂映像檔

---

### 問題

```yaml
# 全域只能設定一個映像檔
SANDBOX_BASE_CONTAINER_IMAGE=openhands-dotnet:latest
```

**無法依專案動態切換 runtime！**

---

### 方案一：多個 Instance

```
┌──────────────────────────────────────────────┐
│              nginx（路由）                     │
│  /dotnet/* → OpenHands 1 (dotnet image)      │
│  /nodejs/* → OpenHands 2 (nodejs image)      │
│  /python/* → OpenHands 3 (python image)      │
└──────────────────────────────────────────────┘
```

**優點：** 真正的隔離、乾淨的映像檔
**缺點：** 吃資源、路由複雜

---

### 方案二：Kitchen Sink Image

```dockerfile
FROM openhands/agent-server:latest
RUN install dotnet nodejs python go rust ...
# 映像檔大小：3-5GB
```

**優點：** 簡單、只需一個 instance
**缺點：** 映像檔大、build 慢

---

### 社群狀態

- Issue #8845 自 2025 年 6 月開啟
- 尚無官方時程
- 社群持續關注

---

## RBAC 的限制

---

### 社群版的問題

- 沒有使用者/角色管理
- 沒有專案層級權限
- 沒有稽核日誌

---

### 影響

- 只要有存取權就能用任何 LLM key
- 無法限制到特定專案
- 不適合多租戶環境

---

### 暫時解法

- 網路層級的存取控制
- 分開部署不同團隊的 instance
- 企業版（付費）

---

## 其他注意事項

- **啟動時間**：sandbox 需要 30-60 秒
- **Token 上限**：大型 codebase 可能超出 context
- **費用**：每次執行 = 多次 LLM 呼叫
- **非確定性**：同樣的輸入可能得到不同結果

---

<!-- .slide: data-background="#16213e" -->

# 第六部分：Q&A

---

## 重點摘要

1. **Agent = LLM + 工具 + 迴圈** — 概念上其實很單純
2. **Webhook → 調度器 → Agent** — 這個模式可以重複使用
3. **OpenHands + n8n = 容易上手** — 不需要打造特殊基礎設施
4. **了解限制** — 單一映像檔、無 RBAC
5. **從小開始、持續迭代** — 先跑通一個 workflow

---

## 資源

- **OpenHands**: https://github.com/All-Hands-AI/OpenHands
- **n8n**: https://n8n.io
- **我的設定**: [repo link]
- **投影片**: [this link]

- [ ] 建立並分享設定 repo
- [ ] 上傳投影片

---

## 謝謝！

**有問題嗎？**

---

Devin Lin

---

<!-- .slide: data-background="#0d0d0d" -->

## TODO Checklist

**截圖待準備：**

- [ ] Create Issue Demo 截圖
- [ ] Web 聊天截圖
- [ ] CLI 聊天截圖
- [ ] OpenHands 聊天介面截圖
- [ ] OpenHands Terminal 檢視截圖
- [ ] OpenHands 檔案瀏覽器截圖
- [ ] docker-compose.yml 範例
- [ ] nginx.conf 範例
- [ ] .env 範例

**Demo Step A（Webhook）：**

- [ ] n8n webhook 節點截圖
- [ ] GitLab webhook 設定截圖
- [ ] n8n 執行日誌截圖

**Demo Step B（REST API）：**

- [ ] REST API 呼叫截圖
- [ ] 帶 repo 的 API 呼叫截圖
- [ ] API 回應範例

**Demo Step C（n8n HTTP Request）：**

- [ ] n8n HTTP Request 節點截圖
- [ ] n8n expression 範例截圖
- [ ] Output 處理截圖

**Demo Step D（整合）：**

- [ ] 完整 workflow 截圖
- [ ] IF 節點設定截圖
- [ ] GitLab API 節點截圖
- [ ] 最終 workflow 總覽截圖

**Demo Step E（端對端）：**

- [ ] 測試 issue 截圖
- [ ] n8n 即時執行截圖
- [ ] OpenHands 工作截圖
- [ ] GitLab 留言結果截圖

**資源：**

- [ ] 準備原始碼包
- [ ] 建立設定 repo
- [ ] 上傳投影片
