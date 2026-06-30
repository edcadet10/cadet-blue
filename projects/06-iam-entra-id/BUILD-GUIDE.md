# BUILD GUIDE · Identity & Access Management on Entra ID

How to reproduce the lab in a Microsoft Entra tenant. Everything is scoped to a pilot group and a
break-glass exclusion, and all the Conditional Access policies run in report-only, so nothing here can
lock you out. Navigation is by blade name in **entra.microsoft.com**.

> **License note:** Conditional Access, PIM, and access reviews need Entra ID P1/P2. A free 30-day
> Entra ID P2 trial covers it. Buy or activate it from the Microsoft 365 admin center
> (`admin.microsoft.com → Billing → Purchase services`) while signed into the *target* tenant, so the
> license lands in the right place. Assign P2 to your test and admin accounts (set each user's Usage
> location first, or the assignment fails).

---

## Phase 0: Safety setup (do this first)

1. **Break-glass account.** Make sure a dedicated emergency admin exists (something like
   `breakglass@<tenant>.onmicrosoft.com`) with a long, unique password kept offline. You'll exclude it
   from every Conditional Access policy so a misfire can't lock you out.
2. **Pilot group.** **Identity → Groups → All groups → + New group:** Security type, name
   `CA-Pilot-Users`, membership **Assigned**, add one non-privileged test user.
   📸 `iam-02-pilot-group.png`.

## Phase 1: CA01, require MFA (report-only)

1. **Protection → Conditional Access → Policies → + New policy.** Name `CA01 - Require MFA (Pilot)`.
2. **Users:** Include → **`CA-Pilot-Users`**; **Exclude →** the break-glass account.
3. **Target resources:** *Select what this policy applies to* = **Resources (formerly cloud apps)** →
   Include → **All resources**.
4. **Grant:** Grant access → **Require multifactor authentication** → Select.
5. **Enable policy = Report-only** → **Create**. 📸 `iam-03-ca-require-mfa.png`.

## Phase 2: CA02, block legacy authentication (report-only)

1. **+ New policy.** Name `CA02 - Block legacy authentication (Pilot)`.
2. **Users:** Include **`CA-Pilot-Users`**; Exclude break-glass.
3. **Target resources:** **All resources**.
4. **Conditions → Client apps:** Configure = **Yes**; tick **Exchange ActiveSync clients** and
   **Other clients** only (leave Browser and Mobile apps/desktop clients unticked).
5. **Grant: Block access.** **Enable = Report-only** → **Create**. 📸 `iam-04-ca-block-legacy.png`.

## Phase 3: CA03, geo-fence (report-only)

1. **Protection → Conditional Access → Named locations → + Countries location.** Name
   `Allowed - United States`, determine by IP, select **United States** → Create.
   📸 `iam-05-named-location.png`.
2. **+ New policy.** Name `CA03 - Block access outside allowed location (Pilot)`.
3. **Users:** Include **`CA-Pilot-Users`**; Exclude break-glass. **Target resources:** All resources.
4. **Conditions → Locations:** Configure = Yes; **Include = Any location**; **Exclude = Selected
   locations → `Allowed - United States`**.
5. **Grant: Block access.** **Enable = Report-only** → **Create**. 📸 `iam-06-ca-geo-block.png`.

> After a few days of report-only data, check **Conditional Access → Insights and reporting** to see
> the impact, then switch policies from Report-only to On one at a time. Break-glass stays excluded.

## Phase 4: PIM, just-in-time Global Administrator

1. Search bar → **Privileged Identity Management → Microsoft Entra roles**.
2. **Manage → Settings → Global Administrator → Edit (Activation):** Activation max **2 hours**;
   **On activation require Azure MFA**; tick **Require justification on activation**; leave approval off
   → **Update**. 📸 `iam-07-pim-role-settings.png`.
3. **Manage → Assignments → + Add assignments:** Role **Global Administrator**, member = the test user,
   **Assignment type = Eligible** → Assign. The account now holds no standing admin rights.
   📸 `iam-08-pim-eligible.png`.

> To prove the cycle (optional): sign in as the test user → **PIM → My roles → Global Administrator →
> Activate**, supplying a justification and MFA. The role stays active only for the 2-hour window, and
> the whole thing is logged.

## Phase 5: Access review (recertification)

1. Start from **PIM → Microsoft Entra roles → Access reviews → + New access review**. Starting here is
   what scopes the review to the role.
2. **Review name** `Quarterly review - Global Administrator eligibility`; **Frequency = Quarterly**;
   **Duration 14 days**; **Role = Global Administrator**; **Assignment type = All active and eligible
   assignments**.
3. **Reviewers:** Selected user(s) → an admin (not self-review).
4. **Upon completion:** Auto-apply results = **Enable**; If reviewers don't respond = **No change** →
   **Start**. 📸 `iam-09-access-review.png`.

> Heads-up on a display quirk: the review's Overview pane may show "Scope: Everyone / Role: ---" even
> though it's correctly scoped. The Role = Global Administrator you set on the create form is the truth,
> not that summary.

## Phase 6: Evidence checklist (fills the README)

- [ ] `iam-01-create-user.png` · [ ] `iam-02-pilot-group.png`
- [ ] `iam-03-ca-require-mfa.png` · [ ] `iam-04-ca-block-legacy.png`
- [ ] `iam-05-named-location.png` · [ ] `iam-06-ca-geo-block.png`
- [ ] `iam-07-pim-role-settings.png` · [ ] `iam-08-pim-eligible.png` · [ ] `iam-09-access-review.png`

## Phase 7: Cleanup (if you used a trial)

- Cancel the Entra ID P2 trial before it converts to paid (`admin.microsoft.com → Billing → Your
  products`). The write-up and screenshots stay valid after the license lapses.
- Optionally delete the pilot group, the CA policies, the PIM eligibility, and the access review to
  put the tenant back the way it was.
