# vLLM

*Evaluated: vLLM 0.20.x (April–May 2026), V1 engine default. As of May 2026.*

## 摘要

vLLM 是開源的標竿 LLM 服務引擎：一個 Python/C++/CUDA 技術堆疊，圍繞著 **PagedAttention**(KV cache 以固定大小的實體區塊組織，並透過每個序列的區塊表進行定址)和**連續批次(continuous batching)**(每次迭代之間進行排程,在前向傳遞之間接納並驅逐請求)所建構。它源自 UC Berkeley 的 Sky Computing Lab(SOSP 2023),如今已成為事實上的通用服務目標 —— 在 Meta、LinkedIn、Mistral 與 Hugging Face 承載生產流量,並支援 NVIDIA Hopper/Blackwell、AMD MI300X 等級、Google TPU、AWS Neuron 與 Intel Gaudi。它與同類產品的差異在於**廣度,而非尖峰吞吐量**:TensorRT-LLM 在 NVIDIA 專屬核心上勝過它、SGLang 在前綴密集型工作負載上勝過它,但兩者都無法涵蓋 vLLM 所支援的模型 × 硬體 × 部署矩陣。當你想用一個引擎在異質硬體上服務多樣化模型,並快速採用新模型時,選擇 vLLM;當某個特定面向(NVIDIA 尖峰效能、前綴快取命中率、HF 生態整合)佔主導地位時,則選擇 TensorRT-LLM、SGLang 或 TGI。

## 比較

| 面向 | **vLLM 0.20** | **SGLang** (latest 2026) | **TensorRT-LLM 0.19+** | **TGI v3** (HF) |
|---|---|---|---|---|
| 類型 / 類別 | 通用型 OSS LLM 服務引擎 | 為前綴共享優化的 OSS 服務引擎 | NVIDIA 專屬(OSS 授權)編譯式推論執行環境 | HF 的生產服務常駐程式(Rust + Python) |
| 核心架構 | PagedAttention KV cache + 連續批次;V1 引擎搭配非同步排程 | RadixAttention(基數樹前綴共享)+ 連續批次 | 每個模型編譯的 TensorRT engine + in-flight batching + 外掛核心 | Rust 路由器/排程器搭配每個模型的 Python worker;多後端(vLLM / TRT-LLM / 原生) |
| 主要介面 | OpenAI-compatible HTTP、gRPC、Python `LLM` API;`vllm serve` CLI | OpenAI-compatible HTTP;`sgl.Runtime`;用於提示詞/程式的結構化 `sglang` DSL | Triton Inference Server;`trtllm-serve`(OpenAI-compatible);C++/Python 執行環境 | OpenAI-compatible HTTP;`text-generation-launcher` |
| 硬體支援 | NVIDIA(Ampere/Hopper/Blackwell,含 B200/GB200)、AMD MI300X/MI325X/MI350X/MI355X(2026 年起為一級支援)、Google TPU v5e/v6e/v8i、AWS Neuron、Intel Gaudi | NVIDIA 為主;AMD ROCm;部分 TPU 工作 | 僅 NVIDIA(SM80–SM103,含 B300/GB300) | NVIDIA 為主;AMD 透過 ROCm;Inferentia/Gaudi 透過後端支援 |
| 量化 | FP8(E4M3/E5M2)、AWQ、GPTQ、BnB、FP4/MXFP4(近期)、TurboQuant 2-bit KV cache | FP8、AWQ、GPTQ;FP4 興起中 | FP8、NVFP4、MXFP4、INT4-AWQ、INT8 SmoothQuant;Hopper/Blackwell 上的 FP4 MLA | FP8、AWQ、GPTQ、BnB |
| 最佳適用場景 | 服務多種模型且頻繁換模型的異質叢集;新模型零時差支援 | 前綴密集型流量 —— 帶共享文件的 RAG、多輪對話、agent 工具呼叫追蹤 | 從固定的 NVIDIA SKU + 穩定模型榨取最高 tok/s/$ | 緊密配合 HF Hub 模型卡片、OpenTelemetry 優先的營運 |
| 優勢 | 最廣的模型 + 硬體涵蓋;新模型架構社群速度最快;成熟生態系(LMCache、llm-d、production-stack) | 最高的前綴快取命中率;在前綴密集型工作負載上最高可達約 6.4× 的優勢;可程式化的提示詞語言 | NVIDIA 上最高的尖峰吞吐量;H100/B200 在負載下最低的 p95 TTFT | 內建 Prometheus/OTel;與 HF 平順整合;v3 改善了長提示詞路徑 |
| 劣勢 | 在 NVIDIA 上並非絕對尖峰(在 70B FP8 上 50 個並發時 TRT-LLM 約領先 13%);分離式 P/D 仍標示為實驗性;排程器熱路徑歷史上有 Python 開銷 | Python 路由器在高並發下會碰到 GIL 上限(150 並發時回報約 127% CPU 上限);硬體涵蓋範圍較窄;生態系較小 | NVIDIA 鎖定;每個模型的 engine 建構(每個 SKU 每版本約 28 分鐘);新增全新模型架構速度最慢 | 現在部分上是個薄層 —— 正在積極與 vLLM 統一作為後端;開發速度不如 vLLM/SGLang 直接 |
| 數值/評估模型 | FP16/BF16 為參考;FP8 在標準評估上與 BF16 相差約 0.5–1%;新量化模式發布時附評估報告 | 在相符精度下與 vLLM 相同的模型準確度(因為是相同的模型) | builder 可能融合與參考有偏差的核心;廠商發布準確度報告 | 使用底層後端的數值表現 |
| 授權 | Apache 2.0 | Apache 2.0 | Apache 2.0(engine),但基於專有的 CUDA/TensorRT 建構 | Apache 2.0(執行環境);HF Inference Endpoints 為商業版 |
| 成本 | 框架免費。**實際基礎設施**:8×H100 SXM 80GB 叢集,1 年預留雲端約 $150–250k/年(依供應商而異)。自管 K8s 部署免費;託管服務(Anyscale、RunPod、Modal 等)增加約 10–30% 額外成本 | 與 vLLM 成本結構相同 | 相同計算成本;模型/SKU 變更時 engine 建構 + 重新編譯的工程師時間成本 | 相同計算成本;HF Inference Endpoints 託管以每副本/小時計價 |

