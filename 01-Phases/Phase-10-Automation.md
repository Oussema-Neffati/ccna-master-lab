---
title: "Phase 10 — Basic Automation & Programmability"
project: CCNA Master Lab — Acme Corp Two-Office Enterprise
phase: 10
status: complete
tags:
  - ccna
  - lab
  - automation
  - restconf
  - netconf
  - python
  - ansible
  - sdn
  - dna-center
  - packet-tracer
created: 2026-09-26
updated: 2026-09-26
---

# Phase 10 — Basic Automation & Programmability

> [!info] Project Context
> This lab models **Acme Corp**, a two-office enterprise connected through a dual-homed core and a single edge router to a simulated ISP. Each phase builds on the previous one, ending with a fully integrated CCNA lab covering VLANs, STP, EtherChannel, OSPF, ACLs, security hardening, services, and automation.

> [!success] Phase Objective
> Understand and demonstrate the automation and programmability concepts that appear on the CCNA 200-301 exam:
> 1. REST APIs / RESTCONF / NETCONF
> 2. HTTP verbs (CRUD)
> 3. Data formats (JSON / XML / YAML)
> 4. Python scripting basics
> 5. Configuration management tools (Ansible, Puppet, Chef)
> 6. SDN concepts and Cisco DNA Center

> [!warning] Packet Tracer has essentially no automation support
> PT rejects `restconf`, `ip http server`, `ip http secure-server`, and every related command. This phase is **conceptual** — commands are documented for real IOS-XE, not for PT execution. To practice hands-on automation, use **Cisco DevNet Sandbox**, **CML**, or a **CSR1000v** image.

---

## 1. Traditional vs. Controller-Based Networking

| Aspect | Traditional | Controller-Based (SDN) |
|--------|-------------|------------------------|
| **Configuration** | Per-device CLI (SSH) | Central controller pushes config |
| **Data plane** | Distributed | Distributed |
| **Control plane** | Distributed | Centralized on the controller |
| **Management plane** | Per-device | Centralized |
| **Southbound API** | N/A | NETCONF, RESTCONF, OpenFlow |
| **Northbound API** | N/A | REST (JSON/XML) — used by apps |
| **Cisco product** | N/A | DNA Center, APIC-EM (legacy) |

**Key exam concepts:**

- The **control plane** decides *how* to forward (routing protocols, ARP, STP).
- The **data plane** actually forwards packets (ASICs, hardware).
- The **management plane** configures and monitors (SSH, SNMP, Syslog).
- In SDN, the **control plane moves to the controller**; devices become "dumb" forwarders.
- **Southbound** = controller → device. **Northbound** = app → controller.

---

## 2. REST API Fundamentals

**REST** = Representational State Transfer. It's an architectural style, not a protocol.

### 2.1 HTTP verbs (CRUD)

| Verb | CRUD | Purpose |
|------|------|---------|
| **GET** | Read | Retrieve data |
| **POST** | Create | Create a new resource |
| **PUT** | Update / Replace | Replace an existing resource |
| **PATCH** | Update / Modify | Modify part of a resource |
| **DELETE** | Delete | Remove a resource |

### 2.2 HTTP status codes

| Code | Meaning |
|------|---------|
| 200 | OK |
| 201 | Created |
| 204 | No Content (successful delete) |
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 500 | Internal Server Error |

### 2.3 RESTCONF vs. NETCONF

| Feature | RESTCONF | NETCONF |
|---------|----------|---------|
| Transport | HTTP/HTTPS | SSH |
| Encoding | JSON or XML | XML only |
| Port | 443 (HTTPS) | 830 |
| Data model | YANG | YANG |
| Use case | Modern APIs, web-friendly | Traditional network automation |

> [!important] CCNA exam tip
> - **JSON**, **HTTPS**, **REST** → RESTCONF
> - **SSH port 830**, **XML only** → NETCONF

---

## 3. Data Formats

| Format | Human-readable? | Structured? | Common use |
|--------|----------------|-------------|------------|
| **JSON** | ✅ | ✅ | REST APIs, modern automation |
| **XML** | ✅ | ✅ | NETCONF, legacy SOAP APIs |
| **YAML** | ✅ | ✅ | Ansible playbooks, config files |
| **Protobuf** | ❌ | ✅ | gRPC telemetry (very high speed) |

**JSON example — interface data:**

```json
{
  "ietf-interfaces:interface": {
    "name": "GigabitEthernet0/0",
    "description": "To CSW1",
    "enabled": true,
    "ietf-ip:ipv4": {
      "address": [
        {
          "ip": "10.0.0.1",
          "prefix-length": 30
        }
      ]
    }
  }
}
```

---

## 4. Configuration Management Tools

