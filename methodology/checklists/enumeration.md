# Enumeration Checklist

## Initial Nmap
- [ ] `nmap -sC -sV -oN nmap/initial TARGET`
- [ ] `nmap -p- --min-rate 5000 -oN nmap/allports TARGET`
- [ ] `nmap -sU --top-ports 200 -oN nmap/udp TARGET`
- [ ] Rescan interesting ports with `-sC -sV` for version/scripts

## DNS (53)
- [ ] `dig axfr @TARGET DOMAIN`
- [ ] `dnsrecon -d DOMAIN -n TARGET`
- [ ] `nslookup -type=any DOMAIN TARGET`

## HTTP/HTTPS (80/443/8080)
- [ ] Browse manually — check source, comments, links
- [ ] `whatweb http://TARGET`
- [ ] `gobuster dir -u http://TARGET -w /usr/share/wordlists/dirmedium.txt -x php,asp,aspx,txt,html`
- [ ] `feroxbuster -u http://TARGET -w /usr/share/wordlists/dirmedium.txt`
- [ ] `nikto -h http://TARGET`
- [ ] Check `/robots.txt`, `/sitemap.xml`, `/.git/`, `/.env`
- [ ] Check for virtual hosts: `gobuster vhost -u http://TARGET -w subdomains.txt`
- [ ] Add hostnames to `/etc/hosts`
- [ ] Check for default creds on login pages
- [ ] Inspect cookies, headers, hidden form fields
- [ ] Test for SQLi, LFI, XSS on all parameters
- [ ] Check CMS: WordPress (`wpscan`), Joomla, Drupal

## SMB (139/445)
- [ ] `nxc smb TARGET -u '' -p ''` (null session)
- [ ] `nxc smb TARGET -u 'guest' -p ''` (guest access)
- [ ] `smbclient -L //TARGET/ -N`
- [ ] `nxc smb TARGET -u USER -p PASS --shares`
- [ ] `smbclient //TARGET/SHARE -U USER`
- [ ] Recursively download interesting shares

## LDAP (389/636)
- [ ] `ldapsearch -x -H ldap://TARGET -b "DC=domain,DC=com"`
- [ ] `nxc ldap TARGET -u '' -p '' --users`
- [ ] `nxc ldap TARGET -u USER -p PASS --users`
- [ ] `windapsearch -d DOMAIN --dc TARGET -U`

## Kerberos (88)
- [ ] `kerbrute userenum --dc TARGET -d DOMAIN userlist.txt`
- [ ] `impacket-GetNPUsers 'DOMAIN/' -dc-ip TARGET -usersfile users.txt -no-pass` (ASREP)
- [ ] `impacket-GetUserSPNs 'DOMAIN/USER:PASS' -dc-ip TARGET -request` (Kerberoast)

## FTP (21)
- [ ] `ftp TARGET` — try anonymous login
- [ ] Check for writable dirs, interesting files
- [ ] Check version for known exploits

## SSH (22)
- [ ] Note version (useful for OS fingerprinting)
- [ ] Try found creds
- [ ] Check for key-based auth files on target later

## SNMP (161/udp)
- [ ] `snmpwalk -v2c -c public TARGET .1`
- [ ] `onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt TARGET`
- [ ] Extract users, software, processes, network info

## RPC (135/111)
- [ ] `rpcclient -U '' -N TARGET` → `enumdomusers`, `enumdomgroups`
- [ ] `enum4linux-ng TARGET`

## MSSQL (1433)
- [ ] `impacket-mssqlclient USER@TARGET -windows-auth`
- [ ] `nxc mssql TARGET -u USER -p PASS`
- [ ] Check `xp_cmdshell`, `enum_impersonate`, `enum_db`

## MySQL (3306)
- [ ] `mysql -h TARGET -u root -p`
- [ ] Check for UDF, file read/write privs

## RDP (3389)
- [ ] `xfreerdp /v:TARGET /u:USER /p:PASS /dynamic-resolution`
- [ ] Check with found creds

## WinRM (5985/5986)
- [ ] `evil-winrm -i TARGET -u USER -p PASS`
- [ ] `evil-winrm -i TARGET -u USER -H NTHASH`

## User Enumeration
- [ ] Collect all usernames found
- [ ] Try `username:username` password spray
- [ ] Try common passwords: `Password1`, `Welcome1`, `Season+Year`
- [ ] Check for password policies first: `nxc smb TARGET -u USER -p PASS --pass-pol`
