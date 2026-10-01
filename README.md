# Active Directory Home Lab

## Overview

I built this lab to get hands-on experience administering a Windows domain instead of only studying Active Directory concepts.

The environment started with one Windows Server 2025 domain controller and one Windows 11 workstation. I configured the domain, DNS, DHCP, organizational units, security groups, file sharing, Group Policy, and PowerShell-based user provisioning.

After the basic environment was working, I added a second domain controller and tested Active Directory, DNS, and DHCP redundancy by actually shutting down the primary server.

I also intentionally broke several parts of the environment so I could practice diagnosing problems from the client and server sides.

### Recruiter Snapshot

| Capability | Verified hands-on work |
|---|---|
| Identity and access | Created organizational units, domain users, global and domain local security groups, and AGDLP-based file permissions |
| Endpoint support | Joined a Windows 11 client to `cobo.test`, validated domain sign-in, mapped a departmental drive, and confirmed Group Policy application |
| User provisioning | Wrote a PowerShell workflow that reads a CSV, creates users, assigns the correct OU and security group, skips existing accounts, and exports results |
| Core services | Configured Active Directory-integrated DNS, forward and reverse lookup, DHCP scope options, and Windows client addressing |
| Resilience | Added a second writable domain controller and DNS server, configured 50/50 DHCP failover, and verified authentication, secure-channel discovery, DNS, and DHCP renewal while DC01 was offline |
| Troubleshooting | Diagnosed incorrect client DNS, a computer outside the GPO-linked OU, broken group-based file access, an account lockout, and primary-server outages |

### Relevance to Entry-Level IT Support and Systems Administration

- Mirrors common support work such as onboarding users, assigning access, joining endpoints to a domain, resolving account lockouts, validating mapped drives, and troubleshooting DNS, DHCP, and Group Policy.
- Uses client and server evidence instead of assuming that a configuration succeeded. Validation included `gpresult`, `Resolve-DnsName`, `repadmin /replsummary`, `nltest`, Active Directory PowerShell cmdlets, and forced DHCP renewal.
- Demonstrates service dependency awareness. A client can have basic IP connectivity while domain discovery, policy processing, authentication, or authorization still fails.
- Includes reproducible evidence through 40 numbered screenshots, the PowerShell provisioning script, and sample CSV data.

> **Scope:** This is a controlled home lab built for hands-on learning. It demonstrates foundational administration and troubleshooting, not production enterprise ownership.

### Main areas covered

- Active Directory Domain Services
- Windows Server 2025
- DNS
- DHCP
- Group Policy
- Organizational Units
- Security groups
- AGDLP permissions
- SMB file sharing
- NTFS permissions
- PowerShell user provisioning
- Active Directory replication
- Domain controller redundancy
- DHCP failover
- Account lockout troubleshooting
- Windows 11 domain administration

---

## Lab Environment

| Component | Configuration |
|---|---|
| Domain | `cobo.test` |
| Primary Domain Controller | `DC01` |
| Secondary Domain Controller | `DC02` |
| Server OS | Windows Server 2025 |
| Client | `CLIENT01` |
| Client OS | Windows 11 Pro |
| Network | `10.10.10.0/24` |
| Default Gateway | `10.10.10.1` |
| DC01 | `10.10.10.10` |
| DC02 | `10.10.10.11` |
| DHCP Pool | `10.10.10.100 - 10.10.10.200` |
| Virtualization | Oracle VirtualBox |

---

## Network Layout

```text
                        AD-Lab
                     10.10.10.0/24
                           |
                     10.10.10.1
                  VirtualBox Gateway
                           |
          +----------------+----------------+
          |                                 |
        DC01                              DC02
   10.10.10.10                       10.10.10.11
   Windows Server 2025               Windows Server 2025
          |                                 |
   AD DS / DNS / DHCP                AD DS / DNS / DHCP
          |                                 |
          +--------- Replication -----------+
          +-------- DHCP Failover ----------+
                           |
                           |
                       CLIENT01
                    Windows 11 Pro
                     DHCP Client
                           |
                     cobo.test domain
```

---

# 1. Building the First Domain Controller

I started with a Windows Server 2025 VM and renamed it:

```text
DC01
```

![DC01 renamed](screenshots/01-dc01-server-renamed.png)

Because a domain controller and DNS server need a predictable address, I configured DC01 with a static IPv4 address.

