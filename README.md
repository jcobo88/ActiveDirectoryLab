# Microsoft 365 / Entra ID / Intune Administration Lab

## Overview

I built this lab to get hands-on experience administering a Microsoft 365 environment instead of only studying the services individually.

The lab started with a new Microsoft 365 Business Premium tenant for a fictional company, **Cobo Technologies**. From there I created users and groups in Microsoft Entra ID, enrolled a Windows 11 VM into Intune, deployed applications and policies, tested Conditional Access, managed BitLocker and Windows LAPS, and worked through several troubleshooting scenarios.

Most changes were first assigned to a small pilot group so I could verify the result before treating the configuration as complete.

### Main areas covered

- Microsoft 365 administration
- Microsoft Entra ID
- Microsoft Intune
- User and group management
- Microsoft 365 licensing
- Windows 11 Entra join and Intune enrollment
- Device configuration
- Compliance policies
- Application deployment
- Conditional Access and MFA
- Windows Update
- BitLocker
- Windows LAPS
- Sign-in troubleshooting
- Remote device actions

---

## Lab Environment

| Component | Configuration |
|---|---|
| Organization | Cobo Technologies |
| License | Microsoft 365 Business Premium |
| Identity | Microsoft Entra ID |
| Endpoint Management | Microsoft Intune |
| Test Device | WIN11-INTUNE01 |
| Operating System | Windows 11 Pro |
| Virtualization | Oracle VirtualBox |
| Primary Test User | Alex Rivera |
| Pilot Device Group | DG-Windows-Pilot |
| Conditional Access Pilot Group | SG-CA-MFA-Pilot |

---

## Architecture

```text
Microsoft 365 Business Premium
            |
            v
     Microsoft Entra ID
        |         |
        |         +---- Users / Groups / MFA
        |
        +---- Conditional Access
                    |
                    v
             Microsoft Intune
              |     |     |
              |     |     +---- Applications
              |     +---------- Compliance
              +---------------- Configuration
                    |
                    v
             WIN11-INTUNE01
             Windows 11 Pro
```

---

# 1. Tenant, Users, Groups, and Licensing

I created a Microsoft 365 Business Premium tenant for Cobo Technologies and used it as the base for the rest of the lab.

![Microsoft 365 tenant created](screenshots/01-microsoft-365-tenant-created.png)

My first test user was **Alex Rivera**, an IT Support Technician.

![First Entra user created](screenshots/02-entra-id-first-user-created.png)

I created an IT security group and added Alex to it.

```text
SG-IT-Users
```

![IT security group](screenshots/03-entra-id-it-security-group.png)

I also wanted to practice a faster provisioning method, so I used Microsoft's bulk-user CSV process to create several additional users.

![Bulk Entra users created](screenshots/04-entra-id-bulk-users-created.png)

I separated the users into basic departmental groups.

```text
SG-IT-Users
SG-HR-Users
SG-Sales-Users
```

![Department security groups](screenshots/05-entra-id-department-security-groups.png)

I assigned Microsoft 365 Business Premium only where it was needed for the lab instead of licensing every test account.

![Microsoft 365 license assigned](screenshots/06-microsoft-365-license-assigned.png)

One thing this cleared up for me was that creating an Entra account, assigning group membership, and assigning a Microsoft 365 license are three separate actions. A user can exist in Entra without having access to licensed Microsoft 365 services.

---

# 2. Intune Enrollment and Windows 11 Entra Join

For automatic Intune enrollment, I scoped MDM enrollment to:

```text
SG-IT-Users
```

![Intune MDM enrollment scope](screenshots/07-intune-mdm-enrollment-scope.png)

I then created a Windows 11 Pro VM named:

```text
WIN11-INTUNE01
```

Alex signed into the VM with the work account during setup.

On the VM, I used:

```powershell
dsregcmd /status
```

to confirm the join state.

```text
AzureAdJoined : YES
EnterpriseJoined : NO
DomainJoined : NO
DeviceAuthStatus : SUCCESS
```

![Windows 11 Entra join verified](screenshots/08-windows-11-entra-join-verified.png)

I also checked the device in Intune to make sure enrollment succeeded.

![Intune enrollment verified](screenshots/09-intune-device-enrollment-verified.png)

