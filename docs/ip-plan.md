# IP Addressing Plan & VLAN Table (D2)

## VLAN / subnet plan

| VLAN | Zone | Subnet | Gateway (pfSense) | DHCP | DNS handed out |
|---|---|---|---|---|---|
| 10 | MGMT / SOC | 10.10.10.0/24 | 10.10.10.1 | Kea, scoped | — |
| 20 | SERVERS | 10.10.20.0/24 | 10.10.20.1 | Kea, scoped | 10.10.20.10 (DC) |
| 30 | USERS | 10.10.30.0/24 | 10.10.30.1 | Kea, scoped | 10.10.20.10 (DC) |
| 99 | QUARANTINE | 10.10.99.0/24 | 10.10.99.1 | Kea, scoped | — |

## Host addressing

| Host | VLAN | IP | Assignment | Role |
|---|---|---|---|---|
| pfSense (gateways) | all | .1 per VLAN | static | Router / firewall / DHCP / DNS forwarder |
| Wazuh SIEM | 10 | 10.10.10.20 | static | Manager + Indexer + Dashboard |
| Windows Server DC | 20 | 10.10.20.10 | static | AD DS, DNS, `aegis.lab` |
| Ubuntu app server | 20 | 10.10.20.20 | static | nginx + SSH (key-only) |
| Windows 10 client | 30 | 10.10.30.100 | DHCP | Domain-joined workstation |
| Kali adversary | 99 | 10.10.99.100 | DHCP | Isolated test host |

## Notes

- **Static vs. DHCP:** infrastructure (DC, Linux server, Wazuh, gateways) is statically addressed; endpoints (Win10 client, Kali) pull from DHCP. The Kea DHCP backend replaced the deprecated ISC engine; because Kea doesn't do static mapping the same way, static IPs are set **inside each server VM** rather than via DHCP reservation.
- **DNS:** VLANs 20 and 30 receive the Domain Controller (10.10.20.10) as their DNS server so `aegis.lab` resolves; the DC forwards external queries to the pfSense resolver.
- **WAN:** pfSense WAN pulls DHCP from the home router (192.168.1.0/24 upstream, gateway 192.168.1.254). The physical NIC never carries a VLAN tag — all tagging lives on the `vmbr1` VLAN-aware bridge between the VMs and pfSense.
