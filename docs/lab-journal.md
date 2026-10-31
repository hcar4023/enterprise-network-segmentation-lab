# Current Project Status — As of October 31, 2026

**Project:** Enterprise Network Segmentation & SSH Security Monitoring Lab  
**Scope:** VirtualBox internal networks, OPNsense firewall, Kali testing workstation, and Ubuntu Server. These are separate virtual networks, **not** 802.1Q VLANs.  
**Evidence standard:** “Verified” refers to the recorded test output and operator-confirmed observations below. Earlier dated checkpoints are historical; the latest verified findings take precedence.

| Area | Current state | Evidence / limitation |
| --- | --- | --- |
| IPv4 inter-segment policy | **Verified for tested flows** | Kali `10.10.30.10` → Ubuntu `10.10.20.10` TCP/22 open; ICMP ping 4 sent, 0 received. Do not generalize to every protocol. |
| Firewall rules | **Configured and tested for specified flows** | Scoped `ALLOW_KALI_TO_UBUNTU_SSH` before `BLOCK_ATTACK_TO_SERVER`; broad default LAN IPv4 allow disabled. |
| Ubuntu SSH hardening | **Verified** | Effective `sshd -T`: root login disabled; public-key authentication enabled; password and keyboard-interactive authentication disabled; X11 forwarding disabled; `MaxAuthTries 3`. Valid key login succeeded; password-only test rejected. |
| Secure time synchronization | **Verified** | Chrony selected an NTS source; operator confirmed normal leap status and synchronized clock after scoped NTS/NTP egress rules. |
| SSH event detection | **Verified in controlled test** | One invalid-username connection produced an `Invalid user` record and a separate pre-auth closure. A broad case-insensitive match counted two lines; a more specific match counted one event. |
| SSH log field extraction | **Verified for displayed journal format** | `awk` extracted time, attempted username, source IPv4, and source port from the controlled event. Fixed field indices are format-dependent. |
| IPv6 segmentation | **Not yet verified** | SSH daemon listens on IPv6; IPv6 reachability and firewall policy have not been audited. |
| Temporary diagnostic ICMP rule | **Cleanup unconfirmed** | Check whether `TEST_UBUNTU_EXTERNAL_ICMP` remains enabled on OPT1. |
| VM performance | **Known issue; investigation deferred** | Ubuntu soft lockups and VirtualBox NEM/Windows hypervisor slow execution were observed; causal relationship unproven. |
| Snapshot baseline | **Pending** | No confirmed clean snapshot checkpoint in the available journal. |
| Custom SIEM | **Separate project; planning only** | No custom collector, database, alerting service, or dashboard deployed as part of this journal. |

**Next network-lab maintenance:** Audit the temporary ICMP rule and IPv6 policy when ready; preserve the working IPv4/SSH control set. The custom SIEM build is tracked independently.  
**Reading guide:** The entries below form a chronological engineering record. Words such as “pending” in older sessions reflect what was known **at that time** and are superseded by subsequent verified entries.

---

# Enterprise Network Segmentation Lab — Session 1

**Date:** October 27, 2026  
**Platform:** Windows 11 host, 32 GB RAM, Oracle VirtualBox  
**Project purpose:** Build an isolated enterprise-style virtual network, implement security controls, and validate segmentation using controlled testing.  
**Scope:** Virtual only. Cisco Catalyst 2960-X is owned but not configured and is **not** part of this project.

## Architecture

```text
                       Windows host / VirtualBox
                                |
                     VirtualBox NAT engine
                                |
                  OPNsense (LAB-FW01) — WAN em0
                         DHCP: 10.0.2.15/24
                          /             \
           LAN em1 / LAB-ATTACK       OPT1 em2 / LAB-SERVER
               10.10.30.1/24           10.10.20.1/24
                    |                         |
             Kali (LAB-KALI01)         Ubuntu (LAB-SRV01)
              10.10.30.10              10.10.20.10
                PLANNED                  PLANNED
```

**VirtualBox adapter mapping (intended):**

| VM | Adapter 1 | Adapter 2 | Adapter 3 |
| --- | --- | --- | --- |
| LAB-FW01 | NAT (WAN) | Internal Network `LAB-ATTACK` (LAN) | Internal Network `LAB-SERVER` (OPT1) |
| LAB-SRV01 | Internal Network `LAB-SERVER` | Disabled | Disabled |
| LAB-KALI01 | Internal Network `LAB-ATTACK` | Disabled | Disabled |

*Internal Network is not the same as Host-Only Network. The host is not directly attached to the internal segments. We have not created actual 802.1Q VLANs.*

## Completed and verified

### Virtual infrastructure
- Selected VirtualBox for the project on a Windows host with 32 GB RAM.
- Chose two isolated Internal Networks: `LAB-ATTACK` and `LAB-SERVER`.
- Excluded the unconfigured physical Cisco switch from the project.
- Created/configured the OPNsense VM and Ubuntu Server VM.

### OPNsense (`LAB-FW01`)
- Installed OPNsense to its **dedicated virtual hard disk** using the UFS installation option.
- Changed the root password during installation.
- Removed the installation ISO and booted into the installed OPNsense console.
- Detected three FreeBSD interfaces: `em0`, `em1`, and `em2`.
- Console screenshot confirmed the following addresses:

| Interface | Role | Address | Verification |
| --- | --- | --- | --- |
| `em0` | WAN, VirtualBox NAT | `10.0.2.15/24` (DHCP) | Visible in console |
| `em1` | LAN, `LAB-ATTACK` | `10.10.30.1/24` | Visible in console |
| `em2` | OPT1, `LAB-SERVER` | `10.10.20.1/24` | Visible in console |

