# 07. 這些模組可以拿來做什麼？從架構理解到實際應用

## 本章目標

前面幾章主要是在理解 **OCUDU、OAI、MAC Scheduler、FAPI 與 NVIDIA Aerial L1** 的架構、功能與資料流程。

這一章想進一步思考：

> 當我已經知道這些模組分別負責什麼之後，實際上可以拿它們做哪些修改、實驗或研究？

目前我把下一步的學習方向整理成：

```text
Architecture Understanding
        ↓
Source Code Modification
        ↓
System Integration
        ↓
Performance Measurement
        ↓
Result Analysis
```

也就是從「知道這個模組是什麼」，進一步走向「知道可以改什麼、量什麼，以及如何驗證結果」。

---

## 學習紀錄

- **學習日期：** 2026/08/25
- **投入時間：** 約 2.5 小時
- **主要整理內容：** OCUDU Functional Split、OAI MAC Scheduler、FAPI、NVIDIA Aerial L1
- **本次重點：** 思考各模組可以實際應用在哪些實驗與研究方向
- **目前優先方向：** `OAI MAC Scheduler → FAPI → PHY`

---

## 7.1 從理解架構到實際應用

前面的筆記主要是在回答：

```text
OCUDU 的架構怎麼拆？
        ↓
OAI 的程式碼放在哪裡？
        ↓
MAC Scheduler 如何做資源配置？
        ↓
FAPI 如何連接 MAC 與 PHY？
        ↓
NVIDIA Aerial L1 負責什麼？
```

但如果要開始進入實作，下一步更重要的問題會變成：

```text
我可以修改什麼？
        ↓
修改後會影響什麼？
        ↓
我要量測什麼？
        ↓
如何比較修改前後的結果？
```

因此我目前希望把學習方向從單純閱讀 documentation，逐步往 **source code modification、實際操作與結果驗證** 的方向前進。

---

## 7.2 OCUDU：研究 RAN Functional Split

OCUDU 對我來說，第一個用途是幫助理解與研究 5G RAN 的 functional split。

前面已經整理過：

```text
CU-CP
CU-UP
DU-high
DU-low
RU
```

理解這些模組之後，可以進一步思考：

- 哪些功能適合放在 CU？
- 哪些功能需要靠近 RU？
- CU / DU 分離後需要經過哪些 interfaces？
- 哪些 interfaces 對 latency 比較敏感？
- 不同 deployment 方式會如何影響 system complexity？
- 功能集中化與分散化之間有什麼 trade-off？

例如：

```text
Centralized CU
      ↓
      F1
      ↓
Distributed DU
      ↓
      RU
```

可以進一步研究 CU 與 DU 分離後的 communication、deployment 與 latency 問題。

因此我目前把 OCUDU 的用途理解成：

> OCUDU 可以作為研究 5G RAN functional split、模組部署方式與介面關係的架構基礎。

---

## 7.3 OAI：實際修改 5G gNB Software

OpenAirInterface 不只是提供 architecture documentation，也提供實際的 RAN source code。

因此可以進一步進行：

- Source code trace
- Protocol behavior observation
- Scheduling modification
- Message flow tracing
- Performance experiment

目前我認為最適合先進入實作的是：

```text
OAI MAC Scheduler
```

因為 Scheduler 會直接決定：

```text
Which UE?
    ↓
Which PRBs?
    ↓
Which MCS?
    ↓
Which HARQ process?
    ↓
Which time-domain resources?
```

所以如果希望從「理解系統」進一步走向「修改系統」，MAC Scheduler 是一個比較明確的起點。

---

## 7.4 MAC Scheduler：做 Radio Resource Allocation 實驗

例如同時存在兩個 UE：

```text
UE 1
Channel Condition = Good
Buffer = High

UE 2
Channel Condition = Poor
Buffer = High
```

Scheduler 必須決定有限的 radio resources 要如何分配。

概念上會經過：

```text
UE Selection
      ↓
PRB Allocation
      ↓
MCS Selection
      ↓
HARQ
      ↓
Scheduling Result
```

因此可以先觀察 OAI 原本的 scheduling policy，再嘗試修改其中一個比較小的部分。

例如：

- UE priority
- UE selection rule
- PRB allocation rule
- Scheduling fairness
- Resource allocation policy

實驗流程可以整理成：

