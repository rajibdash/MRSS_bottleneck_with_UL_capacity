# Technical Analysis: Uplink Capacity Bottlenecks in 5G/6G MRSS Schedulers and 6G Architectural Design

## 1. Executive Summary
Multi-RAT Spectrum Sharing (**MRSS**) serves as the primary mechanism for migrating valuable Frequency Range 1 (**FR1**) spectrum from 5G to 6G dynamically. While MRSS benefits from 5G's "ultra-lean" air interface design (greatly reducing the heavy, always-on common reference signal overheads that plagued 4G-5G DSS), it introduces critical capacity bottlenecks within the radio scheduler domain, particularly concerning the **uplink (UL) path and control-channel resources**. 

This document evaluates these scheduler bottlenecks and presents architectural design frameworks for native 6G deployments to mitigate them.

---

Typical Mid-Band 10 ms MRSS Frame Diagram (100 MHz TDD @ 30 kHz SCS)

```text
?

10 ms Radio Frame = 10 Subframes (SF 0–9) = 20 Slots (Slot 0–19 @ 0.5 ms/slot, 30 kHz SCS)
◄────────────── 2.5 ms Cycle 0 ──────────────►◄─ 2.5 ms Cycle 1 ─►◄─ 2.5 ms Cycle 2 ─►◄─ 2.5 ms Cycle 3 ─►
 SF 0 (S0–S1)  │ SF 1 (S2–S3)  │ SF 2 (S4)    │ SF 2–4 (S5–S9)   │ SF 5–7 (S10–S14) │ SF 7–9 (S15–S19)
  D      D     │  D      S     │  U           │  D  D  D  S  U   │  D  D  D  S  U   │  D  D  D  S  U
▲
│ ┌────────────┬───────────────┬──────────────┬──────────────────┬──────────────────┬──────────────────┐
│ │ Upper Guard (~0.86 MHz)    │              │                  │                  │                  │
│ ├────────────┼───────────────┼──────────────┼──────────────────┼──────────────────┼──────────────────┤
│ │            │ [S3] DL: 6G   │ [S4] 5G/6G   │                  │                  │                  │ PRB 265–272
│ │            │  PDSCH (0-9)  │ PUCCH (Edge) │                  │                  │                  │ (Band Edge)
│ │ 6G PDSCH   ├───────────────┤ 6G PUSCH     │  Dynamic 5G + 6G │  Dynamic 5G + 6G │  Dynamic 5G + 6G │
│ │(PRB 147–272│ [S3] Guard    │ (PRB 147-264)│  DL/UL Sharing   │  DL/UL Sharing   │  DL/UL Sharing   │
│ ├────────────┤ Period (GP)   ├──────────────┤  (Slots 5–9)     │  (Slots 10–14)   │  (Slots 15–19)   │
│ │ 5G SSB     │ (Sym 10–11)   │ 5G PUSCH     │                  │                  │                  │ PRB 127–146
│ │ (20 PRBs,  ├───────────────┤ (PRB 20-146) │                  │                  │                  │ (Rate-Matched
│ │  6G Rate-  │ [S3] UL: SRS  ├──────────────┤                  │                  │                  │  by 6G)
│ │  Matched)  │ / Short PUCCH │ 5G/6G PRACH  │                  │                  │                  │ PRB 8–19
│ ├────────────┤ (Sym 12–13)   │ (PRB 8-19)   │                  │                  │                  │ (RACH Occasion)
│ │ 5G PDSCH   │               │ 5G/6G PUCCH  │                  │                  │                  │ PRB 0–7
│ │ (PRB 0-126)│               │ (PRB 0-7)    │                  │                  │                  │ (Band Edge)
│ ├────────────┴───────────────┴──────────────┼──────────────────┼──────────────────┼──────────────────┤
│ │ Lower Guard (~0.86 MHz)                   │                  │                  │                  │
│ └───────────────────────────────────────────┴──────────────────┴──────────────────┴──────────────────┘
│  Slot 0–2: DL Slots (PDCCH in Sym 0–1)        Slot 4: UL Slot     Slots 5–9          Slots 10–14        Slots 15–19
│  (SSB Burst in Half-Frame 0: Slots 0–1)      (PRACH every 5ms)   (No SSB; full      (Repeat cycle;     (Repeat cycle;
│                                                                   273 PRBs data)     PRACH in Slot 14)  full 273 PRBs)
└──────────────────────────────────────────────────────────────────────────────────────────────────────────────────► Time

```

Full 10 ms Frame Time–Frequency Arrangement (400 MHz / 264 PRBs)

