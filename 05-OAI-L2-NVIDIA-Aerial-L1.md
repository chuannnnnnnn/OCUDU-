# 05. OAI L2 + NVIDIA Aerial L1
## 本章目標

本章的目標是理解 **OAI Layer 2 與 NVIDIA Aerial Layer 1 的整合架構**，釐清 OAI、FAPI、nvIPC、Aerial L2 Adapter 與 cuPHY 各自扮演的角色，並理解一筆資料如何從 OAI MAC 經過 Aerial L1 最後送到 O-RU。
## 5.1 Overview

前一章已經整理：

```text
OAI MAC Scheduler
        ↓
Scheduling Decision
        ↓
FAPI
        ↓
PHY
```

OAI 本身具有自己的 Layer 1 implementation，因此一般情況可以是：

```text
OAI Layer 2
     │
    FAPI
     │
OAI Layer 1
     │
   Radio
```

但是 PHY 中包含大量計算密集的工作，例如：

* Channel Coding / Decoding
* Modulation / Demodulation
* Channel Estimation
* MIMO Processing
* Physical Channel Processing

因此另一種做法是保留 OAI Layer 2，將 Layer 1 改成 NVIDIA Aerial。

```text
OAI Layer 2
     │
    FAPI
     │
   nvIPC
     │
NVIDIA Aerial L1
     │
   cuPHY
     │
 NVIDIA GPU
```

這就是這一章要理解的主要架構。

---

## 5.2 Overall Architecture

OAI L2 + NVIDIA Aerial L1 可以簡化成：

```text
                 OAI

        RRC / PDCP / RLC
               │
              MAC
               │
          MAC Scheduler
               │
              FAPI
               │
             nvIPC

================ L2 / L1 ================

               │
       Aerial L2 Adapter
               │
         cuPHY Driver
               │
             cuPHY
               │
          NVIDIA GPU
               │
        O-RAN Fronthaul
               │
              O-RU
               │
              UE
```

其中兩邊的責任可以簡化成：

| Component         | Responsibility                      |
| ----------------- | ----------------------------------- |
| OAI               | RAN protocol、MAC、Scheduling         |
| FAPI              | MAC-PHY message interface           |
| nvIPC             | OAI L2 ↔ Aerial L1 communication    |
| Aerial L2 Adapter | FAPI command → L1 task              |
| cuPHY Driver      | PHY task / GPU execution management |
| cuPHY             | GPU-accelerated 5G PHY              |
| O-RU              | Lower PHY / RF                      |

---

## 5.3 What Remains in OAI?

接上 Aerial L1 後，OAI 並不是整套被取代。

OAI 仍然負責：

```text
RRC
 ↓
PDCP / SDAP
 ↓
RLC
 ↓
MAC
 ↓
Scheduler
```

尤其是 MAC Scheduler 仍然在 OAI 中。

也就是：

```text
OAI Scheduler

決定：
- UE
- Radio Resource
- MCS
- HARQ
- Time-domain Allocation
```

Scheduler 完成 decision 後，再將結果轉換成 FAPI structures 交給 Layer 1。

因此：

> NVIDIA Aerial L1 不負責取代 OAI MAC Scheduler，而是負責執行 Scheduler 所要求的 PHY processing。

---

## 5.4 What Does NVIDIA Aerial L1 Do?

NVIDIA Aerial L1 主要負責 Layer 1 / PHY。

NVIDIA 的 Aerial L1 software stack 主要包含：

```text
Aerial L1
│
├── L2 Adapter
├── cuPHY Driver
├── cuPHY
└── cuPHY Controller
```

### L2 Adapter

主要負責：

```text
FAPI Command
     ↓
L2 Adapter
     ↓
L1 Task
```

也就是接收 Layer 2 傳來的 FAPI messages，並轉換成 Layer 1 可以執行的工作。

### cuPHY Driver

主要負責：

* 控制 cuPHY kernel execution
* 管理 PHY task
* 管理資料進出 GPU
* 協調 PHY 與 Fronthaul processing

### cuPHY

`cuPHY` 是 NVIDIA 的 GPU-accelerated 5G PHY library。

