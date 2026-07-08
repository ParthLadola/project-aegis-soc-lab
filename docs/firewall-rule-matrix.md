# Firewall Rule Matrix (D3)

pfSense enforces **default-deny** on every internal interface. Every flow below is an explicit, named allow rule; everything not listed is dropped by the catch-all. Rule order matters (pfSense evaluates top-down, first match wins), so per-interface ordering is noted where it changed behavior.

## Inter-zone allow matrix

| From | To | Ports / Protocol | Why |
|---|---|---|---|
| MGMT / SOC (10) | Any | All | Admin + SOC need full reach to manage and monitor the estate. |
| SERVERS (20) | Internet | HTTP/HTTPS/DNS/NTP | Updates and time sync only. |
| SERVERS (20) | Wazuh (10.10.10.20) | 1514 / 1515 / 55000 | Agent enrolment + event telemetry. |
| USERS (30) | Domain Controller (20) | DNS 53, Kerberos 88, LDAP 389, SMB 445, 464, RPC 135, dynamic RPC 49152–65535, GC 3268, NTP 123 | Domain membership essentials — scoped to the DC via a port alias, nothing else on VLAN 20. |
| USERS (30) | Wazuh (10.10.10.20) | 1514 / 1515 | Agent telemetry (added as a scoped rule so the workstation could report in). |
| USERS (30) | *Other VLANs* | **DENY** | Explicit block above the internet-allow rule so users can't reach other segments. |
| USERS (30) | Internet | HTTP/HTTPS | Normal browsing. |
| QUARANTINE (99) | Everything | **DENY + LOG** | Isolated by default. Deny is logged; those logs become SIEM detections. A single temporary rule is added only during a controlled test, then removed. |
| VPN (WireGuard) | MGMT / SOC (10) | scoped | Remote admin reaches the management plane only — never the server or user VLANs. |
| WAN | pfSense WAN : 51820 | UDP | WireGuard handshake inbound (paired with router port-forward). |

## Rule-ordering notes (lessons from the build)

- **VLAN 30 → DC scoping:** the specific "allow only these DC ports" rule had to sit **above** the broad "allow internet" rule, or the broad rule would match first and expose the whole DC. Order: (1) allow DC-specific ports → (2) allow Wazuh ports → (3) deny other VLANs → (4) allow internet.
- **QUARANTINE deny logging** is enabled deliberately — the blocked traffic from the adversary host is itself evidence in the SIEM.
- **Least-privilege VPN:** the WireGuard interface rule permits the tunnel into MGMT/SOC only. Even a compromised admin laptop cannot reach the servers directly.
