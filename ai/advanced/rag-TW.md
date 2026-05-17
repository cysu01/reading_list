# RAG (Retrieval-Augmented Generation)

*架構模式,評估時間為 2026 年 5 月。*

## 摘要

RAG 是一種架構模式,將 LLM 的輸出基於推論時從外部儲存取得的文件,而非依賴模型在訓練中所記憶的內容。其單一決定性特性是知識存在於權重**之外**:你可以在不重新訓練的情況下更新語料庫、引用支撐每個答案的確切段落,並服務一個比任何 context window 都大的知識庫。這種解耦正是為何 RAG(以某種形式)依據 2026 AI Trends 報告位於約 84% 的生產 AI 助手之中,即使 1–2M-token 的前沿模型對較簡單的「全部塞進 context」替代方案施加了真實壓力。RAG 適合在你擁有龐大或變動頻繁的語料庫、需要溯源歸因,且在意每次查詢成本時使用;它不適合語料庫小且穩定(請使用 long-context)、任務關於行為而非事實(請使用 fine-tuning),或查詢需要跨文件深度綜合而單次 retrieve 無法浮現的情況(請使用 agentic 或 graph 變體)。2026 年的預設不再是「naive RAG」——而是 hybrid sparse+dense retrieval 搭配 cross-encoder reranker,且架構爭論已經轉向 LLM 對自身 retrieval 應該擁有*多少自主權*。

## RAG 與架構替代方案比較

真正的選擇在於*知識存放的位置*:在 vector index 中、在 prompt 中、在權重中,或在一個使用工具的 agent 的工作記憶中。

| 維度 | **RAG (hybrid + rerank, canonical)** | **Long-context prompting** | **Fine-tuning (LoRA / SFT)** | **Agentic RAG** |
|---|---|---|---|---|
| 類型 / 類別 | Retrieval + grounded generation | 大規模 in-context learning | 參數更新 | 以 retrieval 為工具的 agent 迴圈 |
| 核心架構 | Embed query → ANN + BM25 fuse → cross-encoder rerank → top-k 進入 prompt | 將所有相關文件串接進 context (≤1–2M tokens) | 在 Q/A 或領域語料庫上訓練 adapter (LoRA) 或完整 SFT | LLM 規劃 queries、呼叫 retrieval 工具、反思、重試直到有信心 |
| 主要介面 | Vector DB SDK + LLM API + reranker API | 僅 LLM API | LLM API + 訓練 pipeline | Agent framework + RAG 工具 |
| 知識更新 | 重新 embed 變更的文件(數分鐘) | 編輯 prompt | 重新訓練(數小時至數天,GPU $) | 重新索引 retrieval 工具 |
| 每次查詢成本 @ 100k QPD | 典型 ~$0.20 (embed + DB + rerank + gen);依 2026 企業拆解約 ~$19k/月 | $1–$10+;長輸入 tokens 主導 | 僅 LLM call (~$0.005–0.05) — 執行時最便宜 | 單次 RAG call 的 2–5 倍(多步驟 LLM) |
| 延遲 (p50) | 200–800 ms | million-token contexts 為 5–60 s | 與 base LLM 相同 | 數秒至數十秒 |
| 溯源歸因 | ✅ 每個答案對應 source chunk | 弱(模型可能指向「context」) | ❌ 知識融合進權重 | ✅ 每步驟皆有 traces |
| Lost-in-the-middle | 有限 (k 很小) | 超過 ~30–50% 填充後嚴重 | N/A | 有限 |
| 最佳適用 | 大型、快速變化、可溯源歸因的知識庫 | 小型/中型穩定語料庫、單文件深度推理 | 風格/格式/行為、低延遲專屬任務 | 多跳、模糊、探索性 queries |
| 優點 | 便宜、可溯源、更新快、可擴展至 TB 級語料庫 | 最簡單的技術堆疊;沒有 retrieval-quality bugs | 最低推論延遲;無 retrieval 相依性 | 在複雜 queries 上最高準確率 |
| 缺點 | Retrieval 現在是瓶頸 — chunking、hybrid、rerank 主導品質 | 成本 + lost-in-the-middle + 上限 = window 大小 | 知識會過時;有降級基礎能力的風險;更新需要重新訓練 | 成本、延遲、評估與除錯較困難 |
| 授權 / 取得方式 | 模式;從 OSS (pgvector, Qdrant, BGE) 或代管 (Pinecone, Cohere Rerank) 組裝 | 與 base LLM 相同 (OpenAI / Anthropic / Google) | Base model + GPU 運算(自有或租用) | 模式 |
| TCO 概估 (3 年, 100k QPD 企業) | $250k–$1M (代管 VDB + rerank + LLM) | LLM API 支出主導;相同 QPS 下可超過 RAG 5–20× | 每個 fine-tune 週期 $10k–$200k + 較低的每次查詢 LLM 帳單 | RAG 成本 × 2–5 |

