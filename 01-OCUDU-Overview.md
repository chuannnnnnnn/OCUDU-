01. OCUDU Overview
1.1 OCUDU 是什麼？

OCUDU 是一套開源 5G RAN CU/DU software stack。

在傳統基地台中，可以先簡化成：

5G Core
   │
  gNB
   │
  UE

而在 5G / O-RAN 架構中，gNB 可以被拆分：

5G Core
   │
  CU
   │
  DU
   │
  RU
   │
  UE

這種拆分的核心概念是：

將原本集中在一個基地台中的功能拆成不同模組，並透過標準化 interface 互相連接。

1.2 為什麼要看 OCUDU？

我目前把 OCUDU 當成理解 5G RAN architecture 的入口。

透過 OCUDU，可以先理解：

CU
DU
RU

以及後續更細的：

CU-CP
CU-UP
DU-high
DU-low

理解這些元件後，再去看 OAI source code，會比較容易知道每一個 protocol/module 在整個 gNB 的什麼位置。

What I learned
5G gNB 可以進行 functional split。
CU、DU、RU 負責不同部分的 RAN 功能。
OCUDU 適合先從 architecture 的角度理解 Open RAN。
下一步需要進一步確認每個 component 中有哪些 protocol。