| Tool | Language | Agent-based? | Push/Pull |
|------|----------|-------------|-----------|
| **Ansible** | YAML (playbooks) | ❌ Agentless (SSH) | Push |
| **Puppet** | DSL (Ruby-based) | ✅ Agent | Pull |
| **Chef** | Ruby DSL | ✅ Agent | Pull |
| **SaltStack** | YAML | Optional | Push |

> [!important] CCNA exam tip
> Ansible is **agentless** and uses **SSH**. Puppet and Chef use agents.

---

## 5. Attempted RESTCONF on R1 (PT Rejects Everything)

### Commands attempted

```cisco
ip http server
ip http secure-server
ip http authentication local
restconf
```

### PT behavior

**All commands were rejected** by Packet Tracer. Even the basic `ip http server` command returned an invalid input error in this PT version.

> [!warning] This is a hard PT limitation
> Unlike other PT quirks in this lab (where *some* commands worked and only specific ones failed), automation is **completely unsupported**. There is no workaround within PT.

### What works on real IOS-XE

On a real Cisco IOS-XE device (CSR1000v, ISR 4000, Catalyst 9000):

```cisco
ip http server
ip http secure-server
ip http authentication local
restconf
```

RESTCONF then listens on **HTTPS port 443**. You can hit it with `curl`, Postman, or Python:

```bash
curl -k -u admin:Cisco123! \
  -H "Accept: application/yang-data+json" \
  https://10.0.0.1/restconf/data/ietf-interfaces:interfaces
```

---

## 6. Python Scripts (Real-World Reference)

These scripts run **on your PC**, not inside PT. They target a real IOS-XE device.

### 6.1 GET — Retrieve all interfaces

```python
#!/usr/bin/env python3
"""
RESTCONF GET — retrieve all interfaces from a Cisco IOS-XE device.
"""

import requests
import json
from requests.auth import HTTPBasicAuth

DEVICE   = "10.0.0.1"
USERNAME = "admin"
PASSWORD = "Cisco123!"

URL = f"https://{DEVICE}/restconf/data/ietf-interfaces:interfaces"

HEADERS = {
    "Accept": "application/yang-data+json",
    "Content-Type": "application/yang-data+json",
}

response = requests.get(
    URL,
    headers=HEADERS,
    auth=HTTPBasicAuth(USERNAME, PASSWORD),
    verify=False,   # lab only — disable cert verification
    timeout=10,
)

print(f"Status Code: {response.status_code}")
print(json.dumps(response.json(), indent=2))
```

### 6.2 PUT — Change an interface description

```python
#!/usr/bin/env python3
"""
RESTCONF PUT — set the description on GigabitEthernet0/0.
"""

import requests
from requests.auth import HTTPBasicAuth

DEVICE   = "10.0.0.1"
USERNAME = "admin"
PASSWORD = "Cisco123!"

URL = f"https://{DEVICE}/restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet0%2F0"

HEADERS = {
    "Accept": "application/yang-data+json",
    "Content-Type": "application/yang-data+json",
}

PAYLOAD = {
    "ietf-interfaces:interface": {
        "name": "GigabitEthernet0/0",
        "description": "Managed by Python RESTCONF",
        "type": "iana-if-type:ethernetCsmacd",
        "enabled": True,
    }
}

response = requests.put(
    URL,
    headers=HEADERS,
    auth=HTTPBasicAuth(USERNAME, PASSWORD),
    json=PAYLOAD,
    verify=False,
    timeout=10,
)

print(f"Status Code: {response.status_code}")
```

### 6.3 Install prerequisites

```bash
pip install requests
```

---

## 7. Ansible Playbook (Real-World Reference)

Ansible is **agentless** — uses SSH to push config from YAML playbooks.

### 7.1 Inventory — `hosts.ini`

```ini
[routers]
R1 ansible_host=10.0.0.1

[routers:vars]
ansible_network_os=ios
ansible_user=admin
ansible_password=Cisco123!
ansible_connection=network_cli
```

### 7.2 Playbook — `configure_ospf.yml`

```yaml
---
- name: Configure OSPF on R1
  hosts: routers
  gather_facts: no

  tasks:
    - name: Enable OSPF process 1
      cisco.ios.ios_config:
        lines:
          - router ospf 1
          - router-id 10.255.255.254
          - network 10.0.0.0 0.0.0.3 area 0
          - default-information originate

    - name: Verify OSPF neighbors
      cisco.ios.ios_command:
        commands:
          - show ip ospf neighbor
      register: ospf_output

    - name: Print neighbors
      debug:
        var: ospf_output.stdout_lines
```

### 7.3 Run

```bash
ansible-playbook -i hosts.ini configure_ospf.yml
```

