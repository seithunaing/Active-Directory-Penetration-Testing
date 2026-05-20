# 🏛️ Active Directory Penetration Testing

> Personal reference notes for Active Directory attack techniques. Covers enumeration, exploitation, privilege escalation, persistence, and domain dominance. Practiced in the GOAD home lab and HackTheBox Pro Labs.
>
[![Topic: Active Directory](https://img.shields.io/badge/Topic-Active%20Directory-blue.svg)]()
[![Cert: CAPE](https://img.shields.io/badge/Cert-CAPE-red.svg)]()
[![Lab: GOAD](https://img.shields.io/badge/Lab-GOAD-orange.svg)](https://github.com/Orange-Cyberdefense/GOAD)

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

## Notes Index

| Topic | File | Summary |
|-------|------|---------|
| Enumeration | [enumeration/bloodhound.md](enumeration/bloodhound.md) | BloodHound setup, collection, key queries |
| Enumeration | [enumeration/powerview.md](enumeration/powerview.md) | Full PowerView command reference |
| Enumeration | [enumeration/ldap-smb.md](enumeration/ldap-smb.md) | LDAP, SMB null session, CME, enum4linux |
| Exploitation | [exploitation/kerberoasting.md](exploitation/kerberoasting.md) | Kerberoast from Linux + Windows, cracking |
| Exploitation | [exploitation/asrep-roasting.md](exploitation/asrep-roasting.md) | AS-REP roast without credentials |
| Exploitation | [exploitation/password-spray.md](exploitation/password-spray.md) | CME spray, Kerbrute, lockout awareness |
| Exploitation | [exploitation/pass-the-hash.md](exploitation/pass-the-hash.md) | PTH with impacket, CME, evil-winrm |
| Exploitation | [exploitation/delegation.md](exploitation/delegation.md) | Unconstrained, constrained, RBCD |
| Exploitation | [exploitation/acl-abuse.md](exploitation/acl-abuse.md) | ACL enumeration and abuse chains |
| Post-Exploitation | [post-exploitation/mimikatz.md](post-exploitation/mimikatz.md) | Full Mimikatz command reference |
| Post-Exploitation | [post-exploitation/dcsync.md](post-exploitation/dcsync.md) | DCSync, NTDS.dit extraction, offline dump |
| Post-Exploitation | [post-exploitation/lateral-movement.md](post-exploitation/lateral-movement.md) | PSExec, WMI, WinRM, RDP |
| Post-Exploitation | [post-exploitation/golden-silver-ticket.md](post-exploitation/golden-silver-ticket.md) | Ticket attacks and persistence |
| Persistence | [persistence/persistence-techniques.md](persistence/persistence-techniques.md) | Registry, tasks, GPO, AdminSDHolder |
| Cheatsheets | [cheatsheets/impacket.md](cheatsheets/impacket.md) | Complete impacket suite reference |
| Cheatsheets | [cheatsheets/crackmapexec.md](cheatsheets/crackmapexec.md) | Full CME command reference |
| Cheatsheets | [cheatsheets/attack-decision-tree.md](cheatsheets/attack-decision-tree.md) | What to do when you find X |

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
