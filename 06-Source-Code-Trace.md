# 06. Source Code Trace
## 本章目標

本章的目標是從架構層面的理解進一步進入 **OAI source code**，實際追蹤一條 MAC 到 PHY 的執行流程，確認 Scheduler 如何產生 FAPI structures，以及這些 structures 最後如何被 OAI PHY 或 NVIDIA Aerial L1 使用。
## 學習紀錄
- **學習日期：** 2026/08/24
- **投入時間：** 約 4 小時
- **OAI Branch / Tag：** `develop`
- **主要閱讀內容：** OAI MAC Scheduler Source Code、FAPI Structures、L1 TX/RX Flow、Aerial FAPI Split
- **Source Trace Target：** `Scheduler → DL_TTI.request → PHY`
- **本次重點：** 從 OAI source code 實際追蹤 `NR_slot_indication()`、`gNB_dlsch_ulsch_scheduler()`、`nr_schedule_ue_spec()` 與 FAPI structures，理解 MAC scheduling result 最後如何被 OAI PHY 或 NVIDIA Aerial L1 使用。
## 6.1 Goal

前面的筆記主要是在理解：

```text
OCUDU Architecture
        ↓
OAI Architecture
        ↓
MAC Scheduler
        ↓
FAPI
        ↓
NVIDIA Aerial L1
```

到了這一章，我希望開始從 architecture 進一步走到 source code。

目前先選一條 Downlink path 進行追蹤：

```text
Slot Timing
    ↓
OAI MAC Scheduler
    ↓
DL Scheduling
    ↓
FAPI Structures
    ↓
OAI PHY / NVIDIA Aerial L1
```

這一章的目標不是一次看完整個 OAI repository，而是先回答：

> 一個 scheduling decision 是如何從 OAI MAC 一路傳到 Layer 1？

---

## 6.2 Version Note

OAI 是持續更新中的 project，因此 source code path 可能會隨 branch 或 tag 改變。

本章以目前 OAI `develop` branch 的 software architecture 為主要參考。

閱讀 source code 時，需要同時確認：

```text
Documentation
     ↓
Branch / Tag
     ↓
Source Code
```

三者是否一致。

舊文章中的 function flow 不一定完全適用目前版本。

---

## 6.3 Source Code Map

這一章主要會接觸以下幾個位置：

```text
openairinterface5g/
│
├── executables/
│   └── nr-gnb.c
│
├── openair2/
│   └── LAYER2/
│       └── NR_MAC_gNB/
│           ├── gNB_scheduler.c
│           ├── gNB_scheduler_dlsch.c
│           ├── gNB_scheduler_ulsch.c
│           ├── gNB_scheduler_primitives.c
│           └── nr_mac_gNB.h
│
├── openair1/
│   └── SCHED_NR/
│       ├── sched_nr.h
│       └── phy_procedures_nr_gNB.c
│
├── nfapi/
│
└── doc/
    ├── MAC/
    │   └── scheduler-architecture.md
    └── SW-archi-graph.md
```

可以先分成：

| Directory     | Role                       |
| ------------- | -------------------------- |
| `executables` | gNB runtime / threads      |
| `openair2`    | MAC / Scheduler            |
| `nfapi`       | FAPI definitions           |
| `openair1`    | OAI PHY                    |
| `doc`         | Architecture documentation |

---

## 6.4 Current OAI L1 Threading Flow

目前 OAI gNB 的 L1 software architecture 中，主要有 TX 與 RX processing threads。

Downlink / TX side 可以先簡化成：

```text
ru_thread
    ↓
L1_tx_thread
    ↓
tx_func()
    ↓
NR_slot_indication()
    ↓
MAC Scheduler
    ↓
PHY TX Processing
    ↓
Radio
```

其中 `L1_tx_thread()` 會執行：

```text
tx_func()
```

而 `tx_func()` 會先透過：

```text
NR_slot_indication()
```

通知 MAC：

> 現在進入新的 slot，可以開始進行 scheduling。

這可以對應到前一章提到的：

```text
SLOT.indication
```

概念。

---

## 6.5 NR_slot_indication()

目前 `NR_slot_indication()` 會依 deployment mode 決定如何處理 Layer 2。

在 monolithic mode 中，可以簡化成：

```text
NR_slot_indication()
        ↓
run_scheduler_monolithic()
        ↓
OAI MAC Scheduler
```

而使用 nFAPI split 時，slot indication 可以傳送給另一端的 MAC / L2。

