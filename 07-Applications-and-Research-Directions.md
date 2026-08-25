07. 這些模組可以拿來做什麼？從架構理解到實際應用

本章目標

前面幾章主要是在理解 OCUDU、OAI、MAC Scheduler、FAPI 與 NVIDIA Aerial L1 的架構與運作方式。

這一章想進一步思考：

當我已經知道這些模組分別負責什麼之後，實際上可以拿它們做哪些修改、實驗或研究？

目前我把可能的方向整理成：

Architecture Understanding
        ↓
Source Code Modification
        ↓
System Integration
        ↓
Performance Measurement
        ↓
Result Analysis

也就是從「知道模組是什麼」，進一步到「知道可以改什麼、量什麼，以及可以驗證什麼」。

⸻

學習紀錄

* 投入時間： 約 2.5 小時
* 主要整理內容： OCUDU、OAI MAC Scheduler、FAPI、NVIDIA Aerial L1
* 本次重點： 思考不同模組可以實際應用在哪些實驗與研究方向
* 目前優先方向： OAI MAC Scheduler → FAPI → PHY

⸻

7.1 從理解架構到實際應用

前面的學習主要是在回答：

OCUDU 的架構怎麼拆？
        ↓
OAI 的程式碼放在哪裡？
        ↓
MAC Scheduler 如何做資源配置？
        ↓
FAPI 如何連接 MAC 與 PHY？
        ↓
NVIDIA Aerial L1 負責什麼？

但如果要開始做實作或研究，還需要進一步回答：

我可以修改什麼？
        ↓
修改後會影響什麼？
        ↓
我要量測什麼？
        ↓
如何比較修改前後的結果？

因此我目前希望從單純閱讀文件，逐步往實際操作與驗證的方向前進。

⸻

7.2 OCUDU：研究 RAN Functional Split

OCUDU 可以拿來理解與研究 5G RAN 中不同功能如何進行拆分。

例如：

CU-CP
CU-UP
DU-high
DU-low
RU

這些模組可以進一步延伸出一些問題：

* 哪些功能適合放在 CU？
* 哪些功能需要靠近 RU？
* 哪些 interface 對 latency 比較敏感？
* CU / DU 分離後會增加哪些 communication overhead？
* 不同 deployment 方式會如何影響 system performance？

因此我目前把 OCUDU 的用途理解成：

OCUDU 可以作為研究 5G RAN functional split、模組部署方式，以及不同介面之間關係的架構基礎。

例如：

Centralized CU
      ↓
      F1
      ↓
Distributed DU
      ↓
      RU

可以進一步研究 CU 與 DU 分離後的 latency、deployment 與 communication 問題。

⸻

7.3 OAI：實際修改 5G gNB Software

OpenAirInterface 提供實際的 RAN source code，因此可以直接進行：

* Source code trace
* Protocol behavior observation
* Scheduling modification
* Performance experiment

目前我認為最適合先開始實作的是：

OAI MAC Scheduler

因為 Scheduler 會決定：

Which UE?
    ↓
Which PRBs?
    ↓
Which MCS?
    ↓
Which HARQ process?
    ↓
Which time-domain resources?

因此可以直接修改其中的 scheduling policy，再觀察修改前後的結果。

⸻

7.4 MAC Scheduler：做無線資源配置實驗

例如同時存在兩個 UE：

UE 1
Channel Condition = Good
Buffer = High
UE 2
Channel Condition = Poor
Buffer = High

Scheduler 需要決定如何分配有限的 radio resources。

可以先觀察原本的 Scheduler：

Original Scheduler
        ↓
UE Selection
        ↓
PRB Allocation
        ↓
MCS Selection
        ↓
Scheduling Result

接著嘗試修改其中一個較小的部分，例如：

* UE priority
* UE selection rule
* PRB allocation
* Scheduling fairness
* Resource allocation policy

再重新執行：

