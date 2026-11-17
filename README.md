# MRSS(Multi RAT Spectrum Sharing) bottleneck with UL Capacity
MRSS is used for easy migration from 5G to 6G SA before full 6G Standardiztion.
Consider MRSS is pre-6G and pre-Commercial delivery for field trial and to operator. 
So, before deploying MRSS in site the operator benefits, bottlenecks, risk and performance are need to be investigated and analysed for both 5G and 6G users. 

![MRSS](MRSS.jpeg)

## Ericsson perspective: uplink capacity on an MRSS carrier

MRSS allows multiple radio access technologies to share a carrier; it improves spectrum use but does not add spectrum. The carrier's finite uplink time/frequency resources must still serve traffic from both technologies. Ericsson describes 5G–6G MRSS as having low sharing overhead, while noting that tight, millisecond-timescale scheduling coordination is needed so one technology can use resources when the other is idle.

Ericsson does not identify a single fixed MRSS-specific uplink capacity limit. In an operator network, the practical limit depends on the uplink resources available under the scheduler and concurrent load, as well as uplink signal quality and coverage (especially at the cell edge). Assess uplink load/resource use, signal quality, and cell-edge throughput separately from downlink capacity; downlink headroom does not guarantee uplink headroom.

**Ericsson references:** [Multi-RAT spectrum sharing and 5G–6G migration](https://www.ericsson.com/en/blog/2026/4/5g-6g-spectrum-sharing-mrss) · [Achieving networks with high-performing uplinks](https://www.ericsson.com/en/reports-and-papers/ericsson-technology-review/articles/achieving-networks-with-high-performing-uplinks)
