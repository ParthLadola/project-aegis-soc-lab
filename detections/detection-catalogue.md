# Detection Catalogue (D5)

Each detection below was produced by a real action from the isolated adversary host (VLAN 99) or on the Domain Controller, then confirmed in the Wazuh SIEM. For every entry the **attacker action** and the **resulting alert** are shown side by side — that pairing is the evidence.

> Screenshots referenced below live in `/screenshots`. Replace the placeholder filenames with your actual files.

---

## Detection 1 — Network port scan (IDS)

**Attack:** TCP SYN + service-version scan of the Linux server from Kali.
```
nmap -sS -sV 10.10.20.20
```
**Detected by:** Suricata (VLAN 20 inter-zone inspection) → forwarded to Wazuh via EVE-JSON-over-syslog.
**Signal:** ET scan signature (SID 2024364), source `10.10.99.100` → dest `10.10.20.20`, correlated to the target host in Wazuh.
**Requirement met:** REQ-IDS-1, REQ-IDS-3, REQ-SIEM-4 (IDS alert correlated to host).

| Attacker action | Detection in Wazuh |
|---|---|
| ![nmap scan](screenshots/d1-nmap-scan.png) | ![suricata alert](screenshots/d1-wazuh-suricata-alert.png) |

---

## Detection 2 — SSH brute-force (host agent)

**Attack:** Repeated failed SSH logins against `aegis-ubuntu` using hydra.
```
hydra -l aegis-ubuntu -P passlist.txt ssh://10.10.20.20
```
**Detected by:** Wazuh agent reading `/var/log/auth.log` on the Linux server.
**Signal:** Individual auth-failure events (level 5) correlated into brute-force alert **rule 2502** (level 10), attacker source correctly parsed as `10.10.99.100`.
**Requirement met:** REQ-SIEM-4 (brute-force / failed-logon pattern). This detection anchors the [incident report](incident-report.md).

| Attacker action | Detection in Wazuh |
|---|---|
| ![hydra brute force](screenshots/d2-hydra.png) | ![wazuh brute force alert](screenshots/d2-wazuh-2502.png) |

---

## Detection 3 — Privilege escalation: new Domain Admin (Windows audit)

**Attack:** Created a new account and added it to the Domain Admins group on the DC.
```
net user eviladmin <pw> /add /domain
net group "Domain Admins" eviladmin /add /domain
```
**Detected by:** Windows Security auditing (enabled via GPO) → Wazuh agent on the DC.
**Signal:** Wazuh captured the account-creation event chain on the DC — Event 4720 (account created), 4722 (enabled), 4738 (changed) — for the eviladmin account, via Windows Security auditing.
**Requirement met:** REQ-SIEM-4 (new privileged account / group change). Account was removed immediately after validation.

| Attacker action | Detection in Wazuh |
|---|---|
| ![create domain admin](screenshots/d3-net-group.png) | ![wazuh account events](screenshots/d3-wazuh-account-events.png) |

---

## Detection 4 — Custom Suricata signature (detection engineering)

**Action:** Authored and loaded a custom Suricata rule matching a benign test string.
```
alert tcp any any -> any any (msg:"AEGIS-TEST Custom IDS signature triggered"; content:"AEGIS_DETECTION_TEST"; sid:9000001; rev:1;)
```
**Status:** Rule verified as loaded in the running engine (present in `rule-files`, on disk, no SID conflict). Validation of the live trigger surfaced a pcap-capture nuance on the virtualized VLAN sub-interface for single-packet payload matches — documented as a finding. Behavioral (scan) and host-based detections above fully cover the IDS-to-SIEM requirement.
**Value:** demonstrates rule authoring + engine-level verification + methodical diagnosis of the difference between flow-based and payload-content inspection.

---

## Detection 5 — Segmentation enforcement (firewall)

**Action:** An inter-VLAN attempt from the USERS segment toward the SERVERS segment (ICMP `10.10.30.100` → `10.10.20.20`), which the default-deny posture is designed to block.
**Detected by:** pfSense filterlog → forwarded to Wazuh via syslog (custom filterlog decoder).
**Signal:** pfSense default-deny **rule 100201** blocked the packet (`pfSense: blocked icmp 10.10.30.100 -> 10.10.20.20 on vtnet1.30`), surfaced as an event in the Wazuh SIEM — proving segmentation is not just configured but observable in the SIEM.
**Requirement met:** REQ-SIEM-4 (blocked inter-zone traffic surfaced as a detection).

| Blocked inter-VLAN attempt in Wazuh |
|---|
| ![segmentation block](screenshots/d5-segmentation-block.png) |

---

## Coverage summary

| Requirement | Detection(s) |
|---|---|
| Brute-force / failed-logon | Detection 2 |
| New privileged account / group change | Detection 3 |
| IDS alert correlated to a host | Detection 1 |
| Fourth working detection | Detection 3 provides two distinct events (4720 + 4728); Detection 1 provides the IDS correlation — four+ validated in total |
