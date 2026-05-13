# LangGraph

*Evaluated: LangGraph 1.0.x (GA Oct 22 2025) and 1.2.0a streaming/timeout previews. As of May 2026.*

## 摘要

LangGraph 是一個低階的 Python/TypeScript runtime,用於將具狀態、長時間執行的 LLM agent 構建為有向圖。每個 node 都是針對一個具型別的 state 物件的函式;edge(靜態或條件式)驅動控制流;可插拔的 checkpointer 在每一個 superstep 持久化 state。它在 2025 年 10 月達到穩定的 1.0,與 0.x 後期相比沒有破壞性變更,而且它是唯一一個主流的 agent framework,其中**持久化執行(durable execution)與 human-in-the-loop interrupt 是核心原語而非附加功能** ——使得執行可以恢復的同一個 checkpoint,也讓它能在 graph 中途暫停以供審閱。它不是像 CrewAI 那種高階的「描述你的 crew」抽象,也不是像 OpenAI 的 Agents SDK 那種廠商 SDK;它更接近一個專門為 LLM 控制流設計的 workflow engine。當你需要對分支、重試、部分失敗以及人工審閱進行細緻控制時,選它——代價是學習曲線比基於角色的 framework 陡峭,而且會強烈地將你拉向 LangChain/LangSmith 生態系。

## 比較

| 維度 | **LangGraph 1.0** | **AutoGen 0.7** (Microsoft) | **OpenAI Agents SDK** (Apr 2026) | **CrewAI 1.10** |
|---|---|---|---|---|
| 類型 / 類別 | 用於具狀態 agent 的 graph runtime | 用於多 agent 對話的非同步 actor framework | 廠商錨定的 agent SDK + sandbox harness | 角色/流程編排 framework |
| 核心架構 | 在共享 state 物件之上,由具型別 node 組成的有向 `StateGraph`;checkpointer 在每個 superstep 持久化 | 三層 actor 模型(Core / AgentChat / Extensions),搭配非同步訊息傳遞 | `Agent` + `Runner` 在沙箱化的 harness 內驅動 LLM-tool 迴圈,具可設定的記憶體 | 由角色化 `Agent` 組成的 `Crew`,透過 `Process`(循序或階層式)執行 |
| 主要介面 | Python + TS SDK;LangGraph Server (REST/SSE/WebSocket);LangGraph Studio | Python SDK;.NET SDK;CLI Studio | Python SDK(TS 已規劃);Responses API + sandbox tool | Python SDK;YAML crew/task 設定 |
| State 模型 | 具型別的 `TypedDict` / Pydantic / dataclass;每個 channel 以 reducer 合併;會被 checkpoint | 每個 agent 的本地 state;訊息在共享 bus 上 | 每個 `Agent` 的設定 + run 範圍內的記憶體 store | 每個 agent 的隱式對話/上下文,task 層級的輸出 |
| 持久化執行 | 一等公民:`exit` / `async` / `sync` durability mode;完整 pause/replay/time-travel | 未內建;使用者自行串接持久化 | 記憶體持久化有;完整的 workflow 可恢復性並非原語 | 未內建;近期的 flow-mode 加入有限的持久化 |
| Human-in-the-loop | 一等公民 `interrupt()` + 綁定 checkpoint 的靜態 `interrupt_before/after` | 透過自訂 agent 或 UserProxy 模式手動處理 | 每個 tool 有 `tool_use_behavior` 與核准 hook | 手動;依賴 tool wrapper |
| Streaming | 原生支援:token、values、updates、debug、custom;v2 具型別事件;v3 具型別的 per-channel 投影(1.2 alpha) | 透過 runtime 提供 token + event streaming | Token + tool-event streaming;原生 Responses API streaming | 透過底層 LLM client 提供 token streaming |
| 最佳適用場景 | 長時間執行、多分支、可稽核且需要 replay 與 HITL 的 agent | 多 agent 對話的研究 / 原型 | 以 OpenAI 為中心、需要程式碼執行沙箱與 Responses API 功能的 agent | 快速構建的角色式管線(research → write → review) |
| 優點 | 明確的 graph + state;預設持久化;豐富的部署選項;緊密的 LangSmith 追蹤 | 乾淨的 actor 模型;強大的 group-chat 模式;.NET 對等性 | 開箱即用的 sandbox + filesystem tool;最貼近 OpenAI Responses/Realtime;透過 Chat Completions 支援 100+ LLM | 從零到第一個 agent 的時間最短;可讀的角色抽象;社群活躍 |
| 缺點 | 學習曲線較陡;state 的心智模型較有主見;可觀測性的故事真心想要 LangSmith | 處於維護模式 —— Microsoft 現在將新工作指向 Microsoft Agent Framework | Python 優先;廠商錨定;對於長期可恢復 workflow 的故事較弱 | 在大規模下的控制力較弱;在等效的多 agent 任務上有報告指出 token 相較 LangGraph 多出約 18%;production 姿態仍在成熟中 |
| 授權 | MIT(OSS);LangGraph Platform 為商業授權 | MIT(OSS) | MIT(OSS);LLM 呼叫透過 OpenAI API 計費 | MIT(OSS);CrewAI Enterprise 為商業授權 |
| 成本(粗估,2026 年 5 月) | Framework 免費。Platform:每執行一個 node $0.001 + 每個 deployment-run $0.005;Self-Hosted Lite 在 1M 個 node 內免費;LangSmith Plus $39/seat/月。典型 5 工程師 production 團隊:LangSmith 約 $140–300/月;大型部署 $650–6,000+/月 | Framework 免費;你自帶的 LLM + 基礎設施另計費用 | Framework 免費;按 OpenAI API 標準費率支付,再加上 sandbox/tool 執行成本 | Framework 免費;CrewAI Enterprise 採客製化報價 |
| 部署選項 | Cloud SaaS、BYOC(目前僅 AWS)、Self-Hosted Enterprise(雲端 / 混合 / 完全地端)、Self-Hosted Lite | 自架任何可執行 Python/.NET 的環境;對 Azure 友善 | OpenAI 的託管 runtime;可自架的 Python 服務呼叫 API | 自架;CrewAI Enterprise SaaS |
| 多語言 | Python、TypeScript | Python、.NET | Python(TS 已規劃) | 僅 Python |

