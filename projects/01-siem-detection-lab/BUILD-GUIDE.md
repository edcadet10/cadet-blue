# BUILD GUIDE · SIEM & Detection Engineering Lab (Microsoft Sentinel)

Reproduce the lab end-to-end. Navigation is by **resource name in the Azure portal search bar**;
exact blades and buttons are named. Everything here is benign and runs only in your own lab.

> **Cost note:** Sentinel/Log Analytics bill on ingested GB. A small lab is a few dollars; turn
> off or cap ingestion when done. There is a free trial tier for Sentinel on new workspaces.

---

## Phase 0 — Prerequisites

- [ ] An Azure subscription you can create resources in.
- [ ] Your Entra tenant (`admintradeproof.onmicrosoft.com` lab) with **Global Admin** or
      **Security Admin** for the identity-log steps.
- [ ] **DC01** (Windows Server 2022) and at least one domain-joined Windows client, both able to
      reach Azure (for the Azure Monitor Agent).
- [ ] **Licensing caveat (be honest in the write-up):** streaming Entra **SigninLogs** to Log
      Analytics requires **Entra ID P1**. If your lab tenant is free, do the Windows-host
      detections (rules 1 & 4) and the audit-log detection (rule 3, available without P1), and
      note the sign-in-log rule (rule 2) as "requires P1" rather than skipping it silently.

## Phase 1 — Create the Log Analytics workspace

1. Portal search bar → **Log Analytics workspaces** → **Create**.
2. Resource group: **rg-soc-lab** (create new). Name: **law-soc-lab**. Region: **East US 2**
   (match your other labs). → **Review + create** → **Create**.

## Phase 2 — Enable Microsoft Sentinel

1. Search bar → **Microsoft Sentinel** → **Create** (or **Add**).
2. Select **law-soc-lab** → **Add Microsoft Sentinel**.
3. You should land on the Sentinel **Overview** for that workspace. 📸 *Screenshot 01 candidate.*

## Phase 3 — Connect Windows Security Events (via AMA)

1. In Sentinel → left nav **Content management → Content hub** → search **Windows Security Events**
   → **Install** the solution.
2. Left nav **Configuration → Data connectors** → open **Windows Security Events via AMA** →
   **Open connector page**.
3. **Create data collection rule (DCR):**
   - Name: **dcr-windows-security**.
   - Resources: **Add** → select **DC01** and your client (Azure Arc-enable them first if they're
     on-prem — the connector page links the Arc onboarding; for Azure VMs they appear directly).
   - Collect: choose **All Security Events** (lab) or **Common** to save cost.
   - **Create**. The AMA deploys automatically to the selected machines.
4. Verify ingestion: Sentinel → **Logs** → run `SecurityEvent | take 10`. Events should appear
   within ~10–15 min.

## Phase 4 — Connect Entra ID logs (sign-in + audit)

1. Search bar → **Microsoft Entra ID** → left nav **Monitoring → Diagnostic settings** →
   **Add diagnostic setting**.
2. Name: **entra-to-law**. Check **SignInLogs** (needs P1) and **AuditLogs** (no P1 needed).
3. Destination: **Send to Log Analytics workspace** → **law-soc-lab** → **Save**.
4. Verify: Sentinel → **Logs** → `SigninLogs | take 10` and `AuditLogs | take 10`.

## Phase 5 — Turn on command-line auditing (for rule 4)

On **DC01** (or a GPO linked to the clients' OU), enable process-creation logging with command line:

1. **Group Policy Management** → edit the policy for your client OU.
2. *Computer Configuration → Policies → Windows Settings → Security Settings → Advanced Audit
   Policy → Detailed Tracking* → **Audit Process Creation** = **Success**.
3. *Computer Configuration → Policies → Administrative Templates → System → Audit Process Creation*
   → **Include command line in process creation events** = **Enabled**.
4. On the client: `gpupdate /force`.

## Phase 6 — Create the four analytics rules

For each rule: Sentinel → **Configuration → Analytics → Create → Scheduled query rule**.
General tab: set **Name**, **Severity**, **MITRE technique** (Set rule logic tab → Techniques).
Set rule logic tab: paste the KQL, set **Run query every** 5–15 min over the last 1 hour, set
**Entity mapping** (Account = TargetAccount/UserPrincipalName; Host = Computer; IP = IPAddress).

1. **Brute force then success** — Severity *Medium*, technique **T1110**. KQL = detection #1 in
   [`README.md`](README.md).
2. **Impossible travel** — Severity *Medium*, technique **T1078**. KQL = detection #2.
   *(Skip if no P1 — note it in the write-up.)*
3. **New privileged-role member** — Severity *High*, technique **T1098**. KQL = detection #3.
4. **Suspicious PowerShell** — Severity *High*, technique **T1059.001**. KQL = detection #4.

📸 *Screenshot 02 candidate: the four enabled rules in the Analytics list.*

## Phase 7 — Simulate the attacks (lab-safe)

> All benign; run only against your own lab accounts/hosts.

- **Brute force (rule 1):** on the client, attempt logon with a wrong password 10+ times in 10 min
  for a test account (e.g., repeated `runas /user:LAB\\testuser cmd` with a bad password, or failed
  RDP attempts), then one **correct** logon.
- **New privileged-role member (rule 3):** Entra ID → **Roles and administrators** → e.g.
  **Security Reader** → **Add assignment** → add a throwaway test user → then **Remove**. (Use a
  low-impact role for the lab; the rule's role list can include it for testing.)
- **Suspicious PowerShell (rule 4):** on the client run a harmless encoded command, e.g.
  `powershell -enc <base64-of:Get-Date>` so `CommandLine` contains `-enc`.
- **Impossible travel (rule 2, if P1):** sign in to `https://portal.azure.com` as a test user from
  your normal IP, then within an hour from a VPN/cloud VM in another country.

## Phase 8 — Triage the incidents

1. Sentinel → **Threat management → Incidents**. Open each generated incident.
2. Review **Entities**, open **Investigate** for the graph, build the **timeline**.
3. Write a 3–4 sentence analyst summary per incident: what fired, the evidence, severity, and
   recommended action. (These become the "Lessons learned" + incident notes in the write-up.)

📸 *Screenshots 03 & 04 candidates: a brute-force incident with entities; the investigation graph.*

## Phase 9 — Evidence checklist (fills the README placeholders)

- [ ] `assets/01-sentinel-overview.png` — workspace + connected data connectors
- [ ] `assets/02-analytics-rules.png` — the four enabled analytics rules
- [ ] `assets/03-bruteforce-incident.png` — brute-force incident with mapped entities
- [ ] `assets/04-investigation-graph.png` — investigation graph for a triaged incident
- [ ] `assets/05-kql-hunt.png` — an ad-hoc KQL hunt (e.g., `SecurityEvent | where EventID==4625`)

## Phase 10 — Clean up (control cost)

- Remove the diagnostic setting, or set a **daily cap** on **law-soc-lab** (workspace → *Usage and
  estimated costs → Daily cap*).
- Delete **rg-soc-lab** when finished to stop all charges.

---

When the screenshots are in `assets/` and the lessons are written, flip the status badge in
[`README.md`](README.md) from **In Progress** to **Documented**.
