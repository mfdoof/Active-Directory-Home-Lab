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

## Lab Environment

| Component | Details |
|---|---|
| Hypervisor | Microsoft Hyper-V |
| Host OS | Windows 10/11 |
| Server OS | Windows Server 2022 Desktop Experience |
| Domain Name | DOOF.local |
| Network Switch | External Switch |

## Phases

**Phase 1 — Virtual Machine Setup**  
Configured a Windows Server 2022 VM in Hyper-V with an External Switch for real networking simulation.

**Phase 2 — Network Configuration**  
Assigned a static IP, configured DNS to loopback address (127.0.0.1), and disabled IPv6.

**Phase 3 — Active Directory Installation**  
Installed AD DS and DNS roles, promoted the server to a Domain Controller, and created the forest.

**Phase 4 — Security Hardening**  
Mapped all open ports, blocked unnecessary protocols, disabled vulnerable services, and enabled firewall logging.

## What is Next
- [ ] Password and lockout policy via GPO
- [ ] Join a client VM to the domain
- [ ] Create OUs, Users, and Groups
- [ ] Configure Group Policy Objects
- [ ] Set up pfSense as virtual router


