<!-- Windows 11 Troubleshooting Bible — TSS Toolset, CBS Log Analysis, DISM Complete Reference -->

## 38. TSS Toolset, CBS Log Analysis & DISM Deep Reference

### TSS (TroubleShootingScript) — Microsoft's Official Diagnostic Collector

TSS is Microsoft's internal toolset for collecting diagnostic data across all Windows components. It's what CSS (Customer Service and Support) engineers use when you open a support ticket.

**Download:** [https://aka.ms/getTSS](https://aka.ms/getTSS) — extracts to a folder with `TSS.ps1`

```powershell
# Basic usage — collect servicing (CBS) logs
.\TSS.ps1 -Scenario CBS_Servicing

# Collect Windows Update logs
.\TSS.ps1 -Scenario WU_Update

# Collect networking diagnostics
.\TSS.ps1 -Scenario NET_General

# Collect everything related to component store issues
.\TSS.ps1 -Scenario CBS_Servicing -WaitEvent Evt:System:EventID=7023

# Start continuous tracing, stop when specific error occurs
.\TSS.ps1 -StartAutoLogger -Scenario CBS_Servicing

# Collect auth/credential diagnostics
.\TSS.ps1 -Scenario ADS_Auth

# Full collection with network trace
.\TSS.ps1 -Scenario CBS_Servicing -NetTrace
```

**What TSS collects for CBS_Servicing:**
- Full CBS.log and DISM.log
- Component store integrity verification results
- Pending.xml (queued servicing operations)
- Sessions.xml (servicing history)
- Poqexec.log (post-reboot install queue)
- All Windows Update ETW traces
- System and Application event logs
- Relevant registry keys
- DISM health check results

### CBS.log Deep Analysis

The CBS (Component Based Servicing) log is the single source of truth for Windows servicing operations. Located at `C:\Windows\Logs\CBS\CBS.log`.

**Key patterns to search for:**

```
# Successful operation
"CSI Transaction @0x... initialized for <package>"
"Exec: Processing complete. Session: ..., Package: ..., Operation: Install, Result: 0x0"

# Failed operation — the error is in the Result
"Exec: Processing complete. Session: ..., Package: ..., Operation: Install, Result: 0x800f0922"

# Component store corruption
"Manifest hash for component ... does not match expected value"
"CSI Manifest All Coverage Check Failed"
"Store corruption detected"
"SPI: Failed to move file"

# Missing file/payload
"Failed to find file ... in component store"
"CBS_E_SOURCE_MISSING"
"Unable to find package source"

# Pending operations stuck
"Exec: Reboot is required"
"Reboot already pending"
"Session: ..., Status: Reboot_Pending"

# TrustedInstaller issues
"Failed to get execution status of servicing session"
"TiWorker.exe failed with HRESULT"
```

**Reading CBS.log efficiently:**
```powershell
# Find all errors in CBS.log
Select-String -Path C:\Windows\Logs\CBS\CBS.log -Pattern "HRESULT|Error|Failed|Corrupt" |
  Select-Object -Last 50

# Find the most recent servicing session
Select-String -Path C:\Windows\Logs\CBS\CBS.log -Pattern "Session: \d+ initialized" |
  Select-Object -Last 5

# Track a specific KB through the log
Select-String -Path C:\Windows\Logs\CBS\CBS.log -Pattern "KB5034441"

# Check for pending reboot operations
Select-String -Path C:\Windows\Logs\CBS\CBS.log -Pattern "Reboot_Pending|RebootRequired"

# The CBS.log rotates — check previous logs too
dir C:\Windows\Logs\CBS\CBS*.log
```

### DISM — Complete Parameter Reference

```powershell
# === HEALTH CHECK COMMANDS (run in this order) ===

# Quick check — verifies component store metadata is intact
DISM /Online /Cleanup-Image /CheckHealth

# Deeper scan — verifies each component against its manifest hash
DISM /Online /Cleanup-Image /ScanHealth

# Repair — downloads or uses local source to fix corrupted components
DISM /Online /Cleanup-Image /RestoreHealth

# Repair using a local Windows image as source (when no internet or WU is broken)
DISM /Online /Cleanup-Image /RestoreHealth /Source:D:\sources\install.wim
DISM /Online /Cleanup-Image /RestoreHealth /Source:D:\sources\install.esd
DISM /Online /Cleanup-Image /RestoreHealth /Source:wim:D:\sources\install.wim:1

# Repair without trying Windows Update at all
DISM /Online /Cleanup-Image /RestoreHealth /Source:D:\sources\install.wim /LimitAccess

# === CLEANUP COMMANDS ===

# Remove superseded components (frees disk space, can't uninstall old updates after this)
DISM /Online /Cleanup-Image /StartComponentCleanup

# Aggressive cleanup — resets component store, removes ALL superseded versions
DISM /Online /Cleanup-Image /StartComponentCleanup /ResetBase

# Remove backup files created during SP install
DISM /Online /Cleanup-Image /SPSuperseded

# Analyze component store size
DISM /Online /Cleanup-Image /AnalyzeComponentStore

# === FEATURE MANAGEMENT ===

# List all optional features and their states
DISM /Online /Get-Features

# Enable a feature
DISM /Online /Enable-Feature /FeatureName:Microsoft-Hyper-V-All /All

# Enable .NET 3.5 from Windows ISO
DISM /Online /Enable-Feature /FeatureName:NetFx3 /Source:D:\sources\sxs /LimitAccess

# Disable a feature
DISM /Online /Disable-Feature /FeatureName:Microsoft-Hyper-V-All

# === PACKAGE MANAGEMENT ===

# List all installed packages
DISM /Online /Get-Packages

# Get detailed info about a specific package
DISM /Online /Get-PackageInfo /PackageName:<package-identity>

# Install a CAB or MSU
DISM /Online /Add-Package /PackagePath:C:\temp\update.cab

# Remove a package
DISM /Online /Remove-Package /PackageName:<package-identity>

# === DRIVER MANAGEMENT ===

# List all third-party drivers
DISM /Online /Get-Drivers

# Add a driver
DISM /Online /Add-Driver /Driver:C:\drivers\mydriver.inf

# Add all drivers in a folder
DISM /Online /Add-Driver /Driver:C:\drivers\ /Recurse

# Remove a driver
DISM /Online /Remove-Driver /Driver:oem42.inf

# === OFFLINE IMAGE OPERATIONS (for repair/deployment) ===

# Mount a WIM for offline servicing
DISM /Mount-Wim /WimFile:D:\sources\install.wim /Index:1 /MountDir:C:\mount

# Apply updates to mounted image
DISM /Image:C:\mount /Add-Package /PackagePath:C:\updates\

# Unmount and save
DISM /Unmount-Wim /MountDir:C:\mount /Commit

# Export a specific edition from install.wim (reduces file size)
DISM /Export-Image /SourceImageFile:D:\sources\install.wim /SourceIndex:6 /DestinationImageFile:C:\temp\pro.wim /Compress:max
```

### DISM Log Location and Analysis

```powershell
# DISM.log location
C:\Windows\Logs\DISM\DISM.log

# Find errors
Select-String -Path C:\Windows\Logs\DISM\DISM.log -Pattern "Error|HRESULT|Failed" | Select-Object -Last 30

# The log line format:
# [timestamp] [PID] [TID] [severity] [component] message
# Example: 2026-03-26 10:15:32, Info CBS Exec: Processing started. Session: 30812345_567890
```

### Pending.xml — The Servicing Queue

When updates need a reboot to complete, they're queued in `C:\Windows\WinSxS\pending.xml`. If this file gets corrupted, Windows can't finish installing updates and may boot-loop.

```powershell
# Check if pending.xml exists (it shouldn't when system is idle)
Test-Path C:\Windows\WinSxS\pending.xml

# If stuck in a reboot loop caused by pending.xml:
# Boot to WinRE (Recovery Environment)
# Open cmd:
ren C:\Windows\WinSxS\pending.xml pending.xml.bak
# Reboot — this abandons the pending servicing operation
# Then run DISM /Online /Cleanup-Image /RestoreHealth to fix any inconsistency
```

---

