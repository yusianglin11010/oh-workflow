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

## Resources

https://github.com/yusianglin11010/oh-workflow

---

## 議程

|     | 主題                      | 時間   |
| --- | ------------------------- | ------ |
| 1   | 環境配置與核心簡介        | 15 min |
| 2   | 技術選型與架構            | 10 min |
| 3   | OpenHands 詳解            | 10 min |
| 4   | 探索 open source codebase | 10 min |
| 5   | Use Case、限制與踩坑      | 10 min |
| 6   | Q&A                       | 5 min+ |

---

<!-- .slide: data-background="#1a1a2e" -->

# 第一部分：使用情境與配置

---

## 目前的痛點

**這些例行事務經常打斷開發者的專注流程：**

- Review MR
- 前期探索 bug root cause
- 熟悉不熟的 codebase

---

- 引入平行 AI agents 多工處理...?
  -> 如何減少 human context switch 開銷

---

## 另一個痛點：平行開發的資源消耗

**想平行跑 agent，但有代價：**

- 需求：要跑 parallel agent
  - 多開 VSCode IDE —— 記憶體很快就爆掉
  - 但其實我們不需要所有工作都開 IDE

---

- **terminal 分割視窗 + web chat** 的組合(tmux 等等)
  - 手動 copy-paste context、切換視窗，效率有限
  - terminal 膨脹過多，context switch 成本提高

---

## 如果 web chat 本身就能接 sandbox？

**—— 這就是 OpenHands 提供的方案**

- Web UI 直接綁一個隔離的 sandbox，不需要本機開 IDE
- 多個 conversation = 多個 sandbox，平行跑也不吃 local 資源
  - 可以把 review, bug fix 資源外包到 vm
- 省下 IDE 的 overhead，又保留互動式操作的體驗

---

## 舉個例子：BizForm 開發現況

**Mattermost `create-issue` 指令：**

- 能從討論串建立 GitLab issue
- 但它**只負責建立** issue
- 後續追蹤仍然靠人工處理

---

**Create Issue 示範**