```text
?

10 ms Radio Frame = 10 Subframes (SF 0–9) = 80 Slots (Slot 0–79 @ 0.125 ms/slot)
◄──────────────────── HALF-FRAME 0 (0–5 ms: Slots 0–39) ────────────────────►◄─── HALF-FRAME 1 (5–10 ms: Slots 40–79) ───►
 SF 0 (Slots 0–7) │ SF 1 (8–15) │ SF 2 (16–23) │ SF 3 (24–31) │ SF 4 (32–39) │ SF 5 (40–47) │ ... │ SF 9 (72–79)
 D D D S U │ D D D S U │ D D D S U │ D D D S U │ D D D S U │ D D D S U │ D D D S U │ D D D S U │ ... │ D D D S U  (16 Cycles)
▲
│ ┌──────────────────────┬──────────────────────────┬──────────────────────┬──────────────────────────┐
│ │ Upper Guard (~9.9MHz)│                          │                      │                          │
│ ├──────────────────────┼──────────────────────────┼──────────────────────┼──────────────────────────┤
│ │  6G PDSCH (PRB 140–  │ 6G PDSCH                 │ [S] DL: 6G PDSCH     │ [U] 5G/6G PUCCH (256-263)│ PRB 256–263
│ │  263) + Shared PDCCH │ (PRB 140–263)            │     (Sym 0–9)        │     6G PUSCH (140–255)   │
│ ├──────────────────────┤                          ├──────────────────────┼──────────────────────────┤
│ │  SSB Burst Window    │ 5G PDSCH                 │ [S] Guard Period (GP)│ [U] 5G PUSCH             │ PRB 120–139
│ │  (PRB 120–139, 6G    │ (PRB 0–139)              │     (Sym 10–11: RF   │     (PRB 20–139)         │ (SSB in SF0)
│ │   Rate-Matched)      │                          │      Switch / TA)    │                          │
│ ├──────────────────────┤                          ├──────────────────────┼──────────────────────────┤
│ │  5G PDSCH (PRB 0–119)│                          │ [S] UL: SRS / Short  │ [U] 5G/6G PRACH (8–19)   │ PRB 8–19
│ │  + Shared PDCCH      │                          │     PUCCH (Sym 12–13)│     5G/6G PUCCH (0–7)    │ PRB 0–7
│ ├──────────────────────┼──────────────────────────┼──────────────────────┼──────────────────────────┤
│ │ Lower Guard (~9.9MHz)│                          │                      │                          │
│ └──────────────────────┴──────────────────────────┴──────────────────────┴──────────────────────────┘
│   Cycle 1: Slots 0–2     Cycle 2–16: Standard       Special Slot (`S`)     Uplink Slot (`U`)
│   (DL + SSB + PDCCH)     DL Slots (PDCCH+PDSCH)     (Every 5th Slot:       (Every 5th Slot:
│                                                      Slots 3, 8, 13...)     Slots 4, 9, 14...; PRACH
│                                                                             in Slots 4 & 44)
└───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────► Time

```

To Understand...(NOT verified)

```text
?

┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                    ERICSSON RAN COMPUTE HW (e.g., RAN Processor 6672 / Ericsson Silicon)        │
│                                                                                                 │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  UNIFIED L2 JOINT SCHEDULER & AI-NATIVE RRM (Tensor Cores + Many-Core Compute)            │  │
│  │  • Maintains real-time PRB occupancy bitmaps across both carriers every slot (0.125/0.5ms)│  │
│  │  • Performs 6G Carrier Aggregation (CA) & Dynamic UL/DL Carrier Decoupling                │  │
│  └─────────────────────────┬───────────────────────────────────────────┬─────────────────────┘  │
│                            │                                           │                        │
│                            ▼                                           ▼                        │
│  ┌───────────────────────────────────────────────┐   ┌───────────────────────────────────────┐  │
│  │ CARRIER 1: 400 MHz MRSS L1/L2 ENGINE          │   │ CARRIER 2: 100 MHz 6G SA L1/L2 ENGINE │  │
│  │ (264 PRBs @ 120 kHz SCS — 5G SA + 6G SA)      │   │ (273 PRBs @ 30 kHz SCS* — Pure 6G)    │  │
│  │ • Semi-Static Mask: Reserves 5G SSB (20 PRBs),│   │ • Zero 5G Legacy Overhead: 100% of    │  │
│  │   orthogonal 5G/6G PRACH & edge PUCCH         │   │   273 PRBs available for 6G SA        │  │
│  │ • Dynamic PDCCH: Shared 5G/6G CORESETs        │   │ • Lean 6G Signaling: On-demand SIBs,  │  │
│  │ • L1 Rate-Matching: Hardware wraps 6G PDSCH   │   │   sparse DMRS/CSI-RS                  │  │
│  │   around 5G SSB & schedules remaining PRBs    │   │ • AI Beamforming: Tensor Core MU-MIMO │  │
│  └─────────────────────────┬─────────────────────┘   └─────────────────┬─────────────────────┘  │
└────────────────────────────┼───────────────────────────────────────────┼────────────────────────┘
                             │ eCPRI Fronthaul                           │ eCPRI Fronthaul
                             ▼ (Frequency-Domain PRB IQ Grids)           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│             ERICSSON ANTENNA-INTEGRATED RADIOS (AIR) + MULTI-BAND POWER POOLING                 │
│  • Maps 264-PRB (400 MHz) & 273-PRB (100 MHz) FFT grids to RF chains                            │
│  • Dynamically shifts Power Amplifier (PA) energy across PRBs based on active 5G/6G traffic     │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
Note: 100 MHz uses 273 PRBs at 30 kHz SCS (FR1/FR3 cmWave) or 66 PRBs at 120 kHz SCS (FR2 mmWave).

```

