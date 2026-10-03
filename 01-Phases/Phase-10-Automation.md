# Phase 10 — Basic Automation & Programmability

> [!NOTE]
> **Prerequisites & Navigation**
> **Previous:** [Phase 9 — Services](Phase-09-Services.md) · **Next:** [Phase 11 — Troubleshooting](Phase-11-Troubleshooting.md)
> Master addressing plan: [MASTER-ADDRESSING.md](../MASTER-ADDRESSING.md)

> [!TIP]
> **Phase Objective**
> Understand and document the automation concepts tested on the CCNA 200-301 exam:
> REST APIs, RESTCONF, NETCONF, HTTP verbs, data formats, Python scripting,
> configuration management tools, and SDN / Cisco Catalyst Center.

> [!WARNING]
> Packet Tracer has no automation support
> PT rejects `ip http server`, `ip http secure-server`, `ip http authentication local`,
> and `restconf`. This phase is **documented, not implemented**.

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
| **Cisco product** | N/A | Catalyst Center (formerly DNA Center) |

**Key exam concepts:**
- **Control plane** decides how to forward (routing protocols, ARP, STP).
- **Data plane** actually forwards packets (ASICs, hardware).
- **Management plane** configures and monitors (SSH, SNMP, Syslog).
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
| Use case | Modern APIs | Traditional automation |

> [!TIP]
> **CCNA exam quick-reference
**
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

> [!IMPORTANT]
> Ansible is agentless
> Ansible uses SSH only. Puppet and Chef require an agent on each device.

---

## 5. Attempted RESTCONF on R1 (PT Rejects Everything)

```cisco
ip http server
ip http secure-server
ip http authentication local
restconf
```

**PT behavior:** All commands rejected.

> [!WARNING]
> Hard PT limitation
> Unlike other PT quirks, automation is **completely unsupported**. There is no workaround in PT.

### On real IOS-XE

```cisco
ip http server
ip http secure-server
ip http authentication local
restconf
```

RESTCONF listens on **HTTPS port 443**. Test with:

```bash
curl -k -u admin:<LAB_PASSWORD> \
  -H "Accept: application/yang-data+json" \
  https://10.0.0.1/restconf/data/ietf-interfaces:interfaces
```

---

## 6. Python Scripts (Reference)

Illustrative RESTCONF examples. These target a real IOS-XE device (CSR1000v,
ISR 4000, Catalyst 9000) — not Packet Tracer.

### 6.1 GET — retrieve all interfaces

```python
import json
import requests
from requests.auth import HTTPBasicAuth

DEVICE   = "10.0.0.1"
USERNAME = "admin"
PASSWORD = "<LAB_PASSWORD>"

URL = f"https://{DEVICE}/restconf/data/ietf-interfaces:interfaces"

HEADERS = {
    "Accept": "application/yang-data+json",
    "Content-Type": "application/yang-data+json",
}

response = requests.get(
    URL,
    headers=HEADERS,
    auth=HTTPBasicAuth(USERNAME, PASSWORD),
    verify=False,   # lab only
    timeout=10,
)

print(f"Status Code: {response.status_code}")
print(json.dumps(response.json(), indent=2))


## 7. Ansible Playbook (Reference)

Reference inventory and playbook are inlined below.

Ansible is **agentless** — uses SSH to push config from YAML playbooks.

ansible-playbook -i hosts.ini configure_ospf.yml
```

---

## 8. SDN & Cisco Catalyst Center

> [!NOTE]
> **Renamed product
**
> Cisco DNA Center is now called **Cisco Catalyst Center** (rebranded 2024).
> Legacy documentation may use "DNA Center".

### 8.1 Features

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
  [ Catalyst Center Controller ]
        ↓  Southbound API (NETCONF / RESTCONF / SNMP)
  [ Switches / Routers / APs ]
```

**Key exam points:**
- Catalyst Center sits **above** the network.
- **Northbound** = REST API for apps (external automation).
- **Southbound** = NETCONF/RESTCONF to devices (internal automation).
- Uses **intent-based networking** — you say *what* you want, controller figures out *how*.

### 8.3 Underlay vs. Overlay

| Layer | Purpose |
|-------|---------|
| **Underlay** | Physical connectivity (routers, switches, OSPF/BGP) |
| **Overlay** | Virtual topology on top (VXLAN, LISP, SD-Access) |

Our lab builds the **underlay**. Catalyst Center / SD-Access would build the overlay.

---

## 9. Phase 10 Checkpoint

- [x] Understand traditional vs. controller-based networking
- [x] Know HTTP verbs and status codes
- [x] Understand RESTCONF vs. NETCONF
- [x] Know JSON, XML, YAML differences
- [x] Know Ansible (agentless) vs. Puppet/Chef (agent-based)
- [x] Understand Catalyst Center northbound/southbound
- [x] Attempted automation commands on R1 (PT rejected all)
- [x] Reviewed Python scripts in `scripts/`
- [x] Reviewed Ansible playbook in `scripts/ansible/`
- [x] Understand real automation requires CML / DevNet Sandbox / real gear

---

## 10. Common Misconceptions (Exam Traps)

| Misconception | Reality |
|---------------|---------|
| "REST is a protocol" | REST is an architectural **style** |
| "NETCONF uses HTTPS" | NETCONF uses **SSH on port 830** |
| "Ansible uses agents" | Ansible is **agentless** — uses SSH |
| "Catalyst Center is on the device" | Catalyst Center is a **central controller** |
| "Northbound API talks to devices" | Northbound talks to **apps**; southbound talks to devices |
| "You can automate IOS via Packet Tracer" | ❌ PT has no automation support |

---

## 11. Next Phase

➡️ **[Phase 11 — Troubleshooting](Phase-11-Troubleshooting.md)**

---

*Phase 10 of the CCNA Master Lab project. See [README](../README.md) for the full phase list.*