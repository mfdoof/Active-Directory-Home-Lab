# Computer Configuration
### Overview
This document covers the creation and management of computer accounts in the DOOF.local domain. 
It includes creating individual computer accounts, organizing them into department OUs, 
and prestaging computers before domain join. Naming convention follows the format 
DEPARTMENT-PC# (e.g. IT-PC01, HR-PC01).

### OU Structure — Computers

```
DOOF.local
└── Computers
    ├── IT
    │   └── IT-PC01
    └── HR
        └── HR-PC01
```

---

## Part 1 — Create Department Sub-OUs Under Computers

### PowerShell
```powershell
# Create IT and HR sub-OUs under the Computers OU
New-ADOrganizationalUnit -Name "IT" -Path "OU=Computers,DC=DOOF,DC=local"
New-ADOrganizationalUnit -Name "HR" -Path "OU=Computers,DC=DOOF,DC=local"

# Verify
Get-ADOrganizationalUnit -Filter * | Where-Object {$_.DistinguishedName -like "*Computers*"} |
    Select-Object Name, DistinguishedName
```

### GUI
1. Open **Active Directory Users and Computers** → `dsa.msc`
2. Expand `DOOF.local` → right-click **Computers** OU → **New** → **Organizational Unit**
3. Enter `IT` → click **OK**
4. Repeat for `HR`

---

## Part 2 — Create a Computer Account

### PowerShell
```powershell
# Create computer account in the correct department OU
New-ADComputer -Name "IT-PC01" `
    -SamAccountName "IT-PC01" `
    -Path "OU=IT,OU=Computers,DC=DOOF,DC=local" `
    -Description "IT Department Workstation 01" `
    -Enabled $true

# Verify
Get-ADComputer -Identity "IT-PC01" | 
    Select-Object Name, DistinguishedName, Enabled
```

### GUI
1. Open **Active Directory Users and Computers** → `dsa.msc`
2. Expand `DOOF.local` → **Computers** → **IT**
3. Right-click **IT** → **New** → **Computer**
4. Enter computer name: `IT-PC01`
5. Click **OK**

---

## Part 3 — Prestage a Computer Account

Prestaging creates the computer account in AD before the machine joins the domain.
This pre-assigns it to the correct OU and ensures proper permissions are in place.

### PowerShell
```powershell
# Prestage IT-PC01 in the IT computers OU
New-ADComputer -Name "IT-PC01" `
    -SamAccountName "IT-PC01" `
    -Path "OU=IT,OU=Computers,DC=DOOF,DC=local" `
    -Description "IT Department Workstation 01 — Prestaged" `
    -Enabled $false  # Disabled until machine joins domain

# Verify prestaged account exists
Get-ADComputer -Identity "IT-PC01" | 
    Select-Object Name, DistinguishedName, Enabled
# Expected: Enabled = False until domain join completes
```

### GUI
1. Open **Active Directory Users and Computers** → `dsa.msc`
2. Expand `DOOF.local` → **Computers** → **IT**
3. Right-click **IT** → **New** → **Computer**
4. Enter computer name: `IT-PC01`
5. Click **Change** to assign which user or group can join this computer to the domain
6. Click **OK** — account is created as disabled until the machine joins

---

## Part 4 — Move a Computer Account to the Correct OU

If a computer joins the domain and lands in the default Computers container 
instead of the correct OU, move it manually.

### PowerShell
```powershell
# Move IT-PC01 to the IT sub-OU under Computers
Move-ADObject -Identity "CN=IT-PC01,CN=Computers,DC=DOOF,DC=local" `
    -TargetPath "OU=IT,OU=Computers,DC=DOOF,DC=local"

# Verify
Get-ADComputer -Identity "IT-PC01" | 
    Select-Object Name, DistinguishedName
# Expected: DistinguishedName shows OU=IT,OU=Computers
```

### GUI
1. Open **Active Directory Users and Computers** → `dsa.msc`
2. Locate `IT-PC01` in the default **Computers** container
3. Right-click `IT-PC01` → **Move**
4. Navigate to `DOOF.local` → **Computers** → **IT**
5. Click **OK**

---

## Part 5 — Verify Computer Account

### PowerShell
```powershell
# Verify computer account details
Get-ADComputer -Identity "IT-PC01" -Properties * | 
    Select-Object Name, DistinguishedName, Enabled, LastLogonDate, OperatingSystem

# List all computers in a specific department OU
Get-ADComputer -Filter * -SearchBase "OU=IT,OU=Computers,DC=DOOF,DC=local" | 
    Select-Object Name, DistinguishedName
```

### GUI
1. Open **Active Directory Users and Computers** → `dsa.msc`
2. Expand `DOOF.local` → **Computers** → **IT**
3. Confirm `IT-PC01` is listed with correct details

---

## Current Lab Computers

| Computer Name | Department | OU Path | Status |
|---|---|---|---|
| IT-PC01 | IT | OU=IT,OU=Computers,DC=DOOF,DC=local | Domain Joined |

---

## Notes
- Always create the computer account in the correct department OU before domain join
- If the machine joins before the account is prestaged, it lands in the default 
  Computers container — use Part 4 to move it
- Naming convention: DEPARTMENT-PC# (e.g. IT-PC01, HR-PC01)
- Computer accounts disabled before domain join is expected behavior — 
  they enable automatically once the machine joins