因此這個 abstraction 讓 OAI 可以支援不同 MAC-PHY deployment。

概念：

```text
                    NR_slot_indication()
                           │
              ┌────────────┴────────────┐
              │                         │
        Monolithic                  Split Mode
              │                         │
     Local Scheduler              Send Slot Indication
```

---

## 6.6 MAC Scheduler Entry

OAI gNB MAC Scheduler 的核心流程主要位於：

```text
openair2/LAYER2/NR_MAC_gNB/
```

重要主函式：

```c
gNB_dlsch_ulsch_scheduler()
```

可以先把它理解成：

> 一個 slot 中 MAC scheduling 的主要 orchestration function。

概念：

```text
gNB_dlsch_ulsch_scheduler()
        │
        ├── Common Scheduling
        │
        ├── UL Scheduling
        │
        └── DL Scheduling
```

其中兩個重要 functions 是：

```c
nr_schedule_ulsch()
```

以及：

```c
nr_schedule_ue_spec()
```

---

## 6.7 UL Scheduler

Uplink scheduling 主要由：

```c
nr_schedule_ulsch()
```

處理。

OAI Scheduler 會先安排 Uplink，再進行 Downlink scheduling。

簡化：

```text
gNB_dlsch_ulsch_scheduler()
        ↓
nr_schedule_ulsch()
        ↓
nr_schedule_ue_spec()
```

其中 UL scheduling 需要安排 future UL slot。

例如：

```text
Current DL Slot
      ↓
Generate UL Grant
      ↓
      K2
      ↓
Future UL Slot
      ↓
UE transmits PUSCH
```

因此 UL Scheduler 的 output 最後會形成對應的 UL FAPI configuration。

---

## 6.8 DL Scheduler

Downlink UE scheduling 可以從：

```c
nr_schedule_ue_spec()
```

開始追。

主要位置：

```text
openair2/LAYER2/NR_MAC_gNB/gNB_scheduler_dlsch.c
```

Scheduler 會根據：

```text
UE Context
+
RLC Buffer
+
Channel State
+
HARQ State
+
Available Resources
```

進行 scheduling。

目前 Scheduler pipeline 可以簡化成：

```text
Candidate UE
     ↓
RI / PMI
     ↓
Beam Selection
     ↓
Time Domain Allocation
     ↓
MCS Selection
     ↓
RB Allocation
     ↓
Final Scheduling Result
```

---

## 6.9 From RLC to MAC

在 Downlink 中，真正的 user data 在被 transmission 之前會先存在 RLC side。

概念：

```text
PDCP
 ↓
RLC
 ↓
RLC Buffer
```

MAC 需要先知道：

> RLC 有多少資料等待傳送？

再由 Scheduler 決定：

```text
How much radio resource
```

應該分配給這個 UE。

因此可以簡化成：

```text
RLC Buffer
     ↓
MAC reads buffer status
     ↓
Scheduler allocates resources
     ↓
MAC obtains data
     ↓
Build MAC PDU
```

---

## 6.10 Scheduling Decision

假設某一個 UE 最後被 Scheduler 選中。

Scheduler 可能決定：

```text
UE1

PRB   = allocated resource
MCS   = selected MCS
HARQ  = selected process
TDA   = selected time-domain allocation
Beam  = selected beam
```

這些仍然是：

> MAC-level scheduling decision。

PHY 不能直接讀 Scheduler 裡面的演算法。

因此下一步需要把這些 decision 填入：

```text
FAPI structures
```

---

## 6.11 Scheduler Output

OAI 使用 scheduling response structures 保存 MAC Scheduler 的結果。

其中重要資訊包括：

```text
DL_req
TX_req
UL_dci_req
UL_tti_req
```

可以先簡化理解：

| Structure    | Purpose                           |
| ------------ | --------------------------------- |
| `DL_req`     | Downlink PHY configuration        |
| `TX_req`     | Downlink payload                  |
| `UL_dci_req` | Uplink grant / DCI                |
| `UL_tti_req` | Future UL reception configuration |

因此：

```text
MAC Scheduler
      ↓
Scheduling Result
      ↓
FAPI Structures
```

---

## 6.12 DL_TTI.request

Downlink configuration 主要透過：

```c
nfapi_nr_dl_tti_request_t
```

表示。

也就是前面提到的：

```text
DL_TTI.request
```

它不是單純代表 PDSCH。

一個 Downlink slot 中可能包含：

```text
DL_TTI.request
│
├── PDCCH PDU
├── PDSCH PDU
├── SSB PDU
└── CSI-RS PDU
```

