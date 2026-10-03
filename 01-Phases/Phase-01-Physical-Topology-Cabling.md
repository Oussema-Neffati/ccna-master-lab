# Phase 1 — Physical Topology & Cabling

> [!NOTE] Prerequisites & Navigation
> **Previous:** (start of project) · **Next:** [Phase 2 — VLANs & 802.1Q Trunks](Phase-02-VLANs-and-Trunks.md)
> Master addressing plan: [MASTER-ADDRESSING.md](../MASTER-ADDRESSING.md)

> [!TIP] Phase Objective
> Place all devices, cable the topology, and verify Layer 1 / Layer 2 link status — **without any configuration**.

---

## 1. Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Dual-homed edge (R1 to CSW1 and CSW2)** | Provides edge redundancy — if one core switch fails, R1 still reaches the other. |
| **3560 switches for CSW & DSW roles** | Needed for SVIs, HSRP, and routed ports (Packet Tracer limitation). |
| **2960 switches for ASW roles** | Sufficient for Layer 2 access only. |
| **Two physical links per EtherChannel** | Allows LACP to bundle without wasting ports; matches enterprise best practice. |
| **Intentional Layer 2 loops** | Created on purpose between DSW/ASW pairs so Rapid PVST+ has something to block and demonstrate. |

> [!IMPORTANT] What this lab does NOT claim
> Office A is homed only to CSW1, and Office B only to CSW2. Losing CSW1 does not
> keep Office A reachable — it isolates Office A. What the dual-homed edge **does**
> provide is uninterrupted Internet reachability for whichever office is still
> connected to a live core switch. Full L2/L3 redundancy across both offices would
> require additional cross-links not present in this design.

---

## 2. Device Inventory

### Network Devices (12)

| # | Hostname | Model | Role |
|---|----------|-------|------|
| 1 | ISP | 2911 | Simulated Internet edge |
| 2 | R1 | 2911 | Edge router, NAT/PAT, OSPF ASBR, **DHCP server**, **NTP master**, SSH |
| 3 | CSW1 | 3560-24PS | Core L3 switch (primary for Office A path) |
| 4 | CSW2 | 3560-24PS | Core L3 switch (primary for Office B path) |
| 5 | DSW-A1 | 3560-24PS | Office A distribution (STP root, HSRP active) |
| 6 | DSW-A2 | 3560-24PS | Office A distribution (STP secondary, HSRP standby) |
| 7 | DSW-B1 | 3560-24PS | Office B distribution (STP root, HSRP active) |
| 8 | DSW-B2 | 3560-24PS | Office B distribution (STP secondary, HSRP standby) |
| 9 | ASW-A1 | 2960-24TT | Office A access switch |
| 10 | ASW-A2 | 2960-24TT | Office A access switch |
| 11 | ASW-B1 | 2960-24TT | Office B access switch |
| 12 | ASW-B2 | 2960-24TT | Office B access switch |

### Endpoints (5)

| # | Hostname | Model | Role |
|---|----------|-------|------|
| 13 | SRV1 | Server-PT | DNS, Syslog, HTTP (SNMP/NTP services not available in PT — see Phase 9) |
| 14 | PC-A1 | PC-PT | Office A user |
| 15 | PC-A2 | PC-PT | Office A user |
| 16 | PC-B1 | PC-PT | Office B user |
| 17 | PC-B2 | PC-PT | Office B user |

**Total devices: 17**

> [!NOTE] SRV1's actual services
> In this lab, **R1** provides DHCP and NTP. SRV1 provides DNS, Syslog, and HTTP.
> Packet Tracer's Server-PT does not expose an SNMP agent, so SRV1 cannot be an
> SNMP trap receiver — that is a PT limitation, noted in Phase 9.

---

## 3. Physical Topology

![Physical-topology](/02-Diagrams/Physical-topology-v2.png)


### 3.2 Physical Cabling Overview

