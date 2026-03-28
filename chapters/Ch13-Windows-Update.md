<!-- Windows 11 Troubleshooting Bible — Windows Update Internals -->

## 33. Windows Update Internals

### Update Process Flow

1. **Scan:** Windows Update client checks for updates
2. **Download:** Updates download via Delivery Optimization (DO) or BITS
3. **Install:** CBS (Component Based Servicing) installs the update
4. **Commit:** After reboot, changes are finalized
5. **Cleanup:** Old components marked for cleanup

### CBS Log Analysis

The most important log for update troubleshooting:

```cmd
:: CBS log location
notepad C:\Windows\Logs\CBS\CBS.log

:: Search for errors
findstr /i "error" C:\Windows\Logs\CBS\CBS.log > C:\temp\cbs_errors.txt

:: Search for specific KB
findstr /i "KB5034441" C:\Windows\Logs\CBS\CBS.log > C:\temp\kb_log.txt
```

**Key patterns in CBS.log:**
- `Store corruption` → Component store is damaged → `DISM /RestoreHealth`
- `Applicable State: Installed` → Update is already installed
- `Error: 0x800f0922` → Not enough space on System Reserved partition
- `Error: 0x800f081f` → Source files could not be found → `DISM /RestoreHealth`
- `Exec: Processing complete` → Update installed successfully

### Windows Update Log

```cmd
:: Generate readable Windows Update log (from PowerShell)
pwsh -NoProfile -Command "Get-WindowsUpdateLog"
:: Creates WindowsUpdate.log on your Desktop

:: Or view ETL traces directly:
:: C:\Windows\Logs\WindowsUpdate\
```

### Delivery Optimization

Controls how updates are downloaded — can use peer-to-peer sharing:

```cmd
:: Check DO status
:: Settings → Windows Update → Advanced options → Delivery Optimization

:: Check DO cache usage
Get-DeliveryOptimizationStatus

:: Clear DO cache
Delete-DeliveryOptimizationCache -Force
```

### Manually Install Updates

When Windows Update is broken:

```cmd
:: Download update from Microsoft Update Catalog
:: https://www.catalog.update.microsoft.com

:: Install MSU manually
wusa.exe update.msu /quiet /norestart

:: Install CAB manually
dism /online /add-package /packagepath:C:\temp\update.cab

:: Uninstall a problematic update
wusa.exe /uninstall /kb:5034441 /quiet /norestart
:: Or
dism /online /remove-package /packagename:Package_for_KB5034441~...
```

---

