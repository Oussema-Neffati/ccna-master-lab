---
title: "Phase 3 — Inter-VLAN Routing (SVIs + HSRP)"
project: CCNA Master Lab — Acme Corp Two-Office Enterprise
phase: 3
status: complete
tags:
  - ccna
  - lab
  - inter-vlan-routing
  - svi
  - hsrp
  - layer-3-switching
  - packet-tracer
created: 2026-09-20
updated: 2026-09-20
---

# Phase 3 — Inter-VLAN Routing (SVIs + HSRP)

> [!info] Project Context
> This lab models **Acme Corp**, a two-office enterprise connected through a dual-homed core and a single edge router to a simulated ISP. Each phase builds on the previous one, ending with a fully integrated CCNA lab covering VLANs, STP, EtherChannel, OSPF, ACLs, security hardening, services, and automation.

> [!success] Phase Objective
> Turn the distribution switches into Layer-3 gateways using SVIs, provide redundant first-hop gateways with HSRP, and verify inter-VLAN routing works within each office.

> [!important] Scope Rules
> - Configure **DSWs and endpoint hosts only**.
> - Do **not** touch R1, CSW1, CSW2, or the ASWs.
> - EtherChannel still deferred to Phase 5.

---

## 1. Design Decisions

| Decision | Rationale |
|----------|-----------|
| **SVIs instead of ROAS** | Router-on-a-Stick bottlenecks inter-VLAN traffic through a single interface. SVIs on the 3560s route at wire speed across the backplane. |
| **HSRP on every VLAN** | Redundant default gateway — hosts don't care which physical switch is active. |
| **HSRP VIP = .1 for every VLAN** | Consistent, easy-to-remember gateway. Hosts always point to `10.X.X.1`. |
| **DSW-A1 / DSW-B1 priority 110 + preempt** | Deterministic Active role. Higher priority wins; preempt reclaims Active after reboot. |
| **DSW-A2 / DSW-B2 default priority 100 + preempt** | Standby by default; can take over during failure. |
| **VLAN 300 SVI on DSW-A1 only (for now)** | SRV1 physically connects to DSW-A1. A redundant VLAN 300 SVI will be added to DSW-A2 in Phase 5. |

---

## 2. Enable IP Routing (All 4 DSWs)

On **DSW-A1, DSW-A2, DSW-B1, DSW-B2**:

```cisco
enable
configure terminal
ip routing
exit
```

> [!warning] #1 CCNA mistake
> The 3560 is Layer-3 capable, but **`ip routing` is OFF by default**. Without it, the switch will not route between VLANs even with SVIs configured.

### Verify
```cisco
show ip route
```
Should show a routing table (may be nearly empty), not the "IP routing not enabled" error.

---

## 3. Office A — SVIs on DSW-A1 (Primary)

```cisco
interface Vlan10
 description Office A - PCs
 ip address 10.10.10.2 255.255.255.0
 standby 10 ip 10.10.10.1
 standby 10 priority 110
 standby 10 preempt
 exit

interface Vlan20
 description Office A - IP Phones
 ip address 10.10.20.2 255.255.255.0
 standby 20 ip 10.10.20.1
 standby 20 priority 110
 standby 20 preempt
 exit

interface Vlan30
 description Office A - WiFi
 ip address 10.10.30.2 255.255.255.0
 standby 30 ip 10.10.30.1
 standby 30 priority 110
 standby 30 preempt
 exit

interface Vlan99
 description Office A - Management
 ip address 10.10.99.2 255.255.255.0
 standby 99 ip 10.10.99.1
 standby 99 priority 110
 standby 99 preempt
 exit
```

---

## 4. Office A — SVIs on DSW-A2 (Secondary)

```cisco
interface Vlan10
 ip address 10.10.10.3 255.255.255.0
 standby 10 ip 10.10.10.1
 standby 10 preempt
 exit

interface Vlan20
 ip address 10.10.20.3 255.255.255.0
 standby 20 ip 10.10.20.1
 standby 20 preempt
 exit

interface Vlan30
 ip address 10.10.30.3 255.255.255.0
 standby 30 ip 10.10.30.1
 standby 30 preempt
 exit

interface Vlan99
 ip address 10.10.99.3 255.255.255.0
 standby 99 ip 10.10.99.1
 standby 99 preempt
 exit
```

