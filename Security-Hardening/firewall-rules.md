# Firewall Rules

### Overview
This document covers the firewall hardening applied to the Windows Server 2022 Domain Controller. The hardening process began by mapping all open and listening ports using `netstat -ano` to identify active services. Ports associated with outdated or unnecessary protocols were then blocked using Windows Defender Firewall with Advanced Security. All block rules were verified from the host machine using `Test-NetConnection` to confirm the ports were no longer reachable externally.

### Why Firewall Hardening is Important
A Domain Controller is the most critical server in an Active Directory environment. If compromised, an attacker gains control over authentication, Group Policy, and all domain joined machines. Hardening the firewall reduces the attack surface by ensuring only necessary ports are accessible.

### Management Interface
All firewall rules were configured through the Windows Defender Firewall with Advanced Security GUI

### Understanding Firewall Profiles
Windows Firewall applies rules based on three network profiles:

| Profile | When It Applies |
|---|---|
| Domain | When the server detects it is connected to the domain network |
| Private | When connected to a trusted private network |
| Public | When connected to an untrusted public network |

All block rules in this document were applied to **Domain, Private, and Public** profiles to ensure consistent enforcement across all network conditions.

### Rule Priority
Windows Firewall processes rules in the following order:
- Block rules always override Allow rules
- More specific rules take priority over general rules
- Rules are processed top to bottom within the same priority level

### Blocked Ports

| Port | Protocol | Service | Reason Blocked |
|---|---|---|---|
| 21 | TCP | FTP | Unencrypted file transfer, not needed on a Domain Controller |
| 23 | TCP | Telnet | Unencrypted remote access, outdated and replaced by SSH |
| 137 | UDP | NetBIOS Name Service | Legacy protocol, vulnerable to NBNS spoofing attacks |
| 138 | UDP | NetBIOS Datagram | Legacy protocol, unnecessary on a Domain Controller |
| 139 | TCP | NetBIOS Session | Legacy protocol, showing in netstat output |
| 593 | TCP | RPC over HTTP | Unnecessary remote management port on a Domain Controller |
| 5985 | TCP | WinRM HTTP | Unencrypted Windows remote management |

### RDP Status

| Port | Service | Status | Reason |
|---|---|---|---|
| 3389 | Remote Desktop Protocol | Disabled | Not needed — Hyper-V console provides direct access |

RDP was found to be disabled by default after Domain Controller promotion. Since the server is managed directly through the Hyper-V console on the host machine, enabling RDP was not necessary.

### How to Create a Block Rule via GUI

**Step 1 — Open Windows Defender Firewall with Advanced Security**  
Server Manager → Tools → Windows Defender Firewall with Advanced Security

**Step 2 — Navigate to Inbound Rules**  
Click Inbound Rules in the left panel -> Click New Rule in the right panel  

**Step 3 — Select Rule Type**  
Select Port -> Click Next

**Step 4 — Configure Protocol and Port**    
Select TCP or UDP depending on the port
-> Select Specific local ports
-> Enter the port number
-> Click Next

**Step 5 — Set the Action**  
Select Block the connection
-> Click Next

**Step 6 — Apply to All Profiles**  
Check Domain, Private, and Public
-> Click Next

**Step 7 — Name the Rule**  
Enter a descriptive name such as Block Telnet 23
-> Click Finish

### How to Verify Block Rules

All block rules were verified from the host machine using PowerShell to simulate an external connection attempt:
```powershell
Test-NetConnection -ComputerName SERVER_IP -Port PORT_NUMBER
```

**Expected result for a correctly blocked port:**  
```powershell
TcpTestSucceeded : False
```

**Important:** Always test block rules from the host machine and not from inside the server itself. Testing from inside the server uses the loopback interface which bypasses firewall rules and will always return True regardless of the rule.

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

### Firewall Logging
Firewall logging was enabled on all three profiles to monitor traffic hitting the server.

**What the log captures:**

| Field | Description |
|---|---|
| Date and Time | When the connection attempt occurred |
| Action | DROP for blocked, ALLOW for permitted |
| Source IP | Where the connection came from |
| Destination IP | Where the connection was going |
| Source Port | Port the connection originated from |
| Destination Port | Port being targeted — match against blocked ports list |

### Notes
- All block rules persist after restart as they are stored in the Windows Registry
- Rules were verified to persist after a full server restart
- Firewall rules should be recreated through Group Policy in a production environment for centralized management













