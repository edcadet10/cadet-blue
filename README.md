# 🛡️ cadet-blue — Blue-Team & IT Operations Portfolio

> Hands-on Security Operations and IT Operations projects across **Help Desk, SOC, Network, GRC, IAM, and Vulnerability Management** — backed by CompTIA A+, Network+, and Security+.

![CompTIA A+](https://img.shields.io/badge/CompTIA-A%2B-E2231A)
![CompTIA Network+](https://img.shields.io/badge/CompTIA-Network%2B-E2231A)
![CompTIA Security+](https://img.shields.io/badge/CompTIA-Security%2B-E2231A)
![Focus](https://img.shields.io/badge/Focus-SOC%20%2F%20Security%20Analyst-1f6feb)
![Lab](https://img.shields.io/badge/Lab-Azure%20%2B%20On--Prem%20AD-0078D4)

**Jeff Cadet** · Columbus, OH · Open to remote
[LinkedIn](https://www.linkedin.com/in/iamedcadet) · [GitHub](https://github.com/edcadet10)

---

## About this portfolio

I'm an IT professional moving into **security operations**. I hold CompTIA A+, Network+, and
Security+, and I learn by building: I run an on-prem Windows Server 2022 domain controller, an
Azure Entra ID lab, a ServiceNow service-desk instance, and Cisco Packet Tracer topologies.

This repo turns that hands-on work into ten focused projects — leading with the **SOC / Security
Analyst** skill set (detection engineering, incident response, vulnerability management, hardening)
and rounding out with the **IT operations** foundation that supports it (IAM, networking, service
desk, automation).

Each project is a self-contained write-up: the objective, the environment, what I did, the skills
it demonstrates, and the evidence. Status is labeled honestly — see the legend below.

## Certifications

| Certification | Status |
|---|---|
| CompTIA Security+ (SY0-701) | Passed — Jun 2026 |
| CompTIA Network+ (N10-009) | Passed — May 2026 |
| CompTIA A+ (220-1201/1202) | Passed — Dec 2025 |
| CompTIA CIOS / CSIS (stackable) | Earned via A+/Net+/Sec+ |

## Skills matrix

| Domain | Skills demonstrated | Evidence |
|---|---|---|
| **SOC / Detection** | SIEM (Microsoft Sentinel), KQL, log analysis, detection rules, MITRE ATT&CK, alert triage | [01](projects/01-siem-detection-lab/) |
| **Incident Response** | IR lifecycle, phishing analysis, containment/eradication/recovery, after-action reporting | [02](projects/02-incident-response-phishing/) |
| **Vulnerability Mgmt** | Authenticated scanning, risk ranking (CVSS), remediation tracking, rescan verification | [03](projects/03-vulnerability-management/) |
| **Endpoint Hardening** | CIS Benchmarks, baseline configuration, compliance scanning, Group Policy | [04](projects/04-endpoint-hardening-cis/) |
| **GRC** | NIST CSF, CIS Controls, risk register, POA&M, security policy authoring | [05](projects/05-grc-risk-assessment/) |
| **IAM** | Entra ID, RBAC, Conditional Access, MFA, least privilege, access reviews | [06](projects/06-iam-entra-id/) |
| **IAM / Automation** | Microsoft Graph, joiner/mover/leaver, automated provisioning & password reset | [07](projects/07-iam-automated-provisioning/) |
| **Networking** | VLANs, inter-VLAN routing, ACLs, DHCP/DNS, network segmentation | [08](projects/08-network-segmentation-lab/) |
| **Help Desk / ITSM** | ServiceNow, ITIL incident & request management, KB, SLAs, reporting | [09](projects/09-servicedesk-servicenow/) |
| **Automation / Applied AI** | Azure AI Foundry agent, tool/action APIs, IT workflow automation | [10](projects/10-ai-helpdesk-agent/) |

## Projects

**Status legend:** ✅ Documented (write-up + evidence complete) · 🚧 In Progress · 🗂️ Outlined (real work done, write-up pending) · 📋 Planned (new build)

| # | Project | Domain | Status |
|---|---------|--------|--------|
| 01 | [SIEM & Detection Engineering Lab (Microsoft Sentinel)](projects/01-siem-detection-lab/) | SOC | 🚧 In Progress |
| 02 | [Incident Response: Phishing → Compromise](projects/02-incident-response-phishing/) | SOC / IR | 📋 Planned |
| 03 | [Vulnerability Management Program](projects/03-vulnerability-management/) | Vuln Mgmt | 📋 Planned |
| 04 | [Endpoint Hardening to CIS Benchmark](projects/04-endpoint-hardening-cis/) | Vuln Mgmt | 📋 Planned |
| 05 | [Risk Assessment & GRC Package (NIST CSF / CIS Controls)](projects/05-grc-risk-assessment/) | GRC | 📋 Planned |
| 06 | [Identity & Access Management on Entra ID](projects/06-iam-entra-id/) | IAM | 🗂️ Outlined |
| 07 | [Automated Joiner/Mover/Leaver (Microsoft Graph)](projects/07-iam-automated-provisioning/) | IAM / Automation | 📋 Planned |
| 08 | [Network Segmentation Lab (Packet Tracer)](projects/08-network-segmentation-lab/) | Network | 🗂️ Outlined |
| 09 | [Service Desk Operations (ServiceNow / ITIL)](projects/09-servicedesk-servicenow/) | Help Desk | 🗂️ Outlined |
| 10 | [AI-Assisted IT Help Desk Agent (Azure AI Foundry)](projects/10-ai-helpdesk-agent/) | Automation / Applied AI | 🗂️ Outlined |

## How this repo is organized

```
projects/<NN>-<slug>/
  README.md        # the project write-up
  BUILD-GUIDE.md   # (some projects) step-by-step to reproduce the lab
  assets/          # diagrams and screenshots
```

Start with **[Project 01](projects/01-siem-detection-lab/)** for the SOC detection work.
