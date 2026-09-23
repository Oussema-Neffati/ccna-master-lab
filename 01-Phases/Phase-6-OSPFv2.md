---
title: "Phase 6 — OSPFv2 Single Area"
project: CCNA Master Lab — Acme Corp Two-Office Enterprise
phase: 6
status: complete
tags:
  - ccna
  - lab
  - ospf
  - ospfv2
  - routing
  - asbr
  - packet-tracer
created: 2026-09-23
updated: 2026-09-23
---

# Phase 6 — OSPFv2 Single Area

> [!info] Project Context
> This lab models **Acme Corp**, a two-office enterprise connected through a dual-homed core and a single edge router to a simulated ISP. Each phase builds on the previous one, ending with a fully integrated CCNA lab covering VLANs, STP, EtherChannel, OSPF, ACLs, security hardening, services, and automation.

> [!success] Phase Objective
> Route between the two offices and toward the Internet using OSPFv2 single-area. This is where PC-A1 finally reaches PC-B1, and where R1 originates a default route into OSPF.

> [!important] Architectural change
> The CSW↔DSW uplinks that were Layer-2 trunks in Phase 2 become **routed ports**. DSWs keep their SVIs and HSRP for office traffic — they now peer with the core via OSPF. STP is unaffected on those ports (they no longer participate in STP).

---

## 1. Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Single OSPF area (area 0)** | Small topology; no need for multi-area. Meets CCNA objective directly. |
| **Routed ports between CSW and DSW** | Core↔Distribution is a routed boundary in enterprise design. Removes STP overhead on transit links. |
| **`passive-interface default` on R1, DSWs** | Secure-by-default — new interfaces are passive unless explicitly activated. |
| **R1 as ASBR with `default-information originate`** | Injects a default route into OSPF so offices can reach the Internet via the edge. |
| **Loopback0 as OSPF router-ID on every device** | Stable router-ID that survives interface changes. |
| **/30 transit subnets** | Standard for point-to-point links; conserves address space. |

---

## 2. OSPF Address Summary

| Device | Router ID | Loopback0 | Active OSPF Interfaces |
|--------|-----------|-----------|------------------------|
| R1 | 10.255.255.254 | 10.255.255.254/32 | Gi0/0, Gi0/2 |
| CSW1 | 10.255.255.1 | 10.255.255.1/32 | Gi0/1, Po1, Fa0/3, Fa0/4 |
| CSW2 | 10.255.255.2 | 10.255.255.2/32 | Gi0/1, Po1, Fa0/3, Fa0/4 |
| DSW-A1 | 10.255.255.11 | 10.255.255.11/32 | Gi0/1 |
| DSW-A2 | 10.255.255.12 | 10.255.255.12/32 | Gi0/1 |
| DSW-B1 | 10.255.255.21 | 10.255.255.21/32 | Gi0/1 |
| DSW-B2 | 10.255.255.22 | 10.255.255.22/32 | Gi0/1 |

**Transit subnets (all /30, all in area 0):**

| Subnet | Link |
|--------|------|
| 10.0.0.0 | R1 ↔ CSW1 |
| 10.0.0.4 | CSW1 ↔ CSW2 (L3 Port-channel) |
| 10.0.0.8 | R1 ↔ CSW2 |
| 10.0.1.0 | CSW1 ↔ DSW-A1 |
| 10.0.2.0 | CSW1 ↔ DSW-A2 |
| 10.0.3.0 | CSW2 ↔ DSW-B1 |
| 10.0.4.0 | CSW2 ↔ DSW-B2 |

---

## 3. ISP Configuration

The ISP is a stub router — no OSPF, just interfaces and a static route back to R1.

```cisco
enable
configure terminal
hostname ISP

interface GigabitEthernet0/0
 description To R1
 ip address 203.0.113.2 255.255.255.252
 no shutdown
 exit

interface Loopback0
 description Simulated Internet
 ip address 8.8.8.8 255.255.255.255
 exit

ip route 0.0.0.0 0.0.0.0 203.0.113.1
end
write memory
```

---

## 4. R1 — Edge Router / OSPF ASBR

