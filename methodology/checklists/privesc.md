# Privilege Escalation Checklist

## Windows Privesc

### Quick Wins
- [ ] `whoami /priv` → SeImpersonate? SeBackup? SeAssignPrimaryToken?
- [ ] `whoami /groups` → Backup Operators? Server Operators? DnsAdmins?
- [ ] `cmdkey /list` → Saved credentials? → `runas /savecred`
- [ ] `type C:\Users\*\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`

### Automated Enum
- [ ] Run `winPEAS.exe`
- [ ] Run `PowerUp.ps1`: `Invoke-AllChecks`
- [ ] Run `Seatbelt.exe -group=all`

### Services
- [ ] Unquoted service paths: `wmic service get name,pathname,startmode | findstr /i /v "C:\Windows"`
- [ ] Writable service binaries: check `icacls` on each path
- [ ] Service DLL hijacking: check with procmon
- [ ] Modifiable service config: `accesschk.exe -uwcqv "Everyone" *`

### Registry
- [ ] AlwaysInstallElevated: check both HKLM and HKCU
- [ ] AutoRun programs with writable paths
- [ ] `reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer`

### Scheduled Tasks
- [ ] `schtasks /query /fo LIST /v`
- [ ] Check for writable scripts called by tasks

### Files & Creds
- [ ] `dir /s /b C:\Users\*.txt C:\Users\*.ini C:\Users\*.cfg C:\Users\*.xml 2>nul`
- [ ] `type C:\Windows\Panther\Unattend.xml`
- [ ] `type C:\inetpub\wwwroot\web.config`
- [ ] Check for KeePass databases: `dir /s /b C:\*.kdbx 2>nul`
- [ ] Check browser stored passwords
- [ ] FileZilla/WinSCP saved sessions

### Token / Potato
- [ ] SeImpersonate → GodPotato, PrintSpoofer, SweetPotato
- [ ] Check for other user tokens: `incognito list_tokens -u`

---

## Linux Privesc

### Quick Wins
- [ ] `sudo -l` → Check GTFOBins for each binary
- [ ] `find / -perm -4000 -type f 2>/dev/null` → SUID binaries
- [ ] `cat /etc/crontab && ls -la /etc/cron.*` → writable cron scripts?
- [ ] `ls -la /etc/passwd` → writable? Add root user
- [ ] `cat /home/*/.bash_history` → creds in history?

### Automated Enum
- [ ] Run `linpeas.sh`
- [ ] Run `linux-smart-enumeration (lse.sh)`
- [ ] Run `pspy` to monitor processes

### Permissions
- [ ] SUID binaries → GTFOBins
- [ ] Capabilities: `getcap -r / 2>/dev/null`
- [ ] Writable `/etc/passwd` or `/etc/shadow`
- [ ] Writable systemd services
- [ ] Writable scripts in PATH

### Cron & Processes
- [ ] Cron jobs running as root with writable scripts
- [ ] Tar wildcard injection
- [ ] PATH hijacking on cron scripts
- [ ] Monitor with `pspy64` for hidden cron/processes

### Sudo
- [ ] `sudo -l` → specific binary abuse
- [ ] `env_keep+=LD_PRELOAD` → LD_PRELOAD hijack
- [ ] Sudo version < 1.8.28 → CVE-2019-14287 (`sudo -u#-1 /bin/bash`)

### Group Memberships
- [ ] `id` → docker? lxd? disk? adm?
- [ ] Docker group → mount host FS
- [ ] lxd/lxc group → privileged container
- [ ] Disk group → debugfs
- [ ] adm group → read logs for creds

### Network & Services
- [ ] Internal services: `ss -tlnp` or `netstat -tlnp`
- [ ] MySQL running as root? UDF exploit
- [ ] NFS: `cat /etc/exports` → no_root_squash?

### Kernel
- [ ] `uname -a` → check kernel version
- [ ] DirtyCow, DirtyPipe, PwnKit
- [ ] `linux-exploit-suggester.sh`

### Files & Creds
- [ ] SSH keys: `find / -name id_rsa 2>/dev/null`
- [ ] Config files: `.env`, `wp-config.php`, `web.config`
- [ ] `.git` repos: `find / -name .git 2>/dev/null`
- [ ] Backup files: `find / -name "*.bak" -o -name "*.old" 2>/dev/null`