```text
Original Scheduler
        ↓
Collect Results
        ↓
Modify One Policy
        ↓
Run Again
        ↓
Compare Results
```

可以觀察的項目包括：

- Throughput
- Latency
- PRB utilization
- UE fairness
- MCS distribution

因此 OAI MAC Scheduler 可以作為一個實際的 **radio resource management experiment platform**。

---

## 7.5 FAPI：驗證 Scheduler 的決策如何進入 PHY

前面已經知道：

```text
MAC Scheduler
      ↓
FAPI
      ↓
PHY
```

但在實際 source code 中，可以再進一步確認：

> Scheduler 做出的 PRB、MCS、HARQ 等決策，最後到底被寫到哪些 structures？

例如 Scheduler 做出：

```text
UE = UE1
PRB = X
MCS = Y
HARQ = Z
```

接著可以追蹤這些資訊如何進入：

```text
DL_TTI.request
TX_DATA.request
UL_TTI.request
```

因此可以建立一個 source trace：

```text
Scheduler Decision
        ↓
FAPI Structure
        ↓
Check PRB / MCS / HARQ Fields
        ↓
PHY Input
```

這樣就可以把原本抽象的：

```text
MAC → FAPI → PHY
```

進一步變成實際可以從 source code 驗證的 data flow。

---

## 7.6 FAPI：整合不同的 Layer 1

FAPI 的另一個用途，是讓 MAC 與 PHY 之間有比較明確的功能邊界。

原本可以是：

```text
OAI MAC
   ↓
FAPI
   ↓
OAI PHY
```

如果使用另一套 Layer 1，也可以變成：

```text
OAI MAC
   ↓
FAPI
   ↓
External PHY
```

因此可以進一步研究：

- MAC / PHY separation
- Different Layer 1 implementations
- FAPI compatibility
- FAPI version compatibility
- Real-time timing
- IPC latency
- L2 / L1 integration

這也是後續研究：

```text
OAI L2+
   +
NVIDIA Aerial L1
```

的重要基礎。

---

## 7.7 NVIDIA Aerial L1：研究 GPU 加速 PHY

如果把 OAI 與 NVIDIA Aerial L1 整合，可以形成：

```text
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
```

這時候研究問題就不只停留在：

```text
Scheduler 怎麼分配資源？
```

還可以進一步變成：

```text
PHY processing 需要多久？
        ↓
GPU 是否能在 deadline 前完成？
        ↓
IPC 是否產生額外 latency？
        ↓
Data movement 是否成為 bottleneck？
```

可以觀察的項目包括：

- PHY processing latency
- GPU utilization
- Slot processing time
- nvIPC communication latency
- CPU / GPU data movement
- Throughput
- Real-time deadline

因此我目前把 NVIDIA Aerial L1 的用途理解成：

> 可以拿來研究 GPU acceleration 如何應用在 5G PHY，以及 GPU computing 是否能滿足 real-time RAN processing 的需求。

---

## 7.8 比較 OAI Native PHY 與 NVIDIA Aerial L1

如果後續測試環境與硬體允許，也可以進一步比較不同 Layer 1 implementation。

例如：

```text
        Same / Similar Workload
                  │
        ┌─────────┴─────────┐
        ↓                   ↓

 OAI Native PHY      NVIDIA Aerial L1
        │                   │
        ↓                   ↓

 Native PHY Path        GPU / cuPHY
        │                   │
        └─────────┬─────────┘
                  ↓
               Compare
```

可以比較：

- Processing latency
- Throughput
- CPU utilization
- GPU utilization
- Resource consumption
- Real-time deadline behavior

這可以進一步回答：

> 將部分 PHY workload 移到 GPU 後，實際獲得了哪些效益，又增加了哪些 IPC、memory transfer 與 system integration cost？

---

## 7.9 我目前最適合先做的實驗

以目前的學習進度來看，我認為還不適合一開始就直接進行完整的：

```text
OAI L2+
   ↓
NVIDIA Aerial L1
   ↓
O-RU
```

比較適合的第一步是先從：

```text
OAI MAC Scheduler
        ↓
FAPI
        ↓
PHY
```

開始。

### Step 1：觀察 Scheduler

第一步先不修改 source code，而是觀察 Scheduler 每次產生的：

```text
UE
PRB
MCS
HARQ
```

等 scheduling information。

