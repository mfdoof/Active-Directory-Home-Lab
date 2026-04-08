# Disabled Services
### Overview
This document covers the unnecessary services that were disabled on the Windows Server 2022 Domain Controller as part of the security hardening process. Disabling unused services reduces the attack surface of the Domain Controller by eliminating potential entry points that serve no purpose in the environment.

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
NetBIOS over TCP/IP was disabled at the NIC level on DC01. Blocking NetBIOS only at the firewall is insufficient — the ports remain bound and the service continues running internally. Disabling at the NIC level stops the binding entirely. Firewall rules on ports 137, 138, and 139 are retained as defense-in-depth.

**Why disable NetBIOS:**
- NetBIOS name resolution is a legacy protocol not required by modern Active Directory
- Active Directory uses DNS and Kerberos for name resolution and authentication
- NetBIOS is an active attack surface for LLMNR/NBT-NS poisoning attacks (e.g. Responder)
- Disabling at the NIC eliminates the binding — firewall rules alone do not

### Configuration — PowerShell

```powershell
# Step 1 — Check current NetBIOS setting per adapter
# TcpipNetbiosOptions values: 0 = Default, 1 = Enabled, 2 = Disabled
$nic = Get-WmiObject Win32_NetworkAdapterConfiguration | Where-Object { $_.IPEnabled -eq $true }
$nic | Select-Object Description, TcpipNetbiosOptions

# Step 2 — Disable NetBIOS over TCP/IP on all IP-enabled adapters
$adapters = Get-WmiObject Win32_NetworkAdapterConfiguration | Where-Object { $_.IPEnabled -eq $true }
foreach ($adapter in $adapters) {
    $adapter.SetTcpipNetbios(2)  # 2 = Disable NetBIOS over TCP/IP
}

# Step 3 — Verify
Get-WmiObject Win32_NetworkAdapterConfiguration |
    Where-Object { $_.IPEnabled -eq $true } |
    Select-Object Description, TcpipNetbiosOptions
# Expected: TcpipNetbiosOptions = 2 on all adapters
```

### Configuration — GUI

1. Open `ncpa.cpl`
2. Right-click your NIC → Properties
3. Select **Internet Protocol Version 4 (TCP/IPv4)** → Properties
4. Click **Advanced** → select the **WINS** tab
5. Select **Disable NetBIOS over TCP/IP**
6. Click OK

### Verification

```powershell  
# Confirm registry reflects disabled state
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\NetBT\Parameters\Interfaces\*" |
    Select-Object PSChildName, NetbiosOptions
# Expected: all values = 2
```

> **Note:** Do not test NetBIOS resolution from the DC itself — test from the Windows 11 client or another machine on the network. Testing from the DC can return false positives via loopback.

```powershell
# Run from Windows 11 client — not the DC
nbtstat -a DC01
# Expected: Host not found / timeout
```

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
- NetBIOS firewall block rules on ports 137, 138, 139 are retained alongside NIC-level disabling as defense-in-depth





  
