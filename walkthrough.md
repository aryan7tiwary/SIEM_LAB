# Phase 1 - Planning and Topology

#### Networking
```
┌─────────────────────────────────────────┐
│              HOST LAPTOP                │
│  ┌─────────────────────────────────────┐│
│  │       VMware NAT Network            ││
│  │        192.168.142.0/24             ││
│  │            (VMnet8)                 ││
│  │                                     ││
│  │  ┌──────────┐  ┌──────────┐         ││
│  │  │WINCLIENT │  │  WINDC   │         ││
│  │  │.142.11   │  │ .142.10  │         ││
│  │  │(Domain   │  │(lab.local│         ││
│  │  │ Member)  │  │   DC)    │         ││
│  │  └─────┬────┘  └────┬─────┘         ││
│  │        │            │               ││
│  │        └────────┬───┘               ││
│  │                 │                   ││
│  │           ┌─────┴─────┐             ││
│  │           │   SIEM    │             ││
│  │           │ .142.12   │             ││
│  │           │(Splunk)   │             ││
│  │           └───┬───────┘             ││
│  │               │                     ││
│  │               ▼                     ││
│  │        Internet Access              ││
│  └─────────────────────────────────────┘│
└─────────────────────────────────────────┘
```

#### Resource Planning & VM Specifications

|VM Name       | OS                    | vCPU | RAM  | Disk  | Purpose
|--------------|-----------------------|------|------|-------|------------------
|WINCLIENT     | Windows 10/11 Pro     | 2    | 4GB  | 60GB  | User workstation
|WINDC         | Server 2019/2022      | 2    | 4GB  | 60GB  | Domain Controller
|SIEM          | Ubuntu 22.04 Server   | 2    | 6GB  | 80GB  | Splunk/ELK host
|TOTAL USAGE:  |                       |  6   | 14GB | 200GB |
|HOST RESERVE: |                       |  -   | 2GB  | -     | For host OS


#### Common Pitfalls
- **Insufficient RAM**: Use snapshot strategy - run 2 VMs max simultaneously
- **Disk space**: Enable disk compression in VMware, use thin provisioning
- **Network isolation concerns**: Host-Only = maximum isolation, NAT = allows package downloads

---
# Phase 2 - Base VM Creation

#### 1. Create Windows Domain Controller (WINDC)
```
1. File → New Virtual Machine
2. Select "Custom (advanced)" → Next
3. Hardware compatibility: Workstation 17.x → Next
4. Guest OS installation: "I will install the operating system later" → Next
5. Guest OS: "Microsoft Windows" 
   Version: "Windows Server 2019" (or 2022) → Next
6. VM Name: "WINDC"
   Location: [Choose your preferred path] → Next
7. Processor: 2 processors, 2 cores per processor → Next
8. Memory: 4096 MB (4GB) → Next
9. Network: "Use host-only networking" → Next
10. I/O Controller: "LSI Logic SAS" → Next
11. Virtual Disk: "Create a new virtual disk" → Next
12. Disk Type: "SCSI" → Next
13. Disk Size: 60 GB, "Store virtual disk as single file" → Next
14. Disk File: Default location → Next
15. Click "Customize Hardware"
```

#### 2. Hardware Customization
```
- Memory: Confirm 4096 MB
- Processors: 2 cores
- Network Adapter: Verify "Host-only: VMnet1"
- CD/DVD: "Use ISO image file" → Browse to Windows Server ISO
- Remove: Floppy, USB Controller, Sound Card, Printer (not needed)
- Click "Close" → Finish
```

## 3. Install Windows Server
```
1. Power on WINDC VM
2. Windows Setup:
   - Language: English (United States)
   - Time/Currency: Your region
   - Keyboard: US
3. Click "Install now"
4. Edition: "Windows Server 2019 Standard (Desktop Experience)"
5. License: Accept terms
6. Installation type: "Custom: Install Windows only"
7. Drive: Select the 60GB disk → Next
8. Wait for installation (15-20 minutes)
```

