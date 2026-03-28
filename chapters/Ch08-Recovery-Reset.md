<!-- Windows 11 Troubleshooting Bible — Safe Mode, WinRE, Developer Features, Nuclear Reset Options -->

## 19. Safe Mode & Windows Recovery Environment

### Accessing Safe Mode

**Method 1: From Settings**
Settings → System → Recovery → Advanced startup → Restart now → Troubleshoot → Advanced options → Startup Settings → Restart → Press F4 (Safe Mode) or F5 (Safe Mode with Networking)

**Method 2: Shift+Restart**
Hold Shift → click Restart from the Start menu power button

**Method 3: From cmd.exe**
```cmd
:: Boot into Safe Mode on next restart
bcdedit /set {current} safeboot minimal

:: Boot into Safe Mode with Networking
bcdedit /set {current} safeboot network

:: IMPORTANT: Remove safeboot flag after you're done
bcdedit /deletevalue {current} safeboot
```

**Method 4: Force WinRE (when nothing else works)**
- Power on the PC
- As soon as the Windows logo appears, hold the power button to force shutdown
- Repeat 3 times
- On the 4th boot, Windows enters Automatic Repair → Advanced options

### Windows Recovery Environment (WinRE) Tools

| Tool | What It Does |
|------|-------------|
| Reset this PC | Reinstall Windows (keep or remove files) |
| Startup Repair | Automatically fixes boot issues |
| Command Prompt | Full cmd.exe in recovery (can run SFC, DISM, diskpart, bcdedit) |
| Startup Settings | Boot into Safe Mode, disable driver signing, etc. |
| System Restore | Roll back to a previous restore point |
| Uninstall Updates | Remove a Windows Update that caused problems |
| UEFI Firmware Settings | Enter BIOS/UEFI |

### Running SFC from Recovery

When SFC can't fix files from within Windows:
```cmd
:: From WinRE Command Prompt, specify the offline Windows installation:
sfc /scannow /offbootdir=C:\ /offwindir=C:\Windows
```

---

## 20. Windows 11 Developer Features & Optimization

### Dev Drive (Windows 11 23H2+)

ReFS-formatted volume with Defender Performance Mode. Up to **30% faster builds**.

```cmd
:: Check if Dev Drive is available
fsutil devdrv query

:: Create via GUI:
:: Settings → System → Storage → Advanced Storage Settings → Disks & Volumes → Create Dev Drive
:: Minimum size: 50GB
```

**Performance Mode:** Defender scans asynchronously on Dev Drives instead of blocking file operations. Enable:
Settings → Windows Security → Virus & threat protection → Manage settings → scroll to Dev Drive protection → Performance mode

### Developer Mode

Settings → System → For developers → Developer Mode toggle

Enables:
- Sideloading apps without Microsoft Store
- WSL and Hyper-V configuration
- Device discovery for debugging
- Loopback networking for UWP apps

### Defender Exclusions for Dev Tools

Real-time scanning kills build performance. Add exclusions:

Settings → Windows Security → Virus & threat protection → Manage settings → Exclusions

**Recommended exclusions:**
```
Folders:
%USERPROFILE%\source
%USERPROFILE%\.docker
%USERPROFILE%\.wsl
C:\Program Files\Docker
%LOCALAPPDATA%\Docker
%APPDATA%\npm
%USERPROFILE%\node_modules
%USERPROFILE%\.cargo
%USERPROFILE%\.rustup
%USERPROFILE%\go
%USERPROFILE%\.m2  (Maven)
%USERPROFILE%\.gradle

Processes:
pwsh.exe, node.exe, docker.exe, python.exe, java.exe, rustc.exe, cargo.exe, go.exe
```

### Windows Terminal Configuration

**Set as default terminal:**
Settings → System → For developers → Terminal → Windows Terminal

**Essential settings (settings.json):**
```json
{
    "defaultProfile": "{your-pwsh-guid}",
    "profiles": {
        "defaults": {
            "font": { "face": "CaskaydiaCove Nerd Font" },
            "startingDirectory": "~"
        }
    }
}
```

### Memory Integrity (Core Isolation)

