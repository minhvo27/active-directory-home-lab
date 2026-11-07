# Active Directory Home Lab

I built this lab to practice the kind of work an IT help desk does every day: setting up user accounts, resetting passwords, and fixing computers that can't connect to the network. Everything runs on my own PC in VirtualBox, so I could break things and fix them without affecting anything real.

## Status

| Phase | What I did | Status |
| --- | --- | --- |
| 1 | Set up the domain controller (DC01) with Windows Server 2022, Active Directory, and DNS | Complete |
| 1 | Created departments, users, and a group with PowerShell | Complete |
| 2 | Set up the Windows 11 computer (PC01) and joined it to the domain | Complete |
| 2 | Reset a user's password and signed in with it on PC01 | Complete |
| 2 | Lock out an account and unlock it | Planned |
| 2 | Disable and re-enable an account | Planned |
| 2 | Share a folder that only the Sales group can open | Planned |
| 2 | Try out a Group Policy | Planned |
| 2 | Break the client's DNS on purpose and fix it | Planned |

## What I used

| Machine | Job | Operating system | Resources | Address |
| --- | --- | --- | --- | --- |
| DC01 | Domain controller (Active Directory and DNS) | Windows Server 2022 Standard Evaluation | 2 CPUs, 4 GB RAM, 50 GB disk | 192.168.50.10 |
| PC01 | Client computer that joins the domain | Windows 11 Enterprise Evaluation | 2 CPUs, 5 GB RAM, 80 GB disk | 192.168.50.20 |

- I ran both machines in **VirtualBox 7.2.22** on my Windows PC.
- In VirtualBox I labeled the client PC01-W11, but its computer name inside Windows is PC01.
- The two machines talk over a private virtual network I named LabNet. It isn't connected to my home network.
- Both operating systems are free trial versions from Microsoft (180 days for Server, 90 days for Windows 11), so I didn't need a product key.

## Setting up the server (DC01)

A domain controller is the server that keeps the list of every user and computer and checks passwords when someone signs in. Here is how I built mine:

1. I made a new virtual machine in VirtualBox using the Windows Server 2022 trial file. I turned off VirtualBox's automatic install so I could pick the edition and set the Administrator password myself.
2. I put its network card on the private LabNet network. VirtualBox only shows this choice in Expert mode.
3. I installed Windows Server with the Desktop Experience option, which gives you the normal Windows screen instead of only a command line.
4. I saved a snapshot called "Fresh Install" so I could go back if I messed something up.
5. I gave the server a fixed IP address (192.168.50.10) and pointed its DNS setting at itself.
6. I turned it into a domain controller with two PowerShell commands:

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "lab.local" -InstallDns
```

7. After it restarted, I signed in as LAB\Administrator and ran `Get-ADDomain` to check. It showed `lab.local`, so it worked.
8. I saved another snapshot called "AD installed".

## Departments, users, and a group

I made three folders inside the domain, called OUs (organizational units), for Sales, IT, and HR. An OU lets you manage one department without touching the others, for example by giving a policy to Sales only.

| Item | Name | Where it lives | Notes |
| --- | --- | --- | --- |
| OU | Sales | lab.local | Holds the sales users and their group |
| OU | IT | lab.local | Holds my own account, which I use as the help desk technician |
| OU | HR | lab.local | Empty for now, so I can test permissions later |
| User | jhilston | Sales | Belongs to Sales-Staff and signs in on PC01 |
| User | jpha | Sales | Belongs to Sales-Staff |
| Group | Sales-Staff | Sales | Holds the two Sales users |
| Computer | DC01 and PC01 | lab.local | The server and the client |

```powershell
New-ADOrganizationalUnit -Name "Sales" -Path "DC=lab,DC=local"
New-ADGroup -Name "Sales-Staff" -GroupScope Global -Path "OU=Sales,DC=lab,DC=local"
Add-ADGroupMember -Identity "Sales-Staff" -Members jhilston,jpha
```

In `OU=Sales,DC=lab,DC=local`, the DC means "domain component," which is just one piece of the domain name. Read it right to left: the Sales folder inside lab.local.

## Setting up the client (PC01)

1. I made a second virtual machine from the Windows 11 Enterprise trial file, with automatic install turned off.
2. Windows 11 needs at least 64 GB of disk space, so I made the disk bigger (50 GB to 80 GB) before installing.
3. I installed Windows 11 and made a local account using the "Domain join instead" option on the sign-in screen.
4. I moved it onto the LabNet network and gave it a fixed address with the server as its DNS:

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.50.20 -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.50.10
```

