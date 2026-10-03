# Phase 7 — Access Control Lists (Standard & Extended)

> [!NOTE] Prerequisites & Navigation
> **Previous:** [Phase 6 — OSPFv2](Phase-06-OSPFv2.md) · **Next:** [Phase 8 — Network Security](Phase-08-Network-Security.md)
> Master addressing plan: [MASTER-ADDRESSING.md](../MASTER-ADDRESSING.md)

> [!TIP] Phase Objective
> Filter traffic at two layers:
> 1. **Standard ACL** — restrict device management (SSH/VTY) to management subnets.
> 2. **Extended ACL** — block Office A PCs from reaching Office B servers, except HTTP/HTTPS.

> [!IMPORTANT] Scope — which devices get which ACL
> - **Standard ACL 10 (VTY restriction):** R1, CSW1, CSW2, DSW-A1, DSW-A2, DSW-B1, DSW-B2 (7 devices).
>   ASWs are excluded — they are Layer 2 only, no VTY exposure outside their own VLAN 99 management segment.
> - **Extended ACL `BLOCK_A_TO_B_SERVERS`:** DSW-A1 **and** DSW-A2 (both HSRP peers).

---

## 1. Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Standard ACL for VTY access** | Simplest tool for source-based filtering. |
| **Extended ACL for inter-VLAN filtering** | Can match source, destination, protocol, port. |
| **Extended ACL applied inbound on Vlan10 SVI** | Closest to the source — drops blocked traffic at the first routing hop. |
| **ACL applied on both DSW-A1 and DSW-A2** | So the filter survives an HSRP failover from Active to Standby. |
| **No `log` keyword (PT limitation)** | PT rejects `log` on ACL entries. Match counters serve the same purpose. |
| **Explicit `permit ip any any` at end of extended ACL** | Without it, the implicit deny blocks all non-listed traffic — including PC-A1 → PC-B1. |

> [!NOTE] HSRP Active/Standby, not load balancing
> In this design DSW-A1 is the Active gateway for all Office A VLANs (priority 110).
> DSW-A2 is Standby. Traffic is **not** balanced across them. The reason for
> duplicating the ACL on both switches is **failover** — if DSW-A1 loses Active
> status, traffic flows through DSW-A2 and the filter must still apply.

---

## 2. ACL Overview

| ACL | Type | Name/Number | Applied On | Direction | Interface |
|-----|------|-------------|------------|-----------|-----------|
| SSH restriction | Standard | 10 | 7 network devices | in | VTY lines |
| Server block | Extended | `BLOCK_A_TO_B_SERVERS` | DSW-A1, DSW-A2 | in | Vlan10 SVI |

---

## 3. Standard ACL 10 — VTY / SSH Restriction

### On R1, CSW1, CSW2, DSW-A1, DSW-A2, DSW-B1, DSW-B2

```cisco
enable
configure terminal

access-list 10 remark === Mgmt networks allowed to SSH ===
access-list 10 permit 10.10.99.0 0.0.0.255
access-list 10 permit 10.20.99.0 0.0.0.255
access-list 10 permit 10.30.0.0 0.0.0.255
access-list 10 deny   any

line vty 0 15
 access-class 10 in
 transport input ssh
 exit

end
write memory
```

> [!WARNING] `log` keyword unsupported in Packet Tracer
> `access-list 10 deny any log` is rejected with `% Invalid input detected at '^' marker`.
> Use `access-list 10 deny any`. On real IOS, `log` works and is best practice.

> [!NOTE] Standard ACL placement rule
> Standard ACLs match **source only** and are normally placed **close to the destination**
> because they lack destination information. When applied as a VTY `access-class`, the
> rule applies implicitly — the destination *is* the device's own VTY subsystem.

---

## 4. Extended ACL — Block Office A PCs from Office B Servers

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

interface Vlan10
 ip access-group BLOCK_A_TO_B_SERVERS in
 exit

