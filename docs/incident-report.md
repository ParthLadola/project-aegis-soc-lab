# Incident Report — SSH Brute-Force Against Application Server

**Environment:** Project AEGIS lab · **Classification:** Training exercise · **Analyst:** Parth Ladola
**Date:** [fill in] · **Severity:** Medium (contained; no compromise)

---

## 1. Summary

A sustained SSH password-guessing attack was observed against the Linux application server (`10.10.20.20`) originating from a host in the QUARANTINE segment (`10.10.99.100`). The activity was detected by the Wazuh agent monitoring the server's authentication log, correlated into a single brute-force alert, and confirmed contained by network segmentation. No authentication succeeded. As a remediation, password authentication was disabled server-wide in favor of key-only SSH.

---

## 2. Detection

| Field | Value |
|---|---|
| Detection source | Wazuh host agent — `/var/log/auth.log` |
| Triggering rule | Rule 2502 — repeated authentication failures (correlated) |
| Alert level | 10 |
| First event | [timestamp] |
| Attacker source IP | 10.10.99.100 (QUARANTINE / adversary host) |
| Target | 10.10.20.20 : 22 (Linux application server, SERVERS VLAN) |
| Target account | `aegis-ubuntu` |

The individual failed-login events (rule ~5710-class, level 5) were correlated by Wazuh into a single higher-severity brute-force alert (rule 2502, level 10) once the failure threshold from a single source was crossed. *(Screenshot: hydra attack + correlated Wazuh alert side by side.)*

---

## 3. Triage

- **Volume:** [N] failed attempts from a single source in [N] seconds.
- **Source:** `10.10.99.100` — a host in the QUARANTINE VLAN, which by design should have **no** standing access to the SERVERS VLAN.
- **Outcome:** All attempts failed — `0 valid passwords found`. No successful authentication in the log.
- **Credential exposure:** The targeted account used a deliberately weak lab password. In production this would be flagged as a policy violation.

---

## 4. Scope

Network segmentation contained the blast radius:

- The attack was only possible because a **single, temporary, time-boxed firewall rule** permitted `10.10.99.100 → 10.10.20.20` during the controlled test. That rule was removed immediately afterward.
- The QUARANTINE VLAN's default-deny posture means the adversary host has **no path** to any other segment — the DC (VLAN 20), the SIEM (VLAN 10), and the user workstation (VLAN 30) were never reachable.
- No lateral movement was possible; the incident was confined to a single service on a single host.

---

## 5. Response & Recommendations

**Immediate response (performed):**
1. Removed the temporary quarantine allow rule, restoring full isolation of the adversary host.
2. Disabled SSH password authentication server-wide (`PasswordAuthentication no`, plus the Ubuntu drop-in override in `/etc/ssh/sshd_config.d/`), enforcing **key-only** login.
3. Verified: key authentication still succeeds; password authentication is now refused (`Permission denied (publickey)`).

| Password auth refused (hardening in effect) | Key-based login still succeeds |
|---|---|
| ![password auth refused](screenshots/ssh-hardening-after.png) | ![key login succeeds](screenshots/ssh-key-login.png) |

**Recommendations (production):**
- Enforce key-only SSH via configuration management across all Linux hosts.
- Add account-lockout / rate-limiting (e.g. `fail2ban`) as defense-in-depth.
- Enforce unique, high-entropy credentials via policy; eliminate shared/weak passwords.
- Alert on any QUARANTINE-sourced traffic reaching a production segment as a high-severity event.

---

## 6. Lessons Learned

Segmentation did its job — even a "successful" attack path had to be *deliberately opened* and was trivially closed. The detection fired on **behavior** (many failures from one source), not a signature, which is the right model for credential attacks. The strongest control was the cheapest: removing password auth eliminated the entire attack class rather than trying to detect every future attempt.
