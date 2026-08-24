02. OCUDU Architecture and Interfaces
2.1 整體架構

OCUDU 可以進一步拆成：

                    5G Core
                 AMF       UPF
                  │         │
                 N2        N3
                  │         │
              CU-CP ← E1 → CU-UP
                  │         │
                F1-C      F1-U
                  │         │
                  └────┬────┘
                       │
                    DU-high
                   RLC + MAC
                       │
                      FAPI
                       │
                    DU-low
                   Upper PHY
                       │
                Open Fronthaul
                       │
                      RU
                 Lower PHY + RF
                       │
                      UE
2.2 Protocol 對應
Component	主要功能
CU-CP	RRC / Control Plane
CU-UP	SDAP / PDCP / User Plane
DU-high	RLC / MAC / Scheduler
DU-low	Upper PHY
RU	Lower PHY / RF

可以簡化成：

RRC
 ↓
PDCP / SDAP
 ↓
RLC
 ↓
MAC
 ↓
PHY
 ↓
RF

對應：

CU-CP
 ↓
CU-UP
 ↓
DU-high
 ↓
DU-low
 ↓
RU
2.3 重要 Interface
Interface	Connection	主要用途
NG-C / N2	CU-CP ↔ AMF	Core Control Plane
NG-U / N3	CU-UP ↔ UPF	Core User Plane
E1	CU-CP ↔ CU-UP	CP/UP Split
F1-C	CU-CP ↔ DU	Control
F1-U	CU-UP ↔ DU	User Data
FAPI	MAC ↔ PHY	L2/L1
Open Fronthaul	DU ↔ RU	PHY / Radio
E2	CU/DU ↔ RIC	RAN Control

最重要的三個 split：

CU-CP ← E1 → CU-UP

代表：

Control Plane / User Plane Split

CU ← F1 → DU

代表：

CU / DU Split

MAC ← FAPI → PHY

代表：

Layer 2 / Layer 1 Split

What I learned
CU/DU 並不只是不同程式，而是不同 protocol functions 的 functional split。
E1 處理 CU-CP / CU-UP split。
F1 處理 CU / DU split。
FAPI 是 MAC 與 PHY 之間的重要邊界。
這些介面是後續理解 OAI 與 Aerial integration 的基礎。
