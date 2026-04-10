# Network Configuration — DNS Setup

## Overview
Documents the DNS configuration for DC01, including static IP assignment, DNS server settings, forwarder configuration, and troubleshooting steps encountered during setup. This document covers both high-level architecture decisions and step-by-step configuration.

## DNS Architecture

```
Client Devices
     ↓
DC01 — authoritative for DOOF.local
     ↓
Cloudflare (1.1.1.1 / 1.0.0.1)
```

- All domain-joined clients point to DC01 for DNS
- DC01 resolves `DOOF.local` internally
- DC01 forwards all external queries to Cloudflare
- Router is excluded from the DNS chain

**Why this design:**
- Clients get a single DNS source of truth for both internal and external resolution
- Cloudflare provides privacy, reliability, and faster response times over ISP or router DNS
- Keeping the router out of the DNS chain removes an uncontrolled variable

## Part 1 — Static IP Configuration

### Overview
DC01 must have a stable static IP outside the router's DHCP pool. A DC with a dynamic or conflicting IP will cause DNS, Kerberos, and AD replication failures.

### Step-by-Step — PowerShell

```powershell
# Step 1 — Identify your NIC name and current IP
Get-NetAdapter
Get-NetIPAddress -AddressFamily IPv4 | Where-Object { $_.PrefixOrigin -eq "Manual" }

# Step 2 — Remove existing IP configuration
Remove-NetIPAddress -InterfaceAlias "Ethernet" -AddressFamily IPv4 -Confirm:$false

# Step 3 — Assign new static IP outside DHCP pool
New-NetIPAddress -InterfaceAlias "Ethernet" `
    -AddressFamily IPv4 `
    -IPAddress "<your-static-ip>" `
    -PrefixLength 24 `
    -DefaultGateway "<your-gateway>"

# Step 4 — Verify AddressState is Preferred, not Duplicate
Get-NetIPAddress -AddressFamily IPv4 | 
    Where-Object { $_.PrefixOrigin -eq "Manual" } | 
    Select-Object InterfaceAlias, IPAddress, AddressState
```

### Step-by-Step — GUI

1. Open `ncpa.cpl`
2. Right-click your NIC → Properties
3. Select **Internet Protocol Version 4 (TCP/IPv4)** → Properties
4. Select **Use the following IP address**
5. Enter your static IP, subnet mask, and default gateway
6. Click OK

### Verification

```powershell
# AddressState must return Preferred
Get-NetIPAddress -AddressFamily IPv4 | 
    Where-Object { $_.PrefixOrigin -eq "Manual" } | 
    Select-Object InterfaceAlias, IPAddress, AddressState
```

## Part 2 — NIC DNS Settings

### Overview
DC01's NIC preferred DNS must point to its own static IP — not `127.0.0.1`. The secondary DNS points to Cloudflare as fallback for the DC's own queries.

**Why not loopback (`127.0.0.1`):** If the DNS Server service stops, `127.0.0.1` fails silently and failover to the secondary is unreliable. Using the DC's own static IP makes failures visible and failover clean. This is explicitly recommended in Microsoft's AD deployment documentation.

### Step-by-Step — PowerShell

```powershell
# Step 1 — Set preferred DNS to DC's own static IP, secondary to Cloudflare
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
    -ServerAddresses ("<your-static-ip>", "1.1.1.1")

# Step 2 — Verify
Get-DnsClientServerAddress -AddressFamily IPv4 | 
    Select-Object InterfaceAlias, ServerAddresses
```

### Step-by-Step — GUI

1. Open `ncpa.cpl`
2. Right-click your NIC → Properties
3. Select **Internet Protocol Version 4 (TCP/IPv4)** → Properties
4. Enter DC's static IP in **Preferred DNS server**
5. Enter `1.1.1.1` in **Alternate DNS server**
6. Click OK


## Part 3 — DNS Server Forwarder Configuration