At this point the machine was both Microsoft Entra joined and managed by Intune.

---

# 3. Device Configuration with Intune

I created a Microsoft Edge Settings Catalog profile called:

```text
WIN-Edge-Homepage-Pilot
```

I assigned it to the device pilot group instead of targeting every device.

The policy configured:

```text
Homepage: https://www.office.com
Show Home button: Enabled
New tab page as home page: Disabled
```

![Edge policy assigned](screenshots/10-intune-edge-policy-assigned.png)

Before syncing, I opened `edge://policy` and confirmed that the new settings had not reached the endpoint yet.

![Edge policy before sync](screenshots/11-intune-edge-policy-before-sync.png)

After syncing the VM, I checked again and the Edge settings were present.

![Edge policy verified](screenshots/12-intune-edge-policy-verified.png)

This was a useful way to verify that I was looking at an actual policy change on the endpoint instead of assuming that creating the profile in Intune meant it had already applied.

---

# 4. Compliance Policy and Firewall Troubleshooting

I created a Windows compliance policy that required Microsoft Defender Firewall.

![Compliance policy assigned](screenshots/13-intune-compliance-policy-assigned.png)

With the firewall running normally, the VM was compliant.

![Firewall compliance verified](screenshots/14-intune-firewall-compliance-verified.png)

## Breaking the configuration

I intentionally disabled the active Windows Firewall profile.

![Firewall intentionally disabled](screenshots/15-firewall-active-profile-disabled.png)

After the next Intune evaluation, the device changed to:

```text
Not compliant
```

![Device marked noncompliant](screenshots/16-intune-firewall-noncompliant.png)

I opened the compliance details instead of stopping at the overall device status. Intune identified the failed setting as the firewall requirement.

![Firewall failure diagnosed](screenshots/17-intune-firewall-failure-diagnosed.png)

I turned the firewall back on, synced the machine again, and waited for Intune to reevaluate it.

![Firewall compliance restored](screenshots/18-intune-firewall-compliance-restored.png)

One thing I initially had to separate mentally was **configuration** versus **compliance**. The compliance policy detected that the firewall was off, but it did not turn the firewall back on for me. I had to correct the problem and then let Intune reevaluate the endpoint.

---

# 5. Application Deployment

## Company Portal

I deployed Company Portal from the Microsoft Store through Intune and made it required for the pilot device group.

![Company Portal assigned](screenshots/19-intune-company-portal-app-assigned.png)

I then verified that the application installed on the VM.

![Company Portal installed](screenshots/20-intune-company-portal-install-verified.png)

## 7-Zip Win32 Deployment

For a more traditional application deployment, I packaged 7-Zip as an Intune Win32 application.

The source installer was:

```text
7z2603-x64.msi
```

I used Microsoft's Win32 Content Prep Tool to create:

```text
7z2603-x64.intunewin
```

The install command was:

```cmd
msiexec /i "7z2603-x64.msi" /qn /norestart
```

The uninstall command was:

```cmd
msiexec /x "{23170F69-40C1-2702-2603-000001000000}" /qn /norestart
```

I configured the application for:

- x64 Windows
- System install context
- Silent installation
- MSI product-code detection
- Required deployment to the pilot device group

![7-Zip Win32 app assigned](screenshots/21-intune-win32-7zip-assigned.png)

The application installed successfully on the VM.

![7-Zip installation verified](screenshots/22-intune-win32-7zip-install-verified.png)

The Intune reporting status took longer to update than the actual installation. That was a good reminder to check both the endpoint and the Intune portal when troubleshooting an app deployment.

---

# 6. Conditional Access and MFA

While moving from Security Defaults to Conditional Access, Microsoft created several Microsoft-managed policies in the tenant. I left those policies enabled and created a separate pilot policy for my own testing.

![Conditional Access policy overview](screenshots/23-conditional-access-policy-overview.png)

My custom policy was:

```text
CA-Pilot-Require-MFA
```

It targeted Alex through:

```text
SG-CA-MFA-Pilot
```

I kept the policy in **Report-only** mode while testing it.

![MFA pilot policy](screenshots/24-conditional-access-mfa-pilot-report-only.png)

The grant control required MFA.