```cisco
enable
configure terminal
hostname R1

interface GigabitEthernet0/0
 description To CSW1
 ip address 10.0.0.1 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet0/1
 description To ISP
 ip address 203.0.113.1 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet0/2
 description To CSW2
 ip address 10.0.0.9 255.255.255.252
 no shutdown
 exit

interface Loopback0
 ip address 10.255.255.254 255.255.255.255
 exit

ip route 0.0.0.0 0.0.0.0 203.0.113.2

router ospf 1
 router-id 10.255.255.254
 passive-interface default
 no passive-interface GigabitEthernet0/0
 no passive-interface GigabitEthernet0/2
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.0.8 0.0.0.3 area 0
 network 10.255.255.254 0.0.0.0 area 0
 default-information originate
 exit

end
write memory
```

> [!tip] `passive-interface default`
> Makes every interface passive by default, then we selectively activate only transit links. Any new interface added later is automatically passive — a secure default.

---

## 5. CSW1 — Core Switch 1

### 5.1 Routed uplinks

```cisco
interface GigabitEthernet0/1
 description To R1
 no switchport
 ip address 10.0.0.2 255.255.255.252
 no shutdown
 exit

interface FastEthernet0/3
 description To DSW-A1
 no switchport
 ip address 10.0.1.1 255.255.255.252
 no shutdown
 exit

interface FastEthernet0/4
 description To DSW-A2
 no switchport
 ip address 10.0.2.1 255.255.255.252
 no shutdown
 exit

interface Loopback0
 ip address 10.255.255.1 255.255.255.255
 exit
```

> `Port-channel1` (10.0.0.5/30 to CSW2) was already routed in Phase 5.

### 5.2 OSPF — final working config

```cisco
router ospf 1
 router-id 10.255.255.1
 no passive-interface default
 passive-interface Loopback0
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.0.4 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.3 area 0
 network 10.0.2.0 0.0.0.3 area 0
 network 10.255.255.1 0.0.0.0 area 0
 exit
```

> [!warning] Packet Tracer limitation — `no passive-interface Port-channel1`
> **Symptom:** `no passive-interface Port-channel1` is rejected by PT with a syntax error, even though `Port-channel1` exists and is up.
>
> **Impact:** If left unfixed, CSW1 and CSW2 never form an OSPF adjacency over the L3 Port-channel. All traffic between CSW1 and CSW2 hairpins through R1 — the EtherChannel sits idle.
>
> **Fix:** Do **not** use `passive-interface default`. Because active is the default state when `passive-interface default` isn't configured, we only need to mark the non-transit interfaces (Loopback0, any SVIs) as passive. Transit links (Gi0/1, Po1, Fa0/3, Fa0/4) stay active automatically.
>
> **Corrected config (see above):** Replace `passive-interface default` with `no passive-interface default`, then explicitly mark only Loopback0 passive. On real Cisco IOS, the original `passive-interface default` + `no passive-interface Port-channel1` works as written.

---

## 6. CSW2 — Core Switch 2

### 6.1 Routed uplinks

```cisco
interface GigabitEthernet0/1
 description To R1
 no switchport
 ip address 10.0.0.10 255.255.255.252
 no shutdown
 exit

interface FastEthernet0/3
 description To DSW-B1
 no switchport
 ip address 10.0.3.1 255.255.255.252
 no shutdown
 exit

interface FastEthernet0/4
 description To DSW-B2
 no switchport
 ip address 10.0.4.1 255.255.255.252
 no shutdown
 exit

interface Loopback0
 ip address 10.255.255.2 255.255.255.255
 exit
```

### 6.2 OSPF — final working config

```cisco
router ospf 1
 router-id 10.255.255.2
 no passive-interface default
 passive-interface Loopback0
 network 10.0.0.4 0.0.0.3 area 0
 network 10.0.0.8 0.0.0.3 area 0
 network 10.0.3.0 0.0.0.3 area 0
 network 10.0.4.0 0.0.0.3 area 0
 network 10.255.255.2 0.0.0.0 area 0
 exit
```

---

## 7. DSW-A1 — Office A Distribution 1

```cisco
interface GigabitEthernet0/1
 description To CSW1
 no switchport
 ip address 10.0.1.2 255.255.255.252
 no shutdown
 exit

interface Loopback0
 ip address 10.255.255.11 255.255.255.255
 exit

router ospf 1
 router-id 10.255.255.11
 passive-interface default
 no passive-interface GigabitEthernet0/1
 network 10.0.1.0 0.0.0.3 area 0
 network 10.10.10.0 0.0.0.255 area 0
 network 10.10.20.0 0.0.0.255 area 0
 network 10.10.30.0 0.0.0.255 area 0
 network 10.10.99.0 0.0.0.255 area 0
 network 10.30.0.0 0.0.0.255 area 0
 network 10.255.255.11 0.0.0.0 area 0
 exit
```

