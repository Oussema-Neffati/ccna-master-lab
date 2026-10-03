# Phase 11 — Troubleshooting Scenarios + Final Review

> [!NOTE]\n> **Prerequisites & Navigation**
> **Previous:** [Phase 10 — Automation](Phase-10-Automation.md) · **Next:** (end of project)
> Master addressing plan: [MASTER-ADDRESSING.md](../MASTER-ADDRESSING.md) · Final device summary: [FINAL-DEVICE-SUMMARY.md](../FINAL-DEVICE-SUMMARY.md)

> > [!TIP]\n> **Phase Objective**
> Prove diagnostic skill — inject misconfigurations, observe symptoms, diagnose,
> fix, verify. This is the exam-critical skill: not "can you configure it" but
> "can you find what's broken."

> [!WARNING] Save a working snapshot first
> ```cisco
> write memory
> ```
> On every device. PT → File → Save As → `acme-corp-lab-WORKING.pkt`.

> [!TIP] How to use this phase
> Inject **one scenario at a time**. Diagnose, fix, verify, then move on.

---

## Scenario 1 — Native VLAN Mismatch on a Trunk

**CCNA Domain:** VLANs / Trunking

**Injection:**
```cisco
! On DSW-A1
interface FastEthernet0/1
 switchport trunk native vlan 1
 exit
```

**Symptom:** Tagged VLAN traffic keeps working. Untagged behavior is inconsistent. CDP may log "native VLAN mismatch".

**Diagnosis:**
```cisco
show interfaces trunk
show cdp neighbors detail
```

**Fix:**
```cisco
interface FastEthernet0/1
 switchport trunk native vlan 1000
 exit
write memory
```

**Lesson:** Native VLAN mismatch is silent for tagged traffic. It's a security risk.

---

## Scenario 2 — OSPF Passive Interface on a Transit Link

**CCNA Domain:** OSPF / HSRP interaction

**Injection:**
```cisco
! On DSW-A1
router ospf 1
 passive-interface GigabitEthernet0/1
 exit
```

**Symptom:** PC-A1 → PC-A2 works. PC-A1 → PC-B1 fails. `show ip ospf neighbor` on DSW-A1 shows no neighbor for CSW1.

**Diagnosis:**
```cisco
show ip ospf neighbor
show ip ospf interface GigabitEthernet0/1   ! → Passive interface: Yes
show ip route ospf
```

**Fix:**
```cisco
router ospf 1
 no passive-interface GigabitEthernet0/1
 exit
write memory
```

### The Critical Lesson — HSRP Does Not Track OSPF State

PC-A1's traffic goes to the HSRP VIP (10.10.10.1) → DSW-A1 (Active). DSW-A1 has no OSPF route to Office B → traffic dropped. DSW-A1 stays Active because its Vlan10 SVI is still up. DSW-A2 remains Standby and never receives the traffic.

> [!IMPORTANT] HSRP only fails over when a tracked interface goes DOWN
> Routing failures do not trigger HSRP failover unless explicitly tracked. Use
> **HSRP object tracking**:
>
> ```cisco
> ! On DSW-A1
> track 1 interface GigabitEthernet0/1 line-protocol
> interface Vlan10
>  standby 10 track 1 decrement 20
>  exit
> ```
>
> For deeper failure detection (OSPF adjacency loss), use **IP SLA**:
> ```cisco
> ip sla 1
>  icmp-echo 10.0.1.1 source-interface GigabitEthernet0/1
>  frequency 5
>  exit
> ip sla schedule 1 life forever start-time now
> track 1 ip sla 1 reachability
> interface Vlan10
>  standby 10 track 1 decrement 20
>  exit
> ```

**Lesson:** Redundancy protocols are only as good as their tracking. Without tracking, HSRP stays Active on a device that has lost upstream connectivity.

---

## Scenario 3 — EtherChannel Member Shutdown

**CCNA Domain:** EtherChannel / LACP

**Injection:**
```cisco
! On DSW-A1
interface FastEthernet0/3
 shutdown
 exit
```

**Symptom:** PortChannel stays up. `show etherchannel summary` shows Fa0/3 as `(D)`.

**Diagnosis:**
```cisco
show etherchannel summary
show interfaces FastEthernet0/3 status
```

**Fix:**
```cisco
interface FastEthernet0/3
 no shutdown
 exit
write memory
```

**Lesson:** EtherChannel survives losing a member, but you must monitor member state.

---

## Scenario 4 — ACL Blocking Legitimate Traffic

**CCNA Domain:** ACLs

**Injection:**
```cisco
! On DSW-A1 AND DSW-A2
ip access-list extended BLOCK_A_TO_B_SERVERS
 no 10
 no 20
 exit
```

**Symptom:** PC-A1 → PC-B1 (VLAN 10 → VLAN 10) works. PC-A1 → Office B server fails even for HTTP/HTTPS.

**Diagnosis:**
```cisco
show ip access-lists BLOCK_A_TO_B_SERVERS
```
Expected: 4 entries; only 2 remain.

**Fix:**
```cisco
! On DSW-A1 AND DSW-A2
ip access-list extended BLOCK_A_TO_B_SERVERS
 10 permit tcp 10.10.10.0 0.0.0.255 10.20.30.0 0.0.0.255 eq 80
 20 permit tcp 10.10.10.0 0.0.0.255 10.20.30.0 0.0.0.255 eq 443
 exit
write memory
```