![MFA grant control](screenshots/25-conditional-access-mfa-grant-control.png)

## What If test

Before relying on a real sign-in, I used the Conditional Access What If tool to confirm that Alex matched the policy.

![Conditional Access What If verified](screenshots/26-conditional-access-what-if-verified.png)

## Checking a real sign-in

Alex already had MFA registered and Windows Hello for Business configured.

The authentication details showed that Windows Hello or a previously satisfied MFA claim could meet the authentication requirement without forcing a brand-new Authenticator prompt every time.

![Windows Hello authentication verified](screenshots/27-windows-hello-authentication-verified.png)

I then checked the Conditional Access results for a real OfficeHome sign-in.

```text
CA-Pilot-Require-MFA
Report-only: Success
```

![Conditional Access report-only success](screenshots/28-conditional-access-report-only-success.png)

I kept this policy in Report-only because the purpose of the lab was to verify how it evaluated sign-ins without accidentally locking myself out of the tenant.

---

# 7. Conditional Access Based on Device Compliance

I created another pilot policy:

```text
CA-Pilot-Require-Compliant-Windows-Device
```

![Compliant device policy](screenshots/29-conditional-access-compliant-device-policy.png)

I limited the device platform to Windows.

![Windows platform condition](screenshots/30-conditional-access-windows-platform-condition.png)

The grant requirement was:

```text
Require device to be marked as compliant
```

![Compliant-device grant control](screenshots/31-conditional-access-compliant-device-grant.png)

## Healthy test

With the firewall enabled and the VM compliant, the policy evaluated successfully.

![Compliant device success](screenshots/32-conditional-access-compliant-device-success.png)

## Failure test

I disabled the firewall again so the device would become noncompliant.

The next sign-in showed:

```text
CA-Pilot-Require-Compliant-Windows-Device
Report-only: Failure
```

![Noncompliant device Conditional Access failure](screenshots/33-conditional-access-noncompliant-device-failure.png)

I also checked the device information included with the sign-in.

```text
Managed: Yes
Compliant: No
Join Type: Azure AD joined
```

![Noncompliant managed device verified](screenshots/34-conditional-access-device-noncompliant-verified.png)

The `Azure AD joined` label is older Microsoft wording that still appears in parts of the portal.

### An issue I ran into

My first attempt at this test was done in an InPrivate browser window. The sign-in did not include the managed device identity, so the result was not testing what I thought it was testing.

I repeated the test through the normal Edge profile signed into the managed VM. The device ID was then present and Entra correctly showed:

```text
Managed: Yes
Compliant: No
```

That let me confirm that the Conditional Access failure was actually caused by compliance status.

After restoring the firewall and syncing the VM, the next sign-in returned to:

```text
Report-only: Success
```

![Conditional Access compliance restored](screenshots/35-conditional-access-compliance-restored.png)

This was one of the more useful parts of the lab because it tied together endpoint security, Intune compliance, device identity, and Entra sign-in decisions.

---

# 8. Windows Update Ring

I created:

```text
WIN-Update-Ring-Pilot
```

and assigned it to the pilot device group.

Some of the settings I used were:

```text
Microsoft product updates: Allow
Windows drivers: Allow

Quality update deferral: 0 days
Feature update deferral: 0 days

Active hours:
8:00 AM - 5:00 PM

Quality update deadline: 2 days
Feature update deadline: 7 days
Grace period: 1 day
```

![Windows Update ring settings](screenshots/36-intune-windows-update-ring-settings.png)

I did not rely only on the Intune configuration page. On `WIN11-INTUNE01`, I opened:

```text
Windows Update
→ Advanced options
→ Configured update policies
```

Windows showed the policies as coming from Mobile Device Management.

![Windows Update endpoint verification](screenshots/37-windows-update-policy-endpoint-verified.png)

I also checked the individual update settings on the endpoint.

![Windows Update policy details](screenshots/38-windows-update-policy-details-verified.png)

---

# 9. BitLocker Management

Before creating a BitLocker policy, I checked the VM itself.

```powershell
Get-Tpm | Select-Object TpmPresent,TpmReady,TpmEnabled,TpmActivated
```

```powershell
Confirm-SecureBootUEFI
```

```powershell
Get-ComputerInfo | Select-Object BiosFirmwareType
```

