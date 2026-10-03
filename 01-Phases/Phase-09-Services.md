# Phase 9 — Network Services (NAT, NTP, SNMP, SSH)

> [!NOTE]
> **Prerequisites & Navigation**
> **Previous:** [Phase 8 — Network Security](Phase-08-Network-Security.md) · **Next:** [Phase 10 — Automation](Phase-10-Automation.md)
> Master addressing plan: [MASTER-ADDRESSING.md](../MASTER-ADDRESSING.md)

> [!TIP]
> **Phase Objective**
> Complete the services layer:
> 1. **NAT/PAT** on R1 — private networks reach the Internet
> 2. **NTP** — R1 is master, switches sync to it
> 3. **SNMPv2c** — community strings on all network devices (PT-supported subset)
> 4. **SSH** — full hardening on all 11 network devices

> [!IMPORTANT]
> Lab credentials
> All passwords in this phase use the placeholder `<LAB_PASSWORD>`.
> In Packet Tracer, substitute a temporary value such as `LabOnly123!`.
> **Never use these credentials on real equipment or a public network.**

---

## 1. Design Decisions

| Decision | Rationale |
|----------|-----------|
| **NAT ACL covering all 10/8 (`10.0.0.0 0.255.255.255`)** | Simple, covers every internal subnet. |
| **PAT (overload) on Gi0/1** | Multiple inside hosts share one public IP. Standard for small/medium offices. |
| **R1 as NTP stratum-5 master** | Lab stand-in for a real time source. |
| **SSHv2 only with `login local`** | No Telnet, no password-only auth. Local user database. |
| **SNMPv2c RO/RW communities** | Simplest SNMP implementation PT supports. |
| **Two NAT inside interfaces** | R1 has dual uplinks (Gi0/0, Gi0/2) — both must be `ip nat inside`. |

---

## 2. SRV1 — Services Host

SRV1 provides DNS, Syslog, and HTTP.

| Service | Setting |
|---------|---------|
| **Static IP** | 10.30.0.10/24, gateway 10.30.0.1 |
| **DNS** | On — records: `r1.acme.local→10.0.0.1`, `srv1.acme.local→10.30.0.10`, `internet.test→8.8.8.8` |
| **Syslog** | On |
| **HTTP** | On |
| **SNMP** | **Not supported** in PT Server-PT (see §10.1) |
| **NTP** | GUI too limited in PT (see §10.3) |

> [!WARNING]
> PT Server-PT limitations
> - **SNMP service is missing** from the Services tab in most PT builds.
> - **NTP service** only exposes Key / Password / Calendar fields — no server address field.
>
> Both work on real gear; only Server-PT emulation is limited.

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

> [!IMPORTANT]
> Both internal uplinks are `inside`
> R1 has two paths into the enterprise (Gi0/0 → CSW1, Gi0/2 → CSW2). Both must be
> `ip nat inside`. Only Gi0/1 (to ISP) is `ip nat outside`.

### 3.2 NAT ACL and PAT

```cisco
access-list 1 remark === NAT: all internal 10/8 networks ===
access-list 1 permit 10.0.0.0 0.255.255.255

ip nat inside source list 1 interface GigabitEthernet0/1 overload

end
write memory
```

> [!CAUTION]
> The wildcard `0.255.255.255` is critical
> A common mistake is typing `0.0.255.255` (which matches only `10.0.x.x`).
> The correct wildcard for "anything starting with 10" is `0.255.255.255`.
> See §10.5 for the troubleshooting story.

### 3.3 Verify

From **PC-A1**:
```
ping 8.8.8.8
```

On **R1**:
```cisco
show ip nat translations
show ip nat statistics
```

Expected: `Hits` > 0, at least one ICMP translation entry.

From **ISP**:
```cisco
ping 203.0.113.1
```
Should succeed.

---

## 4. NTP — R1 as Master

### 4.1 R1 config

```cisco
enable
configure terminal
ntp master 5
clock timezone UTC 0
exit
write memory
```

> [!NOTE]
> **`clock update-calendar` not supported in PT**
> On real Cisco IOS, `clock update-calendar` syncs the hardware calendar with the
> software clock. PT does not support it — skip.

### 4.2 Client config

On **CSW1, CSW2, DSW-A1, DSW-A2, DSW-B1, DSW-B2, ASW-A1, ASW-A2, ASW-B1, ASW-B2**:

```cisco
enable
configure terminal
ntp server 10.255.255.254
clock timezone UTC 0
exit
write memory
```

> [!NOTE]
> **SRV1 cannot be NTP-configured in PT**
> PT's Server-PT NTP service does not expose a server address field. Set the
> Calendar manually to match R1 if you want visual consistency.

### 4.3 Verify

On any client:
```cisco
show ntp status
show ntp associations
show clock
```

Expected: `Clock is synchronized, stratum 6, reference is 10.255.255.254`.

> [!WARNING]
> NTP sync can take 3–5 minutes in PT
> If unsynchronized, verify reachability first (`ping 10.255.255.254`), then wait.

---

## 5. SNMPv2c

### 5.1 Configure what PT accepts

On **all 11 network devices**:

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

> [!WARNING]
> Packet Tracer SNMP support is minimal
> PT typically accepts only the two `community` commands. The `host`, `enable traps`,
> `trap-source`, `location`, and `contact` commands are often rejected.

### 5.3 Verify

```cisco
show snmp community
```

Expected:
```
Community name: ACME-RO   Community access: Read only
Community name: ACME-RW   Community access: Read write
```

---

## 6. SSH Hardening

### 6.1 Block 1 — Pre-SSH config (paste on each device)

