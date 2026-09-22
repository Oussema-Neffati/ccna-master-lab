---
title: "Phase 5 — EtherChannel (LACP)"
project: CCNA Master Lab — Acme Corp Two-Office Enterprise
phase: 5
status: complete
tags:
  - ccna
  - lab
  - etherchannel
  - lacp
  - port-channel
  - hsrp
  - packet-tracer
created: 2026-09-22
updated: 2026-09-22
---

# Phase 5 — EtherChannel (LACP)

> [!info] Project Context
> This lab models **Acme Corp**, a two-office enterprise connected through a dual-homed core and a single edge router to a simulated ISP. Each phase builds on the previous one, ending with a fully integrated CCNA lab covering VLANs, STP, EtherChannel, OSPF, ACLs, security hardening, services, and automation.

> [!success] Phase Objective
> Bundle parallel physical links into logical EtherChannels using LACP. Eliminate STP-blocked ports on inter-switch links, multiply bandwidth, add link-level redundancy, and extend HSRP to the Services VLAN.

> [!important] Scope Rules
> - Configure **switches only**.
> - Do **not** touch R1, ISP, or the PCs.
> - OSPF is still deferred to Phase 6.

---

## 1. Design Decisions

| Decision | Rationale |
|----------|-----------|
| **LACP (`mode active`)** | Industry standard (802.3ad). Vendor-neutral. Negotiation prevents silent misconfiguration. |
| **L3 PortChannel between CSW1 ↔ CSW2** | Carries the OSPF backbone (`10.0.0.4/30`). A routed PortChannel skips STP entirely on this link. |
| **L2 PortChannel between DSW pairs** | Carries all office VLANs as a single trunk. STP sees one logical link instead of two physical ones. |
| **`src-dst-ip` load balancing** | Uplinks carry many hosts; hashing on source+destination IP distributes traffic more evenly than MAC-based hashing. |
| **`standby version 2` for group 300** | HSRPv1 supports group IDs 0–255 only. VLAN 300 needs HSRPv2 (range 0–4095). |

---

## 2. EtherChannel Inventory

| ID | Between | Type | Member Ports | Purpose |
|----|---------|------|--------------|---------|
| Po1 | CSW1 ↔ CSW2 | **L3 (routed)** | Fa0/1–2 | OSPF backbone (10.0.0.4/30) |
| Po1 | DSW-A1 ↔ DSW-A2 | **L2 (trunk)** | Fa0/2–3 | Office A VLANs + VLAN 300 |
| Po1 | DSW-B1 ↔ DSW-B2 | **L2 (trunk)** | Fa0/2–3 | Office B VLANs |

> [!note] Same PortChannel number (1) on every switch
> That's fine — PortChannel numbers are **locally significant**, not global. Each switch's Po1 refers to its own local bundle.

---

## 3. L3 EtherChannel — CSW1 ↔ CSW2

This carries the OSPF backbone between the two core switches.

### On CSW1
```cisco
interface range FastEthernet0/1 - 2
 no switchport
 channel-group 1 mode active
 exit

interface Port-channel1
 description L3 link to CSW2
 ip address 10.0.0.5 255.255.255.252
 no shutdown
 exit
```

### On CSW2
```cisco
interface range FastEthernet0/1 - 2
 no switchport
 channel-group 1 mode active
 exit

interface Port-channel1
 description L3 link to CSW1
 ip address 10.0.0.6 255.255.255.252
 no shutdown
 exit
```

> [!warning] Packet Tracer limitation — `ip ospf network point-to-point`
> On real Cisco IOS, adding `ip ospf network point-to-point` on this PortChannel is standard practice — it skips DR/BDR election between two directly connected routers. **Packet Tracer rejects this command on Gigabit Ethernet interfaces** (it only accepts it on Loopback). Since this is a two-device segment, the DR/BDR election is harmless. **Skip the command in the lab.** On real gear, always apply it.

**Verify:**
```cisco
show etherchannel summary
show ip interface brief | include Port-channel
```
Expected: `Po1(RU)` — **R** = routed, **U** = in use.

---

## 4. L2 EtherChannel — DSW-A1 ↔ DSW-A2

### On DSW-A1
```cisco
interface range FastEthernet0/2 - 3
 channel-group 1 mode active
 exit

interface Port-channel1
 description L2 link to DSW-A2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,30,99,300
 switchport nonegotiate
 exit
```

### On DSW-A2
```cisco
interface range FastEthernet0/2 - 3
 channel-group 1 mode active
 exit

interface Port-channel1
 description L2 link to DSW-A1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,30,99,300
 switchport nonegotiate
 exit
```