```cmd
reagentc /info
```

```powershell
Get-BitLockerVolume -MountPoint "C:"
```

The VM had:

```text
TPM present: True
TPM ready: True
TPM enabled: True
TPM activated: True

Secure Boot: True
Firmware: UEFI
Windows RE: Enabled

VolumeStatus: FullyEncrypted
ProtectionStatus: On
EncryptionPercentage: 100
EncryptionMethod: XtsAes128
```

![BitLocker prerequisite and encryption status](screenshots/39-bitlocker-preflight-and-encryption-status.png)

The VM was already encrypted before I created the Intune policy. I did not decrypt it just to make the lab start from an artificial clean state. Instead, I treated it as an already encrypted corporate device and configured Intune to manage the existing BitLocker setup.

I created:

```text
WIN-BitLocker-Pilot
```

![BitLocker policy assigned](screenshots/40-intune-bitlocker-policy-assigned.png)

I configured TPM-based startup settings.

![BitLocker TPM settings](screenshots/41-intune-bitlocker-tpm-settings.png)

I also configured recovery behavior and required the recovery information to be stored centrally.

![BitLocker recovery settings](screenshots/42-intune-bitlocker-recovery-settings.png)

I verified that a recovery-key record existed in Intune without exposing the actual recovery password.

![BitLocker recovery key escrow verified](screenshots/43-bitlocker-recovery-key-escrow-verified.png)

The policy later reported:

```text
Succeeded: 1
Errors: 0
Conflicts: 0
```

![BitLocker policy succeeded](screenshots/44-intune-bitlocker-policy-succeeded.png)

---

# 10. Help Desk Account Troubleshooting

I wanted one section of the lab to look more like a normal help desk ticket instead of another policy deployment.

I created a new employee:

```text
Elena Marquez
Procurement Coordinator
Operations
```

![Help desk user provisioned](screenshots/45-entra-helpdesk-user-provisioned.png)

I assigned Microsoft 365 Business Premium to the account.

![Business Premium license assigned](screenshots/46-microsoft-365-business-premium-license-assigned.png)

After completing the initial password change and MFA registration, I confirmed that Elena could sign in successfully.

![Help desk baseline sign-in](screenshots/47-entra-helpdesk-baseline-signin-success.png)

## Simulated account problem

I blocked Elena from signing in through the Microsoft 365 admin tools.

![User sign-in blocked](screenshots/48-helpdesk-user-signin-blocked.png)

I then attempted a new sign-in and confirmed that access failed.

![Blocked user sign-in failure](screenshots/49-helpdesk-blocked-user-signin-failure.png)

Instead of treating the user-facing message as the diagnosis, I checked the Entra sign-in logs.

The failure showed:

```text
Error code: 50057
Failure reason: The user account is disabled.
```

![Blocked sign-in diagnosed](screenshots/50-entra-helpdesk-blocked-signin-diagnosed.png)

I restored the account.

![User sign-in restored](screenshots/51-helpdesk-user-signin-restored.png)

A new sign-in succeeded.

![Restored sign-in verified](screenshots/52-entra-helpdesk-signin-restored-verified.png)

This was a simple scenario, but it was useful because the message shown to the user was less specific than the information available to the administrator in the sign-in logs.

---

# 11. Windows LAPS

I enabled Microsoft Entra Windows LAPS for the tenant.

![Microsoft Entra Windows LAPS enabled](screenshots/53-entra-windows-laps-enabled.png)

I created a policy called:

```text
WIN-LAPS-Pilot
```

Some of the main settings were:

```text
Backup directory: Microsoft Entra ID only
Password age: 30 days
Password length: 20 characters
Automatic account management: Enabled
Managed account: Cobo-LAPSAdmin
```

![Windows LAPS policy configured](screenshots/54-intune-windows-laps-policy-configured.png)

After the device processed the policy, Windows created the managed local administrator account.

I checked it with:

```powershell
Get-LocalUser | Select-Object Name,Enabled,Description
```

![LAPS local administrator created](screenshots/55-windows-laps-local-admin-created.png)

The account did not appear immediately after the policy was created. I had to sync the VM and wait for the policy to process before verifying it locally.