*成本數字為截至 2026-Q1 的粗略公開定價估算 (OpenAI text-embedding-3、Cohere Rerank v4、中階代管 vector DB、前沿 LLM 每 1M output tokens $5–15)。任何承諾前請對照當前定價驗證。*

---

## 深入剖析:RAG 實際如何運作、會在哪裡失敗、應該建構什麼

### Pipeline,以及品質在何處決定

一個典型的 2026 RAG pipeline 有六個階段。其中三個是答案品質實際被決定的地方;另外三個是基礎管線。

```
ingest → chunk → embed → index ─┐
                                ├→ retrieve → rerank → generate
                          query ─┘
```

基礎管線 — ingest、embed、index — 大多是廠商選擇。使用 OpenAI text-embedding-3-large 或 BAAI bge-large;把向量放在 Qdrant、Weaviate 或 pgvector 裡。從中階 embedder 移到頂級 embedder 很少能買到超過 5–10 點的 recall。如果你的第一個月花在這裡優化,你就分配錯誤了。

另外三個階段 — chunking、retrieval、reranking — 是生產 RAG 品質所在之處。

**Chunking。** 一個 chunk 是 retrieval 的原子單位。太小你會失去 context(「誰」和「什麼」位於不同的 chunks)。太大你會稀釋 embedding 訊號 — 一個 2000-token chunk 會平均為一個模糊的主題向量。多數生產系統落在 256–512 token chunks 並有 10–20% 重疊,加上一個「parent doc」指標讓 LLM 在 retrieval 後能看到周圍 context。Naive RAG 幾乎毫無例外地會首先在這裡失敗。

**Hybrid retrieval。** 純 dense retrieval — 在 embeddings 上的 cosine similarity — 在 lexical-precision 任務上敗給傳統的 BM25:金融、法律、程式碼、產品 SKUs。2026 年的 benchmark 共識(在多項研究中已被複製)是 BM25 在 table-and-text 文件上在除了 recall@20 以外的每個指標上實際擊敗 text-embedding-3-large。生產系統同時執行兩者,並以 Reciprocal Rank Fusion 或學習得到的組合來融合。因為「embeddings 更聰明」而跳過 BM25 是類別錯誤 — embeddings 和 lexical search 壓縮不同的訊號,你兩者都想要。

**Reranking。** 單一最高 ROI 的元件,也是最常被跳過的元件。便宜的 retrieval 階段使用 *bi-encoder*(query 與 doc 獨立 embedded,然後做點積)。一個像 Cohere Rerank v4 這樣的 *cross-encoder* reranker 共同接收 (query, doc) 對並評分真實相關性 — 代價是每對的延遲。標準架構:用便宜的 bi-encoder retrieve 50–100 個候選,以 cross-encoder rerank 到 top-5。已發表的數字:hybrid + Cohere Rerank 將 Recall@5 從 0.587(僅 dense)提升至 0.816 (+39%),MRR@3 從 0.433 至 0.605(+39.7% 相對)。