> [!tip] Why preempt on both switches?
> Without `preempt`, a switch that comes up after a reboot never reclaims its primary role. With preempt, DSW-A1 reclaims Active whenever it comes back online. Priority (110 vs. 100) determines the winner.

---

## 5. Office B — SVIs on DSW-B1 (Primary)

```cisco
interface Vlan10
 description Office B - PCs
 ip address 10.20.10.2 255.255.255.0
 standby 10 ip 10.20.10.1
 standby 10 priority 110
 standby 10 preempt
 exit

interface Vlan20
 description Office B - IP Phones
 ip address 10.20.20.2 255.255.255.0
 standby 20 ip 10.20.20.1
 standby 20 priority 110
 standby 20 preempt
 exit

interface Vlan30
 description Office B - Servers
 ip address 10.20.30.2 255.255.255.0
 standby 30 ip 10.20.30.1
 standby 30 priority 110
 standby 30 preempt
 exit

interface Vlan99
 description Office B - Management
 ip address 10.20.99.2 255.255.255.0
 standby 99 ip 10.20.99.1
 standby 99 priority 110
 standby 99 preempt
 exit
```

---

## 6. Office B — SVIs on DSW-B2 (Secondary)

```cisco
interface Vlan10
 description Office B - PCs
 ip address 10.20.10.3 255.255.255.0
 standby 10 ip 10.20.10.1
 standby 10 preempt
 exit

interface Vlan20
 description Office B - IP Phones
 ip address 10.20.20.3 255.255.255.0
 standby 20 ip 10.20.20.1
 standby 20 preempt
 exit

interface Vlan30
 description Office B - Servers
 ip address 10.20.30.3 255.255.255.0
 standby 30 ip 10.20.30.1
 standby 30 preempt
 exit

interface Vlan99
 description Office B - Management
 ip address 10.20.99.3 255.255.255.0
 standby 99 ip 10.20.99.1
 standby 99 preempt
 exit
```

---

## 7. Services VLAN 300 (DSW-A1 Only)

```cisco
! On DSW-A1
interface Vlan300
 description Services
 ip address 10.30.0.1 255.255.255.0
 no shutdown
 exit

interface FastEthernet0/4
 description SRV1
 switchport mode access
 switchport access vlan 300
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit
```

> [!note] No HSRP on VLAN 300 yet
> DSW-A2's SVI for VLAN 300 will be added in Phase 5 once the DSW↔DSW EtherChannel is up.

---

## 8. HSRP Address Summary

| VLAN | DSW-A1 / DSW-B1 | DSW-A2 / DSW-B2 | HSRP VIP |
|------|-----------------|------------------|----------|
| 10 | 10.X.10.2 (pri 110) | 10.X.10.3 (pri 100) | 10.X.10.1 |
| 20 | 10.X.20.2 (pri 110) | 10.X.20.3 (pri 100) | 10.X.20.1 |
| 30 | 10.X.30.2 (pri 110) | 10.X.30.3 (pri 100) | 10.X.30.1 |
| 99 | 10.X.99.2 (pri 110) | 10.X.99.3 (pri 100) | 10.X.99.1 |

> `X` = 10 for Office A, 20 for Office B.

---

## 9. Endpoint Static IPs (Temporary)

DHCP comes in Phase 9. For now, configure statically.

### PC Settings (Desktop → IP Configuration)

| Host | IP | Mask | Gateway |
|------|----|----|---------|
| PC-A1 | 10.10.10.10 | 255.255.255.0 | 10.10.10.1 |
| PC-A2 | 10.10.10.11 | 255.255.255.0 | 10.10.10.1 |
| PC-B1 | 10.20.10.10 | 255.255.255.0 | 10.20.10.1 |
| PC-B2 | 10.20.10.11 | 255.255.255.0 | 10.20.10.1 |
| SRV1  | 10.30.0.10 | 255.255.255.0 | 10.30.0.1 |

> [!important] Gateway = HSRP VIP, not physical SVI
> Every PC's gateway must be the **virtual IP (`.1`)**, not `.2` or `.3`. This is the entire point of HSRP.

---

## 10. Verification Steps

### 10.1 SVI Status
```cisco
show ip interface brief | include Vlan
```
Every SVI should be **up / up**. If down/down, the VLAN has no active member port.

### 10.2 HSRP Status
On **DSW-A1** and **DSW-B1**:
```cisco
show standby brief
```
Expected: **State = Active** for all four VLANs.

