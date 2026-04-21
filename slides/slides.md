---
type: slide
theme: black
transition: slide
slideOptions:
  spotlight:
    enabled: true
  allottedMinutes: 60
---

## 用 OpenHands 打造 AI Agent 自動化工作流

**自動化 Code Review、Bug fix，以及更多**

---

Devin Lin | 2026 年 4 月 21 日

---

## 自我介紹

**Devin Lin** | BizForm 產品團隊 — 後端工程師

- 簡報連結 https://github.com/yusianglin11010/oh-workflow

---

## 議程

|     |                           |
| --- | ------------------------- |
| 1   | 背景與工具演進            |
| 2   | 環境配置與核心簡介        |
| 3   | 技術選型與架構            |
| 4   | OpenHands 詳解            |
| 5   | 探索 open source codebase |
| 6   | Use Case、限制與踩坑      |
| 7   | Q&A                       |

---

<!-- .slide: data-background="#1a1a2e" -->

## 第一部分：背景與工具演進

---

## BizForm 團隊

**BizForm 是 SaaS 表單系統**

- 多租戶架構
- active 線上數百租戶

日常工作：

- 需求開發，可能引用 open source
- 線上 bug 即時處理
- 既有 codebase 重構優化
- Code review

---

## 日常的例行事務

**這些事經常打斷開發者的專注流程：**

- 開發到一半被 bug 插單
  - 需要組合 log 以及用戶回報的資訊
- Review MR
  - 需要切換到另一個開發意圖
- -> 開發時間碎片化

---

### Web chat with AI

- 大概很久一年以前......?
- 用 web chat，手動複製貼上程式碼片段
- AI 沒辦法知道上下文，需要很長的複製貼上，浪費 context window

---

### VSCode Extensions：讀得到 codebase 了

- 一年前左右
- 整合成 VSCode Extensions 之後，可以同步翻 code
- AI 可以善用 CLI read, write, grep 等讀寫搜尋工具，拿到更完整的 context
- **多開效率太差**

---

### CLI Agent：平行但吃資源

- 開不同分支可以平行使用 agent
- 操作體驗有限

---

### 四種方式比較

| 方式              | 優點                                               | 痛點                       |
| ----------------- | -------------------------------------------------- | -------------------------- |
| Web chat          | web tab 就可以處理，UI 友善                        | 複製貼上太麻煩，沒有上下文 |
| VSCode Extensions | 可以讀 codebase，維持開發習慣                      | 多開消耗太多資源           |
| CLI               | 可以讀 codebase，資源消耗少                        | 操作體驗不友善             |
| **???**           | 可以讀 codebase＋對話介面＋獨立執行環境＋外包到 VM | —                          |

---

### 我思考的問題

> 如果有工具可以幫我外包 runtime？

---

### 那現在有什麼選擇？

| 工具          | Sandbox | Agent | one-shot workflow | 授權 |
| ------------- | ------- | ----- | ----------------- | ---- |
| **OpenHands** | Docker  | 有    | 可                | MIT  |
| Daytona       | 有      | 無    | 需自己串          | AGPL |

---

### 其他方案的問題

- **Daytona** 只提供 sandbox，agent 邏輯要自己串 LLM + 工具迴圈

---

### Openhands + Daytona...?

https://openhands.daytona.io/

