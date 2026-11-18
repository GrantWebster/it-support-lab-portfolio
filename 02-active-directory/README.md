# Lab 02: Active Directory Domain for ByteKelp Systems

[Back to portfolio](../README.md)

**Outcome:** Built a Windows Server 2025 Active Directory domain for a fictional 15-person company in VirtualBox. I promoted a domain controller, configured DNS, created the OU and group structure, joined a Windows 11 client, handled everyday helpdesk tasks (password resets, group changes, leavers and account lockouts) and applied Group Policy.

|                |                                                         |
| -------------- | ------------------------------------------------------- |
| **Time taken** | 8 - 10 Hours                                            |
| **Tools**      | VirtualBox, Windows Server 2025, Windows 11, PowerShell |
| **Domain**     | `corp.internal` (NetBIOS name `CORP`)                   |
| **Network**    | LabNet `192.168.50.0/24`                                |

## Summary

This lab shows how I installed Windows Server, promoted it to a domain controller and ran an Active Directory domain for a small company. It covers system administration, Active Directory, role-based access control through security groups, and setting up OUs and Group Policy Objects (GPOs).

> **About the company:** ByteKelp Systems is a fictional 15-person Cape Town software company used as the scenario for this lab. All names, roles and data are invented. This lab builds its Active Directory domain: six departments, 14 staff accounts, security groups per department and Group Policy.

## Environment

| Item | Detail |
|---|---|
| Hypervisor | Oracle VirtualBox |
| Domain controller | DC01, Windows Server 2025 Standard Evaluation (Desktop Experience), `192.168.50.10` |
| Client | CLIENT01, Windows 11, `192.168.50.50` |
| Network | VirtualBox NAT Network named LabNet (`192.168.50.0/24`) |
| Domain | `corp.internal` (NetBIOS name `CORP`) |
| Tools | Server Manager, Active Directory Users and Computers, Group Policy Management, Event Viewer, PowerShell |

## Objectives

- Install and configure a domain controller
- Build an OU and security group structure and create user accounts
- Join a client computer to the domain
- Perform core everyday administration tasks
- Create and test Group Policy

---

# Part 1: Build the domain

### Step 1. Create the VM and install the server OS

I started by creating the virtual machine and installing the Windows Server operating system.

![VirtualBox New Virtual Machine wizard for DC01 with the Windows Server 2025 ISO selected](screenshots/01-setup-new-vm.png)

![Installing Windows Server 2025](screenshots/02-install-server-os.png)

### Step 2. Guest Additions, static IP and updates

Once the installation was complete (with a strong administrator password), I installed the VirtualBox Guest Additions. This is a package of drivers that improves performance and adds features such as screen resizing and a shared clipboard.

![Installing the VirtualBox Guest Additions](screenshots/03-install-guest-additions.png)

Next I gave the domain controller a static IP address and a name, then installed all system updates:

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.50.10 -PrefixLength 24 -DefaultGateway 192.168.50.1
Rename-Computer -NewName DC01 -Restart
```

### Step 3. Install Active Directory Domain Services

I installed the Active Directory Domain Services (AD DS) role using Server Manager.

![Installing the AD DS role in Server Manager](screenshots/04-install-adds.png)

### Step 4. Promote the server to a domain controller

Installing the role only adds the tools. Promotion is the separate step that creates the domain. I chose **Add a new forest** and used `corp.internal` as the root domain name.

The below screenshot isn't exactly correct, I took it just before changing the radio button. You would click where its says Add a new forest. 
![Active Directory Domain Services Configuration Wizard, Deployment Configuration page](screenshots/05-setup-dc-name.png)

The wizard showed a DNS delegation warning. This is expected for a new forest, because there is no parent zone to register with, so I continued.

**Note on naming:** In production I would use a subdomain of the company's registered domain, such as `corp.bytekelp.com`. That avoids split-DNS conflicts with the public website. In this lab, the company is fictional, so I used `corp.internal`. The `.internal` top-level domain is reserved for private networks.

### Step 5. Configure DNS and verify the domain controller

DC01 needs to use itself as its DNS server, and a DNS forwarder lets it resolve outside names (for example for Windows Update) by passing them to a public resolver:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.50.10
Add-DnsServerForwarder -IPAddress 1.1.1.1 -PassThru
```