**Lesson:** ACLs are order-sensitive. A missing line changes behavior silently.

---

## Scenario 5 — Port Security Err-Disable

**CCNA Domain:** Port Security

**Injection:**
```cisco
! On ASW-A1
interface FastEthernet0/1
 switchport port-security maximum 1
 exit
```
Then connect two devices via a hub.

**Symptom:** PC-A1 loses connectivity. Fa0/1 err-disabled.

**Diagnosis:**
```cisco
show interfaces FastEthernet0/1 status
show port-security interface FastEthernet0/1
```

**Fix:**
```cisco
interface FastEthernet0/1
 shutdown
 no shutdown
 exit
```
Or:
```cisco
clear port-security sticky interface FastEthernet0/1
```

**Lesson:** Port Security is disruptive by design.

---

## Scenario 6 — DHCP Snooping Trust Boundary Broken

**CCNA Domain:** DHCP Snooping

**Injection:**
```cisco
! On ASW-A1
interface range GigabitEthernet0/1 - 2
 no ip dhcp snooping trust
 exit
```

**Symptom:** Renewing clients cannot get IP. New clients fail.

**Diagnosis:**
```cisco
show ip dhcp snooping
show ip dhcp snooping binding
```

**Fix:**
```cisco
interface range GigabitEthernet0/1 - 2
 ip dhcp snooping trust
 exit
write memory
```

**Lesson:** Trust boundaries matter. Uplinks toward DHCP server must be trusted.

---

## Scenario 7 — NAT ACL Wildcard Regression

**CCNA Domain:** NAT / ACLs

**Injection:**
```cisco
! On R1
no access-list 1
access-list 1 permit 10.0.0.0 0.0.255.255
exit
```

**Symptom:** PC-A1 cannot reach 8.8.8.8. `show ip nat statistics` shows `Hits: 0`.

**Diagnosis:**
```cisco
show ip nat statistics
show access-lists 1
```

**Fix:**
```cisco
no access-list 1
access-list 1 remark === NAT: all internal 10/8 networks ===
access-list 1 permit 10.0.0.0 0.255.255.255
exit
write memory
```

**Lesson:** Wildcards are precise. One digit changes everything.

---

## Scenario 8 — STP Root Bridge Hijack

**CCNA Domain:** STP / Root Guard

**Injection:**
```cisco
! On ASW-A1
spanning-tree vlan 10,20,30,99 priority 0
exit
```

**Symptom:** `show spanning-tree vlan 10` on DSW-A1 shows ASW-A1 as root.

**Diagnosis:**
```cisco
! On DSW-A1
show spanning-tree vlan 10
```

```cisco
! On ASW-A1
show spanning-tree vlan 10
```

**Fix:**
```cisco
! On ASW-A1
no spanning-tree vlan 10,20,30,99 priority
exit
```

If Root Guard was missing, add it now:

> [!IMPORTANT] Root Guard placement — on the distribution side
> Root Guard is applied on the **distribution switches' ports facing the access
> layer** — the ports that should never become the STP root port:
>
> ```cisco
> ! On DSW-A1 (ports toward ASWs)
> interface range GigabitEthernet0/2, FastEthernet0/1
>  spanning-tree guard root
>  exit
> ```
>
> If a rogue switch with a lower bridge ID appears downstream, the DSW's port
> enters **root-inconsistent** state.

> [!WARNING] Packet Tracer may not model Root Guard faithfully
> PT often accepts the command but doesn't enforce the root-inconsistent state.
> On real IOS it works as described. Verify with `show spanning-tree inconsistentports`.

**Lesson:** Root Guard is a critical defense. Placement on the **DSW-side** is the Cisco-standard design.

---

## Final Review Checklist

### Layer 1 / Layer 2

- [ ] All physical links green/orange (no red)
- [ ] `show etherchannel summary` on DSW-A1/A2 → `Po1(SU)` with both members `(P)`
- [ ] `show etherchannel summary` on CSW1/CSW2 → `Po1(RU)` with both members `(P)`
- [ ] `show interfaces trunk` on DSW-A1 shows Po1, Gi0/2, Fa0/1 (native VLAN 1000)
- [ ] `show spanning-tree vlan 10` on DSW-A1 → `This bridge is the root`

> [!NOTE] `show interfaces trunk` returns empty on CSW1/CSW2
> All CSW1/CSW2 uplinks are **routed ports** (Phase 6). No trunks exist on the core switches.

### Layer 3

- [ ] `show ip ospf neighbor` on CSW1 → 4 neighbors, all FULL
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

## Packet Tracer Limitations (Consolidated)

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

## Phase 11 Checkpoint

- [ ] All 8 scenarios injected, diagnosed, and fixed
- [ ] Final review checklist fully passed
- [ ] Lab returned to a working state
- [ ] `write memory` on all devices

---

## Next Steps

- **[Final device summary](../FINAL-DEVICE-SUMMARY.md)** — one-page reference for the whole lab
- **[README](../README.md)** — project overview
- **[Master addressing plan](../MASTER-ADDRESSING.md)** — full IPv4 plan

---

*Phase 11 of the CCNA Master Lab project. See [README](../README.md) for the full phase list.*