5G Radio Unit (RU) and a 6G Radio Unit (RU) connect in a cascaded chain to a common C-RAN Baseband (DU/CU).

```
+---------------------------------------------------------------------------------------------------+
|																									|
|                                        CORE NETWORK (BACKHAUL)                                    |
|                                                                                                   |
|       +-----------------------------------+                   +-------------------------------+   |
|       |          5G Core (5GC)            |                   |         6G Core (6GC)         |   |
|       |   (AMF / SMF / UPF - 5G Slice)    |                   |   (6G Control & User Plane)   |   |
|       +-----------------+-----------------+                   +---------------+---------------+   |
+-------------------------|-----------------------------------------------------|-------------------+

                          | NG-C / NG-U                                         | 6G-NG (SA Backhaul)
                          | (F1/E1 Split)                                       |
+-------------------------v-----------------------------------------------------v-------------------+
|																									|
|                                   C-RAN BASEBAND POOL (COMMON gNB)                                |
|                                                                                                   |
|   +-------------------------------------------------------------------------------------------+   |
|   |                       Multi-RAT Central Unit (CU-CP & CU-UP)                              |   |
|   |             [Separates 5G PDCP/RRC & 6G PDCP/RRC • Routes to 5GC vs. 6GC]                 |   |
|   +---------------------------------------------+---------------------------------------------+   |
|                                                 | F1 Interface (Internal / Midhaul)               |
|   +---------------------------------------------v---------------------------------------------+   |
|   |                       Multi-RAT Distributed Unit (DU)                                     |   |
|   |                                                                                           |   |
|   |   +---------------------------------------+       +-----------------------------------+   |   |
|   |   |         5G NR Stack (RLC/MAC)         |       |       6G Stack (RLC/MAC)          |   |   |
|   |   +-------------------+-------------------+       +-----------------+-----------------+   |   |
|   |                       |                                             |                     |   |
|   |   +-------------------v---------------------------------------------v-----------------+   |   |
|   |   |                 Shared Multi-RAT Spectrum Sharing (MRSS) Scheduler                |   |   |
|   |   +-----------------------------------------+-----------------------------------------+   |   |
|   |                                             |                                             |   |
|   |   +-----------------------------------------v-----------------------------------------+   |   |
|   |   |            Common Low-PHY / eCPRI Multiplexer & PTP Grandmaster Clock             |   |   |
|   |   +-----------------------------------------+-----------------------------------------+   |   |
|   +---------------------------------------------|---------------------------------------------+   |
+---------------------------------------------------|-----------------------------------------------+
                                                    |
                                                    | Single Trunk Fiber (Fronthaul)
                                                    | [25G / 50G / 100G eCPRI + IEEE 1588v2 PTP]
                                                    | (Carries tagged 5G IQ + 6G IQ + Sync)
                                                    |
+---------------------------------------------------v-----------------------------------------------+
|																									|
|                                        TOWER / CELL SITE                                          |
|                                                                                                   |
|   +-------------------------------------------------------------------------------------------+   |
|   |                                5G Radio Unit (5G RU) — [HOP 1]                            |   |
|   |                                                                                           |   |
|   |   [Optical Port 1: IN]  <--- Receives combined 5G + 6G eCPRI & PTP Clock                  |   |
|   |          |                                                                                |   |
|   |          +---> Extracts 5G IQ Data  ---> [5G High-PHY / Beamforming] ---> (( 5G RF ))     |   |
|   |          |                                                                                |   |
|   |          +---> Internal L2 Switch / Pass-Through + PTP Boundary Clock                     |   |
|   |          |                                                                                |   |
|   |   [Optical Port 2: OUT] ---> Forwards 6G IQ Data & Regenerated PTP Sync                   |   |
|   +---------------------------------------------|---------------------------------------------+   |
|                                                 |                                                 |
|                                                 | Short Jumper Fiber (Cascaded Link)              |
|                                                 | [eCPRI Pass-Through + PTP Sync]                 |
|                                                 |                                                 |
|   +---------------------------------------------v---------------------------------------------+   |
|   |                                6G Radio Unit (6G RU) — [HOP 2]                            |   |
|   |                                                                                           |   |
|   |   [Optical Port 1: IN]  <--- Receives 6G eCPRI Data & PTP Clock from 5G RU                |   |
|   |          |                                                                                |   |
|   |          +---> Extracts 6G IQ Data  ---> [6G High-PHY / Sub-THz or FR3] ---> (( 6G RF ))  |   |
|   |          |                                                                                |   |
|   |   [Optical Port 2: OUT] (Terminated or available for Hop 3)                               |   |
|   +-------------------------------------------------------------------------------------------+   |
+---------------------------------------------------------------------------------------------------+

```
I guess in real world deployments, RF engineers typically place the 6G RU at Hop 1 (closest to the baseband) and 
the 5G RU at Hop 2. Because 6G requires tighter latency budgets and stricter phase synchronization than 5G, 
putting the 6G radio first avoids the switching delay and clock jitter (jitter accumulation avoidance) added by passing 
through the 5G radio's internal switch.