- **Successful test:** Ping from OPNsense to `1.1.1.1`, confirming IPv4 connectivity on the firewall's WAN side.
- **Not yet verified:** end-to-end traffic from Ubuntu through OPNsense, firewall policy enforcement, and internet access from lab guest VMs.

### Ubuntu Server (`LAB-SRV01`)
- Created Ubuntu Server VM with approximately 2 GB RAM, 2 vCPUs, and a 25 GB virtual hard disk.
- Assigned its sole NIC to the `LAB-SERVER` Internal Network.
- Installer detected NIC `enp0s3`.
- Initially attempted static addressing during installation; repository/mirror requests failed because outbound connectivity from this segment was not yet configured.
- Restarted the installer and continued with networking disabled/offline.
- Used the complete 25 GB virtual disk with an ext4 root filesystem; LVM was disabled.
- Created local server account (planned name `labadmin`) and host name (planned `lab-srv01`); verify these at login rather than assuming they match exactly.
- Selected **Install OpenSSH server** and allowed password authentication temporarily; no SSH key was imported.
- Installer logs showed core installation success; Ubuntu subsequently reached reboot and boot/login stage.
- At the end of the session, identified `enp0s3` and an installer-created Netplan YAML file.
- **Unconfirmed:** Whether static Netplan settings were saved and applied. Do not mark `10.10.20.10/24` operational until tested.

### Kali Linux
- Kali was included in the network design, but **its lab addressing and connectivity are not yet configured/verified**.

## Issues and lessons learned
1. **Host-Only vs Internal Network:** Internal Network isolates guests from the Windows host; Host-Only adds a host interface. We selected Internal Network for both lab segments.
2. **Package repository timeout:** Ubuntu installer attempted to contact `archive.ubuntu.com` and `security.ubuntu.com`; DNS/network requests failed while the server segment lacked permitted outbound access. Installing offline allowed progress.
3. **Firewall policy not yet established:** Interface IP assignments alone do **not** prove segments are isolated by policy. OPNsense LAN can have permissive default rules; later tests must explicitly validate rule behavior and logs.
4. **Ubuntu release:** Installer logs showed `resolute` repositories (Ubuntu 26.04). Verify actual installed version using `cat /etc/os-release` after login; do not assume 24.04 LTS.

## Resume point — next session

1. Power on `LAB-FW01` **first** and check its console addresses.
2. Power on `LAB-SRV01`, log in locally, and inspect its network:

```bash
ip -br addr
ip route
ls -l /etc/netplan/
sudo cat /etc/netplan/*.yaml
cat /etc/os-release
```

3. If Netplan is not yet configured, **edit the existing installer YAML file** for the actual interface `enp0s3`. Target settings:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 10.10.20.10/24
      routes:
        - to: default
          via: 10.10.20.1
      nameservers:
        addresses:
          - 1.1.1.1
```

Do not overwrite unexpected existing settings without inspecting them. Indentation is significant. If the YAML file already provides these settings, no edit is needed.

4. Validate and apply:

```bash
sudo netplan generate
sudo netplan try
ip -br addr show enp0s3
ip route
ping -c 4 10.10.20.1
```

**Note:** An ICMP ping may be blocked by OPNsense OPT1 firewall rules even if Layer 2 and addressing are correct. Check interface status/rules and firewall logs before changing the topology.

5. From Kali or an appropriately configured management path, access the OPNsense web interface at `https://10.10.30.1` (not yet verified).
6. In OPNsense, check that OPT1 is enabled and plan explicit minimal outbound rules for Ubuntu's DNS (53), HTTP (80), and HTTPS (443) as needed for OS updates, along with any required firewall DNS policy. Avoid broad unrestricted allow rules.
7. Set up Kali's address `10.10.30.10/24`, gateway `10.10.30.1` and verify connectivity.
8. Explicitly enforce and validate **default-deny inter-segment traffic**, with controlled allow rules for selected Ubuntu services, logging, and testing with ping, TCP tools, Nmap, and packet captures.
9. Create snapshots after a verified stable baseline.

## Evidence to collect for GitHub
- Sanitized topology diagram
- VirtualBox adapter settings (no secrets)
- OPNsense console interface summary
- Ubuntu `ip -br addr` and `ip route`
- Netplan configuration (without secrets)
- Firewall rule screenshots and logs
- Test matrix: source, destination, port/protocol, expected result, actual result, timestamp
- Before/after Nmap results and packet captures

## Status summary

| Task | Status |
| --- | --- |
| Choose architecture and VM network types | Complete |
| Install OPNsense | Complete |
| Configure OPNsense WAN/LAN/OPT1 IPs | Complete (console verified) |
| Test firewall WAN reachability | Passed (`1.1.1.1`) |
| Install Ubuntu Server | Complete; reached reboot/login |
| Assign Ubuntu static IP | Pending verification |
| Enable outbound updates for Ubuntu | Not started |
| Configure Kali lab interface | Not started |
| Implement/test inter-segment firewall rules | Not started |
| Physical Cisco switch setup | Out of scope |

---
*Session journal: this records observed progress and separates configured, verified, planned, and unconfirmed steps. Update it after each work session.*


---

# Session 2–3 — Implementation, diagnostics, and validation

**Dates:** October 28–29, 2026  
**System:** VirtualBox; OPNsense 26.7; Kali; Ubuntu Server  
**Documentation convention:** VERIFIED means supported by screenshots or reported test output; CONFIGURED means applied but not necessarily fully validated; PENDING means not completed.

## Current topology (verified since Session 1)

