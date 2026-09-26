---
title: "Phase 9 — Network Services (NAT, NTP, SNMP, SSH)"
project: CCNA Master Lab — Acme Corp Two-Office Enterprise
phase: 9
status: complete
tags:
  - ccna
  - lab
  - nat
  - pat
  - ntp
  - snmp
  - ssh
  - packet-tracer
created: 2026-09-26
updated: 2026-09-26
---

# Phase 9 — Network Services (NAT, NTP, SNMP, SSH)

> [!info] Project Context
> This lab models **Acme Corp**, a two-office enterprise connected through a dual-homed core and a single edge router to a simulated ISP. Each phase builds on the previous one, ending with a fully integrated CCNA lab covering VLANs, STP, EtherChannel, OSPF, ACLs, security hardening, services, and automation.

> [!success] Phase Objective
> Complete the services layer:
> 1. **NAT/PAT** on R1 — private networks reach the Internet
> 2. **NTP** — R1 is master, all devices sync to it
> 3. **SNMPv2c** — read-only community on all network devices
> 4. **SSH** — complete hardening on all 11 network devices

---

## 1. Design Decisions

| Decision | Rationale |
|----------|-----------|
| **NAT ACL covering all 10/8 (`10.0.0.0 0.255.255.255`)** | Simpler and covers every internal subnet. |
| **PAT (overload) on Gi0/1** | Multiple inside hosts share one public IP. Standard for small/medium offices. |
| **R1 as NTP stratum-5 master** | Lab stand-in for a real time source. Easy to verify. |
| **SSHv2 only with `login local`** | No Telnet, no password-only auth. Local user database. |
| **SNMPv2c RO/RW communities** | Simplest SNMP implementation PT supports. |
| **Two NAT inside interfaces** | R1 has dual uplinks (Gi0/0 to CSW1, Gi0/2 to CSW2) — both must be `ip nat inside`. |

---

## 2. SRV1 — Services Host

SRV1 provides DNS, Syslog, HTTP, and (where PT permits) NTP and SNMP.

### Configuration

| Service | Setting |
|---------|---------|
| **Static IP** | 10.30.0.10/24, gateway 10.30.0.1 |
| **DNS** | On — records: `r1.acme.local→10.0.0.1`, `srv1.acme.local→10.30.0.10`, `internet.test→8.8.8.8` |
| **Syslog** | On |
| **HTTP** | On |
| **SNMP** | **Not supported** in PT Server-PT (see §10.1) |
| **NTP** | GUI too limited in PT (see §10.3) |

> [!warning] Packet Tracer limitations on Server-PT
> - **SNMP service is missing** from the Services tab in most PT builds — SRV1 cannot act as an SNMP trap receiver.
> - **NTP service** only exposes Key / Password / Calendar fields — no server address field. SRV1 cannot be configured as an NTP client through the GUI.
>
> Both services work on real Cisco network gear; only the **Server-PT** emulation is limited.

---

## 3. NAT/PAT on R1

### 3.1 Mark inside and outside interfaces

```cisco
enable
configure terminal

interface GigabitEthernet0/0
 description To CSW1
 ip nat inside
 exit

interface GigabitEthernet0/2
 description To CSW2
 ip nat inside
 exit

interface GigabitEthernet0/1
 description To ISP
 ip nat outside
 exit
```

> [!important] Both uplinks toward the internal network are `inside`
> R1 has two paths into the enterprise (Gi0/0 → CSW1, Gi0/2 → CSW2). Both must be `ip nat inside`. Only Gi0/1 (to ISP) is `ip nat outside`.

### 3.2 NAT ACL and PAT

```cisco
! ACL matching all internal 10/8 networks
access-list 1 remark === NAT: all internal 10/8 networks ===
access-list 1 permit 10.0.0.0 0.255.255.255

! PAT overload using the outside interface IP
ip nat inside source list 1 interface GigabitEthernet0/1 overload

end
write memory
```

> [!warning] The wildcard `0.255.255.255` is critical
> A common mistake is typing `0.0.255.255` (which matches only `10.0.x.x`). The correct wildcard for "anything starting with 10" is `0.255.255.255`. See §10.5 for the troubleshooting story.

### 3.3 Verify

From **PC-A1** (or PC-B1):
```
ping 8.8.8.8
```

On **R1**:
```cisco
show ip nat translations
```
Expected:
```
Pro  Inside global      Inside local       Outside local      Outside global
icmp 203.0.113.1:1      10.10.10.10:1      8.8.8.8:1          8.8.8.8:1
```

