# BUILD GUIDE · Endpoint Hardening to CIS Benchmark

How to reproduce the lab end to end: take a default Windows 11 host and a default Ubuntu 22.04 host,
score each against its CIS Benchmark, harden to Level 1, document the controls you deliberately leave
off, and rescan to show the improvement. Everything runs in throwaway VMs with snapshots, so any
change you make is reversible in one click.

> **Cost / license note:** This lab runs on free tooling. The assessor (CIS-CAT Lite) and the Benchmark PDFs both
> download from CIS behind an email registration. The ready-made remediation GPOs (CIS Build Kits)
> are **members-only** (they need a paid CIS SecureSuite membership), so this guide hardens the hosts
> the free way instead: by hand through Local Group Policy / Local Security Policy on Windows, and with
> a script on Linux. Microsoft's Security Compliance Toolkit (also free) is shown as the bulk option
> for Windows. Host the two VMs locally (free) or as small Azure VMs (a few cents an hour, deleted at the
> end, see Phase 0b); the hardening is the same either way.

> **Safety:** Harden VMs, never your daily driver. Snapshot each VM *clean* before you touch it
> (Phase 0). Some CIS settings (SMBv1 off, anonymous restrictions, NTLMv2-only) can cut a host off
> from older devices. That's the point, but you want the rollback there.

---

## Phase 0: Prerequisites and safe setup

- [ ] Two hosts: **Windows 11** (or Windows Server 2022) and **Ubuntu 22.04 LTS**. Run them as local VMs
      (Hyper-V or VirtualBox) or as Azure VMs; for the Azure route and how to pick a size, region, and
      image, see **Phase 0b**.
- [ ] Take a **clean snapshot** of each before any change. Name them `clean-baseline`.
- [ ] Download **CIS-CAT Lite** (register at the CIS site → "CIS-CAT Lite"). Recent builds ship a
      bundled Java runtime; if yours doesn't, install Temurin/OpenJDK 8+.
- [ ] Download the matching **CIS Benchmark PDFs** (free, email-gated): *CIS Microsoft Windows 11
      Enterprise Benchmark* and *CIS Ubuntu Linux 22.04 LTS Benchmark*. These are the source of truth
      for every setting and exception.
- [ ] On the Ubuntu host: `sudo apt update`.

> **Why CIS-CAT Lite:** it's the free assessor, and it covers exactly the two benchmarks this lab uses
> (Windows 10/11 and Ubuntu 22.04). The Pro assessor and the Build Kits are SecureSuite-only; you don't
> need them to demonstrate the skill.

## Phase 0b: Hosting the VMs on Azure (what I used)

I ran both hosts as Azure VMs rather than local Hyper-V. The hardening steps are the same wherever the
VM lives; only the provisioning differs. The exact size and region I picked may not be available to you,
so here's how to choose your own.

**Create the Ubuntu host (Azure portal):**

1. **Resource group.** Make a fresh one (I used `rg-hardening-lab`) so the whole lab deletes in one click
   at the end.
2. **Region.** Pick one with spare capacity. Small VM sizes get capacity-restricted region by region, so
   if a size shows "Size not available," switch regions and try again. Big regions like East US and
   Central US usually have room. To check before you commit, `az vm list-usage -l <region>` shows your
   own vCPU quota per family.
3. **Image.** This is the step people get wrong. In the Marketplace, set the **Publisher name** filter to
   **Canonical**, then pick **Ubuntu Server 22.04 LTS - x64 Gen2** (free). Skip:
   - the third-party rebuilds (cloudimg, Ntegral, and the like) that add an hourly software surcharge on
     top of compute,
   - the **Ubuntu Pro / FIPS / Confidential** plans, which are paid and not a clean baseline,
   - and anything labeled **"CIS Hardening"** or "pre-hardened." Starting from a hardened image defeats
     the exercise. You want a default install so the before-and-after scan shows the work.
4. **Size.** Any small general-purpose size is plenty; 2 vCPU and 4 GB covers Lynis and the hardening
   comfortably. Try the cheapest first (a B-series like `B2s` or `B2as_v2`). If it's greyed out or throws
   a capacity error, the portal only lists sizes that will actually deploy, so take any available small
   one. A `Dasv5` or a current `D`-series works. I landed on `D2s_v7` because the B-series was
   capacity-blocked in my region that day.
5. **Authentication.** SSH public key, username `azureuser`, generate a new key pair, and download the
   `.pem`.
6. **Inbound ports.** Allow SSH (22). Better: scope the network security group rule's source to your own
   public IP so the box isn't open to the whole internet.
7. **Create.** A size this small costs a few cents an hour, and you delete the resource group at the end
   (Phase 10), so the whole run stays well under a dollar.

**Connect:**

