# Phase 2 — VLANs & 802.1Q Trunks

> [!NOTE]
> **Prerequisites & Navigation**
> **Previous:** [Phase 1 — Physical Topology](Phase-01-Physical-Topology-Cabling.md) · **Next:** [Phase 3 — Inter-VLAN Routing](Phase-03-Inter-VLAN-Routing.md)
> Master addressing plan: [MASTER-ADDRESSING.md](../MASTER-ADDRESSING.md)

> [!TIP]
> **Phase Objective**
> Build the Layer 2 foundation: create the VLAN database on all switches,
> configure access ports on the access layer, and configure 802.1Q trunks with
> a hardened native VLAN. **No IP addressing on SVIs yet — that comes in Phase 3.**

> [!IMPORTANT]
> **<text>**
> **Scope Rules**
> - Configure **switches only**.
> - Do **not** touch R1, ISP, or the PCs.
> - Do **not** configure EtherChannels yet — that is Phase 5.

> [!NOTE]
> **Superseded in Phase 6**
> The CSW↔DSW uplinks configured as trunks in this phase are converted to
> **routed ports** in Phase 6 so they can carry OSPF. The trunk configuration
> described here is only valid until Phase 6.

---

## 1. Design Decisions

| Decision | Rationale |
|----------|-----------|
| **VLAN IDs 10, 20, 30, 99, 300, 1000** | Logical separation by department + management + services + native VLAN. |
| **VLAN 1000 as native** | Native VLAN 1 is a well-known attack surface. Use an unused VLAN. |
| **DTP disabled (`switchport nonegotiate`)** | Prevents an attacker from negotiating a trunk via DTP. |
| **No VTP** | VTP is a security and stability risk in production. Create all VLANs manually. |
| **VLAN pruning per office** | Office A only carries Office A VLANs, and vice versa. |
| **Trunk encapsulation `dot1q`** | Industry standard. 3560s also support ISL, but ISL is deprecated. |

---

## 2. VLAN Database (All 10 Switches)

Created manually on **CSW1, CSW2, DSW-A1, DSW-A2, DSW-B1, DSW-B2, ASW-A1, ASW-A2, ASW-B1, ASW-B2**.

```cisco
enable
configure terminal

vlan 10
 name PCs
vlan 20
 name IP_Phones
vlan 30
 name Servers_WiFi
vlan 99
 name Management
vlan 300
 name Services
vlan 1000
 name Native_Unused
exit
```

> [!WARNING]
> **<text>**
> **No VTP**
> Packet Tracer's 2960 and 3560 do not support VTP v3, and this lab deliberately
> avoids VTP entirely. VLANs must be created on every switch manually.

### Verify

```cisco
show vlan brief
```
VLANs 1, 10, 20, 30, 99, 300, 1000 should all appear as **active**.

---

## 3. Access Ports (ASWs Only)

Assigned on **ASW-A1, ASW-A2, ASW-B1, ASW-B2**.

```cisco
interface range FastEthernet0/1 - 10
 switchport mode access
 switchport access vlan 10
 description PCs
 exit

interface range FastEthernet0/11 - 15
 switchport mode access
 switchport access vlan 20
 description IP Phones
 exit

interface range FastEthernet0/16 - 20
 switchport mode access
 switchport access vlan 30
 description WiFi
 exit

interface range FastEthernet0/21 - 23
 switchport mode access
 switchport access vlan 99
 description Management
 exit
```

> [!TIP]
> **Office B cosmetic difference**
> On Office B ASWs, change `description WiFi` to `description Servers`.
> The VLAN ID and behavior are identical — only the description text differs.

---

## 4. Trunk Configuration

> [!NOTE]
> **Superseded in Phase 6**
> The CSW↔DSW trunk config in §4.1 is valid only through Phase 5. Phase 6
> converts those four ports (CSW1 Fa0/3–4 and CSW2 Fa0/3–4, plus the matching
> DSW Gi0/1 ports) into **routed ports**.

### 4.1 Core ↔ Distribution (superseded in Phase 6)

**On CSW1 (uplinks to DSW-A1, DSW-A2):**
```cisco
interface range FastEthernet0/3 - 4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,30,99,300
 switchport nonegotiate
 exit
```

**On CSW2 (uplinks to DSW-B1, DSW-B2):** identical commands on Fa0/3–4.

**On DSW-A1, DSW-A2, DSW-B1, DSW-B2** (uplink Gi0/1 toward the core):
```cisco
interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,30,99,300
 switchport nonegotiate
 exit
```

### 4.2 Distribution ↔ Access

**On DSW-A1, DSW-A2** (downlinks toward Office A ASWs):
```cisco
interface range GigabitEthernet0/2, FastEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,30,99
 switchport nonegotiate
 exit
```

**On DSW-B1, DSW-B2** (downlinks toward Office B ASWs): same commands.