5. Before joining, I checked that PC01 could find the server. `Test-NetConnection` said `True` and `nslookup lab.local` gave back 192.168.50.10.
6. I joined the domain and renamed the computer in one command:

```powershell
Add-Computer -DomainName lab.local -NewName PC01 -Credential LAB\Administrator -Restart
```

7. To prove it worked, I ran `Get-ADComputer -Filter * | Select-Object Name` on DC01 and it listed DC01 and PC01. On PC01, `whoami` showed `lab\jhilston` and `hostname` showed `PC01`.

## Password reset (first help desk scenario)

I pretended a user forgot their password. From DC01 I reset it in PowerShell, then signed in as LAB\jhilston on PC01 with the new password to make sure it worked.

```powershell
Set-ADAccountPassword -Identity jhilston -Reset -NewPassword (Read-Host -AsSecureString "New password")
```

It took me a few tries, because of a typo and a password rule I didn't know about. Both are in the table below.

## Mistakes I ran into

| What went wrong | Why it happened | How I fixed it |
| --- | --- | --- |
| Only NAT and Bridged showed up for the network | VirtualBox hides the private network option in Basic mode | Switched to Expert mode, where the option appeared |
| VirtualBox asked for a domain name and password during setup | It turns on automatic install by itself once you pick an installer file | Unticked it and installed by hand |
| Ctrl+Alt+Del opened my real PC's menu | My PC grabbed the key combination before the VM could | Used Input > Keyboard > Insert Ctrl-Alt-Del in the VirtualBox menu |
| `hostname DC01` said "hostname -s is not supported" | That command only shows the name, it can't change it | Used `Rename-Computer -NewName DC01 -Restart` |
| `Install-WindowFeature` and `Set-DnsClientSeverAddress` weren't recognized | I mistyped the command names | Typed them correctly, then started copying and pasting commands |
| `-Identify` was rejected twice, once on `Add-ADGroupMember` and once on `Set-ADAccountPassword` | I typed Identify instead of Identity | Fixed the spelling and checked the result |
| Black screen after I promoted the server, and Server Manager showed no roles at first | The first startup after promotion is slow | Waited, signed in, and `Get-ADDomain` confirmed it was fine |
| PowerShell froze and I lost my history | Clicking inside the window pauses it, and restarting cleared my session | Restarted PowerShell and used `Get-ADUser` and similar commands to see what was already saved |
| The Windows 11 VM stayed black for hours and froze my PC | The automatic install never started, and the VBoxSVGA graphics controller showed a blank screen | Made the VM again with automatic install off, attached the real installer file, and switched to the VMSVGA controller |
| Windows 11 setup said it couldn't install on the drive | The disk was 50 GB and it needs at least 64 GB | Turned the VM off and made the disk 80 GB with `VBoxManage modifymedium disk --resize 81920` |
| `New-NetIPAddress` said "Access is denied" | I hadn't opened PowerShell as Administrator | Reopened it with Run as administrator |
| The password reset was rejected for not meeting the requirements | Active Directory doesn't allow a password that contains the user's account name | Picked a password without any part of the name |

## Screenshots

<img width="1288" height="936" alt="image" src="https://github.com/user-attachments/assets/f298d957-2891-42d2-88cf-400917ccafb3" />
<img width="1028" height="851" alt="image" src="https://github.com/user-attachments/assets/f97dbefc-cafd-48a3-b3c9-bf9ddf3d3726" />

<!--
![VirtualBox network set to LabNet](screenshots/01-dc01-network-labnet.png)
![Get-ADDomain output showing lab.local](screenshots/02-ad-domain-lab-local.png)
![OU structure in Active Directory Users and Computers](screenshots/03-ou-structure.png)
![Get-ADComputer listing DC01 and PC01](screenshots/04-get-adcomputer.png)
![whoami and hostname on PC01](screenshots/05-pc01-whoami-hostname.png)
-->
