# 5G RAN 學習筆記

這個 repository 用來記錄我目前對 **5G RAN 架構、OCUDU、OpenAirInterface（OAI）、FAPI，以及 NVIDIA Aerial L1** 的學習內容。

這份筆記的目標，是先從整體 5G RAN 架構開始建立基礎概念，再逐步深入到實際的 OAI source code，理解不同模組與介面在程式中的實作方式。

目前的學習路徑如下：

```text
OCUDU Architecture
        ↓
CU / DU / RU
        ↓
OAI Architecture
        ↓
MAC / Layer 2
        ↓
FAPI
        ↓
NVIDIA Aerial L1
        ↓
Source Code Trace
```

也就是先從 **OCUDU 的架構拆解**開始，理解 CU、DU、RU 與各層 protocol 的位置，再進一步對照 OAI 的 software architecture，最後深入 MAC Scheduler、FAPI，以及 OAI L2 與 NVIDIA Aerial L1 的整合方式。