---

**How E/// Hardware will execute both workloads Per Slot (Thoughts)**

* **1. Semi-Static PRB Masking (L1 Hardware Registers)**: At frame initialization, Ericsson Silicon writes a hardware PRB mask for the 400 MHz MRSS carrier flagging which PRBs/symbols carry always-on 5G SSBs, 6G discovery signals, and orthogonal 5G/6G PRACH/PUCCH occasions. The 100 MHz 6G SA carrier receives a clean mask with only lean 6G control/sync resources.(A PRB Bitmap needed for allocated/unallocated PRBs between both carriers)
  
* **2. Joint Slot-by-Slot PRB Allocation (L2 Many-Core Scheduler)**: Every slot, the unified MAC scheduler evaluates 5G SA and 6G SA QoS flows together. On the 400 MHz carrier, it assigns contiguous PRB bundles to 5G UEs and 6G UEs without guard bands between them.
  
* **3. Cross-Carrier Aggregation & UL/DL Decoupling**: For a 6G SA device aggregated across both carriers (up to 537 total PRBs), Ericsson's scheduler applies 6G UL/DL carrier decoupling:</br>
	• Downlink: Steers bursty, high-throughput 6G traffic onto the 400 MHz MRSS carrier whenever 5G traffic dips, achieving sub-3% coexistence overhead.</br>
	• Uplink: Anchors power-limited or latency-sensitive 6G uplink onto the 100 MHz dedicated 6G SA carrier (especially if deployed in mid-band FR1/FR3), avoiding 5G uplink contention and extending cell-edge range.

* **4. Hardware Rate-Matching & Multi-Band Power Pooling**: Dedicated L1 baseband accelerators execute bit-level rate-matching around masked 5G PRBs in real time, while Ericsson’s radio units dynamically pool transmit power across active PRBs on both carriers rather than backing off power statically

## 2. Identified Uplink & Scheduler Bottlenecks

### A. Uplink Control Information (UCI) Aggregation Pressures
In early 6G rollouts, shared FR1 bands (sub-7 GHz) provide the baseline macro coverage anchor, while higher frequency bands (e.g., cmWave / FR3) are aggregated to deliver high-capacity bursts. This architecture forces the shared MRSS carrier to handle a disproportionate amount of **Uplink Control Information (UCI)**, including Hybrid ARQ (HARQ) acknowledgments, Channel State Information (CSI) reporting, and Scheduling Requests (SR) for both the local shared carrier and the aggregated high-band downlinks. This creates a severe control-channel bottleneck in the uplink framework.

