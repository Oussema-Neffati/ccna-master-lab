---
title: "Phase 7 — Access Control Lists (Standard & Extended)"
project: CCNA Master Lab — Acme Corp Two-Office Enterprise
phase: 7
status: complete
tags:
  - ccna
  - lab
  - acl
  - standard-acl
  - extended-acl
  - traffic-filtering
  - packet-tracer
created: 2026-09-24
updated: 2026-09-24
---

# Phase 7 — Access Control Lists (Standard & Extended)

> [!info] Project Context
> This lab models **Acme Corp**, a two-office enterprise connected through a dual-homed core and a single edge router to a simulated ISP. Each phase builds on the previous one, ending with a fully integrated CCNA lab covering VLANs, STP, EtherChannel, OSPF, ACLs, security hardening, services, and automation.

> [!success] Phase Objective
> Filter traffic at two layers:
> 1. **Standard ACL** — restrict device management (SSH/VTY) to management subnets only.
> 2. **Extended ACL** — block Office A PCs from reaching Office B servers, except HTTP/HTTPS.

> [!important] Scope Rules
> - Standard ACL: applied on **R1, CSW1, CSW2, DSW-A1, DSW-A2, DSW-B1, DSW-B2** (all 7 network devices).
> - Extended ACL: applied on **DSW-A1 AND DSW-A2** (both HSRP peers).

---

## 1. Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Standard ACL for VTY access** | Simplest tool for source-based filtering. Placed close to destination (VTY lines) per standard-ACL rule. |
| **Extended ACL for inter-VLAN filtering** | Can match source, destination, protocol, port — required for "block all except HTTP/HTTPS". |
| **Extended ACL applied inbound on Vlan10 SVI** | Closest to the source. Drops blocked traffic at the first routing hop. |
| **ACL applied on both DSW-A1 and DSW-A2** | HSRP balances hosts across both switches. Filtering must exist on both — otherwise failover bypasses the filter. |
| **No `log` keyword (PT limitation)** | PT does not support `log` on ACL entries. Match counters serve the same purpose. |
| **Explicit `permit ip any any` at end of extended ACL** | Without it, the implicit deny would block all non-listed traffic — including legitimate PC-A1 → PC-B1 traffic. |

---

## 2. ACL Overview

| ACL | Type | Name/Number | Applied On | Direction | Interface |
|-----|------|-------------|------------|-----------|-----------|
| SSH restriction | Standard | 10 | All 7 network devices | in | VTY lines |
| Server block | Extended | BLOCK_A_TO_B_SERVERS | DSW-A1, DSW-A2 | in | Vlan10 SVI |

---

## 3. Standard ACL 10 — VTY / SSH Restriction

Only hosts in the management subnets may SSH into network devices.

```cisco
enable
configure terminal

! Define standard ACL 10
access-list 10 remark === Mgmt networks allowed to SSH ===
access-list 10 permit 10.10.99.0 0.0.0.255
access-list 10 permit 10.20.99.0 0.0.0.255
access-list 10 permit 10.30.0.0 0.0.0.255
access-list 10 deny   any

! Apply to VTY lines
line vty 0 15
 access-class 10 in
 transport input ssh
 exit

end
write memory
```

> [!tip] Standard ACL placement rule
> Standard ACLs match **source only** — they must be placed **close to the destination**. Here, the destination is the device's VTY subsystem, so applying the ACL directly on the VTY lines satisfies the rule.

> [!warning] `log` keyword is unsupported in Packet Tracer
> `access-list 10 deny any log` will be rejected with `% Invalid input detected at '^' marker`. Use `access-list 10 deny any` instead. On real IOS, `log` works and is best practice for auditing.

---

## 4. Extended ACL — Block Office A PCs from Office B Servers

Office A PCs (VLAN 10 = `10.10.10.0/24`) should not reach Office B Servers (VLAN 30 = `10.20.30.0/24`) — **except** HTTP/HTTPS.

### 4.1 The ACL

```cisco
enable
configure terminal

ip access-list extended BLOCK_A_TO_B_SERVERS
 permit tcp 10.10.10.0 0.0.0.255 10.20.30.0 0.0.0.255 eq 80
 permit tcp 10.10.10.0 0.0.0.255 10.20.30.0 0.0.0.255 eq 443
 deny   ip  10.10.10.0 0.0.0.255 10.20.30.0 0.0.0.255
 permit ip  any any
 exit

! Apply inbound on Vlan10 SVI
interface Vlan10
 ip access-group BLOCK_A_TO_B_SERVERS in
 exit

end
write memory
```

