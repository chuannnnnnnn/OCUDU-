# 01. 5G Open RAN 學習筆記：以 OCUDU 作為架構拆解的切入點
## 學習紀錄

- **學習日期：** 2026/08/23
- **投入時間：** 約 3 小時
- **主要閱讀內容：** OCUDU Overview、O-RAN Architecture
- **本次重點：** CU / DU / RU 與 Functional Split 的基本概念

---
## 1.1 從傳統基地台到 O-RAN 架構的轉變

在傳統的理解中，5G 基地台架構可以先簡化成：

```text
5G Core
    │
   gNB
    │
    UE
```

也就是由 gNB 負責大部分 Radio Access Network（RAN）相關功能，再與 5G Core 和 UE 進行連接。

但在 5G 與 O-RAN 的架構中，gNB 可以進一步進行功能拆分：

```text
5G Core
    │
    CU
    │
    DU
    │
    RU
    │
    UE
```

其中：

* **CU（Centralized Unit）**：負責較高層的 RAN protocol 與控制功能
* **DU（Distributed Unit）**：負責較接近即時無線資源處理的功能
* **RU（Radio Unit）**：負責較接近實際無線訊號與 RF 的處理

這種設計稱為 **Functional Split（功能拆分）**。

它的核心概念是：

> 將原本集中在單一基地台設備中的不同功能模組化，並透過標準化介面進行互連。

這樣的架構可以讓不同的 software、hardware 或不同廠商的 RAN components 有機會進行更彈性的部署與整合。

---

## 1.2 現階段的學習目標與脈絡

目前我選擇 **OCUDU** 這套開源的 5G RAN CU/DU software stack 作為切入點。

現階段的目標不是直接進入大量 source code，而是先建立對整體 5G RAN architecture 的基本理解。

我的學習順序可以先整理成：

```text
5G gNB
   ↓
CU / DU / RU
   ↓
CU-CP / CU-UP
   ↓
DU-high / DU-low
   ↓
Protocol Stack
   ↓
Interfaces
   ↓
OAI Source Code
```

首先先理解最上層的三個核心元件：

```text
CU
DU
RU
```

再進一步往下拆成：

```text
CU
├── CU-CP
└── CU-UP

DU
├── DU-high
└── DU-low
```

這樣可以逐步釐清不同 protocol 與功能到底位在哪一個 component 中。

例如之後會進一步對應：

```text
RRC
PDCP
RLC
MAC
PHY
```

分別位於 CU、DU 或 RU 的哪一層。

---

## 1.3 為什麼先看 OCUDU？

我目前把 OCUDU 當成理解 5G RAN architecture 的入口。

原因是 OCUDU 將 CU、DU 與相關 interface 的架構劃分得相對清楚，因此可以先從模組化的角度理解：

```text
CU-CP
CU-UP
DU-high
DU-low
RU
```

以及它們之間的關係。

等這些架構概念建立之後，再去閱讀 **OAI（OpenAirInterface）** 的 source code，就比較容易知道：

* 目前看到的 protocol 位在哪一層
* 某個 module 屬於 CU 還是 DU
* MAC 與 PHY 的邊界在哪裡
* 不同 interface 在整個 gNB 中扮演什麼角色

因此我的學習方式不是：

```text
直接打開 OAI Repository
        ↓
從第一個檔案開始看
```

而是先：

```text
理解 RAN Architecture
        ↓
建立 Protocol Mapping
        ↓
再進入 OAI Source Code
```

這樣在閱讀大型 codebase 時會比較有方向。

---

## 1.4 Current Learning Path

目前整體學習路徑可以整理成：

```text
OCUDU
  ↓
理解 CU / DU / RU
  ↓
理解 CU-CP / CU-UP
  ↓
理解 DU-high / DU-low
  ↓
理解 RRC / PDCP / RLC / MAC / PHY
  ↓
理解 E1 / F1 / FAPI / Open Fronthaul
  ↓
OAI Architecture
  ↓
OAI MAC Scheduler
  ↓
FAPI
  ↓
NVIDIA Aerial L1
  ↓
Source Code Trace
```

這份筆記接下來會按照這個順序逐步往下整理。

---

## 1.5 What I Learned

目前我先建立了幾個基本觀念：

* 5G gNB 可以透過 Functional Split 拆成 CU、DU 與 RU。
* O-RAN 的重點不只是「把基地台拆開」，而是利用標準化介面讓不同功能模組可以獨立實作。
* CU、DU 與 RU 還可以進一步拆成 CU-CP、CU-UP、DU-high 與 DU-low。
* OCUDU 可以作為理解這套 RAN architecture 的切入點。
* 在進入 OAI source code 前，先建立 architecture 與 protocol mapping 會比較容易理解大型 codebase。

目前下一步需要進一步理解：

```text
CU-CP
CU-UP
DU-high
DU-low
```

各自負責哪些 protocol，以及它們之間透過哪些 interface 進行 communication。

---

## References

本章主要作為整體學習脈絡整理，後續架構與介面細節主要會參考：

* OCUDU Documentation
* O-RAN Architecture
* 3GPP 5G RAN Architecture
* OpenAirInterface Documentation

---

**Next:** [02. OCUDU Architecture and Interfaces](02-OCUDU-Architecture-and-Interfaces.md)
