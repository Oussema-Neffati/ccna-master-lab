---
title: "Phase 2 — VLANs & 802.1Q Trunks"
project: CCNA Master Lab — Acme Corp Two-Office Enterprise
phase: 2
status: complete
tags:
  - ccna
  - lab
  - vlan
  - trunking
  - 802.1q
created: 2026-09-18
updated: 2026-09-18
---

# Phase 2 — VLANs & 802.1Q Trunks

> [!info] Project Context
> This lab models **Acme Corp**, a two-office enterprise connected through a dual-homed core and a single edge router to a simulated ISP. Each phase builds on the previous one, ending with a fully integrated CCNA lab covering VLANs, STP, EtherChannel, OSPF, ACLs, security hardening, services, and automation.

> [!success] Phase Objective
> Build the Layer 2 foundation: create the VLAN database on all switches, configure access ports on the access layer, and configure 802.1Q trunks with a hardened native VLAN. **No IP addressing on SVIs yet — that comes in Phase 3.**

> [!important] Scope Rules
> - Configure **switches only**.
> - Do **not** touch R1, ISP, or the PCs.
> - Do **not** configure EtherChannels yet — that is Phase 5.

---

## 1. Design Decisions

| Decision | Rationale |
|----------|-----------|
| **VLAN IDs 10, 20, 30, 99, 300, 1000** | Logical separation by department + a management VLAN + a services VLAN + a native VLAN. |
| **VLAN 1000 as native** | Native VLAN 1 is a well-known attack surface. Using an unused VLAN (1000) mitigates VLAN hopping. |
| **DTP disabled (`switchport nonegotiate`)** | Prevents an attacker from negotiating a trunk via DTP. |
| **No VTP** | VTP is a security and stability risk in production. All VLANs are created manually on each switch. |
| **VLAN pruning per office** | Office A only carries Office A VLANs, and vice versa. The core trunks carry everything. |
| **Trunk encapsulation `dot1q`** | Industry standard. 3560s also support ISL, but ISL is deprecated. 2960s only support dot1q. |

---

## 2. VLAN Database (All 10 Switches)

Created manually on **CSW1, CSW2, DSW-A1, DSW-A2, DSW-B1, DSW-B2, ASW-A1, ASW-A2, ASW-B1, ASW-B2**.

```cisco
enable
configure terminal

vlan 10
 name PCs
vlan 20
 name IP_Phones
vlan 30
 name Servers_WiFi
vlan 99
 name Management
vlan 300
 name Services
vlan 1000
 name Native_Unused
exit
```

> [!warning] No VTP
> Packet Tracer's 2960 and 3560 do not support VTP v3, and this lab deliberately avoids VTP entirely. **VLANs must be created on every switch manually.** If you later see `show vlan brief` missing a VLAN on some switch, this is why.

### Verify
```cisco
show vlan brief
```
VLANs 1, 10, 20, 30, 99, 300, 1000 should all appear as **active**.

---

## 3. Access Ports (ASWs Only)

Assigned on **ASW-A1, ASW-A2, ASW-B1, ASW-B2**. Only Fa0/1 is physically used in Phase 1, but configuring the full range saves time later.

```cisco
interface range FastEthernet0/1 - 10
 switchport mode access
 switchport access vlan 10
 description PCs
exit

interface range FastEthernet0/11 - 15
 switchport mode access
 switchport access vlan 20
 description IP Phones
exit

interface range FastEthernet0/16 - 20
 switchport mode access
 switchport access vlan 30
 description WiFi   ! Office B: change description to "Servers"
exit

interface range FastEthernet0/21 - 23
 switchport mode access
 switchport access vlan 99
 description Management
exit
```

> [!tip] Office B cosmetic difference
> VLAN 30 is described as **WiFi** in Office A and **Servers** in Office B. VLAN IDs and behavior are identical — only the description differs.

---

## 4. Trunk Configuration

### 4.1 Core ↔ Distribution

**On CSW1 (uplinks to DSW-A1, DSW-A2):**
```cisco
interface range FastEthernet0/3 - 4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,30,99,300
 switchport nonegotiate
exit
```

**On CSW2 (uplinks to DSW-B1, DSW-B2):** identical commands on Fa0/3–4.

**On DSW-A1, DSW-A2, DSW-B1, DSW-B2** (uplink Gi0/1 toward the core):
```cisco
interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,30,99,300
 switchport nonegotiate
exit
```

### 4.2 Distribution ↔ Access

**On DSW-A1, DSW-A2** (downlinks toward Office A ASWs):
```cisco
interface range GigabitEthernet0/2, FastEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,30,99
 switchport nonegotiate
exit
```

**On DSW-B1, DSW-B2** (downlinks toward Office B ASWs): same commands.

**On ASW-A1, ASW-A2, ASW-B1, ASW-B2** (uplinks):
```cisco
interface range GigabitEthernet0/1 - 2
 switchport mode trunk
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,30,99
 switchport nonegotiate
exit
```

> [!warning] 2960 does not accept `encapsulation dot1q`
> The 2960 only supports 802.1Q. If you type `switchport trunk encapsulation dot1q` on an ASW, it errors — **skip that line** on all 2960s.

---

## 5. Trunk Summary

