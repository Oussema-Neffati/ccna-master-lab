# CCNA Master Lab — Acme Corp Two-Office Enterprise

A complete CCNA 200-301 lab built in Cisco Packet Tracer, modeling a two-office
enterprise with a dual-homed edge, routed core, and a full VLAN / OSPF / security stack.

![Status](https://img.shields.io/badge/status-complete-brightgreen)
![Phases](https://img.shields.io/badge/phases-11%2F11-blue)
![Devices](https://img.shields.io/badge/devices-17-informational)
![License](https://img.shields.io/badge/license-MIT-green)

---

## Overview

Acme Corp operates two regional offices (A and B) connected through a redundant
Layer-3 core. A single edge router (R1) provides Internet access and NAT, running
OSPF with the two core switches (dual-homed). Each office has a distribution /
access layer with HSRP-redundant gateways, VLSM-based subnets, and hardened
access ports.

The lab is split into 11 phases that build on each other — follow them in order.

---

## Topology


![Physical-topology](02-Diagrams\Physical-topology-v2.png)

---

## Phases

| # | Topic | Objective | CCNA Domain | Doc |
|---|-------|-----------|-------------|-----|
| 1 | Physical Topology & Cabling | Place devices, cable 26 links | Foundations | [Phase 1](01-Phases/Phase-01-Physical-Topology-Cabling.md) |
| 2 | VLANs & 802.1Q Trunks | Build VLAN DB, access ports, hardened trunks | Layer 2 | [Phase 2](01-Phases/Phase-02-VLANs-and-Trunks.md) |
| 3 | Inter-VLAN Routing (SVI + HSRP) | Gateways + HSRP redundancy | Layer 3 | [Phase 3](01-Phases/Phase-03-Inter-VLAN-Routing.md) |
| 4 | Rapid PVST+ Tuning | Deterministic root bridges, PortFast, BPDU Guard | Layer 2 | [Phase 4](01-Phases/Phase-04-Rapid-PVST-Tuning.md) |
| 5 | EtherChannel (LACP) | L2 + L3 PortChannels | Layer 2 / Aggregation | [Phase 5](01-Phases/Phase-05-EtherChannel.md) |
| 6 | OSPFv2 Single Area | Inter-office routing + default route | Layer 3 / Routing | [Phase 6](01-Phases/Phase-06-OSPFv2.md) |
| 7 | ACLs (Standard & Extended) | VTY restriction + inter-VLAN filtering | Security | [Phase 7](01-Phases/Phase-07-ACLs.md) |
| 8 | Network Security | Port Security, DHCP Snooping, DAI | Security | [Phase 8](01-Phases/Phase-08-Network-Security.md) |
| 9 | Services | DHCP, NAT/PAT, NTP, SNMP, SSH | IP Services | [Phase 9](01-Phases/Phase-09-Services.md) |
| 10 | Automation (Concepts) | RESTCONF, NETCONF, Python, Ansible, SDN | Automation | [Phase 10](01-Phases/Phase-10-Automation.md) |
| 11 | Troubleshooting | 8 injected scenarios + final review | All | [Phase 11](01-Phases/Phase-11-Troubleshooting.md) |

---

## Master Addressing

Full IPv4 / VLSM plan, transit subnets, loopbacks, and HSRP table:

➡️ **[MASTER-ADDRESSING.md](MASTER-ADDRESSING.md)**

---

## Technologies Covered

**Layer 2:** VLANs · 802.1Q · Rapid PVST+ · LACP EtherChannel · Root Guard · BPDU Guard · PortFast

**Layer 3:** IPv4 · VLSM · SVI · HSRP (v1 + v2) · OSPFv2 · Passive Interfaces · Default Route Origination

**Security:** Standard & Extended ACLs · Port Security · DHCP Snooping · Dynamic ARP Inspection

**IP Services:** DHCP (IOS server) · NAT/PAT · NTP · SNMP · SSH

**Automation (conceptual, PT-unsupported):** RESTCONF · NETCONF · Python · Ansible · SDN (Cisco Catalyst Center / DNA Center)

> Automation is **documented and reviewed** in Phase 10, not implemented — Packet Tracer has no runtime support for it.

---

## Scope

This lab focuses on the **core switching, routing, security, and IP-services domains** of the CCNA 200-301 exam. It does **not** cover:

- IPv6
- Wireless (WLC, CAPWAP, roaming)
- QoS

---

## Lab Statistics

| Metric | Value |
|--------|-------|
| Devices | 17 (12 network + 5 endpoints) |
| Physical links | 26 |
| VLANs | 6 |
| OSPF routers | 7 |
| ACLs | 2 (1 standard + 1 extended) |
| Troubleshooting scenarios | 8 |
| Packet Tracer limitations documented | 12 |

---

## Environment

| Item | Version |
|------|---------|
| Cisco Packet Tracer | **8.2.x** (replace with your version) |
| Python (for reference scripts) | 3.9+ |
| Lab credentials | `<LAB_PASSWORD>` (see Phase 9) |

> Lab-only credentials are placeholders. **Never use these on real equipment.**

---

## How to Use

Phases build on each other. Follow them **in order**:
each phase assumes the previous phase's configuration is in place.

Each phase file is self-contained in its scope (prerequisites, commands,
verification, common pitfalls), but relies on the output of earlier phases.

---

## Repository Structure

```
ccna-master-lab/
├── README.md
├── MASTER-ADDRESSING.md
├── .gitignore
├── 01-Phases/               # 11 phase documents
├── 02-Diagrams/             # topology images
```

---

## Known Limitations

12 Packet Tracer limitations are documented across the phases. A consolidated list
is at the end of [Phase 11](01-Phases/Phase-11-Troubleshooting.md#packet-tracer-limitations).

Every command listed in the phases is valid Cisco IOS syntax. Where PT rejects it,
the doc notes the limitation and provides the workaround.

---

## License

MIT