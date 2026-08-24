# 04. FAPI and OAI MAC Scheduler

## 4.1 Why MAC Scheduler and FAPI Matter

前面已經知道 OAI 的 Layer 2 與 Layer 1 大致位於：

```text
openair2
   ↓
MAC / Layer 2
   ↓
FAPI
   ↓
PHY / Layer 1
   ↓
openair1
```

其中 MAC 最重要的功能之一是 **Scheduler**。

Scheduler 負責決定：

```text
Which UE?
    ↓
Which Slot?
    ↓
How many PRBs?
    ↓
Which MCS?
    ↓
Which HARQ process?
```

但 MAC 本身不執行真正的 PHY signal processing。

因此完整流程是：

```text
UE / Buffer / Channel State
          ↓
     MAC Scheduler
          ↓
 Scheduling Decision
          ↓
         FAPI
          ↓
       PHY / L1
```

這也是理解 OAI Layer 2 與外部 Layer 1 整合的核心。

---

## 4.2 Where Is the OAI MAC Scheduler?

OAI gNB MAC scheduler 主要位於：

```text
openair2/LAYER2/NR_MAC_gNB/
```

重要檔案包括：

```text
NR_MAC_gNB/
│
├── gNB_scheduler.c
├── gNB_scheduler_dlsch.c
├── gNB_scheduler_ulsch.c
├── gNB_scheduler_primitives.c
├── gNB_scheduler_dlsch_default_policies.c
├── gNB_scheduler_ulsch_default_policies.c
└── nr_mac_gNB.h
```

其中：

| File                         | 主要功能                        |
| ---------------------------- | --------------------------- |
| `gNB_scheduler.c`            | Scheduler orchestration     |
| `gNB_scheduler_dlsch.c`      | Downlink scheduling         |
| `gNB_scheduler_ulsch.c`      | Uplink scheduling           |
| `gNB_scheduler_primitives.c` | Shared scheduling functions |
| `*_default_policies.c`       | Default scheduling policies |
| `nr_mac_gNB.h`               | Scheduler data structures   |

目前閱讀 Scheduler 時，最重要的 functions 是：

```c
gNB_dlsch_ulsch_scheduler()
```

```c
nr_schedule_ulsch()
```

```c
nr_schedule_ue_spec()
```

可以先理解為：

```text
gNB_dlsch_ulsch_scheduler()
        │
        ├── nr_schedule_ulsch()
        │        ↓
        │    UL Scheduling
        │
        └── nr_schedule_ue_spec()
                 ↓
             DL Scheduling
```

---

## 4.3 When Does the Scheduler Run?

目前 OAI gNB MAC Scheduler 會在每一個可以進行 Downlink scheduling 的 slot 執行。

每次執行時，都可能需要重新決定：

```text
Which UE should be scheduled?
          ↓
Which resources are available?
          ↓
How should the resources be allocated?
```

Scheduler 同時需要處理：

* Downlink resource allocation
* Uplink grants
* PDCCH / DCI resources
* HARQ state
* UE channel information

因此它是一個具有 real-time requirement 的核心模組。

---

## 4.4 Scheduler Pipeline

目前 OAI 的 DL 與 UL Scheduler 採用類似的 pipeline。

可以簡化成：

```text
Candidate UEs
     ↓
RI / PMI / TPMI
     ↓
Beam Selection
     ↓
Time Domain Allocation
     ↓
MCS Selection
     ↓
RB Allocation
     ↓
Dispatch
```

Downlink 還會再進行 logical channel allocation。

這代表 Scheduler 不是只做：

```text
PRB Allocation
```

而是需要綜合考慮：

```text
UE State
+
Channel Condition
+
Time Resource
+
Frequency Resource
+
HARQ
+
Beam
```

最後才產生 scheduling result。

---

## 4.5 Candidate UE Selection

Scheduler 首先會確認：

> 哪些 UE 在目前這次 scheduling 中需要被考慮？

例如：

```text
UE1
RLC Buffer > 0
HARQ available
Channel information available
        ↓
Candidate
```

但如果：

```text
UE2
No pending data
        ↓
May be skipped
```

因此 Scheduler 的 input 可以簡化成：

```text
UE Context
+
Buffer Status
+
Channel Information
+
HARQ State
+
Available Radio Resources
```

---

## 4.6 MCS and Resource Allocation

對 Candidate UE 進行處理後，Scheduler 需要決定 transmission configuration。

例如：

```text
UE1

PRB  = allocated resource
MCS  = selected MCS
HARQ = selected process
TDA  = time-domain allocation
```

可以理解成：

```text
Channel Condition
      ↓
MCS Selection
      ↓
Resource Allocation
      ↓
Final Scheduling Decision
```

MCS 越高，通常可以提供較高 data rate，但也需要較好的 channel condition。

因此 Scheduler 需要在：

```text
Throughput
    ↕
Reliability
```

之間取得平衡。

---

## 4.7 HARQ

Scheduler 同時需要處理 HARQ。

如果前一次 transmission 失敗：

```text
Transmission
     ↓
NACK / CRC Failure
     ↓
HARQ
     ↓
Retransmission
```

