# 07. 這些模組可以拿來做什麼？從架構理解到實際應用

## 本章目標

本章的目標是從前面對 **OCUDU、OAI、MAC Scheduler、FAPI 與 NVIDIA Aerial L1** 的架構理解，進一步思考：

- 如果已經知道這些模組的位置與功能，實際上可以拿它們做什麼？
- 可以修改哪些地方？
- 可以觀察哪些結果？
- 可以延伸成哪些實作或研究方向？

我希望這一章不只停留在「知道這些模組是什麼」，而是進一步整理出「這些模組可以如何被拿來使用」。

## 學習紀錄

- **學習日期：** 2026/08/25
- **投入時間：** 約 2.5 小時
- **主要整理內容：** OCUDU Functional Split、OAI MAC Scheduler、FAPI、NVIDIA Aerial L1
- **本次重點：** 思考各模組可以對應哪些實作與研究方向
- **目前優先方向：** `OAI MAC Scheduler -> FAPI -> PHY`

## 7.1 Goal

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