### 4.2 Apply on BOTH DSW-A1 AND DSW-A2

Because HSRP load-balances hosts across both switches, the ACL must exist on both — otherwise a host that fails over to the standby switch would bypass the filter.

### 4.3 Entry-by-entry logic

| Seq | Entry | Purpose |
|-----|-------|---------|
| 10 | `permit tcp ... eq 80` | Allow HTTP to Office B servers |
| 20 | `permit tcp ... eq 443` | Allow HTTPS to Office B servers |
| 30 | `deny ip ...` | Block everything else to Office B servers |
| 40 | `permit ip any any` | Allow everything else (e.g., to Office B VLAN 10, Internet, etc.) |

> [!important] Order matters
> The deny line **must come after** the two permit lines (otherwise HTTP/HTTPS would be blocked too) and **before** the final `permit ip any any` (otherwise it would never match).

---

## 5. Verification

### 5.1 Standard ACL 10

```cisco
show access-lists 10
```
Expected: three permits and one deny, no errors.

### 5.2 Extended ACL contents

On **DSW-A1**:
```cisco
show ip access-lists BLOCK_A_TO_B_SERVERS
```
Expected:
```
Extended IP access list BLOCK_A_TO_B_SERVERS
    10 permit tcp 10.10.10.0 0.0.0.255 10.20.30.0 0.0.0.255 eq www
    20 permit tcp 10.10.10.0 0.0.0.255 10.20.30.0 0.0.0.255 eq 443
    30 deny ip 10.10.10.0 0.0.0.255 10.20.30.0 0.0.0.255
    40 permit ip any any
```

### 5.3 Functional tests

**Setup:** Move PC-B2 to VLAN 30 (via ASW-B2 `interface Fa0/1 → switchport access vlan 30`) and set its IP to `10.20.30.20/24`, gateway `10.20.30.1`.

**From PC-A1 (10.10.10.10):**

| Test | Command | Expected | Observed |
|------|---------|----------|----------|
| Ping Office B Server | `ping 10.20.30.20` | ❌ Blocked | Destination host unreachable |
| Ping Office B PC | `ping 10.20.10.10` | ✅ Allowed | Reply, 0% loss |
| HTTP to server | Web browser → `http://10.20.30.20` | ✅ Allowed | TCP handshake succeeds |
| HTTPS to server | Web browser → `https://10.20.30.20` | ✅ Allowed | TCP handshake succeeds |

**Deny counter proof:**
```cisco
show ip access-lists BLOCK_A_TO_B_SERVERS
```
The deny entry should show a non-zero match count incrementing with each blocked ping.

**Observed after testing:**
```
deny ip 10.10.10.0 0.0.0.255 10.20.30.0 0.0.0.255 (4 match(es))
permit ip any any (324 match(es))
```

> [!tip] Match counters are the source of truth
> If `show ip access-lists` shows counters incrementing, the ACL is working — regardless of what `show ip interface` reports.

### 5.4 VTY ACL test

**From allowed mgmt host:**
- Configure PC-A1 IP temporarily to `10.10.99.10/24`, gateway `10.10.99.1`.
- `ssh -l admin 10.10.10.2` → reaches authentication prompt (ACL permits).

**From disallowed host:**
- Revert PC-A1 to `10.10.10.10/24`.
- `ssh -l admin 10.10.10.2` → connection refused/timed out (ACL denies).

---

## 6. Phase 7 Checkpoint

- [x] Standard ACL 10 defined on all 7 network devices
- [x] ACL 10 applied to VTY lines with `access-class 10 in`
- [x] Extended ACL `BLOCK_A_TO_B_SERVERS` on DSW-A1 **and** DSW-A2
- [x] Extended ACL applied inbound on Vlan10 SVI
- [x] PC-A1 → PC-B1 (inter-office, VLAN 10) ping **works**
- [x] PC-A1 → Office B Server ping **blocked** (deny counter increments)
- [x] PC-A1 → Office B Server HTTP/HTTPS **allowed** at TCP layer
- [x] SSH from mgmt subnet **allowed**, from user subnet **blocked**
- [x] All devices saved to startup-config

---

## 7. Troubleshooting Encounters

### 7.1 `log` keyword rejected

**Symptom:** `access-list 10 deny any log` and `deny ip ... log` return `% Invalid input detected`.