### 為什麼需要 re-ranker,而不是更好的 embedder?

這是任何生產 RAG 系統的承重設計決策,值得真正深入探討。

一個 bi-encoder embedder 必須為每份文件產生單一固定向量 — 這個向量必須對*每一種可能的 query* 都有用。在架構上,embedder 被迫壓縮到主題層級的語意,並失去 cross-encoder 透過共同關注 query 與 document tokens 所做的「這個確切段落是否回答這個確切問題」的細粒度判斷。

顯而易見的替代方案 — 「只要訓練一個更好的 embedder」 — 是 Cohere、OpenAI、BAAI 和 Voyage 三年來一直在做的事,且邊際效益遞減。前沿 embedder 仍然在 top-k 精確度上敗給三年前的 cross-encoder,因為 bi-encoder 架構本質上資訊瓶頸:你無法在 1024 個 floats 中無損地表示「這個 300 字段落對*每一種可設想 query* 的相關性」。這也是為什麼 ColBERT 和 late-interaction 方法存在的原因 — 它們朝 cross-attention 移動了一部分,同時保留可檢索性。

Cross-encoder rerank 階段的成本是真實的:另一次網路呼叫、另一個要部署的模型、在 retrieval 之上 ~50–200 ms 的延遲。Cohere 端點以 300K tokens/分鐘處理一個典型的 23K-query benchmark 大約需要一小時 — 並非免費,但相對於它之前的 LLM call 是便宜的。跳過它是虛假的節約。Reranker 是 pipeline 中模型在 retrieval 最終確定前能夠*共同*查看 query 與 document 的唯一地方,而該共同注意力正是大部分精確度勝利的來源。

這就是為什麼「hybrid search + rerank」成為 2026 年的預設起點架構,以及為什麼「我們使用 vector search」不再是描述生產 RAG 系統的充分描述。

### 變體全景

「RAG」不再指單一架構。重要的口味以及何時適用:

| 變體 | 機制 | 何時勝出... | 成本 |
|---|---|---|---|
| **Naive RAG** | 單次 dense retrieval → LLM | Demo、prototype、簡單 FAQ | 錯失 lexical 訊號;無 rerank;脆弱 |
| **Hybrid + rerank**(2026 典型) | BM25 + dense 融合,然後 cross-encoder rerank | 大部分生產 text/table 工作負載 | 兩個階段的搜尋基礎設施 |
| **GraphRAG** (Microsoft, 2024+) | 離線從語料庫建立 entity/relation graph;查詢時 retrieve 子圖 | 多跳、「跨語料庫綜合」queries | 建立 graph 的離線 LLM 成本高昂;在簡單查找上比 vanilla 差 |
| **LazyGraphRAG** (Microsoft, 2024) | 便宜的、查詢時 graph 建構 | 大型語料庫、中等複雜度 queries | 工具較不成熟;僅有廠商數字 |
| **Agentic RAG** | LLM agent 規劃 queries、retrieve、批判、重試 | 模糊 queries、多來源、推理密集 | 單次 RAG 的 2–5× 成本與延遲 |

來自 arXiv 2502.11371 (2025 年 2 月) 的系統性發現:GraphRAG 在多數真實世界任務上**表現遜於** vanilla RAG,且只在多跳綜合密集 queries 上值得其複雜度。Microsoft 自家的 LazyGraphRAG 論文聲稱在其所有 96 個 benchmark 格中對 1M-token context 的 vector RAG 取得勝利 — 但那些 benchmarks 是由同一個團隊撰寫,並嚴重偏向 holistic-summary queries,而 graph 在這類查詢上結構性勝出。把公開數字當作方向性正確,而非在你資料上實測得到的。

2025–2026 的明確方向轉變是從 **single-shot RAG** 移向 **agentic retrieval**,其中 LLM 將 retrieval 視為它可以用不同 queries 重複呼叫的工具。這在困難問題上以延遲與成本換取準確度,而這正是 Anthropic 的 `web_search`/`memory` 工具、OpenAI 的 File Search 以及 Agentic RAG 文獻所匯聚的方向。

