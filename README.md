# Active-Directory-Home-Lab
A hands on project on Active Directory using Windows Server 2022 on Hyper-V

## Skills Demonstrated

- Windows Server 2022 administration
- Active Directory Domain Services (AD DS) deployment
- DNS server configuration
- Network configuration and static IP addressing
- Security hardening and attack surface reduction
- Firewall rule management
- Port mapping and analysis
- Service management and vulnerability mitigation
- Technical documentation
- OU, User, and Groups management

## Lab Environment

| Component | Details |
|---|---|
| Hypervisor | Microsoft Hyper-V |
| Host OS | Windows 10/11 |
| Server OS | Windows Server 2022 Desktop Experience |
| Domain Name | DOOF.local |
| Network Switch | External Switch |

## Phases

### Phase 1 — Server Setup
- Created Windows Server 2022 VM in Hyper-V
- Configured hardware: 4096MB RAM, 4 vCPUs, Dynamic VHDX
- Selected External Switch for real network simulation

### Phase 2 — Network Configuration
- Assigned static IP to Domain Controller
- Set DNS to loopback `127.0.0.1` with router IP as fallback
- Disabled IPv6 to prevent unexpected behavior

### Phase 3 — Active Directory Installation
- Installed AD DS and DNS Server roles
- Promoted server to Domain Controller
- Created new forest with root domain `DOOF.local`
- Fixed Windows Time Service warning via `w32tm`

### Phase 4 — Security Hardening
- Mapped all listening ports via `netstat -ano`
- Created inbound block rules for unnecessary ports
- Disabled unnecessary services

### Phase 5 — AD Structure (OUs, Users, Groups)  
- Created custom OU structure under `DOOF.local`
- Created domain user accounts with enforced password change on first logon
- Created security groups following `GRP-DEPT` naming convention
- Assigned users to groups for RBAC

### Phase 6 — GPO Password and Lockout Policy  
- Configured Default Domain Policy with password and lockout settings
- Applied domain-wide via `Set-ADDefaultDomainPasswordPolicy`
- Verified with `Get-ADDefaultDomainPasswordPolicy`

---

## Tools Used
| Tool | Purpose |
|---|---|
| Hyper-V | Virtualization |
| Windows Server 2022 | Domain Controller OS |
| Active Directory Users and Computers (ADUC) | AD object management |
| Group Policy Management (GPMC) | GPO configuration |
| PowerShell | Automation and verification |
| Windows Defender Firewall | Network hardening |

## Pending
- [ ] Configure Fine-Grained Password Policy (PSO) for privileged accounts
- [ ] Deploy LAPS for local admin password management
- [ ] Set up pfSense as virtual router and firewall
- [ ] Configure SIEM (Wazuh or Splunk) for log ingestion
- [ ] BloodHound AD attack path analysis and remediation

---





