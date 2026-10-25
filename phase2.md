# Microsoft 365 Hardening Plan – Defend the Tenant Challenge

**Customer:** small business, 100 users, two administrators, Microsoft 365 E7.
**Ask:** recommend and rank the five security controls that matter most, and show how to roll them out with very little admin time.

> This plan describes a fictional customer. It contains no tenant data, identifiers or credentials.

---

## How I prioritized

Three things about this customer shaped the ranking:

- **Two admins.** If one admin account is compromised, the attacker controls the whole tenant, and there's nobody to pick up extra manual work.
- **No security operations team.** Controls should fix problems automatically, let users fix them, or at least alert the two admins.
- **E7 licensing.** E7 includes the E5 security stack (Defender for Office 365, Defender for Endpoint, Intune, Purview) plus Entra Suite, Copilot and Agent 365. Licensing doesn't block anything below. The challenge is time, not tooling.

The identity controls come from the Secure Score hardening baseline in the Microsoft Learn module. I added email and devices because those are the two main ways attacks reach a small company that the identity baseline doesn't cover.

The ranking reflects risk reduction. The rollout order further down follows dependencies and lockout risk.

## Before changing anything

Conditional Access can lock out every admin if it's configured badly, so these come first:

- **Two cloud-only break-glass accounts** with permanent Global Administrator. Microsoft now enforces MFA for admin portals on every account, so each gets its own FIDO2 key that doesn't depend on the normal admin sign-in. No mailbox, no license, credentials stored under a documented emergency procedure. Every sign-in raises an alert, and the accounts are tested on a fixed schedule. They are the only accounts excluded from the baseline policies.
- **A small pilot group:** a normal user, an admin, someone working mostly on mobile, and users of unusual clients such as scanners or meeting room devices.
- **An export of the current state:** Conditional Access policies, MFA registration, legacy sign-ins, risky users, app consent grants, role assignments, mail protection policies and device status. This is the rollback reference.
- **Security defaults** turned off through a planned change, once the custom policies cover the same protection.
- **Report-only first.** Conditional Access policies run in report-only for at least a week, then they're enforced one at a time.

---

## Five Controls

### 1. Strong sign-in for every account

**What I'd do** – a small set of Conditional Access policies, each separate so it can be tested and rolled back on its own:
- **Admins:** target the admin directory roles (not a hand-maintained group) and require the *phishing-resistant MFA* authentication strength: FIDO2 keys, passkeys in Microsoft Authenticator or Windows Hello for Business.
- **All users:** require MFA on all resources, with no app exclusions. Guests are included; trusting MFA from partner tenants in the cross-tenant access settings avoids double prompts.
- **Legacy authentication:** block *Exchange ActiveSync* and *Other clients*. Exchange Online has already switched off basic authentication for most protocols, so this mainly closes what's left and keeps it closed. Unused protocols are also disabled at the mailbox level.
- **Risk:** sign-in risk Medium or High requires MFA; user risk High requires risk remediation. Both use sign-in frequency *every time*, and they stay two separate policies.
- **Registration:** registering security info is only allowed under Conditional Access, so someone who only has a password can't add their own MFA method.

**Why it matters**
Stolen and reused passwords, password spray and phishing are still the usual way into small tenants. MFA makes a stolen password far less useful. Admins get the stronger method because push notifications and one-time codes can be relayed by phishing kits, and an admin account opens everything. Legacy protocols can't do MFA, so they would be a way around everything else. The risk policies turn Identity Protection detections (leaked credentials, anonymous IPs, unusual sign-ins) into an automatic MFA prompt or a forced remediation, which matters when nobody watches a console.

**Rolling it out with two admins**
Start with the registration report. In the Phase 1 lab, 9 of 10 users weren't ready yet, which is normal. Run a registration campaign for Microsoft Authenticator and use a Temporary Access Pass for people without a phone. For legacy authentication, filter the sign-in logs by client app over a full month so monthly jobs show up too. In small companies this usually turns up a scanner sending mail, an old mail client or a script. Fix those, then enforce in this order: admins, legacy block, all users once nearly everyone is registered, risk policies last. If users are synced from on-premises, password writeback has to work before the user risk policy is enforced.