### Long-context 重新設計的問題

如果你在 2026 開始一個 RAG 專案,你必須誠實回答:*我會不會就把所有東西塞進一個 1M–2M token 的 Gemini 或 Claude window?* 資深工程師的答案:

- **對於小到中型的穩定語料庫 (<200k tokens、低查詢量):** long-context 勝出。較少基礎設施、較少 bugs、無 retrieval-quality 倒退需追逐。5 分鐘 TTL 的 prompt caching 彌補大部分成本差距。這是最強的「RAG 已死」案例,且是正確的。
- **在大規模或變動時 (>1M tokens、>10k QPD、每日更新):** RAG 仍然決定性地勝出。Long-context 成本隨語料庫 × 查詢線性擴展;RAG 成本隨*被檢索的* chunks × 查詢擴展。在 10M-token 語料庫上 100k QPD,long-context 可以是調校良好的 RAG 帳單的 10–50×,且「lost in the middle」在超過 ~30–50% context 填充後降低準確度 (Stanford 2023,並於 2025 在每個模型家族中被複製)。
- **對於跨已知小型文件集的多跳推理:** long-context 經常徹底打敗 single-shot RAG,因為 k=5 retrieved chunks 可能不包含橋接事實。要嘛走 long-context、agentic RAG,或 GraphRAG — 但不要期待 naive RAG 能處理。

如果我今天從零開始建構,我不會建構「2023 RAG」(單次 retrieve + 塞入)。可辯護的 2026 架構是*在 hybrid+rerank 工具上的 agentic retrieval*,其中 retrieval 是數個工具之一,而 LLM 決定何時已看到足夠的證據。先把 hybrid+rerank pipeline 當作工具建構,然後在評估顯示單次正把準確度留在桌上時,於其上添加自主性。

### 評估:沒人想做的部分

一個 RAG 系統有*兩個*傳統 QA 指標 (BLEU、ROUGE、exact-match) 無法分辨的失敗模式:

1. **Retrieval failure** — 正確的段落不在 top-k 中。
2. **Generation failure** — 正確的段落已被 retrieved,但模型忽略它或在其周圍幻覺。

RAGAS(以及 DeepEval、TruLens)沿這些軸線分解評估:

- **Context Precision / Recall** — retrieval 是否帶回相關 chunks?隔離地診斷 retriever。
- **Faithfulness** — 答案中的每個主張必須由 retrieved context 支持 (LLM-as-judge 在 claim/context 對上)。診斷幻覺。
- **Answer Relevancy** — 答案是否實際處理問題?捕捉「技術上有依據但離題」。

務實工作流程:從真實使用者 queries 建立一個 100–500 題的 golden set,每週執行這四個指標,把 faithfulness 上 5+ 點的任何下降視為 P1。LLM-as-judge 指標在 5 點以下的變動是噪音;追逐噪音是團隊燒掉整季調校實際上沒有移動品質的 prompts 的方式。

評估空間中誠實的差距:沒有人對 agentic RAG 的*整體*評估有很好的答案,因為當 agent 被允許精煉自己的 queries 時,ground truth 會移動。這是開放研究領域。

### 操作上的陷阱

- **Reindexing strategy。** 在每次文件變更時重新 embedding 整個語料庫在大規模下是浪費。2026 趨勢是 *streaming reindex* — 僅重新 embedded 變更的 chunks,通常透過 CDC pipelines (Debezium → Kafka → embedding worker)。在文件密集語料庫上減少 embedding API 支出 5–10×。
- **Embedding drift。** 當你升級 embedding 模型 (3-small → 3-large,或 v3 → v4),你 index 中的每個向量都變得無法比較。你必須重新 embed 整個語料庫並執行 shadow index 直到兩者一致。每 12–18 個月預算一次。
- **Query embedding 成本是沉默的殺手。** 在一個 100k QPD 系統中,*query* embedding 成本超過語料庫 embedding 成本數個數量級 — 你 embed 每個 query,而你只 embed 每份文件一次。團隊在估算規模時忘記這點。
- **Chunking 不是一次性決定。** 當文件範本變更時,chunk 邊界不再與語意單位對齊。把 chunking 當作系統中一個版本化的、被評估的元件,而不是設定檔中的一個設定。