*成本數字為 2026 年 5 月廠商定價頁的公開定價估計值且會變動;請視為數量級的參考。*

## 深入報告

### 1. 架構深入剖析

執行的單位是一個由 state schema(`TypedDict`、Pydantic model 或 `dataclass`)參數化的 **`StateGraph`**:

```
        ┌─────────┐  conditional   ┌─────────┐
START ─▶│  plan   │───────edge────▶│  tool   │──▶ ...
        └────┬────┘                 └────┬────┘
             │                           │
             ▼                           ▼
        ┌─────────┐                 ┌─────────┐
        │ respond │ ◀───────────────│ reflect │
        └────┬────┘                 └─────────┘
             ▼
            END
```

Node 是普通函式 `(state) -> partial_state`。Runtime 透過 per-channel 的 **reducer**(例如 append-only 訊息 channel 的 `add_messages`,或計數器的自訂 merge)將每個 node 的回傳值合併回共享的 state。這比較接近 Pregel 風格的 superstep 模型,而不是程序性的呼叫圖:在單一步驟中,runtime 決定哪些 node 可以執行、執行它們(在 graph 允許的地方並行執行),然後 commit state。

編譯(`graph.compile(checkpointer=...)`)會產生一個 runnable,公開 `invoke`、`stream`、`astream`、`get_state`、`update_state` 與 `aget_state_history`。編譯後的物件也是 **LangGraph Server** 部署的單位,以版本化的「assistant」形式提供 REST、SSE 與 WebSocket 端點。

State 持久化建立在一個 `BaseCheckpointSaver` 介面之上,實作 `put`、`get_tuple`、`list` 與 `delete_thread`。Production 部署通常使用 `PostgresSaver`(`langgraph-checkpoint-postgres`)或 `RedisSaver`;`MemorySaver` 僅供開發使用。Checkpoint 組織為 **thread**(一個 thread ≈ 一段對話 / session),且 thread 內每一步都有一個 checkpoint,內含 metadata、channel 值與 pending writes。這個結構正是讓 `aget_state_history()` 與 time-travel 除錯能夠運作的原因。

### 2. 關鍵設計模式與取捨

