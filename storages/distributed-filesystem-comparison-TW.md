# 開源分散式檔案系統:架構師的比較分析

從實務架構師的角度檢視主要的開源分散式檔案系統,依架構家族分類,著重在架構設計、適用情境、弱點與整體擁有成本 (TCO)。

---

## 架構家族

分散式檔案系統可分為三大模式,而選擇哪種模式幾乎決定了所有的維運行為與 TCO:

- **集中式中繼資料、分散式資料** — 少數節點掌管命名空間並定位物件,實際資料儲存在其他地方。HDFS、MooseFS、JuiceFS、SeaweedFS、BeeGFS、Lustre 都屬於此類。容易推理,但中繼資料服務就是你的擴展瓶頸與故障爆炸半徑。
- **去中心化演算法定位** — 沒有中央放置權威;客戶端透過確定性函式計算物件位置。Ceph (CRUSH) 與 GlusterFS (彈性雜湊) 是經典範例。消除了中繼資料瓶頸,但代價是維運複雜度與資料重平衡 (rebalance) 的痛苦。
- **物件原生、扁平命名空間** — MinIO、SeaweedFS (物件模式)。這類系統不原生支援 POSIX 語意,以階層式中繼資料換取水平擴展的吞吐量。

---

## 逐一分析

### Ceph (RADOS / CephFS / RBD / RGW)

儲存界的瑞士刀。RADOS 是底層物件儲存;CephFS、RBD (區塊)、RGW (S3/Swift) 是上層的呈現介面。

放置採用 **CRUSH** — 一種偽隨機雜湊,根據可設定的拓撲階層 (rack → host → OSD) 將物件對應到 OSD。這代表沒有查找表、可預測的放置位置,且客戶端直接與 OSD 溝通。CephFS 的中繼資料存放於 MDS daemon (主動/待命,或主動/主動加上子樹分割)。

- **擅長**:統一的區塊 + 檔案 + 物件儲存、多 PB 等級規模、強一致性、EC 支援、成熟的快照/複製、OpenStack 整合。自我修復能力出色。
- **不擅長**:維運複雜度惡名昭彰 — 調校 OSD、PG 數量、BlueStore、scrub、復原節流是全職工作。小檔案工作負載對 CephFS 是折磨。記憶體吃重 (每個 OSD 基本要 4–6 GB)。混合工作負載在復原期間的尾延遲 (tail latency) 會很難看。
- **TCO 驅動因素**:需要真正的專業 (或付費支援,例如 Red Hat/IBM、Croit、42on、SoftIron)。EC 池可比 3 倍副本省下約 40% 的原始儲存成本,但增加 CPU 與復原 I/O 開銷。網路至少規劃 10–25 GbE;cluster 與 public network 要分離。

### GlusterFS

僅支援檔案,沒有中央中繼資料伺服器。檔案透過 **彈性雜湊** 依檔名 → brick 的對應關係放置。子卷透過 translator 組合 (DHT 負責分佈、AFR 負責複製、EC 負責分散卷)。

- **擅長**:啟動簡單、NFS/SMB 閘道、跨地理區複寫、處理大型循序檔案的歸檔/媒體工作負載。
- **不擅長**:小檔案效能差 (每個操作要透過 stat 走訪多個 brick)、新增 brick 後的 rebalance 過程痛苦且歷來不可靠、跨子卷無法原子化 rename、修復可能緩慢。Red Hat 已在 2024 年終止 Gluster Storage 產品 — 社群版仍在,但開發步調已大幅放緩。
- **TCO 驅動因素**:架設便宜,規模化後維運昂貴。對於 2026 年的正式環境決策來說,廠商背書消失是個重要的考量點。

### HDFS

大數據 DFS 的元老。NameNode 將命名空間放在記憶體;DataNode 儲存區塊 (通常為 128 MB 或 256 MB)。HA 透過 QJM + ZooKeeper + 待命 NameNode 實現;Federation 則將命名空間分割到多個 NameNode。

- **擅長**:針對大型檔案的大量循序讀取以供分析使用 (Spark、Hive、Presto/Trino、MapReduce)。3.0 開始支援 EC (RS-6-3、RS-10-4)。Locality-aware 排程是其超能力。
- **不擅長**:小檔案 (每個檔案約佔 NameNode 150 bytes 的 heap,所以 1 億個檔案 = 至少 15 GB heap)、隨機寫入 (僅 append-only)、POSIX 語意 (它不是 POSIX)、低延遲存取。即使有 Federation,NameNode 仍是擴展上限。
- **TCO 驅動因素**:被 Hadoop 工具鏈鎖定。許多公司已將分析工作負載遷移到 S3/物件儲存搭配 Trino/Spark,讓 HDFS 變成遺留系統的負擔。硬體面對通用伺服器友善,但維運很吃重。

