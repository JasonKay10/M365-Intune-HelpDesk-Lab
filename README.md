![Microsoft 365 & Intune Lab banner](banner.png)

# Microsoft 365 & Intune Help Desk Lab

A hands-on lab simulating real Tier 1 help desk work in a Microsoft 365 environment: onboarding users, applying least privilege, managing groups, enforcing MFA, joining a Windows device to Microsoft Entra ID, enrolling it in Intune, and enforcing a compliance policy. It also covers troubleshooting real errors along the way.

**Tools:** Microsoft 365 Business Premium · Microsoft 365 Admin Center · Microsoft Entra ID · Microsoft Intune · Windows 11 Enterprise (VirtualBox VM) · Microsoft Authenticator · `dsregcmd`

---

## Lab Environment

| Component | Details |
|---|---|
| Tenant | Microsoft 365 Business Premium trial (`HomeLabs978.onmicrosoft.com`) |
| Admin account | Global Administrator (lab owner) |
| Test users | Megan Lopez (Sales), Alex Wilber (Sales), Priya Shah (IT – Help Desk) |
| Test device | Windows 11 Enterprise Evaluation VM in Oracle VirtualBox 7.2 |
| Identity | Cloud-only Microsoft Entra ID (no on-premises Active Directory) |

---

## Ticket 1: New Hire Onboarding

**Request:** HR has three new hires who need Microsoft 365 accounts with licenses.

**Actions:**
- Created three user accounts with display names, usernames, job titles, and departments.
- Assigned a Microsoft 365 Business Premium license to each user.
- Set the **Department** attribute so users can be grouped and targeted by policy later.

| Basics | License | Profile |
|---|---|---|
| ![Megan basics](01-User-Onboarding/01-megan-basics.png) | ![Megan license](01-User-Onboarding/02-megan-license.png) | ![Megan profile](01-User-Onboarding/03-megan-profile.png) |

![User added](01-User-Onboarding/04-megan-added.png)

> **Note:** For lab convenience, users were not forced to change their password at first sign-in. In production, I would enforce a password change at first sign-in so only the employee knows their password.

---

## Ticket 2: Least Privilege for the Help Desk Technician

**Request:** The new Help Desk Technician (Priya Shah) needs to reset passwords for users but should not have full admin access.

**Actions:**
- Assigned the **Helpdesk Administrator** role instead of Global Administrator.
- Helpdesk Administrator can reset passwords for non-admin users but cannot create/delete users or change other admins.

![Helpdesk Administrator role](01-User-Onboarding/08-priya-helpdesk-role.png)
![Priya review](01-User-Onboarding/10-priya-review.png)

### Troubleshooting: User showed as "Unlicensed"
Right after creation, Priya appeared as **Unlicensed** in Active users. Before making changes, I refreshed the page and confirmed the license had applied after a short propagation delay. No further action was needed.

![All users licensed](01-User-Onboarding/09-active-users.png)

---

## Ticket 3: Security Group for the Sales Team

**Request:** IT needs a group for the Sales team to manage access and target device policies.

**Actions:**
- Created a **Sales Team** security group (role assignment disabled, since Sales users do not need admin roles).
- Added Megan Lopez and Alex Wilber as members.

| Review | Members |
|---|---|
| ![Group review](02-Groups/01-sales-team-review.png) | ![Group members](02-Groups/02-sales-team-members.png) |

---

## Ticket 4: Join a Company Laptop to Entra ID and Enroll in Intune

**Request:** Megan's new company laptop needs to be joined to the organization and managed by Intune.

**Actions:**
1. Enabled **automatic MDM enrollment** in Intune (MDM user scope: All).
2. On the Windows 11 VM, used **Settings → Accounts → Access work or school → Join this device to Microsoft Entra ID** (a full corporate join, not just registering a personal device).
3. Signed in as Megan and confirmed the organization.
4. Verified the join and the Intune enrollment.

![Automatic enrollment](03-Intune-Enrollment/01-auto-enrollment-all.png)
![Join confirmation](03-Intune-Enrollment/06-join-confirm.png)
![Join success](03-Intune-Enrollment/07-join-success.png)

### Troubleshooting: "Intune service principal not found"
**Cause:** I was signed into the wrong tenant (a separate Azure lab directory) that had no Intune license.
**Fix:** Signed into the Intune admin center in a private window with the correct tenant admin account and verified the tenant name before making changes.