```bash
# Windows refuses a key file other accounts can read, so lock it down first:
icacls "<path>\<key>.pem" /inheritance:r /grant:r "<your-username>:R"
ssh -i "<path>\<key>.pem" azureuser@<PUBLIC_IP>
```

On Linux or macOS the equivalent is `chmod 600 <key>.pem`. The first connection asks you to trust the
host key; answer `yes`.

> A real VM (Azure or local Hyper-V) beats WSL here: auditd, the host firewall, and the kernel `sysctl`
> settings all behave normally. WSL skips or fakes several of them, which skews the Lynis score.

## Phase 1: Windows baseline scan (CIS-CAT Lite)

1. On the **clean** Windows VM, unzip CIS-CAT Lite and run **`Assessor-GUI`** (or
   `Assessor-CLI.bat` for the command line).
2. Select the **CIS Microsoft Windows 11 Enterprise Benchmark**, profile **Level 1 (L1)**. L1 is the
   "safe for general use" profile; L2 trades usability for hardening and is out of scope here.
3. Run the assessment. When it finishes, open the **HTML report** and note the **overall compliance
   score** (it'll be low on a default install, your "before" number).
   📸 `cis-01-windows-baseline-scan.png` (the report header with the failing score).

> Keep the report. Save it under `assets/` as a working file if you want, but the screenshot of the
> score is what the write-up references.

## Phase 2: Harden Windows to Level 1

Two ways to apply the settings. **Option A** is manual and transparent (you see every control);
**Option B** bulk-applies a baseline. Do A to learn it; reach for B when you'd do this at scale.

### Option A: by hand (Local Security Policy / Local Group Policy / registry)

Run an elevated PowerShell / `secpol.msc` and apply a representative L1 set:

```powershell
# Account & lockout policy (CIS 1.x): length, history, lockout
net accounts /minpwlen:14 /maxpwage:365 /minpwage:1 /uniquepw:24 `
             /lockoutthreshold:5 /lockoutduration:15 /lockoutwindow:15

# Remove SMBv1 entirely (CIS 18.x / legacy protocol)
Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol -NoRestart
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force

# NTLMv2 only, refuse LM & NTLM (CIS 2.3.11.x)
Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Lsa' LmCompatibilityLevel 5

# Restrict anonymous enumeration of SAM accounts and shares (CIS 2.3.10.x)
Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Lsa' RestrictAnonymous 1
Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Lsa' RestrictAnonymousSAM 1
Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Lsa' EveryoneIncludesAnonymous 0

# UAC: admin approval mode, consent on the secure desktop (CIS 2.3.17.x)
$sys = 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System'
Set-ItemProperty $sys EnableLUA 1
Set-ItemProperty $sys ConsentPromptBehaviorAdmin 2
Set-ItemProperty $sys PromptOnSecureDesktop 1

# Disable AutoPlay/AutoRun on all drives (CIS 18.x)
Set-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer' NoDriveTypeAutoRun 255

# Disable the built-in Guest account (CIS 1.x)
Disable-LocalUser -Name Guest

# Windows Firewall on for every profile, default inbound block (CIS 9.x)
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True -DefaultInboundAction Block

# Advanced audit policy (CIS 17.x): enable the high-value subcategories
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Account Lockout" /success:enable /failure:enable
auditpol /set /subcategory:"Security Group Management" /success:enable /failure:enable
auditpol /set /subcategory:"Sensitive Privilege Use" /success:enable /failure:enable
```

📸 `cis-02-windows-gpo-settings.png` (e.g. the password/lockout policy in `secpol.msc`, or the SMBv1
feature showing removed).

> CIS specifies audit policy per subcategory, not the blanket `auditpol /set /category:*`. The four
> above are the ones a SOC actually reads; the Benchmark lists the full set.

### Option B: bulk via Microsoft Security Compliance Toolkit (free)

1. Download the **Security Compliance Toolkit (SCT)** and the **Windows 11 security baseline** from
   Microsoft, plus **LGPO.exe** (included in the toolkit).
2. From the baseline's `GPOs` folder: `LGPO.exe /g .\GPOs` to import the baseline into Local Group
   Policy, then `gpupdate /force`.
3. Microsoft's baseline is *closely aligned* with CIS L1 but **not identical**. Record the gap as an
   exception in Phase 3 rather than claiming a 1:1 CIS apply.

## Phase 3: Document the Windows exceptions

Open the CIS-CAT report's failing items and decide each one: fix it, or justify leaving it. Typical
lab exceptions, with honest reasons:

| Control | Decision | Justification |
|---|---|---|
| BitLocker / TPM settings | Not applied | VM has no TPM; full-disk encryption is environmental, re-enable on physical hardware. |
| RDP disabled | Kept enabled (NLA on) | Sole management path to the lab VM; mitigated by Network Level Authentication + host firewall scope. |
| Machine inactivity limit (very short) | Relaxed | Convenience on a single-user lab; noted as a deliberate deviation from L1. |

The point isn't zero exceptions. It's that every gap is a *decision on the record*, not an oversight.

## Phase 4: Rescan Windows

Re-run CIS-CAT Lite (same benchmark, L1). The overall score should jump. Note the "after" number.
📸 `cis-03-windows-rescan.png` (the improved score next to the same header).

## Phase 5: Linux baseline scan (Lynis)

On the **clean** Ubuntu VM:

```bash
sudo apt install -y lynis
sudo lynis audit system
```

Note the **Hardening index** (0 to 100) at the bottom, plus the warnings and suggestions. That's the
"before" number. 📸 `cis-04-linux-lynis-baseline.png`.

> Lynis isn't a formal CIS assessor, but its checks overlap the CIS Ubuntu Benchmark heavily and it
> gives a single trendable score, which is what we want for before/after. If you want a true CIS
> percentage on Linux too, CIS-CAT Lite also covers the Ubuntu 22.04 benchmark; run it the same way as
> Phase 1 and screenshot that score instead.

## Phase 6: Harden Ubuntu to CIS L1 (one control group at a time)

Don't paste a giant script and trust it. Work one control group at a time, and after each change
**verify it actually took effect** with the matching command. A config file that never saved, or a
service that never reloaded, looks identical to success until you check. Every unit below is: what it
defends against, the change, and the proof. The Lynis findings from Phase 5 are the syllabus.

### Unit 1: SSH (CIS 5.2 / `SSH-7408`)

SSH is the front door. These options shrink the attack surface and slow brute force. Use a **drop-in
file**, which overrides the main `sshd_config` through its `Include` line and survives package updates.
A here-doc is more reliable than a text editor, where it's easy to forget to save:

```bash
sudo tee /etc/ssh/sshd_config.d/60-cis.conf >/dev/null <<'EOF'
PermitRootLogin no
MaxAuthTries 3
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
LogLevel VERBOSE
EOF
sudo sshd -t && sudo systemctl restart ssh
```

Verify (this is the point), `sshd -T` prints the *effective* running config, so you prove the daemon
actually loaded your values rather than just that a file exists:

```bash
sudo sshd -T | grep -Ei 'maxauthtries|permitrootlogin|x11forwarding|allowtcpforwarding|allowagentforwarding|loglevel'
```

Things that bite you here:
- `MaxAuthTries 3` caps tries *per connection*, then disconnects. It does NOT ban the IP. That's
  fail2ban (Unit 3). The two pair up: this weakens each attempt, fail2ban bans the source.
- `AllowTcpForwarding no` stops a compromised session tunneling deeper into the network (pivoting, MITRE T1090).
- `ClientAliveInterval` may read back lower than you set (e.g. 120) if a lower-numbered Azure drop-in
  already sets it. Lower is stricter, so leave it.
- Run `sshd -t` before the restart so a typo can't down the daemon and lock you out. And use `&&` (run
  next only on success), not a single `&`, which backgrounds the first part and runs the next
  regardless, so a success message proves nothing.

📸 `cis-05-linux-ssh-hardening-before.png` / `-after.png` (the `sshd -T` grep, default vs hardened).

### Unit 2: Host firewall (CIS 3.5 / `FIRE-4512`)

Azure's NSG guards the network edge; CIS wants a firewall on the host too (defense in depth). **The
rule that bites everyone:** you're connected over SSH, so allow SSH *before* enabling a deny-all
firewall, or you lock yourself out.

```bash
sudo ufw allow OpenSSH            # open 22 FIRST
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable                   # answer 'y' to the ssh-disruption warning
sudo ufw status verbose           # verify: active, deny (incoming), 22/tcp ALLOW IN
```

A specific allow rule for 22 overrides the default-deny, so SSH stays up while everything else is
blocked. 📸 `cis-05-linux-firewall.png`.

### Unit 3: fail2ban (`DEB-0880` / T1110)

Watches the auth log and bans an IP at the firewall after repeated failures, completing the brute-force
story started in Unit 1.

```bash
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
sudo systemctl is-active fail2ban          # active
sudo fail2ban-client status sshd           # sshd jail is on by default on Ubuntu
```

📸 `cis-05-linux-fail2ban.png`.

### Unit 4: auditd + process accounting (`ACCT-9628` / `ACCT-9622`)

Prevention stops attacks; auditd gives you *visibility*, the forensic log of logins, privilege use, and
changes to sensitive files. It's the source a SIEM ingests (see project 01, the Protect vs Detect split).

```bash
sudo apt install -y auditd audispd-plugins acct
sudo systemctl enable --now auditd
sudo systemctl enable --now acct
sudo systemctl is-active auditd            # active
sudo auditctl -s | grep enabled            # enabled 1
```

📸 `cis-05-linux-auditd.png`.

### Unit 5: AIDE file integrity (`FINT-4350` / T1554)

auditd records events live; AIDE detects *lasting changes* to system files by comparing them against a
known-good snapshot. Build the snapshot now, while the box is clean.

```bash
sudo apt install -y aide aide-common
sudo aideinit                                            # slow: it hashes every system file
sudo cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

> **Gotcha:** installing AIDE pulls in Postfix (a mail server, for emailing reports). At its prompt,
> choose **`Local only`** and accept the default mail name. Don't stand up an internet-facing mail
> server on a box you're hardening; "Local only" binds it to localhost. That decision is attack-surface
> thinking in the wild, and worth a line in the write-up.

Verify the baseline exists, then prove it works by planting a change and watching AIDE flag it:

```bash
ls -lh /var/lib/aide/aide.db
sudo touch /etc/aide-test-file && sudo aide --check      # reports the file under "Added"
sudo rm /etc/aide-test-file
```

📸 `cis-05-linux-aide.png` (AIDE catching the planted file is strong evidence: the control *working*, not just installed).

### Units 6 to 8: kernel, accounts, attack surface

Same method (apply, then verify) for the rest of the Lynis findings:

- **Kernel hardening** (CIS 3.x / `KRNL-6000`): a `/etc/sysctl.d/60-cis.conf` covering redirects, source
  routing, `log_martians`, `rp_filter`, `tcp_syncookies`, ASLR, and the `kernel.*` restrictions. Apply
  with `sudo sysctl --system`; verify a key with `sudo sysctl kernel.kptr_restrict`.
- **Accounts / passwords** (`AUTH-9230/9286/9328/9262`): `libpam-pwquality` (minlen 14, complexity) plus
  `/etc/login.defs` (PASS_MAX_DAYS 365, PASS_MIN_DAYS 1, UMASK 027, hashing rounds).
- **Attack surface** (`NETW-3200`, `USB-1000`, `BANN-7126/7130`, `FILE-7524`, `HRDN-7230`, `PKGS-7392`):
  blacklist unused protocols, filesystems, and usb-storage via `/etc/modprobe.d/`; add a legal banner to
  `/etc/issue` and `/etc/issue.net`; tighten cron and grub file permissions; install `rkhunter` and
  `debsums`; apply pending security updates.

## Phase 7: Document the Linux exceptions

Each remaining Lynis item is a decision, not an oversight. The skips from this run:

| Control | Why |
|---|---|
| `kernel.modules_disabled` | blocks loading any module until reboot; too risky on a live host |
| `fs.protected_fifos=2` | Ubuntu's `99-protect-links.conf` resets it to 1 (sysctl drop-ins are last-wins) |
| GRUB password | single-user lab VM, no untrusted physical access |
| Separate `/home /tmp /var` partitions | single-disk lab VM; would partition on a real build |
| Remote logging | no external log host in the lab |
| Password policy | host uses SSH keys, so it is belt-and-suspenders here |

## Phase 8: Rescan Linux (Lynis)

```bash
sudo lynis audit system
```

The hardening index should be materially higher and the warning count lower. Note the "after" number.
📸 `cis-06-linux-lynis-rescan.png`.

## Phase 9: Exceptions register and before/after summary

Pull both hosts' numbers and your two exception tables into one short summary (a section in the README,
or a small `exceptions.md`). 📸 `cis-07-exceptions-register.png`.