**Cause:** Packet Tracer does not support the `log` keyword on ACL entries.

**Fix:** Remove `log`. Use `show access-lists` counters for auditing instead.

> [!note] On real Cisco IOS
> `log` works and generates syslog messages for each matching packet. Always apply it in production.

---

### 7.2 Missing deny line in extended ACL

**Symptom:** `show ip access-lists BLOCK_A_TO_B_SERVERS` shows only three entries — the `deny ip` line is missing. Result: `permit ip any any` matches everything and Office A traffic is NOT blocked.

**Cause:** The `deny ip` line was entered with the `log` keyword (rejected by PT), and PT did not insert it. Because PT appends new entries at the end by default, adding the deny line later would place it **after** `permit ip any any` — where it would never match.

**Fix:** Delete and recreate the entire ACL without `log`:

```cisco
no ip access-list extended BLOCK_A_TO_B_SERVERS

ip access-list extended BLOCK_A_TO_B_SERVERS
 permit tcp 10.10.10.0 0.0.0.255 10.20.30.0 0.0.0.255 eq 80
 permit tcp 10.10.10.0 0.0.0.255 10.20.30.0 0.0.0.255 eq 443
 deny   ip  10.10.10.0 0.0.0.255 10.20.30.0 0.0.0.255
 permit ip  any any

interface Vlan10
 ip access-group BLOCK_A_TO_B_SERVERS in
```

---

### 7.3 `show ip interface Vlan10` reports "Inbound access list is not set"

**Symptom:** After applying `ip access-group BLOCK_A_TO_B_SERVERS in` on `interface Vlan10`, the command `show ip interface Vlan10 | include access list` reports:

```
Inbound access list is not set
```

...despite:
- `show running-config interface Vlan10` showing the ACL bound
- `show ip access-lists` showing match counters incrementing
- Traffic being filtered correctly

**Cause:** Packet Tracer display bug for SVI ACL bindings. The ACL is applied and functioning; the command output is wrong.

**Fix:** Ignore the display. Verify with:
```cisco
show ip access-lists BLOCK_A_TO_B_SERVERS
```
If counters increment when traffic is sent, the ACL is working.

> [!warning] This is a display bug only
> On real Cisco IOS, `show ip interface Vlan10` correctly reports the applied ACL. In Packet Tracer, trust `show running-config` and match counters.

---

## 8. Common Issues & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| ACL applied but traffic still flows | ACL applied on wrong interface or direction | Verify with `show running-config interface` |
| PC-A1 can't reach PC-B1 either | Missing `permit ip any any` at the end | Add it |
| VTY blocks all SSH including mgmt | Wrong wildcard mask | Use `0.0.0.255` for a /24 |
| SSH still works from disallowed host | ACL not applied or wrong direction | `access-class 10 in`, not out |
| HTTP to server still blocked | Missing `eq 80` / `eq 443` | Check protocol `tcp`, not `ip` |
| Deny line never matches | Deny placed after `permit ip any any` | Delete + recreate ACL in correct order |
| ACL matches increment but traffic flows | ACL applied on only one DSW | Apply on both DSW-A1 and DSW-A2 |

---

## 9. Lessons Learned

> [!note] ACL ordering is critical
> Entries are evaluated top-down. The first match wins. Placing a permit before a deny (or vice versa) can silently invert the intent.

> [!note] Standard vs. Extended placement
> - **Standard ACLs**: place close to destination (match source only).
> - **Extended ACLs**: place close to source (match source + destination + protocol).

> [!note] HSRP + ACL = apply on both peers
> Any filter that must survive failover has to be on both HSRP switches.

> [!note] Packet Tracer quirks to remember
> - `log` keyword → not supported
> - `show ip interface` on SVIs → does not report ACL bindings reliably
> - Deleting an ACL sometimes silently unbinds it from the interface — always reapply after deletion

---

## 10. Next Phase

➡️ **[Phase 8 — Network Security (Port Security, DHCP Snooping, DAI)](Phase-8-Network-Security.md)**

In Phase 8 we will:
- Enable Port Security with sticky MAC and violation shutdown on access ports
- Enable DHCP Snooping with trusted uplinks and rate-limited untrusted ports
- Enable Dynamic ARP Inspection with additional validation checks
- Verify all three security features work as intended

---

*Document maintained as part of the CCNA Master Lab project. For questions, corrections, or contributions, see the repository README.*