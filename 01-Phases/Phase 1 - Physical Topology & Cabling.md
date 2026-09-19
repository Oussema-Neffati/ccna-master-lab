---
title: "Phase 1 — Physical Topology & Cabling"
project: CCNA Master Lab — Acme Corp Two-Office Enterprise
phase: 1
status: complete
tags:
  - ccna
  - lab
  - topology
  - cabling
  - packet-tracer
created: 2026-09-18
updated: 2026-09-18
---

# Phase 1 — Physical Topology & Cabling

> [!info] Project Context
> This lab models **Acme Corp**, a two-office enterprise connected through a dual-homed core and a single edge router to a simulated ISP. Each phase builds on the previous one, ending with a fully integrated CCNA lab covering VLANs, STP, EtherChannel, OSPF, ACLs, security hardening, services, and automation.

> [!success] Phase Objective
> Place all devices, cable the topology, and verify Layer 1/Layer 2 link status — **without any configuration**.

---

## 1. Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Dual-homed core (R1 → CSW1 + CSW2)** | Provides edge redundancy; if CSW1 fails, Office A/B retain Internet access via CSW2. |
| **3560 switches for CSW & DSW roles** | Needed for SVIs, HSRP, and routed ports (Packet Tracer limitation). |
| **2960 switches for ASW roles** | Sufficient for Layer 2 access only. |
| **Two physical links per EtherChannel** | Allows LACP to bundle without wasting ports; aligns with enterprise best practice. |
| **Intentional Layer 2 loops** | Created on purpose between DSW/ASW pairs so Rapid PVST+ has something to block and demonstrate. |

---

## 2. Device Inventory

### Network Devices (12)

| #   | Hostname | Model     | Role                                             |
| --- | -------- | --------- | ------------------------------------------------ |
| 1   | ISP      | 2911      | Simulated Internet edge                          |
| 2   | R1       | 2911      | Edge router, NAT/PAT, OSPF ASBR, NTP master, SSH |
| 3   | CSW1     | 3560-24PS | Core L3 switch (primary)                         |
| 4   | CSW2     | 3560-24PS | Core L3 switch (secondary)                       |
| 5   | DSW-A1   | 3560-24PS | Office A distribution (STP root)                 |
| 6   | DSW-A2   | 3560-24PS | Office A distribution (STP secondary)            |
| 7   | DSW-B1   | 3560-24PS | Office B distribution (STP root)                 |
| 8   | DSW-B2   | 3560-24PS | Office B distribution (STP secondary)            |
| 9   | ASW-A1   | 2960-24TT | Office A access switch                           |
| 10  | ASW-A2   | 2960-24TT | Office A access switch                           |
| 11  | ASW-B1   | 2960-24TT | Office B access switch                           |
| 12  | ASW-B2   | 2960-24TT | Office B access switch                           |

### Endpoints (5)

| # | Hostname | Model | Role |
|---|----------|-------|------|
| 13 | SRV1 | Server-PT | DHCP, DNS, NTP, SNMP, Syslog |
| 14 | PC-A1 | PC-PT | Office A user |
| 15 | PC-A2 | PC-PT | Office A user |
| 16 | PC-B1 | PC-PT | Office B user |
| 17 | PC-B2 | PC-PT | Office B user |

**Total devices: 17**

---

## 3. Physical Topology


![Physical Topology](../04-Assets/Physical%20Topology.png)


---

## 4. Cabling Matrix

All cables are **Copper Straight-Through**. Packet Tracer handles MDI/MDI-X automatically.

### 4.1 Edge & Core Links

| # | From | Port | To | Port | Purpose |
|---|------|------|----|------|---------|
| 1 | ISP | Gi0/0 | R1 | Gi0/1 | Internet edge (203.0.113.0/30) |
| 2 | R1 | Gi0/0 | CSW1 | Gi0/1 | OSPF backbone (10.0.0.0/30) |
| 3 | R1 | Gi0/2 | CSW2 | Gi0/1 | OSPF backbone — **redundant uplink** (10.0.0.8/30) |
| 4 | CSW1 | Fa0/1 | CSW2 | Fa0/1 | L3 EtherChannel member 1 (10.0.0.4/30) |
| 5 | CSW1 | Fa0/2 | CSW2 | Fa0/2 | L3 EtherChannel member 2 |
| 6 | CSW1 | Fa0/3 | DSW-A1 | Gi0/1 | Trunk → Office A |
| 7 | CSW1 | Fa0/4 | DSW-A2 | Gi0/1 | Trunk → Office A |
| 8 | CSW2 | Fa0/3 | DSW-B1 | Gi0/1 | Trunk → Office B |
| 9 | CSW2 | Fa0/4 | DSW-B2 | Gi0/1 | Trunk → Office B |

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

