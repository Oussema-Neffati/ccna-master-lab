---
title: "Phase 11 — Troubleshooting Scenarios + Final Review"
project: CCNA Master Lab — Acme Corp Two-Office Enterprise
phase: 11
status: complete
tags:
  - ccna
  - lab
  - troubleshooting
  - diagnostics
  - final-review
  - packet-tracer
created: 2026-09-29
updated: 2026-09-29
---

# Phase 11 — Troubleshooting Scenarios + Final Review

> [!info] Project Context
> This lab models **Acme Corp**, a two-office enterprise connected through a dual-homed core and a single edge router to a simulated ISP. This phase wraps up the lab with 8 hands-on troubleshooting scenarios and a complete verification checklist.

> [!success] Phase Objective
> Prove diagnostic skill — inject misconfigurations, observe symptoms, diagnose, fix, and verify. This is the exam-critical skill: not "can you configure it" but "can you find what's broken."

> [!warning] Save a working snapshot first
> Before injecting anything:
> - `write memory` on every device
> - PT → File → Save As → `acme-corp-lab-WORKING.pkt`

> [!tip] How to use this phase
> Inject **one scenario at a time**. Diagnose, fix, verify, then move to the next. Do not inject all 8 at once.

---

## Scenario 1 — Native VLAN Mismatch on a Trunk

**CCNA Domain:** VLANs / Trunking

### Injection
```cisco
! On DSW-A1
interface FastEthernet0/1
 switchport trunk native vlan 1
 exit
```

### Symptom
- Tagged VLAN traffic keeps working
- Untagged traffic behavior is inconsistent
- CDP/STP may log "native VLAN mismatch"

### Diagnosis
```cisco
show interfaces trunk
show cdp neighbors detail
```
Compare native VLAN columns on both ends.

### Fix
```cisco
! On DSW-A1
interface FastEthernet0/1
 switchport trunk native vlan 1000
 exit
write memory
```

### Lesson
Native VLAN mismatch is silent for tagged traffic. It's a **security risk** (VLAN hopping). Always verify both ends.

---

## Scenario 2 — OSPF Passive Interface on a Transit Link

**CCNA Domain:** OSPF / HSRP interaction

### Injection
```cisco
! On DSW-A1
router ospf 1
 passive-interface GigabitEthernet0/1
 exit
```

### Symptom
- PC-A1 → PC-A2 (intra-office): ✅
- PC-A1 → PC-B1 (inter-office): ❌
- `show ip ospf neighbor` on DSW-A1 shows no neighbor for CSW1

### Diagnosis
```cisco
! On DSW-A1
show ip ospf neighbor
show ip ospf interface GigabitEthernet0/1   ! → Passive interface: Yes
show ip route ospf                            ! → missing Office B routes
```

```cisco
! On CSW1
show ip route | include 10.10.10
```

### Fix
```cisco
! On DSW-A1
router ospf 1
 no passive-interface GigabitEthernet0/1
 exit
write memory
```

Wait ~30 seconds. Verify:
```cisco
show ip ospf neighbor
show ip route 10.20.10.0
```

### The Critical Lesson — HSRP Does Not Track OSPF State

This scenario exposes one of the most important real-world behaviors:

**PC-A1's traffic goes to the HSRP VIP (10.10.10.1) → DSW-A1 (Active) → DSW-A1 has no OSPF route to Office B → traffic dropped.**

DSW-A1 remains **HSRP Active** because its Vlan10 SVI is still up. HSRP has no way of knowing that OSPF failed. DSW-A2 remains **Standby** and never receives the traffic.

```
PC-A1 → HSRP VIP → DSW-A1 (still Active) → NO ROUTE → drop
```