### MinIO

S3 相容的物件儲存。分散模式對 server set 內的硬碟做 erasure coding;多個 set 部署則水平擴展。沒有獨立的中繼資料服務 — 中繼資料與物件本身共置於同一個 erasure set。

- **擅長**:S3 API、簡單維運、可預測的效能、現代化的功能 (object lock、版本控制、生命週期、分層、bucket 複寫)。非常適合 AI/ML 資料湖、Trino/Spark 後端、備份目標。
- **不擅長**:不是 POSIX、沒有真正的命名空間操作 (rename 一個「prefix」會重寫所有物件)、授權條款已轉變 (AGPLv3 + 商業授權),部分企業功能轉移到商業版本。社群版近期移除大部分 web console 引起爭議。
- **TCO 驅動因素**:EC 表示 EC:4+2 設定下只需約 33% 額外空間,相較於 3 倍副本的 200% — 在 PB 等級這是巨大的 TCO 優勢。維運負擔遠低於 Ceph。需密切觀察授權發展方向以維持合規。

### JuiceFS

較新的設計,真正有趣:**在任何物件儲存之上提供 POSIX/HDFS/S3 介面**,中繼資料則放在 Redis/TiKV/MySQL/PostgreSQL。檔案被切成 chunk (預設 64 MB),chunk 切成 slice,slice 以物件形式儲存。

- **擅長**:已經有 S3 + 託管資料庫的雲原生部署。在便宜的物件儲存之上提供 POSIX 語意。適合 AI/ML 訓練資料、共享工作區、CI/CD 快取。
- **不擅長**:中繼資料引擎就是你的 SPOF 與擴展軸線 — 你實際上是在大規模運行與調校 Redis/TiKV。冷讀取的延遲下限受限於物件儲存的延遲。社群版與企業版的功能差異不小。
- **TCO 驅動因素**:可在保留 POSIX 的同時享用 S3 (或地端 MinIO) 的經濟效益,但要多維運一個資料庫。在大規模下,若中繼資料負載適合 TiKV,通常比 Ceph 便宜。

### MooseFS / LizardFS

經典的 GFS 風格:單一 Master 將中繼資料放在 RAM (有 metalogger 備份),Chunkserver 儲存 64 MB 的 chunk。MooseFS Pro 透過 Raft 提供 HA master。

- **擅長**:簡潔、POSIX 語意、快照、在中小規模 (百 TB 至低 PB) 的混合工作負載下表現良好。
- **不擅長**:master 記憶體是命名空間上限 (與 HDFS 類似)、MooseFS 的 HA 僅在 Pro/付費版,LizardFS fork 實質上已死 (最後一個有意義的版本是 2017 年)。
- **TCO 驅動因素**:中小規模下維運成本低;想要 HA 就付錢買 Pro。對小型團隊來說是「無聊但可靠」的好選擇。

### Lustre

HPC 原生。MDS (中繼資料) 與 OSS (物件儲存伺服器,內含 OST) 分離;客戶端將檔案 stripe 到多個 OST 上做平行 I/O。專為 InfiniBand/RDMA、大規模平行 I/O 設計。

- **擅長**:極致的聚合頻寬處理大型循序 I/O (頂級超級電腦可達 TB/s 等級)、緊密耦合的 HPC 工作負載。
- **不擅長**:小檔案、通用工作負載、任何雲原生應用。復原與一致性模型相較 Ceph 較為脆弱。需要專用網路與調校。
- **TCO 驅動因素**:硬體吃重 (RDMA 網路昂貴)、維運專業狹窄,但在它被設計來處理的 HPC 工作負載上無人能敵。

### BeeGFS

概念上類似 Lustre — 中繼資料與儲存服務分離 — 但部署與調校較容易。分散式中繼資料 (不像 Lustre 傳統上的單一 MDT,雖然 Lustre DNE 已解決此問題)。

- **擅長**:HPC 與 ML 訓練叢集,想要 Lustre 等級的吞吐量但不想承受 Lustre 的維運痛苦。免費使用;企業支援需付費。
- **不擅長**:社群比 Lustre/Ceph 小、整合較少、在某些邊緣情況下 POSIX 不夠完美。

### SeaweedFS

受 Haystack 啟發 (Facebook 的照片儲存論文)。Master 追蹤 volume、volume server 以 needle 打包儲存 blob、Filer 透過外部中繼資料儲存提供可選的類 POSIX 層。