### 4.3 Inter-Distribution EtherChannels (pending Phase 5)

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

## 5. IP Addressing Plan (Reference for Later Phases)

| Segment | Subnet | Purpose |
|---------|--------|---------|
| Office A – PCs | 10.10.10.0/24 | VLAN 10 |
| Office A – IP Phones | 10.10.20.0/24 | VLAN 20 |
| Office A – Wi-Fi | 10.10.30.0/24 | VLAN 30 |
| Office A – Mgmt | 10.10.99.0/24 | VLAN 99 |
| Office B – PCs | 10.20.10.0/24 | VLAN 10 |
| Office B – IP Phones | 10.20.20.0/24 | VLAN 20 |
| Office B – Servers | 10.20.30.0/24 | VLAN 30 |
| Office B – Mgmt | 10.20.99.0/24 | VLAN 99 |
| Services VLAN | 10.30.0.0/24 | VLAN 300 |
| R1 ↔ CSW1 | 10.0.0.0/30 | OSPF backbone |
| R1 ↔ CSW2 | 10.0.0.8/30 | OSPF backbone (redundant) |
| CSW1 ↔ CSW2 (L3 Po1) | 10.0.0.4/30 | OSPF backbone |
| R1 ↔ ISP | 203.0.113.0/30 | Internet edge |

---

## 6. Verification Steps

After cabling, wait **~30 seconds** for STP to converge, then verify:

### 6.1 Link Lights
- Every cable should show a **green triangle** on both ends.
- **Orange** circles on some links are **expected** — these are STP Alternate/Blocked ports on the intentional Layer‑2 loops.
- **Red** indicates a problem (bad cable, wrong port, or shutdown interface).

### 6.2 STP Sanity Check

On any access switch (e.g., ASW-A1):

```cisco
enable
show spanning-tree
```

**Expected:** at least one port in **Blocking/Alternate** state. This confirms STP has detected the redundant path.

> [!warning] Do NOT try to "fix" orange ports
> Orange ports are the entire point of Phase 1. They prove the Layer‑2 loops are being handled correctly by Spanning Tree.

### 6.3 Ping Sanity Check
- `PC-A1 → PC-A2` should **fail**. No VLANs, no IPs — this is correct.

---

## 7. Lessons Learned & Design Notes

> [!note] Why 3560 for CSW/DSW?
> The 3560 in Packet Tracer supports SVIs, routed ports, HSRP, and full OSPF. The 2960 does not — it is Layer 2 only.

> [!note] Why dual-homed R1?
> A single uplink from R1 to CSW1 would make the entire network dependent on CSW1. Adding a second uplink (R1 Gi0/2 → CSW2 Gi0/1, subnet `10.0.0.8/30`) provides edge redundancy and makes OSPF failover/load-balancing demonstrable.

> [!tip] LACP & EtherChannel Deferred
> The DSW↔DSW and CSW↔CSW parallel links are **plain links for now**. They will be bundled into EtherChannels in **Phase 5**, after STP behavior has been observed on individual links in Phase 4.

---

## 8. Phase 1 Checkpoint

- [x] 17 devices placed and renamed
- [x] 26 cables connected with green/orange link lights
- [x] `show spanning-tree` on ASW-A2 shows a blocked port
- [x] R1 dual-homed to CSW1 and CSW2
- [x] No configuration entered on any device yet

---

## 9. Next Phase

➡️ **[Phase 2 — VLANs & 802.1Q Trunks](Phase-2-VLANs-and-Trunks.md)**

In Phase 2 we will:
- Create the VLAN database on all 10 switches
- Configure access ports on the ASWs
- Configure 802.1Q trunks with native VLAN 1000
- Disable DTP with `switchport nonegotiate`

---

*Document maintained as part of the CCNA Master Lab project. For questions, corrections, or contributions, see the repository README.*