> [!important] HSRP only fails over when the tracked interface goes DOWN
> Routing failures do not trigger HSRP failover unless explicitly tracked. This is why production deployments use **HSRP object tracking**:
>
> ```cisco
> ! On DSW-A1
> track 1 interface GigabitEthernet0/1 line-protocol
>
> interface Vlan10
>  standby 10 track 1 decrement 20
>  exit
> ```
>
> If Gi0/1 goes down, priority drops from 110 to 90, and DSW-A2 (priority 100) takes over as Active.
>
> **Note:** Even this tracking only detects *link* failure. Detecting OSPF adjacency loss requires IP SLA tracking:
> ```cisco
> ip sla 1
>  icmp-echo 10.0.1.1 source-interface GigabitEthernet0/1
>  frequency 5
>  exit
> ip sla schedule 1 life forever start-time now
>
> track 1 ip sla 1 reachability
>
> interface Vlan10
>  standby 10 track 1 decrement 20
>  exit
> ```

### Lesson
Redundancy protocols are only as good as their tracking. Without tracking, HSRP stays Active on a device that has lost upstream connectivity — silently blackholing traffic.

---

## Scenario 3 — EtherChannel Member Shutdown

**CCNA Domain:** EtherChannel / LACP

### Injection
```cisco
! On DSW-A1
interface FastEthernet0/3
 shutdown
 exit
```

### Symptom
- PortChannel stays up (Fa0/2 still active)
- `show etherchannel summary` shows Fa0/3 as `(D)` — down
- No user impact (single member carries traffic) — **but redundancy is lost**

### Diagnosis
```cisco
show etherchannel summary
show interfaces FastEthernet0/3 status
```

### Fix
```cisco
interface FastEthernet0/3
 no shutdown
 exit
write memory
```

Verify Fa0/3 rejoins with `(P)`:
```cisco
show etherchannel summary
```

### Lesson
EtherChannel survives losing a member, but you must **monitor member state**. A degraded bundle is a silent risk.

---

## Scenario 4 — ACL Blocking Legitimate Traffic

**CCNA Domain:** ACLs

### Injection
```cisco
! On DSW-A1 AND DSW-A2
ip access-list extended BLOCK_A_TO_B_SERVERS
 no 10
 no 20
 exit
```
(Removes the HTTP/HTTPS permit lines, leaving only deny + permit any any.)

### Symptom
- PC-A1 → PC-B1 (VLAN 10 → VLAN 10): ✅
- PC-A1 → Office B server (VLAN 30) even via HTTP/HTTPS: ❌
- `show ip access-lists BLOCK_A_TO_B_SERVERS` shows only 2 entries

### Diagnosis
```cisco
show ip access-lists BLOCK_A_TO_B_SERVERS
```
Should show 4 entries; only 2 remain.

### Fix
```cisco
! On DSW-A1 AND DSW-A2
ip access-list extended BLOCK_A_TO_B_SERVERS
 10 permit tcp 10.10.10.0 0.0.0.255 10.20.30.0 0.0.0.255 eq 80
 20 permit tcp 10.10.10.0 0.0.0.255 10.20.30.0 0.0.0.255 eq 443
 exit
write memory
```

> [!warning] PT quirk — resequencing
> PT may place new numbered lines at the end instead of resequencing. If the order is wrong (deny before permit), delete and recreate the entire ACL.

### Lesson
ACLs are order-sensitive. A missing line changes the entire behavior. Always verify with `show ip access-lists` after any edit.

---

## Scenario 5 — Port Security Err-Disable

**CCNA Domain:** Port Security

### Injection
```cisco
! On ASW-A1
interface FastEthernet0/1
 switchport port-security maximum 1
 exit
```
Then connect two devices (e.g., via a hub).

### Symptom
- PC-A1 loses connectivity
- Fa0/1 err-disabled
- `show port-security interface Fa0/1` → `Secure-shutdown`

### Diagnosis
```cisco
show interfaces FastEthernet0/1 status
show port-security interface FastEthernet0/1
show port-security
```

### Fix
Remove the extra device, then:
```cisco
interface FastEthernet0/1
 shutdown
 no shutdown
 exit
```

If sticky MAC entries are the issue:
```cisco
clear port-security sticky interface FastEthernet0/1
```

