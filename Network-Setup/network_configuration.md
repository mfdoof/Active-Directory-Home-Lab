# Network Configuration

### Overview
This document covers the network configuration of the Windows Server 2022 virtual machine prior to Active Directory installation. Proper network configuration is critical for a Domain Controller as it ensures stable connectivity, reliable DNS resolution, and consistent IP addressing.

### Why Static IP

By default, the router assigns IP addresses through **DHCP (Dynamic Host Configuration Protocol)**. DHCP is designed for convenience as it automatically assigns an IP to any device that connects, but that IP can change every time the device reconnects.

This is a problem for a Domain Controller because:
- Every machine that joins the domain relies on the Domain Controller's IP address for authentication, DNS resolution, and Group Policy
- If the IP address changes, those machines lose contact with the Domain Controller causing the domain to break
- Active Directory DNS records would become inaccurate and break

A static IP means manually assigning a permanent IP address that never changes, bypassing DHCP entirely.

### Network Configuration Settings

| Setting | Value | Reason |
|---|---|---|
| IP Address | Static | Permanent address for Domain Controller |
| Subnet Mask | Router default gateway | Gateway for local network |
| Default Gateway | Router IP | Handles internet traffic |
| Preferred DNS | 127.0.0.1 | Server points to itself for AD DNS |
| Alternate DNS | Router IP | Fallback if AD DNS fails |
| IPv6 | Disabled | Not needed, prevents unexpected behavior |

### Why 127.0.0.1 for DNS

`127.0.0.1` is the loopback address which means point to yourself. When the server is promoted to a Domain Controller it runs its own DNS service. The server is told to ask itself first for DNS rather than relying on an external source.

This is critical because Active Directory uses DNS for almost everything:
- Finding the Domain Controller
- Authenticating users
- Locating services like Kerberos and LDAP
- Replication between Domain Controllers

If DNS pointed to the router or a public DNS like Cloudflare, the server would be asking an external server to resolve internal AD names like `DC01.DOOF.local` which they have no knowledge of and would fail.

### Can Public DNS be Used Instead

| DNS | Knows AD Domain | Can Resolve DOOF.local |
|---|---|---|
| 127.0.0.1 (self) | Yes | Yes |
| 1.1.1.1 Cloudflare | No | No |
| 8.8.8.8 Google | No | No |

Public DNS can be used as a **DNS Forwarder** inside DNS Manager. This means:
- The DC handles all internal AD DNS queries locally
- Internet DNS queries are forwarded to Cloudflare or Google
- Best of both worlds without breaking AD

### Why IPv6 Was Disabled

Windows prefers IPv6 over IPv4 by default. Leaving IPv6 enabled but unconfigured on a Domain Controller can cause:
- Slow network lookups
- Authentication delays
- Unexpected behavior as Windows tries IPv6 first before falling back to IPv4

Disabling IPv6 keeps the configuration clean and avoids unnecessary troubleshooting in a home lab environment.

### Configuration Steps

**Step 1 — Open Network Settings**    
Navigate to the Network Adapter settings through Server Manager.

**Step 2 — Open Adapter Properties**    
Right-click on the Network Adapter and open the IPv4 settings.

**Step 3 — Configure Static IP**    
Select **Use the following IP address** and enter your network details.

**Step 4 — Configure DNS**    
Select **Use the following DNS server addresses** and enter the following.

**Step 5 — Disable IPv6**    
In the adapter Properties window uncheck IPv6 to disable it.

**Step 6 — Verify Configuration**      
Open Command Prompt and run the following command to confirm all settings are applied correctly.  
`ipconfig /all`

Confirm the following:
- IPv4 Address shows your static IP
- DNS Servers shows 127.0.0.1 as preferred
- IPv6 is not listed

### Post Configuration Notes
- Default Gateway and DNS were previously pointing to the ISP router via DHCP
- After static IP configuration the server no longer relies on the router for DNS
- The router remains as the Default Gateway for internet traffic only
- All Active Directory DNS is handled internally by the server itself


