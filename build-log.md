# Project AEGIS — Build Log

A chronological record of the build: what I did at each stage, every problem I hit, and how I solved it. The problems are the point — this is where the real learning happened. Nothing here was copied from a tutorial; each fix came from diagnosing an actual failure on real hardware.

> Lab addressing shown is RFC 1918 private space (10.10.x.0/24). Upstream/home details are omitted for privacy.

---

## Pre-flight

- Verified system memory with memtest86 (passed all tests) before committing to the build.
- Prepared the Proxmox VE installer on USB (image is under 2 GB).

---

## Stage 1 — Install Proxmox VE (hypervisor)

**Problem — Ethernet reach.** No Ethernet cable long enough to reach the install location.
**Solution.** Relocated the Dell mini-PC and a monitor next to the modem so it had a wired connection throughout the build (Wi-Fi is not ideal for a hypervisor host).

**Problem — Proxmox couldn't find the disk.** The installer could not detect the internal drive.
**Solution.** In BIOS, changed the SATA mode from **RAID to AHCI**. Proxmox (Linux) can't use the motherboard's proprietary RAID layer; AHCI removes that layer and exposes each drive to the OS directly as an independent unit.

**Problem — update repo error.** Updates failed, prompting for an enterprise subscription.
**Solution.** Disabled the enterprise repo, enabled the no-subscription repo, then updated successfully.

---

## Stage 2 — Virtual networking

Goal: carry all four VLANs over one physical NIC using a VLAN-aware Linux bridge.

Topology: `Internet → home router → (vmbr0 = WAN) → pfSense → (vmbr1, VLAN-aware = LAN trunk)` feeding VLAN 10 (MGMT/SOC), 20 (SERVERS), 30 (USERS), 99 (QUARANTINE).

Reference addressing: 10.10.10.0/24, 10.10.20.0/24, 10.10.30.0/24, 10.10.99.0/24 — each gateway `.1` on pfSense.

**Problem — couldn't create the VLANs at the Proxmox layer.**
**Solution.** Created two bridges: `vmbr0` (WAN, bound to the physical NIC) and `vmbr1` (LAN trunk, VLAN-aware, no physical port). The physical NIC never carries a VLAN tag; all tagged frames live on `vmbr1`, traveling only between the VMs and pfSense. The VLAN interfaces themselves are defined later inside pfSense.

---

## Stage 3 — Install pfSense (firewall / router / DHCP / VPN)

Created the pfSense VM with two NICs: `vtnet0` → WAN (vmbr0), `vtnet1` → LAN (vmbr1).

**DHCP setup — Problem.** Error that the ISC DHCP backend is end-of-life.
**Solution.** Switched the DHCP engine to **Kea**. Kea doesn't do static mapping the same way, so static IPs are set directly inside each server VM instead of via DHCP reservation.

**Problem — VLANs didn't appear under DHCP settings.**
**Solution.** DHCP settings only list interfaces that are enabled and have an IP assigned. Once each VLAN interface was up and addressed, its DHCP scope became configurable.

Configured a DHCP scope per VLAN, basic per-VLAN firewall rules for internet egress, and a **deny + log** rule for QUARANTINE (isolation by default; those denied-packet logs become SIEM detections later).

**VPN setup (WireGuard):**

