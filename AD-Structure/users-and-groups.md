# Users and Groups  

## Overview  
Domain user accounts and security groups created under the `DOOF` OU structure
in `DOOF.local`. All accounts follow a standard naming convention and are
organized by department and role to support RBAC (Role-Based Access Control).

## Naming Conventions  

| Object | Convention | Example |
|---|---|---|
| Users | firstname + lastname initial | `jdoe` |
| Groups | GRP-DEPT| `GRP-HR`, `GRP-IT` |
| UPN | samaccountname@doof.local | `jdoe@doof.local` |


## Users

| Full Name | Username | UPN | Department | OU |
|---|---|---|---|---|
| Jhon Doe | jhondoe | jhondoe@doof.local | HR | Users |
| Mike Rose | mrose | mrose@doof.local | IT | Users |

> **Default Password:** `************`  
> **Password change required on first logon:** Yes

## Groups

| Group Name | Scope | Type | Members |
|---|---|---|---|
| GRP-HR | Global | Security | jhondoe |
| GRP-IT | Global | Security | mrose |


## Creation — GUI (ADUC)

### Launch ADUC
```powershell
dsa.msc
```

### Create a User
1. In ADUC, expand **DOOF** → right-click **Users** OU → **New** → **User**
2. Fill in:
   - First name, Last name
   - User logon name (SamAccountName) e.g. `jhondoe`
3. Click **Next**
4. Set password: `*********!`
5. Check **User must change password at next logon**
6. Uncheck **User cannot change password**
7. Uncheck **Password never expires**
8. Click **Next** → **Finish**

### Create a Group
1. Right-click **Groups** OU → **New** → **Group**
2. Fill in:
   - Group name e.g. `GRP-HR`
   - Group scope: **Global**
   - Group type: **Security**
3. Click **OK**

### Add User to Group
1. Double-click the group → **Members** tab → **Add**
2. Type the username e.g. `jhondoe` → **Check Names** → **OK**
3. Click **Apply** → **OK**


## Creation — PowerShell

### Create a Single User
```powershell
New-ADUser `
    -Name "Mike Rose" `
    -GivenName "Mike" `
    -Surname "Rose" `
    -SamAccountName "mrose" `
    -UserPrincipalName "mrose@doof.local" `
    -Path "OU=Users,OU=DOOF,DC=doof,DC=local" `
    -AccountPassword (ConvertTo-SecureString "******" -AsPlainText -Force) `
    -ChangePasswordAtLogon $true `
    -Enabled $true
```

### Bulk Create Users
```powershell
$defaultPassword = ConvertTo-SecureString "******" -AsPlainText -Force

$users = @(
    @{ Name="Jhon Doe";  Sam="jhondoe"; OU="Users" },
    @{ Name="Mike Rose"; Sam="mrose";   OU="Users" }
)

foreach ($u in $users) {
    $first, $last = $u.Name.Split(" ")
    New-ADUser `
        -Name $u.Name `
        -GivenName $first `
        -Surname $last `
        -SamAccountName $u.Sam `
        -UserPrincipalName "$($u.Sam)@doof.local" `
        -Path "OU=$($u.OU),OU=DOOF,DC=doof,DC=local" `
        -AccountPassword $defaultPassword `
        -ChangePasswordAtLogon $true `
        -Enabled $true
}
```

### Create a Group
```powershell
New-ADGroup -Name "GRP-HR" `
    -GroupScope Global `
    -GroupCategory Security `
    -Path "OU=Groups,OU=DOOF,DC=doof,DC=local"
```

### Add User to Group
```powershell
Add-ADGroupMember -Identity "GRP-HR" -Members "jhondoe"
```

### Bulk Add Members
```powershell
Add-ADGroupMember -Identity "GRP-HR" -Members "jhondoe","mrose"
```

## Verification
```powershell
# List all users in DOOF OU
Get-ADUser -Filter * -SearchBase "OU=DOOF,DC=doof,DC=local" `
    | Select-Object Name, SamAccountName, Enabled

# Confirm specific user
Get-ADUser -Identity "mrose" `
    | Select-Object Name, SamAccountName, Enabled, DistinguishedName

# List group members
Get-ADGroupMember -Identity "GRP-HR" | Select-Object Name, SamAccountName
```

## Security Notes
- Default password meets the 14-character minimum policy
- `ChangePasswordAtLogon` enforced — limits default password exposure window
- Users placed in role-specific OUs to allow granular GPO application per department
- Groups follow GRP-DEPT naming — easy to identify scope during access reviews

