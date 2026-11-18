# Technical Analysis: Uplink Capacity Bottlenecks in 5G/6G MRSS Schedulers and 6G Architectural Design

## 1. Executive Summary
Multi-RAT Spectrum Sharing (**MRSS**) serves as the primary mechanism for migrating valuable Frequency Range 1 (**FR1**) spectrum from 5G to 6G dynamically. While MRSS benefits from 5G's "ultra-lean" air interface design (greatly reducing the heavy, always-on common reference signal overheads that plagued 4G-5G DSS), it introduces critical capacity bottlenecks within the radio scheduler domain, particularly concerning the **uplink (UL) path and control-channel resources**. 

This document evaluates these scheduler bottlenecks and presents architectural design frameworks for native 6G deployments to mitigate them.

---

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

To prevent MRSS from degrading native 6G performance and causing uplink capacity exhaustion, the 6G radio access network (RAN) and scheduler architectures should implement the following guidelines:

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
