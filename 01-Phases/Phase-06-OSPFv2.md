# Phase 6 — OSPFv2 Single Area

> [!NOTE]\n> **Prerequisites & Navigation**
> **Previous:** [Phase 5 — EtherChannel](Phase-05-EtherChannel.md) · **Next:** [Phase 7 — ACLs](Phase-07-ACLs.md)
> Master addressing plan: [MASTER-ADDRESSING.md](../MASTER-ADDRESSING.md)

> > [!TIP]\n> **Phase Objective**
> Route between the two offices and toward the Internet using OSPFv2 single-area.
> This is where PC-A1 finally reaches PC-B1, and where R1 originates a default
> route into OSPF.

> [!IMPORTANT] Architectural change from Phase 2
> The CSW↔DSW uplinks that were Layer 2 trunks in Phase 2 become **routed ports**
> in this phase. The old `switchport mode trunk` configuration on those four
> ports is replaced by `no switchport` + IP addressing.
>
> DSWs keep their SVIs and HSRP for office traffic — they now peer with the core
> via OSPF. STP is unaffected on those ports (they no longer participate in STP).

---

## 1. Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Single OSPF area (area 0)** | Small topology; no need for multi-area. Meets CCNA objective directly. |
| **Routed ports between CSW and DSW** | Core↔Distribution is a routed boundary in enterprise design. Removes STP overhead on transit links. |
| **`passive-interface default` on R1 and DSWs** | Secure-by-default — new interfaces are passive unless explicitly activated. |
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

> Full plan: [MASTER-ADDRESSING.md](../MASTER-ADDRESSING.md)

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

> [!NOTE] ISP default route
> The ISP has a static default route pointing back to R1. This means that once NAT
> is configured on R1 (Phase 9), inside hosts **can** reach `8.8.8.8` — and
> interestingly, may also reach `8.8.8.8` **before** Phase 9 if R1's own IP is
> translated. What they cannot reach without NAT is the public Internet generally.
> See §10.4 for what was actually observed.

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

> [!WARNING] Packet Tracer limitation — `no passive-interface Port-channel1`
> PT rejects `no passive-interface Port-channel1` even though the interface exists
> and is up. With `passive-interface default` still active, Port-channel1 stays
> passive and the CSW1↔CSW2 adjacency never forms — all inter-core traffic then
> hairpins through R1.
>
> **Fix:** Do **not** use `passive-interface default` on the CSWs. Because active
> is the default state when `passive-interface default` isn't configured, only
> the non-transit interfaces (Loopback0) need to be marked passive. Transit links
> (Gi0/1, Po1, Fa0/3, Fa0/4) stay active automatically.
>
> On real Cisco IOS, the original `passive-interface default` +
> `no passive-interface Port-channel1` works as written.

---

## 6. CSW2 — Core Switch 2

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

> [!NOTE] DSWs keep `passive-interface default`
> DSW uplinks (Gi0/1) are physical interfaces, not Port-channels, so
> `no passive-interface GigabitEthernet0/1` works fine. Their SVIs remain
> passive automatically — advertised but no OSPF neighbor on those VLANs.

---

## 8. DSW-A2, DSW-B1, DSW-B2

Same pattern as DSW-A1, with the corresponding transit subnet and loopback:

| Device | Transit | Loopback0 | Office subnets |
|--------|---------|-----------|----------------|
| DSW-A2 | 10.0.2.0/30 | 10.255.255.12/32 | 10.10.0.0/16 |
| DSW-B1 | 10.0.3.0/30 | 10.255.255.21/32 | 10.20.0.0/16 |
| DSW-B2 | 10.0.4.0/30 | 10.255.255.22/32 | 10.20.0.0/16 |

For each:

```cisco
router ospf 1
 router-id <loopback>
 passive-interface default
 no passive-interface GigabitEthernet0/1
 network <transit> 0.0.0.3 area 0
 network <office /16 in /24s> area 0
 network <loopback> 0.0.0.0 area 0
 exit
```

---

## 9. Verification

### 9.1 OSPF Neighbors

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

### 9.2 Routing Table — Best-Path Only

On **CSW1**:
```cisco
show ip route ospf
```

**Expected:** every Office B subnet appears as a **single route**, via the best
path only:

```
O  10.20.10.0/24 [110/3] via 10.0.0.6, ... Port-channel1
```

> [!IMPORTANT] OSPF installs only the best path
> Earlier drafts of this document showed two entries per Office B subnet (cost 3
> via Port-channel1 AND cost 4 via R1). That was incorrect. **OSPF installs only
> the best-cost path** into the routing table. The lower-cost path (via CSW2 over
> the L3 Port-channel) is the one used. If you ever need to see both, use
> `show ip ospf rib` — which shows the OSPF RIB (candidates), not the global
> routing table.

### 9.3 Default Route Propagation