**Problem — WireGuard installed but wouldn't run.**
**Solution.** Confirmed the package/pfSense version compatibility and enabled WireGuard as a service in settings (it wasn't started).

Built the tunnel (listen port 51820), added the laptop peer with its public key on a dedicated VPN subnet, and enabled the tunnel as an interface (no IP — the range lives inside the tunnel) so it could have its own firewall rules. Two rules: WAN allows inbound UDP 51820 (the handshake); the VPN interface allows the VPN subnet to reach **MGMT/SOC only**, per the RFP.

**Problem — couldn't reach the management IP over the VPN.**
**Solution.** Firewall rules were correct, so I checked the home router and added UDP 51820 port-forwarding. Still failed — the dedicated VPN subnet had dropped off the tunnel interface config. Re-applied it at the interface level and the tunnel came up.

---

## Stage 4 — Active Directory (Windows Server 2022 DC)

**Problem — no network after install.** Windows Server had no network adapter, because Proxmox presents virtio (KVM) virtual hardware Windows doesn't recognize natively.
**Solution.** Device Manager → Ethernet controller → manually loaded the **virtio** driver to bring up the network adapter.

Confirmed the DC pulled a VLAN 20 address, set a static IP, added AD DS + DNS, created the `aegis.lab` forest (NetBIOS: AEGIS), and set pfSense as the DNS forwarder. Pointed VLAN 20/30 DHCP DNS at the DC.

**RFP requirement — restrict which DC ports users can reach.** Built a port alias covering only domain-membership essentials: DNS 53, Kerberos 88, LDAP 389, SMB 445, Kerberos pw-change 464, RPC endpoint mapper 135, dynamic RPC 49152–65535, NTP 123, Global Catalog 3268.

**Problem — rule order defeated the scoping.** A broad "VLAN 30 → any" rule sat above the "only these DC ports" rule, so the broad rule matched first (ACLs evaluate top-down) and the scoping never applied.
**Solution.** Reordered to: (1) allow VLAN 30 → DC-specific ports only, (2) deny VLAN 30 → other VLANs, (3) allow VLAN 30 → internet. Now users reach only the intended DC ports, nothing else internal, but still get internet.

Created OUs (Staff / Servers / Admins), three users plus a dedicated admin account (not the built-in Administrator), and two GPOs (password/lockout policy; audit policy). Linked at domain level, applied with `gpupdate /force`. Joined the Windows 10 client to the domain (VLAN 30, DC as DNS) and confirmed the GPOs applied via `gpresult`.

---

## Stage 5 — Linux application server (Ubuntu)

**Problem — no network adapter after install.**
**Solution.** The interface was down; brought it up with `ip link set <iface> up`, checked `ip addr`, found it wasn't pulling DHCP, and confirmed `/etc/netplan` had no config. Wrote a static netplan YAML and confirmed connectivity.

Installed OpenSSH server, net-tools, and nginx (so the box doubles as a monitorable application server). Verified SSH locally and from another host on the subnet (the DC). Took a baseline snapshot of every VM before testing (REQ-HOST-4).

---

## Stage 6 — Wazuh SIEM

Installed a second Ubuntu server on VLAN 10, static IP, SSH enabled (reachable over the WireGuard VPN from my laptop outside the lab). Pre-set the indexer requirement before install:

```
sudo apt update && sudo apt install -y curl apt-transport-https
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-wazuh.conf
sudo sysctl --system
```

**Problem — install failed and self-rolled-back.** All three services (indexer/manager/dashboard) came up "not found." The installer has a failure handler that tears down what it built; the underlying cause was insufficient disk in the VM. Reinstalls then failed because fragments lingered.
**Solution.** After adding disk, performed a full manual teardown of every lingering Wazuh package/service/file (apt purge with force flags, removing stale dpkg pre/post-rm scripts, killing lingering processes, `dpkg -l | grep wazuh` after each pass) until the system was truly clean, then reinstalled successfully.

**Agents:**

Installed agents on the DC and Windows client via the dashboard-generated PowerShell commands. For Ubuntu, added a serial port to the VM and used pfSense WAN port-forwarding (SSH from my specific laptop → pfSense → Ubuntu) to reach it, since the lab is otherwise unreachable from the home network.

**Problem — Windows client agent didn't report in.** VLAN 30 could reach the DC's specific ports but not the Wazuh manager on VLAN 10.
**Solution.** Added an alias for ports 1514/1515 and a rule allowing VLAN 30 → Wazuh IP on those ports, ordered second. Final VLAN 30 rule order: (1) DC-specific ports, (2) Wazuh ports, (3) deny other VLANs, (4) allow internet.

**Firewall-log ingestion — the hard part:**

**Problem — 0 packets to Wazuh on UDP 514.** `tcpdump` showed pfSense wasn't sending.
**Solution.** pfSense's "firewall events" syslog category silently didn't forward filterlog on this version. Enabling *all* remote syslog contents made packets flow immediately; filtering/parsing was then handled on the Wazuh side to avoid overload.

**Problem — events arrived but wouldn't parse.** pfSense sends syslog without a hostname, so Wazuh's pre-decoder read `filterlog[76714]` as the hostname and never set the program name.
**Solution.** Wrote a **custom decoder** matching the raw filterlog pattern directly instead of relying on program_name.

**Problem — alerts fired but never showed on the dashboard.**
**Solution.** `alerts_log` was disabled in `ossec.conf` — the exact feed the dashboard reads. Enabled it.

---

## Stage 7 — Suricata network IDS

Installed Suricata on pfSense, enabled the ET Open ruleset, set daily rule updates (RFP requirement). Ran in **IDS / alert-only** mode on a single interface (not all VLANs) to control RAM, trimming categories to a focused set (scan, exploit, malware, dos, attack_response, current_events).

**Pre-work (crash prevention).** Bumped pfSense RAM 3→4 GB and dropped Windows Server 3→2 GB for host headroom; snapshotted pfSense + Wazuh before touching Suricata. Verified RAM plateaued in a healthy band (it plateaus, doesn't leak).

**Noise tuning (REQ-IDS-2).** The dominant false positive was GID:SID 1:2200074 ("TCPv4 invalid checksum"). Root cause: VM NIC checksum offloading — Suricata inspects packets before the host finalizes checksums, so legitimate traffic looks "invalid." It's a virtualization artifact, not a threat. Suppressed it and documented why.

**Wazuh integration (eve.json → SIEM).** pfSense has no Wazuh agent, so I used the existing syslog transport. Enabled EVE JSON output (type syslog, alerts only) and built a custom `suricata-eve` decoder — a prematch strips the syslog prefix and `offset="after_prematch"` hands the JSON to the JSON decoder (key detail: the prematch must stop *before* the `{`). Added rules to fire a named alert.

**The real bug (the one worth telling in an interview).** Suricata alerts fired on the wire but never reached Wazuh. Traced packet-by-packet to the raw data: the full EVE JSON exceeded the syslog message-length limit and **truncated mid-object**, producing invalid JSON that Wazuh's decoder silently dropped. Fix: disabled packet dumps, HTTP metadata, app-layer metadata, and traffic-type logging so each alert fits the syslog limit and arrives complete. Confirmed end-to-end.

**Known pfSense quirk.** The syslog daemon occasionally drops its remote target mid-session; the fix is Status → System Logs → Settings → Save (restarts the daemon), not a rebuild. It's the sender, not Wazuh.

**Hardening (bonus).** Removed a leftover WAN allow-all management rule. Management is now VPN-into-MGMT only; WAN exposes only WireGuard UDP 51820. Proxmox console retained as break-glass recovery.

---

## Stage 8 — Adversary host & detection validation (Kali)

**SSH hardening prep.** Generated an ed25519 keypair on the client (Windows Server); private key stays on the client, public key installed in the Linux server's `authorized_keys`. Left password auth **on** for now so the brute-force test has something to attack (before/after story).

**Setup.** Installed Kali in VLAN 99, verified it was isolated (couldn't reach other VLANs or the internet), then added a single temporary firewall rule permitting only Kali → Linux server for the test window. Enabled Suricata on VLAN 20 so inter-zone traffic is inspected.

**Test 1 — network scan.** `nmap` scan of the Linux server → Suricata alert, correlated to the target host in Wazuh.

**Test 2 — SSH brute-force.** `hydra` against the Linux server's SSH → Wazuh correlated the repeated failures into a brute-force alert with the correct attacker IP. Removed the temporary allow rule afterward.

**Test 3 — privilege escalation.** Created a domain user on Windows Server and escalated it to admin. Wazuh caught Event 4720 (account created) and Event 4728 (member added to a security-enabled group). Removed the test account afterward.

**Incident response — SSH hardening.** Disabled password authentication on the Linux server. Found (and fixed) that a drop-in override file still had password auth enabled — reconfigured it so password auth is fully off. Verified: a password-only SSH attempt from Windows Server is now refused (`Permission denied (publickey)`), while key auth still works. Detect → triage → remediate → verify.