> [!note] DSWs keep `passive-interface default`
> DSW uplinks (Gi0/1) are physical interfaces, not Port-channels, so `no passive-interface GigabitEthernet0/1` works fine. Their SVIs remain passive automatically — advertised but no OSPF neighbor on those VLANs.

---

## 8. DSW-A2

```cisco
interface GigabitEthernet0/1
 description To CSW1
 no switchport
 ip address 10.0.2.2 255.255.255.252
 no shutdown
 exit

interface Loopback0
 ip address 10.255.255.12 255.255.255.255
 exit

router ospf 1
 router-id 10.255.255.12
 passive-interface default
 no passive-interface GigabitEthernet0/1
 network 10.0.2.0 0.0.0.3 area 0
 network 10.10.10.0 0.0.0.255 area 0
 network 10.10.20.0 0.0.0.255 area 0
 network 10.10.30.0 0.0.0.255 area 0
 network 10.10.99.0 0.0.0.255 area 0
 network 10.30.0.0 0.0.0.255 area 0
 network 10.255.255.12 0.0.0.0 area 0
 exit
```

---

## 9. DSW-B1

```cisco
interface GigabitEthernet0/1
 description To CSW2
 no switchport
 ip address 10.0.3.2 255.255.255.252
 no shutdown
 exit

interface Loopback0
 ip address 10.255.255.21 255.255.255.255
 exit

router ospf 1
 router-id 10.255.255.21
 passive-interface default
 no passive-interface GigabitEthernet0/1
 network 10.0.3.0 0.0.0.3 area 0
 network 10.20.10.0 0.0.0.255 area 0
 network 10.20.20.0 0.0.0.255 area 0
 network 10.20.30.0 0.0.0.255 area 0
 network 10.20.99.0 0.0.0.255 area 0
 network 10.255.255.21 0.0.0.0 area 0
 exit
```

---

## 10. DSW-B2

```cisco
interface GigabitEthernet0/1
 description To CSW2
 no switchport
 ip address 10.0.4.2 255.255.255.252
 no shutdown
 exit

interface Loopback0
 ip address 10.255.255.22 255.255.255.255
 exit

router ospf 1
 router-id 10.255.255.22
 passive-interface default
 no passive-interface GigabitEthernet0/1
 network 10.0.4.0 0.0.0.3 area 0
 network 10.20.10.0 0.0.0.255 area 0
 network 10.20.20.0 0.0.0.255 area 0
 network 10.20.30.0 0.0.0.255 area 0
 network 10.20.99.0 0.0.0.255 area 0
 network 10.255.255.22 0.0.0.0 area 0
 exit
```

---

## 11. Verification

### 11.1 OSPF Neighbors

On **R1**:
```cisco
show ip ospf neighbor
```
Expected: **2 neighbors** — CSW1 (10.0.0.2) and CSW2 (10.0.0.10), both **FULL**.

On **CSW1**:
```cisco
show ip ospf neighbor
```
Expected: **4 neighbors** — R1, CSW2 (via Port-channel1), DSW-A1, DSW-A2, all **FULL**.

On **DSW-A1**:
```cisco
show ip ospf neighbor
```
Expected: **1 neighbor** — CSW1 (10.0.1.1), **FULL**.

### 11.2 Routing Table (CSW1 — reference output)

```cisco
show ip route
```

Key observations after the fix:
- `C` 10.0.0.0/30 via Gi0/1 (R1 uplink)
- `C` 10.0.0.4/30 via Port-channel1 (CSW2 uplink)
- `O` 10.0.0.8/30 via 10.0.0.1
- `O` Office B subnets (10.20.10.0/24, etc.) via **10.0.0.6 (Port-channel1)** with cost 3, and via 10.0.0.1 with cost 4
- `O*E2 0.0.0.0/0` via 10.0.0.1

### 11.3 Default Route Propagation

On **DSW-A1**:
```cisco
show ip route | include 0.0.0.0
```
Expected: `O*E2 0.0.0.0/0 [110/1] via 10.0.1.1, ...`

### 11.4 The Money Test

From **PC-A1**:
```
ping 10.20.10.10
```

