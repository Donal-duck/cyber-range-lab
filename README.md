# Cyber Range Lab

> A segmented, enterprise-style security research environment built for offensive security skill development, detection engineering, and adversary simulation.

![Status](https://img.shields.io/badge/status-active%20build-orange)
![Focus](https://img.shields.io/badge/focus-offensive%20security-red)
![Stack](https://img.shields.io/badge/stack-pfSense%20%7C%20Proxmox%20%7C%20AD%20%7C%20Wazuh-blue)

---

## Project Goal

Build and operate a self-contained adversary simulation range that mirrors a small enterprise network. The lab supports controlled offensive operations against an intentionally misconfigured Active Directory forest, with all activity observable through a production-grade detection stack.

The objective is not to "have a lab" — it is to develop, document, and validate offensive tradecraft against realistic defensive telemetry, producing portfolio artefacts that demonstrate both attack execution and defender awareness.

## Architecture Overview

Four-zone network segmentation enforced by a pfSense stateful firewall, following zero-trust principles between zones:

| Zone | Subnet | Purpose | Trust Level |
|------|--------|---------|-------------|
| **WAN** | `192.168.0.99` | Internet uplink | Untrusted |
| **LAN** | `10.10.10.0/24` | Red team workstations (Kali, Windows 11) | Operator |
| **OPT1** | `10.10.1.0/24` | Management plane (Proxmox, pfSense) | Restricted |
| **OPT2** | `10.10.20.0/24` | Victim zone (Active Directory, Juice Shop) | Hostile |
| **OPT3** | `10.10.30.0/24` | *(Planned)* Monitoring plane (Wazuh SIEM) | Protected |

> See [`docs/01-network-architecture.md`](docs/01-network-architecture.md) for the full design rationale and traffic flow diagrams.

## Hardware Inventory

| Node | Role | Specs | OS |
|------|------|-------|-----|
| Node 1 | Hypervisor | Dell OptiPlex, 32 GB RAM, 2 TB storage, 5× Ethernet | Proxmox VE |
| Node 2 | Red team workstation | Lenovo ThinkPad | Kali Linux |
| Node 3 | User endpoint / victim | Gigabyte PC | Windows 11 |
| Switch | L2 connectivity | Netgear 8-port unmanaged | — |

### Virtualised Workloads on Node 1

- **pfSense** — perimeter firewall, inter-VLAN routing
- **Windows Server (AD DS)** — domain controller for `corp.lab`
- **Windows Enterprise** × 2 — domain-joined workstations
- **Ubuntu** — OWASP Juice Shop (web application attack surface)

## Skills Demonstrated

### Network & Infrastructure Security
- Network segmentation design with zero-trust inter-zone policies
- Stateful firewall rule authoring and audit (pfSense)
- Hypervisor-based lab virtualisation (Proxmox VE)
- Management plane isolation from production workloads

### Offensive Security *(in progress)*
- Active Directory enumeration and exploitation
- Kerberoasting, AS-REP Roasting, ACL abuse
- Command-and-control operations (Sliver)
- MITRE ATT&CK-aligned adversary simulation

### Detection Engineering *(planned)*
- SIEM deployment and log pipeline design (Wazuh)
- Endpoint telemetry via Sysmon
- Network IDS/IPS (Suricata on pfSense)
- Sigma rule authoring

### Engineering Discipline
- Architecture Decision Records (ADRs)
- Infrastructure-as-Code (planned: Ansible, currently Bash)
- Change management with backup/rollback discipline

## Build Progress

- [x] Hardware procurement and physical cabling
- [x] Proxmox installation and storage configuration
- [x] pfSense deployment and interface assignment
- [x] Initial VM deployment (AD, Win11, Juice Shop, Kali)
- [x] Portfolio repository initialised
- [ ] **Firewall rule hardening** *(in progress — see [ADR-001](docs/decisions/ADR-001-network-segmentation.md))*
- [ ] Active Directory forest population with realistic misconfigurations
- [ ] WireGuard VPN for secure remote access (replacing current temporary solution)
- [ ] Wazuh SIEM deployment on dedicated monitoring VLAN
- [ ] Sysmon + Windows Event Forwarding
- [ ] First end-to-end attack chain with detection validation
- [ ] Suricata IDS deployment

## Known Limitations & Technical Debt

Honest documentation of current limitations is a deliberate engineering practice. These are tracked for remediation:

| Item | Current State | Planned Remediation |
|------|---------------|---------------------|
| Remote access method | Chrome Remote Desktop (outbound-tunnel, third-party broker) | WireGuard VPN terminating on pfSense |
| Firewall rules | Permissive legacy defaults | Zero-trust explicit-allow (ADR-001) |
| Automation | Manual GUI provisioning | Ansible playbooks for VM configuration |
| Monitoring | None | Wazuh SIEM on isolated VLAN |

## Repository Structure

```
.
├── README.md                    # This file
├── docs/
│   ├── 01-network-architecture.md
│   ├── 02-firewall-rules.md     # Before/after rule analysis
│   └── decisions/               # Architecture Decision Records
├── diagrams/                    # Network topology, data flow diagrams
├── scripts/                     # Bash automation (IaC)
├── attack-playbooks/            # MITRE ATT&CK-mapped attack chains
└── detections/                  # Sigma rules, Wazuh custom rules
```

## Author

CS student pursuing offensive security. This lab is an ongoing build — commits reflect the real progression of the project, not a retrospective write-up.

## License

MIT — see [LICENSE](LICENSE)