### Lesson
Port Security protects against rogue devices and MAC flooding. It's **disruptive by design** — err-disable stops the attack but also stops legitimate traffic until cleared.

---

## Scenario 6 — DHCP Snooping Trust Boundary Broken

**CCNA Domain:** DHCP Snooping

### Injection
```cisco
! On ASW-A1
interface range GigabitEthernet0/1 - 2
 no ip dhcp snooping trust
 exit
```

### Symptom
- Existing clients keep their leases
- Renewing or new clients **cannot get an IP**
- `show ip dhcp snooping statistics` shows dropped DISCOVER/OFFER packets

### Diagnosis
```cisco
show ip dhcp snooping
show ip dhcp snooping binding
```
Look for: **no trusted interfaces** on uplinks.

On PC-A1:
```
ipconfig /release
ipconfig /renew
```
Expected: fails or 169.254.x.x.

### Fix
```cisco
interface range GigabitEthernet0/1 - 2
 ip dhcp snooping trust
 exit
write memory
```

### Lesson
DHCP Snooping trusts uplinks toward the DHCP server and untrusts host ports. Breaking the trust boundary silently blocks all DHCP.

---

## Scenario 7 — NAT ACL Wildcard Regression

**CCNA Domain:** NAT / ACLs

### Injection
```cisco
! On R1
no access-list 1
access-list 1 permit 10.0.0.0 0.0.255.255
exit
```

### Symptom
- PC-A1 cannot reach 8.8.8.8
- `show ip nat statistics` shows `Hits: 0`, `Misses` incrementing

### Diagnosis
```cisco
show ip nat statistics
show access-lists 1
```
The wildcard `0.0.255.255` matches only `10.0.x.x` — not `10.10.x.x`, `10.20.x.x`, `10.30.x.x`.

### Fix
```cisco
no access-list 1
access-list 1 remark === NAT: all internal 10/8 networks ===
access-list 1 permit 10.0.0.0 0.255.255.255
exit
write memory
```

### Lesson
Wildcards are precise. **One digit** (`0.0` vs. `0.255`) means the difference between "matches all 10/8" and "matches nothing useful". This scenario **actually happened** during the lab — the real-world lesson stuck.

---

## Scenario 8 — STP Root Bridge Hijack

**CCNA Domain:** STP / Root Guard

### Injection
```cisco
! On ASW-A1
spanning-tree vlan 10,20,30,99 priority 0
exit
```

### Symptom
- `show spanning-tree vlan 10` on DSW-A1 shows **ASW-A1** as root — wrong
- Traffic paths shift; inter-VLAN traffic hairpins through the access layer
- Higher latency, potential saturation on ASW uplinks

### Diagnosis
```cisco
! On DSW-A1
show spanning-tree vlan 10
```

