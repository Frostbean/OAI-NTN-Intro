# OAI NTN 開源社群開發導引與架構剖析

## OAI NTN 核心術語對照表
1. 時間與同步補償 (Timing & Sync)：這是 NTN 最核心的修改部分，用於解決衛星與地面間巨大的傳播延遲。 請自行畫圖或找出參考資料呈現這些時間參數的意義。

|術語 (Symbol) | 全名 (Full Name) | 說明與 OAI 實作重點|
|-- | -- | --|
|$T_{\text{TA}}$ | Timing Advance | 提前發送量。UE 必須比預期時間更早發送訊號，以補償長距離傳輸延遲。|
|$N_{\text{TA}}$ | Timing Advance Offset | OAI 中用於調整上行鏈路定時的偏移值，NTN 的數值遠大於地面網路。|
|$K_{\text{offset}}$ | Scheduling Offset | 調度偏移。考慮到衛星往返延遲（RTT），gNB 指令發出後，UE 需要額外的等待時間才能執行。|
|$N_{\text{TA,adj}}^{\text{common}}$ | Common Timing Advance | 衛星覆蓋範圍內所有 UE 共有的延遲部分（通常指衛星到網關的距離）。|
|$N_{\text{TA,adj}}^{\text{UE}}$ | UE-specific Timing Advance | 個別 UE 因位置不同而產生的額外延遲補償。|

2. 軌道與幾何參數 (Orbits & Geometry)
在 OAI 的 rfsimulator 或 gnb_config 中，常用於定義衛星運動。

|術語 | 全名 | 說明|
|-- | -- | --|
|Ephemeris | 星曆資料 | 描述衛星位置與速度的數據，OAI 支援透過 TLE 或 PVT (Position, Velocity, Time) 輸入。|
|Doppler Shift | 多普勒頻移 | 因衛星高速移動產生的頻率偏差，OAI PHY 層需進行頻率補償。|
|Propagation Delay | 傳播延遲 | 訊號在空間中傳輸的時間，LEO 約 10-40ms，GEO 則高達 250-270ms。|

3. 網路架構與模式 (Architecture Modes)
請畫圖說明網路架構，標示以下術語

|術語 | 全名 | 說明|
|-- | -- | --|
|Transparent Payload | 透明轉發模式 | 衛星僅作為「空中反射鏡」，不處理協議棧，gNB 留在地面。這是目前 OAI NTN 主要支援的模式。|
|Regenerative Payload | 再生處理模式 | 衛星上搭載全部或部分 gNB 功能（如 gNB-DU），具有處理封包的能力。|
|SAN | Satellite Access Node | 3GPP 中對衛星基地台設備的統稱。|
|NTN-GW | NTN Gateway | 負責將地面核心網訊號傳送到衛星的地面站。|
|Feeder Link | 饋送鏈路 | 衛星與地面站 (Gateway/gNB) 之間的連線。|
|Service Link | 服務鏈路 | 衛星與終端用戶 (UE) 之間的連線。|

**以下資訊請確認，並標示章節名稱：**

| 核心領域 | 關鍵術語 | 3GPP 規範編號 (Rel-17) | 對應章節 | 說明 |
| -- | -- | -- | -- | -- |
| 總體架構 | NTN Architecture | TS 38.300 | 16.14	Non-Terrestrial Networks | 概述 NTN 的整體架構、透明模式 (Transparent) 與再生模式 (Regenerative)。|
|物理層程序 | Timing Advance ($T_{TA}$, $N_{TA}$), $K_{offset}$ | TS 38.213 |  | 這是最重要的一份。詳細定義了上行鏈路定時、隨機接入 (RACH) 以及 $K_{offset}$ 的計算公式。|
|無線資源控制 | NTN-Config, Cell Selection | TS 38.331 | | 定義 RRC 訊息中的 NTN 參數欄位，例如星曆資訊 (Ephemeris) 的格式。|
|媒體存取控制 | HARQ 停用/優化 | TS 38.321 |  | 討論在高延遲環境下，如何處理 MAC 層的 HARQ 回饋機制（或將其停用）。|
|軌道模型 | Ephemeris (PVT/TLE) | TS 38.101-5 |  | 定義衛星終端 (UE) 與基地台 (gNB) 的無線射頻特性與軌道模型規範。|

## 第一章：OAI NTN 計畫背景
### 1.1 專案源起： 介紹 OAI 如何從原本的 Terrestrial (地面) 5G 演進到支援 3GPP Rel-17 NTN 標準。

