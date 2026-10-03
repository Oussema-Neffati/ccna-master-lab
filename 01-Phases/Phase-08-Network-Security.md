# Phase 8 — Network Security (Port Security, DHCP Snooping, DAI)

> [!NOTE]\n> **Prerequisites & Navigation**
> **Previous:** [Phase 7 — ACLs](Phase-07-ACLs.md) · **Next:** [Phase 9 — Services](Phase-09-Services.md)
> Master addressing plan: [MASTER-ADDRESSING.md](../MASTER-ADDRESSING.md)

> > [!TIP]\n> **Phase Objective**
> Harden the access layer against three classic Layer 2 attacks:
> 1. **MAC flooding / rogue devices** → Port Security
> 2. **Rogue DHCP servers** → DHCP Snooping
> 3. **ARP spoofing / MITM** → Dynamic ARP Inspection

> [!CAUTION] Order matters
> DAI relies on the DHCP Snooping binding table. If DAI is enabled before DHCP is
> working, **every ARP packet gets dropped** and the office goes dark.
> DHCP → verify → DHCP Snooping → verify → DAI.

---

## 1. Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Port Security `maximum 2`** | Allows a PC + IP phone on one port (common office scenario). |
| **Sticky MAC learning** | Auto-populates the secure MAC table — no manual provisioning. |
| **Violation mode `shutdown`** | Port err-disables on violation — safest option. |
| **Cisco IOS DHCP on R1** | Instructional value; central location reachable via helper-address. |
| **`no ip dhcp snooping information option`** | Option 82 breaks DHCP in PT when trust boundaries aren't recognized. |
| **`ip helper-address 10.255.255.254`** | R1's loopback — always reachable regardless of physical uplink. |

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

> [!WARNING] Packet Tracer limitation — `aging type inactivity`
> `switchport port-security aging type inactivity` is **rejected** by PT's 2960
> emulation. The command is valid Cisco IOS and preferred in production (ages
> only silent MACs). PT accepts only `absolute` (the default).
>
> **Workaround:** Skip in PT. Note the intent in documentation. Apply on real gear.

> [!WARNING] Sticky MACs must be saved
> `switchport port-security mac-address sticky` stores learned MACs in the
> **running config** only. Always run `write memory` — otherwise reload wipes them.

### Verification

```cisco
show port-security
show port-security interface FastEthernet0/1
show port-security address
```

**Expected:** `Secure-up`, Violation Mode `Shutdown`, Maximum `2`, sticky MACs learned.

### Live Violation Test (Hub Method)

1. Disconnect PC-A1 from ASW-A1 Fa0/1.
2. Insert an unmanaged hub between the port and the PC.
3. Connect PC-A1 → 1 MAC.
4. Connect a second device → 2 MACs. ✅ (within maximum)
5. Connect a **third** device → **violation**, port goes err-disabled.

```cisco
show interfaces FastEthernet0/1 status
```
Expected: `err-disabled (Port-Security)`.

**Recovery:**
```cisco
interface FastEthernet0/1
 shutdown
 no shutdown
 exit
```
Or clear sticky MACs first:
```cisco
clear port-security sticky interface FastEthernet0/1
```

---

## 3. DHCP Server Configuration (R1)

DHCP is a **prerequisite** for DHCP Snooping.

```cisco
enable
configure terminal

! Exclude reserved ranges (gateways + VIPs + static hosts)
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
! ... (repeat for OFFICE_A_PHONES, OFFICE_A_WIFI)

! Office B pools
ip dhcp pool OFFICE_B_PCS
 network 10.20.10.0 255.255.255.0
 default-router 10.20.10.1
 dns-server 10.30.0.10
 lease 0 8
 exit
! ... (repeat for OFFICE_B_PHONES, OFFICE_B_SERVERS)

end
write memory
```

> [!TIP] Default gateway = HSRP VIP
> All `default-router` values are HSRP virtual IPs (`.1`). Hosts automatically
> get a redundant gateway — no host-side change on HSRP failover.

---

## 4. DHCP Relay (All DSWs)

DHCP is a Layer 2 broadcast. It cannot cross VLAN boundaries on its own, and the
DHCP server (R1) sits outside the office VLANs. Each SVI needs a helper-address.

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

> [!TIP] Why the loopback?
> R1's loopback (`10.255.255.254`) is always up, regardless of which physical
> uplink fails. This makes the helper address robust.

---

## 5. Convert PCs to DHCP Clients

On each PC: **Desktop → IP Configuration → select DHCP**.

| Host | Before | After |
|------|--------|-------|
| PC-A1 | 10.10.10.10 | DHCP |
| PC-A2 | 10.10.10.11 | DHCP |
| PC-B1 | 10.20.10.10 | DHCP |
| PC-B2 | 10.20.10.11 | DHCP |

> SRV1 stays static at `10.30.0.10`.

**Verify on PC-A1:** `ipconfig` → IP in `10.10.10.10–.254`, gateway `10.10.10.1`.

**Verify on R1:** `show ip dhcp binding` → one entry per client.

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