Original Scheduler
        ↓
Collect Results
Modified Scheduler
        ↓
Collect Results
        ↓
Compare

可以比較的項目包括：

* Throughput
* Latency
* PRB utilization
* UE fairness
* MCS distribution

因此 OAI MAC Scheduler 可以作為一個實際的 radio resource management 實驗平台。

⸻

7.5 FAPI：觀察 Scheduler 如何把決策交給 PHY

前面已經知道：

MAC Scheduler
      ↓
FAPI
      ↓
PHY

但在實作上，可以再進一步追蹤：

Scheduling Decision
        ↓
FAPI Structure
        ↓
PHY Processing

例如 Scheduler 做出：

UE = UE1
PRB = X
MCS = Y
HARQ = Z

就可以繼續確認這些資訊如何進入：

DL_TTI.request
TX_DATA.request
UL_TTI.request

因此可以做一個簡單的驗證流程：

Scheduler Decision
        ↓
Trace FAPI Message
        ↓
Check PRB / MCS / HARQ
        ↓
Confirm PHY Input

這樣可以把原本抽象的：

MAC → FAPI → PHY

變成實際可以從 source code 驗證的 data flow。

⸻

7.6 FAPI：整合不同的 Layer 1

FAPI 的另一個重要用途，是讓 MAC 與 PHY 之間有比較明確的 interface。

例如原本：

OAI MAC
   ↓
FAPI
   ↓
OAI PHY

也可以變成：

OAI MAC
   ↓
FAPI
   ↓
External PHY

因此可以進一步研究：

* MAC / PHY separation
* Different Layer 1 implementations
* FAPI compatibility
* Real-time timing
* IPC latency
* L2 / L1 integration

這也是後續整合：

OAI L2+
   +
NVIDIA Aerial L1

的重要基礎。

⸻

7.7 NVIDIA Aerial L1：研究 GPU 加速 PHY

如果把 OAI 與 NVIDIA Aerial L1 整合：

OAI MAC Scheduler
        ↓
       FAPI
        ↓
       nvIPC
        ↓
NVIDIA Aerial L1
        ↓
     cuPHY
        ↓
   NVIDIA GPU

研究問題就可以從：

Scheduler 怎麼分配資源？

進一步延伸到：

PHY processing 需要多久？
        ↓
GPU 是否能在 slot deadline 前完成？
        ↓
IPC / Data Movement 是否造成 overhead？

可以觀察：

* PHY processing latency
* GPU utilization
* Slot processing time
* nvIPC communication latency
* CPU / GPU data movement
* Throughput

因此 NVIDIA Aerial L1 可以拿來研究：

GPU acceleration 如何應用在 5G PHY，以及 GPU processing 如何滿足 real-time RAN 的時間要求。

⸻

7.8 OAI Native PHY 與 NVIDIA Aerial L1 的比較

未來如果測試環境允許，也可以進一步比較兩種 PHY implementation：

             Same / Similar Workload
                      │
            ┌─────────┴─────────┐
            ↓                   ↓
      OAI Native PHY      NVIDIA Aerial L1
            │                   │
            ↓                   ↓
       Native / CPU          GPU / cuPHY
        Processing           Processing
            └─────────┬─────────┘
                      ↓
                   Compare

可能比較：

* Processing latency
* Throughput
* CPU utilization
* GPU utilization
* Real-time deadline
* Resource consumption

這可以進一步了解：

GPU acceleration 帶來多少 processing benefit，同時又增加多少 IPC、memory transfer 與 system integration cost。

⸻

7.9 我目前最適合先做的實驗

以目前的理解，我認為第一步不適合直接從完整：

OAI L2+
   ↓
NVIDIA Aerial L1
   ↓
O-RU

開始。

比較適合先從較小範圍的 OAI MAC Scheduler 進行實驗。

Step 1：觀察 Scheduler

先不修改程式，記錄：

UE
PRB
MCS
HARQ

