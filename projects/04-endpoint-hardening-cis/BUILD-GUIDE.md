# BUILD GUIDE · Endpoint Hardening to CIS Benchmark

How to reproduce the lab end to end: take a default Windows 11 host and a default Ubuntu 22.04 host,
score each against its CIS Benchmark, harden to Level 1, document the controls you deliberately leave
off, and rescan to show the improvement. Everything runs in throwaway VMs with snapshots, so any
change you make is reversible in one click.

> **Cost / license note:** This lab is free. The assessor (CIS-CAT Lite) and the Benchmark PDFs both
> download from CIS behind an email registration. The ready-made remediation GPOs (CIS Build Kits)
> are **members-only** (they need a paid CIS SecureSuite membership), so this guide hardens the hosts
> the free way instead: by hand through Local Group Policy / Local Security Policy on Windows, and with
> a script on Linux. Microsoft's Security Compliance Toolkit (also free) is shown as the bulk option
> for Windows. No cloud resources, so nothing bills.

> **Safety:** Harden VMs, never your daily driver. Snapshot each VM *clean* before you touch it
> (Phase 0). Some CIS settings (SMBv1 off, anonymous restrictions, NTLMv2-only) can cut a host off
> from older devices. That's the point, but you want the rollback there.

---

## Phase 0: Prerequisites and safe setup

- [ ] Two VMs in Hyper-V or VirtualBox: **Windows 11** (or Windows Server 2022) and **Ubuntu 22.04 LTS**.
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

## Phase 6: Harden Ubuntu to CIS L1

Apply a representative L1 set. This is a transparent script, not the full benchmark. The CIS PDF has
~200 items; this hits the high-impact ones. Snapshot first.

```bash
#!/usr/bin/env bash
set -euo pipefail

# --- SSH hardening (CIS 5.2.x): edit /etc/ssh/sshd_config ---
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/'        /etc/ssh/sshd_config
sudo sed -i 's/^#\?MaxAuthTries.*/MaxAuthTries 4/'               /etc/ssh/sshd_config
sudo sed -i 's/^#\?LoginGraceTime.*/LoginGraceTime 60/'          /etc/ssh/sshd_config
sudo sed -i 's/^#\?X11Forwarding.*/X11Forwarding no/'            /etc/ssh/sshd_config
sudo sed -i 's/^#\?PermitEmptyPasswords.*/PermitEmptyPasswords no/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?ClientAliveInterval.*/ClientAliveInterval 300/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?ClientAliveCountMax.*/ClientAliveCountMax 0/'  /etc/ssh/sshd_config
sudo systemctl restart ssh
# Note: do NOT add "Protocol 2"; the directive was removed in OpenSSH 7.6+ and will error on 22.04.

# --- Host firewall (CIS 3.5.x) ---
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw --force enable

# --- Password quality & aging (CIS 5.3.x / 5.4.x) ---
sudo apt install -y libpam-pwquality
sudo sed -i 's/^# minlen.*/minlen = 14/'   /etc/security/pwquality.conf
sudo sed -i 's/^# dcredit.*/dcredit = -1/' /etc/security/pwquality.conf
sudo sed -i 's/^# ucredit.*/ucredit = -1/' /etc/security/pwquality.conf
sudo sed -i 's/^# ocredit.*/ocredit = -1/' /etc/security/pwquality.conf
sudo sed -i 's/^# lcredit.*/lcredit = -1/' /etc/security/pwquality.conf
sudo sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS 365/' /etc/login.defs
sudo sed -i 's/^PASS_MIN_DAYS.*/PASS_MIN_DAYS 1/'   /etc/login.defs
sudo sed -i 's/^PASS_WARN_AGE.*/PASS_WARN_AGE 7/'   /etc/login.defs

# --- Kernel network hardening via sysctl (CIS 3.2.x / 3.3.x) ---
sudo tee /etc/sysctl.d/60-cis.conf >/dev/null <<'EOF'
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.ip_forward = 0
kernel.randomize_va_space = 2
EOF
sudo sysctl --system

# --- Disable unused filesystems (CIS 1.1.1.x) ---
sudo tee /etc/modprobe.d/cis-fs.conf >/dev/null <<'EOF'
install cramfs /bin/true
install freevxfs /bin/true
install jffs2 /bin/true
install hfs /bin/true
install hfsplus /bin/true
install udf /bin/true
EOF

# --- Auditing & file integrity (CIS 4.1.x / 1.3.x) ---
sudo apt install -y auditd audispd-plugins aide aide-common
sudo systemctl enable --now auditd
sudo aideinit            # builds the baseline DB (takes a few minutes)

# --- Automatic security updates (CIS 1.9 / patch hygiene) ---
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -f noninteractive unattended-upgrades
```

For account lockout, Ubuntu 22.04 uses **`pam_faillock`**. Set `deny = 5` and `unlock_time = 900` in
`/etc/security/faillock.conf`, then wire it into `/etc/pam.d/common-auth` and `common-account` (CIS
5.4.2). Edit PAM carefully and keep a root shell open while you test, so a bad stanza can't lock you
out. 📸 `cis-05-linux-hardening-script.png` (the script running, or a `diff` of `sshd_config`).

## Phase 7: Document the Linux exceptions

Same discipline as Windows: each remaining Lynis warning is a decision:

| Control | Decision | Justification |
|---|---|---|
| `PasswordAuthentication yes` | Kept (for now) | Key-based auth not yet provisioned in the lab; tracked to switch to keys, exception noted. |
| GRUB bootloader password | Not set | Single-user lab VM with no untrusted physical access; would set on shared/physical hosts. |
| CUPS / printing service | Left enabled | Needed for a lab task; disable on a server-role build. |

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

- Revert both VMs to the `clean-baseline` snapshots (fastest, total rollback).
- Keep the CIS-CAT HTML reports and Lynis logs as working evidence if you like; the curated screenshots
  are what the write-up cites.

## Optional: make it stronger

- Add **CIS-CAT Lite on Ubuntu** for a true CIS percentage alongside the Lynis index, so both hosts
  report against the same kind of metric.
- Apply hardening with **Ansible** (`ansible-lockdown/UBUNTU22-CIS`, or the `dev-sec` hardening
  collection) instead of the hand script, to show repeatable, idempotent config management.
- Re-run the scans on a schedule and chart the drift. A baseline only matters if you keep measuring it.
