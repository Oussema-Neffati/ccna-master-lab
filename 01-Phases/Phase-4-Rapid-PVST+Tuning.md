---
title: "Phase 4 — Rapid PVST+ Tuning"
project: CCNA Master Lab — Acme Corp Two-Office Enterprise
phase: 4
status: complete
tags:
  - ccna
  - lab
  - stp
  - rapid-pvst
  - spanning-tree
  - packet-tracer
created: 2026-09-20
updated: 2026-09-20
---

# Phase 4 — Rapid PVST+ Tuning

> [!info] Project Context
> This lab models **Acme Corp**, a two-office enterprise connected through a dual-homed core and a single edge router to a simulated ISP. Each phase builds on the previous one, ending with a fully integrated CCNA lab covering VLANs, STP, EtherChannel, OSPF, ACLs, security hardening, services, and automation.

> [!success] Phase Objective
> Take deterministic control of the Spanning Tree topology so that the STP root bridge matches the HSRP active gateway on every VLAN. Harden access ports with PortFast + BPDU Guard, and protect the root bridge position with Root Guard.

> [!important] Scope Rules
> - Configure **switches only**.
> - Do **not** touch R1, ISP, or the PCs.
> - EtherChannel is still deferred to Phase 5.

---

## 1. Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Rapid PVST+ (RSTP)** | Sub-second convergence vs. classic STP's 30–50 s. PVST+ gives per-VLAN topology control. |
| **DSW-A1 / DSW-B1 as root primary (priority 4096)** | Aligns STP root with HSRP active so inter-VLAN traffic doesn't hairpin. |
| **DSW-A2 / DSW-B2 as root secondary (priority 8192)** | Provides deterministic backup root during failover. |
| **PortFast on host ports** | Skips listening/learning states on access ports where a loop is impossible. |
| **BPDU Guard on host ports** | Err-disables the port if a switch is plugged in — stops rogue switch/loop attacks. |
| **Root Guard on ASW uplinks** | Prevents a rogue switch from becoming STP root. |

---

## 2. STP Mode (All 10 Switches)

```cisco
enable
configure terminal
spanning-tree mode rapid-pvst
exit
```

### Verify
```cisco
show spanning-tree summary
```
Expected: **"Spanning tree enabled protocol rstp"**.

---

## 3. Root Bridge Assignment

### Office A

**On DSW-A1 (primary):**
```cisco
spanning-tree vlan 10,20,30,99 root primary
spanning-tree vlan 10,20,30,99 priority 4096
```

**On DSW-A2 (secondary):**
```cisco
spanning-tree vlan 10,20,30,99 root secondary
spanning-tree vlan 10,20,30,99 priority 8192
```

### Office B

**On DSW-B1 (primary):**
```cisco
spanning-tree vlan 10,20,30,99 root primary
spanning-tree vlan 10,20,30,99 priority 4096
```

**On DSW-B2 (secondary):**
```cisco
spanning-tree vlan 10,20,30,99 root secondary
spanning-tree vlan 10,20,30,99 priority 8192
```

### VLAN 300 (Services)

**On DSW-A1 only (for now):**
```cisco
spanning-tree vlan 300 root primary
spanning-tree vlan 300 priority 4096
```

> [!note] DSW-A2 will become secondary root for VLAN 300 in Phase 5, after the DSW↔DSW EtherChannel is up and the VLAN 300 SVI is added to DSW-A2.

---

## 4. Edge Port Hardening (ASWs Only)

On **ASW-A1, ASW-A2, ASW-B1, ASW-B2**:

```cisco
interface range FastEthernet0/1 - 23
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit
```

### Verify
```cisco
show spanning-tree interface FastEthernet0/1 detail | include PortFast
```
Expected: **PortFast: enabled**, **BPDU Guard: enabled**.

> [!warning] BPDU Guard is destructive by design
> Connecting an unauthorized switch to a PortFast port err-disables it. Recover with `shutdown` / `no shutdown` after removing the rogue device.