### 何時選 RAG / 何時不選

**選 RAG 當:**
- 語料庫龐大 (>500k tokens) 或每日/每週變動。
- 答案必須為合規/法律原因引用來源。
- 你需要在不重新訓練的情況下更新知識。
- 在生產量下 (>10k QPD) 每次查詢成本重要。

**選別的當:**
- 知識可容於 200k tokens 且很少變更 → long-context 搭配 prompt caching。
- 任務關於*行為*(語氣、格式、分類規則)而非知識 → fine-tune。
- Queries 需要跨語料庫的多跳綜合 → 從 GraphRAG 或 agentic RAG 開始,而非 naive RAG。
- 你需要 <100 ms 端到端延遲 → rerank + LLM call 是你的下限;考慮較小的 fine-tuned 模型。

**不要打錯誤的問題。** 如果 RAG 品質差,修復幾乎從不是更大的 LLM。在多數拆解中,瓶頸是 chunking、hybrid search,或 reranker — 按此順序。在交換模型*之前*用 RAGAS 診斷。RAG 專案誘人的失敗模式是花一季從 GPT-4o 升級到前沿模型,然後發現整段時間 recall@5 都是 0.4。

### 結語 TL;DR

RAG 不是產品,它是一種紀律:把知識保持在權重之外、精確 retrieve、grounded generation,並將 retrieval 與 generation 作為分開的事物來測量。2026 預設架構 — hybrid sparse+dense retrieval、cross-encoder rerank、top-k 進入前沿 LLM — 刻意無聊;對任何比 context window 更大的知識庫來說,它是在準確度-成本前沿上最便宜的落腳處。Fine-tuning 是為了行為,long-context 是為了小型穩定語料庫,agentic 與 graph 變體是為了多跳問題。先建構無聊的 pipeline、用 RAGAS 評估,只有當指標說無聊 pipeline 已耗盡空間時才伸手去拿異國變體。

## Sources

