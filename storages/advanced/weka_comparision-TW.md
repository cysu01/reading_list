# 儲存平台比較:WEKA vs CephFS vs JuiceFS vs NetApp

| 維度 | **WEKA** | **CephFS** | **JuiceFS** | **NetApp (ONTAP)** |
|---|---|---|---|---|
| **類型** | 專有並行檔案系統 | 開源分散式檔案系統 (LGPL) | 建構於物件儲存之上的開源檔案系統 (Apache 2.0;Enterprise 版本為閉源) | 專有企業級 NAS/SAN |
| **架構** | 使用者空間 DPDK + NVMe、分散式 erasure coding、NVMe + S3 冷層 | RADOS 物件儲存,搭配 MDS 層提供 POSIX;複寫或 EC | 中介資料引擎 (Redis/TiKV/等) + S3 相容物件儲存作為資料層 | 雙控制器設備 (AFF/FAS) 或雲端版 ONTAP;WAFL 檔案系統 |
| **協定** | POSIX、NFS、SMB、S3、GPUDirect | POSIX (CephFS),同一叢集亦可提供 RBD/RGW | POSIX、S3、HDFS、CSI | NFS、SMB、S3、iSCSI、FC、NVMe-oF |
| **效能特性** | 極高 IOPS + 低延遲;專為 GPU 訓練打造 | 中等;針對容量與一般工作負載調校 | 受物件儲存延遲限制;吞吐量強,但小檔案 / 中介資料密集場景較弱 | 高且可預期;AFF 全快閃在許多工作負載上可與 WEKA 競爭 |
| **橫向擴充模型** | 線性,可達數百節點 | 線性,可達數千 OSD | 隨物件儲存擴充;中介資料是瓶頸 | 每個 HA 對為 scale-up;NAS 叢集最多可擴展至 24 節點 |
| **最適用途** | AI/ML 訓練、HPC、量化交易、基因體學 | 通用私有雲、OpenStack、混合 block/file/object | 雲原生應用、資料湖、需 POSIX 存取的歸檔 | 企業 IT、VMware、資料庫、法規/合規嚴格的環境 |
| **優點** | AI 領域最快;單一命名空間整合快閃 + 物件;GPUDirect | 免費;單一叢集可提供 block/file/object;龐大社群 | 核心免費;可運行於任何雲端的低成本物件儲存上;部署極簡 | 成熟、穩定、生態系深厚 (SnapMirror、FlexClone、FabricPool);強大的技術支援 |
| **缺點** | 昂貴;閉源;廠商鎖定 | 維運複雜;效能調校困難;MDS 是已知痛點 | 物件儲存的延遲下限;中介資料 DB 是額外的活動元件;小檔案表現不佳 | 昂貴;AFF/FAS 有硬體鎖定;在採用 GPUDirect 等級功能上較慢 |
| **授權 / 取得模式** | 訂閱制,依可用 TB 計費 | 免費 (Red Hat / SUSE / Canonical 銷售支援服務) | 社群版:免費。Enterprise:商業訂閱 | 硬體 + ONTAP 授權,或雲端訂閱 (CVO、FSx for ONTAP) |
| **每 TB 成本 (軟體/授權)** | 約 $80–200/TB/年 可用容量 | $0 軟體;企業支援約 $30–80/TB/年 | 社群版 $0;Enterprise 約 $30–100/TB/年 | 內含於設備中;AFF 前期成本實際約 $300–800/TB 可用容量 |
| **每 TB 成本 (原始基礎設施)** | NVMe 伺服器:約 $100–250/TB raw | HDD:約 $20–40/TB raw;NVMe:約 $100–250/TB | 物件儲存:約 $5–23/TB/月 (S3 Standard) 或約 $5/TB/月 (Backblaze/R2) | 內含於設備價格中 |
| **典型 3 年 TCO,1 PB 可用容量** | 高:約 $1–2M | 低至中:約 $300–600K (主要為硬體 + 維運人力) | 若純雲端則最低:約 $200–500K (主要為 S3 成本 + egress 費用) | 中至高:約 $800K–1.5M |
| **維運成本 / 人力配置** | 低 — 接近託管式體驗,廠商支援承擔大部分工作 | 高 — 需要專職的 Ceph 專業人員;調校是持續性的工作 | 低至中 — 容易維運,但需自行管理中介資料 DB 的 HA | 低 — 「就是能用」;廠商支援強大 |
| **雲原生部署** | 是 (AWS/GCP/Azure/OCI marketplace) | 可行但不常見;在地端表現較佳 | 原生 — 專為雲端物件儲存設計 | 是 — FSx for ONTAP (AWS)、CVO (所有雲) |
| **鎖定風險** | 高 (專有格式 + 協定) | 無 (開放資料格式、開源程式碼) | 低 (資料存放於標準物件儲存,採 JuiceFS chunk 格式) | 高 (專有 WAFL,但匯出工具成熟) |
| **適用情境…** | 需要 GPU 叢集的最高效能且預算充足 | 需要彈性且願意自行維運 | 需要雲端最便宜的 POSIX 且可容忍延遲 | 需要企業級混合工作負載的穩定可靠性 |

> 成本數字為 2026 年初的粗略公開定價估計;實際採購可議價,街頭價通常比定價低 30–50%。

## S3 支援:WEKA vs MinIO

MinIO 是 S3 優先的物件儲存;WEKA 是並行檔案系統,並提供原生 S3 協定介面。兩者都支援 S3,但架構、效能範疇與維運模型差異甚大。

