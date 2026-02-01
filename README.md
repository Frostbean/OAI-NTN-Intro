# OAI NTN Open Source Development Guide and Architecture Analysis

## Key 5G NTN Terminologies Table (Rel-17)

To help readers who are familiar with 5G but not with the NTN enhancements introduced in Rel-17 quickly grasp the key concepts, the following tables summarize the fundamental aspects of NTN at both the physical layer and protocol layer.

Unless otherwise stated, the terminology in these tables is primarily derived from Clause 16.14 Non-Terrestrial Networks of TS 38.300.

Note: The purpose of these tables is not to exhaustively cover all NTN enhancements, nor to provide perfectly formal or complete definitions.

### Primary spec sources

- **TS 38.300 — *NR and NG-RAN Overall Description***
  - **Clause 16.14 — *Non-Terrestrial Networks***: Provides the overall NTN RAN architecture and a high-level summary of key NTN enhancements (e.g., timing/synchronization, frequency aspects, and mobility).

- **TS 38.213 — *Physical layer procedures for control***
  - **Clause 4.2 — *Transmission timing adjustments***: Specifies the control procedures and signaling related to uplink transmission timing (e.g., how TA is applied/updated and under what conditions timing adjustments are triggered).

- **TS 38.211 — *Physical channels and modulation***
  - **Clause 4.3.1 — *Frames and subframes***: Defines the NR frame/subframe structure and the basic timing reference used by PHY timing expressions (useful when mapping TA-related quantities to time units such as $T_{c}$).

- **TS 38.321 — *Medium Access Control (MAC) protocol specification***
  - **Clause 5.3.2.2 and Clause 5.7**: Defines downlink HARQ operation, including per-HARQ-process enabling/disabling of HARQ feedback (used in NTN to mitigate HARQ stalling).
  - **Clause 5.4.3.1 and Clause 5.7**: Defines uplink HARQ mode configuration (HARQ mode A / HARQ mode B) per HARQ process.

- **TS 38.331 — *Radio Resource Control (RRC) protocol specification***
  - **Clause 6.3.1 — *System information blocks (SIBs), SIB19***: Provides the ASN.1 definitions for SIB19, including NTN-related broadcast information (e.g., `ntn-Config` and other NTN assistance fields).

### 1) System architecture

| Term | Description |
|---|---|
| Transparent Payload | The satellite only relays signals and does not run the RAN protocol stack; the gNB stays on the ground. |
| Regenerative Payload | The satellite hosts all or part of the gNB functions (e.g., gNB-DU) and can process packets/protocols.|
| NTN-GW | Ground gateway that forwards traffic between the terrestrial network/core and the satellite. |
| Feeder Link | Link between the satellite and the ground station (Gateway/gNB). |
| Service Link | Link between the satellite and the user equipment (UE). |
| earth-fixed cell | Beam/cell coverage area stays on the same geographical region on Earth over time (Earth coordinates). |
| earth-moving cell | Beam/cell coverage area slides over the Earth surface with satellite motion. |