On **DSW-A1**:
```cisco
show ip route | include 0.0.0.0
```
Expected: `O*E2 0.0.0.0/0 [110/1] via 10.0.1.1, ...`

### 9.4 The Money Test — Inter-Office

From **PC-A1**:
```
ping 10.20.10.10
```
✅ **Inter-office routing now works.**

### 9.5 Internet Reachability (Observed)

From **PC-A1**:
```
ping 8.8.8.8
```

Observed result at this phase: **partially working / unreliable.**

- A ping to `203.0.113.2` (ISP's outside interface) succeeds — the ISP has a
  static route back to R1's outside IP, and R1 knows how to reach the inside
  network, so routing works **without NAT** for that specific host.
- A ping to `8.8.8.8` (ISP loopback) may or may not succeed depending on how PT
  handles the return path. On real hardware it would typically fail without NAT.

> [!NOTE] Full Internet reachability is completed in Phase 9
> Phase 9 adds PAT on R1, at which point every inside host can reach the Internet
> reliably. Any partial success at this phase is due to PT's simplified ISP
> emulation, not a real routing behavior.

### 9.6 Full Route Table

On **CSW1**:
```cisco
show ip route
```
Expected to see:
- `C` 10.0.0.0/30, 10.0.0.4/30, 10.0.1.0/30, 10.0.2.0/30
- `O` 10.0.0.8/30, 10.0.3.0/30, 10.0.4.0/30
- `O` 10.10.0.0/16 (via DSW-A1 and DSW-A2)
- `O` 10.20.0.0/16 (via CSW2)
- `O` 10.30.0.0/24
- `O*E2` 0.0.0.0/0 (from R1)

---

## 10. Troubleshooting Encounters

### 10.1 `no passive-interface Port-channel1` rejected

**Symptom:** On CSW1/CSW2, entering `no passive-interface Port-channel1` inside
`router ospf 1` returns an invalid input error.

**Consequence:** With `passive-interface default` still active, Port-channel1
stays passive and the CSW1↔CSW2 OSPF adjacency never forms. All inter-office
traffic hairpins through R1.

**Verification:** `show ip route` on CSW1 shows all Office B subnets learned via
**10.0.0.1** (R1) instead of **10.0.0.6** (CSW2).

**Fix:** Do not use `passive-interface default` on the CSWs. Because active is
the default state, only the non-transit interfaces (Loopback0, any SVIs) need
to be explicitly marked passive.

> [!CAUTION] Watch out for inverted logic
> A tempting but **wrong** workaround is to keep `passive-interface default` and
> mark the transit interfaces passive by mistake. That disables **all** OSPF
> adjacencies on that switch. The correct approach is: don't use
> `passive-interface default` at all; explicitly mark only what should be passive.

**On real Cisco IOS:** The original config works as written.

### 10.2 `ip ospf network point-to-point` rejected

**Symptom:** The command fails on Ethernet interfaces in Packet Tracer.

**Impact:** OSPF performs DR/BDR election on the CSW1↔CSW2 Port-channel. With two
routers on the segment, this is harmless.

**Workaround:** Skip in PT. Apply on real gear — it skips the DR/BDR election on
point-to-point transit links.

---

## 11. Common Issues & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| No OSPF neighbors | Interface still in `switchport` mode | `no switchport` on the physical port |
| Neighbor stuck in EXSTART/EXCHANGE | MTU mismatch or duplicate router-ID | `show ip ospf interface` and `show ip ospf` |
| Neighbor stuck in INIT | One side not sending hellos | Verify `no passive-interface` on both ends |
| Routes missing | Network statement wildcard wrong | Wildcard must match subnet size (e.g., `0.0.0.255` for /24) |
| Default route not propagated | `default-information originate` missing on R1 | Add under `router ospf 1` |
| Inter-office ping fails | CSW1↔CSW2 adjacency missing | See §10.1 |

---

## 12. Phase 6 Checkpoint

- [ ] ISP configured with Gi0/0 (203.0.113.2/30) and Loopback0 (8.8.8.8/32)
- [ ] R1 configured with 3 IPs + Loopback0 + static default + OSPF
- [ ] CSW↔DSW uplinks converted to routed ports
- [ ] OSPF enabled on R1, CSW1, CSW2, DSW-A1/A2/B1/B2
- [ ] SVIs and loopbacks passive
- [ ] `show ip ospf neighbor` shows FULL adjacencies on all transit links
- [ ] CSW1 ↔ CSW2 adjacency via Port-channel1 (PT workaround applied)
- [ ] `show ip route` on DSW-A1 shows Office B subnets + default route
- [ ] **PC-A1 ↔ PC-B1 ping succeeds**
- [ ] All devices saved

---

## 13. Next Phase

➡️ **[Phase 7 — ACLs](Phase-07-ACLs.md)**

---

*Phase 6 of the CCNA Master Lab project. See [README](../README.md) for the full phase list.*