- **明確的 state 勝過隱含的對話。** AutoGen 與 CrewAI 傾向透過聊天訊息穿針引線地傳遞 agent 上下文。LangGraph 強迫你預先宣告 channel 與 reducer。對簡單情境而言顯得冗長,但這讓並行性、部分更新與 replay 變得可行。當你加入第二個 agent 或平行分支的那一刻,這個決策就會回本。
- **Graph 勝過 decorator/agent 迴圈。** OpenAI 的 Agents SDK 與大多數「agent」函式庫將迴圈隱藏在 `Runner.run()` 之內。LangGraph 將迴圈作為 graph 暴露出來;你可以攔截任何 edge、插入一個 node,或透過回傳 `Command(goto=...)` 來短路執行。功能強大,代價是樣板程式碼。
- **Durability mode 可設定。** `exit`(最便宜,在完成或 interrupt 時持久化)、`async`(與下一步同時持久化——有小型崩潰窗口風險)、`sync`(在下一步開始前持久化——高風險流程的預設值)。這是一個真正的 workflow-engine 概念,而不是 checkpoint 的揮手帶過。
- **Interrupt 是持久化,而非執行緒。** `interrupt(value)` 拋出 `GraphInterrupt`,checkpointer 儲存 pending state,並透過在相同 `thread_id` 上以 `Command(resume=human_value)` 重新呼叫 graph 來恢復執行。伺服器上沒有等待中的 Python coroutine —— 該行程可以死掉,並在數小時後於另一個 worker 上恢復。
- **Reducer 是讓並行變得理性的方式。** 兩個平行 node 分別回傳 `{"messages": [m1]}` 與 `{"messages": [m2]}` 時,會透過該 channel 的 reducer 合併;沒有 reducer 的話,last-write-wins 會默默丟棄一部分工作。

### 3. 正確性與持久化模型

- **原子性單位:** superstep。要不是該步的所有寫入都落入 checkpoint,就是都沒落入。Node 在步驟中途崩潰會回滾該步;重試會從前一個 checkpoint 重新執行該 node。
- **失敗恢復:** 以 `input=None` 與相同的 `thread_id` 重新呼叫 graph;runtime 會從最後一個成功的 checkpoint 恢復。這在跨 process 重啟、host 故障與 LLM provider 中斷的情境下都能運作。
- **決定性:** 並不保證 —— LLM 呼叫本質上是非決定性的 —— 但給定固定的 state 與 node 輸出,*graph 遍歷*是決定性的,這讓 time-travel 除錯具有意義。重新執行步驟的寫入會取代 pending writes;你不會從幂等的 node 程式碼得到重複的副作用,但你必須自己讓外部副作用(API 呼叫、付款)具備幂等性。
- **Time travel:** `aget_state_history(thread_id)` 回傳所有 checkpoint;`update_state(config, values, as_node=...)` 讓你能從任一 checkpoint 以新值分支並恢復——對於除錯以及「如果使用者當時回答不同會怎樣」的流程很有用。
- **已知的銳利邊角(於 2026 年 3 月的 v2 streaming 中修正):** 較早的版本在 subgraph + interrupt 組合下,replay 之間可能會重複使用過時的 `RESUME` 值;如果你還在使用 pre-`version="v2"` 的 streaming 並搭配 subgraph 與 HITL,請升級。

### 4. 效能特性

LangGraph 的開銷主要由 checkpointer I/O 與 reducer 成本主導,而非 framework 派發。來自公開來源的幾個具體要點,所有都應該針對你的 workload 重新驗證:

- 一份 2026 年的第三方 benchmark 比較 CrewAI 與 LangGraph 在等效的多 agent 工單分流任務上,報告 LangGraph 完成 **76%** 的中等任務(3–5 個 tool 呼叫),CrewAI 為 **71%**,且 CrewAI 在相同形狀的任務上使用了**約多 18%** 的 token。來源是一個與廠商相近的 blog,所以視為方向性、而非權威性的。
- 寫入 Postgres 的 per-step checkpoint 通常比 LLM 呼叫多花幾毫秒;對於高吞吐 workload,**`durability="async"`** 讓 checkpoint 成本離開熱路徑,代價是小型崩潰窗口風險。
- 1.2 alpha(2026 年 5 月)導入了 **`DeltaChannel`**,儲存增量 delta 而非完整 channel 快照——對於訊息列表會持續增長的長時間執行 thread 而言意義重大,因為樸素的 PostgresSaver checkpoint 可能會膨脹。
- Per-node `run_timeout` 與 `idle_timeout`(也是 1.2 alpha)會拋出 `NodeTimeoutError`,可與重試政策組合。在這之前,失控的 node 是 production 中真實存在的問題 —— 你必須自己包裝呼叫。
- 目前 streaming 有三種共存的協定:legacy dict event、**`version="v2"`** 具型別的 `StreamPart`,以及 **`version="v3"`** content-block/per-channel 投影。v3 是新程式碼應該瞄準的目標;v1 仍可運作。

