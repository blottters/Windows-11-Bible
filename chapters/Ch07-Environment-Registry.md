<!-- Windows 11 Troubleshooting Bible — Environment Variables, PATH, Registry Troubleshooting -->

## 17. Environment Variables & PATH

### Viewing PATH

```cmd
:: Full PATH (both User and System combined)
echo %PATH%

:: User PATH only
reg query "HKCU\Environment" /v Path

:: System PATH only
reg query "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment" /v Path
```

### The 8192-Character PATH Trap

PATH has a practical limit of ~8192 characters. Installers keep appending without checking. When PATH overflows, entries silently get truncated from the end.

Symptoms:
- Tools that used to work suddenly "not recognized"
- `where <tool>` returns nothing even though it's installed
- New installs work but old tools break

### Finding Executables

```cmd
:: Which executable runs (first match in PATH)
where pwsh
where powershell
where docker
where node
where python
where code

:: Find ALL matching executables
where /r C:\ pwsh.exe 2>nul
```

### Fixing PATH

**GUI:** Win+R → `sysdm.cpl` → Advanced → Environment Variables

**Command line:**
```cmd
:: Add to User PATH (persistent)
setx PATH "%PATH%;C:\new\path"
:: WARNING: setx has a 1024-character limit! Use GUI or PowerShell for longer values

:: From PowerShell (once fixed):
[Environment]::SetEnvironmentVariable('Path', [Environment]::GetEnvironmentVariable('Path', 'User') + ';C:\new\path', 'User')
```

### Diagnosing PATH Corruption

```cmd
:: Count PATH entries
echo %PATH% | find /c ";"

:: List each PATH entry on its own line (from PowerShell)
pwsh -NoProfile -Command "$env:PATH -split ';' | ForEach-Object { $_ }"

:: Check for duplicates (from PowerShell)
pwsh -NoProfile -Command "$env:PATH -split ';' | Group-Object | Where-Object Count -gt 1"
```

### MAX_PATH Limitation

Windows has a 260-character path limit by default. This breaks npm, deep node_modules trees, and long repo paths.

```cmd
:: Enable long paths
reg add "HKLM\SYSTEM\CurrentControlSet\Control\FileSystem" /v LongPathsEnabled /t REG_DWORD /d 1 /f

:: Also via Group Policy:
:: Computer Configuration → Administrative Templates → System → Filesystem → Enable Win32 long paths
```

---

## 18. Registry Troubleshooting

### Important Registry Locations for Debugging

| Key | What It Contains |
|-----|-----------------|
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` | Programs that start at login (all users) |
| `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` | Programs that start at login (current user) |
| `HKLM\SYSTEM\CurrentControlSet\Services` | All Windows services configuration |
| `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager` | Boot-time programs, environment variables |
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall` | Installed programs |
| `HKCU\Environment` | User environment variables (including PATH) |
| `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment` | System environment variables |
| `HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell` | PowerShell policies (execution policy, logging) |

### Registry Backup Before Editing

**ALWAYS backup before making registry changes:**

```cmd
:: Backup a specific key
reg export "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" C:\temp\run_backup.reg

:: Restore from backup
reg import C:\temp\run_backup.reg

:: Full registry backup (from admin cmd)
reg export HKLM C:\temp\hklm_full.reg
```

### Common Registry Fixes

**Fix: Pending reboot flag stuck (blocks installs):**
```cmd
reg delete "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending" /f 2>nul
reg delete "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired" /f 2>nul
```

**Fix: Clear stuck Windows Update:**
```cmd
reg delete "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate" /v PingID /f 2>nul
reg delete "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate" /v AccountDomainSid /f 2>nul
```

---