| Link | Ports | Native VLAN | Allowed VLANs |
|------|-------|-------------|---------------|
| CSW1 → DSW-A1/A2 | Fa0/3–4 | 1000 | 10,20,30,99,300 |
| CSW2 → DSW-B1/B2 | Fa0/3–4 | 1000 | 10,20,30,99,300 |
| DSW-A1/A2 → CSW1 | Gi0/1 | 1000 | 10,20,30,99,300 |
| DSW-B1/B2 → CSW2 | Gi0/1 | 1000 | 10,20,30,99,300 |
| DSW-A1/A2 → ASW-A1/A2 | Gi0/2, Fa0/1 | 1000 | 10,20,30,99 |
| DSW-B1/B2 → ASW-B1/B2 | Gi0/2, Fa0/1 | 1000 | 10,20,30,99 |
| ASW-A1/A2 → DSWs | Gi0/1–2 | 1000 | 10,20,30,99 |
| ASW-B1/B2 → DSWs | Gi0/1–2 | 1000 | 10,20,30,99 |

---

## 6. Explicitly NOT Configured Yet

> [!note] Deferred to later phases
> - **EtherChannel** on DSW↔DSW links (Fa0/2–Fa0/3) — **Phase 5**
> - **EtherChannel** on CSW↔CSW links (Fa0/1–Fa0/2) — **Phase 5**
> - **SVIs / HSRP** — **Phase 3**
> - **STP root tuning** — **Phase 4**
> - **Port Security / DHCP Snooping / DAI** — **Phase 8**

This is intentional: you need to observe STP behavior on individual links before bundling them.

---

## 7. Verification Steps

### 7.1 VLAN Database
```cisco
show vlan brief
```
- All 10 switches should show VLANs 10, 20, 30, 99, 300, 1000 as **active**.

### 7.2 Trunk Status (CSW1)
```cisco
show interfaces trunk
```
Expected:
- Fa0/3 and Fa0/4 listed as trunks
- **Native VLAN: 1000**
- **Allowed VLANs: 10,20,30,99,300**
- **Encapsulation: 802.1q**

### 7.3 Trunk Status (DSW-A1)
```cisco
show interfaces trunk
```
Expected:
- Gi0/1, Gi0/2, Fa0/1 listed as trunks
- Native VLAN: 1000
- Allowed VLANs: 10,20,30,99

### 7.4 Trunk Status (ASW-A1)
```cisco
show interfaces trunk
```
Expected:
- Gi0/1 and Gi0/2 listed as trunks
- Native VLAN: 1000
- Allowed VLANs: 10,20,30,99

### 7.5 Access Port Check (ASW-A1)
```cisco
show interfaces switchport
```
- Fa0/1 → access mode, VLAN 10
- Fa0/11 → access mode, VLAN 20
- Fa0/16 → access mode, VLAN 30
- Fa0/21 → access mode, VLAN 99

### 7.6 Port-Specific Detail (CSW1)
```cisco
show interfaces FastEthernet0/3 switchport
```
- Administrative Mode: trunk
- Operational Mode: trunk
- Trunking Encapsulation: dot1q
- Negotiation of Trunking: Off (`nonegotiate`)

---

## 8. Phase 2 Checkpoint

- [x] VLANs 10, 20, 30, 99, 300, 1000 exist on **all 10 switches**
- [x] Access ports on all 4 ASWs mapped to VLANs 10/20/30/99
- [x] All uplinks configured as trunks
- [x] Native VLAN = 1000 on **every** trunk
- [x] DTP disabled (`switchport nonegotiate`) on **every** trunk
- [x] No EtherChannel, no SVIs, no HSRP yet
- [x] All switches saved to startup-config

---

## 9. Observed Behavior (Expected Weirdness)

| Observation | Why it's Expected |
|-------------|--------------------|
| `show interfaces trunk` on DSWs shows Fa0/2 and Fa0/3 as "not-trunking" | Those are still plain links pending Phase 5 EtherChannel. |
| PC-A1 cannot ping PC-A2 | No IP addresses configured yet. |
| STP still shows one blocked port per loop | No root tuning yet — that's Phase 4. |
| `show interfaces trunk` on ASWs shows only Gi0/1–2 (not Fa0/x) | Fa0/x ports are access ports, not trunks. |

---

## 10. Common Issues & Fixes

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Trunk doesn't come up | One side is `access`, other is `trunk` | Set both ends to `switchport mode trunk` |
| Native VLAN mismatch warning | One end still has native VLAN 1 | Set `switchport trunk native vlan 1000` on both ends |
| VLAN missing from `show vlan brief` | Not created manually | Re-enter `vlan <id>` and `name <name>` |
| Trunk is up but VLANs are pruned | VLAN not in `allowed vlan` list | Add it with `switchport trunk allowed vlan add <id>` |
| `encapsulation dot1q` errors on 2960 | 2960 doesn't support the command | Skip the line on ASWs |

---

## 11. Next Phase

➡️ **[Phase 3 — Inter-VLAN Routing (SVIs + HSRP)](Phase-3-Inter-VLAN-Routing.md)**

In Phase 3 we will:
- Enable `ip routing` on all 4 DSWs
- Configure SVIs for VLANs 10, 20, 30, 99 on all 4 DSWs
- Configure HSRP with preempt and priority on DSW-A1/B1 as active
- Configure VLAN 300 SVI + SRV1 access port
- Assign static IPs to endpoint hosts (temporary — DHCP comes in Phase 9)
- Test inter-VLAN routing and HSRP failover

---

*Document maintained as part of the CCNA Master Lab project. For questions, corrections, or contributions, see the repository README.*