| VM | Host interface | IPv4 | Virtual network |
| --- | --- | --- | --- |
| `LAB-FW01` OPNsense | WAN `em0` | `10.0.2.15/24` (observed DHCP address) | VirtualBox NAT |
| `LAB-FW01` OPNsense | LAN `em1` | `10.10.30.1/24` | `LAB-ATTACK` |
| `LAB-FW01` OPNsense | OPT1 `em2` | `10.10.20.1/24` | `LAB-SERVER` |
| `LAB-KALI01` Kali | `eth0` | `10.10.30.10/24` | `LAB-ATTACK` |
| `LAB-SRV01` Ubuntu | `enp0s3` | `10.10.20.10/24` | `LAB-SERVER` |

OPNsense's FreeBSD interfaces `em1` and `em2` and their IP addresses were verified in `ifconfig`. These are **two VirtualBox Internal Networks**, not 802.1Q VLANs. The Cisco switch remains unconfigured and out of scope.

## Phase 2 — Network segmentation (VERIFIED for tested IPv4 flows)

1. Confirmed Ubuntu could reach its OPNsense gateway (`10.10.20.1`) once a narrowly scoped OPT1 ICMP pass rule was added; the pass was observed in firewall Live View.
2. Confirmed Kali could initially ping Ubuntu across the routed subnets, prior to effective isolation.
3. Installed/started OpenSSH on Ubuntu and verified its listener on TCP 22. Confirmed Kali's Nmap result was initially **filtered** under the general block, then **open** following the SSH allow exception. An interactive Kali-to-Ubuntu SSH login succeeded.
4. Created and applied `ALLOW_KALI_TO_UBUNTU_SSH`: LAN, in, IPv4 TCP, source `10.10.30.10/32`, destination `10.10.20.10/32`, destination port 22, pass/log/quick.
5. Created and applied `BLOCK_ATTACK_TO_SERVER`: LAN, in, IPv4 any protocol, source `10.10.30.0/24`, destination `10.10.20.0/24`, block/log/quick.
6. Unexpected finding: Kali-to-Ubuntu ICMP continued to pass despite the displayed GUI ordering (SSH permit, general block, default LAN allow). Captures on `em1` and `em2` showed requests and replies passing both interfaces. Firewall states associated the permitted ICMP with `Default allow LAN to any rule`.
7. Verified via `pfctl -sr` that the SSH permit, block rule, and general LAN pass existed in the active PF ruleset. The output displayed the default pass before the quick block; the significance of the non-quick default pass versus quick block was **not conclusively explained**. Do not claim a definitive PF root cause.
8. Disabled the broad **Default allow LAN to any rule (IPv4)** as a controlled test. **Afterward**, Kali-to-Ubuntu ICMP showed 100% packet loss; Live View explicitly logged `LAN / In / ICMP / 10.10.30.10 → 10.10.20.10 / block / BLOCK_ATTACK_TO_SERVER`. Meanwhile, Nmap confirmed TCP 22 remained **open**, and Kali continued to access the OPNsense web UI at `https://10.10.30.1`.
9. After the broad IPv4 LAN permit was disabled, Kali's ping to its OPNsense gateway `10.10.30.1` showed 100% packet loss; HTTPS management still worked, consistent with the automatic management anti-lockout behavior. Do not confuse gateway ICMP with management HTTPS.

**Current Phase 2 conclusion:** The intended *tested IPv4* behavior works: SSH is permitted; other tested ICMP is blocked by the named rule. This is **not** proof that all protocols or IPv6 paths are isolated. IPv6 rules and policies require separate validation. The precise historical cause of the unexpected ICMP allowance has not been fully established.

## Phase 3 — Controlled outbound connectivity (October 29)

### Baseline findings (VERIFIED)

| Test | Initial result | Evidence |
| --- | --- | --- |
| Ubuntu `ping -c 4 10.10.20.1` | Success, 0% packet loss | OPNsense log matched `ALLOW_UBUNTU_GATEWAY_ICMP` |
| Ubuntu `ping -c 4 1.1.1.1` | Failed, 100% packet loss | OPT1 inbound ICMP block logged, default deny/state violation |
| Ubuntu `getent ahostsv4 archive.ubuntu.com` | Initially no results | No DNS resolution at baseline |
| Ubuntu TCP/UDP DNS to `1.1.1.1:53` | Blocked at baseline | OPNsense OPT1 logs showed blocks |

### Changes and positive tests (VERIFIED)

- Created temporary diagnostic rule **`TEST_UBUNTU_EXTERNAL_ICMP`** on OPT1: in, IPv4 ICMP, pass/quick/log; source `10.10.20.10/32`, destination `1.1.1.1/32`. Subsequent ping to `1.1.1.1` **succeeded**. This demonstrates permitted external IPv4 communication and functional forwarding/NAT for that tested flow.
- Created **`ALLOW_UBUNTU_DNS`** on OPT1: in, IPv4 TCP/UDP port 53, source `10.10.20.10/32`, destination `1.1.1.1/32`, pass/quick/log. The user subsequently ran `getent ahostsv4 archive.ubuntu.com` and `getent ahostsv4 security.ubuntu.com`; both returned multiple IPv4 records. **DNS resolution verified.** `STREAM`, `DGRAM`, `RAW` are address lookup socket-type entries, not proof of application connections.
- The user has **not yet added** the proposed repository HTTP/HTTPS rules or run a successful `apt update`. Do not mark Ubuntu package updates or TCP 80/443 external access completed.

### Outstanding work / next session — exact resume point

**Primary objective:** Enable Ubuntu's apt repository metadata retrieval with narrowly scoped TCP 80 and 443 egress, without changing established inter-segment rules.

