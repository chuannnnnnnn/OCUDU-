# 03. OAI Architecture

## 3.1 OpenAirInterface 是什麼？

OpenAirInterface（OAI）是一套開源 cellular RAN software stack，可用來研究與實作：

* 5G NR gNB
* NR UE
* Layer 1 / PHY
* Layer 2
* RRC
* CU / DU functional split
* F1 / E1
* FAPI / nFAPI
* SDR / Radio interface

前一章使用 OCUDU 理解的是：

```text
CU-CP
CU-UP
DU-high
DU-low
RU
```

而 OAI 可以進一步讓我看到：

> 這些 RAN functions 在實際 source code 中放在哪裡，以及它們之間如何互動。

因此我目前把兩者的角色理解成：

```text
OCUDU
  ↓
理解 RAN Architecture

OAI
  ↓
理解 Software Implementation
```

---

## 3.2 OAI Repository Structure

OAI 主要 repository 可以簡化成：

```text
openairinterface5g/
│
├── openair1/
│
├── openair2/
│
├── openair3/
│
├── nfapi/
│
├── radio/
│
├── executables/
├── common/
├── cmake_targets/
└── doc/
```

目前最重要的幾個目錄是：

| Directory     | 主要內容                                          |
| ------------- | --------------------------------------------- |
| `openair1`    | PHY / Layer 1                                 |
| `openair2`    | MAC / RLC / PDCP / SDAP / RRC                 |
| `openair3`    | NGAP / GTP 等 network-related protocol         |
| `nfapi`       | FAPI / nFAPI                                  |
| `radio`       | Radio / SDR interface                         |
| `executables` | gNB / UE runtime 與 executable                 |
| `doc`         | OAI 官方 architecture / developer documentation |

因此之後閱讀 OAI source code 時，不需要從 repository 最上層開始逐個檔案閱讀。

可以先依照要研究的 protocol 找對應目錄。

---

## 3.3 `openair1`：PHY / Layer 1

`openair1/` 主要放：

> Physical Layer（PHY / L1）

包含許多 5G PHY processing，例如：

* LDPC coding / decoding
* Polar coding
* OFDM
* Modulation / Demodulation
* Channel Estimation
* PDSCH
* PUSCH
* PDCCH
* PUCCH
* PBCH
* PRACH

可以簡化成：

```text
openair1
   ↓
PHY / Layer 1
```

例如 Downlink PHY processing：

```text
Transport Block
      ↓
Channel Coding
      ↓
Rate Matching
      ↓
Modulation
      ↓
Resource Mapping
      ↓
OFDM
```

Uplink 則包含：

```text
Received Signal
      ↓
Channel Estimation
      ↓
Equalization
      ↓
Demodulation
      ↓
Decoding
```

因此如果研究：

```text
OAI L2
   +
External L1
```

`openair1` 就代表 OAI 原生 PHY implementation。

---

## 3.4 `openair2`：RAN Layer 2 and Upper Layers

`openair2/` 是目前對我最重要的目錄。

其中包含：

```text
MAC
RLC
PDCP
SDAP
RRC
F1AP
E1AP
E2AP
```

因此可以簡化成：

```text
openair2
│
├── MAC
├── RLC
├── PDCP
├── SDAP
├── RRC
└── RAN signaling protocols
```

---

## 3.5 MAC

MAC 是後續閱讀 OAI Layer 2 時最重要的部分。

主要負責：

* DL Scheduling
* UL Scheduling
* Radio Resource Allocation
* HARQ
* Random Access
* Logical Channel Multiplexing

其中最重要的是：

> **MAC Scheduler**

Scheduler 需要根據 UE 狀態決定：

```text
UE
 ↓
PRB
 ↓
MCS
 ↓
HARQ
 ↓
Time / Frequency Resource
```

最後再把 scheduling result 交給 PHY。

因此：

```text
MAC Scheduler
      ↓
Scheduling Decision
      ↓
FAPI
      ↓
PHY
```

後續第 04 章會再深入 MAC Scheduler。

---

## 3.6 RLC

RLC 位於：

```text
PDCP
 ↓
RLC
 ↓
MAC
```

主要功能包括：

* Segmentation
* Reassembly
* Retransmission
* Sequence Control
* In-order Delivery

在 CU / DU functional split 中，RLC 通常位在 DU side。

因此：

```text
CU
│
PDCP
│
│ F1
│
DU
│
RLC
│
MAC
```

---

## 3.7 PDCP / SDAP

PDCP 與 SDAP 位在較高層的 User Plane。

可以簡化成：

```text
5G Core
   ↓
SDAP
   ↓
PDCP
   ↓
RLC
```

### SDAP

