# BUILD GUIDE · SIEM & Detection Engineering Lab (Microsoft Sentinel)

Reproduce the lab end-to-end. Navigation is by **resource name in the Azure portal search bar**;
exact blades and buttons are named. This guide matches the actual build (identity-first detection
on Microsoft Entra ID logs). Everything is benign and runs only in your own tenant.

> **Cost note:** Sentinel/Log Analytics bill on ingested GB. This identity-only lab is a few cents
> to a couple dollars. Delete the resource group when done (Phase 10).

---

## Phase 0 — Prerequisites

- [ ] An Azure subscription you can create resources in.
- [ ] An account with **Global Administrator** / **Security Administrator** in the tenant (used for
      diagnostic settings + role assignment). Do **not** use a low-privilege user.
- [ ] **Licensing note:** streaming **SignInLogs** to Log Analytics needs **Entra ID P1**.
      **AuditLogs** (which this lab's detection uses) flow on **any** license, including free.

## Phase 1 — Create the workspace (and resource group)

1. Search bar → **Log Analytics workspaces** → **+ Create**.
2. **Basics:** Subscription = your sub; **Resource group → Create new** = `rg-soc-lab`;
   **Name** = `law-soc-lab`; **Region** = `East US 2`.
3. **Review + create** → **Create**. Wait for "deployment complete."

## Phase 2 — Enable Microsoft Sentinel

1. Search bar → **Microsoft Sentinel** → **+ Create**.
2. Select **law-soc-lab** → **Add**. You land on **Microsoft Sentinel | Overview**.
   📸 `01-sentinel-overview.png`.

> **Note:** Sentinel's **Content hub** now redirects to the **Microsoft Defender portal** (Sentinel
> is consolidating there by 2027-03-31). You don't need it — we onboard logs via diagnostic settings.

## Phase 3 — Stream Entra ID logs (diagnostic settings)

1. Search bar → **Microsoft Entra ID** → **Monitoring & health → Diagnostic settings**.
2. **+ Add diagnostic setting.** Name = `entra-to-law`.
3. **Logs:** tick **AuditLogs** and **SignInLogs** *(if you lack P1, SignInLogs save but won't
   actually flow — AuditLogs is enough for this lab)*.
4. **Destination:** **Send to Log Analytics workspace** → **law-soc-lab** → **Save**.
   📸 `02-entra-diagnostic-settings.png`.

> **Latency reality (important):** per Microsoft, after creating a diagnostic setting, data starts
> flowing **within ~90 minutes** and can officially take **up to 3 days** on first setup (often
> ~15 min, but not guaranteed). Don't expect instant results — this is normal.

## Phase 4 — Create the test subject

1. **Microsoft Entra ID → Users → All users → + New user → Create new user.**
2. **UPN** = `soc-test01`; **Display name** = `SOC Test User 01`; set a password (keep it out of any
   repo); leave it a standard member (no roles). **Review + create → Create.**

## Phase 5 — Simulate the attack (privilege escalation)

1. **Microsoft Entra ID → Roles and administrators** → open **Global Reader** (read-only but
   sensitive — zero real risk).
2. **+ Add assignments** → select **soc-test01** → **Add**.
   This writes an Entra audit event: *"Add member to role"*. 📸 `03-role-assignment.png`.

## Phase 6 — Confirm the event arrived (KQL)

1. **Microsoft Sentinel → law-soc-lab → General → Logs.**
2. **Set the editor to `KQL mode`** (top-right toggle — *not* Simple mode). Time range = **Last 24 hours**.
3. Run:
   ```kql
   AuditLogs
   | where OperationName == "Add member to role"
   | order by TimeGenerated desc
   ```
4. When a row appears (after the ingestion latency above), 📸 `04-kql-auditlog.png`.
   *(Tip: `AuditLogs | take 50` tells you the moment any audit data starts landing.)*

## Phase 7 — Create the scheduled analytics rule

1. **Sentinel → Configuration → Analytics → + Create → Scheduled query rule.**
2. **General:** Name = `Privileged role assignment`; Severity = **High**;
   **MITRE ATT&CK** → tick **Privilege Escalation / T1098**.
3. **Set rule logic:** paste the detection KQL (below); **Run query every** 5 min, **lookback** last
   24 hours; **Entity mapping:** Account → `TargetUser`, optionally Account → `Actor`.
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
4. **Incident settings:** leave **Create incidents** on. **Review + create → Save.**
   📸 `05-analytics-rule.png`.

## Phase 8 — Triage the incident

1. **Sentinel → Threat management → Incidents.** Open the incident the rule raises (it will fire on
   the next run that sees the ingested event). 📸 `06-incident.png`.
2. Review **Entities** (soc-test01, Global Reader), open **Investigate** for the graph,
   build a timeline. 📸 `07-investigation.png`.
3. Write a short analyst summary: what fired, the evidence, severity, recommended action.

## Phase 9 — Evidence checklist (fills the README)

- [ ] `01-sentinel-overview.png` · [ ] `02-entra-diagnostic-settings.png` · [ ] `03-role-assignment.png`
- [ ] `04-kql-auditlog.png` · [ ] `05-analytics-rule.png` · [ ] `06-incident.png` · [ ] `07-investigation.png`

Copy them from `C:\Users\jcade\Downloads\soc-lab-assets` into this project's `assets/` folder,
then flip the README status from **In Progress** to **Documented**.

## Phase 10 — Cleanup

1. **Roles and administrators → Global Reader →** remove **soc-test01**.
2. **Users →** delete **soc-test01** (kills the test credential).
3. Delete the resource group **`rg-soc-lab`** to stop all charges.

## Optional expansion (stronger flagship)

- **Add a Windows log source** — a small Azure Windows VM + Azure Monitor Agent → enables
  **brute-force (T1110)** and **suspicious PowerShell (T1059.001)** detections.
- **Enable Entra ID P1** → **SignInLogs** flow → adds **impossible-travel (T1078)** detection.
