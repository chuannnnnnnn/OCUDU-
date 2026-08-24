1. 5G Open RAN 學習筆記：以 OCUDU 作為架構拆解的切入點

1.1 從傳統基站到 O-RAN 架構的轉變
在過去的觀念裡，5G 基地台的架構大致可以簡化成：

5G Core ── gNB ── UE

但在 O-RAN 的架構設計中，gNB 被進一步拆分為更細緻的模組：

5G Core ── CU ── DU ── RU ── UE

這種功能拆分（Functional Split）的核心邏輯，在於把原本集中在單一硬體設備上的基地台功能模組化，並透過標準化的介面進行互連。

1.2 現階段的學習目標與脈絡
目前我選擇把 OCUDU 這套開源的 5G RAN CU/DU 軟體堆疊作為切入點，主要目的是先建立對整體 5G RAN 架構的宏觀理解。

透過看 OCUDU 的架構，可以先釐清最上層的幾大核心元件：

CU（Centralized Unit）

DU（Distributed Unit）

RU（Radio Unit）

接著再往下深入到更細緻的子模組切分：

CU-CP / CU-UP

DU-high / DU-low

先在架構層面把這些元件與介面的關係搞懂，之後直接去翻 OAI（OpenAirInterface）的 source code 時，心裡才比較有個底，知道各個 protocol 和 module 實際上是落在整個 gNB 的哪個位置。