```
1. Set Administrator password: "P@ssw0rd123!" (strong but memorable for lab)
2. Login with Administrator account
3. Server Manager should auto-launch
```
## 4. Configure Static IP
```
1. Right-click Network icon → "Open Network & Internet settings"
2. "Change adapter options"
3. Right-click "Ethernet0" → Properties
4. Select "Internet Protocol Version 4 (TCP/IPv4)" → Properties
5. "Use the following IP address":
   - IP: 192.168.142.10
   - Subnet: 255.255.255.0
   - Gateway: 192.168.142.2
   - Preferred DNS: 192.168.142.2
   - Alternate DNS: 8.8.8.8
6. Click OK → Close
```

```
1. Server Manager → Local Server
2. Click "Computer name" link
3. "Change..." button
4. Computer name: "WINDC"
5. Click OK → Restart when prompted
```

**Verification:**
```
# Open Command Prompt as Administrator
ipconfig /all
# Should show 192.168.142.10

ping 192.168.254.2
# Should respond (VMware host adapter)

nslookup google.com
# Should resolve (confirming DNS connectivity)
```

#### WINDC Network Configuration
```
Ethernet adapter Ethernet0:

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::392d:fdaa:845a:ed9d%13
   IPv4 Address. . . . . . . . . . . : 192.168.254.10
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.254.1
```

---
# Phase 3 - AD DC Promotion
#### 1. Install AD DC Service Role
```
1. Server Manager should be open (if not: Win + R → servermanager)
2. Click "Add roles and features"
3. Before You Begin: Click "Next"
4. Installation Type: "Role-based or feature-based installation" → Next
5. Server Selection: Select "WINDC" (should be highlighted) → Next
6. Server Roles: 
   Check "Active Directory Domain Services"
   - Popup appears: "Add features that are required for Active Directory Domain Services?"
   - Click "Add Features"
7. Features: Keep defaults → Next
8. AD DS: Read the information → Next
9. Confirmation: 
   Check "Restart the destination server automatically if required"
   Click "Install"
```

#### 2. Promote to AD Domain Controller
```
1. Look for yellow warning flag in Server Manager toolbar
2. Click the notification flag
3. Click "Promote this server to a domain controller"
```

```
1. Select "Add a new forest" (radio button)
2. Root domain name: "lab.local"
3. Click "Next"
```

```
1. Forest functional level: "Windows Server 2016" (default - fine)
2. Domain functional level: "Windows Server 2016" (default - fine)
3. Specify domain controller capabilities:
   Domain Name System (DNS) server (auto-checked)
   Global Catalog (GC) (auto-checked)
   Read only domain controller (RODC) (leave unchecked)
4. Site name: "Default-First-Site-Name" (auto-filled)
5. Directory Services Restore Mode (DSRM) password: "P@ssw0rd123!"
   - Confirm password: "P@ssw0rd123!"
6. Click "Next"
```

```
1. Warning about DNS delegation appears - this is NORMAL for lab
2. Message: "A delegation for this DNS server cannot be created because..."
3. This warning is safe to ignore in isolated lab environment
4. Click "Next"
```

```
1. NetBIOS domain name: "LAB" (auto-generated from lab.local)
2. Keep this default
3. Click "Next"
```

```
1. Database folder: C:\Windows\NTDS (default)
2. Log files folder: C:\Windows\NTDS (default)  
3. SYSVOL folder: C:\Windows\SYSVOL (default)
4. Keep defaults → Click "Next"
```

**Review:**
```
1. Review all settings:
   - New forest: lab.local
   - NetBIOS: LAB
   - DNS integrated: Yes
   - Functional levels: 2016
2. Optional: Click "View Script" to see PowerShell equivalent
3. Click "Next"
```

```
1. Prerequisites will be verified (~2-3 minutes)
2. Expected warnings (safe to ignore):
   - "A delegation for this DNS server cannot be created"
   - "Windows Server 2019 domain controllers have a default..."
3. Look for: "All prerequisite checks passed successfully"
4. NO red errors should appear
5. Click "Install"
```