Then I checked that the domain and services were healthy:

```powershell
Get-ADDomain | Select-Object Name, DNSRoot, NetBIOSName, DomainMode, Forest, PDCEmulator, InfrastructureMaster, RIDMaster
Get-ADForest | Select-Object Name, ForestMode, RootDomain, SchemaMaster, DomainNamingMaster
Get-Service NTDS, DNS, Netlogon
```

![PowerShell output confirming the corp.internal domain, forest and running AD services](screenshots/06-verify-domain-controller.png)

### Step 6. Build the organisation structure, users and groups

I set up the OU structure, then added users and security groups for each department.

![Active Directory Users and Computers showing the Company OU and department OUs](screenshots/07-directory-organisational-structure.png)

![Adding a new user in Active Directory Users and Computers](screenshots/08-adding-users.png)

I also filled in each user's job title and department, because the groups are built from this information.

![A user's Organization tab with job title and department](screenshots/09-users-org-tab.png)

![Members of a department security group](screenshots/010-groups-members.png)

**Note:** I did this with the GUI to learn how it works. Creating 14 users by hand took a long time, and it showed me the power of scripting. It would be very inefficient for a company with hundreds of employees.

### Step 7. Build CLIENT01 and join the domain

I built CLIENT01, set its DNS server to DC01 and joined it to the domain.

![CLIENT01 joined to the domain, shown as a computer object in Active Directory](screenshots/011-client-joined-domain.png)

### Step 8. Rename the client

I forgot to set the client's computer name before joining, so it appeared as `DESKTOP-...`. I renamed it to CLIENT01 and cleaned up the old DNS record for the previous name.

![The computer object renamed to CLIENT01](screenshots/02-renamed-client-pc.png)

---

# Part 2: Everyday admin tasks

### Exercise 1: Reset a password (Anika Roux)

**Scenario:** Anika in HR has forgotten her password.

I reset her password in Active Directory Users and Computers and forced a change at next sign-in. In real life I would verify her identity before resetting it.

![Reset Password dialog with "User must change password at next logon" ticked](screenshots/012-password-reset-exercise-1.png)

When Anika signed in, Windows forced her to choose a new password:

![Sign-in screen saying the password must be changed before signing in](screenshots/013-password-reset-exercise-2.png)

![Sign-in screen asking Anika to enter a new password](screenshots/014-password-reset-exercise-3.png)

### Exercise 2: Add and remove group membership

**Scenario:** Zoe Abrahams (Finance) is covering for HR for a week.

I added Zoe to the `GG-HR` security group for the time required, and I would remove her afterwards.

![Adding a user to the GG-HR security group](screenshots/015-group-member-exercise-2.png)

### Exercise 3: Leaver process (Priya Reddy)

**Scenario:** HR says Priya Reddy leaves on Friday.

My leaver steps are: disable the account, remove her from groups such as `GG-Sales`, and move her into the Disabled Users OU. I disable rather than delete so files and records are kept if they are needed later.

![Disabling a user account](screenshots/016-user-disable.png)

![The disabled user moved into the Disabled Users OU](screenshots/017-disabled-user-leaver.png)

### Exercise 4: Account lockout (Tyler Jansen)

**Part A: Set the lockout policy.** I set the lockout policy in the Group Policy Management Editor and forced the domain to update.

![Account lockout policy settings](screenshots/018-set-lockout-policy.png)

![Running gpupdate to apply the policy](screenshots/019-updating-the-policy.png)

**Part B: Trigger a lockout.** I entered a wrong password for a test user until the account locked.

![Sign-in screen saying the account is locked out](screenshots/020-trigger-lockout.png)

**Part C: Find, investigate and unlock it.** I created a saved query to find locked-out accounts, then unlocked the account and made sure the user had to change their password.

![A saved query listing the locked-out account and the Unlock account option](screenshots/021-unlock-account.png)

Event Viewer made the lockout easy to investigate. Event ID 4740 shows which account was locked out and which computer it came from, and event ID 4625 records the failed sign-in attempts.

![Event Viewer showing events 4740 and 4625 for the locked-out account](screenshots/022-investigating-lockout.png)

Finally, the unlocked account:

![The unlocked user account](screenshots/023-unlocked-account-2.png)

---

# Part 3: Group Policy

I created three GPOs and linked each to the right OU.

First, a GPO linked to the Sales OU:

![Creating and linking a new GPO to the Sales OU](screenshots/024-creating-new-gpo.png)

Then I edited it to block Control Panel and PC settings for Sales users:

![Editing the GPO to prohibit access to Control Panel](screenshots/025-editing-new-gpo.png)

Two more GPOs: one for all workstations that sets a machine inactivity limit, and one company-wide GPO that shows a message at sign-in.

![The workstation GPO setting the machine inactivity limit](screenshots/026-new-workstation-gpo.png)

![The company GPO showing a sign-in message](screenshots/027-new-company-gpo.png)

**Test:** a Sales user can no longer open Control Panel or settings.

![Restrictions message shown when a Sales user tries to open Control Panel](screenshots/028-gpo-test-working.png)

---

## GUI vs PowerShell cheat sheet

Learning the cmdlets showed me how much faster the same tasks can be.

| GUI action | Cmdlet |
|---|---|
| Reset password | `Set-ADAccountPassword -Reset` |
| Force change at next logon | `Set-ADUser -ChangePasswordAtLogon $true` |
| Add and remove group member | `Add-ADGroupMember` and `Remove-ADGroupMember` |
| Disable account | `Disable-ADAccount` |
| Move to an OU | `Move-ADObject` |
| Set lockout policy | `Set-ADDefaultDomainPasswordPolicy` |
| Find locked accounts | `Search-ADAccount -LockedOut` |
| Unlock | `Unlock-ADAccount` |

## Redoing it with PowerShell

I learnt more about PowerShell and just how powerful scripts can be by repeating the user creation with a script that reads a CSV file:
![A screenshot of a PowerShell window showing the user creation script](screenshots/029-script-user-creation.png)

I then also had to re-add the users to their correct groups.
![Another screenshot of a script applying the correct groups to the created users](screenshots/030-script-add-user-to-dept.png)

Lastly verify that the scripts worked correctly.
![A screenshot of a PowerShell window showing verification of the created users](screenshots/031-verify-scripts.png)

Creating 14 users by GUI took around [45] minutes. The scripts took less than [10] seconds.

## Problems and fixes

| Problem                                                      | Cause                                                                                                              | Fix                                                                                                                |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| The client could not join the domain, and `ping DC01` failed | The client VM's network adapter was set to **NAT** instead of **NAT Network**, so the VMs could not see each other | Set both VMs to the NAT Network (LabNet). I was then able to join the domain.                                      |
| The "Next" button was greyed out when promoting the server   | I was in the Add Roles and Features wizard, not the promotion wizard                                               | Promoted the server from the Server Manager notification flag                                                      |
| DNS delegation warning during promotion                      | Expected for a new forest, because there is no parent zone to register with                                        | Continued. No action was required.                                                                                 |
| The client appeared as `DESKTOP-...` when joining the domain | I forgot to rename it before joining the domain                                                                    | Renamed it to CLIENT01 and removed the old DNS record                                                              |
| The client VM was very slow                                  | I didn't have space on my SSD, so I had to ran it on a hard drive                                                  | Tick host I/O cache in VM Storage settings, I disabled unneeded services and  trimmed startup apps on the client.  |

## What I learned

- How to install and configure Windows Server, promote it to a domain controller, and set its IP address and DNS
- How an Active Directory forest, domain and OUs fit together, and how to build the OU structure and create users
- How to join a client computer to the domain and why DNS has to be right first
- How to handle core helpdesk tasks: password resets, group changes, leavers and lockouts, and how to investigate lockouts in Event Viewer
- How to create, link and test Group Policy for different OUs
- How to diagnose problems methodically: identify the symptom, test a theory, fix it and verify the result

## Skills demonstrated

- Windows Server 2025 installation and configuration
- Active Directory Domain Services: forest and domain creation, OUs, users and groups
- DNS configuration and forwarders
- User lifecycle management: password resets, group changes, leavers
- Account lockout policy and troubleshooting (Event Viewer events 4740 and 4625)
- Group Policy creation, linking and testing
- PowerShell cmdlets for Active Directory administration
- VirtualBox networking and troubleshooting

## Next steps

The next lab builds on this domain by adding network file shares with group-based permissions: [Lab 04: File shares and permissions](../04-file-shares/).