```text
IP Address: 10.10.10.10
Subnet Mask: 255.255.255.0
Default Gateway: 10.10.10.1
```

![DC01 static IP configuration](screenshots/02-dc01-static-ip.png)

I installed Active Directory Domain Services and created the forest:

```text
cobo.test
```

![Active Directory domain created](screenshots/03-active-directory-domain-created.png)

At this point DC01 was the only domain controller in the environment, so Active Directory and DNS both depended on this server.

---

# 2. Organizational Units and Security Groups

I created an OU structure to keep users and computers organized instead of leaving everything in the default containers.

Departments included:

```text
Accounting
Human Resources
IT
Management
Sales
```

I also created a separate location for workstation objects.

![Organizational Unit structure](screenshots/04-organizational-unit-structure.png)

I created departmental security groups rather than assigning permissions directly to individual users.

![Security groups](screenshots/05-security-groups.png)

For file access, I used an AGDLP-style permission structure.

One example was:

```text
Alex Rivera
     ↓
GG_IT_Users
     ↓
DL_IT_Share_RW
     ↓
IT Shared Folder
```

The user belongs to a Global Group, the Global Group belongs to a Domain Local Group, and the Domain Local Group receives the actual resource permission.

This made it easier to change access later without editing the file permissions every time an employee changes roles.

---

# 3. IT File Share

On DC01, I created an IT department share:

```text
\\DC01\IT
```

I configured the file system and share permissions so access was controlled through the security groups I had already created.

![IT share permissions](screenshots/06-it-share-permissions.png)

Instead of granting Alex Rivera access directly, the permission came through:

```text
GG_IT_Users
        ↓
DL_IT_Share_RW
        ↓
IT Share
```

I later used this same group relationship for one of the troubleshooting exercises.

---

# 4. Joining CLIENT01 to the Domain

I created a Windows 11 Pro VM named:

```text
CLIENT01
```

Before attempting the domain join, I checked its network configuration.

![CLIENT01 network configuration](screenshots/07-client01-network-configuration.png)

One of the most important parts of the setup was making sure CLIENT01 used the internal Active Directory DNS server.

I tested name resolution before attempting the join.

![CLIENT01 DNS validation](screenshots/08-client01-dns-validation.png)

Once DNS was working, I joined CLIENT01 to:

```text
cobo.test
```

![CLIENT01 domain joined](screenshots/09-client01-domain-joined.png)

I then checked Active Directory Users and Computers and confirmed the workstation object had been created.

![CLIENT01 Active Directory object](screenshots/10-client01-active-directory-object.png)

I signed into Windows using the domain account:

```text
COBO\arivera
```

![Domain user authentication](screenshots/11-domain-user-authentication.png)

Finally, I tested access to the IT share from the workstation.

![Domain user file share access](screenshots/12-domain-user-file-share-access.png)

At this point I had a working path from:

```text
Domain account
      ↓
CLIENT01 authentication
      ↓
Security-group membership
      ↓
File-share permissions
```

---

# 5. Group Policy

I created Group Policy Objects for workstation configuration and user settings.

## Workstation policy

I linked a workstation security policy to the Workstations OU and verified that CLIENT01 actually received it.

![Workstation GPO applied](screenshots/13-workstation-gpo-applied.png)

I used tools such as:

```powershell
gpupdate /force
```

and:

```powershell
gpresult
```

to check policy processing instead of assuming that creating the GPO meant the client had received it.

## IT drive mapping

I also created a user policy that mapped:

```text
I:
```

to:

```text
\\DC01\IT
```

![IT drive mapping GPO](screenshots/14-it-drive-mapping-gpo.png)

I signed into CLIENT01 with Alex's account and verified that the mapped drive appeared.

![Drive mapping verified](screenshots/15-user-gpo-drive-mapping-verified.png)

---

# 6. DHCP

I installed the DHCP Server role on DC01 and created a scope for the lab network.

```text
Network: 10.10.10.0/24
Pool: 10.10.10.100 - 10.10.10.200
Gateway: 10.10.10.1
DNS Server: 10.10.10.10
DNS Domain: cobo.test
```

![DHCP scope configuration](screenshots/16-dhcp-scope-configuration.png)

CLIENT01 received an address from the new scope.

![CLIENT01 DHCP lease](screenshots/17-client01-dhcp-lease.png)

