<!-- Windows 11 Troubleshooting Bible — Windows Update Error Codes — Complete Reference Table -->

## 36. Windows Update Error Codes — Complete Reference

### How Windows Update Works (Internals)

The update pipeline: **Scan → Download → Install → Commit → Reboot (if needed)**

Key components:
- **WUAServ (Windows Update Agent Service)** — orchestrates the process
- **CBS (Component Based Servicing)** — the actual installation engine
- **TrustedInstaller** — the only process with permission to modify system files
- **Servicing Stack Update (SSU)** — updates CBS itself; must be current before other updates install
- **USOSvc (Update Orchestrator Service)** — new in Win10/11, coordinates update scheduling

### Error Code Table

| Error Code | Meaning | Root Cause | Fix |
|-----------|---------|------------|-----|
| `0x80070005` | ACCESS_DENIED | TrustedInstaller can't write to the component store or a service lacks permissions | Run `DISM /Online /Cleanup-Image /RestoreHealth`. Check that the TrustedInstaller service is running. Reset Windows Update components (see below). |
| `0x80070057` | E_INVALIDARG | Corrupted registry settings, often in `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate` | Delete `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update` and restart WUAServ. |
| `0x80070570` | ERROR_FILE_OR_DIRECTORY_CORRUPTED | Downloaded update file is corrupt, or the component store has integrity issues | Clear `C:\Windows\SoftwareDistribution\Download`, then `DISM /Online /Cleanup-Image /RestoreHealth`, then re-scan. |
| `0x80072EE2` | WININET_E_TIMEOUT | Can't reach Windows Update servers — proxy, firewall, DNS, or ISP issue | Test: `ping download.windowsupdate.com`. Check proxy: `netsh winhttp show proxy`. Try `netsh winhttp reset proxy`. |
| `0x80072F8F` | WININET_E_DECODING_FAILED | SSL/TLS certificate validation failure — clock wrong, root certs expired, or MITM | Check system clock. Run `certutil -generateSSTFromWU C:\temp\certs.sst` to refresh root certificates. |
| `0x80073712` | ERROR_SXS_COMPONENT_STORE_CORRUPT | Component store (WinSxS) is damaged | `DISM /Online /Cleanup-Image /RestoreHealth`. If that fails with the same error, use an ISO: `DISM /Online /Cleanup-Image /RestoreHealth /Source:D:\sources\install.wim` |
| `0x800F0831` | CBS_E_STORE_CORRUPTION | Required update file missing from component store — usually means a delta/express update can't find its base | Install the full cumulative update manually from [Microsoft Update Catalog](https://catalog.update.microsoft.com). |
| `0x800F081F` | CBS_E_SOURCE_MISSING | .NET Framework or other optional component source files not found | `DISM /Online /Enable-Feature /FeatureName:NetFx3 /Source:D:\sources\sxs /LimitAccess` (use Windows ISO). |
| `0x800F0920` | CBS_E_HANG_DETECTED | Servicing operation hung during install — often a driver or antivirus holding a lock | Boot to Safe Mode, run the update. If that fails, uninstall third-party AV temporarily. |
| `0x800F0922` | CBS_E_INSTALLERS_FAILED | An installer within the update package failed — often a full System Reserved partition | Extend the System Reserved / EFI partition. Check `CBS.log` for the specific sub-installer that failed. |
| `0x800F0986` | CBS_E_PACKAGE_NOT_APPLICABLE | Update doesn't apply to this Windows edition/version/architecture | Verify: `winver` shows the correct build. Download the correct update for your exact build number. |
| `0x80240017` | WU_E_NOT_APPLICABLE | Update already installed or doesn't apply | Check installed updates: `Get-HotFix` or `wmic qfe list`. |
| `0x8024000E` | WU_E_XML_MISSINGDATA | Update metadata XML is malformed | Clear `C:\Windows\SoftwareDistribution` folder and re-scan. |
| `0x8024402F` | WU_E_PT_ECP_SUCCEEDED_WITH_ERRORS | Download partially succeeded but some files failed | Retry. If persistent, manual download from Update Catalog. |
| `0xC1900101 - 0x20017` | SAFE_OS phase, BOOT operation failed | Driver incompatibility during feature update | Uninstall problematic drivers (often Realtek audio, old GPU drivers) before upgrading. |
| `0xC1900101 - 0x30018` | FIRST_BOOT phase, SYSPREP operation failed | OEM software or system configuration incompatible | Uninstall OEM bloatware, disconnect peripherals, try again. |
| `0xC1900208` | MOSETUP_E_COMPAT_SCANONLY | Compatibility hold — hardware/software blocked the upgrade | Run setup with `/compat scanonly` to see what's blocking. Check `C:\$WINDOWS.~BT\Sources\Panther\compat*.xml`. |