**Tradeoffs**
Users notice the extra step and there will be questions in the first week. Old devices and apps stop working until they're replaced or reconfigured. Risk detections produce some false positives, for example while travelling. And remediation lowers the risk state but doesn't undo what an attacker already did, so a confirmed compromise still needs a proper look.

---

### 2. Least-privileged admin roles with Privileged Identity Management

**What I'd do**
- Separate cloud-only admin accounts for both admins, without mailbox or license. Their everyday accounts hold no admin roles.
- No permanent Global Administrator except the two break-glass accounts. Day-to-day work uses task roles such as Exchange Administrator, Teams Administrator, Helpdesk Administrator or Security Reader.
- All admin roles become *eligible* in PIM: activated for a few hours, with phishing-resistant MFA and a short justification, and both admins are notified whenever a role is activated.

**Why it matters**
With two admins, a single phished session or a single mistake can affect the whole tenant. Standing Global Admin rights make that worse. Least privilege limits three things: what an account can do, where, and for how long.

**Rolling it out with two admins**
Export the current role assignments, map each regular task to the smallest role that covers it, and convert standing assignments to eligible ones. Only remove an old assignment once the new role has been tested. This is roughly half a day of work.

**Tradeoffs**
Activating a role adds a few minutes before admin tasks. I wouldn't require approval, because the approver would always be the other admin and one of them will be on holiday at some point. MFA, justification, notifications and the audit trail cover it.

---

### 3. Email protection with Defender for Office 365

**What I'd do**
- Turn on the **preset security policies**: *Standard* for all users, *Strict* for the admins and the people who handle payments, set as priority accounts. They cover anti-phishing, anti-spam, anti-malware, Safe Links and Safe Attachments, and Microsoft keeps the settings current.
- Publish **SPF, DKIM and DMARC** for the company domains. DMARC starts in monitoring mode and moves to quarantine once the reports are clean.
- Keep **automatic forwarding to external addresses** blocked (the default) and make sure the built-in **Report** button is available to users.

**Why it matters**
Most phishing, malware and business email compromise still arrives by mail, and a convincing message is often how the stolen passwords in control 1 are obtained in the first place. Safe Links checks links when they're clicked, not just when the mail arrives. DMARC makes it much harder to send mail that pretends to come from the company. Blocking external forwarding stops a common trick after a mailbox is compromised.

**Rolling it out with two admins**
The presets take less than an hour. DMARC needs a few DNS changes and a couple of weeks of reports before tightening. After that, the main ongoing task is a short weekly look at the quarantine and user reports.

**Tradeoffs**
Some legitimate mail will land in quarantine, especially with *Strict*, and users need to know where to find it. Safe Attachments can delay some attachments slightly. A wrong DMARC setting can block legitimate senders such as a newsletter service, which is why it starts in monitoring mode.

---

### 4. Managed and protected devices with Intune and Defender for Endpoint

**What I'd do**
- Enroll company Windows devices in **Intune** and onboard them to **Defender for Endpoint** through Intune.
- A **compliance policy** that checks BitLocker, a supported OS version, Defender antivirus and firewall, and the device risk level reported by Defender for Endpoint.
- **Attack surface reduction rules** and **Windows LAPS** for the local admin password.
- Conditional Access **require compliant device** for the admins first, later for all users on desktops. Personal phones get **app protection policies** instead of full enrollment.
- Email notifications to both admins for high-severity incidents, with automated investigation and response switched on.

**Why it matters**
Ransomware and info-stealers start on a device. Defender for Endpoint detects and often contains them automatically, which again matters without a security team. The compliance requirement means a stolen password or session is much harder to use from an unmanaged device, and admin access only happens from a known, healthy machine.

**Rolling it out with two admins**
Start with the admin devices and the pilot group. Run the attack surface reduction rules in audit mode for two weeks, then block. New devices can be set up with Windows Autopilot so the effort doesn't grow with every new hire. Move the remaining devices in two or three groups.

