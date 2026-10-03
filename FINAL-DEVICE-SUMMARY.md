# Final Device Summary

One-page reference for the complete lab. Loopback0 is the OSPF Router ID on all
devices. All transit subnets are /30 in OSPF area 0.

---

## Routers

| Hostname | Role | Loopback0 | Key Interfaces |
|----------|------|-----------|----------------|
| **ISP** | Simulated Internet | 8.8.8.8/32 | Gi0/0 = 203.0.113.2/30 |
| **R1** | Edge router / ASBR / DHCP / NTP master | 10.255.255.254/32 | Gi0/0 = 10.0.0.1/30 (to CSW1)<br>Gi0/1 = 203.0.113.1/30 (to ISP)<br>Gi0/2 = 10.0.0.9/30 (to CSW2) |

---

## Core Switches

| Hostname | Role | Loopback0 | Key Interfaces |
|----------|------|-----------|----------------|
| **CSW1** | Core L3 (Office A path) | 10.255.255.1/32 | Gi0/1 = 10.0.0.2/30 (to R1)<br>Po1 = 10.0.0.5/30 (to CSW2)<br>Fa0/3 = 10.0.1.1/30 (to DSW-A1)<br>Fa0/4 = 10.0.2.1/30 (to DSW-A2) |
| **CSW2** | Core L3 (Office B path) | 10.255.255.2/32 | Gi0/1 = 10.0.0.10/30 (to R1)<br>Po1 = 10.0.0.6/30 (to CSW1)<br>Fa0/3 = 10.0.3.1/30 (to DSW-B1)<br>Fa0/4 = 10.0.4.1/30 (to DSW-B2) |

---

## Distribution Switches

| Hostname | Role | Loopback0 | Uplink to Core | HSRP Role |
|----------|------|-----------|----------------|-----------|
| **DSW-A1** | Dist A / STP root / HSRP active | 10.255.255.11/32 | Gi0/1 = 10.0.1.2/30 | Active (pri 110) |
| **DSW-A2** | Dist A / STP secondary / HSRP standby | 10.255.255.12/32 | Gi0/1 = 10.0.2.2/30 | Standby (pri 100) |
| **DSW-B1** | Dist B / STP root / HSRP active | 10.255.255.21/32 | Gi0/1 = 10.0.3.2/30 | Active (pri 110) |
| **DSW-B2** | Dist B / STP secondary / HSRP standby | 10.255.255.22/32 | Gi0/1 = 10.0.4.2/30 | Standby (pri 100) |

---

## Access Switches

| Hostname | Office | Access VLANs | Uplinks |
|----------|--------|--------------|---------|
| **ASW-A1** | A | 10, 20, 30, 99 | Gi0/1–2 to DSW-A1/A2 |
| **ASW-A2** | A | 10, 20, 30, 99 | Gi0/1–2 to DSW-A2/A1 |
| **ASW-B1** | B | 10, 20, 30, 99 | Gi0/1–2 to DSW-B1/B2 |
| **ASW-B2** | B | 10, 20, 30, 99 | Gi0/1–2 to DSW-B2/B1 |

---

## Endpoints

| Hostname | VLAN | Address | Gateway |
|----------|------|---------|---------|
| **SRV1** | 300 | 10.30.0.10/24 (static) | 10.30.0.1 (HSRP VIP) |
| **PC-A1** | 10 | DHCP (10.10.10.x) | 10.10.10.1 |
| **PC-A2** | 10 | DHCP (10.10.10.x) | 10.10.10.1 |
| **PC-B1** | 10 | DHCP (10.20.10.x) | 10.20.10.1 |
| **PC-B2** | 10 | DHCP (10.20.10.x) | 10.20.10.1 |

---

## Subnet Summary

| Subnet | Purpose |
|--------|---------|
| 10.10.10.0/24 | Office A – PCs |
| 10.10.20.0/24 | Office A – IP Phones |
| 10.10.30.0/24 | Office A – Wi-Fi |
| 10.10.99.0/24 | Office A – Management |
| 10.20.10.0/24 | Office B – PCs |
| 10.20.20.0/24 | Office B – IP Phones |
| 10.20.30.0/24 | Office B – Servers |
| 10.20.99.0/24 | Office B – Management |
| 10.30.0.0/24 | Services |
| 10.0.0.0/30 | R1 ↔ CSW1 |
| 10.0.0.4/30 | CSW1 ↔ CSW2 (L3 Po1) |
| 10.0.0.8/30 | R1 ↔ CSW2 |
| 10.0.1.0/30 | CSW1 ↔ DSW-A1 |
| 10.0.2.0/30 | CSW1 ↔ DSW-A2 |
| 10.0.3.0/30 | CSW2 ↔ DSW-B1 |
| 10.0.4.0/30 | CSW2 ↔ DSW-B2 |
| 203.0.113.0/30 | R1 ↔ ISP |
| 10.255.255.0/24 | Loopbacks (router IDs) |

Full plan: [MASTER-ADDRESSING.md](MASTER-ADDRESSING.md)

---

*Final device summary — one-page reference for the CCNA Master Lab.*