Layer 1 會根據：

```text
PDU Type
```

決定需要執行哪一種 PHY processing。

---

## 6.13 PDSCH PDU

如果 Scheduler 要安排 UE Downlink data，會產生對應的 PDSCH configuration。

PDSCH PDU 中會包含與 transmission 有關的資訊，例如：

```text
BWP
PRB allocation
MCS
HARQ
Symbol allocation
DMRS
Layer configuration
Precoding / Beam information
```

因此可以理解成：

```text
MAC Scheduler Decision

PRB
MCS
HARQ
TDA
        ↓
PDSCH FAPI PDU
```

也就是將 Scheduler 的 decision 轉換成 PHY 可以使用的標準 structure。

---

## 6.14 TX_DATA.request

`DL_TTI.request` 主要描述：

> PHY 應該怎麼傳。

但 Layer 1 還需要知道：

> 真正要傳什麼資料。

因此另外有：

```c
nfapi_nr_tx_data_request_t
```

也就是：

```text
TX_DATA.request
```

最簡單可以記：

```text
DL_TTI.request
=
How to transmit
```

而：

```text
TX_DATA.request
=
What to transmit
```

兩者共同組成 Downlink transmission 所需要的重要資訊。

---

## 6.15 Downlink Scheduler to FAPI

目前可以把 MAC side 的 Downlink path 整理成：

```text
RLC Buffer
    ↓
nr_schedule_ue_spec()
    ↓
Candidate Selection
    ↓
MCS / TDA / RB Allocation
    ↓
HARQ / DCI
    ↓
PDCCH / PDSCH Configuration
    ↓
DL_req
+
TX_req
    ↓
FAPI
```

到這裡，Layer 2 的主要工作基本完成。

接下來就是 Layer 1。

---

## 6.16 OAI Native PHY Path

如果使用 OAI 原生 Layer 1，PHY TX processing 可以看到：

```c
phy_procedures_gNB_TX()
```

位於：

```text
openair1/SCHED_NR/phy_procedures_nr_gNB.c
```

目前 function interface 會直接接收：

```c
const nfapi_nr_dl_tti_request_t *DL_req
```

```c
const nfapi_nr_tx_data_request_t *TX_req
```

以及：

```c
const nfapi_nr_ul_dci_request_t *UL_dci_req
```

因此可以直接看到：

```text
MAC Scheduler
      ↓
FAPI Structures
      ↓
phy_procedures_gNB_TX()
```

這是理解 OAI MAC-PHY interface 很重要的一個 source-code evidence。

---

## 6.17 PHY Reads FAPI PDU

OAI PHY 收到：

```text
DL_req
```

後，會依照不同 PDU type 執行不同 PHY processing。

概念：

```text
DL_req
   ↓
Check PDU Type
   │
   ├── PDCCH
   ├── PDSCH
   ├── SSB
   └── CSI-RS
```

例如 PDCCH 最後需要產生實際的 DCI physical signal。

PDSCH 則需要同時使用：

```text
PDSCH Configuration
+
Transport Block
```

才能執行真正的 Downlink PHY processing。

---

## 6.18 Native OAI Downlink Trace

因此 OAI monolithic Downlink path 可以整理成：

```text
L1 Timing
    ↓
NR_slot_indication()
    ↓
MAC Scheduler
    ↓
gNB_dlsch_ulsch_scheduler()
    ↓
nr_schedule_ue_spec()
    ↓
Scheduling Decision
    ↓
DL_req + TX_req
    ↓
FAPI
    ↓
phy_procedures_gNB_TX()
    ↓
OAI PHY
    ↓
Radio
    ↓
UE
```

這就是目前從 Layer 2 追到 OAI native Layer 1 的主要路徑。

---

## 6.19 NVIDIA Aerial Path

如果 Layer 1 改成 NVIDIA Aerial：

```text
OAI MAC
   ↓
FAPI
   ↓
OAI PHY
```

會變成：

```text
OAI MAC
   ↓
FAPI
   ↓
Aerial Transport
   ↓
nvIPC
   ↓
NVIDIA Aerial L1
```

重要的是：

> Scheduler 邏輯仍然位於 OAI Layer 2。

改變的是 FAPI 後面的：

```text
Transport
+
PHY Implementation
```

---

## 6.20 OAI to Aerial Trace

因此使用 NVIDIA Aerial 時，可以簡化成：