I then checked Intune and confirmed that the LAPS password record had been backed up. The actual password is not shown in the screenshot or repository.

![LAPS password backup verified](screenshots/56-laps-password-backup-verified.png)

---

# 12. Remote Device Action

For the last exercise, I sent a remote restart to:

```text
WIN11-INTUNE01
```

from Intune.

![Remote restart initiated](screenshots/57-intune-remote-restart-initiated.png)

The VM received the command and displayed a Windows notification that the device administrator had scheduled a restart.

![Remote restart received](screenshots/58-intune-remote-restart-received.png)

The Intune Device action status page did not give me a useful completed status afterward, so I used the endpoint receiving the restart command as my verification instead of adding another screenshot that did not show anything useful.

---

# Problems I Ran Into

A few parts of the lab did not behave exactly the way I expected on the first attempt.

| Problem | What I found |
|---|---|
| Conditional Access test did not identify the device correctly | I was using InPrivate mode, so I repeated the test from the normal managed Edge profile |
| Firewall compliance changed slowly | The endpoint state changed before Intune reporting fully caught up |
| 7-Zip showed installed locally before Intune reported it | Intune reporting was delayed |
| Windows LAPS account was not created immediately | The device needed another sync and time to process the policy |
| Remote restart status was not useful in Intune | The Windows VM receiving the administrator restart notification confirmed the action reached the endpoint |

These were useful because they forced me to verify the result from more than one place instead of assuming that the portal always updates immediately.

---

# Commands Used

Some of the Windows and PowerShell commands I used during the lab included:

```powershell
dsregcmd /status
```

```powershell
Get-Tpm
```

```powershell
Confirm-SecureBootUEFI
```

```powershell
Get-ComputerInfo | Select-Object BiosFirmwareType
```

```cmd
reagentc /info
```

```powershell
Get-BitLockerVolume -MountPoint "C:"
```

```powershell
Get-LocalUser | Select-Object Name,Enabled,Description
```

For the Win32 deployment:

```cmd
msiexec /i "7z2603-x64.msi" /qn /norestart
```

```cmd
msiexec /x "{23170F69-40C1-2702-2603-000001000000}" /qn /norestart
```

---

# What I Took Away From the Lab

The biggest thing I learned from this project was that Microsoft cloud administration is usually a chain of related systems rather than one isolated setting.

For example, one Conditional Access test depended on all of these being correct:

```text
Windows device
      ↓
Microsoft Entra device identity
      ↓
Intune enrollment
      ↓
Intune compliance
      ↓
Conditional Access evaluation
      ↓
User sign-in
```

I also became much more comfortable checking both sides of a change. Creating a policy in Intune is only half of the work. I still need to verify what the Windows endpoint actually received.

The same applied to troubleshooting. Intune, Entra sign-in logs, Windows settings, PowerShell output, and application state can all show different parts of the same problem.

---

# Skills Used

- Microsoft 365 administration
- Microsoft Entra ID
- Microsoft Intune
- Windows 11 administration
- User and group management
- Microsoft 365 licensing
- Device enrollment
- Settings Catalog policies
- Device compliance
- Conditional Access
- MFA
- Microsoft Store app deployment
- Win32 app packaging and deployment
- MSI installation
- Windows Update Rings
- BitLocker
- Windows LAPS
- Entra sign-in logs
- Help desk troubleshooting
- Remote endpoint administration
- PowerShell

---

# Repository Structure

```text
microsoft-365-entra-intune-lab/
│
├── README.md
│
└── screenshots/
    ├── 01-microsoft-365-tenant-created.png
    ├── 02-entra-id-first-user-created.png
    ├── 03-entra-id-it-security-group.png
    ├── ...
    ├── 56-laps-password-backup-verified.png
    ├── 57-intune-remote-restart-initiated.png
    └── 58-intune-remote-restart-received.png
```

---

# Security

The repository does not contain passwords, temporary passwords, MFA secrets, BitLocker recovery passwords, Windows LAPS passwords, or billing information.

The bulk-user CSV, 7-Zip installer, and `.intunewin` package were used inside the lab but are not included in the repository.

---

# Status

**Completed**

The lab covers the full path from creating cloud identities and enrolling a Windows device through policy management, application deployment, access control, troubleshooting, credential management, and remote administration.
