# S3 Platform Strategy & Resource Allocation Report

---

## Current Status (Today)

### 🎯 Key Focus & Objectives
* **Data Cross-Platform Migration:** S3 Abstraction Layer
* **Operation Automation**

### 👥 Resource Overview
* **Total Headcount (HC):** 22 HCs
  * **HQ:** 15 HCs
  * **NASC:** 7 HCs

### 📊 Resource Allocation Matrix (Today)

| Scope | HQ HC | HQ Tasks | NASC HC | NASC Tasks |
| :--- | :---: | :--- | :---: | :--- |
| **Operation Dev** | 5 | • SRE Automation Tooling<br>• Inventory | 4 | • STLH (User Self Service) |
| **Product Dev** | 3 | • OpenResty<br>• MinIO Bug Fixes | 2 | • S3 Abstraction Layer |
| **Monitor Dev** | 6 | • MZ Migration<br>• Monitoring Stack Dev<br>• User Self Service Dashboard | 1.5 | • MZ Migration |
| **DevOps Support*** | 15* | • All fabs support<br>• DLH support | 7* | • F21 support<br>• Global Night shift |
> \* **Note on DevOps Support:** DevOps Support headcounts represent team members sharing support duties alongside their development scopes.

---

## David’s Plan (Target State)

### 🚀 Strategic Vision (90-Day Goal)

> *"先讓資料「看得見」，再讓資料「放對地方」，最後讓平台「跟得上變化」。"*

* **90-Day Target:** Transform S3 from *"not knowing where data is or why it is there"* to *"every bucket having an access profile and correct storage tiering."*

### 🔑 Key Deliverables & KPIs
1. **Cost Optimization:** Move data to the right tiers. Reduce cost per TB, delay hot tier expansion procurement, and increase cold tier utilization.
2. **Financial Accountability:** Utilize the KPI Dashboard to answer: *"Who is spending this storage budget?"*
3. **Core Technology:** Metadata Service implementation.

### 👥 Resource Overview (Target State)
* **Total Headcount (HC):** 20 HCs *(Net -2)*
  * **HQ:** 12 HCs *(Decrease of 3)*
  * **NASC:** 8 HCs *(Increase of 1)*
* **Focus Shift:** Metadata Service & Data-Driven Tiering

### 📊 Resource Allocation Matrix (David's Plan)
| Scope | HQ HC | HQ Tasks | NASC HC | NASC Tasks |
| :--- | :---: | :--- | :---: | :--- |
| **Operation Dev** | 9 | • SRE Automation Tooling<br>• Inventory<br>• STLH (User Self Service) | 0 | *None* |
| **Product Dev** | 0 | *None (Consolidated to NASC)* | 7 | • Metadata Service<br>• OpenResty<br>• S3 Gateway<br>• MinIO Bug Fixes |
| **Monitor Dev** | 2 | • Monitoring Stack Dev<br>• User Self Service Dashboard | 1 | • KPI Dashboard<br>• Data 
Heatmap<br>• Cost Model |
| **DevOps Support*** | 12 | • All fabs support<br>• DLH support | 8 | • F21 support<br>• Global Night shift |

---
## 💡 Key Changes & Takeaways

1. **Consolidation of Product Dev:** 
   Under David's plan, Product Development is fully consolidated to the **NASC team (7 HCs)** to focus on the core 
Metadata Service and S3 Gateway. Meanwhile, the **HQ team shifts its focus to Operation Dev (9 HCs)** to drive automation.
2. **Monitoring Evolution:** 
   The Monitoring scope shifts away from basic migration tasks (MZ Migration) and evolves toward delivering high-value 
business intelligence, such as the **KPI Dashboard, Data Heatmap, and Cost Model**.
