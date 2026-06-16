# 🏛️ Active Directory Penetration Testing

> Personal reference notes for Active Directory attack techniques. Covers enumeration, exploitation, privilege escalation, persistence, and domain dominance. Practiced in the GOAD home lab and HackTheBox Pro Labs.
>
[![Topic: Active Directory](https://img.shields.io/badge/Topic-Active%20Directory-blue.svg)]()
[![Cert: CAPE](https://img.shields.io/badge/Cert-CAPE-red.svg)]()
[![Lab: GOAD](https://img.shields.io/badge/Lab-GOAD-orange.svg)](https://github.com/Orange-Cyberdefense/GOAD)

---

## ⚠️ Legal Notice

This material is intended **only** for authorized penetration testers and for CAPE exam
preparation in lab environments you own or are explicitly authorized to test. Using these
techniques against systems without explicit written authorization is illegal. You are
responsible for complying with all applicable laws.

---

## Coverage

```
Enumeration        →  BloodHound, PowerView, LDAP, CME, rpcclient
Kerberos Attacks   →  Kerberoasting, AS-REP Roasting, Pass-the-Ticket
Credential Attacks →  Password Spray, Pass-the-Hash, Mimikatz, DCSync
Delegation Abuse   →  Unconstrained, Constrained, RBCD
ACL Abuse          →  GenericAll, GenericWrite, WriteOwner, Shadow Credentials
Lateral Movement   →  PSExec, WMIExec, Evil-WinRM, RDP, DCOM
Domain Dominance   →  Golden Ticket, Silver Ticket, Skeleton Key, DCShadow
MSSQL Attacks      →  xp_cmdshell, linked servers, impersonation
Persistence        →  Registry, scheduled tasks, GPO, AdminSDHolder
Trust Attacks      →  Child→Parent, forest trust, SID history
```

---

## Engagement Methodology 

| Phase | Goal | Key Techniques | 
|-------|------|----------------|
| 1. Initial Foothold | Get valid domain credentials or unauthenticated access | LLMNR/NBT-NS poison, password spray, anonymous LDAP, AS-REP |
| 2. Domain Enumeration | Map the AD environment, users, groups, GPOs, trusts | BloodHound, PowerView, ldapsearch, CrackMapExec, ADSI | 
| 3. Credential Harvesting | Obtain more hashes and plaintext passwords | Kerberorasting, secretdup, LSASS dump, SAM, DPAPI |
| 4. Privilege Escalation | Move form low-priv domain user to Domain Admin | ACL abuse, delegation, GPO abouse, AS-REP, Kerberoast | 
| 5. Lateral Movement | Move across machiens to reach target systems | PTH, PTT, WinRM, SMB, RDP, DCOM, WMI | 
| 6. Domain Dominance | Own the domain: DCSync, Golden/Silver Ticker | DCSync, NTDS.dit, Golden Ticket, Skeleton Key | 
| 7. Forest Attacks | Cross trust boundaries to compromise other forests | SID history, trust ticktes, foreing group memebers | 
| 8. Persistence | Maintenance access after password changes | Golden Ticket, DC Shadow, AdminSDHolder, ACL backdoor | 


---

## Notes Index

| # | Section | Topics |
|---|---------|--------|
| 01 | [Methodology & Lab Setup](01-methodology-and-lab-setup.md) | Exam overview, engagement methodology, tooling |
| 02 | [Enumeration](02-enumeration.md) | LDAP, BloodHound, PowerView, CrackMapExec / NetExec |
| 03 | [Kerberos Attacks](03-kerberos-attacks.md) | Kerberoasting, AS-REP roasting, Pass-the-Ticket, OPtH |
| 04 | [Password Attacks & Credential Abuse](04-password-attacks-and-credential-abuse.md) | Spraying, hash dumping, DPAPI |
| 05 | [Kerberos Delegation Attacks](05-kerberos-delegation-attacks.md) | Unconstrained, constrained, RBCD |
| 06 | [ACL / DACL Abuse](06-acl-dacl-abuse.md) | Dangerous rights, enumeration, exploitation |
| 07 | [Lateral Movement Techniques](07-lateral-movement.md) | PtH, WinRM, RDP, DCOM, token impersonation |
| 08 | [Domain Privilege Escalation](08-domain-privilege-escalation.md) | GPO abuse, AdminSDHolder, Exchange, Backup Operators |
| 09 | [MSSQL Attacks & Linked Server Chains](09-mssql-attacks.md) | Enumeration, command exec, linked servers |
| 10 | [Pivoting & Tunneling](10-pivoting-and-tunneling.md) | SSH, Chisel, Ligolo-ng, proxychains |
| 11 | [Domain Persistence](11-domain-persistence.md) | Golden/Silver Ticket, DCSync, Skeleton Key, DCShadow |
| 12 | [Forest & Trust Attacks](12-forest-and-trust-attacks.md) | Trust enumeration, intra-/inter-forest attacks |
| 13 | [Defense Evasion & AV Bypass](13-defense-evasion.md) | AMSI bypass, execution bypass, EDR evasion, OPSEC |
| 14 | [CAPE Exam Tips & Cheatsheet](14-cape-exam-tips-and-cheatsheet.md) | Strategy, decision tree, one-liners, detection IDs |

---

## Attack Decision Tree (Quick Reference)

| You Find... | First Action | Tools |
|-------------|-------------|-------|
| Port 445 open | `nmap smb-vuln-*`, null session | nmap, crackmapexec |
| Valid credentials | Kerberoast, BloodHound, secretsdump | impacket, bloodhound-python |
| SeImpersonatePrivilege | JuicyPotatoNG / PrintSpoofer | JuicyPotatoNG |
| Unconstrained delegation | Wait for DA connection / coerce | Rubeus, responder |
| GenericAll on user | Shadow credentials or password reset | certipy, PowerView |
| DA credentials | DCSync all hashes, Golden Ticket | impacket, mimikatz |

---

## Key Tools

| Tool | Purpose | Install |
|------|---------|---------|
| [BloodHound](https://github.com/BloodHoundAD/BloodHound) | AD graph analysis | `sudo apt install bloodhound` |
| [Impacket](https://github.com/fortra/impacket) | AD/Windows protocols | `pip install impacket` |
| [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec) | Swiss army knife | `sudo apt install crackmapexec` |
| [Rubeus](https://github.com/GhostPack/Rubeus) | Kerberos attacks | Pre-compiled Windows binary |
| [PowerView](https://github.com/PowerShellMafia/PowerSploit) | AD enumeration | PowerShell module |
| [Certipy](https://github.com/ly4k/Certipy) | ADCS attacks | `pip install certipy-ad` |

---

> ⚠️ See [DISCLAIMER.md](DISCLAIMER.md) — for authorized lab and engagement use only.