```cisco
! On ASW-A1
show spanning-tree vlan 10 | include priority
```
Priority = 0 (lower than DSW-A1's 4096).

> [!note] Root Guard should have prevented this
> With Root Guard on ASW uplinks, ASW-A1's uplink enters **root-inconsistent** state instead of accepting the rogue BPDU. If this injection succeeded, Root Guard is missing on that port.

### Fix
```cisco
! On ASW-A1
no spanning-tree vlan 10,20,30,99 priority
exit
```

Verify on DSW-A1 that root is reclaimed after ~30 seconds. If Root Guard was missing, add it:
```cisco
! On all ASWs
interface range GigabitEthernet0/1 - 2
 spanning-tree guard root
 exit
write memory
```

### Lesson
Root Guard is a critical defense. Without it, any switch with a lower priority can hijack the topology.

---

## Post-Scenario Verification — Key Findings

### Finding 1 — `show interfaces trunk` Empty on CSW1

**This is correct — not a bug.**

After Phase 6, every CSW1 uplink was converted to a **routed port**:

| Interface | Mode | Destination |
|-----------|------|-------------|
| Gi0/1 | Routed | R1 |
| Po1 (Fa0/1–2) | Routed | CSW2 |
| Fa0/3 | Routed | DSW-A1 |
| Fa0/4 | Routed | DSW-A2 |

Routed ports do not participate in VLANs. CSW1 has **no trunks** — `show interfaces trunk` correctly returns empty.

**Where trunks still exist:**
- DSW-A1 ↔ DSW-A2 (Po1)
- DSW-B1 ↔ DSW-B2 (Po1)
- DSW ↔ ASW uplinks
- ASW ↔ DSW uplinks

### Finding 2 — `show spanning-tree root` Not Supported in PT

PT's 3560 accepts:

| Command | PT Support |
|---------|-----------|
| `show spanning-tree summary` | ✅ |
| `show spanning-tree detail` | ✅ |
| `show spanning-tree vlan <id>` | ✅ |
| `show spanning-tree interface <port>` | ✅ |
| `show spanning-tree root` | ❌ |

**Workaround:** Use `show spanning-tree vlan <id>` on DSW or ASW to see root bridge info. On real IOS, `show spanning-tree root` works and lists every VLAN's root.

### Finding 3 — "No spanning tree instance exists" for VLAN 10 on CSW1

Also correct. VLANs 10, 20, 30, 99 exist in CSW1's VLAN database, but **no L2 port on CSW1 carries them** — every port is routed. STP only runs on VLANs with at least one L2 member port.

**Where to check VLAN 10 STP:**
- CSW1: ❌ no instance (routed)
- CSW2: ❌ no instance (routed)
- DSW-A1 / DSW-A2: ✅ STP active
- ASW-A1 / ASW-A2: ✅ STP active

### Finding 4 — PT `show spanning-tree detail` Displays Routed Ports

Your output showed `Port-channel1` in the STP table even though it's routed. **PT display quirk** — on real IOS, routed ports don't appear in STP output.

---

## Final Review Checklist

Run through this **after all 8 scenarios are fixed**. Every item must pass.

### Layer 1 / Layer 2

- [ ] All physical links green/orange (no red)
- [ ] `show etherchannel summary` on DSW-A1/A2 → `Po1(SU)` with both members `(P)`
- [ ] `show etherchannel summary` on CSW1/CSW2 → `Po1(RU)` with both members `(P)`
- [ ] `show interfaces trunk` on DSW-A1 shows Po1, Gi0/2, Fa0/1 (native VLAN 1000)
- [ ] `show spanning-tree vlan 10` on DSW-A1 → `This bridge is the root`

### Layer 3

- [ ] `show ip ospf neighbor` on CSW1 → 4 neighbors (R1, CSW2, DSW-A1, DSW-A2), all FULL
- [ ] `show ip route` on DSW-A1 → Office B subnets + `O*E2 0.0.0.0/0`
- [ ] `show standby brief` on DSW-A1 → Active for 10,20,30,99,300
- [ ] `show standby brief` on DSW-A2 → Standby for 10,20,30,99,300

### Connectivity

- [ ] PC-A1 → PC-A2: ✅
- [ ] PC-A1 → PC-B1: ✅
- [ ] PC-A1 → SRV1: ✅
- [ ] PC-A1 → 8.8.8.8: ✅
- [ ] PC-B1 → 8.8.8.8: ✅

### Security

- [ ] `show port-security` → Secure-up on all access ports
- [ ] `show ip dhcp snooping` → enabled, uplinks trusted
- [ ] `show ip arp inspection` → enabled for VLANs 10,20,30,99
- [ ] `show ip access-lists BLOCK_A_TO_B_SERVERS` → 4 entries
- [ ] `show access-lists 10` → 3 permits + deny
- [ ] PC-A1 cannot ping Office B server (VLAN 30)
- [ ] PC-A1 CAN reach Office B server via HTTP/HTTPS

### Services

- [ ] `show ip nat translations` on R1 → active entries
- [ ] `show ip dhcp binding` on R1 → leases for all clients
- [ ] `show ntp status` on CSW1 → synchronized, ref 10.255.255.254
- [ ] `show snmp community` on any device → ACME-RO (ro)
- [ ] `show ip ssh` on any device → v2.0
- [ ] SSH from SRV1 to DSW-A1: ✅
- [ ] SSH from PC-A1 to DSW-A1: ❌

---

## Project Summary

### Technologies Implemented

| Category | Technologies |
|----------|--------------|
| **Addressing** | IPv4, VLSM, /30 transit links, /24 user subnets |
| **Layer 2** | VLANs, 802.1Q trunks, native VLAN hardening, DTP disable |
| **Redundancy** | Rapid PVST+, Root Guard, BPDU Guard, PortFast |
| **Aggregation** | LACP EtherChannel (L2 + L3) |
| **Layer 3** | SVI, HSRP (v1 + v2), OSPFv2, passive interfaces, default-information originate |
| **Security** | Standard + Extended ACLs, Port Security, DHCP Snooping, DAI |
| **Services** | DHCP (IOS server), NAT/PAT, NTP, SNMP, SSH |
| **Automation** | RESTCONF, NETCONF, Python, Ansible, SDN concepts |

### Statistics

| Metric | Value |
|--------|-------|
| Devices | 17 (12 network + 5 endpoints) |
| Phases | 11 |
| Physical links | 26 |
| Cisco IOS features | ~30 distinct |
| Troubleshooting scenarios | 8 |
| PT limitations documented | 12 |

### Complete List of Packet Tracer Limitations

| # | Command / Feature | Phase | Notes |
|---|-------------------|-------|-------|
| 1 | `ip ospf network point-to-point` | 5 | Not on Ethernet |
| 2 | `no passive-interface Port-channel1` | 6 | Rejected |
| 3 | `log` on ACL entries | 7 | Rejected |
| 4 | `show ip interface Vlan10` (SVI ACL display) | 7 | Misreports bindings |
| 5 | `aging type inactivity` (Port Security) | 8 | Rejected |
| 6 | SNMP service on Server-PT | 9 | Missing |
| 7 | `clock update-calendar` | 9 | Rejected |
| 8 | SRV1 NTP server address field | 9 | GUI incomplete |
| 9 | `snmp-server host` / `location` / `trap-source` | 9 | Partial |
| 10 | `crypto key generate rsa modulus 2048` (inline) | 9 | Requires interactive prompt |
| 11 | `restconf` and related commands | 10 | All rejected |
| 12 | `show spanning-tree root` | 11 | Not supported |

---

## Post-Project Recommendations

### For CCNA Exam Prep

1. **Theory:** Wendell Odom OCG or Jeremy's IT Lab (free)
2. **Practice exams:** Boson ExSim-Max
3. **Additional labs:** Boson NetSim, DevNet Sandbox
4. **Weak areas to revisit:** Wireless, QoS, deeper IP services

### For Real-World Practice

1. **Automation:** Cisco DevNet Sandbox (free, real IOS-XE)
2. **Advanced routing:** Add BGP, multi-area OSPF, redistribution
3. **SDN:** DNA Center in Cisco dCloud
4. **Cloud networking:** AWS/Azure VPC equivalents


## Phase 11 Checkpoint

- [x] All 8 scenarios injected, diagnosed, and fixed
- [x] Final review checklist fully passed
- [x] Lab returned to a working state
- [x] All devices saved to startup-config

---

## 🎉 Project Complete

I've completed an **11-phase, full-stack CCNA master lab** covering every major domain of the CCNA 200-301 exam. I configured, verified, troubleshot, and documented a realistic enterprise topology.

**Key achievements:**
- Built a 17-device, dual-office enterprise network from scratch
- Implemented every major CCNA technology domain
- Solved 8 hands-on troubleshooting scenarios
- Documented 12 Packet Tracer limitations with workarounds
- Produced a complete, publishable technical documentation set

This repository now stands as a **portfolio-grade demonstration** of CCNA skills.

---

*Document maintained as part of the CCNA Master Lab project. For questions, corrections, or contributions, see the repository README.*