> [!IMPORTANT] `no ip dhcp snooping information option`
> Option 82 inserts relay info into DHCP packets. In PT this often breaks DHCP.
> Standard workaround for labs.

### Verify

```cisco
show ip dhcp snooping
show ip dhcp snooping binding
```

Expected: enabled on VLANs 10,20,30,99; Gi0/1–2 trusted; one binding per client.

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

> [!WARNING] DAI needs the binding table
> If DAI is enabled before DHCP Snooping populates the binding table, **every ARP
> gets dropped** and hosts lose connectivity. Always verify
> `show ip dhcp snooping binding` is populated first.

> [!NOTE] What `validate ip` actually checks
> `ip arp inspection validate ip` checks for **invalid sender/target IP addresses**
> in the ARP packet — specifically `0.0.0.0`, `255.255.255.255`, and multicast
> addresses. It does **not** consult the DHCP Snooping binding table.
>
> The binding table is checked by **DAI itself** (it's the default behavior when
> DAI is enabled on a VLAN). The `validate` options are extra sanity checks on
> top of that.
>
> If your PT version causes legitimate ARP to fail with `validate ip`, remove
> the keyword:
> ```cisco
> ip arp inspection validate src-mac dst-mac
> ```

### Verify

```cisco
show ip arp inspection
show ip arp inspection statistics
```

Expected: enabled on VLANs 10,20,30,99; counters for `Forwarded` and `Dropped`.

### Connectivity test

From **PC-A1**:
```
ping 10.10.10.1
ping 10.20.10.10
```
Both should succeed.

---

## 8. Phase 8 Checkpoint

- [ ] Port Security on ASWs (`maximum 2`, `sticky`, `violation shutdown`, `aging time 60`)
- [ ] `aging type inactivity` — **skipped** (PT limitation)
- [ ] DHCP pools on R1 for all VLANs
- [ ] `ip helper-address 10.255.255.254` on all DSW SVIs
- [ ] PCs set to DHCP client — receive IPs
- [ ] `show ip dhcp binding` on R1 shows leases
- [ ] DHCP Snooping enabled on all ASWs for VLANs 10,20,30,99
- [ ] `no ip dhcp snooping information option` applied
- [ ] ASW uplinks trusted for DHCP snooping
- [ ] Host ports rate-limited to 10 pps
- [ ] `show ip dhcp snooping binding` shows entries for all clients
- [ ] DAI enabled for VLANs 10,20,30,99 on all ASWs
- [ ] ASW uplinks trusted for DAI
- [ ] Hosts can ping gateways and remote hosts
- [ ] Port Security `Secure-up` on all access ports
- [ ] Hub-based violation test triggered err-disable as expected
- [ ] All devices saved

---

## 9. Troubleshooting Encounters

### 9.1 `aging type inactivity` rejected

**Symptom:** PT 2960 rejects with `% Invalid input detected`.

**Cause:** PT emulation limitation. Only `absolute` (default) is accepted.

**Fix:** Skip in PT. On real Cisco IOS, `inactivity` is the preferred mode.

### 9.2 Live Port-Security Violation Test (Hub Method)

**Setup:** Hub between ASW-A1 Fa0/1 and PC-A1, then add devices.

**Observed:**
- Fa0/1 → err-disabled (Port-Security)
- `show port-security interface Fa0/1` → `Secure-shutdown`

**Recovery:**
```cisco
interface FastEthernet0/1
 shutdown
 no shutdown
 exit
```

> [!TIP] Strongest verification of Port Security
> The hub method proves the feature end-to-end, not just that it's configured.

---

## 10. Common Issues & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| DHCP clients get 169.254.x.x | Helper address missing | Add `ip helper-address` on SVI |
| No binding entries | Snooping not enabled on VLAN | `ipconfig /release` + `/renew` |
| All ARP blocked after DAI | Binding table empty | Verify snooping binding first |
| Port err-disabled | Port Security violation | `shutdown` / `no shutdown` |
| Sticky MACs lost on reload | Forgot `write memory` | Save after learning MACs |
| Legit DHCP dropped | Uplink not trusted | `ip dhcp snooping trust` on Gi0/1–2 |
| DAI drops everything | Uplinks not trusted | `ip arp inspection trust` on Gi0/1–2 |
| Port Security blocks phone + PC | `maximum` too low | Set to 2 (or higher) |

---

## 11. Lessons Learned

> [!NOTE] Enable security features in the right order
> DHCP → DHCP Snooping → DAI.

> [!NOTE] Trust boundaries are the key concept
> Uplinks toward servers are trusted. Host-facing ports are untrusted.

> [!NOTE] Test features, don't just configure them
> The hub method for Port Security is the gold standard.

> [!NOTE] Packet Tracer quirks accumulate
> By Phase 8 we've hit: `ip ospf network point-to-point`,
> `no passive-interface Port-channel1`, `log` on ACLs, `aging type inactivity`,
> `show ip interface` on SVIs.

---

## 12. Next Phase

➡️ **[Phase 9 — Services](Phase-09-Services.md)**

---

*Phase 8 of the CCNA Master Lab project. See [README](../README.md) for the full phase list.*