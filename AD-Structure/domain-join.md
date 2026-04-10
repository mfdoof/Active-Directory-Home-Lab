# Domain Join — Windows 11
### Overview
This document covers the process of joining a Windows 11 workstation to the DOOF.local domain.
It includes prerequisites, the join process, and post-join verification.
Naming convention follows the format DEPARTMENT-PC# (e.g. IT-PC01, HR-PC01).


## Prerequisites Checklist
Complete all items before proceeding with the domain join.

- [ ] Windows Server 2022 DC is running
- [ ] DC static IP is set to `192.168.1.201`
- [ ] `DOOF.local` DNS zone is clean — no stale records
- [ ] Windows 11 is running **Pro or Enterprise** edition (Home cannot join a domain)
- [ ] Windows 11 preferred DNS is pointing to DC (`192.168.1.201`)
- [ ] Windows 11 can ping the DC (`ping 192.168.1.201`)
- [ ] DNS resolves correctly from Windows 11 (`nslookup -type=A doof.local 192.168.1.201`)
- [ ] Computer account prestaged in correct OU (optional — see `computer-configuration.md`)


## Part 1 — Configure DNS on Windows 11

DNS must point to the DC before joining. The domain join process relies entirely on DNS
to locate the DC.

### PowerShell
```powershell
# Set preferred DNS to DC
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
    -ServerAddresses 192.168.1.201

# Verify
Get-DnsClientServerAddress -InterfaceAlias "Ethernet" -AddressFamily IPv4
# Expected: ServerAddresses = 192.168.1.201
```

### GUI
1. **Settings** → **Network & Internet** → **Ethernet** → **Edit** (next to IP assignment)
2. Set **Preferred DNS** to `192.168.1.201`
3. Click **Save**


## Part 2 — Validate DNS Resolution

Always validate DNS before joining. A failed join gives vague errors — 
confirming DNS works first eliminates the most common failure point.

```powershell
# Flush DNS cache first
ipconfig /flushdns

# Explicitly target DC — do not run without -Server parameter
# Running without -Server may return router cache instead of DC
nslookup -type=A doof.local 192.168.1.201
# Expected: returns 192.168.1.201

# Alternative
Resolve-DnsName "DOOF.local" -Type A -Server 192.168.1.201
# Expected: IPAddress = 192.168.1.201
```

> **Important:** Always use `-Server 192.168.1.201` when testing DNS. Running
> `Resolve-DnsName` or `nslookup` without explicitly targeting the DC may return
> a cached response from your router instead of the DC.


## Part 3 — Join the Domain

### PowerShell
```powershell
# Join DOOF.local domain
Add-Computer -DomainName "DOOF.local" `
    -Credential (Get-Credential) ` # Enter DOOF\Administrator when prompted
    -Restart
```

### GUI
1. **Settings** → **Accounts** → **Access work or school** → **Connect**
2. Select **Join this device to a local Active Directory domain**
3. Enter domain name: `DOOF.local` → click **Next**
4. Enter domain admin credentials when prompted → click **OK**
5. Click **Restart Now**


## Part 4 — Post-Join Verification

### On Windows 11 — confirm domain membership
```powershell
# Verify domain join
(Get-WmiObject Win32_ComputerSystem).Domain
# Expected: DOOF.local

# Alternative
systeminfo | findstr /i "domain"
# Expected: Domain: DOOF.local
```

### On the DC — confirm computer account exists in correct OU
```powershell
# Verify computer account is in AD
Get-ADComputer -Identity "IT-PC01" | 
    Select-Object Name, DistinguishedName, Enabled
# Expected: DistinguishedName shows OU=IT,OU=Computers,DC=DOOF,DC=local

# If computer landed in default Computers container, move it
# See computer-configuration.md Part 4
```

### Login verification
1. On Windows 11 login screen → click **Other user**
2. Enter domain credentials: `DOOF\username`
3. Confirm successful login as domain user


## Current Lab — Joined Computers

| Computer Name | Department | IP Address | Status |
|---|---|---|---|
| IT-PC01 | IT | 192.168.1.2 (DHCP) | Domain Joined |


## Notes
- Always boot the DC first before attempting a domain join
- DNS must resolve `DOOF.local` to `192.168.1.201` before joining — verify explicitly
- If the computer lands in the default Computers container after joining,
  refer to `computer-configuration.md Part 4` to move it to the correct OU
- Windows 11 Home cannot join a domain — Pro or Enterprise required
