# Active Directory Lab - Penetration Testing

A complete Active Directory lab setup for learning penetration testing and offensive security techniques. This lab demonstrates practical AD enumeration, configuration, and security assessments.

## 📋 Project Overview

**Status:** ✅ Complete and Functional  
**Difficulty:** Beginner to Intermediate  
**Use Case:** eJPT/CRTO/CRTE Exam Preparation  
**Time to Build:** 4-6 Hours

## 🏗️ Lab Infrastructure

### Virtual Machines
| VM | Role | OS | IP Address |
|---|---|---|---|
| DC01 | Domain Controller | Windows Server 2022 | 192.168.56.101 |
| Kali | Attacker/Enumeration | Kali Linux | 192.168.56.102 |

### Technologies Used
- **Hypervisor:** Oracle VirtualBox 7.x
- **Network:** Bridged Adapter
- **Domain:** akshay.local
- **Services:** Active Directory, DNS, NTDS

## 📁 Repository Structure

```
Active-Directory-Lab/
├── README.md                          # This file
├── .gitignore                         # Git ignore rules
├── AD_Lab_Complete_Documentation.txt  # Full lab documentation
│
├── setup-scripts/                     # PowerShell scripts
│   ├── 01-install-ad-ds.ps1
│   ├── 02-promote-dc.ps1
│   ├── 03-create-ous.ps1
│   ├── 04-create-users.ps1
│   ├── 05-create-groups.ps1
│   └── 06-add-users-to-groups.ps1
│
├── enumeration-commands/              # Enumeration guides
│   ├── PowerShell-Commands.md
│   ├── LDAP-Queries.md
│   └── Offensive-Techniques.md
│
├── lab-documentation/                 # Reference docs
│   ├── Network-Diagram.txt
│   ├── User-List.txt
│   ├── Group-List.txt
│   └── Troubleshooting-Guide.txt
│
└── notes/                             # Learning notes
    ├── Lessons-Learned.md
    ├── Challenges-Faced.md
    └── Future-Improvements.md
```

## 🎯 What You'll Learn

✅ Windows Server administration  
✅ Active Directory configuration  
✅ PowerShell scripting  
✅ AD enumeration techniques  
✅ LDAP query fundamentals  
✅ Kerberos basics  
✅ Domain security concepts  
✅ Penetration testing methodology  

## 🚀 Quick Start

### Prerequisites
- Oracle VirtualBox installed
- Windows Server 2022 ISO
- Kali Linux ISO
- 8GB+ RAM available
- 100GB+ disk space

### 1. Setup DC01 (Domain Controller)

```powershell
# Run as Administrator on Windows Server 2022

# Step 1: Install AD DS
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# Step 2: Promote to Domain Controller
Install-ADDSForest -DomainName "akshay.local" `
  -SafeModeAdministratorPassword (ConvertTo-SecureString -AsPlainText "YourStrongPassword" -Force) `
  -Force

# Server will restart automatically
```

### 2. Create Active Directory Structure

```powershell
# Create Organizational Units
New-ADOrganizationalUnit -Name "IT" -Path "DC=akshay,DC=local"
New-ADOrganizationalUnit -Name "Sales" -Path "DC=akshay,DC=local"
New-ADOrganizationalUnit -Name "Managers" -Path "DC=akshay,DC=local"

# Create Users
New-ADUser -Name "Rahul Sharma" `
  -SamAccountName "rahul.sharma" `
  -UserPrincipalName "rahul.sharma@akshay.local" `
  -Path "OU=IT,DC=akshay,DC=local" `
  -AccountPassword (ConvertTo-SecureString -AsPlainText "YourPassword" -Force) `
  -Enabled $true

# Create Security Groups
New-ADGroup -Name "IT-Team" `
  -SamAccountName "IT-Team" `
  -GroupCategory Security `
  -GroupScope Global `
  -Path "CN=Users,DC=akshay,DC=local"

# Add Users to Groups
Add-ADGroupMember -Identity "IT-Team" -Members "rahul.sharma"
```

### 3. Configure Firewall (Lab Only!)

```powershell
# Disable firewall for testing (NOT for production!)
netsh advfirewall set allprofiles state off

# Enable anonymous LDAP queries
reg add "HKLM\SYSTEM\CurrentControlSet\Services\LDAP" `
  /v dsHeuristics /t REG_SZ /d 0000002 /f
```

### 4. Enumeration from Kali Linux

```bash
# List all AD users via LDAP
ldapsearch -x -h 192.168.56.101 -b "DC=akshay,DC=local" "(&(objectClass=user)(objectCategory=person))"

# Enumerate groups
ldapsearch -x -h 192.168.56.101 -b "DC=akshay,DC=local" "(objectClass=group)"

# Verify DNS resolution
nslookup akshay.local 192.168.56.101

# Use enum4linux for automated enumeration
enum4linux -u "" -p "" 192.168.56.101
```

