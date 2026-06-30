# 04 · Endpoint Hardening to CIS Benchmark

![Domain](https://img.shields.io/badge/Domain-Endpoint%20Hardening-1f6feb)
![Cert](https://img.shields.io/badge/Maps%20to-Security%2B%20%2F%20A%2B-E2231A)
![Status](https://img.shields.io/badge/Status-In%20Progress-dbab09)

![Endpoint Hardening: the path to CIS compliance](assets/infographic-endpoint-hardening.png)

> Take a default Windows 11 and Ubuntu 22.04 host to their CIS Level 1 baselines, scoring each before
> and after, and documenting every control I deliberately skip. **Linux host done: Lynis 59 → 80.**
> Windows host is the remaining half.

## Objective

A default OS ships for convenience, not safety: legacy protocols on, weak auth, almost nothing logged.
Hardening closes those gaps against a recognized baseline. The honest part is recording what I *didn't*
apply and why. This is the other side of [project 03](../03-vulnerability-management/) (which finds the
gaps) and the preventive half of [project 01](../01-siem-detection-lab/) (which detects the attack).

## Environment / Tools

| | Detail |
|---|---|
| Hosts | Windows 11, Ubuntu 22.04 LTS. Throwaway VMs (Linux on Azure), clean snapshot first |
| Benchmark | CIS Windows 11 / CIS Ubuntu 22.04, **Level 1** |
| Assessors | CIS-CAT Lite (Windows %), Lynis (Linux hardening index) |
| Apply (Linux) | drop-in configs + packages, one control group at a time, verified after each |
| Apply (Windows) | Local Security Policy / registry; MS Security Compliance Toolkit for bulk |

## Linux host: 8 control groups (59 → 80)

Worked one group at a time, proving each change with the matching verify command before moving on. Full
commands and gotchas in the [build guide](BUILD-GUIDE.md).

| # | Control group | Defends against | Verified with |
|---|---|---|---|
| 1 | SSH drop-in (`60-cis.conf`) | brute force, pivoting | `sshd -T` shows hardened values |
| 2 | `ufw` host firewall | open ports (FIRE-4512) | `ufw status` active, SSH allowed |
| 3 | fail2ban | brute force (bans the IP) | `fail2ban-client status sshd` |
| 4 | auditd + process accounting | no forensic trail | `auditctl -s` enabled |
| 5 | AIDE file integrity | tampering (89k files baselined) | 20M `aide.db` built |
| 6 | kernel `sysctl` (14 keys) | spoofing, DoS, memory exploits | `sysctl` shows ASLR, rp_filter |
| 7 | password policy + umask | weak, stale credentials | `pwquality.conf`, `login.defs` |
| 8 | attack-surface cleanup | unused drivers, no banner | banner, blacklist, rkhunter |

## Exceptions (deliberately skipped, on the record)

| Control | Why |
|---|---|
| `kernel.modules_disabled` | blocks loading any module until reboot; too risky on a live host |
| `fs.protected_fifos=2` | Ubuntu's `99-protect-links.conf` resets it to 1 (sysctl drop-ins are last-wins) |
| GRUB password | single-user lab VM, no untrusted physical access |
| Separate `/home /tmp /var` partitions | single-disk lab VM; would partition on a real build |
| Remote logging | no external log host in the lab |
| Password policy | host uses SSH keys, so it is belt-and-suspenders here |

## Results / Evidence

| Host | Tool | Baseline | Hardened |
|---|---|---|---|
| Ubuntu 22.04 | Lynis index | **59** | **80** |
| Windows 11 | CIS-CAT Lite L1 | _pending_ | _pending_ |

**Linux** (done), in [`assets/linux/`](assets/linux/):
- [x] baseline 59: `cis-04-linux-lynis-baseline.png` + `cis-04-lynis-baseline.txt`
- [x] SSH before/after, firewall, fail2ban, auditd, AIDE: `cis-05-linux-{ssh-hardening-before,ssh-hardening-after,firewall,fail2ban,auditd,aide}.png`
- [x] sysctl proof `cis-05-linux-sysctl.txt`; `cis-05-linux-password-policy.png`; `cis-05-linux-attack-surface.png`
- [x] rescan 80: `cis-06-linux-lynis-rescan.png`

**Windows** (pending), in [`assets/windows/`](assets/windows/):
- [ ] `cis-01-windows-baseline-scan.png`, `cis-02-windows-gpo-settings.png`, `cis-03-windows-rescan.png`

## Lessons learned

- **Verify, don't trust the happy message.** An SSH config that never saved looked identical to success.
  `sshd -T` (the effective config) caught it; a screenshot of "OK" would have shipped a false claim.
- **A benchmark isn't version-agnostic.** CIS's old `Protocol 2` directive now *errors* on Ubuntu
  22.04's OpenSSH. Apply controls against the OS in front of you.
- **`sysctl` drop-ins are last-wins; SSH config is first-wins.** File ordering decides which one applies.
- **Hardening tools add surface too.** AIDE pulled in Postfix (a mail server); the hardening move was
  to choose "Local only" rather than expose it.
- **Exceptions are the deliverable.** The score is nice; the table of what you skipped and why is what a
  reviewer actually reads.

## Mapped to

- **CIS Controls v8, Control 4** (Secure Configuration). This project is Control 4.
- **CompTIA Security+ (SY0-701)** Domains 1 and 4; **A+** OS fundamentals.
- **MITRE ATT&CK (mitigations):** M1042 (remove SMBv1), M1027 (passwords), M1028 (OS config / sysctl),
  M1047 (auditd), raising the cost of T1110 (brute force), T1090 (pivoting), T1554 (tampering).

## Reproduce

Step by step: [BUILD-GUIDE.md](BUILD-GUIDE.md).