On **DSW-A2** and **DSW-B2**:
```cisco
show standby brief
```
Expected: **State = Standby**.

### 10.3 Inter-VLAN Routing
From **PC-A1** (10.10.10.10):
```cisco
ping 10.10.20.1
ping 10.10.99.1
```
Both should succeed.

### 10.4 Intra-VLAN Host-to-Host
From **PC-A1**:
```cisco
ping 10.10.10.11
```
Should succeed (PC-A2).

### 10.5 Inter-Office (Expected Failure)
From **PC-A1**:
```cisco
ping 10.20.10.10
```
Should **fail** — no route between offices yet. Fixed in Phase 6 (OSPF).

### 10.6 HSRP Failover Test
On **DSW-A1**:
```cisco
interface Vlan10
 shutdown
 exit
```
Wait ~5 seconds. On **DSW-A2**:
```cisco
show standby brief
```
DSW-A2 should now be **Active** for VLAN 10. Verify PC-A1 can still ping `10.10.10.1`.

Restore:
```cisco
! On DSW-A1
interface Vlan10
 no shutdown
 exit
```
DSW-A1 reclaims Active within ~10 seconds due to `preempt`.

---

## 11. Phase 3 Checkpoint

- [x] `ip routing` enabled on all 4 DSWs
- [x] SVIs for VLANs 10,20,30,99 on DSW-A1/A2/B1/B2
- [x] SVI for VLAN 300 on DSW-A1 + SRV1 access port (Fa0/4)
- [x] HSRP Active on DSW-A1 and DSW-B1
- [x] HSRP Standby on DSW-A2 and DSW-B2
- [x] PC-A1 ↔ PC-A2 intra-VLAN ✅
- [x] PC-B1 ↔ PC-B2 intra-VLAN ✅
- [x] Inter-VLAN ping (host ↔ other VIP) ✅
- [x] Inter-office ping (PC-A1 ↔ PC-B1) ❌ — expected until Phase 6
- [x] HSRP failover test successful
- [x] All 4 DSWs saved to startup-config

---

## 12. Troubleshooting Scenario — PC-A2 Could Not Reach Anything

During the lab, PC-A2 could not ping its own gateway or any other host, while PC-A1 worked perfectly.

### Symptoms
- PC-A1 → gateway ✅, PC-A1 → PC-A2 ❌
- PC-A2 → anything ❌ (not even its own gateway)

### Root Cause
**ASW-A2 Fa0/1 was configured as a trunk instead of an access port in VLAN 10.** The PC-facing port was accidentally included in a bulk `interface range` trunk command intended for uplinks.

### Fix
```cisco
! On ASW-A2
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10
 exit
```

### Verification
```cisco
show interfaces FastEthernet0/1 switchport
```
Expected: Administrative Mode: static access, Access Mode VLAN: 10.

### Lesson Learned
> [!warning] Bulk `interface range` commands must be scoped carefully
> `interface range GigabitEthernet0/1 - 2` (uplinks) and `interface range FastEthernet0/1 - 10` (hosts) look similar. Always double-check the port range before pressing Enter.

---

## 13. Common Issues & Fixes

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| SVI stays `down/down` | No active port in that VLAN | Verify VLAN exists and has a member port/trunk |
| HSRP stuck in `Init` | L2 connectivity broken between the two DSWs | Verify ASW trunks pass VLAN 10,20,30,99 |
| Both DSWs show `Active` | Mismatched `standby` group number, or preempt missing | Confirm group numbers match; add `standby X preempt` |
| Host can't ping its own gateway | Wrong PC gateway (should be HSRP VIP `.1`) | Fix in Desktop → IP Configuration |
| Inter-VLAN ping fails but HSRP is up | `ip routing` not enabled | Reapply `ip routing` |
| One PC works, the other doesn't | Access port misconfigured | `show interfaces <port> switchport` on the ASW |

---

## 14. Next Phase

➡️ **[Phase 4 — Rapid PVST+ Tuning](Phase-4-Rapid-PVST-Tuning.md)**

In Phase 4 we will:
- Set STP mode to Rapid PVST+ on all switches
- Align STP root bridge with HSRP active (DSW-A1 / DSW-B1 as root primary)
- Configure PortFast + BPDU Guard on host ports
- Apply Root Guard on ASW uplinks

---

*Document maintained as part of the CCNA Master Lab project. For questions, corrections, or contributions, see the repository README.*