*Cost figures are public-list / order-of-magnitude estimates as of May 2026; cloud GPU pricing moves quickly.*

### 子比較:Llama 3.3 70B FP8 在單張 H100 SXM5 80GB 上的吞吐量與延遲

取自第三方 2026 基準測試 —— 為方向性參考,非權威來源。輸入約 512 tokens,輸出約 256 tokens。

| Concurrency | vLLM tok/s | TRT-LLM tok/s | SGLang tok/s | vLLM p95 TTFT | TRT-LLM p95 TTFT | SGLang p95 TTFT |
|---|---|---|---|---|---|---|
| 1   | 120  | 130  | 125  | 68 ms   | 55 ms   | 61 ms   |
| 10  | 650  | 710  | 680  | 195 ms  | 170 ms  | 178 ms  |
| 50  | 1,850| 2,100| 1,920| 720 ms  | 620 ms  | 680 ms  |
| 100 | 2,400| 2,780| 2,460| 1,450 ms| 1,280 ms| 1,380 ms|

TensorRT-LLM 在高並發下以約 13% 領先尖峰吞吐量;vLLM 與 SGLang 在這個獨特提示詞工作負載上彼此差距約 5% 以內。SGLang 的優勢出現在**共享前綴**工作負載(RAG、多輪對話),在論文引用的相同設定中,顯示出此表格未捕捉到的、相較於 vLLM 最高達 6.4× 的提升。

## 深入報告

### 1. 架構深入剖析