```cisco
enable
configure terminal
ip domain-name acme.local
username admin privilege 15 secret <LAB_PASSWORD>
exit
write memory
```

### 6.2 Block 2 — Generate RSA keys (interactive)

Run this command **alone**:

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

> [!WARNING]
> Packet Tracer requires interactive key generation
> `crypto key generate rsa modulus 2048` is rejected in PT — the modulus must be
> supplied via the prompt. On 2960 switches, if 2048 fails, retry with `1024`.

### 6.3 Block 3 — SSH & VTY hardening

Run this **after** keys are generated:

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

> [!NOTE]
> **ACL 10 was applied in Phase 7**
> `access-class 10 in` is preserved. Do not remove it.

### 6.4 Verify

```cisco
show ip ssh
show crypto key mypubkey rsa
```

Expected: `SSH Enabled - version 2.0`.

### 6.5 Live SSH Login Test

**From an allowed host** — use **SRV1** (`10.30.0.10`), which is permitted by ACL 10:

```
ssh -l admin 10.10.10.2
```

Enter `<LAB_PASSWORD>` → reach the `DSW-A1#` prompt. ✅

**From a disallowed host** — use **PC-A1** (`10.10.10.x`):

```
ssh -l admin 10.10.10.2
```

Expected: connection refused / timed out. ❌

**Observed during the lab:**
- SSH from **SRV1** (10.30.0.10) → ✅ succeeded
- SSH from **PC-A1** (10.10.10.x) → ❌ denied by ACL 10

This confirms both SSH hardening AND Phase 7 ACL work end-to-end.

> [!WARNING]
> SSH test ordering note
> Earlier phases reference testing SSH at Phase 7 — that test was **deferred** to
> this phase because RSA keys and the local user don't exist until now.

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
| NTP | `show ntp status` | Any switch | synchronized, stratum 6 |
| SNMP | `show snmp community` | Any device | ACME-RO (ro) |
| SSH | `show ip ssh` | Any device | v2.0 |
| SSH | from SRV1 → DSW-A1 | ✅ allowed |
| SSH | from PC-A1 → DSW-A1 | ❌ denied |
| DHCP | `show ip dhcp binding` | R1 | Leases for all clients |

---

## 9. Phase 9 Checkpoint

- [x] SRV1 configured with DNS, Syslog, HTTP
- [x] R1: `ip nat inside` on Gi0/0 and Gi0/2, `ip nat outside` on Gi0/1
- [x] NAT ACL 1 = `permit 10.0.0.0 0.255.255.255`
- [x] `ip nat inside source list 1 interface Gi0/1 overload` applied
- [x] PC-A1 and PC-B1 can ping 8.8.8.8
- [x] R1 configured as `ntp master 5`
- [x] All switches configured with `ntp server 10.255.255.254`
- [x] SNMP community strings on all 11 network devices
- [x] SSH version 2, RSA keys generated on all 11 devices
- [x] `username admin privilege 15 secret <LAB_PASSWORD>` on all devices
- [x] VTY uses `login local` + `transport input ssh`
- [x] SSH from SRV1 works, from PC-A1 blocked
- [x] All devices saved

---

## 10. Troubleshooting Encounters

### 10.1 SNMP service missing from Server-PT

SRV1's Services tab has no SNMP agent. Skip trap-receiver verification. On real
systems, use `snmpd` on a Linux host.

### 10.2 SNMP commands rejected beyond community strings

PT supports only bare-minimum SNMP. Accept the community strings and move on.

### 10.3 SRV1 NTP service has no server address field

Skip SRV1 NTP. On real gear, use `ntp server 10.255.255.254`.

### 10.4 `clock update-calendar` rejected

PT does not emulate the hardware calendar chip. Skip.

### 10.5 NAT ACL wildcard bug — the important one

**Symptom:** `show ip nat statistics` shows `Hits: 0  Misses: 43`. `show ip nat
translations` is empty.

**Root Cause:** The ACL was originally configured as
```
access-list 1 permit 10.0.0.0 0.0.255.255
```
This wildcard matches only `10.0.x.x`. Internal networks are `10.10.x.x`,
`10.20.x.x`, `10.30.x.x` — nothing matched.

**Fix:**
```cisco
no access-list 1
access-list 1 remark === NAT: all internal 10/8 networks ===
access-list 1 permit 10.0.0.0 0.255.255.255
```

> [!CAUTION]
> Wildcard math — the classic CCNA trap
> `0.255.255.255` = first octet `10`, others don't care → matches every `10.x.x.x`.
> `0.0.255.255` = first two octets `10.0` → matches only `10.0.x.x`.
> One misplaced digit silently breaks NAT for the whole network.

### 10.6 `crypto key generate rsa modulus 2048` rejected

PT requires interactive input for the RSA modulus. Run `crypto key generate rsa`
alone and enter `2048` at the prompt.

---

## 11. Common Issues & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Ping 8.8.8.8 fails | NAT ACL wildcard wrong | `permit 10.0.0.0 0.255.255.255` |
| NAT shows 0 translations | ACL missing or wrong | `show access-lists 1`, then `clear ip nat translation *` |
| NAT inside/outside reversed | One interface marked wrong | Verify with `show ip nat statistics` |
| NTP unsynchronized | Client can't reach loopback | Verify OSPF route to `10.255.255.254` |
| SSH refused from mgmt host | ACL 10 missing that subnet | Add mgmt subnet to ACL 10 |
| SSH only asks for password | Missing `login local` or username | Reconfigure VTY |

---

## 12. Next Phase

➡️ **[Phase 10 — Automation](Phase-10-Automation.md)**

---

*Phase 9 of the CCNA Master Lab project. See [README](../README.md) for the full phase list.*