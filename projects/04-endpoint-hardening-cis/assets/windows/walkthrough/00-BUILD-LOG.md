# Windows host build log — Project 04 (Endpoint Hardening to CIS Benchmark)

A step-by-step record of standing up and hardening the Windows 11 host, including the
mistakes I hit and how I fixed them. Screenshots are the numbered files in this folder.
The curated evidence for the write-up is the `cis-*-windows-*.png` files.

## 1. Provision the Azure VM (rg-hardening-lab, East US)
- **01** Create VM, Basics tab. Size Standard_D2s_v7, Trusted launch, admin `cadetadmin`.
  - *Mistake I caught:* the image had defaulted to **Windows Server 2025 Datacenter**. This
    project targets the **Windows 11 Enterprise** benchmark, so I changed the image before deploying.
  - *Sizing note:* the smaller Standard_D2s_v5 was capacity-restricted in East US, so I used
    D2s_v7 (the same size my Linux host runs).
- **02** Marketplace image picker. The list is full of third-party "Enterprise for Windows 11"
  repackages that add an hourly surcharge, plus pre-hardened images that would defeat a
  before/after test. I chose the plain **Windows 11** image published by Microsoft.
- **03** Set it to Windows 11 Enterprise.
  - *Second correction:* I first selected version **25H2**, then changed it to **24H2** so it
    matches the CIS-CAT Lite benchmark version cleanly.
- **04** Allowed RDP (3389) inbound, then scoped the rule's source to my own public IP so the
  VM is not exposed to the whole internet.
- **05** Deployment finished.
- **06** Took a **clean-baseline** OS-disk snapshot before touching anything, as a rollback point.

## 2. Baseline scan
- **07** Connected over RDP to the Windows 11 desktop.
- **08** Downloaded **CIS-CAT Lite v4.64.0** and extracted it. This build ships its own Java
  runtime, so no separate Java install.
- **09-11** In the Assessor: **CIS Microsoft Windows 11 Enterprise Benchmark v5.0.1**, profile
  **Level 1**, Basic (scan this system), HTML report.
- **12** Baseline HTML report. Overall score **22%** (87 of 391 passed). That is the real
  default-install number. (`cis-01-windows-baseline-scan.png` is the summary shot.)
  - *Mistake:* my first screenshot of the 22% summary was pasted from the clipboard and never
    saved as a file, so I re-captured it from the saved HTML report.

## 3. Hardening to Level 1 (one control group at a time, verified after each)
- **13** Unit 1, account and lockout policy: min length 14, history 24, lockout 5 tries / 15 min.
  Verified with `net accounts`.
- **14** Unit 2, User Account Control: Admin Approval Mode, consent prompt, secure desktop.
  - *Minor slip:* I accidentally pasted an explanatory "Expect:" note into the console and got a
    "not recognized" error. Harmless, ignored it.
- **15** Unit 2 re-check in a new PowerShell tab. I ran the verify before re-declaring the `$sys`
  variable in that tab and got "Cannot bind argument to parameter 'Path' because it is null."
  Re-declared `$sys` and it passed. Reminder that PowerShell variables do not carry between windows.
- **16** Unit 3, NTLMv2 only plus blocking anonymous account and share enumeration. Verified
  `LmCompatibilityLevel 5` and the anonymous restrictions.

## 4. Rest of the hardening (units 4-6, then broader batches)
- **17** Unit 4, remove SMBv1 (the EternalBlue / WannaCry protocol). Windows 11 ships with it off,
  so this confirmed it: `State: Disabled`, `EnableSMB1Protocol: False`.
- **18** Unit 5, Windows Firewall on for all three profiles, default-deny inbound. The trap: I was
  connected over RDP, so I enabled the Remote Desktop allow rule *before* the deny-all, or I would
  have cut myself off. Same lesson as "allow SSH before deny-all" on the Linux host.
- **19** Unit 6, advanced audit policy (logon, lockout, privileged use, group changes) to Success
  and Failure, so security events actually get logged.

## 5. Decision: skip the vendor baseline, harden by hand
I researched bulk-applying Microsoft's Windows 11 security baseline with LGPO and found it sets
"Deny log on through Remote Desktop Services" to include the local-account group. On this remote-only
VM that would have locked me out (and it re-applies itself on Azure). So I hardened by hand instead,
in batches, rescanning to measure. The RDP-deny stayed scoped to Guests only.

- **20** Batch 1, registry controls (SMB signing, LSA/credential protection, MSS TCP/IP, interactive
  logon, full UAC set, WinRM, RDP hardening, PowerShell script-block logging).
- **21** Batch 2 and 3, full advanced audit policy, and Defender Attack Surface Reduction rules + PUA/MAPS.
- **22** Batch 4, 5, 6: Windows Firewall GP policy for all profiles + logging; User Rights Assignments
  and password complexity via `secedit` (RDP-deny = Guests only, on purpose); risky services disabled;
  NTLM/Kerberos security options.
- **23** Batch 7, Administrative Templates (lock screen, MS Security Guide, network, printers, system,
  Defender/SmartScreen).
- **24** Batch 8, corrections + the remaining admin templates (logon UI, event-log sizing, RDP
  hardening, Search/Cortana, Defender scan settings, LAPS, Windows Update, VBS/Credential Guard config).
  - *Mistake caught by the rescan:* a few of my Batch registry values first landed under the wrong key
    (for example `LocalAccountTokenFilterPolicy` under `Lsa` instead of `Policies\System`) and quietly
    did nothing. The before-and-after scan is what caught them, and Batch 8 corrected the paths.

## Result
Score climbed **22% → 38% → 59% → 74%** (87 → 290 of 391 passed), all by hand, no lockout. The 101
still failing are documented in the README exceptions register: BitLocker (needs a TPM), VBS /
Credential Guard (activate on reboot, deferred on a throwaway VM), the new 24H2 SMB-audit controls,
the two deliberate safety choices, and a long tail of newer and per-user items.

## Cleanup
Delete the whole resource group to stop the VM billing: `az group delete -n rg-hardening-lab --yes`.