```
                         [ISP]
                           | 203.0.113.0/30
                         [R1]
                    Gi0/0 |  | Gi0/2
                10.0.0.0/30 |  |  10.0.0.8/30
                          |  |
                     [CSW1]══[CSW2]     ← 2x Fa = L3 EtherChannel (10.0.0.4/30)
                       |   |  |   |
                  DSW-A1 DSW-A2 DSW-B1 DSW-B2
                  (Office A pair)  (Office B pair)
                       |   |    |   |
                  ASW-A1 ASW-A2  ASW-B1 ASW-B2
                       |          |
                     PC-A1      PC-B1
                     PC-A2      PC-B2
                       |
                     SRV1 (VLAN 300, on DSW-A1)
```

---

## 4. Cabling Matrix

All cables are **Copper Straight-Through**. Packet Tracer handles MDI/MDI-X automatically.

### 4.1 Edge & Core Links

| # | From | Port | To | Port | Subnet | Purpose |
|---|------|------|----|------|--------|---------|
| 1 | ISP | Gi0/0 | R1 | Gi0/1 | 203.0.113.0/30 | Internet edge |
| 2 | R1 | Gi0/0 | CSW1 | Gi0/1 | 10.0.0.0/30 | OSPF transit |
| 3 | R1 | Gi0/2 | CSW2 | Gi0/1 | 10.0.0.8/30 | OSPF transit (redundant edge) |
| 4 | CSW1 | Fa0/1 | CSW2 | Fa0/1 | 10.0.0.4/30 (Po1) | L3 EtherChannel member 1 |
| 5 | CSW1 | Fa0/2 | CSW2 | Fa0/2 | 10.0.0.4/30 (Po1) | L3 EtherChannel member 2 |
| 6 | CSW1 | Fa0/3 | DSW-A1 | Gi0/1 | 10.0.1.0/30 | Core ↔ Dist A1 |
| 7 | CSW1 | Fa0/4 | DSW-A2 | Gi0/1 | 10.0.2.0/30 | Core ↔ Dist A2 |
| 8 | CSW2 | Fa0/3 | DSW-B1 | Gi0/1 | 10.0.3.0/30 | Core ↔ Dist B1 |
| 9 | CSW2 | Fa0/4 | DSW-B2 | Gi0/1 | 10.0.4.0/30 | Core ↔ Dist B2 |

> [!NOTE] Subnet labels corrected
> Earlier versions of this document mislabeled the CSW1↔CSW2 link as `10.0.0.0/30`
> and R1↔CSW2 as `10.0.0.0/30`. The correct subnets are **10.0.0.4/30** and
> **10.0.0.8/30** respectively.

### 4.2 Distribution ↔ Access (creates STP loops)

**Office A**

| # | From | Port | To | Port |
|---|------|------|----|------|
| 10 | DSW-A1 | Gi0/2 | ASW-A1 | Gi0/1 |
| 11 | DSW-A2 | Gi0/2 | ASW-A1 | Gi0/2 |
| 12 | DSW-A2 | Fa0/1 | ASW-A2 | Gi0/1 |
| 13 | DSW-A1 | Fa0/1 | ASW-A2 | Gi0/2 |

**Office B**

| # | From | Port | To | Port |
|---|------|------|----|------|
| 14 | DSW-B1 | Gi0/2 | ASW-B1 | Gi0/1 |
| 15 | DSW-B2 | Gi0/2 | ASW-B1 | Gi0/2 |
| 16 | DSW-B2 | Fa0/1 | ASW-B2 | Gi0/1 |
| 17 | DSW-B1 | Fa0/1 | ASW-B2 | Gi0/2 |

### 4.3 Inter-Distribution EtherChannels (bundled in Phase 5)