## 📚 Active Directory Structure Created

### Users
1. **Rahul Sharma** (rahul.sharma@akshay.local)
   - Department: IT
   - Group: IT-Team

2. **Priya Verma** (priya.verma@akshay.local)
   - Department: Sales
   - Group: Sales-Team

3. **Administrator** (Domain Admin)
   - Full admin access

### Organizational Units
- **IT** - IT department users
- **Sales** - Sales department users
- **Managers** - Manager-level users

### Security Groups
- **IT-Team** - IT department access control
- **Sales-Team** - Sales department access control

## 🔍 Enumeration Examples

### PowerShell Enumeration
```powershell
# Get all users
Get-ADUser -Filter * -Properties * | Select-Object Name, SamAccountName

# Get all groups
Get-ADGroup -Filter * | Select-Object Name, GroupScope

# Check group membership
Get-ADGroupMember -Identity "IT-Team"

# Get domain info
Get-ADDomain
```

### LDAP Enumeration
```bash
# Get all users
ldapsearch -x -h 192.168.56.101 -b "DC=akshay,DC=local" "(objectClass=user)"

# Get admins
ldapsearch -x -h 192.168.56.101 -b "DC=akshay,DC=local" "(&(objectClass=user)(memberOf=CN=Administrators*))"

# Get service accounts
ldapsearch -x -h 192.168.56.101 -b "DC=akshay,DC=local" "(servicePrincipalName=*)"
```

## 🎓 Learning Path

### Week 1-2: Foundation
- ✅ AD basics and concepts
- ✅ DC setup and configuration
- ✅ User and group creation

### Week 3-4: Enumeration
- ✅ PowerShell enumeration
- ✅ LDAP queries
- ✅ Active Directory reconnaissance

### Week 5-6: Next (Not included in this lab)
- Credential dumping techniques
- Hash extraction
- Pass-the-hash attacks

### Week 7+: Advanced
- Kerberoasting
- ASREPRoasting
- Privilege escalation
- Lateral movement

## ⚙️ Troubleshooting

### VMs can't communicate
```powershell
# Check network adapter settings in VirtualBox
# Ensure both VMs use Bridged Adapter
# Verify IP addresses: 192.168.56.101 and 192.168.56.102
```

### LDAP queries fail
```bash
# Verify firewall is disabled on DC
# Check NTDS service is running
# Ensure LDAP anonymous queries are enabled
```

### Users can't login
```powershell
# Verify account is enabled
Get-ADUser username | Select Enabled

# Reset password
Set-ADAccountPassword -Identity username -Reset -NewPassword (ConvertTo-SecureString -AsPlainText "NewPassword" -Force)
```

## 🔒 Security Notes

⚠️ **Important Security Reminders:**

1. **This is a LAB environment only** - Not for production
2. **Firewall disabled** - Only for testing
3. **Anonymous LDAP enabled** - Simulates misconfiguration
4. **Weak passwords used** - For testing purposes
5. **Never use these settings in production**

## 📝 Key Files

- **AD_Lab_Complete_Documentation.txt** - Full technical documentation
- **setup-scripts/** - PowerShell automation scripts
- **enumeration-commands/** - Command reference guides
- **.gitignore** - Files excluded from version control

## 🎯 Portfolio Value

This lab demonstrates:
- 🏢 Enterprise infrastructure understanding
- 🛡️ Security configuration knowledge
- 💻 PowerShell scripting skills
- 🔍 Penetration testing methodology
- 📚 Technical documentation abilities
- 🧠 Problem-solving and troubleshooting

## 🚦 Next Steps

1. ✅ Complete this basic lab
2. ⬜ Add Windows 10 client machine
3. ⬜ Implement Group Policies
4. ⬜ Practice Kerberoasting
5. ⬜ Learn privilege escalation
6. ⬜ Setup BloodHound

## 📖 References

- [Microsoft Active Directory Documentation](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/active-directory-domain-services)
- [eJPT Exam Guide](https://www.elearnsecurity.com/course/ejpt/)
- [CRTO Certification](https://www.maltego.com/crto/)
- [AD Security Best Practices](https://attack.mitre.org/)

## 📧 Questions & Support

For questions about:
- **Lab setup** - Check AD_Lab_Complete_Documentation.txt
- **Commands** - See enumeration-commands/
- **Troubleshooting** - See lab-documentation/Troubleshooting-Guide.txt
- **Learning** - Check notes/ folder

## 📄 License

This lab documentation is provided for educational purposes. Use responsibly.

## ✨ Credits

Created for cybersecurity learning and pentesting practice.

---

**Last Updated:** July 2026  
**Lab Status:** ✅ Fully Functional  
**Version:** 1.0