負責執行實際的 PHY processing。

### cuPHY Controller

主要負責 L1 initialization，例如：

* Cell configuration
* Fronthaul buffers
* L1 threads
* Runtime configuration

---

## 5.5 Role of FAPI

在這個架構中，OAI 與 Aerial 之間最重要的共同介面仍然是：

```text
FAPI
```

例如 OAI Scheduler 產生：

```text
DL_TTI.request
UL_TTI.request
TX_DATA.request
```

Aerial L1 則根據這些 FAPI messages 執行 PHY 工作。

概念：

```text
OAI MAC Scheduler
        ↓
DL_TTI.request
UL_TTI.request
TX_DATA.request
        ↓
       FAPI
        ↓
Aerial L1
```

因此 OAI 不需要知道 Aerial 裡面的 CUDA kernel 是怎麼實作的。

OAI 只需要按照 MAC-PHY interface 提供 Layer 1 所需要的資訊。

---

## 5.6 Role of nvIPC

FAPI 定義的是：

> MAC 與 PHY 需要交換什麼資訊。

但是這些資訊仍然需要真正從 OAI process 傳到 Aerial process。

這裡使用：

```text
nvIPC
```

NVIDIA Aerial 使用 nvIPC 作為 Layer 2 與 Layer 1 之間的 communication mechanism。

因此：

```text
OAI MAC
   │
FAPI Message
   │
 nvIPC
   │
Aerial L2 Adapter
```

可以簡化成：

```text
FAPI
=
What information is exchanged
```

```text
nvIPC
=
How the information is transported
```

這兩者不要混在一起。

---

## 5.7 OAI Aerial Configuration

OAI 官方 Aerial tutorial 中，可以看到類似的 southbound transport configuration：

```text
tr_s_preference = "aerial";
tr_s_shm_prefix = "nvipc";
```

概念上：

```text
OAI MAC
   ↓
Southbound Interface
   ↓
Aerial Transport
   ↓
nvIPC
   ↓
Aerial L1
```

這表示 OAI MAC Scheduler 本身不需要寫死：

```text
NVIDIA Aerial
```

而是透過 southbound interface 選擇後面的 Layer 1 implementation。

---

## 5.8 Downlink Data Flow

Downlink 可以從 OAI Scheduler 開始看。

假設 Scheduler 已經決定 UE 的 transmission configuration：

```text
OAI MAC Scheduler
        ↓
Scheduling Decision
        ↓
DL_TTI.request
+
TX_DATA.request
```

接下來：

```text
DL_TTI.request
TX_DATA.request
        ↓
       FAPI
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

最後經過 Fronthaul：

```text
GPU PHY
   ↓
O-RAN Fronthaul
   ↓
O-RU
   ↓
RF
   ↓
UE
```

因此完整 Downlink path 可以整理成：

```text
RLC Buffer
    ↓
OAI MAC Scheduler
    ↓
DL_TTI.request
TX_DATA.request
    ↓
FAPI
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
    ↓
O-RAN Fronthaul
    ↓
O-RU
    ↓
UE
```

---

## 5.9 Uplink Data Flow

Uplink 的方向則相反。

首先 OAI Scheduler 先配置未來的 UL resource：

```text
OAI MAC Scheduler
        ↓
UL_TTI.request
        ↓
FAPI
        ↓
nvIPC
        ↓
Aerial L1
```

Aerial 因此知道：

> 在指定的 frame / slot 中，需要準備接收哪些 Uplink transmission。

UE 真正傳送 PUSCH 後：

```text
UE
 ↓
O-RU
 ↓
Fronthaul
 ↓
Aerial L1
 ↓
GPU PHY Processing
```

Layer 1 完成 decoding 後，再回傳：

```text
RX_DATA.indication
CRC.indication
UCI.indication
```

因此：

```text
UE
 ↓
O-RU
 ↓
Aerial cuPHY
 ↓
GPU Decode
 ↓
FAPI Indication
 ↓
nvIPC
 ↓
OAI MAC
 ↓