- **擅長**:大規模的小檔案工作負載 — 數十億個物件且中繼資料負擔極低。S3 閘道、雲端分層儲存。
- **不擅長**:維運成熟度不如 Ceph/MinIO、生態系較小、企業整合較少。

---

## 跨系統比較表

| 系統        | 儲存形式                | 中繼資料模型                  | POSIX        | EC 支援           | 最佳規模點                    |
|-------------|-------------------------|-------------------------------|--------------|-------------------|-------------------------------|
| Ceph        | 區塊 / 檔案 / 物件      | 分散式 (CRUSH + MDS)          | 是 (CephFS)  | 成熟              | PB 到 EB                      |
| GlusterFS   | 檔案                    | 演算法式 (DHT)                | 是           | 是 (dispersed)    | 100 TB – PB                   |
| HDFS        | 僅 append 檔案          | 集中式 NameNode (HA)          | 否           | 是 (3.0+)         | PB,分析用                    |
| MinIO       | 物件                    | 嵌入於每個 erasure set        | 否           | 是 (原生)         | TB 到 EB                      |
| JuiceFS     | 檔案 (POSIX/S3/HDFS)    | 外部資料庫                    | 是           | 透過後端          | PB (受資料庫限制)             |
| MooseFS     | 檔案                    | 集中式 master                 | 是           | 否                | 次 PB                         |
| Lustre      | 檔案                    | MDS + OST                     | 部分         | 否                | EB (HPC)                      |
| BeeGFS      | 檔案                    | 分散式中繼資料                | 是           | buddy mirror      | PB (HPC/ML)                   |
| SeaweedFS   | 物件 (+ filer)          | 集中式 master                 | 透過 filer   | 是                | 數十億個小物件                |

---

## TCO:直話直說

總成本通常由三件事主導,依重要性排序:**維運人力**、**硬體效率**、**網路與電力**。在開源選擇中,軟體授權極少是成本驅動因素。

- **維運人力** 是 Ceph 與 Lustre 真正折磨人的地方 — 它們需要資深人員。MinIO、MooseFS、SeaweedFS 屬於低維運端。JuiceFS 則把負擔轉嫁到資料庫維運。
- **硬體效率** 取決於 EC 與副本的選擇。在 PB 規模下,EC:8+3 之類的設定相較 3 倍副本可省下數百萬。Ceph、MinIO、HDFS、GlusterFS dispersed、SeaweedFS、BeeGFS 都支援 EC。只支援 3 倍副本的系統在大規模下昂貴。
- **復原經濟學** 重要但常被忽視。寬條帶 EC (例如 RS-10-4) 省空間,但讓重建成本倍增 — 故障後的數小時或數天內,復原 I/O 會與客戶端 I/O 競爭頻寬。Ceph 的 PG-based 復原、MinIO 的 per-set 復原、HDFS 的 block-level 復原行為各不相同。

---

## 依場景選擇

- **大型不可變檔案的分析** (parquet、ORC、日誌) — 答案越來越是物件儲存 (地端用 MinIO,雲端用 S3) 加上 Trino/Spark/DuckDB,而非 HDFS。除非你已深陷 Hadoop 且遷移不划算,否則不需要 HDFS。
- **PB 規模、需要強一致性的混合 POSIX 工作負載** — 若有足夠維運能量則選 Ceph;若可以仰賴託管物件儲存 + 託管資料庫,則選 JuiceFS。
- **AI/ML 訓練資料** — 用 MinIO 或 S3 作為資料湖、JuiceFS 或 BeeGFS 作為高吞吐量訓練檔案系統,若需要快取層可在前面加 Alluxio。
- **HPC** — 若有基礎設施與專業,選 Lustre;若想以較低維運成本獲得大部分好處,選 BeeGFS。
- **數十億個小檔案** (縮圖、IoT 事件、感測器資料) — 選 SeaweedFS 或 MinIO,絕對不要 CephFS 或 GlusterFS。
- **VM / Kubernetes 底下的區塊儲存** — Ceph RBD 是唯一成熟的大規模開源答案;較小的 K8s 部署可考慮 Longhorn。
- **歸檔 / 冷儲存** — MinIO 搭配 EC + 生命週期管理分層到磁帶/類 Glacier 儲存,或 Ceph 配上冷層。

---

## 結語

最誠實的建議:大多數團隊高估自己對 POSIX 的需求,並低估自身的維運能量。一個 S3 API 的物件儲存能解決的問題比大家想像中多;而那些試圖把所有功能做齊的系統 (例如 Ceph) 雖然優秀,但要把它運行得好非常昂貴。

> 選擇能解決工作負載的最簡單方案,並編列維運團隊的預算以匹配你選擇的系統。