**Evidence checklist (fills the README):**

- [ ] `cis-01-windows-baseline-scan.png` · [ ] `cis-02-windows-gpo-settings.png` · [ ] `cis-03-windows-rescan.png`
- [ ] `cis-04-linux-lynis-baseline.png` · [ ] `cis-05-linux-hardening-script.png` · [ ] `cis-06-linux-lynis-rescan.png`
- [ ] `cis-07-exceptions-register.png`

Copy the screenshots into this project's `assets/`, fill the before/after scores into the README's
results table, then flip the status from **In Progress** to **Documented**.

## Phase 10: Cleanup

- Local VMs: revert both to the `clean-baseline` snapshots (fastest, total rollback).
- Azure: delete the whole resource group to stop all charges: `az group delete -n rg-hardening-lab --yes`.
- Keep the CIS-CAT HTML reports and Lynis logs as working evidence if you like; the curated screenshots
  are what the write-up cites.

## Optional: make it stronger

- Add **CIS-CAT Lite on Ubuntu** for a true CIS percentage alongside the Lynis index, so both hosts
  report against the same kind of metric.
- Apply hardening with **Ansible** (`ansible-lockdown/UBUNTU22-CIS`, or the `dev-sec` hardening
  collection) instead of the hand script, to show repeatable, idempotent config management.
- Re-run the scans on a schedule and chart the drift. A baseline only matters if you keep measuring it.