> [!important] The core idea of Ansible
> **Declare desired state**, not commands. The playbook runs identically whether you have 1 device or 1000.

---

## 8. SDN & Cisco DNA Center

### 8.1 DNA Center features

| Feature | Purpose |
|---------|---------|
| **Design** | Network hierarchy, site profiles, device templates |
| **Policy** | Group-based access (e.g., "Contractors can't reach HR") |
| **Provision** | Push config to devices via templates |
| **Assurance** | Analytics, health scores, AI/ML-driven insights |
| **Platform** | Northbound REST API for third-party apps |

### 8.2 Architecture

```
  [ App / Script ]
        ↓  Northbound API (REST, JSON)
  [ DNA Center Controller ]
        ↓  Southbound API (NETCONF / RESTCONF / SNMP)
  [ Switches / Routers / APs ]
```

**Key exam points:**
- DNA Center sits **above** the network.
- **Northbound** = REST API for apps (external automation).
- **Southbound** = NETCONF/RESTCONF to devices (internal automation).
- Uses **intent-based networking** — you say *what* you want, controller figures out *how*.

### 8.3 Underlay vs. Overlay

| Layer | Purpose |
|-------|---------|
| **Underlay** | Physical connectivity (routers, switches, OSPF/BGP) |
| **Overlay** | Virtual topology on top (VXLAN, LISP, SD-Access) |

Our lab builds the **underlay**. DNA Center / SD-Access would build the overlay.

---

## 9. Phase 10 Checkpoint

- [x] Understand traditional vs. controller-based networking
- [x] Know HTTP verbs and status codes
- [x] Understand RESTCONF vs. NETCONF
- [x] Know JSON, XML, YAML differences
- [x] Know Ansible (agentless) vs. Puppet/Chef (agent-based)
- [x] Understand DNA Center northbound/southbound
- [x] Attempted automation commands on R1 (PT rejected all)
- [x] Reviewed Python scripts
- [x] Reviewed Ansible playbook
- [x] Understand real automation requires CML / DevNet Sandbox / real gear

---

## 10. Troubleshooting Encounters

### 10.1 All automation commands rejected by Packet Tracer

**Symptom:** On R1, all of these commands return `% Invalid input detected`:

```
ip http server
ip http secure-server
ip http authentication local
restconf
```

**Cause:** Packet Tracer's 2911 IOS emulation does not include the HTTP server or RESTCONF subsystems.

**Impact:** No automation can be demonstrated in PT.

**Workaround:** Use one of:

| Option | Cost | Notes |
|--------|------|-------|
| **Cisco DevNet Sandbox** | Free | Real IOS-XE, always-on, reserves for hours |
| **Cisco CML (formerly VIRL)** | Paid | Full IOS-XE images |
| **EVE-NG / GNS3** | Free + image | Requires a legal IOS-XE image |
| **CSR1000v (Cisco)** | Free trial | Runs on virtualization platforms |

**On real IOS-XE:** All four commands work. RESTCONF listens on HTTPS 443.

---

## 11. Common Misconceptions (Exam Traps)

| Misconception | Reality |
|---------------|---------|
| "REST is a protocol" | REST is an architectural **style** |
| "NETCONF uses HTTPS" | NETCONF uses **SSH on port 830** |
| "Ansible uses agents" | Ansible is **agentless** — uses SSH |
| "DNA Center is on the device" | DNA Center is a **central controller** |
| "Northbound API talks to devices" | Northbound talks to **apps**; southbound talks to devices |
| "XML is JSON's successor" | Parallel formats; JSON is more common now |
| "You can automate IOS via Packet Tracer" | ❌ PT has no automation support |

---

## 12. Lessons Learned

> [!note] Packet Tracer is for CCNA routing & switching, not automation
> PT excels at VLANs, STP, OSPF, ACLs, and services. For automation, you must move to a real or virtual IOS-XE environment.

> [!note] Automation concepts are still exam-critical
> The CCNA exam tests RESTCONF vs. NETCONF, HTTP verbs, JSON/XML/YAML, Ansible vs. Puppet/Chef, and DNA Center — all conceptual. This phase delivers those.

> [!note] Combine PT + DevNet Sandbox for full coverage
> Use PT for the routing/switching/security labs, and DevNet Sandbox for a real RESTCONF/Ansible experience.

---

## 13. Next Phase

➡️ **[Phase 11 — Troubleshooting Scenarios + Final Review](Phase-11-Troubleshooting.md)**

In Phase 11 we will:
- Inject 8 targeted misconfigurations into the lab
- Diagnose and fix each one systematically
- Provide a final verification checklist for the entire project

---

*Document maintained as part of the CCNA Master Lab project. For questions, corrections, or contributions, see the repository README.*