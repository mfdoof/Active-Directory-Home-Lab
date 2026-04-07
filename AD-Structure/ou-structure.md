# Organizational Unit (OU) Structure

## Overview
Custom OU structure created under `DOOF.local` to enable GPO linking, 
delegated administration, and logical separation of AD objects.
Default containers (CN=Users, CN=Computers) are not used for managed 
objects as GPOs cannot be linked to containers.
  
## OU Descriptions  

| OU | Purpose |
|---|---|
| Admins | Privileged user accounts with elevated permissions |
| Users | Standard end-user accounts |
| Groups | All security groups for RBAC and GPO filtering |
| Computers | Domain-joined workstations and devices |
| Service Accounts | Dedicated accounts for services and applications |

## Design Decisions

- **Custom OUs over default containers** — GPOs can only be linked to OUs, 
not default containers like `CN=Users` or `CN=Computers`
- **Protected from accidental deletion** — enabled on all OUs
- **Separate Groups OU** — keeps security groups manageable and 
easy to audit independently from user objects
- **Service Accounts isolated** — reduces attack surface by separating 
service account objects for easier monitoring and policy application

### Create Master OU
1. In ADUC, expand **DOOF.local** in the left panel
2. Right-click **DOOF.local** → **New** → **Organizational Unit**
3. Name: `DOOF`
4. Check **Protect container from accidental deletion**
5. Click **OK**

### Create Child OUs
Repeat for each child OU inside `DOOF`:

1. Right-click the **DOOF** OU → **New** → **Organizational Unit**
2. Enter the OU name from the table below
3. Check **Protect container from accidental deletion**
4. Click **OK**

## PowerShell — OU Creation
```powershell
# Create master OU
New-ADOrganizationalUnit -Name "DOOF" `
    -Path "DC=doof,DC=local" `
    -ProtectedFromAccidentalDeletion $true

# Create child OUs
$childOUs = @("Admins","Users","Groups","Computers","Service Accounts")
foreach ($ou in $childOUs) {
    New-ADOrganizationalUnit -Name $ou `
        -Path "OU=DOOF,DC=doof,DC=local" `
        -ProtectedFromAccidentalDeletion $true
}
```

## Verification
```powershell
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName | Sort-Object Name
```