**Tradeoffs**
This is the most effort of the five, because every device has to be touched once. Some older devices may not meet the requirements, and attack surface reduction rules can block legitimate macros or tools until exceptions are added. Personal devices raise privacy questions, which is why phones get app protection rather than full management.

---

### 5. Block user consent to applications

**What I'd do**
Set user consent to *Do not allow user consent* and turn on the admin consent workflow with reviewer notifications. Stop regular users from registering applications. Then review the permissions that were granted before the change.

**Why it matters**
In a consent phishing attack, the user is tricked into approving a malicious app. The app then gets ongoing access to mail or files without the password, and MFA doesn't stop it. The same path applies to the growing number of AI apps and agents that ask for access to mail and files. Turning off user consent closes it. It doesn't remove existing grants, so those need a separate review.

**Rolling it out with two admins**
The setting itself takes minutes. Send users a short note that they now request apps instead of approving them. For the existing grants, look first at broad permissions such as reading all mail or files and at apps from unverified publishers.

**Tradeoffs**
Users wait for an admin before they can use a new app, and the two admins need to review those requests: who publishes the app, what it asks for, and who in the business owns it.

---

## Quick wins right after the top five

These are low effort but reduce less risk than the five above:

- **Self-service password reset for all users,** with two methods. Fewer password tickets for two admins and less risk from "I lost my phone, please reset me" calls. Most users will already have registered during the MFA rollout. Test with a normal user, because admins have their own, stricter reset policy.
- **Teams meeting defaults** in the Global meeting policy: only invited people bypass the lobby, only organizers and co-organizers present, anonymous users can't join. Changes can take up to a day. The tradeoff is that external contacts without a Microsoft account can no longer join anonymously.

---

## Rollout plan

| Week | Admin A | Admin B |
|---|---|---|
| 0 | Break-glass accounts, alerts, FIDO2 keys, export current state | Pilot group, MFA registration campaign, user communication |
| 1 | Conditional Access policies in report-only, role inventory | Defender for Office 365 presets, SPF/DKIM, DMARC in monitoring mode |
| 2 | Enforce admin MFA and the legacy block, set up PIM | Turn off user consent, review existing grants, SSPR pilot |
| 3 | Enforce MFA for all users once registration is close to complete | Intune and Defender for Endpoint on admin and pilot devices, ASR in audit |
| 4 | Enforce risk policies, Teams meeting defaults | Remaining devices in groups, compliant device for admins, validation evidence |

One change per maintenance window. If a policy causes problems, it gets switched off and investigated, not deleted or edited in a hurry.

## How we know it works

A setting in the portal or a higher Secure Score isn't proof, and Secure Score also lags (Entra recommendations at least weekly, Teams at least monthly). Each control is checked where it runs:

| Control | Check |
|---|---|
| Sign-in | What If tool, sign-in logs filtered by policy, a controlled legacy sign-in that fails, risky users and sign-ins reports |
| Admin roles | Role assignment export, PIM activation history, no permanent Global Admin outside break-glass |
| Email | Test phishing link through Safe Links, DMARC reports, quarantine and user reports |
| Devices | Compliance report, Defender for Endpoint onboarding status, sign-in from an unmanaged device blocked for admins |
| App consent | A normal user can't grant consent, the request shows up in the workflow and audit log |

For each control I'd record the owner, the configuration, what we observed, any exception with reason and expiry date, and the next review date. Every quarter we'd go through exclusions, role assignments, app grants, authentication methods, quarantine trends, device compliance and a break-glass test.

## What changes for the customer

| Attack path | Today | After the plan |
|---|---|---|
| Stolen or sprayed password | Password alone is enough | MFA required, legacy route closed |
| Phished admin | Full tenant access | Phishing-resistant sign-in, no standing Global Admin |
| Phishing or malware by mail | Reaches the inbox | Filtered, links checked on click, spoofing harder |
| Ransomware on a laptop | Depends on the user | Detected and contained, device must be healthy for access |
| Consent phishing | User hands mail access to an app | Blocked and reviewed by an admin |
| Lockout from a bad policy | Possible | Break-glass accounts, report-only, one change at a time |



