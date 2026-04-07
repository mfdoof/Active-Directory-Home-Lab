# GPO Password and Account Lockout Policy

## Overview
Domain-wide password and account lockout policy configured via the
**Default Domain Policy** GPO in `DOOF.local`. These settings apply to
all domain user accounts and establish the baseline security posture
for credential management.

> Fine-Grained Password Policies (PSO) were not implemented in this phase.
> PSOs will be configured in a future phase to apply stricter policies to privileged accounts.

## Policy Values

### Password Policy

| Setting | Value | Rationale |
|---|---|---|
| Minimum Password Length | 14 characters | NIST SP 800-63B recommendation |
| Password History | 24 passwords | Prevents reuse of recent passwords |
| Maximum Password Age | 60 days | Limits exposure window if compromised |
| Minimum Password Age | 1 day | Prevents immediate password cycling to reuse old ones |
| Complexity Enabled | Yes | Requires uppercase, lowercase, number, symbol |
| Reversible Encryption | No | Stores plaintext-equivalent — never enable |

### Account Lockout Policy

| Setting | Value | Rationale |
|---|---|---|
| Lockout Threshold | 5 attempts | Mitigates password spraying attacks |
| Lockout Duration | 30 minutes | Auto-unlocks after 30 min without admin intervention |
| Observation Window | 30 minutes | Resets failed attempt counter after 30 min |


## Configuration — GUI (GPME)

### Launch Group Policy Management
```powershell
gpmc.msc
```

### Navigate to Default Domain Policy
1. Expand **Forest: DOOF.local** → **Domains** → **DOOF.local**
2. Right-click **Default Domain Policy** → **Edit**
3. Navigate to:
```
Computer Configuration
└── Policies
    └── Windows Settings
        └── Security Settings
            └── Account Policies
                ├── Password Policy
                └── Account Lockout Policy
```

### Configure Password Policy
1. Click **Password Policy**
2. Configure each setting by double-clicking and entering the value:

| Setting | Value |
|---|---|
| Enforce password history | 24 passwords remembered |
| Maximum password age | 60 days |
| Minimum password age | 1 day |
| Minimum password length | 14 characters |
| Password must meet complexity requirements | Enabled |
| Store passwords using reversible encryption | Disabled |

### Configure Account Lockout Policy
1. Click **Account Lockout Policy**
2. Configure each setting:

| Setting | Value |
|---|---|
| Account lockout threshold | 5 invalid logon attempts |
| Account lockout duration | 30 minutes |
| Reset account lockout counter after | 30 minutes |

3. Close GPME → changes save automatically


## Configuration — PowerShell

### Set Password Policy
```powershell
Set-ADDefaultDomainPasswordPolicy -Identity "DOOF.local" `
    -MinPasswordLength 14 `
    -PasswordHistoryCount 24 `
    -MaxPasswordAge (New-TimeSpan -Days 60) `
    -MinPasswordAge (New-TimeSpan -Days 1) `
    -ComplexityEnabled $true `
    -ReversibleEncryptionEnabled $false  # Never enable — stores plaintext-equivalent
```

### Set Account Lockout Policy
```powershell
Set-ADDefaultDomainPasswordPolicy -Identity "DOOF.local" `
    -LockoutThreshold 5 `
    -LockoutDuration (New-TimeSpan -Minutes 30) `
    -LockoutObservationWindow (New-TimeSpan -Minutes 30)
```

### Force Policy Refresh
```powershell
gpupdate /force
```


## Verification

### Confirm Policy Applied
```powershell
Get-ADDefaultDomainPasswordPolicy -Identity "DOOF.local"
```

Expected output:
```
ComplexityEnabled          : True
LockoutDuration            : 00:30:00
LockoutObservationWindow   : 00:30:00
LockoutThreshold           : 5
MaxPasswordAge             : 60.00:00:00
MinPasswordAge             : 1.00:00:00
MinPasswordLength          : 14
PasswordHistoryCount       : 24
ReversibleEncryptionEnabled: False
```

### Check Resultant Policy
```powershell
gpresult /r /scope computer
```

## Security Notes
- **Reversible encryption disabled** — enabling this stores a plaintext-equivalent
  hash, exposing credentials if the NTDS.dit is extracted
- **Lockout threshold of 5** mitigates password spraying — attackers typically
  spray 1-3 attempts per account to stay under lockout thresholds
- **Observation window matches lockout duration** — ensures the failed attempt
  counter resets consistently with the lockout period
- **Default Domain Policy** is the only GPO that can enforce domain-wide
  password policy — password settings in other GPOs are ignored by AD