```text
OAI Scheduler
      ↓
DL_req
+
TX_req
      ↓
FAPI
      ↓
Aerial Southbound Transport
      ↓
nvIPC

=========== OAI / Aerial Boundary ===========

      ↓
Aerial L2 Adapter
      ↓
cuPHY Driver
      ↓
cuPHY
      ↓
CUDA / GPU
      ↓
O-RAN Fronthaul
      ↓
O-RU
```

這表示：

```text
Scheduling Algorithm
```

與：

```text
PHY Implementation
```

可以透過 FAPI interface 解耦。

---

## 6.21 Uplink Receive Path

OAI 的 RX side 與 TX side 不完全相同。

目前 L1 RX thread 可以簡化成：

```text
L1_rx_thread
      ↓
rx_func()
      ↓
PHY RX Processing
      ↓
phy_procedures_gNB_uespec_RX()
      ↓
NR_UL_indication()
      ↓
MAC
```

PHY 會先處理：

* PUSCH
* PUCCH
* SRS
* PRACH

然後產生對應的 FAPI Uplink indications。

例如：

```text
RX_DATA.indication
CRC.indication
UCI.indication
RACH.indication
```

---

## 6.22 Uplink FAPI Flow

因此 Uplink 可以整理成兩個階段。

### Scheduling

```text
OAI MAC
   ↓
nr_schedule_ulsch()
   ↓
UL_TTI.request
   ↓
PHY
```

### Reception

```text
UE
 ↓
PHY
 ↓
PUSCH / PUCCH Processing
 ↓
RX_DATA.indication
CRC.indication
UCI.indication
 ↓
NR_UL_indication()
 ↓
OAI MAC
```

所以 MAC-PHY communication 是雙向的：

```text
MAC
 ↓
FAPI Request
 ↓
PHY

PHY
 ↓
FAPI Indication
 ↓
MAC
```

---

## 6.23 Why Slot Timing Is Important

從 source code trace 可以看到：

```text
Frame
+
Slot
```

幾乎存在於整條 scheduling / PHY pipeline。

例如：

```text
Slot Timing
    ↓
Scheduler
    ↓
FAPI Request
    ↓
PHY Processing
    ↓
Radio Transmission
```

因此 FAPI message 不只是內容必須正確。

也必須：

> 在正確的 slot deadline 前到達 Layer 1。

這也是 real-time RAN 與一般 software system 很不一樣的地方。

---

## 6.24 A Better Way to Read OAI Source Code

OAI repository 很大，因此目前我不打算逐行閱讀所有 source code。

我會先針對每一個重要 function 回答四個問題：

```text
1. Input 是什麼？
2. 這個 function 做什麼？
3. 它修改哪些 structure？
4. Output 最後去哪裡？
```

例如：

```text
Function:
nr_schedule_ue_spec()

Location:
openair2/LAYER2/NR_MAC_gNB/gNB_scheduler_dlsch.c

Layer:
MAC / Layer 2

Main Job:
Downlink UE Scheduling

Input:
UE / Slot / Resource State

Output:
Downlink scheduling information
```

這樣比較容易理解大型 codebase 中的 data flow。

---

## 6.25 Useful Search Commands

之後閱讀 OAI source code 時，可以直接使用 `grep` 找 function 或 structure。

找 Scheduler：

```bash
grep -R "gNB_dlsch_ulsch_scheduler" -n openair2/
```

找 Downlink Scheduler：

```bash
grep -R "nr_schedule_ue_spec" -n openair2/
```

找 Uplink Scheduler：

```bash
grep -R "nr_schedule_ulsch" -n openair2/
```

找 Slot Indication：

```bash
grep -R "NR_slot_indication" -n .
```

找 DL FAPI structure：

```bash
grep -R "nfapi_nr_dl_tti_request_t" -n .
```

找 TX Data：

```bash
grep -R "nfapi_nr_tx_data_request_t" -n .
```

找 OAI PHY TX function：

```bash
grep -R "phy_procedures_gNB_TX" -n .
```

找 Aerial configuration：

```bash
grep -R "\"aerial\"" -n .
```

找 nvIPC：

```bash
grep -R "nvipc" -n .
```

這比直接從 repository 第一個檔案開始閱讀有效很多。

---

## 6.26 Current Downlink Source Trace

目前我可以把 Downlink source path 整理成：

```text
L1_tx_thread
     ↓
tx_func()
     ↓
NR_slot_indication()
     ↓
run_scheduler_monolithic()
     ↓
gNB_dlsch_ulsch_scheduler()
     ↓
nr_schedule_ue_spec()
     ↓
Scheduling Decision
     ↓
DL_req
+
TX_req
     ↓
FAPI
```