#### 3. Post Installation Verification and Configuration
**Login Changes:**
- Login screen now shows: "LAB\Administrator"
- Password remains: P@ssw0rd123!

**Verification:**
```
1. Server Manager → Tools → "Active Directory Users and Computers"
2. Expand "lab.local" domain tree
3. You should see:
   - Builtin (container)
   - Computers (OU)
   - Domain Controllers (OU)
   - ForeignSecurityPrincipals (container)
   - Managed Service Accounts (container)
   - Users (OU)
```
<img width="937" height="634" alt="Pasted image 20250825005053" src="https://github.com/user-attachments/assets/e7ac8a57-b741-4e75-8a95-bc2ea7e3abf5" />


```
# Run these commands in Command Prompt
nslookup lab.local
# Should return: 192.168.142.10

nslookup WINDC.lab.local
# Should return: 192.168.142.10

ping lab.local
# Should respond from 192.168.142.10
```
<img width="811" height="532" alt="Pasted image 20250825005208" src="https://github.com/user-attachments/assets/701bd5ab-0e94-4242-bc47-0e6b7b19167f" />


**Configuration:**

**Creating Test Users for SIEM Monitoring:**
```
1. Tools → Active Directory Users and Computers
2. Expand lab.local → Right-click "Users" OU → New → User

First User (Standard):
- First name: "John"
- Last name: "Analyst"
- User logon name: "janalyst"
- Password: "Welcome123!"
- User cannot change password
- Password never expires (lab setting)
- Click "Next" → "Finish"

Second User (Admin):
- First name: "Admin"
- Last name: "User"
- User logon name: "adminuser"
- Password: "P@ssw0rd123!"
- User cannot change password
- Password never expires
- Click "Next" → "Finish"

Make adminuser Domain Admin:
- Right-click "adminuser" → Properties → "Member Of" tab
- Click "Add" → Type "Domain Admins" → Check Names → OK
```
<img width="965" height="333" alt="Pasted image 20250825005017" src="https://github.com/user-attachments/assets/6d9edd95-fe97-4b15-b650-882711cdb9e3" />

**VERIFY ENTERPRISE EVENT LOGGING:**
```
1. Win + R → eventvwr
2. Navigate: Windows Logs → Security
3. Look for recent Event IDs:
   - 4648: A logon was attempted using explicit credentials
   - 4672: Special privileges assigned to new logon
   - 4624: An account was successfully logged on
   - 4634: An account was logged off
```

**Do not worry if you are unable to login as local administrator, i.e., WINDC\Administrator. This is a AD security feature:**
**Active Directory Security Model:**
When a server is promoted to Domain Controller, Windows automatically:
1. **Disables Local Administrator Account**: The local WINDC\Administrator becomes disabled
2. **Migrates to Domain Admin**: Your existing admin becomes LAB\Administrator
3. **Security Enhancement**: Prevents confusion between local vs domain accounts
4. **Enterprise Standard**: This is how all production Domain Controllers work.

---
# PHASE 4 — Create Windows Client VM (WINCLIENT)

```
1. File → New Virtual Machine
2. Select "Custom (advanced)" → Next
3. Hardware compatibility: Workstation 17.x → Next
4. Guest OS installation: "I will install the operating system later" → Next
5. Guest OS: "Microsoft Windows"
   Version: "Windows 10 x64" (or Windows 11 x64) → Next
6. VM Name: "WINCLIENT"
   Location: [Choose your preferred path] → Next
7. Firmware type: "BIOS" (same as WINDC for consistency) → Next
8. Processor: 2 processors, 1 core per processor → Next
9. Memory: 4096 MB (4GB) → Next
10. Network: "Use NAT networking" → Next
11. I/O Controller: "LSI Logic SAS" → Next
12. Virtual Disk: "Create a new virtual disk" → Next
13. Disk Type: "SCSI" → Next
14. Disk Size: 45 GB, "Store virtual disk as single file" → Next
15. Disk File: Default location → Next
16. Click "Customize Hardware"
```

