# Active Directory Home Lab

A two-machine Windows Active Directory environment (`lab.local`) built in VirtualBox to practice the account, access, and troubleshooting work an IT help desk does every day: a Windows Server 2022 domain controller and a domain-joined Windows 11 client.

## Status

| Phase | Description | Status |
| --- | --- | --- |
| 1 | Domain controller (DC01): Windows Server 2022, AD DS, DNS | Complete |
| 1 | OUs, users, and a security group created with PowerShell | Complete |
| 2 | Windows 11 client (PC01) joined to `lab.local` | Complete |
| 2 | Help desk scenario: password reset, then domain sign-in from the client | Complete |
| 2 | Help desk scenarios: account lockout, disable and re-enable, folder permissions, Group Policy, DNS fault | Planned |

## Skills demonstrated

- Installing and configuring Windows Server 2022 and Windows 11 in virtual machines
- Deploying Active Directory Domain Services and DNS
- Static IP addressing and DNS configuration on an isolated virtual network
- Joining a Windows 11 client to a domain and verifying it from both machines
- Managing OUs, users, groups, and password resets through PowerShell
- Diagnosing and fixing real errors during the build (see the troubleshooting log)

## Lab environment

| Machine | Role | Operating system | Resources | IP address |
| --- | --- | --- | --- | --- |
| DC01 | Domain controller (AD DS, DNS) | Windows Server 2022 Standard Evaluation (Desktop Experience) | 2 CPUs, 4 GB RAM, 50 GB disk | 192.168.50.10 |
| PC01 | Domain-joined client | Windows 11 Enterprise Evaluation | 2 CPUs, 5 GB RAM, 80 GB disk | 192.168.50.20 |

- Hypervisor: VirtualBox 7.2.22 on a Windows PC. The client's VirtualBox label is `PC01-W11`; its Windows computer name is `PC01`.
- Network: an Internal Network named `LabNet`, subnet 192.168.50.0/24, no default gateway, so the lab is isolated from my home network.
- Licensing: free Microsoft evaluation editions (Windows Server for 180 days, Windows 11 Enterprise for 90 days).

## Build steps

### DC01 (domain controller)