Can conflict with certain drivers and debugging tools (e.g., older WinDbg plugins, some AV kernel modules):

Settings → Windows Security → Device security → Core isolation → Memory integrity

If dev tools are crashing inexplicably, temporarily disable to test.

---

## 21. Nuclear Options & Full Reset Procedures

### In-Place Repair Install (Best Non-Destructive Fix)

Re-installs Windows while keeping ALL files, apps, settings, and drivers:

1. Download Windows 11 ISO from Microsoft (media creation tool)
2. Mount the ISO (double-click it)
3. Run `setup.exe` from the mounted drive
4. Choose "Keep personal files and apps"
5. Wait 30-60 minutes
6. Your PC restarts with a fresh Windows installation but all your stuff intact

This fixes nearly every system-level corruption.

### Complete Environment Reset (PS + WSL + Docker)

```cmd
:: === PHASE 1: DESTROY ===

wsl --shutdown
taskkill /f /im "Docker Desktop.exe" 2>nul
net stop LxssManager 2>nul

winget uninstall Docker.DockerDesktop 2>nul

for /f "tokens=1" %%i in ('wsl --list --quiet 2^>nul') do wsl --unregister %%i

rmdir /s /q "%LOCALAPPDATA%\Docker" 2>nul
rmdir /s /q "%APPDATA%\Docker" 2>nul
rmdir /s /q "%PROGRAMDATA%\Docker" 2>nul
rmdir /s /q "%USERPROFILE%\.docker" 2>nul

ren "%USERPROFILE%\Documents\PowerShell\Microsoft.PowerShell_profile.ps1" "profile.ps1.bak" 2>nul
ren "%USERPROFILE%\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1" "profile.ps1.bak" 2>nul

shutdown /r /t 0

:: === PHASE 2: REPAIR ===

DISM /Online /Cleanup-Image /RestoreHealth
sfc /scannow

dism /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

shutdown /r /t 0

:: === PHASE 3: REBUILD ===

wsl --update
wsl --set-default-version 2

echo [wsl2] > "%USERPROFILE%\.wslconfig"
echo memory=4GB >> "%USERPROFILE%\.wslconfig"
echo processors=4 >> "%USERPROFILE%\.wslconfig"
echo swap=2GB >> "%USERPROFILE%\.wslconfig"
echo autoMemoryReclaim=gradual >> "%USERPROFILE%\.wslconfig"

wsl --install -d Ubuntu

winget install Microsoft.PowerShell
pwsh -NoProfile -Command "Set-ExecutionPolicy RemoteSigned -Scope CurrentUser -Force"
powershell -NoProfile -Command "Set-ExecutionPolicy RemoteSigned -Scope CurrentUser -Force"

winget install Docker.DockerDesktop

shutdown /r /t 0
```

### Your Specific Fix — Right Now

For your exact situation (broken PS + stalled Docker + Vmmem at 85% RAM):

```cmd
:: FROM CMD.EXE ONLY — don't try PowerShell

:: 1. Kill Vmmem immediately
wsl --shutdown

:: 2. Disable all PS profiles
ren "%USERPROFILE%\Documents\PowerShell\Microsoft.PowerShell_profile.ps1" "profile.broken" 2>nul
ren "%USERPROFILE%\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1" "profile.broken" 2>nul
ren "%USERPROFILE%\OneDrive\Documents\PowerShell\Microsoft.PowerShell_profile.ps1" "profile.broken" 2>nul
ren "%USERPROFILE%\OneDrive\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1" "profile.broken" 2>nul

:: 3. Test PowerShell
pwsh
:: Type "exit" if it works

:: 4. Clean Docker remnants
wsl --unregister docker-desktop 2>nul
wsl --unregister docker-desktop-data 2>nul
rmdir /s /q "%LOCALAPPDATA%\Docker" 2>nul
rmdir /s /q "%APPDATA%\Docker" 2>nul
rmdir /s /q "%USERPROFILE%\.docker" 2>nul

:: 5. Reboot
shutdown /r /t 0

:: 6. After reboot — reinstall Docker
winget install Docker.DockerDesktop
```

---