SDAP 主要處理：

```text
QoS Flow
   ↓
Data Radio Bearer
```

### PDCP

PDCP 主要處理：

* Ciphering
* Integrity Protection
* Header Compression
* Sequence Number Handling

在 CU / DU split 中，PDCP 與 SDAP 通常位在 CU side。

---

## 3.8 RRC

RRC 是 gNB Control Plane 的核心 protocol。

主要負責：

* UE Connection Setup
* UE Context Management
* Radio Bearer Configuration
* Measurement Configuration
* Mobility
* Handover
* Security Configuration

可以簡化成：

```text
RRC
 ↓
控制 UE Connection
```

在 CU / DU split 中，RRC 位在：

```text
CU
```

更進一步分離 CU-CP / CU-UP 時：

```text
CU-CP
 ↓
RRC
```

因此 RRC 是理解 CU-CP implementation 的重要入口。

---

## 3.9 `openair3`：Core Network Interface

`openair3/` 主要包含：

* NGAP
* GTP
* 其他 network / core-related functions

其中對 gNB 最重要的是：

```text
NGAP
+
GTP-U
```

---

## 3.10 NGAP

NGAP 主要處理：

```text
gNB
 │
 N2
 │
AMF
```

相關 Control Plane signaling。

例如：

* Initial UE Message
* UE Context
* PDU Session signaling
* Mobility-related signaling

所以可以先理解：

```text
NGAP
=
gNB ↔ 5G Core Control Plane
```

---

## 3.11 GTP-U

GTP-U 主要負責：

```text
gNB
 │
 N3
 │
UPF
```

的 User Plane Data。

因此：

```text
NGAP
→ Control Plane

GTP-U
→ User Plane
```

---

## 3.12 `nfapi`：MAC-PHY Interface

OAI repository 中還有：

```text
nfapi/
```

這個目錄主要與：

```text
FAPI
nFAPI
```

有關。

它位於：

```text
MAC
 │
FAPI / nFAPI
 │
PHY
```

之間。

目前我只先記住：

> `nfapi` 是 OAI 中研究 MAC-PHY interface 的重要目錄。

FAPI 的詳細內容放到：

**[04. FAPI and OAI MAC Scheduler](04-FAPI-and-MAC-Scheduler.md)**

再深入。

---

## 3.13 `radio`：Radio Interface

`radio/` 主要負責 OAI 與實際 radio / SDR hardware 的連接。

概念：

```text
OAI MAC
   ↓
OAI PHY
   ↓
Radio Driver
   ↓
SDR / RU
   ↓
Antenna
```

因此完整流程不只有 protocol stack：

```text
RRC
 ↓
PDCP
 ↓
RLC
 ↓
MAC
 ↓
PHY
```

最後還需要：

```text
Radio Driver
 ↓
Radio Hardware
```

才能真正 transmit / receive RF signal。

---

## 3.14 `executables`

`executables/` 包含 OAI 執行 gNB / UE 時的重要 runtime code。

其中 OAI 5G gNB 最重要的 executable 之一是：

```text
nr-softmodem
```

可以把它理解成：

> OAI 5G gNB 的主要執行程式。

概念：

```text
Configuration
     ↓
nr-softmodem
     ↓
Initialize RAN
     ↓
RRC / PDCP / RLC / MAC / PHY
     ↓
Radio
```

不同 configuration 可以讓 OAI 以不同 deployment mode 執行。

---

## 3.15 Monolithic gNB

最簡單的 OAI deployment 是：

```text
          OAI gNB
┌──────────────────────┐
│ RRC                  │
│ PDCP / SDAP          │
│ RLC                  │
│ MAC                  │
│ PHY                  │
└──────────┬───────────┘
           │
         Radio
           │
          UE
```

也就是：

> RRC 到 PHY 都存在同一套 gNB implementation 中。

這可以先稱為：

```text
Monolithic gNB
```

---

## 3.16 OAI CU / DU Split

OAI 也支援 CU / DU functional split。

概念：

```text
            CU
      ┌─────────────┐
      │ RRC         │
      │ PDCP        │
      │ SDAP        │
      └──────┬──────┘
             │
             F1
             │
      ┌──────▼──────┐
      │ RLC         │
      │ MAC         │
      │ PHY         │
      │      DU     │
      └──────┬──────┘
             │
           Radio
```

因此可以先記成：

```text
CU
=
RRC + PDCP + SDAP
```

```text
DU
=
RLC + MAC + PHY
```

這和前面 OCUDU 學到的 functional split 是相同的概念。

---

## 3.17 CU-CP / CU-UP

CU 還可以再進一步拆分：

```text
CU
│
├── CU-CP
└── CU-UP
```