end
write memory
```

### 4.2 Apply on **both** DSW-A1 AND DSW-A2

Because HSRP failover shifts traffic from DSW-A1 to DSW-A2, the filter must exist
on both. Otherwise a host that fails over to the Standby switch bypasses the
filter entirely.

### 4.3 Entry-by-entry logic

| Seq | Entry | Purpose |
|-----|-------|---------|
| 10 | `permit tcp ... eq 80` | Allow HTTP to Office B servers |
| 20 | `permit tcp ... eq 443` | Allow HTTPS to Office B servers |
| 30 | `deny ip ...` | Block everything else to Office B servers |
| 40 | `permit ip any any` | Allow all other traffic (VLAN 10, Internet, etc.) |

> [!IMPORTANT] Order matters
> The two `permit tcp` lines must come **before** the `deny ip`, and the final
> `permit ip any any` must come **last**. First match wins.

---

## 5. Verification

### 5.1 Standard ACL 10

```cisco
show access-lists 10
```
Expected: three permits and one deny.

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

**Setup:** Move PC-B2 to VLAN 30 (via ASW-B2 `interface Fa0/1 → switchport access vlan 30`)
and set its IP to `10.20.30.20/24`, gateway `10.20.30.1`.

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

### 5.4 VTY ACL test

> [!IMPORTANT] SSH keys are not yet created at this phase
> Full SSH configuration (RSA keys, local user `admin`) is completed in **Phase 9**.
> At Phase 7 you can test the ACL **behavior** with a Telnet attempt (which will
> also fail because `transport input ssh` is set), or **defer the full SSH test to
> Phase 9**. This doc defers it — the actual SSH login test lives in Phase 9 §6.5.

**ACL behavior test at Phase 7** — from a disallowed host:

Set **PC-A1** to `10.10.10.10/24` (its normal VLAN 10 IP, gateway `10.10.10.1`):

```
telnet 10.10.10.2
```

Expected: `Connection refused` / `Destination unreachable` — ACL 10 denies the
source, and `transport input ssh` blocks Telnet anyway. Either way, the ACL's
`deny any` counter increments.

On **DSW-A1**:
```cisco
show access-lists 10
```
The `deny any` line should show matches incrementing.

### 5.5 Test from an allowed host (deferred to Phase 9)

The full "log in successfully from a management host" test requires:
- Moving a PC to a **VLAN 99 port** (not just changing its IP — the port must also
  be in VLAN 99), or using **SRV1** at `10.30.0.10` (already on VLAN 300, which is
  permitted by ACL 10).
- SSH keys and the `admin` user (Phase 9).

See **Phase 9 §6.5** for the real SSH login test.

> [!WARNING] Just changing a PC's IP does not move it to VLAN 99
> In earlier drafts, the test said "set PC-A1 to `10.10.99.10/24`". That is
> insufficient — the PC's access port must be reassigned to VLAN 99 as well:
> ```cisco
> ! On the ASW the PC is connected to
> interface FastEthernet0/1
>  switchport access vlan 99
>  exit
> ```
> Otherwise the PC's frames are dropped at the uplink (VLAN 10 is the port's
> PVID). Use SRV1 (`10.30.0.10`) instead to avoid reconfiguring access ports.

---

## 6. Phase 7 Checkpoint

- [ ] Standard ACL 10 defined on all **7** network devices
- [ ] ACL 10 applied to VTY lines with `access-class 10 in`
- [ ] Extended ACL `BLOCK_A_TO_B_SERVERS` on DSW-A1 **and** DSW-A2
- [ ] Extended ACL applied inbound on Vlan10 SVI
- [ ] PC-A1 → PC-B1 (inter-office, VLAN 10) ping **works**
- [ ] PC-A1 → Office B Server ping **blocked** (deny counter increments)
- [ ] PC-A1 → Office B Server HTTP/HTTPS **allowed** at TCP layer
- [ ] Telnet/SSH from PC-A1 to DSW-A1 **denied** at ACL 10
- [ ] All devices saved to startup-config

---

## 7. Troubleshooting Encounters

### 7.1 `log` keyword rejected

**Symptom:** `access-list 10 deny any log` and `deny ip ... log` return `% Invalid input detected`.

**Cause:** Packet Tracer does not support the `log` keyword on ACL entries.

**Fix:** Remove `log`. Use `show access-lists` counters for auditing instead.

### 7.2 Missing deny line in extended ACL

**Symptom:** `show ip access-lists BLOCK_A_TO_B_SERVERS` shows only three entries — the `deny ip` line is missing.

**Cause:** The `deny ip` line was entered with the `log` keyword (rejected by PT). Because PT appends new entries at the end by default, adding it later would place it **after** `permit ip any any` — where it would never match.

**Fix:** Delete and recreate the entire ACL without `log`.

### 7.3 `show ip interface Vlan10` reports "Inbound access list is not set"

**Symptom:** After applying `ip access-group BLOCK_A_TO_B_SERVERS in` on `interface Vlan10`, `show ip interface Vlan10` reports the ACL is not set — despite `show running-config` showing it and `show ip access-lists` counters incrementing.

**Cause:** Packet Tracer display bug for SVI ACL bindings.

**Workaround:** Ignore the display. Verify with:
```cisco
show ip access-lists BLOCK_A_TO_B_SERVERS
```
If counters increment, the ACL is working.

---

## 8. Common Issues & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| ACL applied but traffic still flows | Applied on wrong interface or direction | Verify with `show running-config interface` |
| PC-A1 can't reach PC-B1 either | Missing `permit ip any any` at the end | Add it |
| VTY blocks all SSH including mgmt | Wrong wildcard mask | Use `0.0.0.255` for a /24 |
| SSH still works from disallowed host | ACL not applied or wrong direction | `access-class 10 in`, not out |
| HTTP to server still blocked | Missing `eq 80` / `eq 443` | Use `tcp`, not `ip` |
| Deny line never matches | Deny placed after `permit ip any any` | Delete + recreate ACL in correct order |
| ACL matches increment but traffic flows | Applied on only one DSW | Apply on both DSW-A1 and DSW-A2 |

---

## 9. Lessons Learned

> [!NOTE] ACL ordering is critical
> Entries are evaluated top-down. First match wins. A misplaced permit can silently invert the intent.

> [!NOTE] Standard vs. Extended placement
> - **Standard ACLs:** place close to destination.
> - **Extended ACLs:** place close to source.

> [!NOTE] HSRP + ACL = apply on both peers
> Any filter that must survive failover has to be on both HSRP switches.

> [!NOTE] Packet Tracer quirks to remember
> `log` unsupported; `show ip interface` unreliable for SVI ACLs; deleting and
> recreating an ACL sometimes silently unbinds it — always reapply.

---

## 10. Next Phase

➡️ **[Phase 8 — Network Security](Phase-08-Network-Security.md)**

---

*Phase 7 of the CCNA Master Lab project. See [README](../README.md) for the full phase list.*