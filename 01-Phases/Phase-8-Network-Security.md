---
title: "Phase 8 — Network Security (Port Security, DHCP Snooping, DAI)"
project: CCNA Master Lab — Acme Corp Two-Office Enterprise
phase: 8
status: complete
tags:
  - ccna
  - lab
  - port-security
  - dhcp-snooping
  - dai
  - layer-2-security
  - packet-tracer
created: 2026-09-25
updated: 2026-09-25
---

# Phase 8 — Network Security (Port Security, DHCP Snooping, DAI)

> [!info] Project Context
> This lab models **Acme Corp**, a two-office enterprise connected through a dual-homed core and a single edge router to a simulated ISP. Each phase builds on the previous one, ending with a fully integrated CCNA lab covering VLANs, STP, EtherChannel, OSPF, ACLs, security hardening, services, and automation.

> [!success] Phase Objective
> Harden the access layer against three classic Layer-2 attacks:
> 1. **MAC flooding / rogue devices** → Port Security
> 2. **Rogue DHCP servers** → DHCP Snooping
> 3. **ARP spoofing / MITM** → Dynamic ARP Inspection

> [!warning] Order matters
> DAI relies on the DHCP Snooping binding table. If DAI is enabled before DHCP is working, **every ARP packet gets dropped** and the office goes dark. DHCP first, verify, then DAI.

---

## 1. Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Port Security `maximum 2`** | Allows a PC + IP phone on one port (common office scenario). |
| **Sticky MAC learning** | Auto-populates the secure MAC table — no manual provisioning. |
| **Violation mode `shutdown`** | Port err-disables on violation — safest option. |
| **Cisco IOS DHCP on R1** | Instructional value; central location reachable via helper-address. |
| **`no ip dhcp snooping information option`** | Option 82 causes DHCP to break in Packet Tracer when trust boundaries aren't recognized. |
| **`ip helper-address 10.255.255.254`** | Loopback of R1 — always reachable regardless of physical uplink state. |
| **DAI with `src-mac dst-mac ip` validation** | Full ARP validation; if it causes issues, remove `ip`. |

---

## 2. Port Security (All ASWs)

### Configuration

On **ASW-A1, ASW-A2, ASW-B1, ASW-B2**:

```cisco
enable
configure terminal

interface range FastEthernet0/1 - 23
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 switchport port-security aging time 60
 exit

end
write memory
```

> [!warning] Packet Tracer limitation — `aging type inactivity`
> `switchport port-security aging type inactivity` is **rejected** by PT's 2960 emulation. The command is valid on real Cisco IOS and is preferred in production (only ages silent MACs). PT accepts only `absolute` (the default), which ages MACs on a fixed timer regardless of activity.
>
> **Workaround:** Skip the command in PT. Note the intent in documentation. Apply on real gear.

> [!warning] Sticky MACs need to be saved
> `switchport port-security mac-address sticky` stores learned MACs in the **running config** only. Always run `write memory` — otherwise a reload wipes them.

### Verification

```cisco
show port-security
show port-security interface FastEthernet0/1
show port-security address
```

**Expected:**
- Port Status: `Secure-up`
- Violation Mode: `Shutdown`
- Maximum MAC Addresses: `2`
- Sticky MAC Addresses: 1–2 learned

### Live Violation Test (Hub Method)

This is the recommended way to prove Port Security works.

1. Disconnect PC-A1 from ASW-A1 Fa0/1.
2. Insert an unmanaged **hub** between the port and the PC.
3. Connect **PC-A1** to the hub → Port learns 1 MAC.
4. Connect a second device (e.g., PC-A2 temporarily) to the hub → Port learns 2 MACs. ✅ (within maximum)
5. Connect a **third** device → **violation triggered**, port goes err-disabled.

Check:
```cisco
show interfaces FastEthernet0/1 status
```
Expected: `err-disabled (Port-Security)`.

**Recover:**
```cisco
interface FastEthernet0/1
 shutdown
 no shutdown
 exit
```

Or clear the sticky MAC and remove the offending device first:
```cisco
clear port-security sticky interface FastEthernet0/1
```

---

## 3. DHCP Server Configuration (R1)

DHCP is a **prerequisite** for DHCP Snooping.

### On R1

