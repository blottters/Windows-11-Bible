<!-- Windows 11 Troubleshooting Bible — Print System, Disk/Storage Management, Remote Desktop -->

## 30. Print System Troubleshooting

### Clear Stuck Print Queue

```cmd
:: Stop spooler
net stop spooler

:: Delete all print jobs
del /q /f %SYSTEMROOT%\System32\spool\PRINTERS\*

:: Restart spooler
net start spooler
```

### Print Spooler Crashes

If the spooler keeps crashing:

```cmd
:: Enable driver isolation (per-driver, so one bad driver doesn't crash everything)
:: Registry:
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Print" /v DriverIsolationOverride /t REG_DWORD /d 2 /f
:: 0 = use driver setting, 1 = disable isolation, 2 = force isolation for all

:: Remove problematic printer driver
printui /s /t2
:: Opens Print Server Properties → Drivers tab → select and remove

:: Or from cmd
pnputil /delete-driver oem##.inf /force
```

### Network Printer Issues

```cmd
:: Add a network printer
rundll32 printui.dll,PrintUIEntry /in /n "\\server\printername"

:: List all installed printers
wmic printer list brief

:: Check printer ports
wmic printer get name, portname

:: Test connection to network printer
ping print-server-name
net view \\print-server-name
```

### Printer Troubleshooter

```cmd
msdt.exe /id PrinterDiagnostic
```

---

## 31. Disk & Storage Management

### chkdsk — Disk Integrity Check

```cmd
:: Check C: for errors (read-only scan)
chkdsk C:

:: Fix errors (requires reboot if C: is in use)
chkdsk C: /f

:: Fix errors AND scan for bad sectors (thorough, slow)
chkdsk C: /f /r

:: Fix errors with offload to spare area (SSD-friendly)
chkdsk C: /f /offlinescanandfix

:: Schedule check on next boot
chkdsk C: /f
:: When prompted "schedule for next restart?" → Y
```

**Warning for SSDs:** Avoid `/r` on SSDs unless you suspect actual bad sectors. Use `/f` or `/offlinescanandfix` instead to avoid unnecessary wear.

### diskpart — Advanced Disk Management

```cmd
diskpart

:: List all disks
list disk

:: Select a disk
select disk 0

:: List partitions
list partition

:: List volumes
list volume

:: Clean a disk (DESTRUCTIVE — removes all partitions)
clean

:: Create a new partition
create partition primary size=102400

:: Format
format fs=ntfs label="Data" quick

:: Assign a drive letter
assign letter=D

:: Extend a partition (into unallocated space)
extend
```

### SSD Health Monitoring

```cmd
:: Built-in Windows 11 SSD health (NVMe only)
:: Settings → System → Storage → Disks & volumes → Properties → Drive health

:: PowerShell method
Get-PhysicalDisk | Select-Object FriendlyName, MediaType, HealthStatus, OperationalStatus
Get-PhysicalDisk | Get-StorageReliabilityCounter | Select-Object *

:: SMART data via wmic (all drives)
wmic diskdrive get model, status
:: "OK" = healthy, "Pred Fail" = replace immediately
```

### TRIM Verification (SSD)

```cmd
:: Check if TRIM is enabled
fsutil behavior query DisableDeleteNotify
:: DisableDeleteNotify = 0 means TRIM IS enabled (counterintuitive naming)
:: DisableDeleteNotify = 1 means TRIM is disabled

:: Enable TRIM
fsutil behavior set DisableDeleteNotify 0
```

### WinSxS (Component Store) Management

```cmd
:: Analyze size (actual size vs reported size — different due to hard links)
DISM /Online /Cleanup-Image /AnalyzeComponentStore

:: Clean up superseded components (30-day grace period)
DISM /Online /Cleanup-Image /StartComponentCleanup

:: Aggressive cleanup (no grace period, can't uninstall updates after this)
DISM /Online /Cleanup-Image /StartComponentCleanup /ResetBase

:: Scheduled task for automatic cleanup
schtasks /Run /TN "\Microsoft\Windows\Servicing\StartComponentCleanup"
```

### Storage Sense (Automatic Cleanup)

Settings → System → Storage → Storage Sense → Configure

Automatically cleans:
- Temporary files
- Recycle Bin (configurable age)
- Downloads folder (configurable age)
- Locally available cloud content
- Windows Update cleanup

---

## 32. Remote Desktop (RDP)

### Enable Remote Desktop

```cmd
:: Via registry (no GUI needed)
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f

:: Enable firewall rule
netsh advfirewall firewall set rule group="remote desktop" new enable=yes

:: Check if RDP is enabled
reg query "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections
:: 0 = enabled, 1 = disabled
```

### RDP Connection Refused

| Symptom | Cause | Fix |
|---------|-------|-----|
| "Remote Desktop can't connect" | RDP disabled, firewall blocking, or wrong IP | Check `fDenyTSConnections`, firewall rule, and `ipconfig` |
| "The remote computer requires NLA" | NLA mismatch or domain controller unreachable | Disable NLA: `reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v SecurityLayer /t REG_DWORD /d 0 /f` |
| Black screen after connecting | GPU driver issue in session | `mstsc /admin` to connect as console, or lower color depth |
| "Authentication error" / "CredSSP" | CredSSP encryption oracle remediation | `reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" /v AllowEncryptionOracle /t REG_DWORD /d 2 /f` |
| Stuck on "Configuring remote session" | Profile corruption on remote machine | Log in with a different account, fix profile |

### RDP from Command Line

```cmd
:: Connect to a remote machine
mstsc /v:192.168.1.100

:: Admin/console session
mstsc /v:192.168.1.100 /admin

:: Specify resolution
mstsc /v:192.168.1.100 /w:1920 /h:1080

:: Full screen
mstsc /v:192.168.1.100 /f

:: Save connection settings
mstsc /v:192.168.1.100 /savecred
```

### RDP Port Change (Security)

```cmd
:: Change from default 3389 to custom port
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v PortNumber /t REG_DWORD /d 33389 /f

:: Update firewall
netsh advfirewall firewall add rule name="RDP Custom Port" dir=in action=allow protocol=tcp localport=33389

:: Connect to custom port
mstsc /v:192.168.1.100:33389
```

---