等 scheduling information。

Step 2：Trace FAPI

接著追蹤：

Scheduler
    ↓
DL_TTI.request
TX_DATA.request
UL_TTI.request

確認 scheduling decision 最後如何進入 FAPI structures。

Step 3：做一個小修改

例如：

Modify UE Priority

或：

Modify PRB Allocation Rule

先避免一次改動太多 Scheduler logic。

Step 4：比較結果

最後比較：

Original Scheduler
        vs.
Modified Scheduler

觀察：

* Throughput
* PRB allocation
* MCS
* UE fairness
* Latency

是否出現明顯差異。

⸻

7.10 不同模組可以做什麼？

Module	可以做的事情	可以觀察的項目
OCUDU	Functional Split / Deployment	Architecture、Latency
OAI MAC Scheduler	修改 scheduling policy	Throughput、Fairness、PRB
FAPI	Trace MAC-PHY messages	PRB、MCS、HARQ
OAI PHY	PHY processing analysis	Processing Time
NVIDIA Aerial L1	GPU-accelerated PHY	GPU Utilization、Latency
nvIPC	L2 / L1 communication	IPC Latency
O-RAN Fronthaul	DU-low / O-RU communication	Transport、Timing

因此這些模組並不是完全獨立的研究方向，而是可以串成：

5G Protocol
      +
Scheduling Algorithm
      +
Software Architecture
      +
Parallel Processing
      +
GPU Computing
      +
Real-Time System
      +
Networking

⸻

7.11 我目前的實作方向

目前我認為最適合優先深入的是：

OAI MAC Scheduler
        ↓
Scheduling Decision
        ↓
FAPI Structure
        ↓
PHY

先把這一條 path 真正從 source code 到實際執行看懂。

之後再逐步延伸：

OAI MAC Scheduler
        ↓
FAPI
        ↓
nvIPC
        ↓
NVIDIA Aerial L1
        ↓
GPU

也就是先從單一 module modification 開始，再逐步走向 L2 / L1 integration。

⸻

What I Learned

完成前面幾章的整理後，我目前理解這些模組不只是用來描述 5G RAN architecture，也可以拿來建立實際的實驗。

目前我認為可以進一步做的事情包括：

* 修改 OAI MAC Scheduler
* 比較不同 PRB allocation strategy
* Trace FAPI messages
* 驗證 scheduling decision 如何進入 PHY
* 比較不同 Layer 1 implementation
* 研究 OAI L2+ 與 NVIDIA Aerial L1 integration
* 評估 throughput、latency、fairness 與 processing time

因此接下來我的學習方向會逐漸從：

What is this module?

轉變成：

What can I modify?
        ↓
What can I measure?
        ↓
What can I compare?
        ↓
What can I verify?

這也是我接下來希望補強的實作方向。

⸻

Next Step

目前預計先從：

OAI MAC Scheduler
        ↓
Observe Scheduling Result
        ↓
Trace FAPI
        ↓
Small Scheduler Modification
        ↓
Performance Comparison

開始。

等熟悉 OAI Scheduler 與 FAPI 的實際操作後，再進一步研究：

OAI L2+
   ↓
FAPI
   ↓
nvIPC
   ↓
NVIDIA Aerial L1

的完整 integration。

⸻

References

* OCUDU Documentation
* OpenAirInterface RAN Repository
* OpenAirInterface MAC Scheduler Architecture
* OpenAirInterface FAPI / nFAPI Documentation
* OpenAirInterface Aerial FAPI Split Tutorial
* NVIDIA Aerial L1 Documentation
* Small Cell Forum — 5G FAPI Specification

本章主要整理目前根據 architecture、documentation 與 source code 閱讀後想到的實作方向。實際可執行的實驗仍需要依照使用的 OAI branch、測試環境與硬體平台進一步確認。

⸻

Previous: 06. Source Code Trace

Back to: README