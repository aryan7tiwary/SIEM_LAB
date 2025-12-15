**Suspicious Process Execution Detection:**
```
# Save as: "Suspicious PowerShell Activity"
index=wineventlog EventCode=1 (Image="*powershell.exe" OR Image="*cmd.exe") 
(CommandLine="*-encodedcommand*" OR CommandLine="*invoke-expression*" OR CommandLine="*downloadstring*")
| stats count by ComputerName, User, CommandLine
| where count > 0
```

**Failed Authentication:**
```
# Save as: "Multiple Failed Logins"  
index=wineventlog EventCode=4625
| bucket _time span=5m
| stats count by _time, Account_Name, Workstation_Name
| where count >= 5
```

**New Service Installation:**
```
# Save as: "New Service Installed"
index=wineventlog EventCode=7045
| eval Service_Name=Service_File_Name
| stats count by ComputerName, Service_Name, Account_Name
```