✅ **Inter-office routing now works.**

### 11.5 Internet Reachability

From **PC-A1**:
```
ping 203.0.113.2
ping 8.8.8.8
```

- 203.0.113.2 → likely fails until NAT (Phase 9)
- 8.8.8.8 → fails until NAT (Phase 9)

> This is **expected** — private IPs cannot traverse the Internet without NAT, which is Phase 9.

---

## 12. Phase 6 Checkpoint

- [x] ISP configured with Gi0/0 (203.0.113.2/30) and Loopback0 (8.8.8.8/32)
- [x] R1 configured with 3 IPs + Loopback0 + static default + OSPF
- [x] CSW↔DSW uplinks converted to routed ports
- [x] OSPF enabled on R1, CSW1, CSW2, DSW-A1/A2/B1/B2
- [x] SVIs and loopbacks passive
- [x] `show ip ospf neighbor` shows FULL adjacencies on all transit links
- [x] CSW1 ↔ CSW2 adjacency via Port-channel1 (fixed PT limitation)
- [x] `show ip route` on DSW-A1 shows Office B subnets + default route
- [x] **PC-A1 ↔ PC-B1 ping succeeds**
- [x] All devices saved

---

## 13. Troubleshooting Encounters

### 13.1 `no passive-interface Port-channel1` rejected by Packet Tracer

**Symptom:** On CSW1/CSW2, entering `no passive-interface Port-channel1` inside `router ospf 1` returns an invalid input error.

**Consequence:** With `passive-interface default` still active, Port-channel1 stays passive and the CSW1↔CSW2 OSPF adjacency never forms. All inter-office traffic hairpins through R1.

**Verification:** `show ip route` on CSW1 shows all Office B subnets learned via **10.0.0.1** (R1) instead of **10.0.0.6** (CSW2).

**Fix:** Do not use `passive-interface default` on CSWs. Because active is the default state, only the non-transit interfaces (Loopback0, any SVIs) need to be explicitly marked passive:

```cisco
router ospf 1
 no passive-interface default
 passive-interface Loopback0
 ...
```

> [!warning] Watch out for inverted logic
> A tempting but **wrong** workaround is to keep `passive-interface default` and mark the transit interfaces passive by mistake. This disables all OSPF adjacencies on that switch. The correct approach is: don't use `passive-interface default` at all; explicitly mark only the interfaces that should be passive.

**On real Cisco IOS:** The original config (`passive-interface default` + `no passive-interface Port-channel1`) works as written. This is a Packet Tracer emulation gap.

---

### 13.2 `ip ospf network point-to-point` rejected (from Phase 5)

**Symptom:** The command fails on Ethernet interfaces in Packet Tracer.

**Impact:** OSPF performs DR/BDR election on the CSW1↔CSW2 Port-channel. With two routers on the segment, this is harmless.

**Workaround:** Skip in PT. Apply on real gear — it skips the DR/BDR election on point-to-point transit links.

---

## 14. Common Issues & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| No OSPF neighbors | Interface still in `switchport` mode | `no switchport` on the physical port |
| Neighbor stuck in EXSTART/EXCHANGE | MTU mismatch or duplicate router-ID | `show ip ospf interface` and `show ip ospf` |
| Neighbor stuck in INIT | One side not sending hellos | Verify `no passive-interface` on both ends |
| Neighbor stuck in 2-WAY | DR/BDR election issue | With only two routers, check for duplicate router-IDs |
| Routes missing | Network statement wildcard wrong | Wildcard must match subnet size (e.g., `0.0.0.255` for /24) |
| Default route not propagated | `default-information originate` missing on R1 | Add under `router ospf 1` |
| Inter-office ping fails | CSW1↔CSW2 adjacency missing | See §13.1 — verify Port-channel1 is not passive |
| Asymmetric routing | Both DSWs advertising the same /24 | Normal with HSRP + equal-cost OSPF. Not an issue. |

---

## 15. Next Phase

➡️ **[Phase 7 — ACLs (Standard & Extended)](Phase-7-ACLs.md)**

In Phase 7 we will:
- Restrict VTY/SSH access to the management subnets (standard ACL)
- Block Office A PCs from reaching Office B servers, except HTTP/HTTPS (extended ACL)
- Verify traffic is filtered as intended

---

*Document maintained as part of the CCNA Master Lab project. For questions, corrections, or contributions, see the repository README.*