RLC
```

---

## 5.10 Why Use GPU for PHY?

5G PHY 中有大量可以平行運算的 workload，例如：

```text
LDPC
MIMO
Channel Estimation
Modulation
Physical Channel Processing
```

而且實際系統中還可能同時存在：

```text
Multiple UEs
Multiple Antennas
Multiple PRBs
Multiple Code Blocks
```

因此這些工作適合利用 GPU 的 parallel processing capability。

概念：

```text
PHY Workload
     ↓
Parallel Processing
     ↓
CUDA
     ↓
NVIDIA GPU
```

所以 OAI 與 Aerial 的分工可以理解成：

```text
CPU / OAI
↓
Protocol + Scheduling
```

```text
GPU / Aerial
↓
Compute-intensive PHY
```

---

## 5.11 O-RAN 7.2x Fronthaul

FAPI 與 Open Fronthaul 是兩個不同的介面。

完整位置是：

```text
OAI MAC
   │
 FAPI
   │
Aerial L1
   │
O-RAN 7.2x Fronthaul
   │
 O-RU
```

因此：

```text
FAPI
=
MAC ↔ PHY
```

而：

```text
O-RAN Fronthaul
=
O-DU / Upper PHY ↔ O-RU
```

在 O-RAN 7.2x architecture 中，可以簡化理解為：

```text
Aerial L1
Upper PHY
    │
O-RAN 7.2x
    │
O-RU
Lower PHY + RF
```

---

## 5.12 Real-Time Timing

這個 integration 最大的挑戰之一是：

> Real-Time Timing

5G RAN 不是只要求：

```text
Message eventually arrives
```

而是要求：

```text
Message arrives before the slot deadline
```

例如：

```text
Slot Timing
    ↓
OAI Scheduler
    ↓
FAPI
    ↓
nvIPC
    ↓
Aerial
    ↓
GPU Processing
    ↓
Fronthaul
    ↓
RU Transmission
```

整條 pipeline 都必須在正確的時間內完成。

如果 FAPI message 太晚到：

```text
Late Message
    ↓
Miss PHY Deadline
    ↓
Miss Transmission Opportunity
```

因此 RAN 的 correctness 不只是：

```text
Correct Result
```

還包括：

```text
Correct Result
+
Correct Timing
```

---

## 5.13 FAPI Version Compatibility

另一個實際 integration 問題是：

```text
FAPI Version Compatibility
```

即使：

```text
OAI supports FAPI
```

而：

```text
Aerial supports FAPI
```

也不能直接假設：

```text
OAI ↔ Aerial
一定相容
```

仍然需要確認：

* Message version
* PDU definition
* Field definition
* Optional fields
* Structure compatibility

OAI 官方 Aerial tutorial 目前也特別提到，從較新的 OAI version 開始，部分 SRS / RX Beamforming PDU 使用較新的 FAPI definition，因此 Aerial SDK build configuration 也需要配合。

這讓我理解到：

> 使用相同標準介面只是 integration 的第一步，實際版本仍然必須互相匹配。

---

## 5.14 Data Movement

GPU acceleration 不只是 GPU 計算速度快就足夠。

還需要考慮：

```text
CPU
 ↓
Data Movement
 ↓
GPU
```

如果大量時間消耗在：

```text
Memory Copy
```

那麼 GPU acceleration 的效益可能下降。

因此系統還需要考慮：

* Buffer Management
* Memory Transfer
* IPC Latency
* GPU Memory
* Data Movement

NVIDIA Aerial 的 cuPHY Driver 也負責管理 cuPHY kernel 執行以及資料進出相關工作。

所以整個系統的 performance 可以理解成：

```text
Computation
+
Communication
+
Data Movement
+
Timing
```

---

## 5.15 Integration Challenges

因此真正的：

```text
OAI L2 + NVIDIA Aerial L1
```

不只是：

```text
OAI
 ↓
FAPI
 ↓
Aerial
```

還需要同時考慮：

```text
FAPI Compatibility
        +
FAPI Version
        +
nvIPC Transport
        +
Slot Synchronization
        +
Real-Time Deadline
        +
CPU / GPU Data Movement
        +