| # | From | Port | To | Port |
|---|------|------|----|------|
| 18 | DSW-A1 | Fa0/2 | DSW-A2 | Fa0/2 |
| 19 | DSW-A1 | Fa0/3 | DSW-A2 | Fa0/3 |
| 20 | DSW-B1 | Fa0/2 | DSW-B2 | Fa0/2 |
| 21 | DSW-B1 | Fa0/3 | DSW-B2 | Fa0/3 |

### 4.4 Server & Host Connections

| # | From | Port | To | Port |
|---|------|------|----|------|
| 22 | SRV1 | Fa0 | DSW-A1 | Fa0/4 |
| 23 | PC-A1 | Fa0 | ASW-A1 | Fa0/1 |
| 24 | PC-A2 | Fa0 | ASW-A2 | Fa0/1 |
| 25 | PC-B1 | Fa0 | ASW-B1 | Fa0/1 |
| 26 | PC-B2 | Fa0 | ASW-B2 | Fa0/1 |

**Total cables: 26**

---

## 5. Master Addressing Reference

The full addressing plan lives in one place:

➡️ **[MASTER-ADDRESSING.md](../MASTER-ADDRESSING.md)**

That file contains:
- VLSM base blocks
- All 9 user VLAN subnets
- All 8 transit /30 links
- All 8 loopback addresses
- HSRP v1 / v2 groups and versions
- NAT rules
- Management access policy

> [!NOTE] Phases 1–5 use a partial view of the plan
> Only the transit links needed at each phase are shown in that phase's doc.
> Phases 6+ add OSPF transit subnets (`10.0.1.0` – `10.0.4.0/30`) and loopbacks
> (`10.255.255.x/32`). The master file always reflects the current state.

---

## 6. Verification Steps

After cabling, wait ~30 seconds for STP to converge, then verify:

### 6.1 Link Lights

- Every cable should show a **green triangle** on both ends.
- **Orange** circles on some links are **expected** — these are STP Alternate/Blocked ports on the intentional Layer 2 loops.
- **Red** indicates a problem (bad cable, wrong port, or shutdown interface).

> [!CAUTION] Do NOT try to "fix" orange ports
> Orange ports prove the Layer 2 loops are being handled correctly by Spanning Tree.
> They will remain orange until Phase 4 tunes STP priorities and Phase 5 collapses
> the DSW↔DSW segments into EtherChannels.

### 6.2 STP Sanity Check

On any access switch (e.g., ASW-A1):

```cisco
enable
show spanning-tree
```

**Expected:** at least one port in **Blocking/Alternate** state.

### 6.3 Ping Sanity Check

- `PC-A1 → PC-A2` should **fail**. No VLANs, no IPs — this is correct.

---

## 7. Lessons Learned & Design Notes

> [!NOTE] Why 3560 for CSW/DSW?
> The 3560 in Packet Tracer supports SVIs, routed ports, HSRP, and full OSPF. The
> 2960 does not — it is Layer 2 only.

> [!NOTE] Why dual-homed R1?
> A single uplink from R1 to CSW1 would make the whole network dependent on CSW1.
> Adding a second uplink (R1 Gi0/2 → CSW2 Gi0/1, subnet `10.0.0.8/30`) provides
> edge redundancy and enables OSPF failover.

> [!TIP] EtherChannel and trunk-to-routed conversion deferred
> The DSW↔DSW and CSW↔CSW parallel links are **plain links** until Phase 5.
> The CSW↔DSW links are trunks in Phase 2, then converted to **routed ports**
> in Phase 6.

---

## 8. Phase 1 Checkpoint

- [ ] 17 devices placed and renamed
- [ ] 26 cables connected with green/orange link lights
- [ ] `show spanning-tree` on ASW-A1 shows a blocked port
- [ ] R1 dual-homed to CSW1 and CSW2
- [ ] No configuration entered on any device yet

---

## 9. Next Phase

➡️ **[Phase 2 — VLANs & 802.1Q Trunks](Phase-02-VLANs-and-Trunks.md)**

---

*Phase 1 of the CCNA Master Lab project. See [README](../README.md) for the full phase list.*