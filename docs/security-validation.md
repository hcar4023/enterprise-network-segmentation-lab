# Security Validation Report

**Project:** Enterprise Network Segmentation & SSH Security Monitoring Lab  
**Evidence cutoff:** October 31, 2026  
**Environment:** Windows 11 / VirtualBox / OPNsense 26.7 / Kali Linux / Ubuntu Server  
**Scope:** Controlled virtual-lab testing only; this is **not** a comprehensive penetration test or security certification.

> **Assessment:** The configured IPv4 segmentation, SSH authentication controls, and SSH event monitoring behaved as intended **for the documented test cases**. IPv6 segmentation and broader protocol coverage remain unverified.

## 1. Test environment

| Node | Role | Network | IPv4 address |
| --- | --- | --- | --- |
| `LAB-FW01` | OPNsense firewall | WAN via VirtualBox NAT | `10.0.2.15/24` (observed DHCP) |
| `LAB-FW01` | LAN / attack-side gateway | `LAB-ATTACK` | `10.10.30.1/24` |
| `LAB-FW01` | OPT1 / server-side gateway | `LAB-SERVER` | `10.10.20.1/24` |
| `LAB-KALI01` | Authorized test client | `LAB-ATTACK` | `10.10.30.10/24` |
| `LAB-SRV01` | Protected SSH server | `LAB-SERVER` | `10.10.20.10/24` |

The segments use **VirtualBox Internal Networks**, not physical VLANs or 802.1Q tagging. Inter-segment traffic crosses the OPNsense firewall.

## 2. Firewall policy validation

### Relevant configured IPv4 policy

| Rule / control | Interface / direction | Match | Action | Verification |
| --- | --- | --- | --- | --- |
| `ALLOW_KALI_TO_UBUNTU_SSH` | LAN / in | TCP from `10.10.30.10/32` to `10.10.20.10/32`, destination port 22 | Pass, log, quick | TCP/22 reported open and interactive SSH succeeded |
| `BLOCK_ATTACK_TO_SERVER` | LAN / in | IPv4 traffic from `10.10.30.0/24` to `10.10.20.0/24` | Block, log, quick | ICMP block seen in Live View after broad default allow disabled |
| Default LAN IPv4 allow | LAN | Broad default pass | **Disabled** | The observed Kali-to-Ubuntu ICMP behavior changed to blocked |

### Recorded test matrix

| Test ID | Test / observation | Actual recorded result | Assessment |
| --- | --- | --- | --- |
| NET-01 | Kali → Ubuntu TCP/22 using Nmap | `22/tcp open ssh syn-ack` | **Pass:** intended SSH reachability |
| NET-02 | Kali → Ubuntu ICMP (`ping -c 4`) | 4 packets sent, 0 received; OPNsense Live View recorded `BLOCK_ATTACK_TO_SERVER` | **Pass:** tested ICMP blocked |
| NET-03 | Kali → OPNsense management HTTPS | `curl -kI` received HTTP/2 403 from OPNsense; GUI was previously accessible | **Observed:** HTTPS service responds; HEAD response does **not** prove authenticated GUI access |
| NET-04 | Kali → LAN gateway ICMP after broad allow disabled | 100% packet loss | **Observed:** gateway ICMP not allowed by tested policy |
| NET-05 | Ubuntu → OPT1 gateway ICMP under its scoped pass rule | 0% packet loss | **Pass:** tested gateway reachability |

**Important limitation:** Neither an ICMP block nor a single open TCP port establishes an exhaustive default-deny guarantee across all IPv4 protocols, sources, ports, or IPv6. The historical reason the initial ICMP pass persisted despite a displayed block rule was not conclusively identified. Disabling the broad default LAN IPv4 pass produced the desired observed outcome.

### Reproduction commands (execute from Kali unless noted)

```bash
nmap -Pn -sT -p 22 --reason 10.10.20.10
ping -c 4 10.10.20.10
curl -kI --max-time 10 https://10.10.30.1
```

These commands document the test method; their recorded outcomes above are historical, not a claim of a fresh retest.

## 3. SSH configuration and authentication validation

The effective Ubuntu OpenSSH configuration was verified using `sudo sshd -T` after moving the hardening drop-in ahead of a conflicting cloud-init configuration file.

| Effective setting | Verified value |
| --- | --- |
| `permitrootlogin` | `no` |
| `pubkeyauthentication` | `yes` |
| `passwordauthentication` | `no` |
| `kbdinteractiveauthentication` | `no` |
| `x11forwarding` | `no` |
| `maxauthtries` | `3` |

The working hardening drop-in was `/etc/ssh/sshd_config.d/01-lab-hardening.conf`. An earlier `99-lab-hardening.conf` was insufficient because `/etc/ssh/sshd_config.d/50-cloud-init.conf` supplied `PasswordAuthentication yes` first. The revised configuration passed `sudo sshd -t` and was reloaded.

