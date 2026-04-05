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





  