```
                      ┌────────────────────────────────────┐
   HTTP/gRPC ───▶ Frontend (OpenAI-compatible)             │
                  │                                        │
                  ▼                                        │
            ┌──────────────┐    admit/evict      ┌────────┴────────┐
            │  Scheduler   │◀───────────────────▶│  Block Manager  │
            │  (V1)        │   per-iteration     │  (PagedAttn)    │
            └──────┬───────┘                     └────────┬────────┘
                   │ batched request                      │
                   ▼                                      ▼
            ┌──────────────┐                       ┌─────────────┐
            │ Model Runner │ ── attention ───────▶│  KV Cache   │
            │ (V2)         │ ── matmul/MoE ─────▶│  blocks     │
            └──────────────┘                       └─────────────┘
                   │
                   ▼
              Sampler (greedy / nucleus / spec)
                   │
                   ▼
                 Output
```

**前端(Frontend)。** 接收 OpenAI 格式的 `/v1/chat/completions`、`/v1/completions`、`/v1/embeddings`。透過 SSE 串流。Tokenization 在此進行(tokenizer 可以在 worker pool 中執行以避免阻塞事件迴圈)。

**排程器(Scheduler V1)。** 每次迭代的排程器,與 model runner 在同一個程序中執行。V1 重寫版(自 0.18 左右起為預設,不再需要 `VLLM_USE_V1=1`)消除了舊版排程器每步驟所做的 GPU↔CPU 張量複製;它鎖定主機記憶體並在取樣/輸出處理時使用零拷貝 DMA。排程器維護三個佇列 —— `waiting`、`running`、`swapped` —— 並在每步驟中填滿一個 token 預算。啟用**分塊預填(chunked prefill)**後,它可以將預填區塊與解碼步驟共同排程,讓預填不致於餓死互動式解碼。

**區塊管理器(Block Manager,PagedAttention)。** KV cache 被切分為 **16 tokens 的實體區塊**(可透過 `--block-size` 設定,常見值為 8/16/32)。每個請求擁有一個區塊表,將邏輯位置對應至實體區塊 ID。由於區塊非連續且採用引用計數,碎片化基本上被消除了 —— 已發布的浪費 KV 記憶體相較於樸素連續配置可減少達約 55%。同樣的間接性也使**自動前綴快取(automatic prefix caching, APC)**成為可能:相同的提示詞前綴會雜湊到相同的區塊 ID,K/V 張量可在請求間重複使用。

**Model Runner V2。** 擁有 CUDA graphs、attention 後端(FlashAttention、FlashInfer、ROCm-native)、MoE 核心(TritonMoE、fused experts),以及 pipeline/tensor-parallel 通訊(NCCL/RCCL、自訂 all-reduce)。V2 為 pipeline parallelism 加入了 piecewise CUDA graphs,以及更靈活的 spec-decode 拒絕取樣器。

**推測式解碼(Speculative decoding)。** 原生支援多種 draft 策略:小型 draft 模型、n-gram 查找、suffix decoding、MLP speculators,以及 EAGLE/EAGLE-3。非同步排程現在支援推測式解碼,並具備**零氣泡重疊(zero-bubble overlap)** —— 排程器將驗證延遲藏在下一步驟的準備工作之後。

**分離式預填/解碼(Disaggregated prefill/decode, P/D)。** vLLM 將預填與解碼拆分為獨立的 engine 實例,透過 KV cache 傳輸連接器連接 —— `NixlConnector`、`MooncakeConnector` 或 `P2pNcclConnector`。重點在於讓 TTFT(預填)與 ITL(解碼)各自獨立擴展,並避免長預填工作阻塞解碼步驟。截至 0.20,此功能可用且已被 Meta/LinkedIn/Mistral/HF 在生產環境中執行,但文件仍標示為實驗性,而且它對 interconnect 頻寬敏感:一個 70B FP8 預填若需要在節點間搬運數 GB 的 KV,則需要 ≥4–5 GB/s 的穩定頻寬,否則會超出你的 500 ms TTFT 預算。

### 2. 關鍵設計模式與權衡

