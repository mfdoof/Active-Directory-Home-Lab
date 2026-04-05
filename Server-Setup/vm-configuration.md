# Virtual Machine Configuration

### Overview
This document covers the initial setup and configuration of the Windows Server 2022 virtual machine using Hyper-V on Windows 10/11. The VM serves as the foundation for the Active Directory home lab environment.

### Hypervisor
- **Platform:** Microsoft Hyper-V
- **Host OS:** Windows 10/11
- **VM OS:** Windows Server 2022 (Desktop Experience)

### VM Hardware Configuration

| Component | Configuration | Notes |
|---|---|---|
| Memory | 4096 MB | Minimum recommended for a Domain Controller |
| Processors | 4 Virtual CPUs | Sufficient for a single DC lab environment |
| Storage | Dynamic VHDX | Expands as needed |
| Network Adapter | External Switch | Used for real networking simulation |

### Network Switch Decision
For this lab, an **External Switch** was chosen over Internal or Private Switch. This decision was made to simulate real networking concepts including:
- Real DHCP behavior from the router
- Real DNS resolution
- Network traffic capture using Wireshark
- Understanding how NAT works

| Switch Type | Internet Access | Isolated from Host | Selected |
|---|---|---|---|
| External Switch | Yes | No | Yes |
| Internal Switch | Via host sharing | Yes | No |
| Private Switch | No | Fully isolated | No |

### Installation Steps

**Step 1 — Create a New Virtual Machine**  
Open Hyper-V Manager on your host machine. On the right panel click Action then select New followed by Virtual Machine. This will launch the New Virtual Machine Wizard. Give your VM a name and choose the storage location for the VM files.

**Step 2 — Configure Hardware**
Assign the following hardware resources to the VM:
- Memory: 4096 MB (minimum recommended for a Domain Controller)
- Enable Dynamic Memory: Optional
- Processors: 4 Virtual CPUs

**Step 3 — Configure Storage**  
Create a new Virtual Hard Disk (VHDX) for the VM. Select Dynamic expansion so the disk only uses as much space as it needs rather than allocating the full size immediately. Set an appropriate size to accommodate the OS and Active Directory database.

**Step 4 — Attach Installation Media**  
Navigate to the VM Settings and select DVD Drive. Attach the Windows Server 2022 ISO file as the installation media. Make sure the boot order is set to boot from the DVD Drive first so the installer launches on first start.

**Step 5 — Configure Network Adapter**  
In the VM Settings navigate to Network Adapter. Select External Switch as the virtual switch. Leave VLAN ID unchecked as it is not required for a home lab environment. IPv6 will be disabled at the OS level after installation.

**Step 6 — Boot and Install Windows Server 2022**  
Start the VM and boot from the ISO. When prompted select Windows Server 2022 Desktop Experience for the GUI interface. Choose Custom Installation when asked for the installation type. Follow the installation wizard to completion and set a strong Administrator password when prompted. The server will restart automatically once installation is complete.


### Post Installation Notes
- IPv6 was disabled at the OS level during network configuration
- Static IP was configured after OS installation
- VM was renamed to DC01 before promoting to Domain Controller

### Why Desktop Experience

| Version | GUI | Best For |
|---|---|---|
| Server Core | No | Production, minimal attack surface |
| Desktop Experience | Yes | Learning and lab environments |

Desktop Experience was selected for this lab as it provides a familiar GUI interface which is beneficial for learning and documentation purposes.
