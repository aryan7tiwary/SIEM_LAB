# Troubleshoot for Windows Event Logs not showing up on SIEM Dashboard

- Step 1: Check if Forwarder is running
```
# Check if service is running
sc query SplunkForwarder

# If not running, start it
net start SplunkForwarder

# Navigate to Splunk directory
cd "C:\Program Files\SplunkUniversalForwarder\bin"

# Check forwarder configuration
splunk status
```

- Step 2: Check connectivity
```
# Test if WINCLIENT can reach SIEM
ping 192.168.142.12

# Test Splunk receiving port
telnet 192.168.142.12 9997
# Should connect successfully
```

- Step 3: Check for forward server configuration
```
cd "C:\Program Files\SplunkUniversalForwarder\bin"

# Check forward server configuration
splunk list forward-server -auth admin:P@ssw0rd123!
# Should show: 192.168.142.12:9997 as "Active"

# If not configured, add it:
splunk add forward-server 192.168.142.12:9997 -auth admin:P@ssw0rd123!
```

- Step 4: Check for WinEventLog entries
```
# List current monitors
splunk list monitor -auth admin:P@ssw0rd123!
# Should show WinEventLog entries

# Check inputs.conf file exists
type "C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf"
```

If not there by default, create it:
```
# Create the local directory if it doesn't exist
mkdir "C:\Program Files\SplunkUniversalForwarder\etc\system\local"

# Create inputs.conf
notepad "C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf"
```

Add this content to input.conf:
```
[WinEventLog://Application]
disabled = 0
index = wineventlog

[WinEventLog://Security]  
disabled = 0
index = wineventlog

[WinEventLog://System]
disabled = 0
index = wineventlog
```

- Step 5: Check current service account
```
# Check current service account
sc qc SplunkForwarder

# If not LocalSystem, change it:
sc config SplunkForwarder obj= LocalSystem
net stop SplunkForwarder
net start SplunkForwarder
```

- Step 6: Check Windows Firewall
```
# Check Windows Firewall status
netsh advfirewall show allprofiles

# Allow outbound connections to SIEM
netsh advfirewall firewall add rule name="Splunk Forwarder Out" dir=out action=allow protocol=TCP remoteport=9997
```

- Step 7: Restart and Verify
```
# Restart Universal Forwarder service
net stop SplunkForwarder
net start SplunkForwarder

# Wait 2-3 minutes, then check logs on forwarder
type "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" | findstr -i "error\|tcpout\|connect"
```
