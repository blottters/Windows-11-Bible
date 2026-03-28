# Health Check Commands & Thresholds

All commands run via `mcp__Windows-MCP__PowerShell`. Each check includes the command, what to flag, and severity.

---

## TIER 1 — QUICK SCAN (6 checks)

### 1. Memory & CPU — Top 10 Processes

```powershell
Get-Process | Sort-Object WorkingSet64 -Descending |
  Select-Object -First 10 Name, @{N='RAM_MB';E={[math]::Round($_.WorkingSet64/1MB)}}, CPU |
  Format-Table -AutoSize
```

**Flags:**
- 🔴 CRITICAL: Any single process using >4GB RAM
- ⚠️ WARNING: Any single process using >2GB RAM
- ⚠️ WARNING: Vmmem in top 5 (WSL memory leak candidate)
- ⚠️ WARNING: MsMpEng using >1GB (Defender scan stuck)

### 2. Disk Space — All Volumes

```powershell
Get-Volume | Where-Object {$_.DriveLetter} |
  Select-Object DriveLetter,
    @{N='FreeGB';E={[math]::Round($_.SizeRemaining/1GB,1)}},
    @{N='TotalGB';E={[math]::Round($_.Size/1GB,1)}},
    @{N='PctFree';E={[math]::Round(($_.SizeRemaining/$_.Size)*100,1)}} |
  Format-Table -AutoSize
```

**Flags:**
- 🔴 CRITICAL: Any drive below 5% free
- ⚠️ WARNING: Any drive below 10% free

### 3. Critical Services — 12 Core Services

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

**Flags:**
- 🔴 CRITICAL: WinDefend or EventLog not running
- ⚠️ WARNING: Any other service not in "Running" state

| Service | Purpose |
|---------|---------|
| wuauserv | Windows Update |
| WinDefend | Windows Defender |
| Schedule | Task Scheduler |
| Winmgmt | WMI (system management) |
| LanmanWorkstation | SMB client (network shares) |
| Dhcp | DHCP client (IP address) |
| Dnscache | DNS resolver cache |
| EventLog | Windows Event Log |
| PlugPlay | Plug and Play (device detection) |
| Power | Power management |
| Spooler | Print spooler |
| AudioSrv | Windows Audio |

### 4. Pending Reboot Check

```powershell
$reboot = $false
$reasons = @()
if (Test-Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending') {
    $reboot = $true; $reasons += 'Component Based Servicing'
}
if (Test-Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired') {
    $reboot = $true; $reasons += 'Windows Update'
}
if (Test-Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager') {
    $pfrn = (Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager' -Name PendingFileRenameOperations -ErrorAction SilentlyContinue).PendingFileRenameOperations
    if ($pfrn) { $reboot = $true; $reasons += 'Pending File Rename Operations' }
}
if ($reboot) { "REBOOT PENDING — Reasons: $($reasons -join ', ')" }
else { "No pending reboot." }
```

**Flags:**
- ⚠️ WARNING: Any reboot pending

### 5. System Uptime

```powershell
$boot = (Get-CimInstance Win32_OperatingSystem).LastBootUpTime
$uptime = (Get-Date) - $boot
"Last boot: $($boot.ToString('yyyy-MM-dd HH:mm:ss'))"
"Uptime: $($uptime.Days) days, $($uptime.Hours) hours"
```

**Flags:**
- ⚠️ WARNING: Uptime > 14 days (memory leaks compound, updates stall)
- 🔴 CRITICAL: Uptime > 30 days

### 6. Windows Defender Status

```powershell
try {
    $def = Get-MpComputerStatus
    $age = ((Get-Date) - $def.AntivirusSignatureLastUpdated).TotalHours
    [PSCustomObject]@{
        RealTimeProtection = $def.RealTimeProtectionEnabled
        DefinitionAge_Hours = [math]::Round($age, 1)
        LastFullScan = $def.FullScanEndTime
        LastQuickScan = $def.QuickScanEndTime
        ThreatDetected = $def.ThreatDetected
    } | Format-List
} catch { "Defender status check failed: $_" }
```

**Flags:**
- 🔴 CRITICAL: RealTimeProtection = False
- ⚠️ WARNING: Definition age > 72 hours (3 days)
- ⚠️ WARNING: No quick scan in last 7 days

---

## TIER 2 — STANDARD SCAN (Tier 1 + 6 more checks)

### 7. Windows Update Status