```cisco
enable
configure terminal

! Exclude reserved addresses (gateways + VIPs + static hosts)
ip dhcp excluded-address 10.10.10.1 10.10.10.9
ip dhcp excluded-address 10.10.20.1 10.10.20.9
ip dhcp excluded-address 10.10.30.1 10.10.30.9
ip dhcp excluded-address 10.10.99.1 10.10.99.9

ip dhcp excluded-address 10.20.10.1 10.20.10.9
ip dhcp excluded-address 10.20.20.1 10.20.20.9
ip dhcp excluded-address 10.20.30.1 10.20.30.9
ip dhcp excluded-address 10.20.99.1 10.20.99.9

ip dhcp excluded-address 10.30.0.1 10.30.0.9

! Office A pools
ip dhcp pool OFFICE_A_PCS
 network 10.10.10.0 255.255.255.0
 default-router 10.10.10.1
 dns-server 10.30.0.10
 lease 0 8
 exit

ip dhcp pool OFFICE_A_PHONES
 network 10.10.20.0 255.255.255.0
 default-router 10.10.20.1
 dns-server 10.30.0.10
 lease 0 8
 exit

ip dhcp pool OFFICE_A_WIFI
 network 10.10.30.0 255.255.255.0
 default-router 10.10.30.1
 dns-server 10.30.0.10
 lease 0 8
 exit

! Office B pools
ip dhcp pool OFFICE_B_PCS
 network 10.20.10.0 255.255.255.0
 default-router 10.20.10.1
 dns-server 10.30.0.10
 lease 0 8
 exit

ip dhcp pool OFFICE_B_PHONES
 network 10.20.20.0 255.255.255.0
 default-router 10.20.20.1
 dns-server 10.30.0.10
 lease 0 8
 exit

ip dhcp pool OFFICE_B_SERVERS
 network 10.20.30.0 255.255.255.0
 default-router 10.20.30.1
 dns-server 10.30.0.10
 lease 0 8
 exit

end
write memory
```

> [!tip] Default gateway = HSRP VIP
> All `default-router` values are HSRP virtual IPs (`.1`). Hosts automatically get a redundant gateway — no host-side changes on HSRP failover.

---

## 4. DHCP Relay (All DSWs)

DHCP is a Layer-2 broadcast. It cannot cross VLAN boundaries on its own, and the DHCP server sits outside the office VLANs (on R1). Each SVI needs a helper-address pointing to R1.

### On DSW-A1, DSW-A2, DSW-B1, DSW-B2

```cisco
interface Vlan10
 ip helper-address 10.255.255.254
 exit

interface Vlan20
 ip helper-address 10.255.255.254
 exit

interface Vlan30
 ip helper-address 10.255.255.254
 exit

interface Vlan99
 ip helper-address 10.255.255.254
 exit
```

> [!tip] Why the loopback (`10.255.255.254`) and not R1's physical IP?
> The loopback is always up, regardless of which physical uplink fails.

---

## 5. Convert PCs to DHCP Clients

On each PC: **Desktop → IP Configuration → select DHCP**.

| Host | Before | After |
|------|--------|-------|
| PC-A1 | 10.10.10.10 | DHCP |
| PC-A2 | 10.10.10.11 | DHCP |
| PC-B1 | 10.20.10.10 | DHCP |
| PC-B2 | 10.20.10.11 | DHCP |

> **SRV1 stays static at `10.30.0.10`** — it's a server.

### Verify

On **PC-A1**:
```
ipconfig
```
Expected: IP in `10.10.10.10 – 10.10.10.254`, gateway `10.10.10.1`.

On **R1**:
```cisco
show ip dhcp binding
```
Expected: one entry per client, with MAC and lease.

---

## 6. DHCP Snooping (All ASWs)

### Configuration

On **ASW-A1, ASW-A2, ASW-B1, ASW-B2**:

```cisco
enable
configure terminal

ip dhcp snooping
ip dhcp snooping vlan 10,20,30,99
no ip dhcp snooping information option

interface range GigabitEthernet0/1 - 2
 ip dhcp snooping trust
 exit

interface range FastEthernet0/1 - 23
 no ip dhcp snooping trust
 ip dhcp snooping limit rate 10
 exit

end
write memory
```

> [!important] `no ip dhcp snooping information option`
> Option 82 inserts relay info into DHCP packets. In Packet Tracer this often breaks DHCP because the trust boundary isn't recognized correctly. Standard workaround for labs.

### Verify

```cisco
show ip dhcp snooping
show ip dhcp snooping binding
```

Expected: Snooping enabled on VLANs 10,20,30,99, Gi0/1–2 trusted, and one binding per DHCP client.

> If the binding table is empty, run `ipconfig /release` then `ipconfig /renew` on each PC.

---

## 7. Dynamic ARP Inspection (All ASWs)

### Configuration

On **ASW-A1, ASW-A2, ASW-B1, ASW-B2**:

```cisco
enable
configure terminal

ip arp inspection vlan 10,20,30,99
ip arp inspection validate src-mac dst-mac ip

interface range GigabitEthernet0/1 - 2
 ip arp inspection trust
 exit

interface range FastEthernet0/1 - 23
 no ip arp inspection trust
 exit

end
write memory
```

> [!warning] DAI needs the binding table
> If you enable DAI before DHCP snooping has populated the binding table, **every ARP gets dropped** and hosts lose connectivity to their own gateways. Always verify `show ip dhcp snooping binding` is populated first.

> [!note] `ip arp inspection validate ip`
> On some PT versions this check drops legitimate ARP because the sender IP isn't yet in the binding table. If ARP breaks, remove the `ip` keyword:
> ```cisco
> ip arp inspection validate src-mac dst-mac
> ```

