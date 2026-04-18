# ADR-001: Zero-Trust Network Segmentation

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | 2026-04-18 |
| **Deciders** | Lab owner |
| **Related** | [`docs/02-firewall-rules.md`](../02-firewall-rules.md) |

## Context

The lab initially used pfSense default firewall rules after interface creation. On LAN, pfSense auto-generates a permissive "allow LAN to any" rule. As additional interfaces (OPT1, OPT2) were added, allow-any rules were copied across tabs to maintain connectivity during the build phase.

This produced a working network but one with no meaningful trust boundaries between zones. Specifically:

- The red team workstation zone could initiate traffic to the management plane
- The intentionally vulnerable victim zone had unrestricted outbound internet access
- Victim hosts could reach the Proxmox management interface

In an enterprise context, this configuration would fail a baseline segmentation audit.

## Decision

Adopt a **zero-trust, default-deny, explicit-allow** posture between all zones. Every inter-zone traffic flow must be justified, documented, and represent a legitimate operational need.

### Explicit Flow Matrix

Only the following flows are permitted; all others are denied by default.

```
LAN   ──allow──> OPT2     (red team operations)
LAN   ──allow──> LAN      (intra-zone: Kali ↔ Win11)
LAN   ──allow──> WAN      (operator internet access, tooling updates)
LAN   ──DENY──>  OPT1     (no pivot to management plane)

OPT2  ──allow──> OPT2     (intra-zone: AD replication, service dependencies)
OPT2  ──allow──> LAN      (deliberate C2 callback channel)
OPT2  ──DENY──>  OPT1     (compromised victim cannot reach hypervisor)
OPT2  ──DENY──>  WAN      (vulnerable AD must not beacon externally)

OPT1  ──allow──> WAN      (hypervisor package updates)
OPT1  ──DENY──>  LAN      (management plane does not initiate to workloads)
OPT1  ──DENY──>  OPT2     (management plane does not initiate to workloads)
```

## Rationale

### Why block LAN to OPT1?
The management plane is the crown jewel. A compromised red team workstation must not be able to pivot to the hypervisor. This is the direct analogue of isolating vSphere management networks from user VLANs.

### Why block OPT2 to WAN?
The victim zone hosts an intentionally misconfigured Active Directory and a vulnerable web application. Allowing outbound internet from this zone would mean real malware could communicate externally and the detection story for C2 becomes meaningless.

### Why allow OPT2 to LAN?
This is the deliberate attack path. A C2 callback from a compromised victim to the operator workstation must succeed; this is the flow being studied. It is narrow, documented, and observable.

### Why block OPT1 to LAN/OPT2?
Management systems should never initiate connections to workloads. This constraint catches misconfigurations and reduces blast radius.

## Consequences

### Positive
- Lab architecture is defensible in interview discussions as a zero-trust design
- Compromised victims cannot pivot to the hypervisor
- Outbound isolation of vulnerable systems matches real enterprise DMZ practice
- Segmentation makes detection engineering meaningful

### Negative / Accepted Trade-offs
- More complex rule management
- Some operator convenience lost (cannot reach Proxmox UI from Kali)
- Future tooling deployment requires explicit firewall exceptions

### Risks Introduced
- **Lockout risk during rule transition:** Mitigated by config backup, direct console access, and bottom-up rule application with connectivity testing.

## Future Revisions

This ADR will be superseded when:
- A dedicated monitoring VLAN (OPT3) is introduced for Wazuh SIEM
- Remote access is migrated from Chrome Remote Desktop to WireGuard VPN
- Any new zone is added