| 維度 | **WEKA S3** | **MinIO** |
|---|---|---|
| **產品定位** | 並行檔案系統 (WekaFS),S3 為其多種協定介面之一 | 純 S3 相容物件儲存 |
| **S3 實作** | 運行於 WEKA 叢集節點上的原生協定服務;非閘道器 | 原生 S3 伺服器;唯一支援的協定 |
| **後端儲存** | NVMe 上的 WekaFS 分散式命名空間,可選擇分層至外部物件儲存 | 直接寫入本機磁碟的 erasure-coded 分片 (通常為 JBOD 上的 XFS) |
| **統一檔案 + 物件** | ✅ 同一份位元組可同時透過 POSIX/NFS/SMB **以及** S3 存取 | ❌ 僅支援 S3 (內部為 XFS 上的 KV;無 POSIX 視圖) |
| **S3 API 涵蓋範圍** | 核心 API、multipart、versioning、lifecycle、Object Lock/WORM、IAM 風格政策、presigned URL、稽核記錄、tagging、CORS | 廣泛的 S3 相容性:SigV4、multipart、versioning、lifecycle、Object Lock、replication (站點/儲存桶)、notifications、STS、bucket policies、透過 KES 的 SSE-KMS |
| **多站點複寫** | 透過底層 WekaFS snap-to-object 或外部協調 | 一級公民等級的 active-active 與 active-passive 儲存桶複寫 |
| **加密** | 透過 WEKA 進行靜態加密;S3 路徑上的 SSE | SSE-S3、SSE-C、透過 KES 的 SSE-KMS;每儲存桶金鑰 |
| **位元腐蝕偵測 / 修復** | WekaFS 分散式資料保護 (N+2/N+4) + scrubbing | 每分片的 HighwayHash + erasure coding;`mc admin heal`;背景掃描器 (機率式,例如 `healObjectSelectProb`) |
| **小物件延遲** | 次毫秒至低毫秒;專為數十億小檔案設計 | NVMe 上數毫秒,HDD 較高;對約 128 KiB 以下的物件有 inline-data 最佳化 |
| **大規模小物件 IOPS** | 極佳 — 中介資料服務是架構重點 | 良好但受 erasure-set fan-out 限制 (每次 PUT 需 N×中介資料寫入) |
| **大物件吞吐量** | 在 NVMe + RDMA 上表現極佳 | 極佳 — 輕鬆飽和網路卡與磁碟;針對大型平行傳輸調校完善 |
| **超大儲存桶的列表 / 掃描** | 快速 — 採 FS 風格的中介資料索引 | 每個版本持續改善;非常大的扁平命名空間仍比 WEKA 成本高 |
| **硬體規格** | NVMe 為主,通常為 100 GbE / InfiniBand,核心旁路 (DPDK) | 商規硬體;HDD 或 SSD;標準乙太網路 |
| **部署模型** | 設備或部署於認證伺服器上的軟體;雲端 marketplace 映像 | 單一靜態執行檔;可運行於任何地方 — 裸機、VM、容器、K8s |
| **Kubernetes / 雲原生** | CSI 驅動;雲端 marketplace 部署 | 一級公民等級的 K8s operator;Helm charts;常見於 EKS/GKE/AKS 與地端 K8s |
| **分層至外部物件儲存** | 內建冷層,支援 AWS S3 / Azure Blob / GCS / S3 相容儲存 (包含 MinIO 本身) | ILM 分層至 AWS S3 / Azure / GCS / 其他 MinIO 叢集 |
| **授權** | 專有,依可用 TB 訂閱 | 開源 (AGPLv3) + 商業 (Enterprise / SUBNET) |
| **每 TB 成本 (粗估)** | 約 $80–200/TB/年 軟體 + 約 $100–250/TB 原始 NVMe | $0 軟體 (社群版);商業訂閱分級計費;商規 HDD/SSD 約 $20–80/TB raw |
| **維運模型** | 廠商支援,接近託管式體驗 | 自行維運 (社群版) 或廠商支援 (Enterprise) |
| **鎖定** | 高 — 專有 FS 格式 | 低 — 標準 S3 API;資料為 XFS 上極為標準的 erasure-coded 檔案 |
| **最適用途** | AI/ML 訓練、HPC、混合 POSIX+S3 管線、延遲敏感的小檔案工作負載 | 雲原生 S3 工作負載、資料湖、備份、K8s 應用儲存、日誌/事件歸檔 |
| **適用情境…** | 需要 NVMe 等級的 S3 延遲,且需透過 POSIX 存取 *相同資料* 供 GPU dataloader 使用 | 需要在商規硬體上運行的開放、可攜、S3 優先的物件儲存 |

### WEKA 的 S3 何時值得付出溢價

- 每秒數百萬個小物件且有 P99 延遲要求 (AI 訓練、特徵儲存)
- 需要同一份資料集同時透過 POSIX *與* S3 存取且不複製
- GPU 叢集中 dataloader 偏好檔案,但管線寫入物件

### MinIO 何時是正確選擇

- 純 S3 工作負載 — 無 POSIX 需求
- 重視開源與可攜性 (可於筆電、K8s、裸機、任何雲端上以相同方式運行)
- 對成本敏感、部署於商規 HDD/SSD 的環境
- 受吞吐量限制而非小物件延遲限制

### TL;DR

兩者都是有效的 S3 端點。**WEKA 在小物件延遲與統一檔案+物件存取上勝出**;**MinIO 在開放性、可攜性與商規硬體成本上勝出**。兩者常為互補關係 — WEKA 在前作為熱層,MinIO 在後作為冷層。
