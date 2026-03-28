<!-- Windows 11 Troubleshooting Bible — Task Scheduler Troubleshooting -->

## 35. Task Scheduler Troubleshooting

### Architecture

Task Scheduler runs as a Windows service (`Schedule`) with its own engine and persistence store. Tasks are defined as XML and stored in:

```
# File system location
C:\Windows\System32\Tasks\              # Root tasks
C:\Windows\System32\Tasks\Microsoft\    # Microsoft built-in tasks

# Registry location (task registration metadata)
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree\
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tasks\
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Boot\
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Logon\
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Plain\
```

### Common Error Codes (Last Run Result)

| Exit Code | Hex | Meaning | Fix |
|-----------|-----|---------|-----|
| 0 | 0x0 | Success | — |
| 1 | 0x1 | Incorrect function or general error | Check the action command path and arguments |
| 2 | 0x2 | File not found | The executable/script path in the action is wrong |
| 10 | 0xA | Environment incorrect | Usually wrong user context or missing env vars |
| 267008 | 0x41300 | Task is currently running | Normal — means it hasn't finished yet |
| 267009 | 0x41301 | Task is queued | Waiting for a trigger condition to be met |
| 267010 | 0x41302 | Task has not yet run | Never been triggered since creation |
| 267011 | 0x41303 | No more instances of task | Task reached its repetition limit |
| 267012 | 0x41304 | Task terminated by user | Someone manually stopped it |
| 267014 | 0x41306 | Task was terminated because missed a start | Start-when-available wasn't set and the trigger time passed |
| 2147750671 | 0x8004130F | Credentials became corrupted or expired | Delete and recreate the task, re-enter credentials |
| 2147750687 | 0x8004131F | Unknown — general service error | Restart `Schedule` service, check Event Log |
| 2147942401 | 0x80070001 | Incorrect function | Exe returns non-zero — check the script itself |
| 2147942402 | 0x80070002 | System cannot find the file | Path is wrong or the user context can't see it |
| 2147942405 | 0x80070005 | Access denied | Task user lacks permissions to run the action or write output |
| 2147943645 | 0x800704DD | User not logged in | "Run only when user is logged on" selected but nobody's logged in |
| 2147946720 | 0x80071060 | Task XML parse error | Export task, validate XML, fix and re-import |

### Corrupted Task Cleanup

**Symptoms:** Task Scheduler won't open, shows "The task image is corrupt or has been tampered with," or tasks fail silently.

```powershell
# Step 1: Identify the corrupt task
# Check Event Viewer: Microsoft-Windows-TaskScheduler/Operational
# Event ID 413: "Task registered but with errors"
# Event ID 414: "Engine received message to start task but instance already running"

# Step 2: Find orphaned entries (registry has entry, file system doesn't, or vice versa)
# Export registry tree
reg export "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree" C:\temp\taskcache.reg

# Compare against file system
dir C:\Windows\System32\Tasks /s /b > C:\temp\taskfiles.txt

# Step 3: For a specific corrupt task, remove it manually
# Delete the file
del "C:\Windows\System32\Tasks\<TaskName>"

# Delete registry entries (find the GUID in Tree\<TaskName>, then remove from Tasks\{GUID})
# Use regedit to navigate to:
# HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree\<TaskName>
# Note the "Id" value (a GUID), then delete:
# - The Tree\<TaskName> key
# - The Tasks\{GUID} key
# - Any entry in Boot\, Logon\, or Plain\ with that GUID

# Step 4: Restart the service
net stop Schedule && net start Schedule
```

### PowerShell Task Management

```powershell
# List all tasks with their last result
Get-ScheduledTask | Get-ScheduledTaskInfo |
  Select-Object TaskName, LastRunTime, LastTaskResult, NextRunTime |
  Sort-Object LastRunTime -Descending

# Find all failed tasks (non-zero last result, excluding "running" codes)
Get-ScheduledTask | Get-ScheduledTaskInfo |
  Where-Object { $_.LastTaskResult -ne 0 -and $_.LastTaskResult -ne 267009 } |
  Select-Object TaskName, LastTaskResult, @{N='HexResult';E={'0x{0:X}' -f $_.LastTaskResult}}

# Export a task to XML for inspection
Export-ScheduledTask -TaskName "MyTask" | Out-File C:\temp\task.xml

# Re-register a task from XML
Register-ScheduledTask -TaskName "MyTask" -Xml (Get-Content C:\temp\task.xml -Raw)

# Create a task that runs a script at logon
$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-NoProfile -File C:\Scripts\startup.ps1"
$trigger = New-ScheduledTaskTrigger -AtLogon
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries
Register-ScheduledTask -TaskName "StartupScript" -Action $action -Trigger $trigger -Settings $settings -RunLevel Highest

# Run with SYSTEM account (no credential storage issues)
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest
Register-ScheduledTask -TaskName "SystemTask" -Action $action -Trigger $trigger -Principal $principal -Settings $settings

# Disable all tasks in a folder
Get-ScheduledTask -TaskPath "\Microsoft\Windows\Defrag\" | Disable-ScheduledTask
```

### Common Issues & Fixes

**Task runs but does nothing:** Usually a working directory problem. The task runs in `C:\Windows\System32` by default. Set `-WorkingDirectory` in `New-ScheduledTaskAction` or specify the full path to everything.

**Task runs as wrong user:** `Run whether user is logged on or not` stores credentials encrypted. Password changes = task breaks silently. Use SYSTEM account when possible, or use Group Managed Service Accounts (gMSA) in domain environments.

**Task won't run after Windows Update:** Some updates reset the Task Scheduler service configuration. Check `services.msc` → Task Scheduler → Startup type should be `Automatic`.

**Task Scheduler GUI is blank/won't open:** Delete `C:\Windows\System32\Tasks\Microsoft\Windows\UpdateOrchestrator\Schedule Scan` if it's corrupted (known Windows 11 bug). Or run `sfc /scannow` to repair.

---

