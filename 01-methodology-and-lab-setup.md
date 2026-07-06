# 01 — Methodology & Lab Setup

[← Back to index](README.md)

## 1.1 CAPE Exam Overview

The HackTheBox CAPE (Certified Active Directory Pentesting Expert) exam tests your ability
to compromise a multi-machine Active Directory environment from an initial foothold to full
Domain/Forest compromise. Key areas:

- AD enumeration with multiple tools (BloodHound, PowerView, ldapsearch, CME)
- Kerberos attack chains: Kerberoasting, AS-REP roasting, delegation, S4U2*, pass-the-ticket
- ACL/DACL abuse for privilege escalation within the domain
- Lateral movement: pass-the-hash, over-pass-the-hash, RDP, WinRM, PSExec
- Cross-domain and cross-forest attacks via trust relationships
- Domain persistence: Golden Ticket, Silver Ticket, DCSync, skeleton key
- MSSQL linked server chains and `xp_cmdshell` abuse

## 1.2 Engagement Methodology

| Phase | Goal | Key Techniques |
|-------|------|----------------|
| 1. Initial Foothold | Get valid domain credentials or unauthenticated access | LLMNR/NBT-NS poison, password spray, anonymous LDAP, AS-REP |
| 2. Domain Enumeration | Map the AD environment, users, groups, GPOs, trusts | BloodHound, PowerView, ldapsearch, CrackMapExec, ADSI |
| 3. Credential Harvesting | Obtain more hashes and plaintext passwords | Kerberoasting, secretsdump, LSASS dump, SAM, DPAPI |
| 4. Privilege Escalation | Move from low-priv domain user to Domain Admin | ACL abuse, delegation, GPO abuse, AS-REP, Kerberoast |
| 5. Lateral Movement | Move across machines to reach target systems | PtH, PtT, WinRM, SMB, RDP, DCOM, WMI |
| 6. Domain Dominance | Own the domain: DCSync, Golden/Silver Ticket | DCSync, NTDS.dit, Golden Ticket, Skeleton Key |
| 7. Forest Attacks | Cross trust boundaries to compromise other forests | SID history, trust tickets, foreign group members |
| 8. Persistence | Maintain access after password changes | Golden Ticket, DCShadow, AdminSDHolder, ACL backdoor |

## 1.3 Lab Setup & Tool Installation

### Attacker Machine (Kali/Parrot)

```bash
# Essential AD pentest tools
sudo apt update && sudo apt install -y \
    bloodhound neo4j impacket-scripts crackmapexec \
    evil-winrm ldap-utils smbclient enum4linux-ng \
    responder certipy-ad coercer netexec

# Python-based tools
pip3 install impacket bloodhound --break-system-packages
pip3 install certipy-ad --break-system-packages
```

```bash
# PowerShell tools (for Windows or pwsh on Linux)
# Download PowerView.ps1
wget https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Recon/PowerView.ps1

# Download PowerView Dev (more features)
wget https://raw.githubusercontent.com/lucky-luk3/ActiveDirectory/master/PowerViewDev.ps1
```

```bash
# BloodHound setup
sudo neo4j start
# Default creds: neo4j/neo4j (change on first login)
bloodhound &
```

```bash
# Kerbrute — fast Kerberos user enumeration and brute-force
wget https://github.com/ropnop/kerbrute/releases/latest/download/kerbrute_linux_amd64
chmod +x kerbrute_linux_amd64 && sudo mv kerbrute_linux_amd64 /usr/local/bin/kerbrute

# Rubeus (Windows — compile or use pre-built)
# https://github.com/GhostPack/Rubeus

# NetExec (successor to CrackMapExec)
pip3 install netexec --break-system-packages
```

### Target Environment Variables (Set These First!)

```bash
# Set these variables before starting — saves time [Example]
export DC_IP=10.10.10.100
export DC_HOST=DC01.corp.local
export DOMAIN=corp.local
export DOMAIN_SHORT=CORP
export USER=john.doe
export PASS='Password123!'
export HASH='aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0'
export LHOST=10.10.14.50   # Your attacker IP

# CrackMapExec / NetExec aliases for speed
alias cme='crackmapexec'
alias nxc='netexec'

# Quick Kerberos ticket setup
export KRB5CCNAME=/tmp/krb5cc_ticket.ccache
```

## 1.4 Main ports focused on 

| Ports | Services | 
|-------|----------|
| 53 | DNS |
| 80/8530 | HTTP | 
| 88 | Kerberos |
| 139/445 | SMB |
| 1433 | MS-SQL |
| 389/636 | LDAP/LDAPS |
| 464 | kpasswd |
| 3389 | RDP | 
| 3268/3269 | Global Catalog |
| 5985 | WinRM | 


---

[← Back to index](README.md) | [Next: 02 — Enumeration →](02-enumeration.md)