### Verify

```cisco
show ip arp inspection
show ip arp inspection statistics
```

Expected: Enabled on VLANs 10,20,30,99; counters for `Forwarded` and `Dropped`.

### Connectivity test

From **PC-A1**:
```
ping 10.10.10.1
ping 10.20.10.10
```
Both should succeed.

---

## 8. Phase 8 Checkpoint

- [x] Port Security on ASWs (`maximum 2`, `sticky`, `violation shutdown`, `aging time 60`)
- [x] `aging type inactivity` — **skipped** (PT limitation)
- [x] DHCP pools on R1 for all VLANs
- [x] `ip helper-address 10.255.255.254` on all DSW SVIs
- [x] PCs set to DHCP client — receive IPs
- [x] `show ip dhcp binding` on R1 shows leases
- [x] DHCP Snooping enabled on all ASWs for VLANs 10,20,30,99
- [x] `no ip dhcp snooping information option` applied
- [x] ASW uplinks trusted for DHCP snooping
- [x] Host ports rate-limited to 10 pps
- [x] `show ip dhcp snooping binding` shows entries for all clients
- [x] DAI enabled for VLANs 10,20,30,99 on all ASWs
- [x] ASW uplinks trusted for DAI
- [x] Hosts can ping gateways and remote hosts
- [x] Port Security `Secure-up` on all access ports
- [x] Hub-based violation test triggered err-disable as expected
- [x] All devices saved to startup-config

---

## 9. Troubleshooting Encounters

### 9.1 `switchport port-security aging type inactivity` rejected

**Symptom:** PT 2960 rejects `switchport port-security aging type inactivity` with `% Invalid input detected`.

**Cause:** Packet Tracer's 2960 emulation does not support `aging type inactivity`. Only `absolute` (the default) is accepted.

**Fix:** Skip in PT. On real Cisco IOS, `inactivity` is the preferred mode — MACs age out only if they have been **silent** for the configured time.

---

### 9.2 Live Port-Security Violation Test (Hub Method)

**Setup:**
1. Disconnect PC-A1 from ASW-A1 Fa0/1.
2. Insert an unmanaged **hub** between the port and the PC.
3. Connect **PC-A1** → 1 MAC.
4. Connect a second device → 2 MACs (within maximum).
5. Connect a **third** device → violation triggered.

**Observed:**
- Port Fa0/1 → err-disabled (Port-Security)
- `show port-security interface Fa0/1` → `Secure-shutdown`
- `show interfaces Fa0/1 status` → `err-disabled`

**Recovery:**
```cisco
interface FastEthernet0/1
 shutdown
 no shutdown
 exit
```

> [!tip] Strongest verification of Port Security
> The hub method proves the feature works end-to-end — not just that it's configured. It's the recommended classroom test.

---

## 10. Common Issues & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| DHCP clients get 169.254.x.x (APIPA) | Helper address missing or wrong | Add `ip helper-address` on SVI |
| DHCP works but no binding entries | Snooping not enabled on that VLAN, or client hasn't renewed | `ipconfig /release` + `ipconfig /renew` |
| All ARP blocked after DAI | Binding table empty, or `validate ip` too strict | Verify snooping binding first; remove `ip` from validate |
| Port err-disabled | Port Security violation | `shutdown` / `no shutdown`, or `clear port-security sticky interface Fa0/1` |
| Sticky MACs lost on reload | Forgot `write memory` | Save after learning MACs |
| Legit DHCP packets dropped | Uplink not trusted | `ip dhcp snooping trust` on Gi0/1–2 |
| DAI drops every packet | Uplinks not trusted | `ip arp inspection trust` on Gi0/1–2 |
| Port Security blocks phone + PC | `maximum` too low | Set to 2 (or higher for IP phones) |

---

## 11. Lessons Learned

> [!note] Enable security features in the right order
> DHCP → DHCP Snooping → DAI. Skipping a step breaks everything downstream.

> [!note] Trust boundaries are the key concept
> Uplinks toward servers are trusted. Host-facing ports are untrusted. Both DHCP Snooping and DAI follow this rule.

> [!note] Test features, don't just configure them
> The hub method for Port Security is the gold standard. The same principle applies elsewhere — you'll do more of this in Phase 11.

> [!note] Packet Tracer quirks accumulate
> By Phase 8 we've hit: `ip ospf network point-to-point`, `no passive-interface Port-channel1`, `log` on ACLs, `aging type inactivity`, and `show ip interface` on SVIs. Note each one in documentation as you go.

---

## 12. Next Phase

➡️ **[Phase 9 — Services (NAT, NTP, SNMP, SSH)](Phase-9-Services.md)**

In Phase 9 we will:
- Configure NAT/PAT on R1 for Internet access
- Configure NTP master on R1, clients on all devices
- Configure SNMPv2c read-only with traps to SRV1
- Complete SSH hardening on all devices

---

*Document maintained as part of the CCNA Master Lab project. For questions, corrections, or contributions, see the repository README.*