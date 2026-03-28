<!-- Windows 11 Troubleshooting Bible — Application Install Failures, winget Package Manager -->

## 9. Windows 11 Application Install Failures

### MSI Installer Debugging

When an MSI install fails silently or with a generic error:

```cmd
:: Run with full verbose logging
msiexec /i "installer.msi" /l*vx "C:\temp\install_log.txt"

:: Log flags:
:: /l* = log everything
:: v = verbose
:: x = extra debugging info
:: The log file shows EXACTLY which action failed and why
```

**Reading MSI logs:** Search the log for:
- `Return value 3` — the action that actually failed
- `Error` or `ERROR` — error descriptions
- `CustomAction` — custom actions that threw exceptions

### Stalled Installations — Root Causes

| Cause | How to Check | Fix |
|-------|-------------|-----|
| Pending reboot | `reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending"` | Reboot |
| Windows Installer service stuck | `sc query msiserver` | `net stop msiserver && net start msiserver` |
| Locked files | Use Process Explorer → Find Handle | Kill the locking process |
| Disk space | `dir C:\` (check free space) | Free space; also check `%TEMP%` |
| Smart App Control blocking | Settings → Privacy & Security → Windows Security → App & Browser Control | Turn off Smart App Control (permanent — can't re-enable without reset) |
| SmartScreen blocking | Right-click file → Properties → "Unblock" checkbox | Check "Unblock" and Apply |
| Broken Windows Update components | See Section 11 | SFC/DISM repair |
| Antivirus quarantine | Check AV quarantine logs | Add exclusion for installer |

### SmartScreen and Unblock

When you download a file from the internet, Windows adds an Alternate Data Stream (ADS) called `Zone.Identifier` that marks it as "from the internet." SmartScreen reads this.

```cmd
:: Check if a file is blocked
more < "installer.exe:Zone.Identifier"

:: Remove the block
echo. > "installer.exe:Zone.Identifier"

:: Or via PowerShell (if working)
Unblock-File -Path "installer.exe"
```

### Alternative Package Managers

When native installs fail, try:

```cmd
:: Chocolatey
:: Install: from admin cmd
@"%SystemRoot%\System32\WindowsPowerShell\v1.0\powershell.exe" -NoProfile -InputFormat None -ExecutionPolicy Bypass -Command "[System.Net.ServicePointManager]::SecurityProtocol = 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))" && SET "PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin"

choco install docker-desktop -y

:: Scoop (user-level, no admin needed)
:: Install: from PowerShell
irm get.scoop.sh | iex
scoop install docker
```

---

## 10. winget — Windows Package Manager Troubleshooting

### Common winget Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `winget` not recognized | App Execution Alias disabled or WinGet not installed | Settings → Apps → Advanced app settings → App execution aliases → enable "Windows Package Manager Client" |
| `0x8a15000f` — Data required by source is missing | Corrupted source database | `winget source reset --force` |
| `No applicable installer found` | Wrong architecture or Windows version | Check with `winget show <package>` for available installers |
| `Installer hash does not match` | Package updated since source last synced | `winget source update` |
| Install hangs | Hidden UAC prompt or installer window | Check taskbar for prompts |
| Source update fails | Network/proxy issue | Try `winget source update --name winget` |

### winget Diagnostics

```cmd
:: View winget info and log directory
winget --info

:: Logs are at:
:: %LOCALAPPDATA%\Packages\Microsoft.DesktopAppInstaller_8wekyb3d8bbwe\LocalState\DiagOutputDir

:: Reset all sources
winget source reset --force

:: List current sources
winget source list

:: Update sources
winget source update

:: Search for a package
winget search "Docker"

:: Show package details
winget show Docker.DockerDesktop

:: Install with verbose logging
winget install Docker.DockerDesktop --verbose-logs
```

### Fixing Broken winget

```cmd
:: Re-register the App Installer package
powershell -NoProfile -Command "Add-AppxPackage -RegisterByFamilyName -MainPackage Microsoft.DesktopAppInstaller_8wekyb3d8bbwe"

:: If that fails, reinstall from Microsoft Store
:: Search "App Installer" in Microsoft Store

:: Nuclear: reset winget entirely
rmdir /s /q "%LOCALAPPDATA%\Packages\Microsoft.DesktopAppInstaller_8wekyb3d8bbwe\LocalState" 2>nul
```

---

