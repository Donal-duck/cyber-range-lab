# Network Architecture

## Topology

```
                         ┌──────────────────┐
                         │   Internet (WAN) │
                         │   192.168.0.99   │
                         └────────┬─────────┘
                                  │
                         ┌────────┴─────────┐
                         │     pfSense      │
                         │  (Stateful FW)   │
                         └──┬────┬────┬─────┘
                            │    │    │
              ┌─────────────┘    │    └──────────────┐
              │                  │                   │
         ┌────┴─────┐      ┌─────┴─────┐      ┌─────┴─────┐
         │   LAN    │      │   OPT1    │      │   OPT2    │
         │10.10.10/24      │10.10.1/24 │      │10.10.20/24│
         │          │      │           │      │           │
         │ ┌──────┐ │      │ ┌───────┐ │      │ ┌───────┐ │
         │ │ Kali │ │      │ │Proxmox│ │      │ │   AD  │ │
         │ └──────┘ │      │ └───────┘ │      │ └───────┘ │
         │ ┌──────┐ │      │ ┌───────┐ │      │ ┌───────┐ │
         │ │Win 11│ │      │ │pfSense│ │      │ │ Juice │ │
         │ └──────┘ │      │ │ (mgmt)│ │      │ │  Shop │ │
         │          │      │ └───────┘ │      │ └───────┘ │
         └──────────┘      └───────────┘      └───────────┘
           OPERATOR          MANAGEMENT           VICTIM
            ZONE               PLANE               ZONE
```

## Zone Definitions

### WAN (Wide Area Network)
- **Purpose:** Internet uplink for operator tooling updates, hypervisor package repositories
- **Trust:** Untrusted
- **Policy:** Bogon networks blocked; stateful outbound connections permitted; no inbound initiated traffic

### LAN — Operator Zone (`10.10.10.0/24`)
- **Purpose:** Hosts the red team workstation (Kali Linux) and a domain-joined Windows 11 endpoint
- **Trust:** Operator-level — trusted to perform authorised actions, but not trusted to reach the management plane
- **Residents:**
  - Kali Linux — offensive tooling, C2 server (future: Sliver)
  - Windows 11 — domain-joined simulated user endpoint

### OPT1 — Management Plane (`10.10.1.0/24`)
- **Purpose:** Hosts the hypervisor management interface and the firewall itself
- **Trust:** Highest — compromise here equals total lab compromise
- **Residents:**
  - Proxmox VE web UI and SSH
  - pfSense management interface
- **Ingress:** Restricted to authenticated admin access only (out-of-band or explicit rule)
- **Egress:** To WAN for package updates only

### OPT2 — Victim Zone (`10.10.20.0/24`)
- **Purpose:** Target surface for offensive operations — intentionally misconfigured
- **Trust:** Hostile — assume every host here is or will be compromised
- **Residents:**
  - Windows Server (Active Directory Domain Controller for `corp.lab`)
  - Ubuntu VM running OWASP Juice Shop
- **Ingress:** From LAN (attack simulation)
- **Egress:** To LAN only (deliberate C2 callback path); no WAN access

### OPT3 — Monitoring Plane *(planned)* (`10.10.30.0/24`)
- **Purpose:** SIEM and security telemetry aggregation
- **Trust:** Protected — receives data from all zones but initiates connections to none
- **Planned residents:**
  - Wazuh Manager
  - Log storage

## Physical Layout

```
[ ISP Router (192.168.0.1) ]
         │
         └── [ Dell OptiPlex — NIC0 (WAN) ]
              │
              │ pfSense VM with 5 vNICs mapped to physical ports
              │
              ├── NIC1 ──► [ Netgear 8-port switch ] ──► Kali, Win 11
              ├── NIC2 ──► Proxmox mgmt (internal vmbr)
              ├── NIC3 ──► AD + Juice Shop (internal vmbr)
              └── NIC4 ──► (free — reserved for OPT3 SIEM VLAN)
```

## Design Principles

1. **Segmentation by trust, not by function.** Zones are defined by how much they are trusted, not what kind of machine lives there.
2. **Management plane isolation.** The hypervisor and firewall management interfaces are never reachable from workload zones.
3. **Victim outbound isolation.** Intentionally vulnerable systems cannot reach the internet, ensuring that accidental real-malware detonation cannot escape the lab.
4. **Observable attack paths.** Every permitted inter-zone flow is intentional and will be monitored once the SIEM is deployed.
5. **Documented exceptions only.** No "convenience" rules. If a flow is needed, it is added explicitly and recorded in an ADR.

## Current vs Target State

See [`02-firewall-rules.md`](02-firewall-rules.md) for the detailed before/after firewall rule analysis and migration plan.
