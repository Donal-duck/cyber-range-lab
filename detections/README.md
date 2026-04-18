# Detections

Detection content developed against the attack playbooks in this lab.

## Format
- **Sigma rules** (`.yml`) — vendor-neutral detection format
- **Wazuh custom rules** — `local_rules.xml` snippets
- **Suricata rules** *(planned)* — for network-layer detection

## Planned Coverage

| Attack Playbook | Detection Approach | Status |
|-----------------|--------------------|--------|
| Kerberoast | Event ID 4769 with unusual encryption type (RC4) | Planned |
| AS-REP Roast | Event ID 4768 with no pre-auth | Planned |
| BloodHound enumeration | LDAP query rate anomaly | Planned |
| Responder poisoning | LLMNR/NBT-NS traffic on unexpected hosts | Planned |
| DCSync | Directory replication from non-DC source | Planned |

Detections are written after the attack is successfully executed and its telemetry observed.