![image](https://www.3gpp.org/images/2024/NTN_Fig2.jpg)
**Transparent** vs. **Regenerative** payload and elliptic beam patterns [1]

![image](https://media.discordapp.net/attachments/1175780348093792427/1467566469268115670/By2xanhIWl.png?ex=6980d948&is=697f87c8&hm=9140923a6c383c43ae04880ff91822c496cf4c80c0a9b572dc93572a4a38c9a5&=&format=webp&quality=lossless&width=326&height=645)

Overall illustration of an NTN with transparent payload [2]

![image](https://media.discordapp.net/attachments/1175780348093792427/1467566469603791020/Byp763hUZx.png?ex=6980d948&is=697f87c8&hm=26f5fcd93117bfe673e2a8f2b3a24d49672a000a7cae5072f1ec5d97fa3936b2&=&format=webp&quality=lossless&width=765&height=678)

Overall illustration of an NTN with **regenerative** payload hosting a gNB  [2]

![image](https://media.discordapp.net/attachments/1175780348093792427/1467566470069354537/rkP5p33Ubx.png?ex=6980d948&is=697f87c8&hm=86677354edc09628152585a5ff9d3650a49b1b46de3e5d395f90fdffefd40be2&=&format=webp&quality=lossless&width=947&height=662)

Illustration of timing relationship, including $T_{\text{TA}}$, service link RTT, common TA, and $k_{\text{mac}}$  (for collocated gNB and NTN Gateway) [2]

![image](https://free5gc.org/blog/20240626/Service_link_types.png)

Service Link Types, including **Earth-fixed**, **Quasi-Earth-fixed**, **Earth-moving** [3]

### 2) Timing advance & scheduling

| Term | Description |
|---|---|
| uplink timing reference point (RP) | DL and UL frames are aligned at the uplink time synchronization reference point (RP). See TS 38.300 and TS 38.211. |
| service link RTT | Round-trip time over the service link (UE ↔ NTN payload). |
| feeder link RTT | Round-trip time over the feeder link (NTN-GW ↔ NTN payload). |
| $T_{\text{TA}}$ | The UE transmits earlier than the nominal timing to compensate for the long propagation delay. See TS 38.300, TS 38.213 and TS 38.211 |
| $N_{\text{TA}}$ | Timing advance value/command used to adjust uplink transmission timing (in units of NR timing $T_{c}$). See TS 38.300, TS 38.213 and TS 38.211 |
| $N_{\text{TA,offset}}$ | Offset used for DL/UL frame alignment at RP. Its default value depends on frequency, duplex mode, and coexistence with other RATs. See TS 38.300, TS 38.213 and TS 38.211 |
| $N_{\text{TA,adj}}^{\text{common}}$ | Common TA adjustment: timing offset to compensate the propagation delay common to UE transmissions, i.e., the delay between the serving NTN payload (satellite) and RP. See TS 38.300, TS 38.213 and TS 38.211 |
| $N_{\text{TA,adj}}^{\text{UE}}$ | UE-specific TA adjustment: timing offset to compensate the UE-specific service-link delay, i.e., the propagation delay between the UE and the serving satellite/payload. See TS 38.300, TS 38.213 and TS 38.211 |
| $K_{\text{offset}}$ | Configured scheduling offset added to several timing relationships so that an uplink transmission occurs after the corresponding downlink reception at the UE (must be large enough considering service-link RTT and Common TA). |
| $k_{\text{mac}}$ | Configured offset approximately equal to the RTT between RP and gNB, used to delay when a downlink configuration indicated by MAC CE takes effect (also used for UE–gNB RTT related procedures). |

The overall TA is computed as:

$T_{\text{TA}} = (N_{\text{TA}} + N_{\text{TA,offset}} + N_{\text{TA,adj}}^{\text{common}} + N_{\text{TA,adj}}^{\text{UE}}) * T_{c}$

where $N_{\text{TA}}$ denotes the closed-loop TA which can be updated based on TA Command, $N_{\text{TA,offset}}$ is a fixed offset related
to frequency as in terrestrial networks, $T_{c}$ is the basic time
unit, and $N_{\text{TA,adj}}^{\text{common}}$ and $N_{\text{TA,adj}}^{\text{UE}}$ are open-loop TA which corresponds to common TA and UE-specific TA, respectively. [4]

![image](https://media.discordapp.net/attachments/1175780348093792427/1467566470434132121/r11WH6hU-x.png?ex=6980d948&is=697f87c8&hm=cdd8bf8fe8ed506912f15efde36e879d9af49c9ad3b70acfbc156f2e464f0186&=&format=webp&quality=lossless&width=996&height=744)

Pictorial view of the TA calculation in NTN, and UL/DL radio frame timing at the NTN UE [5]

![image](https://media.discordapp.net/attachments/1175780348093792427/1467566470845436239/HyVyZa3U-g.png?ex=6980d948&is=697f87c8&hm=9a1372e92cc53df739728e63a9a55970b3b09144c6f47948d0e0cf28cc7e6d59&=&format=webp&quality=lossless&width=1299&height=744)

An illustration of PUSCH transmission timing with and without $K_{\text{offset}}$ enhancement [5]

![image](https://media.discordapp.net/attachments/1175780348093792427/1467566471126319116/HJVS-6nL-l.png?ex=6980d948&is=697f87c8&hm=f7129b0d61f6ac38f2bf1d099090ab03aa2ddc399e4f88871b609fb5b445fc72&=&format=webp&quality=lossless&width=1575&height=744)

An illustration of MAC CE timing relationship with and without $k_{\text{mac}}$ enhancement when downlink and uplink frame timing are not aligned at gNB [5]

### 3) Frequency synchronization

| Term | Description |
|---|---|
| Doppler Shift | Frequency offset caused by the satellite’s high speed; the PHY needs frequency/Doppler compensation. |
| Doppler compensation | Receiver-side frequency correction/tracking to counter Doppler-induced frequency shifts (Rx-side correction). |
| Doppler pre-compensation | Transmitter-side pre-correction applied before uplink transmission so the uplink arrives frequency-aligned despite service-link Doppler (Tx-side pre-correction). |


### 4) Orbits & geometry

| Term | Description |
|---|---|
| Ephemeris | Orbit/state data describing the satellite/payload position and velocity over time. See TS 38.300 and TS 38.331. |
| SIB19 | System information block used to broadcast NTN-specific information such as ephemeris and assistance information for UL time/frequency synchronization. See TS 38.300 and TS 38.331. |

For a curated summary of NTN UL synchronization assistance parameters (e.g., ephemeris-related fields carried in SIB19), see [5], which extracts and organizes the relevant items from the SIB19 ASN.1 definitions.

For the full ASN.1 definition of SIB19, refer to [6] (useful when implementing or parsing the RRC messages).

### 5) Enhancement on Hybrid-ARQ

In large-RTT NTN deployments, the stop-and-wait nature of HARQ can cause **HARQ stalling**, since the UE may need to wait much longer for DL packets or UL grants when HARQ retransmissions are used. Rel-17 mitigations include increasing the number of HARQ processes and **disabling HARQ feedback** (with reliability ensured by RLC ARQ instead of MAC HARQ retransmissions).

| Term | Description |
|---|---|
| disabling HARQ feedback (DL) | HARQ feedback can be enabled/disabled **per HARQ process**. When feedback is disabled, there is **no MAC-layer HARQ retransmission**; reliability is handled by **RLC ARQ**. Disabling feedback also enables scheduling the same HARQ process before one HARQ RTT has elapsed. See TS 38.300 and TS 38.321. |
| HARQ mode A (UL) | UL HARQ mode configurable **per HARQ process** (baseline UL HARQ operation; see TS 38.321 for the exact behavior). See TS 38.300 and TS 38.321. |
| HARQ mode B (UL) | UL HARQ mode configurable **per HARQ process**. **Mode B allows scheduling a HARQ process before one HARQ RTT has elapsed since it was last scheduled**, reducing HARQ stalling under large RTT. See TS 38.300 and TS 38.321. |

### 6) Mobility enhancement

In NTN deployments, the beam/cell footprint may be earth-fixed or may move smoothly over the Earth surface, so the received signal strength/quality can vary more slowly than in terrestrial networks. As a result, purely RRM measurement-based CHO triggering (e.g., relying only on fast signal-level variations) may become less efficient. To better align handover triggering with predictable NTN geometry and timing, Rel-17 allows **timer-based** and **location-based** trigger conditions to be configured together with CHO, so that the UE performs the conditional handover when the measurement condition is met and the additional time/location condition is also satisfied.

| Term | Description |
|---|---|
| CHO | Conditional Handover: UE is pre-configured with target cells and trigger conditions, and executes handover when the trigger is met. |
| timer-based trigger condition | CHO trigger uses an additional time condition (e.g., trigger when a timer expires), used together with a measurement-based trigger. See TS 38.300 and TS 38.331 |
| location-based trigger condition | CHO trigger uses an additional location condition (e.g., trigger when the UE reaches a configured area/position), used together with a measurement-based trigger. See TS 38.300 and TS 38.331 |

### Reference of Figures

[1] https://www.3gpp.org/technologies/ntn-overview

[2] [TS 38.300 - 16.14 Non-Terrestrial Networks](https://www.etsi.org/deliver/etsi_ts/138300_138399/138300/17.00.00_60/ts_138300v170000p.pdf)

[3] https://free5gc.org/blog/20240626/20240626/#transparent-mode

[4] [Uplink Time Synchronization Method and Procedure in Release-17 NR NTN](https://ieeexplore.ieee.org/document/9860357)

[5] [Physical Layer Enhancements in 5G-NR for Direct Access via Satellite Systems](https://www.researchgate.net/publication/362737767_Physical_layer_enhancements_in_5G-NR_for_direct_access_via_satellite_systems)

[6] https://www.sharetechnote.com/html/5G/5G_NTN.html#NTN_Config_r17

### Supplementary Materials

In addition to the 3GPP specifications cited above, the following references are useful for clarifying the terminology and building intuition for Rel-17 NTN enhancements. The following materials, which we have read thoroughly, may help you build a solid foundation for 5G NTN. For NTN physical-layer enhancements, we strongly recommend the technical report **Physical Layer Enhancements in 5G-NR for Direct Access via Satellite Systems**.

- **Overview of physical-layer enhancements**
  - *Physical Layer Enhancements in 5G-NR for Direct Access via Satellite Systems*
- **Time synchronization**
  - *Uplink Time Synchronization Method and Procedure in Release-17 NR NTN* (covers $N_{\text{TA,adj}}^{\text{common}}$, $N_{\text{TA,adj}}^{\text{UE}}$)
- **Timing-relationship enhancements: $K_{\text{offset}}$, $k_{\text{mac}}$**
  - $K_{offset}$
    - Before studying $K_{\text{offset}}$, it helps to understand basic scheduling parameters such as $K_0$, $K_1$, and $K_2$:
      https://www.sharetechnote.com/html/5G/5G_ResourceAllocation.html
  - $k_{\text{mac}}$
    - TS 38.300: definition and context. See Clause 16.14.2.1 *Scheduling and Timing*
    - *Physical Layer Enhancements in 5G-NR for Direct Access via Satellite Systems*: describes how $k_{\text{mac}}$ works in practice
    - *ALifecom White Paper — Unlocking Connectivity Frontiers: Exploring IoT NTN Innovations*: although it focuses on IoT-NTN, it explains the relationship between $k_{\text{mac}}$ and implementation/design complexity (useful for intuition)
      https://www.oversight24.com/wp-content/uploads/2024/04/ALifecom-White-Paper_Unlocking-Connectivity-Frontiers-Exploring-IoT-NTN-Innovations.pdf
  - A short explanation of $k_{\text{offset}}$ and $k_{\text{mac}}$:  
    https://www.scribd.com/document/847612531/Additional-Material-to-5G-NTN-Course
- **Frequency synchronization**
  - Frequency pre-compensation: appears to be left to network implementation
  - Potential reference (not yet validated): *Ephemeris-Assisted Doppler Frequency Compensation in Satellite Communication Systems*
- **Enhancements on Hybrid-ARQ**
  - LTE basic procedures (HARQ background): https://www.sharetechnote.com/html/BasicProcedure_LTE_HARQ.html#ProcessID_Asynchronous_HARQ
  - 5G/NR HARQ overview:  
    https://www.sharetechnote.com/html/5G/5G_HARQ.html
  - HARQ mathematical concept (video):  
    https://youtu.be/6deFGsOG5yY?si=o83fGK-SKAHjNdko
  - HARQ procedure walkthrough (video):  
    https://youtu.be/SGFnqdpSOs4?si=X3M7FxMj-hMjatuS
  - Disabling HARQ feedback (practical note):  
    https://www.linkedin.com/posts/gleb-marchenko-72161aba_harq-5g-ntn-activity-7201575539889885184-Xzqx/
- **Enhancements on mobility**
  - *An Overview of Mobility Enhancements for Non-Terrestrial Networks in 3GPP 5G NR System*
    (covers not only Rel-17 but also Rel-18 and later; still useful for a broader understanding)
- **OAI NTN modifications and experiments for GEO**
  - *5G Non-Terrestrial Networks With OpenAirInterface: An Experimental Study Over GEO Satellites* 
     https://ieeexplore.ieee.org/document/10723292

## Chapter 1: OAI NTN Project Background

### 1.1 OAI NTN Projects

OAI’s support for the 3GPP Release 17 NTN specifications started from two projects funded by the European Space Agency (ESA): [5G-GOA](https://connectivity.esa.int/archives/projects/5ggoa) and [5G-LEO](https://connectivity.esa.int/archives/projects/5gleo).

| Item | 5G-GOA | 5G-LEO |
|---|---|---|
| Goal | Implement the required NTN modifications in OAI based on 3GPP Rel-17, and validate integration/testing with a real GEO transparent-payload satellite system. | Extend the OAI codebase to support LEO transparent-payload satellite simulation. |
| Timeline | 2021/02–2024/04 | 2021/12–Ongoing |
| Members | Eurescom GmbH; Fraunhofer Institute for Integrated Circuits (IIS); Universität der Bundeswehr München; University of Luxembourg; Eurecom | Eurescom GmbH; Fraunhofer Institute for Integrated Circuits (IIS); University of Luxembourg; Allbesmart LDA |
| Implementation scope | Changes from PHY up to higher layers, including synchronization, timers, and random access procedures. | Changes from PHY up to higher layers, including timing drift compensation, 32 HARQ process support, LEO-specific timer extensions, handover and paging protocols. |
| Contribution | Developed and released on the OAI branch [goa-5g-ntn](https://gitlab.eurecom.fr/oai/openairinterface5g/-/commits/goa-5g-ntn), later integrated into the `develop` branch. | Developed and released on the `develop` branch. |


### 1.2 Contributors

Contributors related to OAI NTN includes:

- European Space Agency (ESA): Funder of 5G-GOA and 5G-LEO
- Eurescom GmbH: Prime contractor of 5G-GOA and 5G-LEO
- Fraunhofer Institute for Integrated Circuits (IIS): Modifications to OAI PHY/MAC and RFsimulator
- Universität der Bundeswehr München: PoC tests and Over-the-air tests of GEO
- University of Luxembourg
- Eurecom: Founder of OAI
- Allbesmart LDA
- Lasting Software: Implementaion of SIB19 and NTN CI test


## Chapter 2: Current Milestone
### Overview of Relating Merge Requests
> https://docs.google.com/spreadsheets/d/1lHc8zCu4OynKzIa4UFOTOevQhWXOOJb-/edit?gid=185138274#gid=185138274

- RFSimulator
    - GEO, [!2712](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2712)
    - LEO, [!2893](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2893)
- NTN support gNB, [!2722](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2722)
- NTN support UE, [!2723](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2723)
- Disable HARQ, [!2749](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2749)
- Support 32 HARQ proc, [!2876](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2876)
- Some fixes for NTN, [!2995](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2795)
    - some commits of them are not listed below because...
- SIB19
    - gNB, [!2899](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2899)
    - UE, [!2920](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2920), [!3019](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/3019)
### Timing Advance
- [gNB: change TA update interval to 100 frames instead of 10](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2722/diffs?commit_id=8e4526e22189becf154e8649e079b045a35e0a8e)
- [NR UE: add NTN specific parameter ta-Common and use it for initial timing advance, RA_contention_resolution_timer and RA_window_cnt](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2723/diffs?commit_id=75efcf49b9d660b8146a4936ef69821a5665de66)


### Scheduling
- [gNB: add NTN specific parameter cellSpecificKoffset_r17 and use it for UL scheduling](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2722/diffs?commit_id=ccba7c875df082c924f65a65c5b8ec2996d5d624)
- [gNB: fix phytest scheduler in case no HARQ processes are available](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2722/diffs?commit_id=6b01cfb8e2f0892c6edead8d57b6796688da3a55)
- [gNB: add sr_ProhibitTimer_v1700](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2722/diffs?commit_id=4cdf4d61f3350003ef1f0ff7074ea95b579d9e88)
- [gNB: make SR timers configurable in CONF file](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2722/diffs?commit_id=90fa3d5534001676e0dc98ed311e371fb3f56981)
- [gNB: make ue_TimersAndConstants configurable via conf file](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2722/diffs?commit_id=f1938acbab0d75f1591eb021b7760c611f5c3de6)
- [
NR UE: add NTN specific parameter cellSpecificKoffset_r17 and use it for UL scheduling](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2723/diffs?commit_id=26175c5413219f26d299d242ebdbd29796b23e4a)
- [
NR UE: increase DURATION_RX_TO_TX by cellSpecificKoffset_r17](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2723/diffs?commit_id=021f97cf10adffb15397b1d9ae05321d78ede337)
- [NR UE: use sr_ProhibitTimer_v1700 if present](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2723/diffs?commit_id=1e473112597cf4f87cfb76256ac4c1306ba1be26)
- [gNB: consider NTN_Koffset when scheduling PRACH](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2995/diffs?commit_id=64380eaca2add4f4b6b4d4af6211ed53267651f0)

### Frequency Synchronization
### Initial Access (Random Access Procedure)
- [gNB: use 2 * K2 for contention_resolution_timer, as the timer starts with Msg2...](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2722/diffs?commit_id=be41a4ea60cd83c89af94ae906dc90b1a168cb31)
<!-- 2.1 NTN 修改範疇： 說明為了支持衛星通訊，OAI 在物理層 (PHY) 與 MAC 層做了哪些改動（例如：TA 補償、隨機接入程序修改）。 -->
<!-- 3.1 基礎流程實作：已支援 LEO/GEO 的初始接入 (Initial Access) 與連線建立。 -->
<!-- ### 3.2 定時補償機制：介紹已實現的 NTN 特定參數（如 $T_{ta}$ 補償、Common TA）。 -->

### Enhancements on HARQ
- [gNB: make checks for missed feedbacks more robust w.r.t. frame number warp-around in find_harq()](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2722/diffs?commit_id=0ce0282238e617bb3758275723b21503f7bf7adb)
- [MR-Disable HARQ](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2749/commits)
- [MR-Support 32 HARQ proc](https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2876)

### SIB19

### Simulation with RFSimulator (Long Propagation Delay & Doppler Shift)
<!-- 2.2 模擬環境架構： 介紹如何使用 rfsimulator 模擬衛星的長延遲 (RTT) 與多普勒偏移 (Doppler Shift)。 -->

Related MRs:
- for GEO https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2712
- for LEO https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/2893

### USRP Hardware Support
<!-- 3.3 硬體與 SDR 支援：說明目前支援的 USRP 硬體清單及運作頻段。 -->

According to [USRP device documentation](https://gitlab.eurecom.fr/oai/openairinterface5g/-/tree/develop/radio/USRP), OAI currently supports USRP devices including B200, B200mini/B205mini, B210, X310, N300, N310, N320, X410.

## Chapter 3: Future Work & To-Do List
<!-- ### 4.1 換手流程 (Handover)： 在高速移動的 LEO 衛星間切換 (Inter-satellite Handover) 的完善。
### 4.2 下行同步優化： 針對更極端多普勒頻移下的同步演算法改進。
### 4.3 協定效能最佳化： 解決在高延遲環境下 TCP 吞吐量下降的問題（涉及 HARQ 機制的優化）。
### 4.4 大規模模擬測試： 目前多為單一 UE 仿真，缺乏多用戶在大範圍覆蓋下的效能評估。 -->
### Handover

### Optimization of Downlink Synchronization

### WIP
<!-- > 兩位可使用以下三個方法在報告中列出「待完成」功能：
> 1. 查閱 GitLab Issue： 到 OAI 的官方 GitLab 搜尋標籤為 NTN 的 Issues。
> 2. 比對 3GPP 標準： 將目前的 OAI 代碼與 3GPP TS 38.300 (Rel-17 NTN 部分) 進行對照，看看哪些 RRC 訊息或參數尚未在程式碼中定義。
> 3. 分析開發日誌 (Changelog)： 觀察最近的 Commit，看看開發者們在討論什麼技術痛點。 -->

## Chapter 4: Developer Guide
> https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/code-style-contrib.md

### Build OAI NTN

[How to run a NTN configuration](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/RUNMODEM.md#how-to-run-a-ntn-configuration) introduces settings for simulating GEO and LEO satellite.

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

### GitLab Workflow

## Appendix
### Example of Bug Report
### Example of Code Review
### 其他參考資料

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

[3GPP NTN Questions](/4xCZ46aVSlSe3d3rPh2RJw)

TS 38.101-5, https://gitlab.eurecom.fr/oai/openairinterface5g/-/merge_requests/3093/diffs?commit_id=db629e6b585a3d0d5453fac5bedbb354ad2a94d1