1. Start OPNsense first, then Ubuntu and Kali as needed. Confirm the GUI is accessible from Kali at `https://10.10.30.1`.
2. Open **Firewall → Rules** in OPNsense 26.7. Confirm `ALLOW_UBUNTU_DNS` and `TEST_UBUNTU_EXTERNAL_ICMP` are present on OPT1, and `ALLOW_KALI_TO_UBUNTU_SSH` / `BLOCK_ATTACK_TO_SERVER` remain intact on LAN. The broad default IPv4 LAN allow was disabled during Phase 2; preserve that until an intentional policy review.
3. Create **two separate OPT1 pass rules** for Ubuntu's package traffic. Use the actual 26.7 rule editor fields, including `Enabled`, `Description`, `Interface`, `Quick`, `Action`, `Direction`, `Version`, `Protocol`, `Source`, `Source Port`, `Destination`, `Destination Port`, `Log`, `Gateway`, and inversion settings. Proposed rules:

| Rule description | Interface / direction | IP version / protocol | Source | Destination | Destination port | Action |
| --- | --- | --- | --- | --- | --- | --- |
| `ALLOW_UBUNTU_HTTP` | OPT1 / In | IPv4 TCP | `10.10.20.10/32` | any | 80 | Pass |
| `ALLOW_UBUNTU_HTTPS` | OPT1 / In | IPv4 TCP | `10.10.20.10/32` | any | 443 | Pass |

For both: enabled, Quick checked, source port any, inverted conditions off, log enabled, gateway None, categories blank, other advanced settings unchanged. These are restricted by **source host and destination port**, but destination `any` allows web connectivity to any reachable IPv4 address; consider more restrictive destinations later if warranted.

4. **Save and Apply** the two rules. Then on Ubuntu run:

```bash
sudo apt update
```

This retrieves package indexes; it **does not install upgrades**. If it fails, record the specific error and inspect Live View for OPT1 blocks to TCP 80/443 or DNS rather than guessing. Check Ubuntu's configured release/repositories with `cat /etc/os-release` and relevant apt sources if needed (earlier installer logs referenced `resolute`).

5. Verify permitted OPT1 TCP 80/443 flows in firewall logs if used. Preserve inter-segment separation: from Kali, `nmap -Pn -sT -p 22 --reason 10.10.20.10` should still show `open` and `ping -c 4 10.10.20.10` should still show blocked with a matching `BLOCK_ATTACK_TO_SERVER` log.
6. Remove/disable `TEST_UBUNTU_EXTERNAL_ICMP` after completing the diagnostic use case (with a fresh state retest if necessary). Decide later whether a narrowly scoped gateway ICMP rule for Kali is useful; its lack of ping response currently does not block HTTPS management.
7. Document actual outcomes; create clean shutdown snapshots after a fully verified checkpoint. Plan IPv6 segmentation validation separately.

## Useful commands and their purpose

```bash
# Ubuntu: view addressing and routing
ip -br addr
ip route

# Ubuntu: confirm gateway and external destination
ping -c 4 10.10.20.1
ping -c 4 1.1.1.1

# Ubuntu: verify DNS resolution
getent ahostsv4 archive.ubuntu.com
getent ahostsv4 security.ubuntu.com

# Ubuntu: retrieve apt package lists after HTTP/HTTPS rules
sudo apt update

# Kali: prove SSH permitted and ICMP blocked
nmap -Pn -sT -p 22 --reason 10.10.20.10
ping -c 4 10.10.20.10
```

OPNsense console/shell troubleshooting (reference only; no need to repeat unless a new fault occurs):

```sh
ifconfig -l
ifconfig | less
pfctl -sr | grep -E '10\.10\.20|10\.10\.30'
pfctl -ss
tcpdump -ni em1 'icmp and host 10.10.20.10'
tcpdump -ni em2 'icmp and host 10.10.20.10'
```

## Workflow standards for the rest of the lab

- Explain the **objective → network concept → full exercise → evidence → conclusion** in a connected sequence, without requiring screenshots after every step.
- Distinguish **observed evidence**, **hypotheses**, and **configuration changes** explicitly.
- Avoid repeating already-verified steps; only request screenshots when the actual OPNsense 26.7 interface differs or an error is ambiguous.
- Maintain a reproducible test matrix for the GitHub/resume portfolio. Exclude credentials, private keys, and sensitive backups.

**Next-session one-line checkpoint:** Configure two OPT1 IPv4 TCP allow rules from Ubuntu `10.10.20.10` to destination ports **80 and 443**, run `sudo apt update`, verify logs and segmentation, then clean up the temporary external ICMP rule.

---

# Day 3 continuation — October 30, 2026: NTS time synchronization restored

**Environment:** Windows 11 / VirtualBox; `LAB-FW01` (OPNsense), `LAB-SRV01` (Ubuntu, `10.10.20.10/24`, OPT1 / `LAB-SERVER`), `LAB-KALI01` (Kali / `LAB-ATTACK`).  
**Status:** Time synchronization **VERIFIED RESOLVED** from operator-reported command results; kernel soft lockup and frozen console **PENDING**.  
**Administration path:** SSH from Kali to Ubuntu (`ssh labadmin@10.10.20.10`). Commands below were run in the Ubuntu SSH session, not on Kali itself.

## Objective
Restore Ubuntu Server's secure network time synchronization without introducing broad allow rules or changing the working inter-segment security posture.

## Investigation — observations