因此 Scheduler 不能只考慮新的資料，也需要考慮：

```text
New Transmission
        vs.
Retransmission
```

這也是為什麼 HARQ state 是 MAC scheduling 中重要的資訊。

---

## 4.8 Uplink Scheduling

Uplink scheduling 主要由：

```c
nr_schedule_ulsch()
```

處理。

與 Downlink 不同的是：

> gNB 必須先告訴 UE，它未來應該在哪一個 UL slot 傳送資料。

因此會涉及：

```text
K2
```

可以簡化成：

```text
Current DL Slot
      ↓
Generate UL Grant
      ↓
      K2
      ↓
Future UL Slot
      ↓
UE sends PUSCH
```

因此 UL Scheduler 實際上是在目前的 DL slot 中，替未來的 UL slot 配置資源。

---

## 4.9 Downlink Scheduling

Downlink UE scheduling 主要可以從：

```c
nr_schedule_ue_spec()
```

開始閱讀。

概念上：

```text
RLC Buffer
     ↓
Candidate UE
     ↓
MCS
     ↓
PRB / TDA
     ↓
HARQ
     ↓
PDCCH / PDSCH Configuration
```

Scheduler 最後需要決定：

> UE 的資料應該怎麼透過 PDSCH 傳送。

但這些還只是 MAC 的 decision。

接下來需要轉成 PHY 可以使用的資訊。

---

# 4.10 What Is FAPI?

FAPI（Functional Application Platform Interface）是 Small Cell Forum 定義的 MAC-PHY interface。

它位於：

```text
Layer 2

  MAC
   │
 FAPI
   │
  PHY

Layer 1
```

FAPI 的核心作用可以理解成：

> 將 MAC Scheduler 的 scheduling decision 轉換成 PHY 可以執行的標準化 message / structure。

因此：

```text
MAC Scheduler
      ↓
PRB / MCS / HARQ / TDA
      ↓
FAPI Message
      ↓
PHY Processing
```

---

## 4.11 Important FAPI Messages

目前最值得先理解的是以下幾個 message。

### MAC → PHY

```text
DL_TTI.request
UL_TTI.request
TX_DATA.request
```

### PHY → MAC

```text
SLOT.indication
RX_DATA.indication
CRC.indication
UCI.indication
RACH.indication
```

可以先整理成：

| Message              | Direction | 簡單理解                |
| -------------------- | --------- | ------------------- |
| `DL_TTI.request`     | MAC → PHY | DL 要怎麼傳             |
| `TX_DATA.request`    | MAC → PHY | DL 真正要傳的 data       |
| `UL_TTI.request`     | MAC → PHY | UL 要準備接收什麼          |
| `SLOT.indication`    | PHY → MAC | 進入新的 slot           |
| `RX_DATA.indication` | PHY → MAC | 收到的 UL data         |
| `CRC.indication`     | PHY → MAC | UL decoding result  |
| `UCI.indication`     | PHY → MAC | HARQ-ACK / CSI / SR |
| `RACH.indication`    | PHY → MAC | PRACH detection     |

---

## 4.12 Downlink FAPI Flow

假設 Scheduler 決定：

```text
UE1

PRB  = X
MCS  = Y
HARQ = Z
```

這些 configuration 最後需要進入：

```text
DL_TTI.request
```

另外，PHY 還需要真正要送給 UE 的 payload，因此還需要：

```text
TX_DATA.request
```

所以 Downlink 可以簡化成：

```text
RLC Buffer
     ↓
MAC Scheduler
     ↓
PRB / MCS / HARQ
     ↓
┌─────────────────┐
│ DL_TTI.request  │
│ TX_DATA.request │
└────────┬────────┘
         ↓
        FAPI
         ↓
        PHY
         ↓
       PDCCH
       PDSCH
         ↓
        UE
```

最簡單的理解：

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

---

## 4.13 Uplink FAPI Flow

Uplink 則可以簡化成：

```text
MAC Scheduler
      ↓
Future UL Resource
      ↓
UL_TTI.request
      ↓
PHY
      ↓
Wait for PUSCH
      ↓
UE transmits
      ↓
PHY Decoding
      ↓
RX_DATA.indication
CRC.indication
      ↓
MAC
```

其中：

```text
UL_TTI.request
```

告訴 PHY：

> 未來某個 UL slot 中，應該在哪些 resources 上準備接收 UE transmission。

---

## 4.14 Why Does UL Scheduling Run Before DL Scheduling?

目前 OAI Scheduler 中：

```text
UL Scheduler
     ↓
DL Scheduler
```

UL scheduling 會先執行。

原因之一是 UL 與 DL 的 DCI 都需要使用 PDCCH resources，例如：

```text
CORESET
+
CCE
```

兩邊共享相同的 CCE resource map。

因此：

```text
Available CCE
     ↓
UL Scheduler
     ↓
UL DCI claims CCE
     ↓
DL Scheduler
```

也就是說，MAC Scheduler 除了管理 data resource，還要管理 control channel resource。

---

## 4.15 P5 and P7

FAPI specification 中還會看到：