**On ASW-A1, ASW-A2, ASW-B1, ASW-B2** (uplinks):
```cisco
interface range GigabitEthernet0/1 - 2
 switchport mode trunk
 switchport trunk native vlan 1000
 switchport trunk allowed vlan 10,20,30,99
 switchport nonegotiate
 exit
```

> [!WARNING]
> **<text>**
> **2960 does not accept `encapsulation dot1q`**
> The 2960 only supports 802.1Q. If you type `switchport trunk encapsulation dot1q`
> on an ASW, it errors — skip that line on all 2960s.

---

## 5. Trunk Summary

| Link | Ports | Native VLAN | Allowed VLANs |
|------|-------|-------------|---------------|
| CSW1 → DSW-A1/A2 | Fa0/3–4 | 1000 | 10,20,30,99,300 |
| CSW2 → DSW-B1/B2 | Fa0/3–4 | 1000 | 10,20,30,99,300 |
| DSW-A1/A2 → CSW1 | Gi0/1 | 1000 | 10,20,30,99,300 |
| DSW-B1/B2 → CSW2 | Gi0/1 | 1000 | 10,20,30,99,300 |
| DSW-A1/A2 → ASW-A1/A2 | Gi0/2, Fa0/1 | 1000 | 10,20,30,99 |
| DSW-B1/B2 → ASW-B1/B2 | Gi0/2, Fa0/1 | 1000 | 10,20,30,99 |
| ASW-A1/A2 → DSWs | Gi0/1–2 | 1000 | 10,20,30,99 |
| ASW-B1/B2 → DSWs | Gi0/1–2 | 1000 | 10,20,30,99 |

> [!NOTE]
> The CSW↔DSW rows in this table are **superseded in Phase 6** (routed ports).

---

## 6. Explicitly NOT Configured Yet

> [!NOTE]
> **Deferred to later phases**
> - **EtherChannel** on DSW↔DSW links (Fa0/2–Fa0/3) — Phase 5
> - **EtherChannel** on CSW↔CSW links (Fa0/1–Fa0/2) — Phase 5
> - **SVIs / HSRP** — Phase 3
> - **STP root tuning** — Phase 4
> - **Port Security / DHCP Snooping / DAI** — Phase 8

---

## 7. Verification Steps

### 7.1 VLAN Database
```cisco
show vlan brief
```

### 7.2 Trunk Status (CSW1)
```cisco
show interfaces trunk
```
Expected: Fa0/3, Fa0/4 as trunks, native VLAN 1000, allowed 10,20,30,99,300.

### 7.3 Trunk Status (DSW-A1)
```cisco
show interfaces trunk
```
Expected: Gi0/1, Gi0/2, Fa0/1 as trunks, native VLAN 1000, allowed 10,20,30,99.

### 7.4 Trunk Status (ASW-A1)
```cisco
show interfaces trunk
```
Expected: Gi0/1, Gi0/2 as trunks, native VLAN 1000, allowed 10,20,30,99.

### 7.5 Access Port Check (ASW-A1)
```cisco
show interfaces switchport
```

### 7.6 Port-Specific Detail (CSW1)
```cisco
show interfaces FastEthernet0/3 switchport
```

---

## 8. Phase 2 Checkpoint

- [x] VLANs 10, 20, 30, 99, 300, 1000 exist on **all 10 switches**
- [x] Access ports on all 4 ASWs mapped to VLANs 10/20/30/99
- [x] All uplinks configured as trunks
- [x] Native VLAN = 1000 on every trunk
- [x] DTP disabled (`switchport nonegotiate`) on every trunk
- [x] No EtherChannel, no SVIs, no HSRP yet
- [x] All switches saved to startup-config

---

## 9. Observed Behavior (Expected Weirdness)

| Observation | Why it's Expected |
|-------------|--------------------|
| `show interfaces trunk` on DSWs shows Fa0/2 and Fa0/3 as "not-trunking" | Those are still plain links pending Phase 5 EtherChannel. |
| PC-A1 cannot ping PC-A2 | No IP addresses configured yet. |
| STP still shows one blocked port per loop | No root tuning yet — that's Phase 4. |

---

## 10. Common Issues & Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Trunk doesn't come up | One side is `access`, other is `trunk` | Set both ends to `switchport mode trunk` |
| Native VLAN mismatch warning | One end still has native VLAN 1 | `switchport trunk native vlan 1000` on both ends |
| VLAN missing from `show vlan brief` | Not created manually | Re-enter `vlan <id>` and `name <name>` |
| Trunk is up but VLANs are pruned | VLAN not in `allowed vlan` list | `switchport trunk allowed vlan add <id>` |
| `encapsulation dot1q` errors on 2960 | 2960 doesn't support it | Skip the line on ASWs |

---

## 11. Next Phase

➡️ **[Phase 3 — Inter-VLAN Routing (SVI + HSRP)](Phase-03-Inter-VLAN-Routing.md)**

---

*Phase 2 of the CCNA Master Lab project. See [README](../README.md) for the full phase list.*