```cisco
show ip nat statistics
```
Expected:
```
Total translations: X (0 static, X dynamic, 0 extended)
Outside Interfaces: GigabitEthernet0/1
Inside Interfaces: GigabitEthernet0/0 , GigabitEthernet0/2
Hits: X  Misses: Y
```

`Hits` must be greater than 0.

From **ISP**:
```cisco
ping 203.0.113.1
```
Should succeed — R1's outside interface is reachable.

---

## 4. NTP — R1 as Master

### 4.1 Configure R1 as NTP master

```cisco
enable
configure terminal

ntp master 5
clock timezone UTC 0
exit

end
write memory
```

> [!note] `clock update-calendar` not supported in PT
> On real Cisco IOS, `clock update-calendar` syncs the hardware calendar chip with the software clock. PT does not support this command — skip it. The software clock (`show clock`) is what matters for the lab.

### 4.2 Configure NTP clients

On **CSW1, CSW2, DSW-A1, DSW-A2, DSW-B1, DSW-B2, ASW-A1, ASW-A2, ASW-B1, ASW-B2**:

```cisco
enable
configure terminal

ntp server 10.255.255.254
clock timezone UTC 0
exit

end
write memory
```

> [!note] SRV1 cannot be NTP-configured in PT
> PT's Server-PT NTP service does not expose a server address field. Set SRV1's Calendar manually to match R1 (`show clock` on R1) if you want visual consistency.

### 4.3 Verify

On any client (e.g., CSW1):
```cisco
show ntp status
```
Expected: `Clock is synchronized, stratum 6, reference is 10.255.255.254`.

```cisco
show ntp associations
```
Expected: R1 listed with `*` as the selected source.

```cisco
show clock
```
Expected: same time as R1.

> [!warning] NTP sync can take 3–5 minutes in PT
> If `show ntp status` reports unsynchronized, verify reachability first (`ping 10.255.255.254`), then wait.

---

## 5. SNMPv2c

### 5.1 Configure what PT accepts

On **R1, CSW1, CSW2, DSW-A1, DSW-A2, DSW-B1, DSW-B2, ASW-A1, ASW-A2, ASW-B1, ASW-B2**:

```cisco
enable
configure terminal

snmp-server community ACME-RO ro
snmp-server community ACME-RW rw

end
write memory
```

### 5.2 Optional commands (PT may reject some)

```cisco
snmp-server host 10.30.0.10 ACME-RO
snmp-server enable traps snmp linkdown linkup
snmp-server trap-source Loopback0
snmp-server location "Acme Corp - Lab"
snmp-server contact "netops@acme.local"
```

> [!warning] Packet Tracer SNMP support is minimal
> PT typically accepts only the two `community` commands. The other commands (`host`, `enable traps`, `trap-source`, `location`, `contact`) are often rejected. See §10.2 and §10.4.

### 5.3 Verify

On any device:
```cisco
show snmp community
```
Expected:
```
Community name: ACME-RO
Community access: Read only
Community name: ACME-RW
Community access: Read write
```

---

## 6. SSH Hardening

### 6.1 Block 1 — Pre-SSH config

Run this **first** on each of the 11 network devices:

```cisco
enable
configure terminal
ip domain-name acme.local
username admin privilege 15 secret Cisco123!
exit
write memory
```

### 6.2 Block 2 — Generate RSA keys (interactive)

Run this command **alone** (do not paste it with anything else):

```
crypto key generate rsa
```

When prompted:
```
How many bits in the modulus [512]:
```

Enter:
```
2048
```

Wait 10–30 seconds for the "keys generated" message.

> [!warning] Packet Tracer requires interactive key generation
> `crypto key generate rsa modulus 2048` is rejected in PT — the modulus must be supplied via the interactive prompt.
>
> **On 2960 switches**, if 2048 fails, retry with `1024` (PT emulation limit).

### 6.3 Block 3 — SSH & VTY hardening

Run this on each device **after** keys are generated:

```cisco
enable
configure terminal

ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3

line vty 0 15
 login local
 transport input ssh
 exec-timeout 10 0
 exit

end
write memory
```

> [!note] ACL 10 is preserved
> `access-class 10 in` was applied to VTY lines in Phase 7. Do not remove it.

### 6.4 Verify

On any device:
```cisco
show ip ssh
```
Expected:
```
SSH Enabled - version 2.0
Authentication timeout: 60 secs; Authentication retries: 3
```

```cisco
show crypto key mypubkey rsa
```
Expected: the public key.

### 6.5 Live SSH Test

**From an allowed mgmt host** — set a PC (temporarily) to `10.10.99.10/24` gateway `10.10.99.1`, or use **SRV1** (`10.30.0.10`) which is permitted by ACL 10.

