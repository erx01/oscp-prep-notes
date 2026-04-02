# Initial Access Checklist

## Web Application Attacks
- [ ] Default credentials on login pages
- [ ] SQL injection on all input fields and parameters
- [ ] LFI/RFI on file inclusion parameters
- [ ] File upload → webshell
- [ ] Command injection in input fields
- [ ] Deserialization (Java, PHP, .NET)
- [ ] SSRF on URL parameters
- [ ] XXE on XML endpoints
- [ ] WordPress: `wpscan` → plugin/theme exploits, xmlrpc brute
- [ ] CMS admin panel → code execution (theme/plugin editor)

## Service Exploits
- [ ] Check all service versions against `searchsploit`
- [ ] Check ExploitDB, Google "service version exploit"
- [ ] FTP anonymous upload → web root overlap?
- [ ] SMB writable share → web root overlap?
- [ ] SNMP community string → extract creds/info
- [ ] NFS exports → sensitive file access

## Credential-Based Access
- [ ] Spray found creds against all services (SMB, WinRM, RDP, SSH, MSSQL)
- [ ] Try `username:username` combinations
- [ ] Try common passwords: `Password1`, `Welcome1`, `Company+Year`
- [ ] Hydra brute force: `hydra -l USER -P /usr/share/wordlists/rockyou.txt TARGET ssh`
- [ ] Check for password reuse across services

## Network Attacks
- [ ] LLMNR/NBT-NS poisoning: `responder -I eth0`
- [ ] NTLM relay: `impacket-ntlmrelayx -tf targets.txt -smb2support`
- [ ] ARP spoofing (if on same network)
- [ ] Check for cleartext protocols (FTP, Telnet, HTTP)

## Client-Side (if in scope)
- [ ] Phishing with Office macros
- [ ] HTA file delivery
- [ ] LNK file with UNC path (hash capture)

## Post-Discovery
- [ ] Found creds? → Test everywhere immediately
- [ ] Found hash? → Pass-the-Hash or crack with hashcat
- [ ] Found SSH key? → Try on all Linux hosts
- [ ] Found webshell/shell? → Stabilize and start privesc