### Windows Update Reset Procedure (Nuclear)

When nothing else works, reset the entire Windows Update subsystem:

```batch
:: Run as Administrator in cmd.exe
:: Step 1: Stop all update-related services
net stop wuauserv
net stop cryptSvc
net stop bits
net stop msiserver
net stop UsoSvc

:: Step 2: Rename the software distribution folders (backup, not delete)
ren C:\Windows\SoftwareDistribution SoftwareDistribution.bak
ren C:\Windows\System32\catroot2 catroot2.bak

:: Step 3: Re-register critical DLLs
regsvr32.exe /s atl.dll
regsvr32.exe /s urlmon.dll
regsvr32.exe /s mshtml.dll
regsvr32.exe /s shdocvw.dll
regsvr32.exe /s browseui.dll
regsvr32.exe /s jscript.dll
regsvr32.exe /s vbscript.dll
regsvr32.exe /s scrrun.dll
regsvr32.exe /s msxml.dll
regsvr32.exe /s msxml3.dll
regsvr32.exe /s msxml6.dll
regsvr32.exe /s actxprxy.dll
regsvr32.exe /s softpub.dll
regsvr32.exe /s wintrust.dll
regsvr32.exe /s dssenh.dll
regsvr32.exe /s rsaenh.dll
regsvr32.exe /s gpkcsp.dll
regsvr32.exe /s sccbase.dll
regsvr32.exe /s slbcsp.dll
regsvr32.exe /s cryptdlg.dll
regsvr32.exe /s oleaut32.dll
regsvr32.exe /s ole32.dll
regsvr32.exe /s shell32.dll
regsvr32.exe /s initpki.dll
regsvr32.exe /s wuapi.dll
regsvr32.exe /s wuaueng.dll
regsvr32.exe /s wuaueng1.dll
regsvr32.exe /s wucltui.dll
regsvr32.exe /s wups.dll
regsvr32.exe /s wups2.dll
regsvr32.exe /s wuweb.dll
regsvr32.exe /s qmgr.dll
regsvr32.exe /s qmgrprxy.dll
regsvr32.exe /s wucltux.dll
regsvr32.exe /s muweb.dll
regsvr32.exe /s wuwebv.dll

:: Step 4: Reset Winsock and proxy
netsh winsock reset
netsh winhttp reset proxy

:: Step 5: Restart services
net start wuauserv
net start cryptSvc
net start bits
net start msiserver
net start UsoSvc

:: Step 6: Force re-scan
UsoClient StartScan
```

### Manual Update Installation

When automatic update fails, go manual:

```powershell
# 1. Find the KB number from Windows Update history or error message
# 2. Go to https://catalog.update.microsoft.com
# 3. Search for the KB, download the .msu file for your architecture (x64)

# Install MSU manually
wusa.exe C:\temp\windows11-kb5034441-x64.msu /quiet /norestart

# If wusa fails, extract and use DISM
expand -f:* C:\temp\windows11-kb5034441-x64.msu C:\temp\expanded\
dism /online /add-package /packagepath:C:\temp\expanded\*.cab

# Install SSU (Servicing Stack Update) first if updates keep failing
# SSUs are on the Update Catalog, search "Servicing Stack Update Windows 11"
```

---