如果使用 OAI PHY：

```text
FAPI
 ↓
phy_procedures_gNB_TX()
 ↓
OAI PHY
```

如果使用 NVIDIA Aerial：

```text
FAPI
 ↓
Aerial Transport
 ↓
nvIPC
 ↓
Aerial L2 Adapter
 ↓
cuPHY Driver
 ↓
cuPHY
 ↓
GPU
```

---

## 6.27 What Changed From Architecture-Level Understanding?

一開始我的理解只有：

```text
MAC
 ↓
FAPI
 ↓
PHY
```

現在則可以進一步指出：

```text
MAC Scheduler
      ↓
nr_schedule_ue_spec()
      ↓
Scheduling Decision
      ↓
nfapi_nr_dl_tti_request_t
+
nfapi_nr_tx_data_request_t
      ↓
Layer 1
```

因此 FAPI 對我來說不再只是：

> 一個 MAC-PHY interface 名稱。

而是：

> Scheduler 將 scheduling decision 填進實際 FAPI structures，再由 Layer 1 讀取並執行 PHY processing。

---

## 6.28 What I Learned

完成這一章後，我目前的理解是：

* OAI 的 TX processing 由 L1 thread 與 slot timing 驅動。
* `NR_slot_indication()` 是 MAC scheduling 與 L1 timing 之間的重要連接點。
* `gNB_dlsch_ulsch_scheduler()` 是 OAI MAC scheduling 的主要 orchestration function。
* `nr_schedule_ulsch()` 負責 UL scheduling。
* `nr_schedule_ue_spec()` 是追 DL UE scheduling 的重要入口。
* Scheduler 最後會把 scheduling decision 寫入 FAPI structures。
* `DL_TTI.request` 主要描述 Downlink PHY configuration。
* `TX_DATA.request` 提供真正要傳送的 payload。
* OAI native PHY 可以直接使用這些 FAPI structures。
* 如果換成 NVIDIA Aerial，Scheduler 可以保留在 OAI，改變的是 FAPI 後面的 transport 與 Layer 1 implementation。
* Uplink 則會由 PHY 產生 `RX_DATA`、`CRC`、`UCI` 等 indications 回到 MAC。
* 閱讀 OAI source code 時，需要確認 branch / tag，避免使用已過時的 function flow。

目前我最重要的理解是：

```text
Architecture
     ↓
Source Code
     ↓
Function
     ↓
Structure
     ↓
Message Flow
```

也就是從「知道 MAC 和 PHY 中間有 FAPI」，進一步到：

> **知道 Scheduler 如何產生 FAPI structures，以及 Layer 1 如何使用這些 structures。**

---

## 6.29 Next Step

目前這份筆記已經從 OCUDU architecture 一路走到 OAI source code，因此接下來我不打算繼續新增大量 architecture 章節。

比較值得繼續深入的是選擇單一 message 進行更細的 trace，例如：

```text
DL_TTI.request
```

進一步確認：

```text
Where is it created?
        ↓
Which PDU is added?
        ↓
Where are PRB / MCS fields filled?
        ↓
How is TX_DATA associated with PDSCH?
        ↓
How does Layer 1 consume it?
```

或者追：

```text
UL_TTI.request
```

確認：

```text
UL Scheduler
     ↓
K2
     ↓
Future UL Slot
     ↓
PUSCH Configuration
     ↓
Layer 1 Reception
```

這會是接下來真正閱讀 OAI source code的方向。

---

## References

本章主要參考：

* OpenAirInterface — `doc/SW-archi-graph.md`
* OpenAirInterface — `doc/MAC/scheduler-architecture.md`
* OpenAirInterface — `openair2/LAYER2/NR_MAC_gNB/`
* OpenAirInterface — `openair1/SCHED_NR/sched_nr.h`
* OpenAirInterface — `openair1/SCHED_NR/phy_procedures_nr_gNB.c`
* OpenAirInterface — FAPI / nFAPI Documentation
* OpenAirInterface — Aerial FAPI Split Tutorial
* NVIDIA Aerial — L2 Adapter Documentation
* NVIDIA Aerial — cuPHY Documentation

> Note: OpenAirInterface is under active development. Function names and source-code paths should always be verified against the branch or tag used when reproducing this trace.

---

**Previous:** [05. OAI L2 + NVIDIA Aerial L1](05-OAI-L2-NVIDIA-Aerial-L1.md)

**Back to:** [README](README.md)

