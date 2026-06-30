# BUILD GUIDE · Identity & Access Management on Entra ID

Reproduce the lab end to end in a Microsoft Entra tenant. Everything is scoped to a **pilot group**
and a **break-glass** exclusion, and all Conditional Access policies are created in **report-only**,
so nothing can lock you out. Navigation is by blade name in **entra.microsoft.com**.

> **License note:** Conditional Access, PIM, and access reviews require **Entra ID P1/P2**. A free
> 30-day **Entra ID P2** trial is enough. Buy/activate it from the **Microsoft 365 admin center**
> (`admin.microsoft.com → Billing → Purchase services`) while signed into the *target* tenant, so the
> license lands in the right place. Assign P2 to your test + admin accounts (set each user's
> **Usage location** first, or assignment fails).

---

## Phase 0 — Safety setup (do this first)

1. **Break-glass account.** Ensure a dedicated emergency admin exists (e.g.
   `breakglass@<tenant>.onmicrosoft.com`), with a long unique password kept offline. It will be
   **excluded from every Conditional Access policy** so a misfire can't lock you out.
2. **Pilot group.** **Identity → Groups → All groups → + New group:** Security type, name
   `CA-Pilot-Users`, membership **Assigned**, add a **non-privileged test user**. 📸 `iam-02-pilot-group.png`.

## Phase 1 — CA01: Require MFA (report-only)

1. **Protection → Conditional Access → Policies → + New policy.** Name `CA01 - Require MFA (Pilot)`.
2. **Users:** Include → **`CA-Pilot-Users`**; **Exclude →** the **break-glass** account.
3. **Target resources:** *Select what this policy applies to* = **Resources (formerly cloud apps)** →
   Include → **All resources (formerly 'all cloud apps')**.
4. **Grant:** Grant access → **Require multifactor authentication** → Select.
5. **Enable policy = Report-only** → **Create**. 📸 `iam-03-ca-require-mfa.png`.

## Phase 2 — CA02: Block legacy authentication (report-only)

1. **+ New policy.** Name `CA02 - Block legacy authentication (Pilot)`.
2. **Users:** Include **`CA-Pilot-Users`**; Exclude **break-glass**.
3. **Target resources:** **All resources**.
4. **Conditions → Client apps:** Configure = **Yes**; check **Exchange ActiveSync clients** and
   **Other clients** only (uncheck Browser and Mobile apps/desktop clients).
5. **Grant: Block access.** **Enable = Report-only** → **Create**. 📸 `iam-04-ca-block-legacy.png`.

## Phase 3 — CA03: Geo-fence (report-only)

1. **Protection → Conditional Access → Named locations → + Countries location.** Name
   `Allowed - United States`, determine by IP, select **United States** → Create. 📸 `iam-05-named-location.png`.
2. **+ New policy.** Name `CA03 - Block access outside allowed location (Pilot)`.
3. **Users:** Include **`CA-Pilot-Users`**; Exclude **break-glass**. **Target resources:** All resources.
4. **Conditions → Locations:** Configure = Yes; **Include = Any location**; **Exclude = Selected
   locations → `Allowed - United States`**.
5. **Grant: Block access.** **Enable = Report-only** → **Create**. 📸 `iam-06-ca-geo-block.png`.

> After a few days of report-only data, review **Conditional Access → Insights and reporting** to see
> impact, then flip policies from Report-only to **On** one at a time (break-glass stays excluded).

## Phase 4 — PIM: just-in-time Global Administrator

1. Search bar → **Privileged Identity Management → Microsoft Entra roles**.
2. **Manage → Settings → Global Administrator → Edit (Activation):** Activation max **2 hours**;
   **On activation require Azure MFA**; **Require justification on activation** = checked; approval off
   → **Update**. 📸 `iam-07-pim-role-settings.png`.
3. **Manage → Assignments → + Add assignments:** Role **Global Administrator**, member = **test user**,
   **Assignment type = Eligible** → Assign. The account now holds *no* standing admin rights.
   📸 `iam-08-pim-eligible.png`.

> **(Optional) Prove the cycle:** sign in as the test user → **PIM → My roles → Global Administrator →
> Activate** (supply justification + MFA). The role is active only for the 2-hour window, fully logged.

## Phase 5 — Access review (recertification)

1. In **PIM → Microsoft Entra roles → Access reviews → + New access review** (starting here scopes the
   review to the role).
2. **Review name** `Quarterly review - Global Administrator eligibility`; **Frequency = Quarterly**;
   **Duration 14 days**; **Role = Global Administrator**; **Assignment type = All active and eligible
   assignments**.
3. **Reviewers:** Selected user(s) → an **admin** (not self-review).
4. **Upon completion:** Auto-apply results = **Enable**; If reviewers don't respond = **No change** →
   **Start**. 📸 `iam-09-access-review.png`.

> **Display quirk:** the review's **Overview** pane may show *Scope: Everyone / Role: ---* even though
> it's correctly scoped — the **Role = Global Administrator** you set on the create form is the truth.

## Phase 6 — Evidence checklist (fills the README)

- [ ] `iam-01-create-user.png` · [ ] `iam-02-pilot-group.png`
- [ ] `iam-03-ca-require-mfa.png` · [ ] `iam-04-ca-block-legacy.png`
- [ ] `iam-05-named-location.png` · [ ] `iam-06-ca-geo-block.png`
- [ ] `iam-07-pim-role-settings.png` · [ ] `iam-08-pim-eligible.png` · [ ] `iam-09-access-review.png`

## Phase 7 — Cleanup (if using a trial)

- **Cancel the Entra ID P2 trial** before it converts to paid (`admin.microsoft.com → Billing → Your
  products`). The write-up and screenshots persist after the license lapses.
- Optionally delete the pilot group, CA policies, PIM eligibility, and access review to return the
  tenant to its prior state.