- **區塊分頁 KV vs. 連續 KV。** 最重大的單一設計選擇。在記憶體使用率與前綴共享上回報甚多;代價是 attention 核心中多一層間接性,而這是競爭引擎(早期 TGI、FasterTransformer 時代)所避免的。多虧分頁感知的 FlashAttention/FlashInfer 核心,這個成本現在已可忽略。
- **連續批次 vs. 靜態批次。** 靜態批次浪費 GPU 在 padding 上,且強迫等待批次的尾端。連續批次每次迭代都接納新請求,只付出小幅的每步驟排程成本。V1 的主機鎖定 DMA 路徑就是要把這個成本降到在數千個並發請求下不可見的程度。
- **每個引擎一個 Python 程序,而非微服務。** 排程器、區塊管理器與 model runner 共置,以將熱路徑保留在同一程序內。多 GPU 透過同一引擎下 worker 之間的 TP/PP 來實現,而非透過多個 HTTP 服務副本。權衡:GIL 歷史上對排程器是個真實的天花板 —— V1 重寫與近期請求路徑的 C++ 工作將更多東西推離 Python,但極高並發設定仍可從在路由器後執行多個 engine 程序中受益。
- **通用 vs. 特化。** vLLM 有意識地以最後 10–15% 的 NVIDIA 專屬效能換取可攜性 —— AMD、TPU、Neuron、Gaudi 上的程式碼路徑相同。TRT-LLM 做了相反的取捨,因而稱霸頭條的 NVIDIA 基準測試。
- **零時差模型支援是個特性,不是副作用。** vLLM 在模型卡發布後數日內就推出新模型架構(DeepSeek V4、Qwen3-Next、GPT-OSS、Granite 4.1 Vision)。TRT-LLM 通常落後數週,因為新模型需要 engine 建構路徑與核心覆蓋。

### 3. 數值/準確度模型

- **預設精度:** 啟動值為 BF16,權重視模型而定。FP8(E4M3)同時用於權重與 KV 是 Hopper+ 上常見的生產設定;在標準評估上相較於 BF16 的基準準確度差距通常 ≤1%(MMLU、GSM8K、HumanEval),但在切換之前進行工作負載特定的評估是必要的。
- **量化路徑:** AWQ 與 GPTQ 用於僅權重的 INT4;SmoothQuant 變體;BnB 用於 QLoRA 式;0.20.x 中的 **TurboQuant 2-bit KV cache** 可帶來約 4× 的 KV 容量,並附帶已發布的準確度報告 —— 對於 KV 主導記憶體的超長上下文工作負載很有用。
- **決定性:** 固定模型 + 固定核心的 greedy 取樣可重現。Nucleus/temperature 取樣則不行。連續批次**不會**影響數值 —— 每個請求的前向傳遞在數學層級上是獨立的。核心 autotuning 造成的混合批次差異可能引入小幅位元層級漂移;若你需要跨副本的位元精確一致性,則固定 autotuner。

### 4. 效能特性

- **吞吐量隨並發成長,直到 KV 記憶體或計算飽和。** 在 80GB H100 上以 FP8 服務 Llama-3-70B 時,2K 上下文下通常在並發 100–150 左右 KV 會先於計算飽和。
- **TTFT 與 ITL 是核心張力。** 長預填區塊會壓垮 in-flight 解碼的 ITL;chunked prefill 緩解了這點;disaggregated P/D 完全解耦這兩者,代價是營運複雜度。
- **前綴快取的收益與工作負載相關。** 跨使用者共享系統/文件提示詞的 RAG 可獲得數量級的 TTFT 減少;純使用者產生的提示詞收益約為 0。
- **SpecPrefill(ICML 2025)** —— 一種 draft-model 輔助的稀疏預填 —— 是值得追蹤的 2026 能力:在 Llama-3.1-405B-Instruct-FP8 上發表了最高 7.66× TTFT 與 7× 端到端 QPS,準確度損失 <5%。它可與 chunked 與 disaggregated 預填組合。
- **要留意的擴展性懸崖:** (1) 排程數千個並發極短請求時的 GIL 壓力 —— 執行多個 engine + 路由器;(2) PCIe vs NVLink 上 TP 的 NCCL all-reduce 延遲 —— 跨 PCIe 做 TP 通常是個錯誤;(3) 長上下文工作負載上的 KV 耗竭 —— 開啟 KV 卸載至 CPU 或採用激進的驅逐策略。

### 5. 營運模型