```
ssh -l admin 10.10.10.2
```

Enter password `Cisco123!` → reach the `DSW-A1#` prompt. ✅

**From a disallowed host** — use **PC-A1** (e.g., `10.10.10.10`).

```
ssh -l admin 10.10.10.2
```

Expected: connection refused/timed out — blocked by ACL 10. ❌

**Observed during the lab:**
- SSH from **SRV1** (10.30.0.10) → ✅ succeeded
- SSH from **PC-A1** (10.10.10.x) → ❌ denied by ACL 10

This confirms both SSH hardening AND Phase 7 ACL work end-to-end.

---

## 7. Save

On **all 11 network devices** plus **SRV1** and **ISP**:
```cisco
write memory
```

---

## 8. Comprehensive Verification

### 8.1 End-to-End Connectivity

| From | To | Expected | Verified |
|------|----|----------|----------|
| PC-A1 | PC-A2 | ✅ | ✅ |
| PC-A1 | PC-B1 | ✅ | ✅ |
| PC-A1 | SRV1 (10.30.0.10) | ✅ | ✅ |
| PC-A1 | Internet (8.8.8.8) | ✅ | ✅ |
| PC-B1 | Internet (8.8.8.8) | ✅ | ✅ |
| PC-A1 | 10.20.30.20 (Office B server) | ❌ (Phase 7 ACL) | ✅ |

### 8.2 Services Verification

| Service | Command | Device | Expected |
|---------|---------|--------|----------|
| NAT | `show ip nat translations` | R1 | Active ICMP entries |
| NAT | `show ip nat statistics` | R1 | Hits > 0 |
| NTP | `show ntp status` | Any switch | synchronized, stratum 6, ref 10.255.255.254 |
| SNMP | `show snmp community` | Any device | ACME-RO (ro), ACME-RW (rw) |
| SSH | `show ip ssh` | Any device | version 2.0 |
| SSH | `ssh` from mgmt host | SRV1 → DSW-A1 | ✅ allowed |
| SSH | `ssh` from user host | PC-A1 → DSW-A1 | ❌ denied |
| DHCP | `show ip dhcp binding` | R1 | Leases for all clients |

### 8.3 Phase 7 ACL Still Active

On **DSW-A1**:
```cisco
show ip access-lists BLOCK_A_TO_B_SERVERS
```
The deny counter should still increment when PC-A1 pings PC-B2.

---

## 9. Phase 9 Checkpoint

- [x] SRV1 configured with DNS, Syslog, HTTP
- [x] R1: `ip nat inside` on Gi0/0 and Gi0/2, `ip nat outside` on Gi0/1
- [x] NAT ACL 1 = `permit 10.0.0.0 0.255.255.255` (corrected wildcard)
- [x] `ip nat inside source list 1 interface Gi0/1 overload` applied
- [x] PC-A1 and PC-B1 can ping 8.8.8.8
- [x] `show ip nat translations` on R1 shows active entries
- [x] R1 configured as `ntp master 5`
- [x] All devices configured with `ntp server 10.255.255.254`
- [x] `show ntp status` on clients shows `synchronized`
- [x] SNMP community strings on all 11 network devices
- [x] SSH version 2 enabled, RSA keys generated on all 11 devices
- [x] `username admin privilege 15 secret Cisco123!` on all devices
- [x] VTY uses `login local` + `transport input ssh`
- [x] SSH from SRV1 (mgmt) works, from PC-A1 (user) blocked
- [x] All devices saved to startup-config

---

## 10. Troubleshooting Encounters

### 10.1 SNMP service missing from Server-PT

**Symptom:** SRV1's Services tab does not include SNMP.

**Cause:** Packet Tracer's Server-PT emulation doesn't provide an SNMP agent.

**Impact:** SRV1 cannot act as an SNMP trap receiver in this lab.

**Workaround:** Configure SNMP community strings on the network devices and skip trap-receiver verification. On real IOS, configure SRV1 (or a Linux host) with `snmpd` and point traps at it.

---

### 10.2 SNMP commands rejected beyond community strings

**Symptom:** `snmp-server host`, `snmp-server enable traps`, `snmp-server trap-source`, `snmp-server location`, `snmp-server contact` are all rejected by PT.

**Cause:** PT supports only the bare minimum SNMP configuration.

**Workaround:** Accept the community strings only. Document the intent — on real IOS, all those commands work.

---

### 10.3 SRV1 NTP service has no server address field

**Symptom:** SRV1's Services → NTP only shows Key, Password, and Calendar fields — no way to specify the upstream NTP server address.