```powershell
try {
    $Session = New-Object -ComObject Microsoft.Update.Session
    $Searcher = $Session.CreateUpdateSearcher()
    $Results = $Searcher.Search("IsInstalled=0")
    if ($Results.Updates.Count -eq 0) { "No pending updates." }
    else {
        "Pending updates: $($Results.Updates.Count)"
        $Results.Updates | ForEach-Object { "  - $($_.Title)" }
    }
} catch { "Windows Update COM check failed: $_" }
```

**Flags:**
- ⚠️ WARNING: Any pending security updates
- 🔴 CRITICAL: Pending updates > 30 days old

### 8. Recent Event Log Errors — Last 4 Hours

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='System','Application';
    Level=1,2;
    StartTime=(Get-Date).AddHours(-4)
} -MaxEvents 20 -ErrorAction SilentlyContinue |
  Select-Object TimeCreated, LogName, Id, LevelDisplayName,
    @{N='Source';E={$_.ProviderName}},
    @{N='Msg';E={$_.Message.Substring(0,[Math]::Min(120,$_.Message.Length))}} |
  Format-Table -Wrap
```

**Flags:**
- 🔴 CRITICAL: Any Level 1 (Critical) events
- ⚠️ WARNING: Same Event ID appearing 3+ times (recurring issue)

### 9. WSL/Docker Status

```powershell
$vmmem = Get-Process -Name "Vmmem" -ErrorAction SilentlyContinue
if ($vmmem) {
    "Vmmem running — RAM: $([math]::Round($vmmem.WorkingSet64/1MB))MB, CPU: $([math]::Round($vmmem.CPU,1))s"
} else {
    "Vmmem not running (WSL idle or not installed)"
}
wsl --list --verbose 2>$null
```

**Flags:**
- ⚠️ WARNING: Vmmem using >2GB RAM
- 🔴 CRITICAL: Vmmem using >4GB RAM (memory leak — suggest `wsl --shutdown`)

### 10. Startup Programs — Bloat Check

```powershell
$startup = Get-CimInstance Win32_StartupCommand | Select-Object Name, Command, Location
$count = ($startup | Measure-Object).Count
"Startup programs: $count"
$startup | Format-Table -AutoSize -Wrap
```

**Flags:**
- ⚠️ WARNING: More than 15 startup programs
- 🔴 CRITICAL: More than 25 startup programs (significantly impacts boot time)

### 11. Driver Problems — Device Manager Errors

```powershell
$problemDevices = Get-PnpDevice | Where-Object { $_.Status -ne 'OK' -and $_.Status -ne 'Unknown' } |
  Select-Object Status, Class, FriendlyName, InstanceId |
  Format-Table -AutoSize -Wrap
$count = ($problemDevices | Measure-Object).Count
if ($count -eq 0) { "All devices OK." }
else { "Problem devices: $count"; $problemDevices }
```

**Flags:**
- ⚠️ WARNING: Any device with Status = Error or Degraded
- 🔴 CRITICAL: Network adapter or display adapter in error state

### 12. Disk Health — SMART / Physical Disk

```powershell
Get-PhysicalDisk | Select-Object FriendlyName, MediaType, BusType, HealthStatus,
  OperationalStatus, @{N='SizeGB';E={[math]::Round($_.Size/1GB,1)}} |
  Format-Table -AutoSize
```

**Flags:**
- 🔴 CRITICAL: HealthStatus = "Warning" or "Unhealthy" (drive failing — back up immediately)
- ⚠️ WARNING: OperationalStatus != "OK"

---

## TIER 3 — DEEP SCAN (Tier 2 + 8 more checks)

### 13. Temp File Bloat

```powershell
$paths = @{
    'User Temp' = $env:TEMP
    'Windows Temp' = "$env:SystemRoot\Temp"
    'SoftwareDistribution' = "$env:SystemRoot\SoftwareDistribution\Download"
    'Prefetch' = "$env:SystemRoot\Prefetch"
}
$paths.GetEnumerator() | ForEach-Object {
    $size = 0
    if (Test-Path $_.Value) {
        $size = (Get-ChildItem $_.Value -Recurse -ErrorAction SilentlyContinue |
                 Measure-Object Length -Sum -ErrorAction SilentlyContinue).Sum
    }
    [PSCustomObject]@{
        Location = $_.Key
        Path = $_.Value
        SizeMB = [math]::Round($size/1MB, 1)
    }
} | Format-Table -AutoSize
```

**Flags:**
- ⚠️ WARNING: Any single temp location > 2GB
- 🔴 CRITICAL: Combined temp > 10GB

### 14. Firewall Status — All 3 Profiles

```powershell
Get-NetFirewallProfile | Select-Object Name, Enabled,
  DefaultInboundAction, DefaultOutboundAction, LogFileName |
  Format-Table -AutoSize