---

## 5. Root Guard (ASW Uplinks)

On **ASW-A1, ASW-A2, ASW-B1, ASW-B2**:

```cisco
interface range GigabitEthernet0/1 - 2
 spanning-tree guard root
 exit
```

This causes an uplink to enter **root-inconsistent** state if it receives a superior BPDU, preventing a rogue switch from becoming root.

---

## 6. Expected STP Topology After Phase 4

| VLAN | Root | Secondary | Blocked Port |
|------|------|-----------|--------------|
| 10 | DSW-A1 | DSW-A2 | One uplink on each ASW |
| 20 | DSW-A1 | DSW-A2 | One uplink on each ASW |
| 30 | DSW-A1 | DSW-A2 | One uplink on each ASW |
| 99 | DSW-A1 | DSW-A2 | One uplink on each ASW |
| 300 | DSW-A1 | — | N/A |

Office B mirrors with DSW-B1 / DSW-B2.

---

## 7. Verification Steps

### 7.1 Root Confirmation
```cisco
show spanning-tree root
```
- VLANs 10,20,30,99 → root address = DSW-A1 (Office A) / DSW-B1 (Office B)
- Root path cost should be 0 on the root switch

### 7.2 Blocked Port Confirmation
On **ASW-A1**:
```cisco
show spanning-tree vlan 10
```
- One uplink: **Root/FWD**
- Other uplink: **Altn/BLK**

### 7.3 PortFast / BPDU Guard
```cisco
show spanning-tree interface FastEthernet0/1 detail
```

### 7.4 BPDU Guard Live Test
Connect a spare switch to a PortFast access port (e.g., ASW-A1 Fa0/5):
```cisco
show interfaces FastEthernet0/5 status
```
Expected: `err-disabled (BPDU Guard)`.

### 7.5 Root Guard
```cisco
show spanning-tree inconsistentports
```
Should return "No inconsistencies".

---

## 8. Phase 4 Checkpoint

- [x] All 10 switches running RSTP
- [x] DSW-A1 / DSW-B1 = root primary (priority 4096) for VLANs 10,20,30,99
- [x] DSW-A2 / DSW-B2 = root secondary (priority 8192)
- [x] DSW-A1 = root primary for VLAN 300
- [x] PortFast + BPDU Guard on Fa0/1–23 of all ASWs
- [x] Root Guard on ASW uplinks (Gi0/1–2)
- [x] BPDU Guard test err-disabled the test port
- [x] All switches saved to startup-config

---

## 9. Common Issues & Fixes

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| `show spanning-tree root` shows unexpected root | Priority not applied or another switch has lower priority | Check `show spanning-tree vlan 10 \| include priority` on every switch |
| Both ASW uplinks show FWD | Wrong VLAN examined, or loop isn't formed | `show spanning-tree vlan 10` |
| PortFast missing on host port | Port is trunk mode | `switchport mode access` first |
| BPDU Guard not triggering | PortFast missing on the port | Enable both together |
| Root Guard blocking legitimate uplink | Applied on wrong switch | Root Guard goes on **non-root** switches, never on the root itself |
| Port err-disabled after BPDU Guard test | Expected behavior | `shutdown` / `no shutdown` on the port |

---

## 10. Next Phase

➡️ **[Phase 5 — EtherChannel (LACP)](Phase-5-EtherChannel.md)**

In Phase 5 we will:
- Bundle CSW1↔CSW2 Fa0/1–2 into an **L3 PortChannel**
- Bundle DSW-A1↔DSW-A2 Fa0/2–3 into an **L2 PortChannel**
- Bundle DSW-B1↔DSW-B2 Fa0/2–3 into an **L2 PortChannel**
- Add the **VLAN 300 SVI on DSW-A2** as HSRP standby
- Watch STP re-converge with the loop collapsed into a single logical link

---

*Document maintained as part of the CCNA Master Lab project. For questions, corrections, or contributions, see the repository README.*