### Overview
A forwarder tells the DNS Server service where to send queries it cannot resolve locally. DC01 is authoritative for `DOOF.local` — anything outside that zone gets forwarded to Cloudflare.

**Important distinction:**

| Setting | Tool | Affects |
|---|---|---|
| Forwarder | `Set-DnsServerForwarder` / DNS Manager | All clients querying DC01 for external names |
| NIC DNS | `Set-DnsClientServerAddress` / ncpa.cpl | Only DC01's own DNS queries |

These are two separate settings. Both must be configured correctly.

**Why Cloudflare over router:**
- ISP DNS logs and may sell query data
- Router DNS is unencrypted and visible on the wire
- Cloudflare has a formal no-logging policy, audited annually
- Significantly more reliable infrastructure than a home router

> **Note:** The router was originally auto-configured as the forwarder by the AD DS promotion wizard at time of domain promotion. It inherited whatever DNS the NIC was pointing to at promotion time. Always audit forwarder config post-promotion.

### Step-by-Step — PowerShell

```powershell
# Step 1 — Set Cloudflare as forwarder
Set-DnsServerForwarder -IPAddress "1.1.1.1","1.0.0.1"

# Step 2 — Verify
Get-DnsServerForwarder
```

### Step-by-Step — GUI

1. Open `dnsmgmt.msc`
2. Right-click your server name → Properties
3. Select the **Forwarders** tab
4. Click **Edit**
5. Remove existing forwarder entries
6. Add `1.1.1.1` and `1.0.0.1`
7. Click OK


## Part 4 — Root Hints

### Overview
Root hints are a hardcoded list of the 13 internet root DNS servers maintained by IANA. When configured alongside a forwarder, they create ambiguity — both serve the same purpose of resolving external names. Having both causes `dcdiag /test:dns` Forw failures.

**Root hints vs Forwarder:**
| | Root Hints | Forwarder |
|---|---|---|
| How it works | DC queries root servers directly, walks DNS hierarchy | DC hands query to upstream resolver |
| Use case | ISP-level or internet-facing authoritative servers | Internal DCs with an upstream resolver |
| Home lab | Not appropriate | Correct choice |

### Current State
Root hints were disabled rather than removed as a precaution. Disabling prevents DNS resolution from falling back to root hints while keeping them intact if the forwarder configuration needs to be rolled back. Removing root hints entirely is the cleaner production approach but carries more risk in a lab environment where configuration changes are ongoing.

> **Note:** Microsoft does not officially support removing all root hints from a DNS server. Root hints removed via DNS Manager may reappear after ~15 minutes as they persist in `cache.dns` and Active Directory. Disabling fallback via the Forwarders tab is the safer and more reliable approach.

### Configuration — GUI
1. Open **DNS Manager** → `dnsmgmt.msc`
2. Right-click your DNS server → **Properties**
3. Select the **Forwarders** tab
4. Uncheck **"Use root hints if no forwarders are available"**
5. Click **OK**

### Configuration — PowerShell
```powershell
# Step 1 — Check current forwarder and root hint fallback setting
Get-DnsServerForwarder

# Step 2 — Disable root hints fallback
Set-DnsServerForwarder -UseRootHint $false

# Step 3 — Verify
Get-DnsServerForwarder
# Expected: UseRootHint = False
\```
```

## Part 5 — Verification

### Full DNS Health Check

```powershell
# Run full DNS diagnostic
dcdiag /test:dns /v

# Internal resolution
Resolve-DnsName "DOOF.local"

# External resolution via forwarder
Resolve-DnsName "google.com"

# Check DNS Server service is running
Get-Service DNS | Select-Object Name, Status, StartType

# Confirm forwarder
Get-DnsServerForwarder

# Confirm NIC DNS settings
Get-DnsClientServerAddress -AddressFamily IPv4 | 
    Select-Object InterfaceAlias, ServerAddresses

# Confirm all A records in zone — check for stale entries
Get-DnsServerResourceRecord -ZoneName "DOOF.local" -RRType A | 
    Select-Object HostName, RecordType, RecordData
```