- **單節點:** `vllm serve <model>` 是進入點。在受支援硬體上以 pip 安裝;有 NVIDIA 與 ROCm 的 Docker 映像。
- **多節點 / 叢集:** 透過官方 Helm chart 與 `vllm-project/production-stack` 部署於 Kubernetes。常見堆疊將 vLLM 與 **LMCache**(KV cache 後端,可卸載至 CPU/SSD/遠端)、**llm-d**(具備前綴感知路由與 P/D 協調的請求路由器)、以及用於 RAG 的向量資料庫配對。
- **可觀測性:** 開箱即用的 Prometheus 指標(`/metrics`);OpenTelemetry 追蹤;`--enable-server-load-tracking` 旗標用於 autoscaler 訊號。歷史上不如 TGI 的指標表面那麼成熟,但差距已縮小。
- **模型推出:** 沒有 engine 編譯步驟(不像 TRT-LLM),所以換模型就等於重啟伺服器 —— 實務上很快。大多數模型從本地快取冷啟動約 90 秒以下。
- **常見故障模式:**
  - **長上下文上的 OOM** → 降低 `--max-model-len`、啟用 KV 卸載、改用較小的 block size,或量化 KV。
  - **突發負載下吞吐量崩潰** → 檢查 `--max-num-seqs` 與 `--max-num-batched-tokens` 預算。
  - **升級後神秘的準確度退化** → 核心 autotuner 選了不同的設定;固定或重新調校。
  - **分離式 P/D 的 KV 傳輸錯誤** → 連接器設定錯誤或 interconnect 頻寬不足,並非 vLLM 的 bug。

### 6. 安全性與多租戶

- **AuthN/AuthZ:** 透過 `--api-key` 提供 API-key 控管,但很初階;生產部署會在 vLLM 前端放置 API gateway 來處理認證、速率限制與配額。
- **In-flight 加密:** TLS 在入口處;vLLM 本身提供 HTTP 服務,並預期你在 sidecar/ingress 終結 TLS。
- **租戶隔離:** 程序內**沒有**租戶隔離 —— 所有請求共享同一個 KV cache pool,而**前綴快取意味著一個租戶原則上可以透過快取命中時序推論另一個租戶提示詞前綴的存在**。如果這點很重要,請將租戶分散到不同的 engine 程序/副本,或在敏感路徑停用 APC。
- **沙盒:** 沒有內建的工具/程式碼執行沙盒(vLLM 是模型伺服器,不是 agent 執行環境)。工具呼叫有對外公開;工具端的沙盒由你自己處理。

### 7. 生態系與整合

- **模型涵蓋:** Llama 3/4、Mistral/Mixtral、Qwen 2/3 + Qwen3-Next、DeepSeek V2/V3/V4、Gemma 3/4、Granite 4、Phi 4、GPT-OSS、Hunyuan v3,加上 VLMs(Llava-OneVision、InternVL、Qwen-VL、Gemma Vision、Granite Vision)。
- **包裝或嵌入 vLLM 的框架:** Ray Serve LLM、Anyscale、Modal、RunPod、BentoML、KServe、OpenLLM、LangChain(`langchain-vllm`)、LlamaIndex、Triton(vLLM 後端)、TGI(多後端模式)、SkyPilot。
- **超大規模雲廠等同/託管:** AWS SageMaker(vLLM 容器)、GCP Vertex(社群支援)、Azure ML(社群)、Databricks Model Serving(在多個模型家族底層使用 vLLM)。
- **相鄰 OSS:** **LMCache**(KV cache 層)、**llm-d**(Kubernetes-native 分散式推論路由器)、**production-stack**(參考部署)。

### 8. 何時選擇 vLLM —— 何時不選

選擇它,當:

- 你服務**多種模型**或預期頻繁更換模型 —— vLLM 最快推出新架構。
- 你在**混合硬體**(NVIDIA + AMD,或 NVIDIA + TPU/Neuron)上運行,並想要單一引擎。
- 你想要 **OSS、Apache 2.0**,流程中沒有廠商編譯器。
- 你需要**緊密回饋的實驗** —— 短迭代時間、沒有 engine 建構、OpenAI 相容 API。

略過它(或搭配使用),當:

- 你**只用 NVIDIA、且模型固定**,並需要榨取每一個百分點的吞吐量 → **TensorRT-LLM**。
- 你的流量是**前綴主導**(帶有共享文件的大量 RAG、多輪 agent 追蹤) → **SGLang**,或搭配 APC 與 llm-d 前綴感知路由器的 vLLM。
- 你**深度 HF 原生**,並想要 HF 的營運表面,且他們會把你當作 TGI 多後端 → **TGI v3**。
- 你需要 **CPU/邊緣**推論 → **llama.cpp** / **Ollama** / **MLX** 是不同類別,並非 vLLM。
- 你的工作負載是**單一模型、單一 SKU、數百個副本**,流量穩定 —— TRT-LLM 的每模型 engine 編譯成本可被攤銷;vLLM 的彈性稅就不再划算。

### 9. 結尾 TL;DR

vLLM 在 2026 年是自託管 LLM 服務的安全預設。PagedAttention + 連續批次如今已是業界標準理念 —— 讓 vLLM 維持領先的是社群速度(零時差模型、快速硬體啟用)與廣度(NVIDIA + AMD + TPU + Neuron + Gaudi 同一引擎)。它不是 NVIDIA-only 固定模型工作負載的吞吐量冠軍(那是 TensorRT-LLM),也不是 RAG/多輪上的前綴快取冠軍(那是 SGLang),但它是你能放心地將整個機隊標準化於其上,而不至於把自己畫進廠商或工作負載死角的唯一引擎。

## Sources

- [vLLM GitHub](https://github.com/vllm-project/vllm) — accessed 2026-05
- [vLLM releases](https://github.com/vllm-project/vllm/releases) — accessed 2026-05
- [vLLM docs (latest)](https://docs.vllm.ai/en/latest/) — accessed 2026-05
- [PagedAttention design — official docs](https://docs.vllm.ai/en/latest/design/paged_attention/) — accessed 2026-05
- [Automatic Prefix Caching — official docs](https://docs.vllm.ai/en/stable/design/prefix_caching/) — accessed 2026-05
- [Disaggregated prefilling — official docs](https://docs.vllm.ai/en/latest/features/disagg_prefill/) — accessed 2026-05
- [Optimization & tuning — official docs](https://docs.vllm.ai/en/stable/configuration/optimization/) — accessed 2026-05
- [Disaggregated Inference at Scale with PyTorch & vLLM (PyTorch blog)](https://pytorch.org/blog/disaggregated-inference-at-scale-with-pytorch-vllm/) — accessed 2026-05
- [vLLM × AMD ROCm attention backend (vLLM blog, Feb 2026)](https://blog.vllm.ai/2026/02/27/rocm-attention-backend.html) — accessed 2026-05
- [SGLang GitHub](https://github.com/sgl-project/sglang) — accessed 2026-05
- [SGLang vs vLLM benchmark (Particula, 2026)](https://particula.tech/blog/sglang-vs-vllm-inference-engine-comparison) — accessed 2026-05
- [vLLM vs TensorRT-LLM vs SGLang H100 benchmarks (Spheron, 2026 — directional)](https://www.spheron.network/blog/vllm-vs-tensorrt-llm-vs-sglang-benchmarks/) — accessed 2026-05
- [TensorRT-LLM GitHub](https://github.com/NVIDIA/TensorRT-LLM) — accessed 2026-05
- [TensorRT-LLM release notes](https://nvidia.github.io/TensorRT-LLM/0.19.0/release-notes.html) — accessed 2026-05
- [TGI v3 overview (HF docs)](https://huggingface.co/docs/text-generation-inference/en/conceptual/chunking) — accessed 2026-05
- [TGI multi-backend support (HF blog)](https://huggingface.co/blog/tgi-multi-backend) — accessed 2026-05
- [Comparative analysis of vLLM and TGI (arXiv 2511.17593)](https://arxiv.org/abs/2511.17593) — accessed 2026-05
- [Inside vLLM: Anatomy of a High-Throughput LLM Inference System (Aleksa Gordić)](https://www.aleksagordic.com/blog/vllm) — accessed 2026-05