#### Initial Setup
```
1. Region: Select your region
2. Keyboard: US
3. Add second keyboard: Skip
4. Connect to internet: "I don't have internet" → Continue with limited setup
5. Create account: "John" (local account for now)
6. Password: "Welcome123!" (will change after domain join)
7. Security questions: Fill out
8. Privacy settings: Accept defaults or minimize
```

```
1. Right-click "This PC" → Properties
2. Click "Rename this PC" 
3. Computer name: "WINCLIENT"
4. Click "Next" → Restart when prompted
```

#### Join lab.local domain:
```
1. Right-click "This PC" → Properties
2. Click "Rename this PC (advanced)"
3. Click "Change..."
4. Select "Domain" radio button
5. Domain: "lab.local"
6. Click OK
7. Credentials prompt:
   - Username: LAB\Administrator
   - Password: P@ssw0rd123!
8. Welcome message: "Welcome to the lab.local domain"
9. Restart when prompted
```

#### Restart and login as:
```
1. Login screen shows "Other user"
2. Username: LAB\janalyst  
3. Password: Welcome123!
4. Should successfully login to domain
```

#### Verification Commands:
```
# Run these in Command Prompt
whoami
# Should show: lab\janalyst

echo %LOGONSERVER%
# Should show: \\WINDC

ping lab.local
# Should respond from 192.168.142.10

ping WINDC.lab.local  
# Should respond from 192.168.142.10
```
<img width="534" height="213" alt="Pasted image 20250825183038" src="https://github.com/user-attachments/assets/56128526-261c-4e84-a5e6-ff12f4ba2b8c" />


#### When viewing EventViewer:
```
Current Status:
├── LAB\janalyst (Standard User) -> Cannot view Security logs
├── Local Administrator -> Can view Security logs  
└── LAB\Administrator (Domain Admin) -> Can view Security logs
```

---
# PHASE 5 - Ubuntu SIEM Server & Splunk Installation

#### WHY UBUNTU + SPLUNK FOR SIEM LAB
- **Industry Standard**: Most enterprise SIEMs run on Linux
- **Splunk Free**: 500MB/day limit perfect for lab (plenty for our 3 VMs)
- **Real-World Skills**: Matches actual SOC analyst environment
- **Performance**: Linux handles log ingestion more efficiently than Windows

#### Create UBUNTU VM SIEM
```
1. File → New Virtual Machine
2. Select "Custom (advanced)" → Next
3. Hardware compatibility: Workstation 17.x → Next
4. Guest OS installation: "I will install the operating system later" → Next
5. Guest OS: "Linux"
   Version: "Ubuntu 64-bit" → Next
6. VM Name: "SIEM"
   Location: [Choose your preferred path] → Next
7. Firmware type: "BIOS" → Next
8. Processor: 2 processors, 1 core per processor → Next
9. Memory: 6144 MB (6GB) → Next
10. Network: "Use NAT networking" → Next
11. I/O Controller: "LSI Logic SAS" → Next
12. Virtual Disk: "Create a new virtual disk" → Next
13. Disk Type: "SCSI" → Next
14. Disk Size: 50 GB, "Store virtual disk as single file" → Next
15. Disk File: Default location → Next
16. Click "Customize Hardware"
```

#### Boot and Install:
```
1. Power on SIEM VM
2. Select "Try or Install Ubuntu Server"
3. Language: English
4. Installer update: Continue without updating (faster)
5. Keyboard: English (US)
6. Installation type: "Ubuntu Server"
7. Network connections: 
   - eth0 DHCPv4 (should get 192.168.142.x automatically)
   - Continue
8. Proxy: Leave blank → Continue
9. Mirror: Use default → Continue
10. Guided storage: Use entire disk → Continue
11. Storage configuration: Confirm → Continue
12. Profile setup:
    - Name: "SOC Analyst"
    - Server name: "siem"
    - Username: "analyst" 
    - Password: "P@ssw0rd123!"
13. SSH Setup: Install OpenSSH server
14. Snaps: Skip all → Continue
15. Wait for installation (~10-15 minutes)
16. Reboot when prompted
```