> [!tip] Order of operations
> Apply `channel-group` on the physical ports first, then configure the PortChannel interface. This avoids the "suspended member" state that occurs when the physical port has conflicting config.

---

## 5. L2 EtherChannel — DSW-B1 ↔ DSW-B2

### On DSW-B1
```cisco
interface range FastEthernet0/2 - 3
 channel-group 1 mode active
 exit

interface Port-channel1
 description L2 link to DSW-B2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,30,99
 switchport nonegotiate
 exit
```

### On DSW-B2
```cisco
interface range FastEthernet0/2 - 3
 channel-group 1 mode active
 exit

interface Port-channel1
 description L2 link to DSW-B1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,30,99
 switchport nonegotiate
 exit
```

> [!note] Office B does not carry VLAN 300
> Only Office A hosts the Services VLAN. SRV1 connects to DSW-A1.

---

## 6. VLAN 300 HSRP — Extend to DSW-A2

Previously, `Vlan300` on DSW-A1 was the only gateway (`10.30.0.1`). Now that DSW-A2 can reach the Services VLAN through the L2 PortChannel, we make it redundant.

### On DSW-A1
```cisco
interface Vlan300
 ip address 10.30.0.2 255.255.255.0
 standby version 2
 standby 300 ip 10.30.0.1
 standby 300 priority 110
 standby 300 preempt
 exit
```

### On DSW-A2
```cisco
interface Vlan300
 description Services
 ip address 10.30.0.3 255.255.255.0
 standby version 2
 standby 300 ip 10.30.0.1
 standby 300 preempt
 exit
```

> [!important] Why `standby version 2`?
> HSRPv1 supports group numbers **0–255** only. Because we used group **300** (matching the VLAN ID), the switch requires HSRPv2. Applying `standby version 2` on **both ends** resolves the error. HSRPv1 and HSRPv2 are **not** interoperable on the same group.

> [!important] SRV1's gateway does NOT change
> SRV1 still uses `10.30.0.1` as its default gateway — but that address is now a **virtual IP**, not DSW-A1's physical SVI. No host changes required.

**Verify:**
```cisco
show standby brief | include Vlan300
```
- DSW-A1 → **Active**
- DSW-A2 → **Standby**

---

## 7. PortChannel Load Balancing (Optional but Recommended)

By default, EtherChannel hashes on source MAC. For routed uplinks with many hosts, `src-dst-ip` distributes traffic more evenly.

### On CSW1, CSW2, DSW-A1, DSW-A2, DSW-B1, DSW-B2
```cisco
port-channel load-balance src-dst-ip
```

**Verify:**
```cisco
show etherchannel load-balance
```
Expected: `Source and Destination IP address`.

---

## 8. STP Behavior After EtherChannel

Before Phase 5, the DSW↔DSW pair had **two physical links** — STP blocked one.

After Phase 5, those two links form **one logical PortChannel**. STP sees them as a single path. The remaining loop (`DSW-A1 → ASW-A1 → DSW-A2 → PortChannel → DSW-A1`) still requires STP to break it — and it does so on the ASW uplink.

### Verify on ASW-A1
```cisco
show spanning-tree vlan 10
```
Expected:
- One uplink: **Root/FWD**
- Other uplink: **Altn/BLK**

### Verify PortChannel forwarding
On **DSW-A1**:
```cisco
show etherchannel port-channel
```
All member ports show `(P)` — bundled and forwarding. The previously blocked member is now part of the bundle.

---

## 9. Verification Steps

### 9.1 EtherChannel Summary (all six switches)
```cisco
show etherchannel summary
```
- CSW1 / CSW2: `Po1(RU)` with Fa0/1, Fa0/2 marked `(P)`
- DSW-A1/A2/B1/B2: `Po1(SU)` with Fa0/2, Fa0/3 marked `(P)`

### 9.2 Trunk Status
```cisco
show interfaces trunk
```
`Po1` listed as trunk with native VLAN 1000 and allowed VLANs.

### 9.3 L3 PortChannel Reachability
On **CSW1**:
```cisco
ping 10.0.0.6
```
Should succeed — proves the routed link works even before OSPF is enabled.

### 9.4 End-to-End Sanity
- PC-A1 → PC-A2: ✅
- PC-A1 → 10.10.99.1: ✅
- PC-A1 → SRV1 (10.30.0.10): ✅
- PC-A1 → PC-B1: ❌ — expected (no inter-office routing until Phase 6)

### 9.5 HSRP Consistency
```cisco
show standby brief
```
- DSW-A1 / DSW-B1: **Active** for VLANs 10, 20, 30, 99
- DSW-A2 / DSW-B2: **Standby** for VLANs 10, 20, 30, 99
- DSW-A1: **Active** for VLAN 300
- DSW-A2: **Standby** for VLAN 300

