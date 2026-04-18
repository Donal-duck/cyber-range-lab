# Firewall Rules — Before / After Analysis

This document captures the firewall rule state of the lab at two points in time: the initial permissive configuration ("before") and the zero-trust hardened configuration ("after"), along with the migration process used to transition between them without loss of remote access.

This is a deliberate documentation practice: capturing not just the final state but the reasoning and process that led to it. In enterprise operations, this is the equivalent of a change record plus post-change review.

---

## Before: Initial Permissive Configuration

The lab was initially brought online with rules that prioritised connectivity over security, a common pattern during early build phases.

### WAN
| # | Action | Proto | Source | Dest | Description |
|---|--------|-------|--------|------|-------------|
| — | BLOCK | — | Bogon networks | * | Default pfSense bogon block |

### LAN
| # | Action | Proto | Source | Dest | Port | Description |
|---|--------|-------|--------|------|------|-------------|
| 1 | PASS | TCP | * | LAN address | 80 | Anti-Lockout Rule (auto) |
| 2 | PASS | IPv4 * | LAN address | * | * | (permissive artefact) |
| 3 | PASS | IPv4 * | LAN subnets | * | * | Default allow LAN to any |

### OPT1
| # | Action | Proto | Source | Dest | Description |
|---|--------|-------|--------|------|-------------|
| 1 | PASS | IPv4 * | OPT1 address | * | (permissive artefact) |

### OPT2
| # | Action | Proto | Source | Dest | Description |
|---|--------|-------|--------|------|-------------|
| 1 | PASS | IPv4 * | OPT2 subnets | OPT1 subnets | Victim to Management |
| 2 | PASS | IPv4 * | OPT1 subnets | LAN subnets | Misplaced rule (OPT1 src on OPT2 tab) |
| 3 | PASS | IPv4 * | OPT2 subnets | * | Victim to Any (including WAN) |

### Problems With This Configuration

1. **Management plane exposure.** OPT2 rule 1 explicitly permits intentionally vulnerable hosts to reach the hypervisor management network. A compromise of the Active Directory VM could be used to attack Proxmox directly.

2. **Misplaced rule.** OPT2 rule 2 has `OPT1 subnets` as the source but sits on the OPT2 interface tab. pfSense evaluates rules on the interface the traffic *arrives on*, so this rule is effectively dead — it will never match, because OPT1-sourced traffic arrives on the OPT1 interface, not OPT2. This indicates rule drift without a clear model of pfSense rule semantics.

3. **Unrestricted victim egress.** OPT2 rule 3 allows victim VMs to reach the internet. If real malware were accidentally detonated, it would beacon out successfully. This undermines both safety and the meaningfulness of future detection work.

4. **No explicit deny.** Absence of a default-deny rule means pfSense's implicit default-deny applies, which works, but the *explicit* version forces the rule author to justify every flow and produces cleaner logs.

---

## After: Zero-Trust Hardened Configuration

Rules are designed per ADR-001. Every rule is justified by a legitimate operational need.

### LAN (Operator Zone — Kali + Windows 11)
| # | Action | Proto | Source | Port | Dest | Port | Description |
|---|--------|-------|--------|------|------|------|-------------|
| 1 | PASS | TCP | * | * | LAN address | 443, 80 | Anti-Lockout Rule (do not remove) |
| 2 | PASS | * | LAN net | * | LAN net | * | Intra-LAN (Kali to Win 11) |
| 3 | PASS | * | LAN net | * | OPT2 net | * | Red team to Victims (attack path) |
| 4 | BLOCK | * | LAN net | * | OPT1 net | * | No pivot to management plane |
| 5 | PASS | * | LAN net | * | ! RFC1918 | * | Internet egress (NOT private nets) |
| 6 | BLOCK | * | LAN net | * | * | * | Default deny (explicit) |

### OPT2 (Victim Zone — AD + Juice Shop)
| # | Action | Proto | Source | Port | Dest | Port | Description |
|---|--------|-------|--------|------|------|------|-------------|
| 1 | PASS | * | OPT2 net | * | OPT2 net | * | Intra-zone (AD replication etc.) |
| 2 | BLOCK | * | OPT2 net | * | OPT1 net | * | No pivot to management |
| 3 | PASS | * | OPT2 net | * | LAN net | * | C2 callback (intentional) |
| 4 | BLOCK | * | OPT2 net | * | * | * | Default deny — includes WAN |

### OPT1 (Management Plane — Proxmox + pfSense)
| # | Action | Proto | Source | Port | Dest | Port | Description |
|---|--------|-------|--------|------|------|------|-------------|
| 1 | PASS | * | OPT1 net | * | ! RFC1918 | * | Hypervisor updates only |
| 2 | BLOCK | * | OPT1 net | * | * | * | Default deny |

### Key Enterprise Patterns Applied

- **Default deny with explicit allow:** Every zone ends with a documented block-all rule.
- **Directional trust asymmetry:** The management plane can reach the internet for updates but cannot initiate to workloads.
- **Negated destination on internet egress:** Using `! RFC1918` instead of `any` for internet rules ensures that zones cannot reach private networks not explicitly whitelisted.
- **Documented attack path:** The OPT2 to LAN rule is labelled as the intentional C2 callback channel.

---

## Migration Procedure

### Pre-Change Controls
1. Full pfSense configuration backup taken and stored outside the lab
2. Console access to the Dell hypervisor verified
3. pfSense console menu option 4 confirmed as last-resort rollback

### Change Sequence

Rules are applied bottom-up: add the new rules above the existing permissive rule, test, then remove the old rule.

**Order across tabs (least risky first):**
1. **OPT2** — no risk to operator access
2. **OPT1** — no risk to operator access
3. **LAN** — operator sits here; done last with greatest caution

### Post-Change Validation

- [ ] Kali can reach OPT2 hosts (attack path intact)
- [ ] Kali can reach internet (operator tooling and remote access intact)
- [ ] Kali cannot reach Proxmox management UI (segmentation effective)
- [ ] OPT2 AD cannot reach 8.8.8.8 (victim outbound isolated)
- [ ] OPT2 AD cannot reach Proxmox (victim segmentation effective)
- [ ] Post-change configuration backup saved

---

## Lessons / Reflection

*To be completed after migration.*
