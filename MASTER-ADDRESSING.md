# Master Addressing Plan

Single source of truth for every IPv4 address, subnet, VLAN, and loopback in the lab.

> **Referenced from:** Phase 1 §5, Phase 3, Phase 6, Phase 11

---

## 1. VLSM Base

Base network: **10.0.0.0/8** (private). VLSM applied to conserve addresses on
point-to-point transit links.

| Block | Purpose |
|-------|---------|
| 10.0.0.0/22 | Transit links (all /30) |
| 10.10.0.0/16 | Office A |
| 10.20.0.0/16 | Office B |
| 10.30.0.0/16 | Services |
| 10.255.0.0/16 | Loopbacks (router IDs) |
| 203.0.113.0/24 | ISP / NAT (public) |

---

## 2. User VLAN Subnets

| Office | VLAN | Name | Subnet | Gateway (HSRP VIP) |
|--------|------|------|--------|--------------------|
| A | 10 | PCs | 10.10.10.0/24 | 10.10.10.1 |
| A | 20 | IP Phones | 10.10.20.0/24 | 10.10.20.1 |
| A | 30 | Wi-Fi | 10.10.30.0/24 | 10.10.30.1 |
| A | 99 | Management | 10.10.99.0/24 | 10.10.99.1 |
| B | 10 | PCs | 10.20.10.0/24 | 10.20.10.1 |
| B | 20 | IP Phones | 10.20.20.0/24 | 10.20.20.1 |
| B | 30 | Servers | 10.20.30.0/24 | 10.20.30.1 |
| B | 99 | Management | 10.20.99.0/24 | 10.20.99.1 |
| — | 300 | Services | 10.30.0.0/24 | 10.30.0.1 (HSRPv2) |

### SVI Physical Addresses

| VLAN | DSW-A1 / DSW-B1 | DSW-A2 / DSW-B2 | HSRP Group | HSRP Version |
|------|------------------|------------------|-----------|--------------|
| 10 | 10.X.10.2 (pri 110) | 10.X.10.3 (pri 100) | 10 | v1 |
| 20 | 10.X.20.2 (pri 110) | 10.X.20.3 (pri 100) | 20 | v1 |
| 30 | 10.X.30.2 (pri 110) | 10.X.30.3 (pri 100) | 30 | v1 |
| 99 | 10.X.99.2 (pri 110) | 10.X.99.3 (pri 100) | 99 | v1 |
| 300 | 10.30.0.2 (pri 110) | 10.30.0.3 (pri 100) | 300 | **v2** |

> `X` = 10 (Office A) or 20 (Office B).
> Group 300 requires HSRPv2 because HSRPv1 supports group IDs 0–255 only.

---

## 3. Transit / Point-to-Point Links (all /30, all OSPF area 0)

| Subnet | A side | Z side | Purpose |
|--------|--------|--------|---------|
| 10.0.0.0/30 | R1 Gi0/0 .1 | CSW1 Gi0/1 .2 | Edge ↔ Core 1 |
| 10.0.0.4/30 | CSW1 Po1 .5 | CSW2 Po1 .6 | Core L3 EtherChannel |
| 10.0.0.8/30 | R1 Gi0/2 .9 | CSW2 Gi0/1 .10 | Edge ↔ Core 2 |
| 10.0.1.0/30 | CSW1 Fa0/3 .1 | DSW-A1 Gi0/1 .2 | Core ↔ Dist A1 |
| 10.0.2.0/30 | CSW1 Fa0/4 .1 | DSW-A2 Gi0/1 .2 | Core ↔ Dist A2 |
| 10.0.3.0/30 | CSW2 Fa0/3 .1 | DSW-B1 Gi0/1 .2 | Core ↔ Dist B1 |
| 10.0.4.0/30 | CSW2 Fa0/4 .1 | DSW-B2 Gi0/1 .2 | Core ↔ Dist B2 |
| 203.0.113.0/30 | R1 Gi0/1 .1 | ISP Gi0/0 .2 | Public edge |

---

## 4. Loopbacks (OSPF Router IDs)

| Device | Interface | Address |
|--------|-----------|---------|
| R1 | Loopback0 | 10.255.255.254/32 |
| CSW1 | Loopback0 | 10.255.255.1/32 |
| CSW2 | Loopback0 | 10.255.255.2/32 |
| DSW-A1 | Loopback0 | 10.255.255.11/32 |
| DSW-A2 | Loopback0 | 10.255.255.12/32 |
| DSW-B1 | Loopback0 | 10.255.255.21/32 |
| DSW-B2 | Loopback0 | 10.255.255.22/32 |
| ISP | Loopback0 | 8.8.8.8/32 (simulated Internet) |

---

## 5. Static Hosts

| Host | Address | Gateway | Note |
|------|---------|---------|------|
| SRV1 | 10.30.0.10/24 | 10.30.0.1 (VIP) | DHCP, DNS, Syslog, HTTP |

All other endpoints use DHCP (Phase 8) and receive the HSRP VIP as their gateway.

---

## 6. NAT

| Item | Value |
|------|-------|
| Inside interfaces | R1 Gi0/0, R1 Gi0/2 |
| Outside interface | R1 Gi0/1 |
| NAT ACL | `permit 10.0.0.0 0.255.255.255` |
| Translation | PAT overload on Gi0/1 |

---

## 7. Management Access

| Item | Value |
|------|-------|
| Allowed VTY sources (ACL 10) | 10.10.99.0/24, 10.20.99.0/24, 10.30.0.0/24 |
| SSH version | 2 |
| Local user | `admin` / `<LAB_PASSWORD>` |

---

*Master addressing plan — update this file whenever addresses change.*