**Not having GUI is completely NORMAL and exactly what we want! Ubuntu Server doesn't have a GUI by design. SIEM Dashboard can be accessed from any device within the network with credentials.**

#### Configure Network:
```
# Login as analyst
# Check current IP
ip addr show

# Edit netplan configuration
sudo nano /etc/netplan/00-installer-config.yaml

# Replace content with:
network:
  ethernets:
    ens33:
      addresses:
        - 192.168.142.12/24
      gateway4: 192.168.142.2
      nameservers:
        addresses: [192.168.142.10, 192.168.142.2]
  version: 2

# Apply network configuration
sudo netplan apply

# Verify new IP
ip addr show ens33
```

#### Setting Up Splunk on SIEM Ubuntu:
```
cd /tmp
# Use your working Splunk download link
wget -O splunk.tgz "[YOUR_WORKING_DOWNLOAD_LINK]"

# Extract to /opt
sudo tar -xzf splunk.tgz -C /opt/

# Create splunk user
sudo useradd -r -s /bin/bash -d /opt/splunk splunk
sudo chown -R splunk:splunk /opt/splunk

# Start with license acceptance
sudo -u splunk /opt/splunk/bin/splunk start --accept-license
# Enter: admin / P@ssw0rd123! when prompted

# Update Splunk disk thresholds to normal levels
sudo -u splunk tee /opt/splunk/etc/system/local/server.conf << 'EOF'
[diskUsage]
minFreeSpace = 5000

[diskUsage:dispatch]
minFreeSpace = 2000
EOF

# Enable receiving and services
sudo -u splunk /opt/splunk/bin/splunk enable listen 9997 -auth admin:P@ssw0rd123!
sudo /opt/splunk/bin/splunk enable boot-start -user splunk
sudo -u splunk /opt/splunk/bin/splunk add index wineventlog -auth admin:P@ssw0rd123!

# Configure firewall
sudo ufw allow 8000/tcp
sudo ufw allow 9997/tcp

# Restart to apply settings
sudo -u splunk /opt/splunk/bin/splunk restart
```
**Web Interface Test:**
- Navigate to: `http://192.168.142.12:8000`
- Login: `admin` / `P@ssw0rd123!`
- Should see Splunk Enterprise dashboard
<img width="1450" height="601" alt="Pasted image 20250825224337" src="https://github.com/user-attachments/assets/677509c5-580c-48f1-a9c3-d0df668871dd" />


<img width="1780" height="957" alt="Pasted image 20250825224515" src="https://github.com/user-attachments/assets/7e51a78a-f165-4f31-abc0-a9eca2d654fd" />


---
## PHASE 6 - Install Splunk Universal Forwarders & Collect Windows Logs

#### Setting up Splunk Universal Forwarder on WINDC:
```
1. Open Internet Explorer/Edge on WINDC
2. Navigate to: https://www.splunk.com/en_us/download/universal-forwarder.html
3. Sign up with email (quick process)
4. Download: "Windows 64-bit" MSI package
5. Save to Downloads folder
```

```
1. Run the MSI file as Administrator
2. Installation Wizard:
   - License Agreement: Accept
   - Destination Folder: Default (C:\Program Files\SplunkUniversalForwarder\)
   - User Account: Local System (recommended)
   - Administrator Account: Create new
     - Username: admin
     - Password: P@ssw0rd123!
   - Deployment Server: Skip (leave blank)
   - Receiving Indexer: 192.168.142.12:9997
   - Windows Event Logs: 
     + Application Event Log
     + Security Event Log
     + System Event Log
   - Install
```