```

**Flags:**
- 🔴 CRITICAL: Any profile where Enabled = False
- ⚠️ WARNING: DefaultInboundAction = Allow on any profile

### 15. Failed Scheduled Tasks — Last 24 Hours

```powershell
Get-ScheduledTask | Where-Object { $_.State -ne 'Disabled' } | ForEach-Object {
    $info = $_ | Get-ScheduledTaskInfo -ErrorAction SilentlyContinue
    if ($info -and $info.LastRunTime -gt (Get-Date).AddHours(-24) -and $info.LastTaskResult -ne 0) {
        [PSCustomObject]@{
            TaskName = $_.TaskName
            TaskPath = $_.TaskPath
            LastRun = $info.LastRunTime
            Result = "0x{0:X}" -f $info.LastTaskResult
        }
    }
} | Format-Table -AutoSize
```

**Flags:**
- ⚠️ WARNING: Any task with non-zero exit code in last 24h
- 🔴 CRITICAL: System-critical tasks failing (e.g., \Microsoft\Windows\WindowsUpdate\*)

### 16. Network Connectivity

```powershell
$gateway = (Get-NetRoute -DestinationPrefix '0.0.0.0/0' -ErrorAction SilentlyContinue |
            Select-Object -First 1).NextHop
$results = @()
if ($gateway) {
    $ping = Test-Connection $gateway -Count 2 -ErrorAction SilentlyContinue
    $results += [PSCustomObject]@{
        Test = "Gateway Ping ($gateway)"
        Status = if ($ping) { "OK — $([math]::Round(($ping.Latency | Measure-Object -Average).Average))ms avg" } else { "FAILED" }
    }
} else {
    $results += [PSCustomObject]@{ Test = "Gateway"; Status = "NO DEFAULT GATEWAY" }
}
$dns = Resolve-DnsName "dns.msftncsi.com" -ErrorAction SilentlyContinue
$results += [PSCustomObject]@{
    Test = "DNS Resolution"
    Status = if ($dns) { "OK — resolved to $($dns[0].IPAddress)" } else { "FAILED" }
}
$web = try {
    $r = Invoke-WebRequest "http://www.msftconnecttest.com/connecttest.txt" -TimeoutSec 5 -UseBasicParsing
    if ($r.Content -match "Microsoft Connect Test") { "OK" } else { "UNEXPECTED RESPONSE" }
} catch { "FAILED — $_" }
$results += [PSCustomObject]@{ Test = "Internet Access"; Status = $web }
$results | Format-Table -AutoSize
```

**Flags:**
- 🔴 CRITICAL: Gateway ping fails (no local network)
- 🔴 CRITICAL: DNS resolution fails
- ⚠️ WARNING: Internet access fails but gateway/DNS work (proxy or firewall issue)
- ⚠️ WARNING: Gateway latency > 50ms (local network congestion)

### 17. Pagefile / Virtual Memory Health

```powershell
$os = Get-CimInstance Win32_OperatingSystem
$totalPhysicalMB = [math]::Round($os.TotalVisibleMemorySize/1KB)
$freePhysicalMB = [math]::Round($os.FreePhysicalMemory/1KB)
$totalVirtualMB = [math]::Round($os.TotalVirtualMemorySize/1KB)
$freeVirtualMB = [math]::Round($os.FreeVirtualMemory/1KB)
$commitPct = [math]::Round((($totalVirtualMB - $freeVirtualMB) / $totalVirtualMB) * 100, 1)
$pagefile = Get-CimInstance Win32_PageFileUsage -ErrorAction SilentlyContinue |
  Select-Object Name, @{N='AllocatedMB';E={$_.AllocatedBaseSize}},
    @{N='CurrentUsageMB';E={$_.CurrentUsage}},
    @{N='PeakUsageMB';E={$_.PeakUsage}}
