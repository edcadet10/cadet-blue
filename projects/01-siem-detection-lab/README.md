# 01 · SIEM & Detection Engineering Lab (Microsoft Sentinel)

![Domain](https://img.shields.io/badge/Domain-SOC-1f6feb)
![Cert](https://img.shields.io/badge/Maps%20to-Security%2B-E2231A)
![Status](https://img.shields.io/badge/Status-Documented-2ea44f)

> Stood up Microsoft Sentinel on a fresh Log Analytics workspace, streamed Microsoft Entra ID
> identity logs into it, and built an **identity-threat detection for privileged-role assignment** —
> then validated it end-to-end by elevating a test account and tracing the event through the pipeline.

## Objective

Build the security analyst's core loop — **collect → detect → triage** — in a live Azure tenant.
The first detection targets a high-value attacker behavior: **privilege escalation via directory
role assignment** (an attacker or insider granting an account elevated rights). The goal is a
detection that fires on the real behavior, validated against a controlled simulation.

## Environment / Tools

| Component | Detail |
|---|---|
| SIEM | **Microsoft Sentinel** (managed in the unified **Microsoft Defender portal**) |
| Workspace | **Log Analytics** `law-soc-lab` (resource group `rg-soc-lab`, East US 2) |
| Identity source | **Microsoft Entra ID** → **Diagnostic setting** `entra-to-law` streaming **AuditLogs** + **SignInLogs** |
| Query language | **KQL** (Kusto Query Language) |
| Framework | **MITRE ATT&CK** |
| Test subject | `soc-test01` (standard member account, created for the lab) |

## Architecture

```mermaid
flowchart LR
    subgraph Entra["Microsoft Entra ID"]
        AUD["Audit logs<br/>(role changes, user mgmt)"]
        SGN["Sign-in logs<br/>(needs Entra ID P1)"]
    end
    DS["Diagnostic setting<br/>entra-to-law"]
    LAW[("Log Analytics<br/>law-soc-lab")]
    SENT["Microsoft Sentinel<br/>Analytics rule + Incident"]
    AN(["SOC analyst<br/>triage & investigate"])

    AUD --> DS
    SGN --> DS
    DS --> LAW
    LAW --> SENT
    SENT -->|"Alert -> Incident"| AN
```

## What I did

1. **Created the workspace** — new resource group `rg-soc-lab` and Log Analytics workspace `law-soc-lab`.
2. **Enabled Microsoft Sentinel** on `law-soc-lab`.
3. **Connected the identity log source.** The Sentinel Content hub now redirects to the Microsoft
   Defender portal, so I wired logs the supported way — **Entra ID → Diagnostic settings**
   (`entra-to-law`) streaming **AuditLogs** and **SignInLogs** to `law-soc-lab`. Per Microsoft's
   docs, configuring the diagnostic setting **auto-enables the Sentinel Entra connector**.
4. **Created a test subject** — `soc-test01`, a standard (no-privilege) member account.
5. **Simulated the attack** — assigned `soc-test01` to sensitive read-only directory roles
   (**Global Reader**, then **Security Reader**), each generating an Entra *"Add member to role"*
   audit event for the detection to catch.
6. **Validated the pipeline end to end** — confirmed the audit event lands in `law-soc-lab` via KQL,
   promoted the query to a scheduled **analytics rule** (built in the unified **Microsoft Defender
   portal**, where Sentinel analytics-rule management now lives), and watched it raise a
   **High-severity incident** with `soc-test01` auto-mapped as the impacted entity — then tuned out
   duplicate alerts caused by the 24-hour lookback.

## Detection — Privileged role assignment (`T1098`)

Fires when an account is added to a sensitive directory role:

```kql
let sensitiveRoles = dynamic([
    "Global Administrator", "Privileged Role Administrator", "Security Administrator",
    "User Administrator", "Global Reader", "Security Reader"
]);
AuditLogs
| where OperationName == "Add member to role"
| mv-expand prop = TargetResources[0].modifiedProperties
| where tostring(prop.displayName) == "Role.DisplayName"
| extend RoleAdded = trim('"', tostring(prop.newValue))
| where RoleAdded in (sensitiveRoles)
| extend Actor = tostring(InitiatedBy.user.userPrincipalName),
         TargetUser = tostring(TargetResources[0].userPrincipalName)
| project TimeGenerated, RoleAdded, TargetUser, Actor, Result
| order by TimeGenerated desc
```

### Planned expansion (next iterations)
- **Anomalous sign-in / impossible travel** (`T1078`) — from `SignInLogs`; requires **Entra ID P1**.
- **Brute force** (`T1110`) and **suspicious PowerShell** (`T1059.001`) — require a Windows log
  source (a small Azure VM with the Azure Monitor Agent), documented as the next build-out.

## Skills demonstrated

- Microsoft Sentinel deployment and workspace setup
- Identity log onboarding via **Entra ID diagnostic settings** (audit + sign-in)
- **Detection engineering in KQL** (parsing `AuditLogs`, `mv-expand`, role allow-list)
- Mapping detections to **MITRE ATT&CK** (T1098 privilege escalation)
- Controlled attack simulation and end-to-end pipeline validation
- Identity & access fundamentals (directory roles, least privilege)

## Results / Evidence

> Screenshots captured to `C:\Users\jcade\Downloads\soc-lab-assets`, then copied into `assets/`.

- [x] `assets/01-sentinel-overview.png` — Sentinel enabled on `law-soc-lab`
- [x] `assets/02-entra-diagnostic-settings.png` — `entra-to-law` connector → `law-soc-lab`
- [x] `assets/03-role-assignment.png` — `soc-test01` granted Global Reader (the trigger)
- [x] `assets/04-kql-auditlog.png` — detection KQL returning the captured "Add member to role" event (Security Reader)
- [x] `assets/05-analytics-rule.png` — the scheduled rule (Enabled · High · T1098) in the Defender portal
- [x] `assets/06-incident.png` — the resulting **High** incident with `soc-test01` mapped as the entity
- [x] `assets/07-investigation.png` — the incident's alert queue (Privilege Escalation) for triage

## Lessons learned

Real findings from building this lab:

- **Sentinel has moved into the unified Defender portal.** Both the Content hub and **analytics-rule
  creation** now redirect out of the Azure portal — you connect the workspace to Defender
  (System → Settings → Microsoft Sentinel) and build rules under **Microsoft Sentinel → Configuration
  → Analytics**. Onboarding Entra logs is still done via **diagnostic settings**, which auto-enables
  the connector. Right after connecting a workspace, the Analytics page briefly redirects to workspace
  settings until onboarding propagates (~tens of minutes) — expected, not an error.
- **Lookback vs. run frequency drives alert noise.** Running every 5 minutes with a 24-hour lookback
  meant each run re-detected the same event and raised a duplicate alert (one incident, three alerts).
  Fixed with **alert suppression** (stop querying for 24h after an alert) so one event maps to one
  alert — a core detection-tuning trade-off.
- **First-time log ingestion is slow.** Microsoft documents that after creating a diagnostic
  setting, data starts flowing **within ~90 minutes** and can officially take **up to three days**
  on first setup. Plan validation around that latency rather than expecting instant results.
- **Sign-in logs need Entra ID P1**; audit logs flow on any (including free) license. The license
  gate is enforced silently, so confirm which log types actually arrive.
- **Never store credentials in a portfolio repo.** The lab's test password is kept out of git and
  the account is deleted in cleanup — basic hygiene that a SOC review should enforce.

## Mapped to

- **CompTIA Security+ (SY0-701) — Domain 4, Security Operations:** monitoring & alerting, SIEM /
  log data analysis, incident response.
- **MITRE ATT&CK:** T1098 (Account Manipulation / privilege escalation). Planned: T1078, T1110, T1059.001.

## Reproduce this lab

Step-by-step instructions: **[BUILD-GUIDE.md](BUILD-GUIDE.md)**