GPU Processing
        +
O-RAN Fronthaul
```

我目前認為真正的 integration 難點是：

> 如何讓 OAI Layer 2、Aerial Layer 1、GPU 與 Radio Unit 在正確的 slot timing 下穩定協同工作。

---

## 5.16 Software and Hardware View

整個系統也可以從 Software / Hardware 的角度來看：

```text
             Software

5G Core
   ↓
OAI RRC
   ↓
OAI PDCP
   ↓
OAI RLC
   ↓
OAI MAC
   ↓
Scheduler
   ↓
FAPI
   ↓
nvIPC
   ↓
Aerial L2 Adapter
   ↓
cuPHY Driver
   ↓
cuPHY

================================

             Hardware

NVIDIA GPU
   ↓
NIC
   ↓
O-RAN Fronthaul
   ↓
O-RU
   ↓
RF
   ↓
Antenna
   ↓
UE
```

因此這個架構同時涉及：

```text
5G Protocol
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

## 5.17 Complete Architecture

最後可以將整個架構濃縮成：

```text
                     5G Core
                        │
                        ↓
                     OAI RRC
                        │
                     OAI PDCP
                        │
                     OAI RLC
                        │
                     OAI MAC
                        │
                    Scheduler
                        │
                       FAPI
                        │
                      nvIPC

============= OAI / Aerial Boundary =============

                        │
                Aerial L2 Adapter
                        │
                  cuPHY Driver
                        │
                      cuPHY
                        │
                  NVIDIA GPU
                        │
                O-RAN Fronthaul
                        │
                       O-RU
                        │
                        RF
                        │
                        UE
```

這是目前我理解 OAI L2 + NVIDIA Aerial L1 最重要的一張圖。

---

## 5.18 What I Learned

完成這一章後，我目前的理解是：

* 使用 Aerial L1 時，OAI 仍然保留 RRC、PDCP、RLC、MAC 與 Scheduler。
* NVIDIA Aerial 主要負責 Layer 1 / PHY processing。
* OAI Scheduler 仍然負責產生 radio resource scheduling decision。
* OAI 與 Aerial 使用 FAPI 作為 MAC-PHY communication interface。
* nvIPC 負責 OAI L2 與 Aerial L1 之間的實際 communication。
* Aerial L2 Adapter 將 FAPI command 轉換成 Layer 1 task。
* cuPHY Driver 負責管理 cuPHY kernel execution 與 data movement。
* cuPHY 使用 CUDA / NVIDIA GPU 執行 5G PHY processing。
* FAPI 與 O-RAN Fronthaul 位於不同的 functional split。
* 實際 integration 還需要處理 FAPI version、slot timing、IPC latency、GPU processing 與 Fronthaul。

目前我對整個架構最簡單的理解是：

```text
OAI
↓
決定「怎麼傳」
```

而：

```text
NVIDIA Aerial
↓
執行「PHY 要怎麼把它算出來並傳出去」
```

因此下一步不需要再繼續增加 architecture 定義，而是開始從 source code 驗證：

```text
OAI Scheduler
      ↓
FAPI Structure
      ↓
Aerial Transport
      ↓
Aerial L1
```

這條實際 software path。

---

## References

本章主要參考：

* OpenAirInterface — Aerial FAPI Split Tutorial
* OpenAirInterface — FAPI / nFAPI Documentation
* OpenAirInterface — `nr-softmodem` Documentation
* NVIDIA Aerial — L1 Software Architecture
* NVIDIA Aerial — L2 Adapter
* NVIDIA Aerial — cuPHY
* NVIDIA Aerial — cuPHY Driver
* Small Cell Forum — 5G FAPI Specification
* O-RAN Alliance — Open Fronthaul / 7.2x Architecture

> Note: OAI and NVIDIA Aerial are actively developed projects. Configuration, FAPI compatibility and source-code paths should always be checked against the specific software version being used.

---

**Previous:** [04. FAPI and OAI MAC Scheduler](04-FAPI-and-MAC-Scheduler.md)

**Next:** [06. Source Code Trace](06-Source-Code-Trace.md)