### Troubleshooting: MFA "BadRequest" during device join
**Cause:** MFA registration failed inside the Windows join window.

![MFA error](03-Intune-Enrollment/04-mfa-error-badrequest.png)

### Troubleshooting: Device only *registered*, not *joined*
After the first attempt, the account appeared under Access work or school, but `dsregcmd /status` showed **AzureAdJoined: NO**. The device had only been registered, so Intune could not fully manage it.

**Fix:** Disconnected the registered account and repeated the join using the **Join this device to Microsoft Entra ID** option.

| Before | After |
|---|---|
| ![Not joined](03-Intune-Enrollment/05-dsregcmd-not-joined.png) | ![Joined](03-Intune-Enrollment/08-dsregcmd-joined.png) |

### Result
The device appeared as **Managed** in the Microsoft 365 admin center and as **Compliant / Corporate / Intune** in the Intune admin center, with Megan as the primary user.

![Managed in admin center](03-Intune-Enrollment/09-device-managed-admin-center.png)
![Intune device page](03-Intune-Enrollment/10-device-intune-admin-center.png)

---

## Ticket 5: Device Compliance Policy

**Request:** Security requires every Sales Windows device to have a firewall, antivirus, and antispyware enabled, or be flagged.

**Actions:**
- Created a **Windows 10/11 compliance policy** named *Windows Baseline Compliance*.
- Required **Firewall**, **Antivirus**, and **Antispyware** under System Security.
- Kept the default action: mark the device noncompliant immediately.
- Assigned the policy to the **Sales Team** group.
- Forced a sync from Intune and from the device, then verified the result.

| Basics | Settings | Assignment |
|---|---|---|
| ![Policy basics](04-Compliance-Policy/01-policy-basics.png) | ![Policy settings](04-Compliance-Policy/02-compliance-settings.png) | ![Assignment](04-Compliance-Policy/04-policy-assignment.png) |

![Device sync](04-Compliance-Policy/07-device-sync.png)
![Sync successful on device](04-Compliance-Policy/08-vm-sync-successful.png)

### Result: 1 device compliant
![Policy compliant](04-Compliance-Policy/09-policy-compliant.png)

> **Design note:** I left Device Health settings like BitLocker and Secure Boot out of this policy because they often fail on virtual machines. In production, I would add BitLocker and a password requirement and use a grace period with user notifications before marking a device noncompliant.

---

## Ticket 6: User Has No MFA Registered

**Request:** Verify that MFA is enforced and that Megan has a working MFA method.

**Actions:**
- Confirmed **Security defaults** are enabled, which enforces MFA tenant-wide.
- Checked Megan's **Authentication methods** in Entra ID and found **no methods registered** (her earlier registration failed).
- Had Megan register **Microsoft Authenticator** at `mysignins.microsoft.com/security-info` from her device.
- Re-checked Entra ID and confirmed Authenticator was now her default sign-in method.

![Security defaults](05-MFA/01-security-defaults-enabled.png)

| Before | After |
|---|---|
| ![No methods](05-MFA/02-megan-no-methods.png) | ![Methods after](05-MFA/04-megan-methods-after.png) |

![Authenticator added](05-MFA/03-megan-registered.png)

---

## Skills Demonstrated

- Microsoft 365 user lifecycle: account creation, licensing, profile attributes
- Role-based access control (RBAC) and least privilege
- Microsoft Entra ID security groups
- Microsoft Entra ID device join vs. device registration
- Microsoft Intune automatic enrollment, device management, and compliance policies
- MFA enforcement with Security defaults and Microsoft Authenticator
- Troubleshooting with `dsregcmd /status`, sign-in context, and propagation delays

## Key Takeaways

- **Verify before changing:** refreshing first resolved the "unlicensed" issue without unnecessary changes.
- **Check the tenant:** many admin errors come from being signed into the wrong directory.
- **Joined ≠ registered:** `dsregcmd /status` shows the true join state of a device.
- **Least privilege:** Tier 1 support only needs Helpdesk Administrator, not Global Administrator.

## Next Labs

- Conditional Access (require MFA, require compliant devices)
- Windows Autopilot
- Intune app deployment and configuration profiles
- Exchange Online help desk tickets
- Hybrid identity with Active Directory and Entra Connect