### 5. 維運模型

- **本地開發:** `langgraph dev` 啟動具熱重載的 Studio;`MemorySaver` 即可。
- **自架:** `langgraph build` 產出 LangGraph Server 的 Docker image;以 Postgres 作為 state 後端、Redis 作為 task queue,並使用你自己的 LLM 憑證。**Self-Hosted Lite** 在執行 1M 個 node 內免費,之後升級到 **Self-Hosted Enterprise**(無遷移)。
- **BYOC(截至 2026 年 5 月僅 AWS):** LangChain 操作 control plane;data plane 在你的 VPC 中執行。GCP/Azure BYOC 已被承諾但尚未 GA —— 在決定之前請先確認。
- **Cloud SaaS:** 託管的 control + data plane。
- **可觀測性:** LangSmith 是阻力最小的路徑 —— 自動儀器化的追蹤、評估器、prompt 版本管理、資料集。OpenTelemetry export 存在但較不成熟;如果你有一個有立場的 tracing stack(Datadog、Honeycomb、Phoenix、Langfuse),你會需要做更多接線工作。為此預留預算。
- **常見故障模式:** (a) 長 thread 上的 checkpointer-table 膨脹 —— 使用 `DeltaChannel` 或 thread-rollover;(b) 高並發下 Postgres 連線池耗竭;(c) 被 interrupt 後從未恢復的執行靜悄悄地在 `checkpoints` table 中累積 —— 加一個 sweeper。

### 6. 安全與多租戶

- **AuthN/AuthZ:** LangGraph Server 透過 `langgraph_sdk.Auth` 支援自訂驗證(per-thread/assistant 範圍);某些部署組合在特定版本之前歷史上於自架時搭配自訂驗證有問題 —— pin 到目前版本。
- **加密:** at-rest 取決於 checkpointer 後端(例如 Postgres TDE、RDS encryption);伺服器端的 in-flight TLS。
- **租戶隔離:** 通常建模為以租戶命名空間化的 `thread_id` + `Auth` 範圍。Framework 並不在 OS/process 層隔離租戶 —— 那是你的部署層的工作。
- **Prompt/資料外洩面:** state 在 checkpointer 中是明文;如果你透過長時間執行的 thread 持久化使用者 PII,checkpointer table 現在就納入你資料處理政策的範圍。

### 7. 生態系與整合

- **模型:** 透過 `langchain` 模型包裝器支援任何 —— OpenAI、Anthropic、Google、Bedrock、vLLM、Ollama 等。
- **Tool:** LangChain tool 生態系(約 750+ 整合)加上 MCP(Model Context Protocol)透過 `langchain-mcp-adapters` 支援。
- **向量儲存 / 檢索:** LangChain retriever 作為 node 接入。
- **追蹤與評估:** LangSmith(一等公民)、OTel(可用)、Phoenix/Langfuse(社群)。
- **前端:** LangGraph Studio 提供 graph 視覺化 / state 檢視 / time-travel;CopilotKit、AG-UI 與 Generative UI 功能用於應用程式整合。
- **部署鄰近物:** 透過已發佈的 Helm chart 為 Self-Hosted Enterprise 上 Kubernetes;有 Terraform 模組但涵蓋程度不一。

### 8. 子比較:持久化執行 stack

LangGraph 有時會被拿來與通用的 workflow engine(Temporal、Restate)比較,而非與其他 agent framework 比較。聚焦來看:

| 維度 | **LangGraph** | **Temporal** | **AutoGen** |
|---|---|---|---|
| 主要抽象 | 由 LLM 感知 node 組成的具型別 state graph | Workflow + activity 函式 | 交換訊息的非同步 actor |
| 決定性保證 | Graph 遍歷具決定性;node 主體自由格式 | 透過 replay 強制執行的嚴格 workflow 決定性 | 未強制執行 |
| 內建 HITL | `interrupt()` 是核心 | 自訂 signal/query | 自訂 |
| LLM 人因工程 | 一等公民(streaming、tool node、message reducer) | 無 —— 自帶 | 對於 chat 是一等公民 |
| 最佳適用場景 | 需要持久化的 LLM agent | 任何需要持久化的 workflow,不論是否 LLM | 多 agent 對話研究 |