**Cause:** PT Server-PT NTP emulation is incomplete.

**Workaround:** Skip SRV1 NTP. Set the Calendar manually to match R1 if visual consistency matters. On real gear, use `ntp server 10.255.255.254` on the server OS.

---

### 10.4 `clock update-calendar` rejected

**Symptom:** PT rejects `clock update-calendar` with `% Invalid input detected`.

**Cause:** PT does not emulate the hardware calendar chip.

**Workaround:** Skip in PT. On real gear, `clock update-calendar` syncs the hardware calendar with the software clock.

---

### 10.5 NAT ACL wildcard bug — the important one

**Symptom:** After configuring NAT, `show ip nat statistics` on R1 showed:
```
Total translations: 0 (0 static, 0 dynamic, 0 extended)
Hits: 0  Misses: 43
```
`show ip nat translations` was empty — NAT was matching nothing.

**Root Cause:** The ACL was configured as:
```
access-list 1 permit 10.0.0.0 0.0.255.255
```
This wildcard (`0.0.255.255`) matches only `10.0.x.x` — i.e., the `10.0.0.0/16` block.

The lab's internal networks are:
- 10.10.10.0/24, 10.10.20.0/24, 10.10.30.0/24, 10.10.99.0/24
- 10.20.10.0/24, 10.20.20.0/24, 10.20.30.0/24, 10.20.99.0/24
- 10.30.0.0/24

**None of these fall inside `10.0.0.0/16`.** Every packet missed the ACL, so nothing was translated. The `Misses: 43` counter confirmed every packet was checked and rejected.

**Fix:**
```cisco
no access-list 1
access-list 1 remark === NAT: all internal 10/8 networks ===
access-list 1 permit 10.0.0.0 0.255.255.255
```

> [!warning] Wildcard math — the classic CCNA trap
> `0.255.255.255` = first octet `10`, remaining octets don't care → matches every `10.x.x.x` address.
> `0.0.255.255` = first two octets `10.0`, remaining don't care → matches only `10.0.x.x`.
> A single misplaced digit in the wildcard silently breaks NAT for the entire network.

**Verification after fix:** `show ip nat translations` shows ICMP entries, `Hits` > 0.

---

### 10.6 `crypto key generate rsa modulus 2048` rejected

**Symptom:** PT rejects the inline modulus specification.

**Cause:** PT requires interactive input for the RSA modulus.

**Workaround:** Run `crypto key generate rsa` alone, then answer the prompt with `2048`. On 2960 switches, if 2048 is rejected, use `1024`.

---

## 11. Common Issues & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Ping 8.8.8.8 fails | NAT ACL wildcard wrong | `permit 10.0.0.0 0.255.255.255` |
| NAT shows 0 translations | ACL missing or wrong | `show access-lists 1`, clear with `clear ip nat translation *` |
| NAT inside/outside reversed | One interface marked wrong | Verify with `show ip nat statistics` |
| NTP unsynchronized | Client can't reach R1's loopback | Verify OSPF route to `10.255.255.254` |
| NTP takes too long | Normal (3–5 min) | Wait |
| SNMP traps not arriving | PT Server-PT lacks SNMP service | Skip; document intent |
| SSH key generation fails | Inline modulus rejected | Use interactive prompt |
| SSH refused from mgmt host | ACL 10 doesn't include that subnet | Add mgmt subnet to ACL 10 |
| SSH works but only password prompt | Missing `login local` or username | Reconfigure VTY line |

---

## 12. Lessons Learned

> [!note] Wildcards are the #1 NAT/ACL bug source
> Always double-check the wildcard mask. `0.0.255.255` and `0.255.255.255` differ by exactly one character and produce completely different results.

> [!note] Packet Tracer vs. real IOS — know the differences
> Nine PT limitations now documented across phases. Every one of them is valid IOS syntax that PT can't emulate. On real gear, use the commands as written in the CCNA course.

> [!note] SSH end-to-end test proves two things at once
> Testing SSH from SRV1 (allowed) and PC-A1 (denied) verifies both Phase 9 SSH configuration AND Phase 7 ACL 10 in a single test.

---

## 13. Next Phase

➡️ **[Phase 10 — Basic Automation & Programmability](Phase-10-Automation.md)**

In Phase 10 we will:
- Attempt RESTCONF on R1 (`ip http server` + `ip http secure-server` + `restconf`)
- Document PT's RESTCONF support (limited or absent)
- Write a Python script using `requests` to demonstrate the concept
- Discuss Ansible / NETCONF / YANG as alternative automation tools

---

*Document maintained as part of the CCNA Master Lab project. For questions, corrections, or contributions, see the repository README.*