I also checked the client configuration directly to make sure DHCP had supplied the expected address, gateway, DNS server, and domain information.

![CLIENT01 DHCP configuration verified](screenshots/18-client01-dhcp-configuration-verified.png)

---

# 7. PowerShell User Provisioning

After creating users manually, I wanted a faster way to provision multiple employees.

I wrote a PowerShell script:

```text
scripts/New-COBOUsers.ps1
```

The script reads employee information from:

```text
data/new-users.csv
```

and handles the account setup automatically.

The script can:

- Generate usernames
- Create Active Directory users
- Set department information
- Place users into the correct OU
- Add users to departmental security groups
- Require a password change at first sign-in
- Skip accounts that already exist
- Catch provisioning errors
- Export a results report

After running the script, I checked Active Directory to verify that the accounts were actually created.

![Automated Active Directory users](screenshots/19-automated-ad-users-verified.png)

I also checked the resulting group membership.

![Security group membership verified](screenshots/20-security-group-membership-verified.png)

The script returned provisioning results after processing the CSV.

![PowerShell provisioning results](screenshots/21-powershell-provisioning-results.png)

I then checked the exported report.

![Provisioning report verified](screenshots/22-provisioning-report-verified.png)

The script is included in the repository:

[`New-COBOUsers.ps1`](scripts/New-COBOUsers.ps1)

The sample CSV is also included:

[`new-users.csv`](data/new-users.csv)

---

# 8. DNS

I configured both forward and reverse DNS lookup.

The forward record allowed:

```text
dc01.cobo.test
```

to resolve to:

```text
10.10.10.10
```

I also created the reverse lookup zone and PTR record.

![DNS reverse lookup zone](screenshots/23-dns-reverse-lookup-zone.png)

I tested both directions:

```text
dc01.cobo.test → 10.10.10.10
10.10.10.10 → dc01.cobo.test
```

![DNS forward and reverse lookup verified](screenshots/24-dns-forward-reverse-lookup-verified.png)

DNS ended up being one of the most important parts of the lab because Active Directory depended on it for more than basic hostname lookup.

---

# 9. Troubleshooting: Incorrect DNS Server

For the first troubleshooting exercise, I intentionally changed CLIENT01 to use:

```text
8.8.8.8
```

instead of the internal DNS server.

The workstation still had a valid IP address and could reach DC01 by IP, but internal domain name resolution and Active Directory discovery stopped working.

![DNS troubleshooting failure](screenshots/25-troubleshooting-dns-failure.png)

Because IP connectivity was still working, I did not treat it as a general network failure. The problem was DNS.

I restored the proper DNS configuration, flushed the cache, and tested the domain names again.

![DNS troubleshooting resolved](screenshots/26-troubleshooting-dns-resolved.png)

This was a useful distinction:

```text
Can reach server by IP
        +
Cannot resolve domain services
        ↓
Investigate DNS
```

---

# 10. Troubleshooting: Group Policy Not Applying

I intentionally moved CLIENT01 out of the Workstations OU and into the default Computers container.

The machine remained joined to the domain and network connectivity still worked, but the workstation GPO was no longer in scope.

![GPO not applied](screenshots/27-troubleshooting-gpo-not-applied.png)

I moved CLIENT01 back into the correct OU and refreshed Group Policy.

```powershell
gpupdate /force
```

Afterward, `gpresult` showed the workstation policy again.

![GPO restored](screenshots/28-troubleshooting-gpo-restored.png)

The problem was not that Group Policy itself was broken. The computer object had simply been moved outside the location where the policy was linked.

---

# 11. Troubleshooting: File Share Access Denied

For the file-share test, I removed:

```text
GG_IT_Users
```

from:

```text
DL_IT_Share_RW
```

After refreshing the user's security token, Alex could still reach DC01 over SMB, but:

```text
\\DC01\IT
```

returned:

```text
Access Denied
```

![File share access denied](screenshots/29-troubleshooting-file-share-access-denied.png)

Since the workstation could still communicate with the file server, I focused on authorization rather than network connectivity.

I restored the group relationship with:

```powershell
Add-ADGroupMember DL_IT_Share_RW -Members GG_IT_Users
```

![Permission group restored](screenshots/30a-file-share-permission-group-restored.png)

After signing out and back in again, the user's new security token included the restored group membership.

Access to the share returned.