```
# Run Command Prompt as Administrator
cd "C:\Program Files\SplunkUniversalForwarder\bin"

# Check status
splunk status

# List configured inputs
splunk list monitor

# Check connection to indexer
splunk list forward-server
```

#### On SIEM Ubuntu server, create Windows Event Index:
```
1. Open Splunk Web: http://192.168.142.12:8000
2. Login: admin / P@ssw0rd123!
3. Settings → Indexes
4. New Index:
   - Index Name: "wineventlog"
   - Max Size: 1000MB
   - App: search
   - Save
```

<img width="1804" height="988" alt="Pasted image 20250826200358" src="https://github.com/user-attachments/assets/fb6fd187-783b-4257-aa36-fc68c27e71e2" />


**Install Splunk Universal Forwarder on WINCLIENT. You'll be able to view Forwarders connected to SIEM Dashboard:**

```
index=_internal source="*metrics.log*" sourcetype="splunkd" fwdType="*" | dedup sourceHost | table sourceHost
```
<img width="1037" height="477" alt="Pasted image 20250827155349" src="https://github.com/user-attachments/assets/6153618c-2902-4b0f-b924-371e4aa31135" />

#### Validate if Windows Event Logs are being forwarded:
```
index=* earliest=-1h | stats count by host, source | sort -count
```

<img width="1797" height="667" alt="Pasted image 20250827160258" src="https://github.com/user-attachments/assets/52400862-b350-45eb-a080-1b99af25f082" />


---
# PHASE 7 - Advanced SIEM Operations

#### Install Sysmon for Enhanced Visibility
**On WINDC (as Administrator):**
```
# Download Sysmon
powershell -Command "Invoke-WebRequest -Uri 'https://download.sysinternals.com/files/Sysmon.zip' -OutFile 'C:\temp\Sysmon.zip'"
powershell -Command "Expand-Archive -Path 'C:\temp\Sysmon.zip' -DestinationPath 'C:\temp\Sysmon'"

# Download SwiftOnSecurity configuration (industry standard)
powershell -Command "Invoke-WebRequest -Uri 'https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml' -OutFile 'C:\temp\sysmonconfig.xml'"

# Install Sysmon with configuration
cd C:\temp\Sysmon
sysmon64.exe -accepteula -i ..\sysmonconfig.xml
```

**Repeat same installation on WINCLIENT.**
#### Configure Universal Forwarder to Collect Sysmon Logs
**Add Sysmon input to inputs.conf on both systems:**
```
# Edit: C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
# Add this section:

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
index = wineventlog
sourcetype = WinEventLog:Sysmon
```

**Restart Universal Forwarders:**
```
net stop SplunkForwarder
net start SplunkForwarder
```

#### Creating SOC Dashboard:
```
┌─────────────────────────────────────────────────────────┐
│                    SOC OVERVIEW DASHBOARD               │
├─────────────────────────────────────────────────────────┤
│ Executive Summary  │  Threat Activity  │  System Health │
│ - Total Events     │  - Active Alerts  │  - Data Sources│ 
│ - Security Score   │  - Failed Logins  │  - Forwarders  │
│ - Incident Count   │  - Malware Detect │  - Disk Usage  │
├─────────────────────────────────────────────────────────┤
│           Real-Time Threat Detection Timeline           │
├─────────────────────────────────────────────────────────┤
│  Top Threats   │  Geographic View  │  User Activity     │
│  - Event Types │  - Attack Sources │  - Privilege Use   │
│  - Attack IPs  │  - Country Map    │  - Admin Actions   │
└─────────────────────────────────────────────────────────┘
```

## **CREATE EXECUTIVE SUMMARY PANELS**
```
1. Navigate: Settings → User Interface → Views
2. Click "New Dashboard"
3. Title: "SOC Security Operations Center"
4. Description: "Real-time security monitoring and threat detection"
```

<img width="1312" height="859" alt="Pasted image 20250829185601" src="https://github.com/user-attachments/assets/409fd5c3-df18-4f6e-94e1-cc0d8a4d2101" />