概念上：

```text
CU-CP
 ↓
RRC / Control Plane
```

而：

```text
CU-UP
 ↓
PDCP / SDAP / User Plane
```

因此完整架構可以理解成：

```text
              CU-CP
               RRC
                │
               E1
                │
              CU-UP
           PDCP / SDAP
                │
               F1
                │
                DU
           RLC / MAC / PHY
```

這與前面 OCUDU 的 architecture 可以直接對照。

---

## 3.18 OCUDU and OAI Mapping

目前可以把前面 OCUDU 與 OAI 對照：

| Architecture Function | OAI Implementation       |
| --------------------- | ------------------------ |
| CU-CP                 | RRC / Control Plane      |
| CU-UP                 | PDCP / SDAP / User Plane |
| DU-high               | RLC / MAC                |
| DU-low                | PHY / Layer 1            |
| MAC-PHY Interface     | `nfapi` / FAPI           |
| Radio                 | `radio`                  |

因此我目前的理解是：

```text
OCUDU
 ↓
告訴我 RAN 功能應該怎麼拆
```

而：

```text
OAI
 ↓
讓我看到這些功能實際寫在哪裡
```

---

## 3.19 Where Should I Start Reading?

OAI repository 很大，因此目前我不打算：

> 從第一個檔案一路看到最後一個檔案。

而是沿著一條明確的 path 閱讀。

目前最重要的路徑是：

```text
openair2
   ↓
MAC
   ↓
Scheduler
   ↓
FAPI
   ↓
openair1 / External PHY
```

也就是：

> **MAC → FAPI → PHY**

---

## 3.20 Source Code Reading Map

如果要找不同功能，可以先按照這張表：

| 想研究的內容        | 先找哪裡                          |
| ------------- | ----------------------------- |
| PHY           | `openair1/`                   |
| MAC Scheduler | `openair2/LAYER2/NR_MAC_gNB/` |
| RLC           | `openair2/`                   |
| PDCP          | `openair2/`                   |
| RRC           | `openair2/`                   |
| NGAP          | `openair3/`                   |
| FAPI          | `nfapi/`                      |
| Radio         | `radio/`                      |
| gNB Runtime   | `executables/`                |

目前最值得深入的是：

```text
openair2/LAYER2/NR_MAC_gNB/
```

因為這裡包含：

> OAI gNB MAC Scheduler。

---

## 3.21 Current Learning Path

到目前為止，我的閱讀順序可以整理成：

```text
OCUDU
   ↓
理解 CU / DU / RU
   ↓
理解 RRC / PDCP / RLC / MAC / PHY
   ↓
OAI Repository
   ↓
找到每個 protocol 的 source code
   ↓
MAC Scheduler
   ↓
FAPI
   ↓
PHY
```

下一步不需要再花很多時間重新理解：

```text
CU 是什麼
DU 是什麼
RLC 是什麼
MAC 是什麼
```

而是開始深入：

```text
OAI MAC Scheduler
       ↓
Scheduling Decision
       ↓
FAPI
       ↓
PHY
```

---

## 3.22 What I Learned

完成這一章後，我目前的理解是：

* OAI 不只是 gNB，也包含 UE、PHY、Layer 2、RRC、FAPI 與 Radio support。
* `openair1` 主要負責 PHY / Layer 1。
* `openair2` 是閱讀 MAC、RLC、PDCP、RRC 最重要的地方。
* `openair3` 主要包含 NGAP、GTP 等 network-related protocol。
* `nfapi` 是研究 MAC-PHY interface 的重要入口。
* OAI 可以支援 monolithic gNB，也可以做 CU / DU functional split。
* 前面學到的 OCUDU architecture 可以直接幫助理解 OAI source tree。
* 不需要一次閱讀整個 repository，應該沿著一條明確的 data path 進行。

目前我最想深入的是：

```text
OAI MAC
   ↓
Scheduler
   ↓
FAPI
   ↓
PHY
```

因為這條路徑正好是後續理解：

```text
OAI L2
   +
NVIDIA Aerial L1
```

的基礎。

---

## References

本章主要參考：

* OpenAirInterface / Duranta RAN Repository
* OpenAirInterface Repository README
* OpenAirInterface F1 Split Documentation
* OpenAirInterface E1 Split Documentation
* OpenAirInterface `nr-softmodem` Documentation
* OpenAirInterface Source Tree
* OpenAirInterface Developer Documentation

---

**Previous:** [02. OCUDU Architecture and Interfaces](02-OCUDU-Architecture-and-Interfaces.md)

**Next:** [04. FAPI and OAI MAC Scheduler](04-FAPI-and-MAC-Scheduler.md)

