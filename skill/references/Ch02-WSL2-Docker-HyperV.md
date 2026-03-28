<!-- Windows 11 Troubleshooting Bible — WSL2/Vmmem, Docker Desktop, Hyper-V Virtualization -->

## 6. WSL2 & Vmmem — Complete Reference

### What Vmmem Actually Is

`Vmmem` (or `VmmemWSL` on newer builds) is a **synthetic process** that represents memory consumed by the Hyper-V lightweight utility VM running your WSL2 Linux kernel. It is NOT a real Windows process — it's Windows' way of surfacing VM resource usage in Task Manager.

**You cannot kill it from Task Manager.** The "Access Denied" error is by design. Windows manages this process — you control it through WSL commands.

### Properly Shutting Down Vmmem

```cmd
:: Graceful shutdown of ALL WSL instances
wsl --shutdown

:: Verify it's gone (wait 5-10 seconds)
tasklist | findstr -i vmmem

:: If it persists, force-stop the WSL service
net stop LxssManager
:: Wait 10 seconds
net start LxssManager
```

### The .wslconfig File — Master Resource Control

**Location:** `%USERPROFILE%\.wslconfig` (e.g., `C:\Users\LO\.wslconfig`)

This controls resource allocation for ALL WSL2 distros globally. Without this file, WSL2 defaults can consume **50-80% of your total RAM**.

**Create/edit from cmd.exe:**
```cmd
notepad %USERPROFILE%\.wslconfig
```

**Recommended configuration:**
```ini
[wsl2]
# === MEMORY ===
# Default: 50% of total RAM or 8GB, whichever is less
# Set this to prevent Vmmem from eating all your RAM
memory=4GB

# === CPU ===
# Default: all logical processors
processors=4

# === SWAP ===
# Default: 25% of RAM size
swap=2GB

# Swap file location (default: %USERPROFILE%\AppData\Local\Temp\swap.vhdx)
# swapFile=C:\\temp\\wsl-swap.vhdx

# === NETWORKING (WSL 2.0.5+) ===
# Mirrored mode fixes most VPN/networking issues
networkingMode=mirrored
dnsTunneling=true
autoProxy=true

# === MEMORY RECLAIM (WSL 2.0.4+) ===
# Automatically reclaim cached memory from Linux
autoMemoryReclaim=gradual

# === MISC ===
# Disable GUI apps support if you don't use Linux GUI apps
# guiApplications=false

# Nested virtualization (needed for Docker-in-Docker)
nestedVirtualization=true
```

**CRITICAL: File encoding pitfall.** Notepad on Windows may add a BOM (Byte Order Mark) or use CRLF line endings that WSL2 ignores or misreads. If your settings don't take effect:

```cmd
:: Create the file from WSL to avoid encoding issues
wsl --shutdown
wsl -e bash -c "printf '[wsl2]\nmemory=4GB\nprocessors=4\nswap=2GB\nautoMemoryReclaim=gradual\n' > /mnt/c/Users/%USERNAME%/.wslconfig"
wsl
```

### WSL2 Deep Troubleshooting

**Status commands:**
```cmd
:: List all distros with state and WSL version
wsl --list --verbose
:: Output: NAME | STATE | VERSION
:: Ubuntu | Running | 2
:: docker-desktop | Stopped | 2

:: WSL system info
wsl --status
:: Shows: default distro, WSL version, kernel version, Windows version

:: WSL version
wsl --version
```

**Common WSL Error Codes:**

| Error | Cause | Fix |
|-------|-------|-----|
| `0x80370102` | Virtualization disabled in BIOS | Enter BIOS → Enable VT-x (Intel) or AMD-V |
| `0x80004005` | Virtual Machine Platform not enabled | `dism /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart` then reboot |
| `0x80370114` | WSL kernel not installed | `wsl --update --web-download` |
| `0x800701bc` | WSL2 kernel package missing | Download from aka.ms/wsl2kernel or `wsl --update` |
| `WslRegisterDistribution failed` | Corrupt distro or stale state | `wsl --unregister <distro>` then reinstall |
| `Element not found` | WSL component missing | `wsl --install --no-distribution` |
| `The WSL 2 kernel file is not found` | Kernel deleted by Windows Update | `wsl --update --web-download` |
| DNS fails inside WSL | WSL auto-generates broken resolv.conf | See DNS fix below |