如果你會*純粹*為了一個 LLM workflow 使用 Temporal,LangGraph 能以遠較短的學習曲線給你大部分你想要的。如果你的 workflow 主要不是 LLM 性質(訂單、付款、對帳),裡面有一個 LLM 步驟,那 Temporal 仍然是正確的骨幹。

### 9. 何時選擇 LangGraph —— 何時不要

選擇它的時機:

- 你需要**可恢復、可稽核**的 agent 執行(合規、客服、研究、code-mod)—— 持久化執行是差異化關鍵。
- 你需要**細緻的分支**,搭配平行 node、條件式 edge 與部分 state 更新。
- **HITL** 是產品需求而非事後想法 —— 核准、編輯、escalation。
- 你已經在 **LangChain/LangSmith** stack 上,或願意進入。
- 你需要對相同的 agent 定義有**多語言**(Python + TypeScript)的對等性。

跳過它的時機:

- 你的 agent 是單一 LLM 的 tool 迴圈,沒有持久化或分支需求 —— `openai-agents`、純 LangChain,甚至是直接 API 呼叫,活動零件較少。
- 你想要**開箱即用的角色/persona 抽象** —— CrewAI 從零開始更快。
- 你**錨定 OpenAI** 並想要一等公民的 Responses-API 功能(computer use、code interpreter、file search)—— OpenAI Agents SDK 在那邊更貼合。
- 你致力於把 **.NET** 作為主要語言 —— 在這些 framework 中,AutoGen 仍是唯一具有可信 .NET 故事者。
- 你的團隊無法容忍 **LangChain 生態系的足跡**(許多套件、快速演變的 API),或你無法負擔 LangSmith 又不想手動接線 OTel。

### 10. 結語 TL;DR

LangGraph 在 2026 年,當你把 agent 視為長時間執行、具狀態的 workflow 而非一次性 LLM 呼叫時,是事實上的選擇。v1.0 穩定了 API;活躍的 1.2 alpha(delta channel、per-node timeout、v3 streaming)正在磨平 production 的粗糙邊角。它相對於 CrewAI、AutoGen 與 OpenAI Agents SDK 的差異化不在於更好的 LLM 人因工程 —— 而在於**有 checkpoint 的持久化執行與 human-in-the-loop interrupt 是同一個原語**。買下那個原語,陡峭的學習曲線與 LangSmith 引力通常是一筆公道交易。

## Sources

- [LangGraph 1.0 GA announcement (LangChain changelog)](https://changelog.langchain.com/announcements/langgraph-1-0-is-now-generally-available)
- [What's new in LangGraph v1 — official docs](https://docs.langchain.com/oss/python/releases/langgraph-v1)
- [LangGraph changelog (full)](https://docs.langchain.com/oss/python/releases/changelog)
- [LangGraph releases on GitHub](https://github.com/langchain-ai/langgraph/releases)
- [Persistence & checkpointers — official docs](https://docs.langchain.com/oss/python/langgraph/persistence)
- [Durable execution — official docs](https://docs.langchain.com/oss/python/langgraph/durable-execution)
- [LangGraph Platform deployment options](https://changelog.langchain.com/announcements/langgraph-platform-new-deployment-options-for-agent-infrastructure)
- [LangSmith pricing](https://www.langchain.com/pricing)
- [AutoGen v0.4 architecture (Microsoft Research)](https://www.microsoft.com/en-us/research/blog/autogen-v0-4-reimagining-the-foundation-of-agentic-ai-for-scale-extensibility-and-robustness/)
- [AutoGen GitHub](https://github.com/microsoft/autogen)
- [OpenAI Agents SDK — next evolution (Apr 2026)](https://openai.com/index/the-next-evolution-of-the-agents-sdk/)
- [OpenAI Agents SDK docs](https://openai.github.io/openai-agents-python/)
- [CrewAI GitHub](https://github.com/crewaiinc/crewai)
- [LangGraph vs CrewAI vs AutoGen 2026 benchmarks (third-party blog — directional)](https://pooya.blog/blog/crewai-vs-langgraph-autogen-comparison-2026/)
- [LangGraph vs CrewAI 2026 (Redwerk)](https://redwerk.com/blog/langgraph-vs-crewai/)