![image](https://hackmd.io/_uploads/ryznEDQ6-x.png)

---

### OpenHands：sandbox + chat + 多開 + 獨立執行環境

- 可以讀 codebase，同時又提供對話介面
- 又有獨立執行環境，又可以把環境外包在 VM
- 多個 conversation = 多個 sandbox，平行跑也不吃 local 資源
  - 可以把 review, bug fix 資源外包到 VM
- 省下 IDE 的 overhead，又保留互動式操作的體驗
- Web chat 介面讓 agent first shot 結束之後，開發者可以沿用既有 context

---

### 本機開發環境

![local-dev-env](https://hackmd.io/_uploads/rkbjsd7abg.png)

---

### UI 概覽

---

### Overview

![openhands-overview](https://hackmd.io/_uploads/SyTumOmTbg.png)

---

### 聊天介面

- ![openhands-chat-terminal](https://hackmd.io/_uploads/BkJqXuXaWx.png)

---

### Edited code

- ![openhands-code-change](https://hackmd.io/_uploads/HyP97OQpbl.png)

---

## 可設定的部分

### LLM

- ![openhands-llm](https://hackmd.io/_uploads/B17iQd7p-x.png)

---

### Repository

- ![openhands-gitlab](https://hackmd.io/_uploads/H1sjQ_76Zg.png)

---

### git config

- ![openhands-git-user](https://hackmd.io/_uploads/Skfa7_7T-l.png)

---

### OpenHands Demo 時間～

#### 來使用 openhands 閱讀 codebase 看看

- [ref](http://192.168.31.51:3000/conversations/975c16e9bc6945aab267ae940f1b38d2)

---

## 回到 BizForm 開發現況

日常開發流程

![openhands-without-webhook-workflow](https://hackmd.io/_uploads/ryh8quX6Zl.png)

- 部分流程自動化

---

**Mattermost `create-issue` 指令：**

- 能從討論串建立 GitLab issue
- 但它**只負責建立** issue
- 後續追蹤仍然靠人工處理

---

**Create Issue 示範**

- ![mattermost-create-issue](https://hackmd.io/_uploads/SylgafChbx.png)

---

### 現在的問題：如果可以自動偵測 Issue？

- 也許我們可以不需要主動 prompt AI
  > _「如果我們被動讓 AI 偵測 Issue 並直接處理 Issue 呢」_

---

### 我們需要讓 Agent 自動觸發

Agent 可以自動化讀 issue 並直接發 MR

- 讀 issue → 讀 code → 改 code → 跑 test → 開 MR
- 減少在本機環境切換環境、手動 prompt

---

** Issue 品質很重要 **

---

<!-- .slide: data-background="#16213e" -->

# 第三部分：自動觸發 Agent

---

## 工作流架構

```
┌─────────────────────────────────────────────────┐
│        通用 AI Agent 工作流                       │
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

### 拆解

1. **Webhook** 接收事件（MR、issue、tag）
   - 我們需要一個發送端
2. **觸發** AI agent 並帶入上下文
   - 我們需要一個接收端
3. **接收** 通知與結果

---

## 技術選型

---

### 事件來源：Webhook + Label 觸發

**以 issue 為核心的設計：**

- 讓開發者與 PM 專注在 **issue 規格**
- 貼上特定 label（如 `agent-trigger`）才觸發

---

**考慮定期 scan？**

- Issue-driven 比 schedule-driven 更貼近實際需求
- 避免浪費 LLM 額度掃描無變化的 board
- 讓開發人員更 `主動` 決定觸發時間

---

### 我選了 n8n!

原因是...

- BizForm 本來就有內網跑好的 n8n 可以用 XD
- 已經整合在多個需求
  - stand-up 通知
  - gitlab pipelines status notifications 等等

---

### 更多優點

- **豐富的節點生態** — 通知可彈性串接 Mattermost / Slack / Discord / Telegram
- 視覺化編輯器托拉操作很方便

---

### 架構總覽

![openhands-with-webhook](https://hackmd.io/_uploads/S1IoJjmp-g.png)

---

### 接著來看看 n8n 怎麼做

---

## 最終結果總覽

- [example](http://192.168.102.4:3000/conversations/1d31f91f075a4a9e8ca5e25ceddf8a2d)
- [n8n](http://192.168.102.4/workflow/EOWPBcfwCj7XEk0W/executions/74956)

**接下來將實作以下：**

1. **Step A**：在 n8n 接收 webhook
2. **Step B**：透過 REST API 觸發 OpenHands
3. **Step C**：n8n 串接通知

---

## Demo 前置準備

**需要準備的東西：**

- Docker
- 我們將用 docker 跑 n8n + openhands

---

### 1. Build custom sandbox image(optional)

```bash
docker build -f oh/Dockerfile.sandbox-dotnet -t openhands-dotnet:latest oh/
```

---

### 2. 準備環境變數

```bash
cp .env.example .env
```

編輯 `.env` 填入所需設定。

---

### 3. 啟動服務

```bash
docker-compose up -d
```

- OpenHands Web UI：<http://localhost:3000>
- n8n Web UI：<http://localhost:5678>

---

### sandbox image 調整

```yaml
# Sandbox env 設定
AGENT_SERVER_IMAGE_REPOSITORY=your-custom-image
AGENT_SERVER_IMAGE_TAG=latest
```

---

### 為什麼需要自訂？

- 預設映像檔以 Python 為主
- 需要特定 runtime（.NET、Node.js、Go）or tools(net tool, db cli...etc)
- 專案特有的相依性

---

### 範例：.NET Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS dotnet-sdk

# Stage 2: Your actual image
FROM ghcr.io/openhands/agent-server:1.12.0-python

USER root

# Copy .NET SDK from stage 1
COPY --from=dotnet-sdk /usr/share/dotnet /usr/local/dotnet

# Install runtime dependencies only
RUN apt-get update && apt-get install -y libicu-dev \
    && rm -rf /var/lib/apt/lists/*

ENV DOTNET_ROOT=/usr/local/dotnet
ENV PATH=$PATH:/usr/local/dotnet
```

---

<!-- .slide: data-background="#1a1a2e" -->

# 第六部分：Use Case、限制與踩坑

---

## 三個實際 Use Case

**用既有 conversation 帶過，看真實任務跑起來的樣子**

---

### Use Case 1：自動化程式碼審查

**情境：** MR 開立後，agent 自動讀 diff、留評論

- 觸發：MR 加上 `agent-trigger` label
- Agent 行為：clone repo → 讀 diff → 跑 lint / test → 在 MR 留評論
- 適用：例行性 review、檢查命名規範、找明顯 bug

- [ ] 放既有 conversation 截圖

---

### Use Case 2：Bug 修正

**情境：** Issue 描述 bug，agent 嘗試定位並提 MR

- 觸發：Issue 加上 `agent-trigger` label
- Agent 行為：讀 issue → 找相關 code → 寫修復 → 跑 test → 開 MR
- 適用：規格清楚的小 bug、有對應 test case 的問題

- [ ] 放既有 conversation 截圖

---

### Use Case 3：探索陌生 codebase

**情境：** 接手新專案 / 評估 open source library

- 觸發：手動或 issue label
- Agent 行為：讀 README → 跑起專案 → 回報架構摘要與關鍵檔案
- 適用：第一次接觸大型 repo、評估技術債

- [ ] 放既有 conversation 截圖

---

## Openhands 限制

---

## 只能設定一個 runtime image

---

### 問題

```yaml
# 全域只能設定一個映像檔
AGENT_SERVER_IMAGE_REPOSITORY=openhands-dotnet
AGENT_SERVER_IMAGE_TAG=latest
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
**缺點：** 吃資源（多個 openhands server）、路由複雜

---

### 方案二：all-in-one Image

```dockerfile
FROM openhands/agent-server:latest
RUN install dotnet nodejs python go rust ...
# 映像檔大小：3-5GB
```

**優點：** 簡單、只需一個 instance
**缺點：** 映像檔大、build 慢

---

### 我們的選擇：Kitchen Sink

**原因：**

- 沿用前面 [`Dockerfile.sandbox-dotnet`](slides/slides.md#L466) 的做法
- 透過 docker layer 疊上各種 runtime（.NET、Node.js…）
- **必須以 OpenHands agent image 為 base**，OpenHands web server 才能調用 agent server
- 多 instance 方案需要額外維護 nginx 路由——目前需求未到，先不投入

---

## Open source 版本的限制

---

### 社群版的問題

- 沒有使用者/角色管理 -> 只能本地自己用，不然其他 user 會共用 gitlab 認證以及 apikey
- 團隊共用可能沒什麼問題...?

---

### 缺點

- 只要有存取權就能用任何 LLM key
- 無法限制到特定專案
- 不適合多租戶環境

---

### 暫時解法

- 網路層級的存取控制
- 分開部署不同團隊成員的 instance
  - 自己墊一層 nginx 做 multi openhands instances proxy
- 企業版（付費）

---

## 實戰踩坑

---

### 記得清理閒置的 Sandbox Runtime

- 每次觸發都會起一個 Docker container
- **不會自動回收**——堆積會吃掉大量 disk / memory
- 解法：定期 `docker container prune` 或在 workflow 結尾呼叫刪除 API

---

## Extra: 沒有 API Key？用 CLIProxyAPI

---

### 問題

OpenHands 需要 `LLM_API_KEY` 來呼叫 LLM provider。

但很多開發者只有**訂閱方案**（Claude Pro、Gemini 等）— 沒有獨立的 API key。

---

### 解法：CLIProxyAPI

一個自建的 proxy，把 **OAuth/CLI 登入**轉成**相容 API 的 endpoint**。

https://github.com/router-for-me/CLIProxyAPI

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
cd CLIProxyAPI && docker-compose up -d
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

<!-- .slide: data-background="#16213e" -->

# 第七部分：Q&A

---

## 資源

- **OpenHands**: https://github.com/All-Hands-AI/OpenHands
- **n8n**: https://n8n.io
- **slides and setups**: https://github.com/yusianglin11010/oh-workflow

---