**Fix DNS inside WSL:**
```bash
# Inside WSL terminal
# Option 1: Manual DNS
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf  # Prevent WSL from overwriting

# Option 2: Disable auto-generation (add to /etc/wsl.conf)
echo -e "[network]\ngenerateResolvConf = false" | sudo tee -a /etc/wsl.conf

# Option 3: Use .wslconfig (preferred for new WSL versions)
# [wsl2]
# dnsTunneling=true
```

**File system performance warning:** Accessing Windows files from WSL (`/mnt/c/...`) is 5-10x slower than native Linux filesystem. Keep your code INSIDE the WSL filesystem (`~/projects/`) not in `/mnt/c/Users/...`.

### Complete WSL Reinstall

```cmd
:: Step 1: Unregister all distros (DESTRUCTIVE — backs up nothing)
wsl --list --quiet
wsl --unregister Ubuntu
wsl --unregister docker-desktop
wsl --unregister docker-desktop-data
:: Repeat for each distro listed

:: Step 2: Disable features
dism /online /disable-feature /featurename:Microsoft-Windows-Subsystem-Linux /norestart
dism /online /disable-feature /featurename:VirtualMachinePlatform /norestart

:: Step 3: Reboot
shutdown /r /t 0

:: Step 4: Re-enable features
dism /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

:: Step 5: Reboot again
shutdown /r /t 0

:: Step 6: Fresh install
wsl --update
wsl --set-default-version 2
wsl --install -d Ubuntu
```

---

## 7. Docker Desktop on Windows 11

### Why Docker Installs Stall

Specific to your case — Docker install stalls + Vmmem lingering:

1. **WSL2 not in clean state** — Previous Docker left broken WSL distros (`docker-desktop`, `docker-desktop-data`) that the new installer can't initialize
2. **Vmmem holding resources** — Old WSL VM never shut down; new installer can't create its own VM
3. **Windows Features missing/broken** — Virtual Machine Platform or WSL not properly enabled
4. **Antivirus blocking** — Defender or third-party AV quarantining Docker binaries during install
5. **Leftover registry/AppData** — Previous Docker uninstall didn't clean up config files, and new installer reads stale config
6. **config.json corruption** — `~/.docker/config.json` has bad credential helper entries that cause hangs
7. **Stuck on "Deploying component: Use WSL 2"** — Known Docker bug where installer hangs on WSL2 setup

### Complete Docker Cleanup Procedure — The Full Scorched Earth

**Run BEFORE attempting reinstall:**

```cmd
:: === PHASE 1: Kill everything ===
wsl --shutdown
taskkill /f /im "Docker Desktop.exe" 2>nul
taskkill /f /im "com.docker.backend.exe" 2>nul
taskkill /f /im "com.docker.proxy.exe" 2>nul
taskkill /f /im "com.docker.service.exe" 2>nul
net stop com.docker.service 2>nul

:: === PHASE 2: Uninstall Docker Desktop ===
winget uninstall Docker.DockerDesktop 2>nul
:: If winget fails, use: Settings → Apps → Docker Desktop → Uninstall

:: === PHASE 3: Remove Docker's WSL distros ===
wsl --unregister docker-desktop 2>nul
wsl --unregister docker-desktop-data 2>nul

:: === PHASE 4: Delete ALL leftover data ===
rmdir /s /q "%LOCALAPPDATA%\Docker" 2>nul
rmdir /s /q "%APPDATA%\Docker" 2>nul
rmdir /s /q "%APPDATA%\Docker Desktop" 2>nul
rmdir /s /q "%PROGRAMDATA%\Docker" 2>nul
rmdir /s /q "%PROGRAMDATA%\DockerDesktop" 2>nul
rmdir /s /q "%USERPROFILE%\.docker" 2>nul

:: === PHASE 5: Delete program files ===
rmdir /s /q "C:\Program Files\Docker" 2>nul

:: === PHASE 6: Clean service registration ===
sc delete com.docker.service 2>nul

:: === PHASE 7: Reboot ===
shutdown /r /t 0
```

