# 6G HARQ Framework: CB-Level Repair & Feedback Reliability

This document summarizes the architectural evolution from traditional 5G Transport Block (TB) retransmission methods to the precise **6G Code Block (CB) level surgical repair** mechanism, detailing the message flows and system configurations required for ultra-low latency.

The 6G Hybrid Automatic Repeat Request (HARQ) framework is not yet a frozen specification, 
as **3GPP Release 20** is currently running its initial 6G study phase (targeted for commercial 
rollout around 2030). However, across 3GPP RAN1/RAN2 studies and telco industry research, 
the HARQ framework is evolving from 5G NR's rigid, single-bit ACK/NACK mechanism into an 
AI-native, cross-layer, and memory-optimized error-control architecture.

---

## 1. 6G CB-Level HARQ Message Flow

In the 6G framework, when a data transmission suffers localized corruption, 
only the specific corrupted Code Blocks (CBs) along with necessary redundancy 
data are retransmitted. This signaling sequence is performed over a hardened, 
tightly synchronized feedback channel.

```text
  UE (User Equipment)                                     gNodeB (6G Base Station)
       |                                                             |
       | <------------------ 1. Initial Transmission --------------- |
       |                      (Entire Transport Block)               |
       |                                                             |
[Decodes TB]                                                         |
[Identifies 4 failed CBs]                                            |
[Out of 16 total CBs]                                                |
       |                                                             |
       | ------------------ 2. Precise CB-Level NACK --------------> |
       |                      (Hardened Feedback Channel)            |
       |                      (Indicates specific failed CBs)        |
       |                                                             |
       |                                                       [Processes NACK]
       |                                                       [Extracts precise CB IDs]
       |                                                       [Generates Redundancy Data]
       |                                                             |
       | <---------------- 3. Surgical Retransmission -------------- |
       |                      (Only Failed CBs + Redundancy)         |
       |                                                             |
[Combines & Repairs]                                                 |
[Successful Decode]                                                  |
       |                                                             |
       | -------------------------- 4. ACK ------------------------> |
```

---

## 2. Comparative Analysis

The table below contrasts the legacy "Wideband Inefficiency" method with the new "Precise & Surgical" 6G HARQ mechanism:

| Metric / Feature | Old Method (TB Retransmission) | New 6G Method (CB-Level Repair) |
| :--- | :--- | :--- |
| **Retransmission Granularity** | Retransmits the **entire Transport Block (TB)** | Retransmits **only failed Code Blocks (CBs)** |
| **Feedback Mechanism** | Standard wideband NACK | Precise, **CB-level NACK** |
| **Spectral Efficiency** | **Low** (Wastes spectrum on already successful data) | **High** (Saves spectrum via precise target repair) |
| **Overhead & Latency** | High overhead / Higher latency | **Lower overhead / Faster recovery** |
| **Feedback Protection** | Vulnerable to timing & misread errors | **Hardened channels & tighter timing alignment** |
| **Application Suitability**| Not practical for high-rate systems | **Ideal for tight latency applications** |

---

## 3. Feedback Reliability & Robustness Measures

To avoid structural drops in throughput caused by mismatches (such as NACKs misread as ACKs, timing mismatches, 
and Bandwidth Part (BWP) desynchronization), the 6G implementation integrates specialized safety layers:

*   **Hardened Feedback Channels:** Robust signal modulation to guarantee zero-loss feedback paths.
*   **Strict Timing Alignment:** Sub-microsecond synchronization between the UE and gNodeB to prevent transmission window drift.
*   **Explicit Fallback Options:** Pre-allocated state-machine rules that kick in instantly if the main feedback channel undergoes sudden degradation.

> **Overall Result:** High spectral efficiency, consistent continuity of service, and predictable delivery times across high-rate systems.

---

## 4. AI/ML-Native HARQ Operations

As per understanding 6G embeds AI directly into the physical (PHY/L1) and medium access control (MAC) 
layers to break the latency bottleneck of conventional Stop-and-Wait (SAW) HARQ loops:

* **Predictive ACK/NACK (Early Retransmission)**: Machine learning models at the transmitter predict whether
  a transmission will fail based on channel state trends and interference patterns, triggering proactive
  retransmissions before waiting for the round-trip time (RTT) of the UE's feedback.

  > Benefits and effectiveness in operation??
  
* **ML Demodulation & Outer-Loop Convergence**: Neural receivers and ML-assisted demodulators (often paired
  with probabilistic constellation shaping) accelerate convergence to target Block Error Rates (BLER) and
  reduce the total number of retransmissions required at the cell edge. 

> Under study for 6G L2 streamlining, the physical/MAC layer directly exposes local HARQ failure events to
> the RLC layer. Upon detecting a final HARQ failure, the scheduler autonomously triggers an immediate RLC
> ARQ retransmission without waiting for upper-layer signaling.
> In 5G, today if a physical-layer HARQ process reaches its maximum retransmission limit and fails, the upper
> Radio Link Control (RLC) layer does not know immediately — it has to wait for a slow peer **RLC status report**,
> causing latency spikes.

