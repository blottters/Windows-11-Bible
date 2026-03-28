<!-- Windows 11 Troubleshooting Bible — SFC, DISM, CBS, Component Store, Windows Services -->

## 11. System Repair Tools

### The Correct Order — ALWAYS

DISM repairs the source image. SFC uses that source to repair files. Wrong order = SFC fails because its source is also corrupt.

```cmd
:: Step 1: Quick health check
DISM /Online /Cleanup-Image /CheckHealth

:: Step 2: Deep scan (takes 5-15 minutes)
DISM /Online /Cleanup-Image /ScanHealth

:: Step 3: Repair (downloads from Windows Update if needed)
DISM /Online /Cleanup-Image /RestoreHealth

:: Step 4: Now repair system files
sfc /scannow
```

### When DISM Can't Download Repair Files

```cmd
:: Use a Windows 11 ISO as the source
:: Mount the ISO first (double-click it), then:
DISM /Online /Cleanup-Image /RestoreHealth /Source:D:\sources\install.wim:1 /LimitAccess
:: D: = your mounted ISO drive letter
:: :1 = image index (usually 1)
:: /LimitAccess = don't try Windows Update
```

### Reading the Logs

```cmd
:: SFC results (extract from CBS log)
findstr /c:"[SR]" %windir%\Logs\CBS\CBS.log > "%USERPROFILE%\Desktop\sfcdetails.txt"
notepad "%USERPROFILE%\Desktop\sfcdetails.txt"

:: SFC result interpretation:
:: [SR] Cannot repair member file [l:...] → SFC found corruption but couldn't fix it
:: [SR] Verify and Repair Transaction completed → SFC attempted repairs
:: [SR] Beginning Verify and Repair transaction → Start of scan

:: DISM log
notepad %windir%\Logs\DISM\dism.log
:: Look for "Error" entries to find what failed
```

### SFC Output Messages

| Message | Meaning | Next Step |
|---------|---------|-----------|
| "did not find any integrity violations" | System is clean | Nothing to do |
| "found corrupt files and successfully repaired them" | Fixed! | Check SFC log for details |
| "found corrupt files but was unable to fix some of them" | Source is also corrupt | Run DISM first, then SFC again |
| "could not perform the requested operation" | SFC itself is broken | Run from Safe Mode or WinRE |

### Windows Update Reset

When Windows Update is broken (cascades to DISM failures):

```cmd
:: Stop all update-related services
net stop wuauserv
net stop cryptSvc
net stop bits
net stop msiserver

:: Rename cache folders (effectively clears the cache)
ren C:\Windows\SoftwareDistribution SoftwareDistribution.old
ren C:\Windows\System32\catroot2 catroot2.old

:: Re-register DLLs
regsvr32 /s wuaueng.dll
regsvr32 /s wuaueng1.dll
regsvr32 /s atl.dll
regsvr32 /s wups.dll
regsvr32 /s wups2.dll
regsvr32 /s wuweb.dll
regsvr32 /s wucltux.dll

:: Restart services
net start wuauserv
net start cryptSvc
net start bits
net start msiserver
```

### Component Store Cleanup

```cmd
:: Check component store size
DISM /Online /Cleanup-Image /AnalyzeComponentStore

:: Clean up old components (frees disk space)
DISM /Online /Cleanup-Image /StartComponentCleanup

:: Aggressive cleanup (removes ALL superseded versions, irreversible)
DISM /Online /Cleanup-Image /StartComponentCleanup /ResetBase
```

---

## 12. Windows Services Debugging

### Service States and Control

```cmd
:: List all services
sc query state= all

:: Query a specific service
sc query <ServiceName>
sc qc <ServiceName>  :: Shows config (binary path, startup type, dependencies)

:: Start/Stop/Restart
net start <ServiceName>
net stop <ServiceName>
sc start <ServiceName>
sc stop <ServiceName>

:: Change startup type
sc config <ServiceName> start= auto      :: Automatic
sc config <ServiceName> start= delayed-auto  :: Delayed start
sc config <ServiceName> start= demand    :: Manual
sc config <ServiceName> start= disabled  :: Disabled
```

### Service Dependencies

When you get "Error 1068: The dependency service or group failed to start":

```cmd
:: Show what a service depends on
sc qc <ServiceName>
:: Look for "DEPENDENCIES" line

:: Show what depends on a service
sc enumdepend <ServiceName>

:: Fix broken dependencies
sc config <ServiceName> depend= <Dep1>/<Dep2>
```

### Critical Services for Dev Work

| Service | Display Name | Why It Matters |
|---------|-------------|----------------|
| `LxssManager` | Windows Subsystem for Linux | WSL2 — if stopped, `wsl` commands fail |
| `WslService` | WSL Service | WSL2 networking (newer builds) |
| `vmcompute` | Hyper-V Host Compute Service | Container/VM management — Docker needs this |
| `wuauserv` | Windows Update | System updates, DISM source |
| `msiserver` | Windows Installer | All MSI-based installs |
| `Winmgmt` | Windows Management Instrumentation | WMI queries, many tools depend on it |
| `CryptSvc` | Cryptographic Services | Certificate validation, Windows Update |
| `BITS` | Background Intelligent Transfer Service | Downloads for Windows Update |
| `com.docker.service` | Docker Desktop Service | Docker backend |

### Service Debugging with Event Viewer

Services log to: **Windows Logs → System**

Key Event IDs:
- **7000**: Service failed to start
- **7001**: Service dependency failure
- **7009**: Timeout waiting for service
- **7023**: Service terminated with error
- **7031**: Service crashed and restart was attempted
- **7034**: Service crashed unexpectedly

---