**After reboot:**
```cmd
:: Verify clean state
wsl --list --verbose
:: Should show NO docker distros

tasklist | findstr -i vmmem
:: Should return nothing

:: Update WSL
wsl --update

:: Create .wslconfig if you haven't (see Section 6)

:: Install Docker
winget install Docker.DockerDesktop

:: Or download directly: https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe
```

### Docker Desktop Won't Start After Install

```cmd
:: Check Docker's internal logs
type "%LOCALAPPDATA%\Docker\log\vm\dockerd.log" 2>nul
type "%LOCALAPPDATA%\Docker\log\vm\containerd.log" 2>nul
type "%APPDATA%\Docker\log.txt" 2>nul

:: Reset to factory defaults
:: Docker Desktop tray icon → Troubleshoot → Reset to factory defaults

:: Switch daemon mode (sometimes fixes stuck state)
"C:\Program Files\Docker\Docker\DockerCli.exe" -SwitchDaemon 2>nul

:: Check WSL integration
wsl --list --verbose
:: docker-desktop should show as Running
```

### Docker config.json Corruption

The file at `%USERPROFILE%\.docker\config.json` can get corrupted, causing docker commands to hang:

```cmd
:: Backup and recreate
ren "%USERPROFILE%\.docker\config.json" "config.json.bak" 2>nul
echo {} > "%USERPROFILE%\.docker\config.json"
```

### Docker WITHOUT Docker Desktop

If Docker Desktop keeps failing, go native:

```bash
# Inside WSL2 Ubuntu
sudo apt-get update
sudo apt-get install -y docker.io docker-compose-v2

# Start daemon
sudo service docker start

# Add yourself to docker group (no more sudo needed after re-login)
sudo usermod -aG docker $USER
newgrp docker

# Test
docker run hello-world
docker compose version
```

From Windows cmd.exe:
```cmd
wsl docker ps
wsl docker run -p 8080:80 nginx
```

### Port Conflicts

Docker needs specific ports. Silent failures happen when ports are occupied:

```cmd
:: Check common Docker ports
netstat -ano | findstr ":2375 :2376 :445"

:: The hidden killer: Hyper-V reserved port ranges
netsh interface ipv4 show excludedportrange protocol=tcp
:: If Docker's ports fall in these ranges, it silently fails

:: Fix: reset the port reservation system
net stop winnat
net start winnat
```

---

## 8. Hyper-V & Virtualization Layer

### Check Virtualization Status

```cmd
:: Quick check
systeminfo | findstr /i "hyper-v"

:: Task Manager: Performance tab → CPU → "Virtualization: Enabled"

:: Detailed check
powershell -NoProfile -Command "Get-ComputerInfo | Select-Object HyperVRequirementDataExecutionPreventionAvailable, HyperVRequirementSecondLevelAddressTranslation, HyperVRequirementVirtualizationFirmwareEnabled, HyperVRequirementVMMonitorModeExtensions"
```

### Required Windows Features

```cmd
:: Enable all required features for WSL2/Docker
dism /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
dism /online /enable-feature /featurename:Microsoft-Hyper-V-All /all /norestart

:: Verify
dism /online /get-features | findstr -i "hyper virtual subsystem"
```

### Hypervisor Launch Type

```cmd
:: Check current state
bcdedit /enum | findstr -i hypervisor

:: Enable (required for WSL2/Docker)
bcdedit /set hypervisorlaunchtype auto

:: Disable (needed for older VirtualBox/VMware)
bcdedit /set hypervisorlaunchtype off

:: REBOOT REQUIRED for either change
```

### Virtualization Conflicts

| Software | Conflict Level | Notes |
|----------|:---:|-------|
| VirtualBox < 6.0 | ❌ Hard conflict | Cannot coexist with Hyper-V |
| VirtualBox ≥ 6.0 | ⚠️ Degraded | Works but with reduced performance |
| VMware Workstation < 15.5.5 | ❌ Hard conflict | Cannot coexist |
| VMware Workstation ≥ 15.5.5 | ⚠️ Degraded | Works with Hyper-V compatibility mode |
| Android Emulator (old) | ❌ Hard conflict | Use WHPX backend instead |
| QEMU | ✅ Compatible | Uses WHPX acceleration when available |

---