![File share access restored](screenshots/30-troubleshooting-file-share-access-restored.png)

This exercise made the AGDLP structure much easier to understand because breaking one group relationship removed the user's access without changing the NTFS permission itself.

---

# 12. Account Lockout and Recovery

I configured a domain account-lockout policy:

```text
Account lockout threshold: 5 attempts
Account lockout duration: 15 minutes
Reset counter after: 15 minutes
```

![Account lockout policy configured](screenshots/31-account-lockout-policy-configured.png)

I entered the wrong password repeatedly for:

```text
arivera
```

until the account locked.

On the domain controller, I checked the account with:

```powershell
Get-ADUser arivera -Properties LockedOut
```

and:

```powershell
Search-ADAccount -LockedOut
```

![Account locked](screenshots/32-troubleshooting-account-locked.png)

I unlocked it with:

```powershell
Unlock-ADAccount -Identity arivera
```

and checked the account again.

![Account lockout restored](screenshots/33-account-lockout-restored.png)

---

# 13. Adding a Second Domain Controller

The original environment depended entirely on DC01.

If DC01 was unavailable, the lab lost:

```text
Active Directory
DNS
DHCP
```

I added a second Windows Server 2025 VM named:

```text
DC02
```

and gave it the static address:

```text
10.10.10.11
```

Before promoting it, I joined DC02 to `cobo.test` as a normal domain member.

![DC02 domain member network configuration](screenshots/34-dc02-domain-member-network-config.png)

I then installed Active Directory Domain Services and promoted DC02 as an additional writable domain controller and DNS server.

## Replication

After promotion, I checked replication with:

```powershell
repadmin /replsummary
```

Both domain controllers reported zero replication failures.

![DC02 replication verified](screenshots/35-dc02-domain-controller-replication.png)

I did more than check the replication summary. I queried DC02 directly for Active Directory and DNS information.

Examples included:

```powershell
Get-ADUser arivera -Server DC02.cobo.test -Properties Department
```

```powershell
Get-ADGroupMember GG_IT_Users -Server DC02.cobo.test
```

```powershell
Resolve-DnsName dc01.cobo.test -Server 10.10.10.11
```

![DC02 Active Directory and DNS replication verified](screenshots/36-dc02-ad-dns-replication-verified.png)

---

# 14. Redundant DNS

Once DC02 was working as a DNS server, I updated DHCP so CLIENT01 received both domain DNS servers.

```text
Primary DNS: 10.10.10.10
Secondary DNS: 10.10.10.11
```

![CLIENT01 redundant DNS configuration](screenshots/37-client01-redundant-dns-configuration.png)

I wanted to test whether that redundancy actually worked instead of just seeing two addresses in `ipconfig`.

---

# 15. Domain Controller Failover Test

I shut down DC01.

On CLIENT01, I forced domain-controller discovery:

```powershell
nltest /dsgetdc:cobo.test /force
```

The workstation found:

```text
DC: \\DC02.cobo.test
Address: \\10.10.10.11
```

I also checked the secure channel:

```powershell
nltest /sc_verify:cobo.test
```

The check succeeded while DC01 was offline.

![DC02 domain failover verified](screenshots/38-dc02-domain-failover-verified.png)

That gave me much more confidence in the second domain controller than simply seeing it listed in Active Directory Sites and Services.

After the test, I brought DC01 back online and checked replication again.

---

# 16. DHCP Failover

Active Directory and DNS now had redundancy, but DHCP was still dependent on DC01.

I installed and authorized DHCP on DC02 and created a failover relationship using the existing client scope.

The relationship was configured as:

```text
Relationship: COBO-DHCP-Failover
Partner: DC02.cobo.test
Mode: Load Balance
Distribution: 50/50
Maximum Client Lead Time: 1 hour
Scope: 10.10.10.0/24
```

I checked the relationship with:

```powershell
Get-DhcpServerv4Failover -ComputerName DC01 |
    Format-List Name,PartnerServer,Mode,State,LoadBalancePercent,MaxClientLeadTime
```

I also confirmed that the scope was present on DC02:

```powershell
Get-DhcpServerv4Scope -ComputerName DC02
```

![DHCP failover relationship](screenshots/39-dhcp-failover-relationship.png)

## Testing it

I shut down DC01 again.

On CLIENT01, I forced a new DHCP request:

```powershell
ipconfig /release
ipconfig /renew
ipconfig /all
```

CLIENT01 received its lease from:

```text
DHCP Server: 10.10.10.11
```

![DHCP failover through DC02](screenshots/40-dhcp-failover-dc02-lease.png)

That confirmed DC02 could continue servicing DHCP clients while DC01 was unavailable.

After bringing DC01 back online, I checked both Active Directory replication and the DHCP relationship again to make sure the environment recovered normally.

---

# Problems I Intentionally Tested

A large part of this lab was learning how to narrow down a problem instead of changing settings randomly.

| Problem | What pointed me toward the cause |
|---|---|
| Internal DNS stopped working | IP connectivity still worked, but domain names and AD discovery failed |
| Workstation GPO disappeared | CLIENT01 was healthy but had been moved outside the linked OU |
| File share returned Access Denied | SMB connectivity worked, so I checked group-based authorization |
| User could not authenticate | The account showed as locked in Active Directory |
| DC01 was offline | CLIENT01 discovered DC02 and kept a valid domain secure channel |
| DHCP on DC01 was unavailable | CLIENT01 successfully renewed from DC02 |

The main troubleshooting pattern I used was:

```text
Check what still works
        ↓
Narrow down the affected service
        ↓
Verify the suspected cause
        ↓
Make one correction
        ↓
Test again
```

---

# Commands I Used Frequently

### Active Directory

```powershell
Get-ADUser
Get-ADGroupMember
Add-ADGroupMember
Unlock-ADAccount
Search-ADAccount
```

### Group Policy

```powershell
gpupdate /force
gpresult
```

### DNS

```powershell
Resolve-DnsName
ipconfig /flushdns
```

### Domain Controller Testing

```powershell
repadmin /replsummary
nltest /dsgetdc:cobo.test /force
nltest /sc_verify:cobo.test
```

### DHCP

```powershell
Get-DhcpServerv4Scope
Get-DhcpServerv4Failover
```

```cmd
ipconfig /release
ipconfig /renew
ipconfig /all
```

---

# What I Took Away From the Lab

The biggest thing this project changed for me was how I think about Active Directory dependencies.

A domain login is not just "Active Directory working." A workstation may depend on several services at the same time:

```text
CLIENT01
   |
   +---- IP configuration
   |
   +---- DNS
   |
   +---- Domain Controller discovery
   |
   +---- User authentication
   |
   +---- Group membership
   |
   +---- Group Policy
   |
   +---- Resource permissions
```

The troubleshooting exercises helped me separate those layers.

For example, being able to ping a server did not mean Active Directory was healthy if DNS was wrong. Likewise, being able to connect to TCP 445 did not mean a user was authorized to access an SMB share.

Adding DC02 was also useful because I could test redundancy instead of only configuring it. Shutting down DC01 and confirming that authentication, DNS, and DHCP could continue through DC02 made the purpose of the second server much clearer.

---

# Skills Used

- Windows Server 2025 administration
- Active Directory Domain Services
- User and computer administration
- Organizational Unit design
- Security groups
- AGDLP permissions
- Group Policy
- DNS administration
- DNS troubleshooting
- DHCP administration
- DHCP failover
- Active Directory replication
- Domain controller redundancy
- SMB file sharing
- NTFS permissions
- Account lockout administration
- PowerShell
- CSV-based user provisioning
- Windows 11 domain administration
- Network troubleshooting
- Identity and access troubleshooting

---

# Repository Structure

```text
ActiveDirectoryLab/
│
├── README.md
│
├── scripts/
│   └── New-COBOUsers.ps1
│
├── data/
│   └── new-users.csv
│
└── screenshots/
    ├── 01-dc01-server-renamed.png
    ├── 02-dc01-static-ip.png
    ├── 03-active-directory-domain-created.png
    ├── ...
    ├── 38-dc02-domain-failover-verified.png
    ├── 39-dhcp-failover-relationship.png
    └── 40-dhcp-failover-dc02-lease.png
```

---

# Security

The repository does not contain real user passwords or production credentials.

The users, domain, network, company information, and employee data in this project were created for the lab environment.

---

# Status

**Completed**

The finished environment includes two domain controllers, redundant DNS, load-balanced DHCP failover, centralized users and groups, Group Policy, shared-resource permissions, PowerShell provisioning, and tested client/server troubleshooting scenarios.