| Test ID | Validation | Actual recorded result | Assessment |
| --- | --- | --- | --- |
| SSH-01 | Key-only login from Kali using a dedicated Ed25519 private key | Successful shell login after key-passphrase prompt | **Pass** |
| SSH-02 | Client configured to disallow public-key authentication and try password/keyboard-interactive | `Permission denied (publickey)` | **Pass:** password-only path rejected in test |
| SSH-03 | Inspect effective SSH daemon policy | Root login disabled; password and keyboard-interactive login disabled | **Pass:** configuration verified |

**Scope qualification:** `PermitRootLogin no` was confirmed in effective configuration, not through a separately documented root-login attempt. An SSH key passphrase is distinct from server-side password authentication. The SSH private key remains on Kali and is **not included** in this repository.

## 4. SSH security event generation and detection

### Controlled test

A single connection attempt was made from Kali using a nonexistent Ubuntu username:

```bash
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password -o ConnectTimeout=5 invalidlabuser@10.10.20.10
```

Ubuntu recorded these associated messages at **2026-10-31 06:43:41 UTC** (October 30, 23:43:41 Arizona time):

```text
Oct 31 06:43:41 lab-srv01 sshd-session[1343]: Invalid user invalidlabuser from 10.10.30.10 port 47000
Oct 31 06:43:41 lab-srv01 sshd-session[1343]: Connection closed by invalid user invalidlabuser 10.10.30.10 port 47000 [preauth]
```

These are **two log records associated with one observed connection**, not proof of two attempts. There was no established password failure or brute-force sequence.

### Detection/counting precision

Initial query (returned **2** matching lines):

```bash
sudo journalctl -u ssh --since today --no-pager | grep -Eic 'Invalid user'
```

This case-insensitive expression matched both messages. Narrowed query (returned **1**):

```bash
sudo journalctl -u ssh --since today --no-pager | grep -Ec 'sshd-session\[[0-9]+\]: Invalid user '
```

The refined expression resolved the demonstrated double count. It is **not** a general-purpose unique-session deduplication algorithm.

### Verified field extraction

```bash
sudo journalctl -u ssh --since today --no-pager | grep -E 'sshd-session\[[0-9]+\]: Invalid user ' | awk '{print "Time:", $3, "| Username:", $8, "| Source IP:", $10, "| Port:", $12}'
```

Recorded output:

```text
Time: 06:43:41 | Username: invalidlabuser | Source IP: 10.10.30.10 | Port: 47000
```

**Engineering limitation:** `awk`'s positional fields depend on this journal text layout. Future SIEM ingestion should use structured journal output or a tested parsing strategy, normalize timestamps, and support event/session correlation.

## 5. Supporting operational validation: secure time synchronization

Ubuntu initially reported no synchronized time source. OPNsense logged a blocked TCP/4460 flow used for NTS key exchange. Scoped OPT1 outbound rules were configured for Ubuntu `10.10.20.10/32` (TCP destination 4460 for NTS-KE; UDP destination 123 for NTP). Afterward, `chronyc sources -v` showed usable Canonical sources and a selected `^*` source; the operator confirmed `chronyc tracking` showed `Leap status: Normal` and `timedatectl` showed `System clock synchronized: yes`.

**Assessment:** Time synchronization was restored for the recorded test. This does not establish that clock synchronization caused or resolved VirtualBox guest soft lockups. Destination `any` on the NTS/NTP port-scoped rules remains broader than a trusted-server address allowlist.

## 6. Open findings and coverage gaps

| ID | Status | Required follow-up |
| --- | --- | --- |
| GAP-01 | **Unverified** | Audit IPv6 firewall policy and test IPv6 reachability; SSH was observed listening on IPv6. |
| GAP-02 | **Cleanup unconfirmed** | Check whether the temporary `TEST_UBUNTU_EXTERNAL_ICMP` OPT1 rule remains enabled and remove/disable if no longer needed. |
| GAP-03 | **Deferred** | Investigate Ubuntu kernel soft lockups if they affect operation. VirtualBox NEM / Windows hypervisor execution was verified, but causal attribution is unresolved. |
| GAP-04 | **Pending** | Confirm clean VM snapshots and recovery baseline. |
| GAP-05 | **Coverage limitation** | Extend testing to other ports, protocols, source hosts, and both directions before asserting comprehensive segmentation. |
| GAP-06 | **Separate future project** | Develop the custom SIEM; no custom collector, database, alerting, or dashboard is reported as implemented here. |

## 7. Conclusion and evidence provenance

The completed tests support a **scoped, reproducible validation** of a two-segment virtual network: the authorized SSH pathway remained reachable while the tested inter-segment ICMP pathway was blocked; Ubuntu's effective SSH configuration rejected password-only access while allowing Ed25519 key authentication; and a controlled invalid-user event was generated, counted precisely, and parsed into relevant fields.

All results in this report are drawn from the project journal and operator-confirmed observations through **October 31, 2026**. Detailed timestamps, configuration decisions, troubleshooting history, and unresolved issues appear in [`lab-journal.md`](lab-journal.md). This report does not represent a new execution of the tests.