1. Created the VM in VirtualBox from the Windows Server 2022 evaluation ISO, with unattended installation switched off so I could choose the edition and set the password myself.
2. Attached the network adapter to the `LabNet` internal network (this option only appears in VirtualBox's Expert settings mode).
3. Installed Windows Server 2022 Standard Evaluation (Desktop Experience) with a custom install and set the Administrator password.
4. Took a VirtualBox snapshot named "Fresh Install" so I could roll back after mistakes.
5. Set a static IPv4 address of 192.168.50.10 (mask 255.255.255.0) and pointed the server's DNS at itself.
6. Installed the AD DS role and promoted the server to a domain controller for a new forest, `lab.local`, with DNS:

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "lab.local" -InstallDns
```

7. Verified after the automatic restart by signing in as `LAB\Administrator` and running `Get-ADDomain`, which returned the DNS root `lab.local`.
8. Took a second snapshot named "AD installed".

### PC01 (Windows 11 client)

1. Created the VM from the Windows 11 Enterprise evaluation ISO with EFI, TPM 2.0, and Secure Boot, unattended installation off, and the `VMSVGA` graphics controller.
2. Enlarged the virtual disk from 50 GB to 80 GB, because Windows 11 setup requires at least 64 GB (see the troubleshooting log).
3. Installed Windows 11 Enterprise and created a local account using the "Domain join instead" sign-in option.
4. Moved the adapter to the `LabNet` internal network and set a static IP of 192.168.50.20 with DNS 192.168.50.10:

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.50.20 -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.50.10
```

5. Confirmed the client could reach the domain controller before joining:

```powershell
Test-NetConnection -ComputerName 192.168.50.10 -Port 53
nslookup lab.local
```

`TcpTestSucceeded` returned `True`, and `nslookup` resolved `lab.local` to 192.168.50.10.

6. Joined the domain and renamed the computer:

```powershell
Add-Computer -DomainName lab.local -NewName PC01 -Credential LAB\Administrator -Restart
```

7. Verification: on DC01, `Get-ADComputer -Filter * | Select-Object Name` lists **DC01** and **PC01**. On PC01, `whoami` returns `lab\jhilston` and `hostname` returns `PC01`.

## Active Directory structure

| Object | Name | Location | Notes |
| --- | --- | --- | --- |
| OU | Sales | lab.local | Holds the sales users and their group |
| OU | IT | lab.local | Holds my own account, used as the help desk technician |
| OU | HR | lab.local | Department OU for HR, available for permission tests |
| User | jhilston | Sales OU | Member of Sales-Staff; signs in on PC01 |
| User | jpha | Sales OU | Member of Sales-Staff |
| Group | Sales-Staff | Sales OU | Security group used to control access |
| Computer | DC01, PC01 | lab.local | Domain controller and domain-joined client |

An OU is a folder inside the domain, so I can apply a Group Policy or delegate permissions to one department without touching the others. The path `OU=Sales,DC=lab,DC=local` reads right to left as the Sales OU inside the lab.local domain, where `DC` means domain component.

```powershell
New-ADOrganizationalUnit -Name "Sales" -Path "DC=lab,DC=local"
New-ADGroup -Name "Sales-Staff" -GroupScope Global -Path "OU=Sales,DC=lab,DC=local"
Add-ADGroupMember -Identity "Sales-Staff" -Members jhilston,jpha
```

## Help desk scenarios

### Completed: password reset

A user needed a new password. From DC01, I reset the account in PowerShell, then confirmed the new password worked by signing in as `LAB\jhilston` on the domain-joined PC01.

```powershell
Set-ADAccountPassword -Identity jhilston -Reset -NewPassword (Read-Host -AsSecureString "New password")
```

The first attempts failed, which is worth knowing: a typo in the parameter name, then a rejection because the domain's complexity rules do not allow a password that contains the user's account name. Details are in the troubleshooting log.

### Planned

- [ ] Lock out an account with failed sign-ins, then unlock it with `Unlock-ADAccount`
- [ ] Disable and re-enable an account
- [ ] Share a folder to Sales-Staff only and test with a Sales user and a non-Sales user
- [ ] Link a Group Policy to the Sales OU and confirm it applied with `gpupdate /force` and `gpresult /r`
- [ ] Break the client's DNS on purpose and fix it with `ipconfig /all` and `nslookup`

## Troubleshooting log

| Problem | Cause | Fix |
| --- | --- | --- |
| Only NAT and Bridged adapters were offered | VirtualBox's Basic settings mode hides the Internal Network type | Switched to the Expert tab and chose Internal Network named `LabNet` |
| Unattended install screen asked for a domain name and password | VirtualBox ticks unattended installation automatically once an ISO is chosen | Unticked it and installed manually to pick the edition and set the Administrator password |
| Ctrl+Alt+Del opened my real computer's menu | The host PC captured the key combination | Used Input > Keyboard > Insert Ctrl-Alt-Del in the VirtualBox menu |
| `hostname DC01` returned "hostname -s is not supported" | `hostname` only displays the name and cannot change it | Used `Rename-Computer -NewName DC01 -Restart` instead |
| `Install-WindowFeature` and `Set-DnsClientSeverAddress` were not recognized | Typos in the cmdlet names (`Install-WindowsFeature`, `Set-DnsClientServerAddress`) | Retyped the commands correctly and confirmed the results |
| `Add-ADGroupMember` and `Set-ADAccountPassword` rejected `-Identify` | Typo: the parameter is `-Identity` | Corrected the parameter and verified the result; pasting commands avoids the slip |
| Black screen after the domain controller restart, and Server Manager first showed no roles | The first boot after promotion is slow, and Server Manager had not finished loading | Waited, signed in as `LAB\Administrator`, and confirmed the domain with `Get-ADDomain` |
| PowerShell froze and its history was lost | Clicking inside the console pauses it in selection mode, and a restart cleared my session variable | Reopened PowerShell and re-queried Active Directory to see which objects had been saved before recreating anything |
| Windows 11 client VM stayed on a black screen for hours and froze my PC | The unattended install never started, the optical drive held an empty placeholder disc, and the `VBoxSVGA` graphics controller showed a blank display | Recreated the VM with unattended install off, attached the real ISO, and switched the graphics controller to `VMSVGA`, after which setup appeared |
| Windows 11 setup said it "can't be installed on this drive" | The virtual disk was 50 GB, below the 64 GB minimum | Powered off the VM and enlarged the disk to 80 GB with `VBoxManage modifymedium disk --resize 81920` |
| `New-NetIPAddress` returned "Access is denied" | PowerShell was not running as Administrator | Reopened PowerShell with Run as administrator |
| Password reset failed with "does not meet the length, complexity, or history requirement" | Active Directory rejects passwords that contain the user's account name | Chose a password with no part of the name in it, and the reset succeeded |

## Screenshots

Add these to a `screenshots/` folder and uncomment the lines below once the images are uploaded. Crop out any typed passwords first.

<!--
![VirtualBox network set to LabNet](screenshots/01-dc01-network-labnet.png)
![Get-ADDomain output showing lab.local](screenshots/02-ad-domain-lab-local.png)
![OU structure in Active Directory Users and Computers](screenshots/03-ou-structure.png)
![Get-ADComputer listing DC01 and PC01](screenshots/04-get-adcomputer.png)
![whoami and hostname on PC01](screenshots/05-pc01-whoami-hostname.png)
-->