OAI 對於 NTN 標準的支援始於歐洲太空總署（European Space Agency, ESA）支持的兩個計畫：[5G-GOA](https://connectivity.esa.int/archives/projects/5ggoa) 及 [5G-LEO](https://connectivity.esa.int/archives/projects/5gleo)。

| 計畫     | 5G-GOA                                                                                                                                   | 5G-LEO                                                                                                      |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| 目標     | 根據 3GPP Rel-17，於 OAI 上實作 NTN 所需的修改，完成與實際運行的 GEO, Transparent Payload 衛星的對接測試                                       | 延伸 OAI 程式碼至支援 LEO, Transparent Payload 衛星模擬                                                     |
| 時間     | 2021/02~2024/04                                                                                                                          | 2021/12~                                                                                                    |
| 成員     | Eurescom GmbH、Fraunhofer Institute for Integrated Circuits (IIS)、Universität der Bundeswehr München、University of Luxembourg、Eurecom | Eurescom GmbH、Fraunhofer Institute for Integrated Circuits (IIS)、University of Luxembourg、Allbesmart LDA |
| 實作內容 | PHY及更上層的修改，包括同步、timers 及 random access procedure 等技術                                                                    |                                                                                                             |
| 貢獻     | 開發並發布於 OAI [goa-5g-ntn](https://gitlab.eurecom.fr/oai/openairinterface5g/-/commits/goa-5g-ntn) 分支，後續整合進 develop 分支       | 開發並發布於 develop 分支                                                                                   |

### 1.2 開源社群現狀： 說明目前主要貢獻者（如 ESA, Fraunhofer IIS, Eurecom 等）及其在社群中的地位。

與 OAI NTN 計畫相關的貢獻者包括

| 組織                                               | 貢獻內容                                    |
| -------------------------------------------------- | ------------------------------------------- |
| European Space Agency, ESA                         | 5G-GOA 及 5G-LEO 計畫的出資者、支持者       |
| Eurescom GmbH                                      |                                             |
| Fraunhofer Institute for Integrated Circuits (IIS) | OAI 在 PHY 、 MAC 層及 RFsimulator 的修改等 |
| Universität der Bundeswehr München                 |                                             |
| University of Luxembourg                           |                                             |
| Eurecom                                            |                                             |
| Allbesmart LDA                                     |                                             |
| Lasting Software                                   |                                             |

## 第二章：OAI NTN 軟體架構與關鍵技術
### 2.1 NTN 修改範疇： 說明為了支持衛星通訊，OAI 在物理層 (PHY) 與 MAC 層做了哪些改動（例如：TA 補償、隨機接入程序修改）。
### 2.2 模擬環境架構： 介紹如何使用 rfsimulator 模擬衛星的長延遲 (RTT) 與多普勒偏移 (Doppler Shift)。

在 OAI 上相關的 MR：
- for GEO：https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2712
- for LEO：https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2893

### 2.3 程式碼對照：
> openairinterface5g/openair2/LAYER2/NR_MAC_gNB/ (gNB 端的 NTN 定時修改)
openairinterface5g/targets/ARCH/rfsimulator/ (通道仿真器的實現)

## 第三章：已完成工作 (Current Milestones)
嘗試直接看 code 有點難切入，因此正在參閱 [5G Non-Terrestrial Networks With OpenAirInterface: An Experimental Study Over GEO Satellites](https://ieeexplore.ieee.org/document/10723292)，了解 OAI NTN 相關的修改。

### 3.1 基礎流程實作：已支援 LEO/GEO 的初始接入 (Initial Access) 與連線建立。

### 3.2 定時補償機制：介紹已實現的 NTN 特定參數（如 $T_{ta}$ 補償、Common TA）。

### 3.3 硬體與 SDR 支援：說明目前支援的 USRP 硬體清單及運作頻段。

根據 [USRP device documentation](https://gitlab.eurecom.fr/oai/openairinterface5g/-/tree/develop/radio/USRP)，OAI 目前支援的 USRP 包括常見的 B200, B200mini/B205mini, B210, X310, N300, N310, N320, X410。

## 第四章：待完成工作與社群挑戰 (Future Work & To-Do List)
> 這部分是兩位的「切入點」，應重點說明目前 OAI 官方正在徵求貢獻或尚未完善的部分，可參考https://docs.google.com/spreadsheets/d/1lHc8zCu4OynKzIa4UFOTOevQhWXOOJb-/edit?gid=185138274#gid=185138274：

對於現在正在開發的功能，或許可參考 clause 16.14 Non-Terrestrial Networks of TS 38.300 

### 4.1 換手流程 (Handover)： 在高速移動的 LEO 衛星間切換 (Inter-satellite Handover) 的完善。
### 4.2 下行同步優化： 針對更極端多普勒頻移下的同步演算法改進。
### 4.3 協定效能最佳化： 解決在高延遲環境下 TCP 吞吐量下降的問題（涉及 HARQ 機制的優化）。
### 4.4 大規模模擬測試： 目前多為單一 UE 仿真，缺乏多用戶在大範圍覆蓋下的效能評估。



## 第五章：如何參與 OAI 開源開發 (Developer Guide)
> 可參考：https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/code-style-contrib.md

### 5.1 開發環境建置： 快速教導如何使用 Docker 編譯與運行 OAI NTN。

參考 [How to run a NTN configuration](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/RUNMODEM.md#how-to-run-a-ntn-configuration) 可以找到 GEO 及 LEO 衛星模擬的設定。(目前已有根據以下設定跑通測試，確認 UE 與 gNB 能夠相連)

#### GEO Simulation

gNB
```bash
cd cmake_targets
sudo ./ran_build/build/nr-softmodem -O ../ci-scripts/conf_files/gnb.sa.band254.u0.25prb.rfsim.ntn.conf --rfsim
```

UE
```bash
cd cmake_targets
sudo ./ran_build/build/nr-uesoftmodem -O ../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf --band 254 -C 2488400000 --CO -873500000 -r 25 --numerology 0 --ssb 60 --rfsim --rfsimulator.prop_delay 238.74
```

`--rfsimulator.prop_delay`: 設定 gNB 與 UE 之間的 propagation delay。

#### LEO Simulation

gNB
```bash
cd cmake_targets
sudo ./ran_build/build/nr-softmodem -O ../ci-scripts/conf_files/gnb.sa.band254.u0.25prb.rfsim.ntn-leo.conf --rfsim
```

UE
```bash
cd cmake_targets
sudo ./ran_build/build/nr-uesoftmodem -O ../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf --band 254 -C 2488400000 --CO -873500000 -r 25 --numerology 0 --ssb 60 --rfsim --rfsimulator.prop_delay 20 --rfsimulator.options chanmod --time-sync-I 0.1 --ntn-initial-time-drift -46 --initial-fo 57340 --cont-fo-comp 2
```

### 5.2 Git 工作流： 說明如何向 GitLab 提交 Merge Request (MR)，以及 CI/CD 測試流程。

### 5.3 相關文件獲取： 介紹3GPP TS 38.300/38.213 等相關 NTN 標準文件。
> 兩位可使用以下三個方法在報告中列出「待完成」功能：
> 1. 查閱 GitLab Issue： 到 OAI 的官方 GitLab 搜尋標籤為 NTN 的 Issues。
> 2. 比對 3GPP 標準： 將目前的 OAI 代碼與 3GPP TS 38.300 (Rel-17 NTN 部分) 進行對照，看看哪些 RRC 訊息或參數尚未在程式碼中定義。
> 3. 分析開發日誌 (Changelog)： 觀察最近的 Commit，看看開發者們在討論什麼技術痛點。


## 附錄
### Bug Report 撰寫練習

### Code Review 範例

## 參考資料

規格的描述相當抽象，需要額外的閱讀材料輔助理解。以下是已經大致閱讀完畢的補充材料，

5G-GOA 及 5G-LEO 相關資料：
https://openairinterface.org/wp-content/uploads/2024/09/20240913_Fraunhofer_IIS_OAI_NTN_reduced.pdf
https://connectivity.esa.int/archives/projects/5ggoa
https://connectivity.esa.int/archives/projects/5gleo

How to run NTN configuration: 
https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/RUNMODEM.md#how-to-run-a-ntn-configuration

衛星軌道對通訊的影響：
https://www.recast.ncu.edu.tw/static/file/17/1017/img/211/409626021.pdf
- 重要的資訊包含
    - 軌道六元素（軌道參數）
    - 通訊仰角
    - 衛星軌道參數對通訊的影響，例如 time/frequency synchronization

[Physical layer enhancements in 5G-NR for direct access via satellite systems](https://onlinelibrary.wiley.com/doi/abs/10.1002/sat.1461)
- 業界專家介紹 3GPP 在 Rel-17 對 Physical layer for NTN 的進一步優化，例如：
    - Enhancement on UL time and frequency synchronization
    - Enhancements on Hybrid-ARQ

[Uplink Time Synchronization Method and
Procedure in Release-17 NR NTN](https://ieeexplore.ieee.org/document/9860357)
- 業界專家詳細介紹 3GPP 在 Rel-17 對 time synchronization for NTN 的進一步優化