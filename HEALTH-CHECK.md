# Windows 11 Health Check — Scheduled Task

**Task ID:** `windows-health-check`
**Schedule:** Every 4 hours (cron: `0 */4 * * *`)
**Output:** `C:\Users\gavin\Documents\Claude\Health-Logs\health-check.log` (rolling) + `latest.txt` (most recent only)
**Notification:** Windows desktop toast after every run

---

## What It Checks

### 1. Memory & CPU — Top 10 Processes
```powershell
Get-Process | Sort-Object WorkingSet64 -Descending |
  Select-Object -First 10 Name, @{N='RAM_MB';E={[math]::Round($_.WorkingSet64/1MB)}}, CPU |
  Format-Table -AutoSize
```
- **Flag:** Any single process using >2GB RAM
- **Watch for:** Vmmem (WSL memory leak), MsMpEng (Defender scan stuck), unexpected processes in top 5

### 2. Disk Space — All Volumes
```powershell
Get-Volume | Where-Object {$_.DriveLetter} |
  Select-Object DriveLetter,
    @{N='FreeGB';E={[math]::Round($_.SizeRemaining/1GB,1)}},
    @{N='TotalGB';E={[math]::Round($_.Size/1GB,1)}},
    @{N='PctFree';E={[math]::Round(($_.SizeRemaining/$_.Size)*100,1)}} |
  Format-Table -AutoSize
```
- **Warning:** Any drive below 10% free
- **Critical:** Below 5% free

### 3. Windows Update Status
```powershell
try {
    $Session = New-Object -ComObject Microsoft.Update.Session
    $Searcher = $Session.CreateUpdateSearcher()
    $Results = $Searcher.Search("IsInstalled=0")
    if ($Results.Updates.Count -eq 0) { "No pending updates." }
    else { $Results.Updates | ForEach-Object { "$($_.Title) — KB$($_.KBArticleIDs)" } }
} catch { "Windows Update COM check failed: $_" }
```
- **Flag:** Any pending security updates

### 4. Critical Services — 12 Core Services
```powershell
$critical = @('wuauserv','WinDefend','Schedule','Winmgmt',
              'LanmanWorkstation','Dhcp','Dnscache','EventLog',
              'PlugPlay','Power','Spooler','AudioSrv')
$critical | ForEach-Object {
    $svc = Get-Service -Name $_ -ErrorAction SilentlyContinue
    if ($svc) {
        [PSCustomObject]@{Name=$svc.Name; Display=$svc.DisplayName; Status=$svc.Status}
    } else {
        [PSCustomObject]@{Name=$_; Display='NOT FOUND'; Status='Missing'}
    }
} | Format-Table -AutoSize
```
- **Flag:** Any service not in "Running" state

| Service | What It Does |
|---------|-------------|
| `wuauserv` | Windows Update |
| `WinDefend` | Windows Defender |
| `Schedule` | Task Scheduler |
| `Winmgmt` | WMI (system management) |
| `LanmanWorkstation` | SMB client (network shares) |
| `Dhcp` | DHCP client (IP address) |
| `Dnscache` | DNS resolver cache |
| `EventLog` | Windows Event Log |
| `PlugPlay` | Plug and Play (device detection) |
| `Power` | Power management |
| `Spooler` | Print spooler |
| `AudioSrv` | Windows Audio |

### 5. Recent Event Log Errors — Last 4 Hours
```powershell
Get-WinEvent -FilterHashtable @{
    LogName='System','Application';
    Level=1,2;
    StartTime=(Get-Date).AddHours(-4)
} -MaxEvents 15 -ErrorAction SilentlyContinue |
  Select-Object TimeCreated, LogName, Id, LevelDisplayName,
    @{N='Source';E={$_.ProviderName}},
    @{N='Msg';E={$_.Message.Substring(0,[Math]::Min(120,$_.Message.Length))}} |
  Format-Table -Wrap
```
- **Critical:** Any Level 1 (Critical) events
- **Warning:** Same Event ID repeating = active issue

### 6. WSL/Docker Status
```powershell
$vmmem = Get-Process -Name "Vmmem" -ErrorAction SilentlyContinue
if ($vmmem) {
    "Vmmem running — RAM: $([math]::Round($vmmem.WorkingSet64/1MB))MB, CPU: $([math]::Round($vmmem.CPU,1))s"
} else {
    "Vmmem not running (WSL idle or not installed)"
}
wsl --list --verbose 2>$null
```
- **Flag:** Vmmem using >4GB RAM → suggest `wsl --shutdown`

---

## Output

### Log File: `C:\Users\gavin\Documents\Claude\Health-Logs\health-check.log`

Rolling log — every check appends. Format:

```
========================================
HEALTH CHECK — 2026-03-28 16:00:00
========================================

[MEMORY & CPU]
<top 10 processes>

[DISK SPACE]
<volume table>

[WINDOWS UPDATE]
<pending updates or "No pending updates.">

[CRITICAL SERVICES]
<service status table>

[EVENT LOG ERRORS]
<recent Critical/Error events>

[WSL/DOCKER]
<Vmmem status + WSL distro list>

[STATUS] ✅ All Clear / ⚠️ Warnings / ❌ Critical
[ISSUES] 0 issue(s) found
<list of flagged items if any>
========================================
```

### Latest File: `C:\Users\gavin\Documents\Claude\Health-Logs\latest.txt`
Overwritten each run — always shows just the most recent check.

### Windows Notification
- All clear: `Health Check ✅ — All systems nominal`
- Warnings: `Health Check ⚠️ — [count] warning(s): [summary]`
- Critical: `Health Check ❌ — [count] critical issue(s): [summary]`

---

## Managing the Task

**Run manually:** Find `windows-health-check` in Scheduled section of Claude sidebar → "Run now"

**Change frequency:** Update the cron expression via `mcp__scheduled-tasks__update_scheduled_task`
- Every 2 hours: `0 */2 * * *`
- Every 6 hours: `0 */6 * * *`
- Once daily at 9am: `0 9 * * *`
- Twice daily (9am + 5pm): `0 9,17 * * *`

**Pause:** Set `enabled: false` on the task

**First run:** Click "Run now" to approve the Windows-MCP tool permissions. After that, all future runs are fully autonomous.
