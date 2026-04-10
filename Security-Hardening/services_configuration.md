# Services
### Overview
This document covers the services configured on the Windows Server 2022 Domain Controller as part of the security hardening process. It includes services that were disabled to reduce the attack surface, services that were temporarily re-enabled due to lab requirements, and critical services that must remain running for Active Directory to function correctly.

### Why Disable Unnecessary Services
Every running service on a server is a potential attack surface. Services that are not needed on a Domain Controller should be disabled because:
- They consume system resources unnecessarily
- They may contain unpatched vulnerabilities
- They increase the number of potential entry points for attackers
- They violate the principle of least functionality — a server should only run what it needs

### How to Disable a Service

**Via Services GUI**  
Start Menu -> search Services
-> Right-click the service -> Properties
-> Change Startup Type to Disabled
-> Click Stop if the service is running
-> Click Apply -> OK

**Via PowerShell**  
```powershell
Stop-Service -Name SERVICE_NAME -Force
Set-Service -Name SERVICE_NAME -StartupType Disabled
```

### Disabled Services

| Service | Service Name | Reason Disabled |
|---|---|---|
| Print Spooler | Spooler | Critical — vulnerable to PrintNightmare (CVE-2021-1675) |
| Remote Registry | RemoteRegistry | Allows remote editing of Windows registry, major attack surface |
| Windows Error Reporting | WerSvc | Sends diagnostic data to Microsoft, unnecessary on a Domain Controller |
| Bluetooth Support Service | bthserv | Hardware service with no purpose on a server |
| DFS Namespace | Dfs | Not needed in a single server lab environment |

### Verification

After disabling, all services were verified via PowerShell:
```powershell
Get-Service Spooler, RemoteRegistry, WerSvc, bthserv, Dfs | Select-Object Name, Status, StartType
```

## NetBIOS over TCP/IP

### Overview
NetBIOS over TCP/IP was temporarily re-enabled on DC01 after disabling it caused DNS resolution failures and prevented Windows 11 from joining the domain. This section documents the configuration, the conflict encountered, and the security reasoning for eventual re-disabling.

### Why Disable NetBIOS
- NetBIOS name resolution is a legacy protocol not required by modern Active Directory
- Active Directory uses DNS and Kerberos for name resolution and authentication
- NetBIOS is an active attack surface for LLMNR/NBT-NS poisoning attacks (e.g. Responder)
- Disabling at the NIC level eliminates the binding entirely — firewall rules alone do not

### Why It Was Re-enabled
Disabling NetBIOS at the NIC level blocked DNS traffic on the Domain Controller, which caused:
- `nslookup` queries from Windows 11 to time out
- Windows 11 failing to join the domain

**Root cause:** Block rules on the DC were overriding allow rules for DNS (UDP/TCP 53). NetBIOS being disabled at the NIC level compounded this by removing bindings that AD relies on during the domain join process. Re-enabling NetBIOS restored DNS resolution and allowed the domain join to succeed.

**Current state:** NetBIOS is temporarily re-enabled. Proper resolution requires configuring firewall rules so block rules do not override DNS allow rules before re-disabling NetBIOS.

### Configuration — Disable NetBIOS (PowerShell)
```powershell
# Step 1 — Check current NetBIOS setting per adapter
# TcpipNetbiosOptions values: 0 = Default, 1 = Enabled, 2 = Disabled
$nic = Get-WmiObject Win32_NetworkAdapterConfiguration | Where-Object { $_.IPEnabled -eq $true }
$nic | Select-Object Description, TcpipNetbiosOptions
```

```powershell
# Step 2 — Disable NetBIOS over TCP/IP on all IP-enabled adapters
$adapters = Get-WmiObject Win32_NetworkAdapterConfiguration | Where-Object { $_.IPEnabled -eq $true }
foreach ($adapter in $adapters) {
    $adapter.SetTcpipNetbios(2)  # 2 = Disable NetBIOS over TCP/IP
}
```
```powershell
# Step 3 — Verify
Get-WmiObject Win32_NetworkAdapterConfiguration |
    Where-Object { $_.IPEnabled -eq $true } |
    Select-Object Description, TcpipNetbiosOptions
# Expected: TcpipNetbiosOptions = 2 on all adapters
```
### Configuration — Re-enable NetBIOS (PowerShell)

```powershell
# Re-enable NetBIOS over TCP/IP on all IP-enabled adapters
$adapters = Get-WmiObject Win32_NetworkAdapterConfiguration | Where-Object { $_.IPEnabled -eq $true }
foreach ($adapter in $adapters) {
    $adapter.SetTcpipNetbios(1)  # 1 = Enable NetBIOS over TCP/IP
}
```
```powershell
# Verify
Get-WmiObject Win32_NetworkAdapterConfiguration |
    Where-Object { $_.IPEnabled -eq $true } |
    Select-Object Description, TcpipNetbiosOptions
# Expected: TcpipNetbiosOptions = 1 on all adapters
```

### Configuration — GUI

1. Open `ncpa.cpl`
2. Right-click your NIC → Properties
3. Select **Internet Protocol Version 4 (TCP/IPv4)** → Properties
4. Click **Advanced** → select the **WINS** tab
5. Select **Enable** or **Disable NetBIOS over TCP/IP** accordingly
6. Click OK

### Verification
```powershell
# Confirm registry reflects current state
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\NetBT\Parameters\Interfaces\*" |
    Select-Object PSChildName, NetbiosOptions
# Expected: 2 = Disabled, 1 = Enabled
```

> **Note:** Do not test NetBIOS resolution from the DC itself — test from the Windows 11 client or another machine on the network. Testing from the DC can return false positives via loopback.

**Reminder:** NetBIOS is temporarily re-enabled. Revisit this after firewall rules are properly configured to ensure DNS allow rules are not overridden by block rules.

## Verification — All Disabled Services

```powershell
# Verify all disabled services
Get-Service Spooler, RemoteRegistry, WerSvc, bthserv, Dfs, WinRM |
    Select-Object Name, Status, StartType

# Verify NetBIOS disabled at NIC level
Get-WmiObject Win32_NetworkAdapterConfiguration |
    Where-Object { $_.IPEnabled -eq $true } |
    Select-Object Description, TcpipNetbiosOptions

```
---

### Services Left Running

The following services may appear unnecessary but are critical for Active Directory and should not be disabled:

| Service | Reason to Keep |
|---|---|
| Windows Time | Critical for Kerberos authentication — AD requires time sync |
| DNS Client | Required for Active Directory name resolution |
| Netlogon | Core AD authentication service |
| Active Directory Domain Services | Core domain controller service |
| Group Policy Client | Required for Group Policy application |
| Windows Firewall | Must remain running for firewall rules to apply |

### Notes
- All disabled services persist after restart as Startup Type is set to Disabled
- Disabling a service without setting Startup Type to Disabled will allow it to restart on reboot
- In a production environment, service hardening should be enforced through Group Policy for centralized management




  