### Expected dcdiag Results

All tests should pass after root hints removal:

| Test | Expected | Notes |
|---|---|---|
| Auth | PASS | DC is authoritative for DOOF.local |
| Basc | PASS | Basic DNS connectivity |
| Forw | PASS | Forwarder responding correctly — requires root hints removed |
| Del | PASS | Delegation records valid |
| Dyn | PASS | Dynamic DNS updates working |
| RReg | PASS | DC successfully registered its DNS records |


## Troubleshooting — Issues Encountered

### Issue 1 — IP Address Conflict (AddressState: Duplicate)

**Symptom:** Static IP showed `AddressState: Duplicate`. DC fell back to APIPA (`169.254.x.x`) and dropped off the network. DNS resolution for `DOOF.local` timed out.

**Root cause:** Static IP was inside the router's DHCP pool. Another device was assigned the same IP via DHCP.

**Diagnosis:**

```powershell
# Confirm duplicate state
Get-NetIPAddress -AddressFamily IPv4 | 
    Where-Object { $_.PrefixOrigin -eq "Manual" } | 
    Select-Object InterfaceAlias, IPAddress, AddressState

# Sweep subnet to identify conflicting device
1..254 | ForEach-Object { ping -n 1 -w 50 "192.168.1.$_" | Out-Null }
arp -a
```

**Resolution:**
- Audited router DHCP client table
- Shrunk DHCP pool ceiling to free up upper IP range for static assignments
- Reassigned DC01 to static IP in the freed range
- Verified `AddressState: Preferred`


### Issue 2 — Stale DNS A Records After IP Change

**Symptom:** After IP change, old IP persisted as A records in `DOOF.local`. `Resolve-DnsName "DOOF.local"` timed out. `dcdiag` failing Basc, Forw, RReg.

**Root cause:** AD DNS does not automatically remove old A records when a DC's IP changes. Two stale records were found — one under the DC01 hostname, one at the zone apex (`@`).

**Diagnosis:**

```powershell
Get-DnsServerResourceRecord -ZoneName "DOOF.local" -RRType A | 
    Select-Object HostName, RecordType, RecordData
```

**Resolution:**

```powershell
# Remove stale record at zone apex
Remove-DnsServerResourceRecord -ZoneName "DOOF.local" `
    -RRType A -Name "@" `
    -RecordData "<old-ip>" -Force

# Remove stale record under DC hostname
Remove-DnsServerResourceRecord -ZoneName "DOOF.local" `
    -RRType A -Name "DC01" `
    -RecordData "<old-ip>" -Force

# Force re-registration of correct records
ipconfig /registerdns
```


### Issue 3 — dcdiag Forw Failure

**Symptom:** `dcdiag /test:dns` failing Forw despite external DNS resolving correctly.

**Root cause:** Root hints and forwarder configured simultaneously. dcdiag flags the ambiguity even when resolution works in practice.

**Resolution:** Remove root hints — see Part 4 above.


### Issue 4 — AD DS Promotion Auto-Configured Router as Forwarder

**Symptom:** After promotion, DNS forwarder was pointing to router instead of intended upstream resolver.

**Root cause:** AD DS promotion wizard automatically sets the forwarder to whatever DNS server the NIC was pointing to at time of promotion.

**Resolution:** Always audit `Get-DnsServerForwarder` immediately after AD DS promotion and replace with intended forwarder.


## Final State

| Setting | Value |
|---|---|
| DC01 Static IP | See network config |
| AddressState | Preferred |
| Preferred DNS (NIC) | DC01 own static IP |
| Alternate DNS (NIC) | Cloudflare 1.1.1.1 |
| DNS Forwarder | Cloudflare 1.1.1.1 / 1.0.0.1 |
| Root Hints | Disabled |
| DOOF.local resolution | Working |



