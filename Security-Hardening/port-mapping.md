# Port Mapping

### Overview
This document covers the port mapping process performed on the Windows Server 2022 Domain Controller as part of the security hardening process. Port mapping was conducted using the `netstat -ano` command to identify all open and listening ports on the server. The results were then reviewed and categorized as required or unnecessary.

### How Port Mapping Was Done

All listening ports were identified using the following command in Command Prompt:
```cmd
netstat -ano | findstr LISTENING
```

| Field | Description |
|---|---|
| Proto | Protocol being used (TCP or UDP) |
| Local Address | IP and port the service is listening on |
| Foreign Address | Remote address of the connection |
| State | Current state of the connection |
| PID | Process ID — used to identify which service owns the port |

### Understanding Port Ranges

| Range | Name | Purpose |
|---|---|---|
| 0 - 1023 | Well-known ports | Standard services like DNS (53) and HTTP (80) |
| 1024 - 49151 | Registered ports | Specific applications and services |
| 49152 - 65535 | Ephemeral ports | Temporary dynamic ports used for active connections |

### Why High Numbered Ports Are Open

High numbered ports in the ephemeral range (49152-65535) are completely normal and expected on a Domain Controller. These are dynamic ports assigned by RPC (Remote Procedure Call) for temporary connections between services.

Think of it this way:
- Fixed ports (53, 88, 389) are the front doors of the server — always open and in the same location
- High numbered ports are temporary meeting rooms — open only when needed and closed when the conversation ends

Every time a client authenticates, queries AD, or applies Group Policy, a temporary high numbered port is opened for that session and closed when it finishes. Blocking these ports would break Active Directory functionality.

### Mapped Ports

#### Required Ports — Active Directory and DNS

| Port | Protocol | Service | Purpose |
|---|---|---|---|
| 53 | TCP/UDP | DNS | Resolves domain names — critical for Active Directory |
| 88 | TCP/UDP | Kerberos | Handles all domain authentication |
| 135 | TCP | RPC | Remote Procedure Call — Windows services communication |
| 389 | TCP/UDP | LDAP | Active Directory directory queries |
| 445 | TCP | SMB | Windows file sharing and AD communication |
| 464 | TCP/UDP | Kerberos Password | Handles password changes in Active Directory |
| 636 | TCP | LDAPS | Secure LDAP — encrypted version of port 389 |
| 3268 | TCP | Global Catalog | Allows clients to search the entire AD forest |
| 3269 | TCP | Global Catalog SSL | Encrypted version of Global Catalog |
| 9389 | TCP | AD Web Services | Used by AD Administrative Center and PowerShell AD module |

#### Ports Reviewed and Blocked

| Port | Protocol | Service | Action Taken | Reason |
|---|---|---|---|---|
| 21 | TCP | FTP | Blocked | Unencrypted file transfer, not needed on a Domain Controller |
| 23 | TCP | Telnet | Blocked | Unencrypted remote access, outdated protocol |
| 137 | UDP | NetBIOS Name Service | Blocked | Legacy protocol, vulnerable to NBNS spoofing |
| 138 | UDP | NetBIOS Datagram | Blocked | Legacy protocol, unnecessary on a Domain Controller |
| 139 | TCP | NetBIOS Session | Blocked | Legacy protocol, found in netstat output |
| 593 | TCP | RPC over HTTP | Blocked | Unnecessary remote management port |
| 5985 | TCP | WinRM HTTP | Blocked | Unencrypted Windows remote management |

#### Ports Reviewed and Left Open

| Port | Protocol | Service | Reason Left Open |
|---|---|---|---|
| 464 | TCP/UDP | Kerberos Password | Required for AD password changes |
| 636 | TCP | LDAPS | Secure encrypted LDAP — preferred over 389 |
| 3268 | TCP | Global Catalog | Required for AD forest searches |
| 3269 | TCP | Global Catalog SSL | Encrypted Global Catalog — secure AD forest searches |
| 5986 | TCP | WinRM HTTPS | Encrypted remote management — monitored |
| 9389 | TCP | AD Web Services | Required for AD Administrative Center |

#### RDP Status

| Port | Protocol | Service | Status | Reason |
|---|---|---|---|---|
| 3389 | TCP | Remote Desktop Protocol | Disabled by default | Managed via Hyper-V console instead |

### Ephemeral Port Range

| Detail | Value |
|---|---|
| Range | 49152 - 65535 |
| Controlled by | RPC (Remote Procedure Call) |
| Behavior | Opens temporarily for each AD session, closes when done |
| Should be blocked | No — blocking breaks Active Directory |

### Verification Method

All blocked ports were verified from the host machine using PowerShell to simulate an external connection attempt:
```powershell
Test-NetConnection -ComputerName SERVER_IP -Port PORT_NUMBER
```

**Important:** Always verify block rules from the host machine and not from inside the server. Testing from inside the server bypasses the firewall through the loopback interface and always returns True regardless of the actual firewall rule.

### Verification Results

| Port | Service | TcpTestSucceeded | Status |
|---|---|---|---|
| 21 | FTP | False | Confirmed Blocked |
| 23 | Telnet | False | Confirmed Blocked |
| 137 | NetBIOS Name Service | False | Confirmed Blocked |
| 138 | NetBIOS Datagram | False | Confirmed Blocked |
| 139 | NetBIOS Session | False | Confirmed Blocked |
| 593 | RPC over HTTP | False | Confirmed Blocked |
| 5985 | WinRM HTTP | False | Confirmed Blocked |

### Notes
- Port mapping should be repeated after any new role or feature is installed to check for newly opened ports
- Any port that is not recognized or expected should be investigated before deciding to block or allow it
- In a production environment, port mapping and firewall rules should be documented and reviewed regularly as part of a security audit