---

## 10. Phase 5 Checkpoint

- [x] CSW1 ↔ CSW2 L3 PortChannel up (`RU`), IPs `10.0.0.5` / `10.0.0.6`
- [x] DSW-A1 ↔ DSW-A2 L2 PortChannel trunk carrying `10,20,30,99,300`
- [x] DSW-B1 ↔ DSW-B2 L2 PortChannel trunk carrying `10,20,30,99`
- [x] All member ports `(P)` in `show etherchannel summary`
- [x] VLAN 300 SVI added on DSW-A2 with HSRPv2
- [x] SRV1's gateway (`10.30.0.1`) still works as a VIP
- [x] CSW1 can ping CSW2 PortChannel IP `10.0.0.6`
- [x] PC-A1 → PC-A2, → SRV1, → other SVIs ✅
- [x] PC-A1 → PC-B1 ❌ (expected)
- [x] `port-channel load-balance src-dst-ip` set on all six switches
- [x] All six switches saved to startup-config

---

## 11. Troubleshooting Encounters

### 11.1 HSRP group 300 rejected — `PT ERROR: HSRP version 2 is required for specified group number`

**Symptom:** Applying `standby 300 ip 10.30.0.1` on `Vlan300` fails.

**Root Cause:** HSRPv1 supports group numbers **0–255**. Group **300** exceeds that range.

**Fix:**
```cisco
interface Vlan300
 standby version 2
 standby 300 ip 10.30.0.1
 ...
```
Apply `standby version 2` on **both** DSW-A1 and DSW-A2 for that SVI.

> [!warning] HSRPv1 and HSRPv2 are NOT interoperable
> Both ends of the same group must use the same version. A version mismatch will keep the HSRP state stuck in `Init`.

---

### 11.2 `ip ospf network point-to-point` rejected on Port-channel1

**Symptom:** On CSW1/CSW2, entering `ip ospf network point-to-point` on `Port-channel1` returns `% Invalid input detected at '^' marker`.

**Root Cause:** Packet Tracer does not support this command on Gigabit Ethernet (or PortChannel) interfaces. It only accepts it on Loopback interfaces.

**Fix:** **Skip the command in Packet Tracer.** On real Cisco IOS, the command works and is standard practice for a routed link between two devices — always apply it on production gear.

**Impact in this lab:** OSPF will perform a DR/BDR election on the `10.0.0.4/30` segment. With only two switches present, the election is harmless — one becomes DR, the other BDR.

---

## 12. Common Issues & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| PortChannel stuck in `SD` (down) | Member ports have mismatched config | Ensure identical speed, duplex, VLAN settings on both members |
| Member shows `(I)` — standalone | LACP negotiation failed | Verify `mode active` on both ends; check speed/duplex |
| Member shows `(s)` — suspended | Physical port config conflicts with PortChannel | `default interface Fa0/2`, then reapply `channel-group` |
| L3 PortChannel doesn't ping | PortChannel still L2 | `no switchport` on both physical members **before** adding to bundle |
| STP topology unchanged | EtherChannel didn't form | Recheck `show etherchannel summary` |
| VLAN 300 ping from SRV1 fails | `standby 300` group mismatch or missing `standby version 2` | Confirm group number 300 and HSRPv2 on **both** DSWs |
| HSRP stuck in `Init` on Vlan300 | Version mismatch between DSW-A1 and DSW-A2 | Apply `standby version 2` on both ends |

---

## 13. Lessons Learned

> [!note] Bulk `channel-group` and `interface range` safety
> Always apply `channel-group` on the **physical** range first, then configure the **PortChannel** interface separately. Configuring them in the wrong order causes `suspended` member states.

> [!note] HSRP group number ≠ HSRP version
> Group numbers are just identifiers; version determines the group-ID range and packet format. If your group number is > 255, you must use HSRPv2.

> [!note] Packet Tracer ≠ real IOS
> Some valid Cisco commands are unsupported in Packet Tracer (like `ip ospf network point-to-point` on Ethernet). In those cases, note the intent, apply it on real gear, and move on in the lab.

---

## 14. Next Phase

➡️ **[Phase 6 — OSPFv2 Single Area](Phase-6-OSPFv2.md)**

In Phase 6 we will:
- Enable OSPF process 1 on R1, CSW1, CSW2
- Advertise the 10.0.0.0/16 space and office subnets
- Set **passive interfaces** on all SVIs and loopbacks
- Originate a default route from R1
- Verify **PC-A1 ↔ PC-B1** now works

---

*Document maintained as part of the CCNA Master Lab project. For questions, corrections, or contributions, see the repository README.*