- ![mattermost-create-issue](https://hackmd.io/_uploads/SylgafChbx.png)

---

### 現在流程

- User -> Prompt AI check issue(skills or CLI) -> Implement

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

- [chat sample](https://claude.ai/chat/41508b62-4e3f-4c8a-9edc-44a2c549f16e)

---

### CLI Agent

**採取行動來達成目標**

```
Input → LLM → Action → Observe → LLM → Action → … → Done
```

_例：「修好這個 bug 然後送 PR」_

---

![local-dev-terminal](https://hackmd.io/_uploads/SybbaMA2-g.png)

---

### 關鍵比較

| 面向     | Chatbot        | Agent        |
| -------- | -------------- | ------------ |
| 互動方式 | 單次回應       | 多輪迭代     |
| 執行動作 | 無（只有文字） | 使用工具執行 |
| 自主性   | 無             | 目標導向     |
| 適用場景 | 問答、解釋     | 完成任務     |

---

### 所以我們需要 Agent

**自動化「處理 issue」不是產生文字，而是要完成一連串動作：**

- 讀 issue → 讀 code → 改 code → 跑 test → 開 MR
- 每一步都依賴上一步的結果
- Chatbot 做不到這種「觀察 → 行動 → 再觀察」的迴圈

---

## 如果可以...

> _「如果我們被動讓 AI 偵測 Issue 並直接處理 Issue 呢」_

---

### OpenHands 提供整合了兩者特性的 solution

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

### What about runtime：Sandbox

**為什麼不是 Worktree？**

- BizForm 的 repo **大量使用 submodule**
- Worktree 對 submodule 支援有侷限（容易踩到邊界情況）
- Sandbox 直接 clone 完整 repo，行為與開發者本機一致

_（資料來源：`[TODO: 補 worktree + submodule 限制的文件連結]`）_

---

### 核心觀念

Agent 需要**自主性**和**工具使用能力**

Sandbox 提供**安全的執行環境**

---

## 最終配置

![local-dev-env](https://hackmd.io/_uploads/rJAWafR3Wg.png)

---

<!-- .slide: data-background="#16213e" -->

# 第二部分：技術選型與架構

---

## 工作流架構

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

### 拆解

1. **Webhook** 接收事件（MR、issue、tag）
   - 我們需要一個發送端
2. **觸發** AI agent 並帶入上下文
   - 我們需要一個接收端
3. **接收** 通知與結果

---

## 技術選型

---

### 事件來源：我們選 Webhook + Label 觸發

**以 issue 為核心的設計：**

- 讓開發者與 PM 專注在 **issue 規格**
- 貼上特定 label（如 `agent-trigger`）才觸發
- 利用 GitLab webhook 的 `changes` 欄位判斷 label 是否剛被加上

**為什麼不用 cron？**

- Issue-driven 比 schedule-driven 更貼近實際需求
- 避免浪費 LLM 額度掃描無變化的 board
- 讓開發人員更 `主動` 決定觸發時間

---

### 其他候選（未採用）

- **Slack / Mattermost 指令**：可銜接現有 `create-issue`，但需要 issue one shot 就到位，容易出錯
- **Cron 排程**：未來若要做 issue board 定期掃描可再加

---

### 工作流調度器

| 工具           | 優點                 | 缺點                  |
| -------------- | -------------------- | --------------------- |
| **n8n**        | 視覺化、可自建、免費 | 有學習曲線            |
| Temporal       | 企業級、強一致性     | 設定複雜、需寫程式    |
| GitHub Actions | 原生整合 GitHub      | 我們用 GitLab，不適用 |
| 自寫腳本       | 完全掌控             | 維護與擴充成本高      |

---

### 為什麼排除其他？

- **Temporal**：為高可靠性分散式系統設計，對 webhook → API 觸發這種輕量流程過重
- **GitHub Actions**：公司走 GitLab
- **自寫腳本**：之後要串 Mattermost / Slack 通知、加條件判斷，每個都要自己寫

---

### 最重要的一點！！

- BizForm 本來就有跑好的 n8n 可以用
- 已經整合在多個需求
  - standup 通知
  - gitlab pipelines status notifications 等等

---

### AI Agent 框架

| 工具          | Sandbox | Agent Harness                 | 可 one-shot 完整 workflow | 授權   |
| ------------- | ------- | ----------------------------- | ------------------------- | ------ |
| **OpenHands** | Docker  | 有                            | 可                        | MIT    |
| Daytona       | 有      | 無                            | 需自己串                  | AGPL   |
| SWE-agent     | 有限    | 有（偏 SWE bench）            | 偏研究用途                | MIT    |
| Aider         | 無      | 有（偏 CLI pair programming） | 需人在旁邊                | Apache |

---

### 為什麼選 OpenHands？關鍵是 Sandbox + Agent 一起

**我們要的 workflow：**

```
讀 issue → 讀 code → 改 code → compile → 跑 test → 開 MR
```

- **Daytona** 只提供 sandbox，agent 邏輯要自己串 LLM + 工具迴圈
- **SWE-agent** 專注在 SWE-bench 類 benchmark 場景，不適合日常 workflow
- **Aider** 是 CLI pair programming 工具，需要開發者在旁邊互動
- **OpenHands** 把 sandbox + agent harness 綁在一起，一次 API 呼叫就能跑完整條 pipeline

---

## 我們選的組合：n8n + OpenHands

---

### 為什麼選 n8n？

**為什麼是它：**

1. **公司既有內網 n8n server** — 直接可串內網 GitLab，無額外部署成本
2. **豐富的節點生態** — 通知可彈性串接 Mattermost / Slack / Discord / Telegram
3. 視覺化編輯器讓非工程角色也能看懂 / 調整 workflow

---

### 為什麼選 OpenHands？

**決定性理由：Sandbox + Agent Harness 綁在一起**

- 一次 API 呼叫即可完成 `review → compile → test → MR` 整條 workflow
- 不用自己寫 agent 迴圈、工具定義、prompt 調度
- REST API 設計為自動化整合而生（streaming 狀態更新）
- MIT 授權、社群活躍、映像檔可自訂

---

### OpenHands

- Web chat 介面讓 agent first shot 結束之後，開發者可以沿用既有 context

---

### 架構總覽

- ![openhands-workflow](https://hackmd.io/_uploads/S1820MC3Zl.png)

---

<!-- .slide: data-background="#1a1a2e" -->

# 第三部分：OpenHands 詳解

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

| 模式         | 用途                                |
| ------------ | ----------------------------------- |
| **GUI**      | 互動式操作（後續 follow up 進度）   |
| **CLI**      | 本機使用、手動觸發                  |
| **REST API** | 自動化整合（我們觸發 agent 的方式） |

---

## OpenHands 內部運作原理（可能略過）

**為什麼要看這個迴圈？**

- Debug 失敗任務時，知道卡在哪一步
- 估計 token 消耗與執行時間
- 理解為什麼 agent 有時會「繞路」

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
SANDBOX_STARTUP_GRACE_SECONDS: 120

# LLM 設定
LLM_MODEL: anthropic/claude-3-5-sonnet
LLM_API_KEY: sk-xxx
```

---

### 可自訂的項目

- Base container image（執行環境）
- LLM provider 和 model
- Agent 類型和行為
- Timeout 和資源限制

---

## 公司運算資源申請

- https://mis.gss.com.tw/ -> 申請GPU運算模型

---

## 公司 AI model 串接

- [ ] openhands-llm-setup
- https://mlpub.gss.com.tw/v1

## 自訂映像檔

---

### 為什麼需要自訂？

- 預設映像檔以 Python 為主
- 需要特定 runtime（.NET、Node.js、Go）or tools(net tool, db cli...etc)
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

- 詳見 repo：https://github.com/yusianglin11010/oh-workflow
  - `docker-compose.yaml`
  - `.env.example`
  - `oh/Dockerfile.sandbox-dotnet`
  - `n8n/workflow.json`

---

### 設定檔

- [ ] 待補

---

<!-- .slide: data-background="#16213e" -->

# 第四部分：探索 open source codebase

**以 OpenHands 為範例，動手建一個自動化 workflow**

---

### 為什麼用 OpenHands 當範例？

- 它本身就是個典型的 open source codebase（Python、FastAPI、React、Docker 多 runtime）
- 文件分散、issue 多、PR 量大——正是 agent 能幫上忙的場景
- 同時也是我們這次要整合的工具，順便熟悉它的內部

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
- 已建置好的 kitchen sink image（.NET）
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

**結論：自動化可以用 REST API**

---

<!-- .slide: data-background="#0d1117" -->

## Step C：n8n 串接 HTTP Request

---

### C.1 新增 HTTP Request 節點

**在 n8n 裡呼叫 OpenHands API**

- 新增「HTTP Request」node
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

### D.1 事件過濾：兩層 IF

**只在「issue 剛被貼上 `agent-trigger`」時觸發**

- **Check Issue Webhook**：`body.event_type == "issue"`
- **Check If Trigger Tag**：
  - `prev_labels` 不含 `agent-trigger`
  - `current_labels` 新增了 `agent-trigger`
- 比對 label 差異，避免同一個 issue 重複觸發

- [ ] 放兩個 IF 節點設定截圖

---

### D.2 組 Prompt 與啟動 Agent

**Set Variables + HTTP Request（OpenHands）**

- 組出給 agent 的指令，包含：
  - 讀取 issue URL、開 branch、實作、開 MR
  - 要求最後一行輸出 `SUMMARY: <摘要>, MR: <連結>`
- 記錄 `start_time` 用來計算耗時
- POST `/api/v1/app-conversations/stream-start`
- 用 **Limit** 節點只保留 streaming 最後一筆
- **Set Conversation Id**：擷取 `app_conversation_id`

- [ ] 放 Set Variables + HTTP Request 節點截圖

---

### D.3 先送一則「Task Started」通知

**讓使用者立刻知道 agent 已啟動**

- 分支到 **Send Task Started**（HTTP Request → Mattermost）
- 訊息附上 conversation URL，方便點進去看 agent 即時動作
- 主線同時進入輪詢狀態

- [ ] 放 Task Started 通知截圖

---

### D.4 Polling 直到 Agent 完成

**Get Status → If Task Finished → Wait 循環**

- **Get Status**：GET `/api/v1/app-conversations?ids={{conversation_id}}`
- **If Task Finished**：`execution_status == "finished"`
  - false → **Wait 10 秒** → 回到 Get Status
  - true → 進入結果處理階段
- **Set End Time**：計算 `elapsed_time`（秒）

- [ ] 放 polling 子流程截圖

---

### D.5 擷取 Summary 並通知 Mattermost

**Fetch Events → Extract Summary → Format Message → Send Notification**

- **Fetch Events**：GET `/api/v1/conversation/{id}/events/search?limit=100`
- **Extract Summary**（Code 節點）：
  - 過濾 `source == 'agent'` 且 `role == 'assistant'` 的最後一則訊息
  - 用 regex 抓 `SUMMARY:` 那一行
- **Format Message**：組出 Markdown（摘要 + 耗時 + conversation 連結 + 完整內容）
- **Send Notification**：POST 到 Mattermost `chat.gss.tw/api/v4/posts`

- [ ] 放結果通知節點截圖

---

### D.6 完整工作流圖

```
Webhook ─► Check Issue ─► Set Vars ─► Check Tag ─► OpenHands API
                                                        │
                                              Limit ◄───┘
                                                │
                                      Set Conversation Id
                                       ├─► Send Task Started (Mattermost)
                                       └─► Get Status ◄──────┐
                                               │             │
                                        If Task Finished ────┤
                                          │ (false)          │
                                          ▼                  │
                                         Wait 10s ───────────┘
                                          │ (true)
                                          ▼
                              Set End Time ─► Fetch Events
                                               │
                                       Extract Summary
                                               │
                                       Format Message ─► Send Notification (Mattermost)
```

- [ ] 放最終 workflow 總覽截圖

---

<!-- .slide: data-background="#0d1117" -->

## Step E：Workflow Demo

---

### E.1 Workflow 總覽

**直接在 n8n 打開 workflow 走一遍**

- 逐個介紹節點：Webhook、IF、Set、HTTP Request、Code、Wait
- 說明資料如何在節點之間流動
- 點開幾個關鍵節點看設定

- [ ] 放 n8n workflow 畫布截圖

---

### E.2 觸發並觀察執行

**Issue 加上 `agent-trigger` label，看 workflow 跑起來**

- 節點依序被點亮
- Polling loop 反覆執行 Get Status / Wait
- 最後送出 Mattermost 通知

- [ ] 放 n8n 執行中截圖

---

## 原始碼 Walkthrough

**帶走這些設定檔，自己跑看看：**

https://github.com/yusianglin11010/oh-workflow

- `docker-compose.yaml`
- `.env.example`
- `oh/Dockerfile.sandbox-dotnet`
- `n8n/workflow.json`

---

<!-- .slide: data-background="#1a1a2e" -->

# 第五部分：Use Case、限制與踩坑

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

## 限制與踩坑

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
**缺點：** 吃資源（多個 openhands server）、路由複雜

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

### 我們的選擇：Kitchen Sink

**原因：**

- 沿用前面 [`Dockerfile.sandbox-dotnet`](slides/slides.md#L466) 的做法
- 透過 docker layer 疊上各種 runtime（.NET、Node.js…）
- **必須以 OpenHands agent image 為 base**，OpenHands web server 才能調用 agent server
- 多 instance 方案需要額外維護 nginx 路由——目前需求未到，先不投入

---

## Dockerfile

```
# Stage 1: Grab .NET SDK
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

- 一定要有 `FROM ghcr.io/openhands/agent-server:1.12.0-python`
- openhands server 需要跟 agent container 溝通

---

## RBAC 的限制

---

### 社群版的問題

- 沒有使用者/角色管理 -> 只能本地自己用，不然其他 user 會共用 gitlab 認證以及 apikey
- 團隊共用可能沒什麼問題...?

---

### 影響

- 只要有存取權就能用任何 LLM key
- 無法限制到特定專案
- 不適合多租戶環境

---

### 暫時解法

- 網路層級的存取控制
- 分開部署不同團隊的 instance
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

<!-- .slide: data-background="#16213e" -->

# 第六部分：Q&A

---

## 資源

- **OpenHands**: https://github.com/All-Hands-AI/OpenHands
- **n8n**: https://n8n.io
- **slides and setups**: https://github.com/yusianglin11010/oh-workflow

---