```text
P5
P7
```

這裡目前只需要先建立基本概念。

### P5

主要負責：

```text
PHY Configuration
+
Management
```

例如 PHY initialization / configuration。

### P7

主要負責 runtime scheduling 與 data exchange，例如：

```text
DL_TTI.request
UL_TTI.request
TX_DATA.request
RX_DATA.indication
```

對目前研究：

```text
OAI MAC
   ↓
FAPI
   ↓
PHY
```

而言，**P7 是比較重要的部分**。

---

## 4.16 FAPI and nFAPI

FAPI 與 nFAPI 不完全一樣。

可以先簡化為：

```text
FAPI
=
MAC-PHY message/interface
```

而：

```text
nFAPI
=
讓 MAC 與 PHY 可以進一步透過 transport/network 分離
```

例如：

```text
Host A

MAC / L2
   │
 nFAPI
   │
========= Network =========
   │
 nFAPI
   │
PHY / L1

Host B
```

OAI 本身即使在 monolithic gNB 中，也會使用對應的 FAPI messages。

因此 functional split 之後，可以重複利用這些 MAC-PHY message structures，再使用不同 transport 將它們送到 PHY。

---

## 4.17 FAPI Is Not the Transport

這裡有一個很重要的區別。

例如：

```text
DL_TTI.request
```

是一個 FAPI message。

但 FAPI message 實際上可以利用不同 transport mechanism 傳送。

因此：

```text
FAPI
=
What MAC and PHY exchange
```

而：

```text
Transport
=
How the information is delivered
```

這個概念在下一章：

```text
OAI L2
   ↓
FAPI
   ↓
NVIDIA Aerial L1
```

會非常重要。

---

## 4.18 From Scheduler to PHY

目前可以把整章濃縮成：

```text
UE / RLC / Channel State
          ↓
     OAI MAC Scheduler
          ↓
Candidate Selection
          ↓
MCS / TDA / RB Allocation
          ↓
HARQ / DCI
          ↓
     FAPI Structures
          ↓
 ┌────────┴─────────┐
 ↓                  ↓
DL_TTI.request   UL_TTI.request
TX_DATA.request
          │
          ↓
       PHY / L1
```

因此：

> Scheduler 負責「決定怎麼使用 radio resource」，FAPI 負責「把這個決定交給 PHY」。

這是這一章最重要的概念。

---

## 4.19 Why This Matters for NVIDIA Aerial

目前 OAI 原生架構可以理解成：

```text
OAI MAC
   ↓
 FAPI
   ↓
OAI PHY
```

如果使用 external Layer 1，架構可以變成：

```text
OAI MAC
   ↓
 FAPI
   ↓
External PHY
```

因此後續研究：

```text
OAI L2
   +
NVIDIA Aerial L1
```

時，不需要重新設計 OAI MAC Scheduler。

真正需要深入的是：

```text
OAI Scheduler
      ↓
FAPI Structures
      ↓
Transport
      ↓
NVIDIA Aerial L1
```

---

## 4.20 What I Learned

完成這一章後，我目前的理解是：

* OAI MAC Scheduler 位於 `openair2/LAYER2/NR_MAC_gNB/`。
* Scheduler 會在 DL slot 中進行 radio resource scheduling。
* UL scheduling 主要由 `nr_schedule_ulsch()` 處理。
* DL scheduling 可以從 `nr_schedule_ue_spec()` 開始追。
* Scheduler 不只決定 PRB，也會考慮 MCS、HARQ、TDA、beam 與 control channel resources。
* UL scheduling 需要利用 K2 安排 future UL slot。
* Scheduler 的結果最後會轉成 FAPI structures。
* `DL_TTI.request` 描述 Downlink PHY configuration。
* `TX_DATA.request` 帶 Downlink payload。
* `UL_TTI.request` 告訴 PHY 未來要準備接收的 UL transmission。
* FAPI 定義 MAC 與 PHY 交換的資訊，而 transport 則決定這些資訊實際如何傳送。

我目前最重要的理解可以濃縮成：

```text
MAC Scheduler
     ↓
Scheduling Decision
     ↓
FAPI
     ↓
PHY
```

接下來要理解的是：

```text
OAI MAC Scheduler
       ↓
FAPI
       ↓
nvIPC
       ↓
NVIDIA Aerial L1
```

也就是如何將 OAI 的 Layer 2 保留下來，再使用另一套 Layer 1 implementation。

---

## References

本章主要參考：

* OpenAirInterface — gNB MAC Scheduler Architecture
* OpenAirInterface — FAPI / nFAPI Documentation
* OpenAirInterface — `openair2/LAYER2/NR_MAC_gNB/`
* Small Cell Forum — 5G FAPI Specification (SCF222)
* Small Cell Forum — nFAPI Specification (SCF225)

> Note: OAI is under active development. Source-code function names and execution paths should be checked against the branch or tag used when reading the code.

---

**Previous:** [03. OAI Architecture](03-OAI-Architecture.md)

**Next:** [05. OAI L2 + NVIDIA Aerial L1](05-OAI-L2-NVIDIA-Aerial-L1.md)