1. `timedatectl` initially reported `System clock synchronized: no`, `NTP service: active`, `Time zone: Etc/UTC`, and `RTC in local TZ: no`. UTC is an acceptable server time zone; it was **not** the fault.
2. `systemctl status systemd-timesyncd --no-pager` returned `Unit systemd-timesyncd.service could not be found`. This alone did **not** mean NTP was unconfigured; Ubuntu was using another daemon.
3. `systemctl status chrony --no-pager` showed `chrony.service` **active (running)** and enabled. Logs repeatedly showed `NTS-KE session ... :4460 ... timed out` when contacting Ubuntu/Canonical NTS servers. One log line also showed `Forward time jump detected!`; cause/impact were not established.
4. `chronyc tracking` initially showed reference ID `00000000`, stratum `0`, reference time `1970-01-01`, and `Leap status: Not synchronised`. Chrony had no usable selected reference at that point.
5. OPNsense Firewall → Log Files → Live View confirmed **blocked TCP destination port 4460** from Ubuntu `10.10.20.10` on OPT1. This established a specific network-policy impediment to NTS key exchange. UDP 123 blocking was not separately demonstrated in the supplied log description.

## Remediation — OPNsense OPT1

Two explicit IPv4 pass rules were configured/applied for **source host** `10.10.20.10/32` and destination `any` during testing. Both were directed **in** on OPT1, with Quick enabled, source port any, and logging enabled:

| Rule description | Protocol | Destination port | Purpose | Status |
| --- | --- | --- | --- | --- |
| `ALLOW_UBUNTU_NTS_4460` | TCP | `4460` | Network Time Security key exchange | CONFIGURED; NTS connectivity subsequently observed |
| `ALLOW_UBUNTU_NTP_123` | UDP | `123` | NTP time exchanges | CONFIGURED; working sources subsequently observed |

No unrestricted all-ports rule was required. Destination `any` is wider than a fixed trusted-server allowlist; consider narrowing later only if the server set is maintained appropriately (NTS server IPs may change). The NTS protocol uses TCP 4460 for key establishment and UDP 123 for authenticated time queries.

## Validation — observed output

After firewall changes, `chronyc sources -v` reported **five Canonical sources**, including:

```text
^+ ntp-nts-2.ps5.canonical.>   2  7  377   44   -4584us[-4692us] +/- 78ms
^+ ntp-nts-3.ps5.canonical.>   2  7  377  104     +14ms[ +14ms] +/- 93ms
^* ntp-nts-2.ps6.canonical.>   2  8  377   41   +4339us[+4230us] +/- 68ms
^+ ntp-nts-3.ps6.canonical.>   2  8  377   40    -896us[ -896us] +/- 64ms
^- ntp-nts-1.ps5.canonical.>   2  8  377   44   -1580us[-1689us] +/- 82ms
```