[PSCustomObject]@{
    PhysicalRAM_MB = $totalPhysicalMB
    FreeRAM_MB = $freePhysicalMB
    CommitCharge_Pct = "$commitPct%"
    VirtualTotal_MB = $totalVirtualMB
    VirtualFree_MB = $freeVirtualMB
} | Format-List
$pagefile | Format-Table -AutoSize
```

**Flags:**
- 🔴 CRITICAL: Commit charge > 90% (approaching crash threshold)
- ⚠️ WARNING: Commit charge > 75%
- ⚠️ WARNING: Free physical RAM < 500MB

### 18. Reliability Monitor — Stability Index

```powershell
try {
    $rel = Get-CimInstance Win32_ReliabilityStabilityMetrics |
      Sort-Object TimeGenerated -Descending | Select-Object -First 1
    $score = [math]::Round($rel.SystemStabilityIndex, 2)
    "Reliability Index: $score / 10.00"
    "Measured: $($rel.TimeGenerated.ToString('yyyy-MM-dd'))"
    $recent = Get-CimInstance Win32_ReliabilityRecords |
      Where-Object { $_.TimeGenerated -gt (Get-Date).AddDays(-7) -and $_.EventIdentifier -ne 0 } |
      Select-Object -First 10 TimeGenerated, SourceName,
        @{N='Event';E={$_.Message.Substring(0,[Math]::Min(100,$_.Message.Length))}}
    if ($recent) { "Recent reliability events (7 days):"; $recent | Format-Table -Wrap }
    else { "No reliability events in last 7 days." }
} catch { "Reliability data unavailable: $_" }
```

**Flags:**
- 🔴 CRITICAL: Score below 3.0
- ⚠️ WARNING: Score below 5.0
- ⚠️ WARNING: More than 5 reliability events in 7 days

### 19. Certificate Expiry — LocalMachine\My

```powershell
$certs = Get-ChildItem Cert:\LocalMachine\My -ErrorAction SilentlyContinue |
  Select-Object Subject, NotAfter,
    @{N='DaysLeft';E={[math]::Round(($_.NotAfter - (Get-Date)).TotalDays,0)}},
    @{N='Expired';E={$_.NotAfter -lt (Get-Date)}}
if ($certs) {
    $certs | Sort-Object NotAfter | Format-Table -AutoSize
    $expired = $certs | Where-Object { $_.Expired }
    $expiring = $certs | Where-Object { -not $_.Expired -and $_.DaysLeft -lt 30 }
    if ($expired) { "EXPIRED CERTS: $($expired.Count)" }
    if ($expiring) { "EXPIRING SOON (<30 days): $($expiring.Count)" }
} else { "No certificates in LocalMachine\My store." }
```

**Flags:**
- 🔴 CRITICAL: Any expired certificate
- ⚠️ WARNING: Any certificate expiring within 30 days

### 20. GPU / Display Driver Crashes — TDR Events

```powershell
$tdr = Get-WinEvent -FilterHashtable @{
    LogName='System';
    ProviderName='Display';
    StartTime=(Get-Date).AddHours(-24)
} -MaxEvents 10 -ErrorAction SilentlyContinue
if ($tdr) {
    "Display driver events in last 24h: $($tdr.Count)"
    $tdr | Select-Object TimeCreated, Id, LevelDisplayName,
      @{N='Msg';E={$_.Message.Substring(0,[Math]::Min(120,$_.Message.Length))}} |
      Format-Table -Wrap
} else {
    "No display driver events in last 24 hours."
}
$tdrReg = Get-WinEvent -FilterHashtable @{
    LogName='System';
    Id=14;
    StartTime=(Get-Date).AddDays(-7)
} -MaxEvents 5 -ErrorAction SilentlyContinue
if ($tdrReg) {
    "TDR (Timeout Detection Recovery) events in last 7 days: $($tdrReg.Count)"
    $tdrReg | Select-Object TimeCreated,
      @{N='Msg';E={$_.Message.Substring(0,[Math]::Min(120,$_.Message.Length))}} |
      Format-Table -Wrap
}
```

**Flags:**
- 🔴 CRITICAL: TDR events in last 24h (active driver instability)
- ⚠️ WARNING: TDR events in last 7 days (intermittent GPU issue)

---

## Tier Summary

| Tier | Name | Checks | Approx Time |
|------|------|--------|-------------|
| 1 | Quick Scan | 1-6 | ~30 seconds |
| 2 | Standard Scan | 1-12 | ~2 minutes |
| 3 | Deep Scan | 1-20 | ~5 minutes |

Tier 2 includes all of Tier 1. Tier 3 includes all of Tier 2.