### Step 2：Trace FAPI

接著確認 Scheduler 的結果如何進入：

```text
DL_TTI.request
TX_DATA.request
UL_TTI.request
```

並找出相關欄位在 source code 中是在哪裡被設定。

### Step 3：做一個小修改

例如：

```text
Modify UE Priority
```

或：

```text
Modify PRB Allocation Rule
```

一開始先避免同時改動太多 Scheduler logic。

### Step 4：比較修改前後結果

最後比較：

```text
Original Scheduler
        vs.
Modified Scheduler
```

觀察：

- Throughput
- Latency
- PRB allocation
- UE fairness
- MCS

是否產生差異。

這樣可以從比較小的 modification 開始，逐步建立真正操作 OAI source code 的經驗。

---

## 7.10 不同模組可以做什麼？

目前我把不同模組與可能的實作方向整理如下：

| Module | 可以做的事情 | 可以觀察的項目 |
|---|---|---|
| OCUDU | Functional Split / Deployment | Architecture、Latency |
| OAI MAC Scheduler | 修改 scheduling policy | Throughput、Fairness、PRB |
| FAPI | Trace MAC-PHY messages | PRB、MCS、HARQ |
| OAI PHY | PHY processing analysis | Processing Time |
| NVIDIA Aerial L1 | GPU-accelerated PHY | GPU Utilization、Latency |
| nvIPC | L2 / L1 communication | IPC Latency |
| O-RAN Fronthaul | DU-low / O-RU communication | Transport、Timing |

因此這些模組並不是彼此完全獨立，而是可以串成：

```text
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
```

---

## 7.11 我目前的實作方向

目前我認為最值得優先深入的是：

```text
OAI MAC Scheduler
        ↓
Scheduling Decision
        ↓
FAPI Structure
        ↓
PHY
```

先真正看懂：

- Scheduler 在哪裡做決策
- PRB / MCS / HARQ 在哪裡產生
- 這些資訊如何寫進 FAPI structures
- PHY 最後如何使用這些資訊

等這條 path 熟悉之後，再逐步延伸到：

```text
OAI MAC Scheduler
        ↓
FAPI
        ↓
nvIPC
        ↓
NVIDIA Aerial L1
        ↓
GPU
```

也就是先從較小的 source code modification 開始，再逐步走向完整的 L2 / L1 integration。

---

## 7.12 What I Learned

經過前面 01～06 的整理後，我目前理解這些模組不只是用來描述 5G RAN architecture，也可以作為實際實驗與研究的平台。

目前我認為可以進一步做的事情包括：

- 修改 OAI MAC Scheduler
- 比較不同 PRB allocation strategy
- Trace FAPI messages
- 驗證 scheduling decision 如何進入 PHY
- 比較不同 Layer 1 implementation
- 研究 OAI L2+ 與 NVIDIA Aerial L1 integration
- 評估 throughput、latency、fairness 與 processing time

因此接下來我的學習方向會逐漸從：

```text
What is this module?
```

轉變成：

```text
What can I modify?
        ↓
What can I measure?
        ↓
What can I compare?
        ↓
What can I verify?
```

這也是我下一步希望補強的實作方向。

---

## 7.13 Next Step

目前預計先從：

```text
OAI MAC Scheduler
        ↓
Observe Scheduling Result
        ↓
Trace FAPI
        ↓
Small Scheduler Modification
        ↓
Performance Comparison
```

開始。

等熟悉 OAI Scheduler 與 FAPI 的實際操作之後，再進一步研究：

```text
OAI L2+
   ↓
FAPI
   ↓
nvIPC
   ↓
NVIDIA Aerial L1
```

的完整 integration。

---

## References

- OCUDU Documentation
- OpenAirInterface RAN Repository
- OpenAirInterface MAC Scheduler Architecture
- OpenAirInterface FAPI / nFAPI Documentation
- OpenAirInterface Aerial FAPI Split Tutorial
- NVIDIA Aerial L1 Documentation
- Small Cell Forum — 5G FAPI Specification

> 本章主要整理目前根據 architecture、documentation 與 source code 閱讀後想到的可行實作方向。實際實驗方式仍需要依照後續使用的 OAI branch、測試環境與硬體平台進一步確認。

---

**Previous:** [06. Source Code Trace](06-Source-Code-Trace.md)

**Back to:** [README](README.md)