Interpretation: `^*` identifies the selected server; `^+` are usable contributors; `^-` is reachable but omitted from combining; `Reach 377` is an octal reachability register recording eight successful recent polls. Remote source stratum was `2` (this is not necessarily the local client's stratum).

The operator then confirmed that **both follow-up checks succeeded**:

```bash
chronyc tracking
# Reported result: Leap status: Normal

timedatectl
# Reported result: System clock synchronized: yes; NTP service: active
```

The precise full output of those final two commands was not supplied; the successful fields above are documented from the operator's explicit confirmation, not invented raw logs.

**Conclusion: VERIFIED RESOLVED.** Chrony selected a valid NTS time source, and Ubuntu reported a synchronized system clock after OPNsense policy changes. The initial NTS-KE firewall block explains the failed key-exchange attempts. This does **not** establish that the earlier kernel issue was caused by clock synchronization.

## Current status and remaining work

| Item | Status | Evidence / next step |
| --- | --- | --- |
| SSH administration from Kali to Ubuntu | VERIFIED | Active working session to `labadmin@lab-srv01` |
| Chrony service | VERIFIED | `chrony.service` active (running) |
| NTS key exchange firewall rule TCP 4460 | CONFIGURED / FUNCTIONALLY VALIDATED | Initial block confirmed; later sources usable |
| NTP exchange firewall rule UDP 123 | CONFIGURED / FUNCTIONALLY VALIDATED | Reachability `377` on multiple sources |
| Ubuntu clock synchronization | VERIFIED RESOLVED | `^*` selected source; user confirms `Leap status: Normal` and synchronized `yes` |
| 543-second kernel soft lockup involving `snapd` | PENDING INVESTIGATION | Inspect previous-boot and kernel journal; don't assume link to NTP |
| `snapd` watchdog failure/restart | PENDING INVESTIGATION | Correlate timestamps and resource/VM performance |
| Ubuntu VirtualBox console froze while SSH continued | PENDING INVESTIGATION | Check guest graphics/console and host resource behavior; avoid hard power-off where SSH works |
| Temporary diagnostic ICMP allow rule | VERIFY CLEANUP STATUS | Prior journal listed `TEST_UBUNTU_EXTERNAL_ICMP`; no documented confirmation of removal |
| IPv6 firewall segmentation and snapshots | PENDING | Validate separately; snapshot stable baseline after investigation |

## Next session — read-only diagnostic commands (run on Ubuntu over SSH)

```bash
# Current boot and recent kernel warnings
uname -a
journalctl -k -b --no-pager | grep -Ei 'soft lockup|watchdog|snapd|blocked|hung task'

# Service history for snapd in current boot
systemctl status snapd --no-pager
journalctl -u snapd -b --no-pager -n 100

# Recent kernel/system warnings, including previous boot if journal persists
journalctl -p warning..alert -b --no-pager -n 150
journalctl --list-boots
```

If the lockup occurred during a *previous* boot and that boot is listed, inspect it with `journalctl -k -b -1 --no-pager` and `journalctl -u snapd -b -1 --no-pager`. Preserve original timestamp/time zone information, particularly since the clock was unsynchronized before remediation. Do not restart or disable `snapd` until evidence is gathered.

**Safe shutdown order:** Ubuntu → Kali → OPNsense. **Startup order:** OPNsense → Ubuntu → Kali.


---

# Day 3 continuation — October 30, 2026: SSH hardening and authentication monitoring

**Status:** SSH key authentication and password-authentication denial **VERIFIED**. Basic SSH log collection **VERIFIED**. Further detection test **PAUSED** at operator request after observing something on the main Ubuntu console; the observation has not yet been characterized.

## Configuration baseline and remediation

Effective baseline from `sudo sshd -T`:

```text
maxauthtries 6
permitrootlogin prohibit-password
pubkeyauthentication yes
passwordauthentication yes
x11forwarding yes
allowtcpforwarding yes
```

- Kali's `~/.ssh` was inspected; a dedicated Ed25519 key pair was created under `~/.ssh/lab_srv01_ed25519` (private) and `.pub` (public). User reported file permissions were appropriate; private key must stay on Kali. A passphrase was used, as confirmed by successful passphrase prompt on later authentication.
- Installed the public key for Ubuntu account `labadmin` at `10.10.20.10` with `ssh-copy-id`. Validated successful key-only SSH login before disabling password-based authentication.
- Added `/etc/ssh/sshd_config.d/99-lab-hardening.conf` initially, containing:

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
X11Forwarding no
MaxAuthTries 3
```

- `sshd -T` still showed `passwordauthentication yes`. Diagnosed first-applicable-value precedence: `/etc/ssh/sshd_config` includes `sshd_config.d/*.conf`; `/etc/ssh/sshd_config.d/50-cloud-init.conf` contained `PasswordAuthentication yes` and loaded before `99-lab-hardening.conf`.
- Renamed our own file (without altering cloud-init's file) to `/etc/ssh/sshd_config.d/01-lab-hardening.conf`. `sudo sshd -t` passed with no output. `sudo sshd -T` then confirmed:

```text
maxauthtries 3
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
x11forwarding no
```

- Applied changes using `sudo systemctl reload ssh`; logs subsequently confirmed the service reload.

## Authentication verification from Kali

1. **Positive control:** `ssh -i ~/.ssh/lab_srv01_ed25519 -o PreferredAuthentications=publickey -o PasswordAuthentication=no labadmin@10.10.20.10` prompted for the key's *passphrase* and successfully opened an Ubuntu 26.04.1 LTS shell (kernel `7.0.0-42-generic`).
2. **Negative control:** after exiting back to Kali, `ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password,keyboard-interactive -o NumberOfPasswordPrompts=1 labadmin@10.10.20.10` returned `Permission denied (publickey)`. This verifies password/keyboard-interactive authentication was not offered to that test client.
3. An accidental earlier SSH attempt from within Ubuntu used a Kali-only private-key path and failed with `Identity file ... not accessible`; this was an execution-location mistake, **not** evidence the hardening failed. Do not copy the private key to Ubuntu.

## Security monitoring baseline

Commands used on Ubuntu:

```bash
sudo journalctl -u ssh --since '15 minutes ago' --no-pager
sudo journalctl -u ssh --since today --no-pager | grep -E 'Accepted publickey|Failed password|Invalid user|Connection closed'
sudo journalctl -u ssh --since today --no-pager | grep -c 'Accepted publickey'
```

Results for October 30 (UTC timestamps):
- **3** `Accepted publickey` records for `labadmin` from Kali `10.10.30.10`, all using the same Ed25519 public-key fingerprint (fingerprint deliberately omitted from this summary): at `21:23:51`, `21:38:35`, and `21:40:57`.
- **4** `Connection closed by authenticating user ... [preauth]` records: `21:22:18` (two from Kali), `21:36:56` (from Ubuntu `10.10.20.10` during accidental self-SSH), and `21:39:42` (from Kali, consistent with password-only negative test).
- **0 explicit `Failed password` records** in displayed filtered output. A `[preauth]` closure does not, by itself, establish brute force or an attempted password.
- The SSH journal logged service reload at `21:36:37`, and listening on IPv4 `0.0.0.0:22` and IPv6 `[::]:22`. **Host SSH is listening on IPv6; IPv6 firewall segmentation remains unvalidated.**

## Network regression validation (same day)

From Kali `10.10.30.10`:

| Test | Evidence | Interpretation |
| --- | --- | --- |
| `ping -c 4 10.10.20.10` | 4 sent, 0 received | Kali → Ubuntu IPv4 ICMP blocked |
| `nmap -Pn -sT -p 22 --reason 10.10.20.10` | `22/tcp open ssh syn-ack` | Authorized SSH port reachable |
| `curl -kI --max-time 10 https://10.10.30.1` | `HTTP/2 403`, `server: OPNsense` | HTTPS application responding; HEAD request forbidden, not an authenticated GUI test |

## Open issues / explicit pause point

- **Paused before controlled invalid-username SSH detection exercise.** That test has **not** been executed or validated.
- The user noticed something on Ubuntu's **main VirtualBox console** and wants it examined before proceeding. **No description or screenshot of that new console observation has been supplied yet.** Prior console freeze, kernel soft lockups (543–544 s and 33 s), `snapd` watchdog error, and `vmwgfx` warning remain documented, unresolved, and deferred at user's preference unless new symptoms warrant investigation.
- OPNsense temporary `TEST_UBUNTU_EXTERNAL_ICMP` rule removal/disable status is still unconfirmed.
- IPv6 exposure/segmentation and snapshot baseline remain pending.
- **Next action:** Ask user to show/describe the Ubuntu console observation. Do not run another detection test or make SSH/firewall changes until it is understood. Preserve the current working key-authenticated SSH session.

**Key lesson:** Always inspect `sshd -T` effective policy; a correctly written file can be overridden by an earlier `Include` drop-in. Test *allow* and *deny* cases separately, retain an existing session as a recovery path, and distinguish observed log facts from inferred intent.

---

# Day 3 continuation — October 30, 2026: VirtualBox soft-lockup analysis and Phase 4 checkpoint

**Status:** Host virtualization execution mode **VERIFIED**; cause of Ubuntu guest soft lockups **UNCONFIRMED / DEFERRED**. Phase 4 SSH detection exercise **READY / NOT EXECUTED**. No virtualization or Windows security configuration changes were made.

## Ubuntu kernel evidence (current boot)

Ubuntu `LAB-SRV01` (`10.10.20.10`) was examined over SSH with `sudo journalctl -k -b --no-pager | grep -Ei 'soft lockup|watchdog|clocksource|vmwgfx'` and `uptime`:

- Kernel logged `clocksource: Long readout interval, skipping watchdog check` at **19:12:29**, **21:35:02**, and **21:51:05** (Oct 30, Ubuntu's log time zone UTC).
- Kernel logged CPU #1 soft lockups with `snapd:1128`: **43 seconds** at **21:35:02**, and **507 seconds** at **21:51:05**.
- `vmwgfx` initialized under VirtualBox VMSVGA and warned of an unsupported hypervisor / graphics configuration; no causal link to the CPU lockups established.
- `uptime` at 21:59:28 UTC: **2:54 uptime**, **3 users**, load averages **0.03, 0.02, 0.00**. Low *current* load does not disprove earlier CPU scheduling stalls.
- Correlation of clocksource warnings with the lockups motivates investigation of guest scheduling/timekeeping; it does not prove the specific cause or implicate `snapd` as defective.

## VM and Windows host evidence

- Ubuntu VM configuration: **2 vCPUs**, **Default** paravirtualization, **nested paging enabled**, **VMSVGA** graphics, **16 MB** video memory. No setting was changed.
- Windows PowerShell `Get-CimInstance Win32_ComputerSystem | Select-Object HypervisorPresent` returned **True**. `msinfo32`: virtualization-based security **Running** and hypervisor-detected message present.
- `Win32_DeviceGuard` reported `VirtualizationBasedSecurityStatus = 2`, `SecurityServicesRunning = {2}`, `SecurityServicesConfigured = {2}`: VBS running, Memory Integrity / HVCI configured and running.
- VirtualBox release reported by operator: **7.2.22 r175245** (Windows, Qt 6.8.0).
- `LAB-SRV01` log: `LAB-SRV01/Logs/VBox.log (local VirtualBox VM log)`. Entries include `HM: ... Attempting fall back to NEM: VT-x is not available`, `NEM: WHvCapabilityCodeHypervisorPresent is TRUE`, and `NEMR3Init: Snail execution mode is active!`. VirtualBox says it cannot operate at full potential in this mode.
- **Finding:** `LAB-SRV01` is executing via VirtualBox NEM / the Windows Hypervisor Platform rather than native VirtualBox VT-x execution. The performance limitation is confirmed; its role in guest soft lockups is plausible but **not demonstrated experimentally**.

## Decision and safeguards

- Operator chose to **defer remediation** because Ubuntu remains operational and the issue is not presently blocking lab functionality. Revisit if freezes become frequent, SSH becomes unreliable, or system availability changes.
- Preserve **Windows VBS/HVCI**, nested paging, and the present 2-vCPU/VMSVGA configuration; do not disable Windows security features as an untested fix.
- Kernel lockups remain an **open, non-blocking issue**. Do not mark as resolved or attribute the issue conclusively to Hyper-V, clocksource, `snapd`, or the OPNsense rules.

## Phase 4 — SSH security monitoring / detection engineering (next resume point)

**Existing controls, verified earlier:** Kali `10.10.30.10` → Ubuntu `10.10.20.10` SSH allowed; ICMP blocked; Ed25519 key authentication works; password-only SSH rejected; root login disabled in effective SSHD configuration; authentication records visible via journal. Prior SSH log baseline: **3** accepted public-key events, **4** pre-auth connection closures, no explicit `Failed password` events in the filtered view.

**Pending controlled exercise:** Produce **one** SSH connection attempt from Kali with nonexistent account `invalidlabuser`, observe the resulting Ubuntu SSH event(s), and determine if an `Invalid user` or pre-auth closure is logged. No invalid-username test output has been supplied or confirmed yet.

Run on **Kali** (not inside Ubuntu SSH):

```bash
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password -o ConnectTimeout=5 invalidlabuser@10.10.20.10
```

Then run on **Ubuntu**, using an existing authenticated SSH session:

```bash
sudo journalctl -u ssh --since '10 minutes ago' --no-pager
sudo journalctl -u ssh --since '10 minutes ago' --no-pager | grep -Ei 'invalid|failed|connection closed|authentication'
```

Capture the raw command results and matching source IP / timestamps before drafting a Sigma detection or SIEM correlation query. One invalid account connection is **not** brute-force evidence.

**Session pause:** October 30, 2026, approximately 15:28 Arizona local time; operator leaving for work. **Resume directly with Phase 4's single invalid-username SSH test**, unless the Ubuntu VM becomes unstable. Do not repeat completed SSH hardening or hypervisor diagnostics without new symptoms.

---

# Day 4 — October 31, 2026: Phase 4 SSH event detection and parsing validated

**Status:** Phase 4 controlled SSH detection test **VERIFIED COMPLETE**; custom SIEM design **PLANNED**, not installed. This entry supersedes the earlier October 30 “pending controlled exercise” checkpoint without altering historical entries.

## Objective
Generate a controlled SSH connection using a nonexistent username from the Kali segment; validate that Ubuntu records the event; distinguish log-line counts from authentication-attempt counts; extract incident-relevant fields.

## Environment and controls retained
- Windows 11 host, **32 GB RAM**, VirtualBox; OPNsense `LAB-FW01` (reported allocation 3 GB), Kali `LAB-KALI01` (3 GB), Ubuntu `LAB-SRV01` (2 GB).
- Kali `10.10.30.10` on `LAB-ATTACK`, Ubuntu `10.10.20.10` on `LAB-SERVER`; the OPNsense firewall continues to permit authorized SSH between these hosts and block the previously tested Kali-to-Ubuntu ICMP flow.
- Previously verified SSH policy: Ed25519 key login works for `labadmin`, password-only authentication rejected, root SSH login disabled in effective configuration.
- Guest kernel soft lockups / VirtualBox NEM performance concerns remain **unresolved but non-blocking**; no VBS, Hyper-V, or VM configuration changes made in this exercise.

## Exercise / actual evidence
**Kali command, executed once:**

```bash
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password -o ConnectTimeout=5 invalidlabuser@10.10.20.10
```

**Ubuntu journal output (Oct 31 06:43:41, Ubuntu's UTC time, equal to Oct 30 23:43:41 Arizona):**

```text
Oct 31 06:43:41 lab-srv01 sshd-session[1343]: Invalid user invalidlabuser from 10.10.30.10 port 47000
Oct 31 06:43:41 lab-srv01 sshd-session[1343]: Connection closed by invalid user invalidlabuser 10.10.30.10 port 47000 [preauth]
Oct 31 06:44:35 lab-srv01 sshd-session[1347]: Accepted publickey for labadmin from 10.10.30.10 port 54504 ssh2: ED25519 [public-key fingerprint redacted]
```

The accepted public-key event verifies a subsequent legitimate login, not a successful authentication for `invalidlabuser`. The controlled invalid-user connection was closed before authentication. No password failure or brute-force attempt was established.

## Detection specificity and verification
Initial broad query returned **2 matching lines** for the single test attempt:

```bash
sudo journalctl -u ssh --since today --no-pager | grep -Eic 'Invalid user'
```

This matched both `Invalid user ...` and `Connection closed by invalid user ...` because `-i` is case-insensitive. The refined signature returned **1**:

```bash
sudo journalctl -u ssh --since today --no-pager | grep -Ec 'sshd-session\[[0-9]+\]: Invalid user '
```

**Lesson:** counting related messages is not the same as counting unique connections or incidents; this exact event signature avoids the demonstrated double-counting case but is not a universal connection deduplication mechanism.

## Field extraction — verified
An `awk` field-index diagnostic confirmed these mappings for the displayed journal format: `$3` time; `$8` attempted username; `$10` source IP; `$12` source port. The first proposed indexes `$9` and `$11` were an indexing mistake; they would return `from` and `port` respectively.

```bash
sudo journalctl -u ssh --since today --no-pager | grep -E 'sshd-session\[[0-9]+\]: Invalid user ' | awk '{for (i=1; i<=NF; i++) print i, $i}'
```

Confirmed sequence: `1 Oct`, `2 31`, `3 06:43:41`, `4 lab-srv01`, `5 sshd-session[1343]:`, `6 Invalid`, `7 user`, `8 invalidlabuser`, `9 from`, `10 10.10.30.10`, `11 port`, `12 47000`.

**Final tested extraction command:**

```bash
sudo journalctl -u ssh --since today --no-pager | grep -E 'sshd-session\[[0-9]+\]: Invalid user ' | awk '{print "Time:", $3, "| Username:", $8, "| Source IP:", $10, "| Port:", $12}'
```

**Operator confirmed actual output matched expected:**

```text
Time: 06:43:41 | Username: invalidlabuser | Source IP: 10.10.30.10 | Port: 47000
```

**Caveat:** fixed whitespace field indices are specific to the observed rendering and may break if journal formatting changes; structured journal fields or a validated parser are preferable for future SIEM collection.

## Decisions / handoff
- Current network segmentation / SSH monitoring exercise has a verified, portfolio-ready endpoint. GitHub update is **planned**, not yet claimed as pushed.
- Start a **separate custom SIEM project** rather than immediately installing Wazuh. Selectively use reputable open-source libraries while writing collection, normalization, detection, storage, alerting, and visualization logic ourselves.
- **No collector, SIEM VM, database, detection daemon, dashboard, or repository has been deployed or created yet.**
- Initial SIEM development can use the existing Ubuntu journal for a read-only proof of concept, then migrate to a separately segmented analytics node after resource and network review.
- Continue documenting every verified result in both the lab journal and command reference; prefer single-line copyable commands.
- Open items inherited from earlier days: VM soft lockups (deferred); Windows Hyper-V-backed VirtualBox NEM performance; IPv6 segmentation not verified; temporary OPNsense external ICMP test rule cleanup not confirmed; snapshot baseline pending.

**Next network lab action:** Preserve current validated controls and prepare the GitHub summary. Begin any new detection experiments under the separate SIEM journal, using reproducible tests and explicit pass/fail evidence.