- [Retrieval-Augmented Generation: A Comprehensive Survey of Architectures, Enhancements, and Robustness Frontiers (arXiv 2506.00054)](https://arxiv.org/abs/2506.00054) — accessed 2026-05
- [A Comprehensive Survey of Retrieval-Augmented Generation (RAG): Evolution, Current Landscape and Future Directions (arXiv 2410.12837)](https://arxiv.org/abs/2410.12837) — accessed 2026-05
- [Agentic Retrieval-Augmented Generation: A Survey on Agentic RAG (arXiv 2501.09136)](https://arxiv.org/html/2501.09136v4) — accessed 2026-05
- [RAG vs. GraphRAG: A Systematic Evaluation and Key Insights (arXiv 2502.11371)](https://arxiv.org/abs/2502.11371) — accessed 2026-05
- [When to use Graphs in RAG: A Comprehensive Analysis for Graph Retrieval-Augmented Generation (arXiv 2506.05690)](https://arxiv.org/html/2506.05690v3) — accessed 2026-05
- [GraphRAG-Bench (ICLR'26)](https://github.com/GraphRAG-Bench/GraphRAG-Benchmark) — accessed 2026-05
- [BenchmarkQED: Automated benchmarking of RAG systems — Microsoft Research](https://www.microsoft.com/en-us/research/blog/benchmarkqed-automated-benchmarking-of-rag-systems/) — accessed 2026-05
- [From BM25 to Corrective RAG: Benchmarking Retrieval Strategies for Text-and-Table Documents (arXiv 2604.01733)](https://arxiv.org/html/2604.01733v1) — accessed 2026-05
- [Optimizing RAG with Hybrid Search & Reranking — VectorHub by Superlinked](https://superlinked.com/vectorhub/articles/optimizing-rag-with-hybrid-search-reranking) — accessed 2026-05
- [Sparse vs Dense Retrieval for RAG: BM25, Embeddings, and Hybrid Search — ML Journey](https://mljourney.com/sparse-vs-dense-retrieval-for-rag-bm25-embeddings-and-hybrid-search/) — accessed 2026-05
- [Ultimate Guide to Choosing the Best Reranking Model in 2026 — ZeroEntropy](https://www.zeroentropy.dev/articles/ultimate-guide-to-choosing-the-best-reranking-model-in-2025) — accessed 2026-05
- [RAG vs Long Context: Do Vector Databases Still Matter in 2026? — Markaicode](https://markaicode.com/vs/rag-vs-long-context/) — accessed 2026-05
- [Is RAG Dead? Long Context, Grep, and the End of the Mandatory Vector DB — AkitaOnRails](https://akitaonrails.com/en/2026/04/06/rag-is-dead-long-context/) — accessed 2026-05
- [Context Length Comparison: Leading AI Models in 2026 — elvex](https://www.elvex.com/blog/context-length-comparison-ai-models-2026) — accessed 2026-05
- [Traditional RAG vs. Agentic RAG — NVIDIA Technical Blog](https://developer.nvidia.com/blog/traditional-rag-vs-agentic-rag-why-ai-agents-need-dynamic-knowledge-to-get-smarter/) — accessed 2026-05
- [What is Agentic RAG? — IBM](https://www.ibm.com/think/topics/agentic-rag) — accessed 2026-05
- [Agentic RAG: How enterprises are surmounting the limits of traditional RAG — Redis](https://redis.io/blog/agentic-rag-how-enterprises-are-surmounting-the-limits-of-traditional-rag/) — accessed 2026-05
- [A complete guide to RAG vs fine-tuning — Glean](https://www.glean.com/blog/retrieval-augemented-generation-vs-fine-tuning) — accessed 2026-05
- [RAG vs Fine-Tuning: Comparison Guide for Enterprise AI — Contextual AI](https://contextual.ai/blog/rag-vs-fine-tuning-which-approach-is-right-for-enterprise-ai) — accessed 2026-05
- [Faithfulness — Ragas docs](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/) — accessed 2026-05
- [Available metrics — Ragas docs](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/) — accessed 2026-05
- [RAG Evaluation Metrics — Confident AI](https://www.confident-ai.com/blog/rag-evaluation-metrics-answer-relevancy-faithfulness-and-more) — accessed 2026-05
- [OpenAI Embeddings API Pricing Calculator (May 2026) — costgoat](https://costgoat.com/pricing/openai-embeddings) — accessed 2026-05
- [Enterprise RAG System Cost Guide for 2026 AI Teams — Ment.tech](https://www.ment.tech/cost-to-build-rag-system-enterprise-ai-2026) — accessed 2026-05
- [RAG System Cost: 2026 Pricing, Build & Ops Guide — AlphaCorp](https://www.alphacorp.ai/blog/how-much-does-a-rag-system-cost-infrastructure-development-and-ongoing-expenses) — accessed 2026-05
- [The Embedding Model Selection Crisis — RAG About It](https://ragaboutit.com/the-embedding-model-selection-crisis-why-your-enterprise-rag-cost-is-300-higher-than-it-should-be/) — accessed 2026-05
- [RAG Architecture in 2026: How to Keep Retrieval Actually Fresh — RisingWave](https://risingwave.com/blog/rag-architecture-2026/) — accessed 2026-05
- [RAG in 2026: Bridging Knowledge and Generative AI — Squirro](https://squirro.com/squirro-blog/state-of-rag-genai) — accessed 2026-05