### B. Cross-RAT Coexistence Signaling & Rate-Matching Overheads
Even though 5G is lean, the scheduler must dynamically protect "always-on" 5G transmissions (such as SS/PBCH blocks, tracking reference signals, and CORESET #0) from 6G interference. The scheduler must implement tight, millisecond-timescale cross-RAT coordination or **rate-matching around 5G signals**. 
* **Impact:** This structural masking creates physical resource block (PRB) fragmentation in the uplink, complicating scheduling algorithms and adding processing latency.

### C. Physical Random Access Channel (PRACH) & Cross-RAT Coexistence
MRSS raises complex questions regarding how **PRACH resources are shared across RATs** on the same spectrum. Coexistence of 5G and 6G random access procedures can lead to preamble collision or restrictive semi-static partitions, which directly limits initial access capacity and increases latency for cell-edge users.

### D. Multi-Domain Scheduler Latency & Disjoint Power Headroom
Unlike standalone scheduling, an MRSS scheduler must arbitrate resources between independent 5G NR and 6G baseband protocol stacks. 
* **Inter-Scheduler Communication:** Realizing ultra-fast scheduling loop interaction over existing interfaces (like Xn) falls short of expectations due to millisecond-level transport delay.
* **Uplink Power Headroom Limitations:** Schedulers struggle to dynamically track a user equipment's (UE) real-time total transmit power headroom if power-sharing is split across decoupled 5G and 6G MAC schedulers.

---

## 3. Recommended 6G Architectural Design Framework

To prevent MRSS from degrading native 6G performance and causing uplink capacity exhaustion, 
the 6G radio access network (RAN) and scheduler architectures should implement the following guidelines:

| Design Dimension | Recommended 6G Implementation Strategy |
| :--- | :--- |
| **Waveform & Numerology Alignment** | Mandate strict alignment of Subcarrier Spacing (SCS) and physical frame configurations (symbol/slot/frame grids) between 5G NR and 6G in shared FR1 bands to completely eliminate mixed-numerology guard bands. |
| **CA-Centric Control Architecture** | Prefer **Carrier Aggregation (CA)** mechanisms over Dual Connectivity (DC). A single, integrated scheduling domain allows a unified scheduler to dynamically manage cross-carrier power headroom and seamlessly distribute UCI across available carriers instead of bottle-necking the shared anchor. |
| **Invisible 6G Preamble & Access Signaling** | Embed "invisible" 6G synchronization signals (SSB) and prioritize AI/ML-driven beam prediction to shorten initial access loops, bypassing traditional heavy-overhead RACH procedures whenever prior connected-state context is available. |
| **AI-Native Dynamic Sharing Ratios** | Utilize Network Data Analytics Function (**NWDAF**) and Real-Time Intelligent Controllers (**RIC**) to predict traffic bursts. The scheduler should shift spectrum partitioning adaptively rather than relying on reactive millisecond slot-by-slot arbitration. |
| **Lightweight Rate-Matching** | Keep the 6G frame clear of hardcoded legacy placeholders by utilizing soft, flexible rate-matching configurations around 5G always-on signals, protecting 6G native standalone performance. |

---

## 4. Ericsson-published MRSS and uplink-scheduling context

**Assessment (public Ericsson material available as of November 18, 2026):** Ericsson has described how it expects 5G–6G MRSS to operate, but the sources below do not declare the specific MRSS uplink bottlenecks listed in Section 2 as established Ericsson findings. They also do not specify a 5G/6G bandwidth-split algorithm driven by the two RATs' channel conditions. The bottlenecks in Sections 2 and the recommendations in Section 3 should therefore be read as this document's analysis, not as Ericsson-attributed conclusions.

Ericsson's MRSS discussion states that:

- 5G and 6G should use the same OFDM waveform and numerology with aligned subcarriers to reduce interference and enable sharing.
- Since traffic is bursty, spectrum sharing should be dynamic: when one RAT has no traffic, the carrier should be available to the other. Ericsson describes tight, millisecond-timescale scheduling coordination, supplemented by semi-statically reserved resources and resource-block alignment.
- This is a traffic/resource-availability sharing model in the cited discussion; it does not prescribe bandwidth division based on radio-channel quality or different channel conditions between 5G and 6G.

Ericsson separately discusses uplink scheduling latency in 5G: a UE's scheduling request (SR) is followed by a small grant carrying a buffer status report (BSR), after which the base station can schedule the remaining buffered data. Ericsson proposes a 6G contention-based BSR mechanism and reports a 51% tail-latency reduction versus the 5G SR baseline in its evaluation. This is a general uplink-scheduling discussion, not a declaration that MRSS creates UCI aggregation, PRACH collision, scheduler-split, or power-headroom bottlenecks.

**Sources**

- Ericsson, [Multi-RAT spectrum sharing and 5G–6G migration](https://www.ericsson.com/en/blog/2026/4/5g-6g-spectrum-sharing-mrss), April 2026.
- Ericsson, [Faster 6G applications with better UL scheduling](https://www.ericsson.com/en/reports-and-papers/ericsson-technology-review/articles/faster-6g-applications-with-better-ul-scheduling), October 2026.
