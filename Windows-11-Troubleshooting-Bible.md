<!-- Windows 11 Troubleshooting Bible — PowerShell — Profiles, Frozen Shell, PSReadLine, Updates, Debugging -->

## 1. PowerShell Profile System — Complete Anatomy

### What Profiles Are

A PowerShell profile is a `.ps1` script that auto-executes every time PowerShell starts. It customizes your environment — aliases, functions, modules, prompt themes, etc. **If a profile script hangs, errors out, or runs an infinite loop, your shell is dead on arrival.**

PowerShell does NOT create profiles by default. They only exist if you (or a tool like oh-my-posh, conda, starship) created one.

### All Profile Paths (Windows)

There are **4 profiles per host**, loaded in this exact order:

| # | Scope | Variable | PowerShell 7+ (pwsh) Path | Windows PowerShell 5.1 Path |
|---|-------|----------|---------------------------|----------------------------|
| 1 | All Users, All Hosts | `$PROFILE.AllUsersAllHosts` | `$PSHOME\Profile.ps1` | `C:\Windows\System32\WindowsPowerShell\v1.0\Profile.ps1` |
| 2 | All Users, Current Host | `$PROFILE.AllUsersCurrentHost` | `$PSHOME\Microsoft.PowerShell_profile.ps1` | `C:\Windows\System32\WindowsPowerShell\v1.0\Microsoft.PowerShell_profile.ps1` |
| 3 | Current User, All Hosts | `$PROFILE.CurrentUserAllHosts` | `$HOME\Documents\PowerShell\Profile.ps1` | `$HOME\Documents\WindowsPowerShell\Profile.ps1` |
| 4 | Current User, Current Host | `$PROFILE.CurrentUserCurrentHost` | `$HOME\Documents\PowerShell\Microsoft.PowerShell_profile.ps1` | `$HOME\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1` |

**Load order: 1 → 2 → 3 → 4.** Last one wins on conflicts. `$PROFILE` by itself points to #4.

### Where `$PSHOME` Actually Is

- **PowerShell 7+:** `C:\Program Files\PowerShell\7\`
- **Windows PowerShell 5.1:** `C:\Windows\System32\WindowsPowerShell\v1.0\`

### Where `$HOME\Documents` Actually Is — THE TRAP

This gets people constantly:

- **Default:** `C:\Users\<you>\Documents\`
- **OneDrive sync enabled?** → `C:\Users\<you>\OneDrive\Documents\` (this breaks modules and profiles CONSTANTLY because OneDrive locks files, syncs them mid-write, and creates conflicts)
- **Folder redirection (domain/GPO)?** → Could be a network path like `\\server\share\Documents\`
- **Check actual path from cmd.exe:** `echo %USERPROFILE%\Documents`

Microsoft explicitly warns: *"We don't recommend redirecting the Documents folder to a network share or including it in OneDrive. Redirecting the folder can cause modules to fail to load and create errors in your profile scripts."*

### VS Code Has Its Own Profile

The PowerShell extension in VS Code uses a separate host-specific profile:
- PS7: `$HOME\Documents\PowerShell\Microsoft.VSCode_profile.ps1`
- PS5.1: `$HOME\Documents\WindowsPowerShell\Microsoft.VSCode_profile.ps1`

This means you can have a broken VS Code profile but a working terminal profile, or vice versa.

### View All Profile Paths At Once

```powershell
$PROFILE | Select-Object *
```

Output example:
```
AllUsersAllHosts       : C:\Program Files\PowerShell\7\profile.ps1
AllUsersCurrentHost    : C:\Program Files\PowerShell\7\Microsoft.PowerShell_profile.ps1
CurrentUserAllHosts    : C:\Users\LO\Documents\PowerShell\profile.ps1
CurrentUserCurrentHost : C:\Users\LO\Documents\PowerShell\Microsoft.PowerShell_profile.ps1
```

### How to Find Profiles WITHOUT PowerShell

Since your PowerShell is broken, use **cmd.exe** (Win+R → `cmd` → Enter):

```cmd
:: Check ALL possible profile locations
echo === PowerShell 7 User Profiles ===
dir "%USERPROFILE%\Documents\PowerShell\*profile*" 2>nul
dir "%USERPROFILE%\OneDrive\Documents\PowerShell\*profile*" 2>nul

echo === Windows PowerShell 5.1 User Profiles ===
dir "%USERPROFILE%\Documents\WindowsPowerShell\*profile*" 2>nul
dir "%USERPROFILE%\OneDrive\Documents\WindowsPowerShell\*profile*" 2>nul

echo === System-wide Profiles ===
dir "C:\Windows\System32\WindowsPowerShell\v1.0\*profile*" 2>nul
dir "C:\Program Files\PowerShell\7\*profile*" 2>nul
```

Or paste these into File Explorer's address bar:
```
%USERPROFILE%\Documents\PowerShell
%USERPROFILE%\Documents\WindowsPowerShell
%USERPROFILE%\OneDrive\Documents\PowerShell
%USERPROFILE%\OneDrive\Documents\WindowsPowerShell
```

### What Tools Inject Into Your Profile

These are the most common culprits that write to your profile and cause breakage:

| Tool | What It Adds | How It Breaks |
|------|-------------|---------------|
| **oh-my-posh** | `oh-my-posh init pwsh \| Invoke-Expression` | Hangs if binary is missing, theme file path is wrong, or you downloaded an HTML page instead of a raw JSON theme |
| **Starship** | `Invoke-Expression (&starship init powershell)` | Hangs if starship binary not in PATH |
| **Conda/Anaconda** | ~20 lines of `conda init` code | Slows startup by 4+ seconds; hangs if conda is broken or PATH is wrong; `auto_activate_base` adds overhead |
| **nvm-windows** | PATH modifications | Can corrupt PATH if multiple Node versions conflict |
| **chocolatey** | `Import-Module $env:ChocolateyInstall\helpers\chocolateyProfile.psm1` | Fails if Chocolatey is partially uninstalled |
| **posh-git** | `Import-Module posh-git` | Slow on large repos; fails if module is outdated |
| **Az (Azure)** | `Import-Module Az` | Massive module, adds 5-15 seconds to startup |
| **PSReadLine config** | Various `Set-PSReadLineOption` calls | Crashes if PSReadLine version doesn't support the options |

### Execution Policy and Profiles

Execution policy determines whether your profile runs at all:

| Policy | Profile Runs? | Notes |
|--------|:---:|-------|
| `Unrestricted` | ✅ | Warns on downloaded scripts |
| `RemoteSigned` | ✅ | Local scripts run freely; downloaded scripts need signatures. **USE THIS** |
| `AllSigned` | ⚠️ | Only if profile is digitally signed |
| `Restricted` | ❌ | No scripts run at all (Windows default!) |
| `Bypass` | ✅ | No restrictions, no warnings |

Check from cmd.exe:
```cmd
powershell -NoProfile -Command "Get-ExecutionPolicy -List"
```

Fix:
```cmd
powershell -NoProfile -Command "Set-ExecutionPolicy RemoteSigned -Scope CurrentUser -Force"
pwsh -NoProfile -Command "Set-ExecutionPolicy RemoteSigned -Scope CurrentUser -Force"
```

---

## 2. PowerShell Frozen / Can't Type — Root Cause Analysis

### Symptoms

- PowerShell opens, shows a cursor, but you **cannot type anything**
- Window title says "PowerShell" but no prompt (`PS C:\>`) ever appears
- Shell opens then immediately closes
- Shell opens but hangs with a blinking cursor for 30+ seconds
- Shell opens with garbled output or error text, then freezes

### Root Causes (Ordered by Likelihood)

**A. Profile Script Is Hanging (90% of cases)**

Your profile calls something that never returns. Most common:

1. **oh-my-posh with bad config**: `oh-my-posh init pwsh | Invoke-Expression` hangs if:
   - oh-my-posh binary was uninstalled or moved
   - Theme file path points to nothing or a corrupted file
   - You downloaded an HTML web page instead of the raw theme JSON
   - oh-my-posh is doing an update check that times out (Issue #5309)

2. **starship init**: Same class of issue — binary missing from PATH

3. **conda init**: The conda initialization block in profile.ps1 adds 4+ seconds. If conda itself is broken, it hangs indefinitely. Fix conda overhead:
   ```cmd
   conda config --set auto_activate_base false
   ```

4. **Import-Module for a missing/broken module**: `Import-Module SomeModule` hangs if the module's `.psm1` file has its own initialization that hangs

5. **Network calls in profile**: Checking for updates, fetching remote configs, hitting APIs — all hang if network is down or DNS is broken

**B. PSReadLine Module Is Broken (see Section 3)**

**C. .NET Runtime Conflict**

PowerShell 7 runs on .NET 8+. If your system has conflicting .NET installations or broken .NET components, PowerShell can fail to initialize. Symptoms: type initializer errors, assembly load failures.

**D. Antivirus Interference**

Some antivirus products (especially Webroot, Kaspersky, and Bitdefender) hook into PowerShell's script execution engine and can cause hangs, especially during profile loading.

**E. Windows Terminal vs Legacy Console**

If you're using ConHost (old black window), known input bugs exist with certain PSReadLine versions. Windows Terminal handles PSReadLine better.

### The Fix — Step by Step

**Step 1: Bypass the profile entirely**

Open **cmd.exe** (Win+R → `cmd` → Enter):

```cmd
:: For PowerShell 7+
pwsh -NoProfile

:: For Windows PowerShell 5.1
powershell -NoProfile
```

If this works → profile is the problem. Go to Step 2.
If this ALSO hangs → PSReadLine or .NET issue. Go to Section 3.

**Step 2: Disable all profiles at once**

From cmd.exe:
```cmd
:: Rename PS7 profiles
ren "%USERPROFILE%\Documents\PowerShell\Microsoft.PowerShell_profile.ps1" "Microsoft.PowerShell_profile.ps1.broken" 2>nul
ren "%USERPROFILE%\Documents\PowerShell\Profile.ps1" "Profile.ps1.broken" 2>nul

:: Rename PS5.1 profiles
ren "%USERPROFILE%\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1" "Microsoft.PowerShell_profile.ps1.broken" 2>nul
ren "%USERPROFILE%\Documents\WindowsPowerShell\Profile.ps1" "Profile.ps1.broken" 2>nul

:: Check OneDrive paths too
ren "%USERPROFILE%\OneDrive\Documents\PowerShell\Microsoft.PowerShell_profile.ps1" "Microsoft.PowerShell_profile.ps1.broken" 2>nul
ren "%USERPROFILE%\OneDrive\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1" "Microsoft.PowerShell_profile.ps1.broken" 2>nul

:: Check system-wide profiles (requires admin cmd)
ren "C:\Program Files\PowerShell\7\Microsoft.PowerShell_profile.ps1" "Microsoft.PowerShell_profile.ps1.broken" 2>nul
ren "C:\Windows\System32\WindowsPowerShell\v1.0\Microsoft.PowerShell_profile.ps1" "Microsoft.PowerShell_profile.ps1.broken" 2>nul
```

**Step 3: Test**
```cmd
pwsh
powershell
```

Both should open clean now.

**Step 4: Read the broken profile to find the culprit**
```cmd
type "%USERPROFILE%\Documents\PowerShell\Microsoft.PowerShell_profile.ps1.broken"
type "%USERPROFILE%\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1.broken"
```

Look for the patterns described in the table above.

**Step 5: Create a clean profile**

From working PowerShell (`pwsh -NoProfile`):
```powershell
# Create minimal profile
@'
# Clean PowerShell Profile - rebuilt after breakage
# Add customizations below this line

'@ | Set-Content -Path $PROFILE -Encoding UTF8
```

**Step 6: Re-add your customizations one at a time**

Add one line, restart PowerShell, test. Repeat. This isolates exactly which line breaks things.

---

## 3. PSReadLine — The Module That Controls Your Keyboard

### What PSReadLine Does

PSReadLine is the module that provides:
- Keyboard input handling in the console
- Syntax highlighting
- Command history with search (Ctrl+R)
- Tab completion behavior
- Multi-line editing
- Key bindings

**If PSReadLine is broken, you literally cannot type in PowerShell.**

### Common PSReadLine Problems

1. **Version conflict**: PS7 ships PSReadLine 2.3.x, but you might have an older version installed in your user module path that loads first
2. **Multiple versions installed**: `Get-Module PSReadLine -ListAvailable` shows more than one → conflict
3. **.NET runtime mismatch**: PSReadLine compiled for one .NET version, PowerShell running another
4. **Corrupt module files**: Partial update, disk error, or antivirus quarantine
5. **Type initializer error**: After Windows 11 upgrade, PSReadLine's type initializer fails

### Diagnosing PSReadLine Issues

From cmd.exe:
```cmd
:: Check if PS7 works without PSReadLine
pwsh -NoProfile -Command "Remove-Module PSReadLine -ErrorAction SilentlyContinue; Write-Host 'PSReadLine removed, can you type?'"

:: Check PSReadLine version
pwsh -NoProfile -Command "Get-Module PSReadLine -ListAvailable | Format-Table Name, Version, ModuleBase"

:: Check for multiple versions
pwsh -NoProfile -Command "(Get-Module PSReadLine -ListAvailable).Count"
```

### Fixing PSReadLine

**Option 1: Update PSReadLine** (most common fix)
```cmd
pwsh -NoProfile -Command "Install-Module PSReadLine -Force -SkipPublisherCheck"
```

**Option 2: Remove all versions and reinstall**
```cmd
pwsh -NoProfile -Command "Uninstall-Module PSReadLine -AllVersions -Force; Install-Module PSReadLine -Force -SkipPublisherCheck"
```

**Option 3: Nuclear — delete PSReadLine module folder manually**

From cmd.exe:
```cmd
:: Find where PSReadLine lives
pwsh -NoProfile -Command "Get-Module PSReadLine -ListAvailable | Select-Object ModuleBase"

:: Delete it (replace path with your actual path)
rmdir /s /q "%USERPROFILE%\Documents\PowerShell\Modules\PSReadLine"

:: Restart PowerShell — it will use the built-in version
pwsh
```

**Option 4: For Windows PowerShell 5.1 specifically**

PSReadLine for PS5.1 can get stuck on an old version. The fix requires admin:
```cmd
:: Run as admin
powershell -NoProfile -Command "Install-Module PSReadLine -Force -SkipPublisherCheck -AllowPrerelease"
```

If that fails with "cannot load":
```cmd
powershell -NoProfile -Command "[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12; Install-Module PSReadLine -Force -SkipPublisherCheck"
```

### PSReadLine Configuration That Can Break Things

If your profile contains `Set-PSReadLineOption` calls that reference options not available in your installed version:

```powershell
# This crashes on older PSReadLine versions:
Set-PSReadLineOption -PredictionSource History  # Added in 2.2.0
Set-PSReadLineOption -PredictionViewStyle ListView  # Added in 2.2.0
```

Fix: Wrap in version checks:
```powershell
if ((Get-Module PSReadLine).Version -ge [version]'2.2.0') {
    Set-PSReadLineOption -PredictionSource History
    Set-PSReadLineOption -PredictionViewStyle ListView
}
```

---

## 4. PowerShell Update Gone Wrong — Side-by-Side Conflicts

### The Two PowerShells — They Are COMPLETELY Separate

| | Windows PowerShell | PowerShell 7+ |
|---|---|---|
| Executable | `powershell.exe` | `pwsh.exe` |
| Version | 5.1 (frozen forever) | 7.x (actively developed) |
| Ships with Windows | Yes | No (install separately) |
| .NET Runtime | .NET Framework 4.x | .NET 8/9+ |
| Module path | `$HOME\Documents\WindowsPowerShell\Modules` | `$HOME\Documents\PowerShell\Modules` |
| Profile folder | `WindowsPowerShell` | `PowerShell` |
| Registry PSModulePath | Includes both | Includes both + PS7 paths |

### Module Path Conflicts

`$env:PSModulePath` includes paths for BOTH versions. Modules compiled for .NET Framework (5.1) may not work in .NET 8 (7+) and vice versa:

```powershell
# Check module paths
$env:PSModulePath -split ';'
```

Symptoms of module path conflicts:
- `Import-Module` failures with type load exceptions
- Assembly binding redirect errors
- "Could not load type" or "Method not found" errors
- Modules work in 5.1 but crash in 7 (or vice versa)

### Clean Reinstall of PowerShell 7

```cmd
:: Uninstall first
winget uninstall Microsoft.PowerShell

:: Kill any remaining processes
taskkill /f /im pwsh.exe 2>nul

:: Clean up leftover module conflicts
rmdir /s /q "%USERPROFILE%\Documents\PowerShell\Modules\PSReadLine" 2>nul

:: Reinstall
winget install Microsoft.PowerShell --source winget

:: Verify
pwsh -NoProfile -Command "$PSVersionTable"
```

Alternative install methods:
```cmd
:: From Microsoft Store (MSIX, auto-updates)
winget install --id 9MZ1SNWT0N5D --source msstore

:: Direct MSI from GitHub
:: https://github.com/PowerShell/PowerShell/releases
msiexec /i PowerShell-7.x.x-win-x64.msi /qn ADD_EXPLORER_CONTEXT_MENU_OPENPOWERSHELL=1 ADD_FILE_CONTEXT_MENU_RUNPOWERSHELL=1 ENABLE_PSREMOTING=0 REGISTER_MANIFEST=1
```

### Repair Windows PowerShell 5.1

Windows PowerShell is a Windows component — you can't uninstall/reinstall it normally:

```cmd
:: Step 1: Repair system files (fixes corrupted PS5.1 components)
DISM /Online /Cleanup-Image /RestoreHealth
sfc /scannow

:: Step 2: Re-enable the Windows feature
:: GUI: Settings → Apps → Optional Features → Add a feature → "Windows PowerShell"
:: Or: Control Panel → Programs → Turn Windows features on/off → Windows PowerShell 2.0

:: Step 3: Reset Windows PowerShell's execution environment
powershell -NoProfile -Command "Register-PSSessionConfiguration -Name Microsoft.PowerShell -Force"

:: Step 4: Fix .NET Framework (PS5.1 depends on it)
:: GUI: Control Panel → Programs → Turn Windows features on/off → .NET Framework 3.5 and 4.8+
```

### PATH Conflicts After Update

```cmd
:: Check for duplicate or stale PowerShell entries
where pwsh
where powershell

:: Should show:
:: pwsh → C:\Program Files\PowerShell\7\pwsh.exe
:: powershell → C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
```

Fix stale PATH entries:
1. Win+R → `sysdm.cpl` → Advanced → Environment Variables
2. Check both User PATH and System PATH
3. Remove any stale PowerShell 7 paths (old version folders)
4. Ensure `C:\Program Files\PowerShell\7\` is in System PATH

---

## 5. Advanced PowerShell Debugging & Forensics

### Trace Profile Loading

See exactly where your profile hangs:

```cmd
:: Verbose tracing from cmd.exe
pwsh -NoProfile -Command "& { $VerbosePreference = 'Continue'; . $PROFILE }"

:: More detailed command discovery trace
pwsh -NoProfile -Command "& { Trace-Command -Name CommandDiscovery -Expression { . $PROFILE } -PSHost }"

:: Time each section of your profile
pwsh -NoProfile -Command "& { $sw = [Diagnostics.Stopwatch]::StartNew(); . $PROFILE; Write-Host ('Profile loaded in {0}ms' -f $sw.ElapsedMilliseconds) }"
```

### Profile Performance Profiling

Add this to the TOP of your profile temporarily:

```powershell
$profileStopwatch = [System.Diagnostics.Stopwatch]::StartNew()
$profileMarkers = @()
function Mark-ProfileTime($label) {
    $script:profileMarkers += [PSCustomObject]@{
        Label = $label
        ElapsedMs = $profileStopwatch.ElapsedMilliseconds
    }
}
```

Then sprinkle `Mark-ProfileTime "after oh-my-posh"` between sections. At the end:

```powershell
$profileStopwatch.Stop()
$profileMarkers | Format-Table -AutoSize
Write-Host "Total: $($profileStopwatch.ElapsedMilliseconds)ms"
```

### Event Viewer Logs for PowerShell

Open Event Viewer: `eventvwr.msc`

**Key logs:**

| Log Path | Event IDs | What It Shows |
|----------|-----------|---------------|
| Applications and Services → Microsoft → Windows → PowerShell → Operational | 4103 | Module logging — what modules loaded |
| Same | 4104 | Script block logging — what code ran |
| Same | 40961/40962 | Engine start/stop |
| Windows Logs → Windows PowerShell | 400 | Engine started |
| Same | 403 | Engine stopped |
| Same | 600 | Provider started |
| Same | 800 | Pipeline execution (if enabled) |

### Enable Full PowerShell Logging

**Script Block Logging** (logs every script block executed):
```cmd
:: Via registry (from admin cmd.exe)
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" /v EnableScriptBlockLogging /t REG_DWORD /d 1 /f
```

**Transcription** (full session recording to file):
```cmd
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\Transcription" /v EnableTranscripting /t REG_DWORD /d 1 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\Transcription" /v OutputDirectory /t REG_SZ /d "C:\PSTranscripts" /f
mkdir C:\PSTranscripts 2>nul
```

**Module Logging:**
```cmd
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging" /v EnableModuleLogging /t REG_DWORD /d 1 /f
```

### Quick Diagnostics Battery (from cmd.exe)

Run all of these when PowerShell is acting up:

```cmd
:: PS7 version
pwsh -NoProfile -Command "$PSVersionTable" 2>&1

:: PS5.1 version
powershell -NoProfile -Command "$PSVersionTable" 2>&1

:: Execution policy
powershell -NoProfile -Command "Get-ExecutionPolicy -List" 2>&1

:: Profile existence
pwsh -NoProfile -Command "Test-Path $PROFILE" 2>&1

:: PSReadLine version
pwsh -NoProfile -Command "Get-Module PSReadLine -ListAvailable" 2>&1

:: All installed modules (potential conflicts)
pwsh -NoProfile -Command "Get-Module -ListAvailable | Select-Object Name, Version, ModuleBase | Sort-Object Name" 2>&1

:: .NET version
dotnet --list-runtimes 2>&1

:: PowerShell module paths
pwsh -NoProfile -Command "$env:PSModulePath -split ';'" 2>&1
```

---



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



<!-- Windows 11 Troubleshooting Bible — Network Stack Troubleshooting -->

## 13. Network Stack Troubleshooting

### The Complete Network Reset Sequence

Run in this exact order from admin cmd.exe:

```cmd
:: Step 1: Reset Winsock (network socket layer)
netsh winsock reset catalog

:: Step 2: Reset TCP/IP stack
netsh int ip reset

:: Step 3: Reset IPv6 stack
netsh int ipv6 reset

:: Step 4: Flush DNS cache
ipconfig /flushdns

:: Step 5: Re-register DNS
ipconfig /registerdns

:: Step 6: Release current IP
ipconfig /release

:: Step 7: Renew IP
ipconfig /renew

:: Step 8: REBOOT (required for winsock/TCP/IP reset to take effect)
shutdown /r /t 0
```

### What Each Command Does

| Command | What It Resets | When To Use |
|---------|---------------|-------------|
| `netsh winsock reset` | Winsock catalog (socket API) | After malware, bad network drivers, VPN uninstall |
| `netsh int ip reset` | TCP/IP stack registry entries | "Limited connectivity", DHCP failures, IP conflicts |
| `ipconfig /flushdns` | Local DNS cache | Stale DNS entries, can't reach websites that recently changed IPs |
| `ipconfig /registerdns` | Forces DNS re-registration | Computer not visible on network |
| `ipconfig /release` + `/renew` | DHCP lease | Get a fresh IP address |

### DNS Troubleshooting

```cmd
:: Check current DNS servers
ipconfig /all | findstr "DNS Servers"

:: Test DNS resolution
nslookup google.com
nslookup google.com 8.8.8.8  :: Test with Google's DNS directly

:: If first fails but second works → your DNS server is the problem
:: Fix: Change DNS to 8.8.8.8 / 8.8.4.4 (Google) or 1.1.1.1 / 1.0.0.1 (Cloudflare)
```

### Proxy Issues

```cmd
:: Check system proxy settings
netsh winhttp show proxy

:: Reset proxy
netsh winhttp reset proxy

:: Check environment proxy variables
echo %HTTP_PROXY%
echo %HTTPS_PROXY%
echo %NO_PROXY%
```

### Firewall Troubleshooting

```cmd
:: Check Windows Firewall status
netsh advfirewall show allprofiles state

:: Temporarily disable (TESTING ONLY)
netsh advfirewall set allprofiles state off

:: Re-enable
netsh advfirewall set allprofiles state on

:: Check if a specific port is allowed
netsh advfirewall firewall show rule name=all dir=in | findstr "8080"

:: Add a firewall rule
netsh advfirewall firewall add rule name="Docker" dir=in action=allow protocol=tcp localport=2375
```

---



<!-- Windows 11 Troubleshooting Bible — BSOD/Crash Dumps, Sysinternals Deep Dive, Event Viewer -->

## 14. Blue Screen (BSOD) & Crash Dump Analysis

### Minidump File Location

```
C:\Windows\Minidump\
```

Each BSOD creates a `.dmp` file here. If the folder doesn't exist or is empty:

```cmd
:: Enable minidumps
reg add "HKLM\SYSTEM\CurrentControlSet\Control\CrashControl" /v CrashDumpEnabled /t REG_DWORD /d 3 /f
reg add "HKLM\SYSTEM\CurrentControlSet\Control\CrashControl" /v MinidumpDir /t REG_EXPAND_SZ /d "%SystemRoot%\Minidump" /f
```

### Quick Analysis with WinDbg

1. Install WinDbg: `winget install Microsoft.WinDbg`
2. Open WinDbg → File → Open Dump File → select `.dmp` from `C:\Windows\Minidump\`
3. Set symbol path: `srv*https://msdl.microsoft.com/download/symbols`
4. Type: `!analyze -v` and press Enter
5. Look for:
   - **BUGCHECK_STR**: The type of crash
   - **FAULTING_MODULE / MODULE_NAME**: The driver/module that caused it
   - **IMAGE_NAME**: The specific file responsible
   - **STACK_TEXT**: The call stack at crash time

### Common BSOD Stop Codes

| Stop Code | Common Cause | Fix |
|-----------|-------------|-----|
| `IRQL_NOT_LESS_OR_EQUAL` | Bad driver (usually network/GPU) | Update drivers |
| `SYSTEM_SERVICE_EXCEPTION` | Faulty driver or system file | SFC/DISM, update drivers |
| `KERNEL_DATA_INPAGE_ERROR` | Disk/SSD failure | Run `chkdsk /f /r C:` |
| `NTFS_FILE_SYSTEM` | Disk corruption | `chkdsk /f /r C:` |
| `CRITICAL_PROCESS_DIED` | System process crashed | Safe Mode → SFC/DISM |
| `DPC_WATCHDOG_VIOLATION` | Driver taking too long | Update SSD/storage drivers |
| `SYSTEM_THREAD_EXCEPTION_NOT_HANDLED` | Specific driver crashing | Check minidump for driver name |
| `HYPERVISOR_ERROR` | Hyper-V/WSL issue | Update Windows, check virtualization |
| `WHEA_UNCORRECTABLE_ERROR` | Hardware failure (RAM/CPU) | Run `mdsched.exe` (memory diagnostic) |

### Without WinDbg — Quick Triage

```cmd
:: Use built-in tools to read minidumps
powershell -NoProfile -Command "Get-WinEvent -FilterHashtable @{LogName='System'; Id=1001} -MaxEvents 5 | Format-List TimeCreated, Message"

:: Or check Reliability Monitor
perfmon /rel
:: Shows BSOD history with timestamps and faulting modules
```

---

## 15. Power User Debugging Toolkit

### Sysinternals Suite

Install: `winget install Microsoft.Sysinternals` or visit `https://live.sysinternals.com`

### Process Explorer — Task Manager Replacement

**What it does that Task Manager doesn't:**
- Shows DLL list per process
- Shows open handles (files, registry keys, mutexes, network connections)
- Verifies digital signatures on processes
- Submits to VirusTotal for malware scanning
- Shows process tree (parent-child relationships)
- Shows per-process GPU, I/O, and network usage

**Key actions:**
- **Find DLL/Handle:** Ctrl+F → search for a filename → shows which process has it locked
- **Replace Task Manager:** Options → Replace Task Manager
- **Verify signatures:** Options → Verify Image Signatures (unsigned = suspicious)
- **VirusTotal:** Options → VirusTotal.com → Check VirusTotal.com

### Process Monitor (ProcMon) — Real-Time System Activity

**The single most powerful debugging tool on Windows.** Shows every file access, registry read/write, network connection, and process start/stop in real-time.

**Use cases:**
- See what an installer is doing that causes it to stall
- Find "Access Denied" errors that apps swallow silently
- Trace what happens during PowerShell startup
- Find which process is writing to a file
- Diagnose why an app can't find a DLL or config file

**Essential ProcMon filters:**

| What You're Debugging | Filter |
|----------------------|--------|
| PowerShell startup hang | `Process Name is pwsh.exe` |
| Docker install stall | `Process Name is Docker Desktop Installer.exe` |
| App can't find DLL | `Path contains .dll AND Result is NAME NOT FOUND` |
| Permission errors | `Result is ACCESS DENIED` |
| Specific file access | `Path contains <filename>` |

**Command-line ProcMon:**
```cmd
:: Capture to file silently (great for boot-time issues)
procmon64.exe /AcceptEula /Quiet /Minimized /BackingFile C:\temp\procmon.pml

:: Boot logging (captures what happens during Windows startup)
procmon64.exe /AcceptEula /EnableBootLogging
:: Reboot, then open the saved log
```

### Autoruns — Everything That Auto-Starts

Shows ALL autostart locations — more than you knew existed:
- Registry Run keys (HKCU and HKLM)
- Startup folder items
- Services
- Drivers
- Scheduled tasks
- WMI event subscriptions
- Shell extensions
- Browser helper objects
- Codecs
- And more

**Power user tips:**
- **Hide Microsoft entries:** Options → Hide Microsoft Entries (shows only third-party)
- **Compare snapshots:** File → Save → (make changes) → File → Compare (find what changed)
- **Disable without deleting:** Uncheck an entry to disable it temporarily

### TCPView — Network Connection Monitor

Real-time view of all TCP/UDP connections:
- Which process is using which port
- Connection state (ESTABLISHED, LISTENING, TIME_WAIT)
- Remote address and port
- Traffic volume

**Essential for:** Docker port conflicts, finding what's using port 80/443/3000/etc.

### Handle — Find Who's Locking a File

```cmd
:: Find which process has a file open
handle64.exe "filename.dll"
handle64.exe "C:\path\to\locked\file"

:: Find all handles for a specific process
handle64.exe -p <PID>
```

---

## 16. Event Viewer — Complete Guide

### Opening Event Viewer

```cmd
eventvwr.msc
```

### Key Logs

| Log | Path in Event Viewer | What It Shows |
|-----|---------------------|---------------|
| Application | Windows Logs → Application | App crashes (1000), install events (11724), .NET errors |
| System | Windows Logs → System | Driver failures, service crashes, BSOD info, disk errors |
| Security | Windows Logs → Security | Login attempts, privilege changes (needs audit enabled) |
| Setup | Windows Logs → Setup | Windows Update install results |
| PowerShell Operational | Apps & Services → Microsoft → Windows → PowerShell → Operational | Every PS command, script blocks, module loads |
| Windows PowerShell | Apps & Services → Windows PowerShell | PS5.1 engine events |
| Windows Update | Apps & Services → Microsoft → Windows → WindowsUpdateClient → Operational | Update download/install details |
| Task Scheduler | Apps & Services → Microsoft → Windows → TaskScheduler → Operational | Scheduled task results |

### Critical Event IDs

**Application Log:**
| Event ID | Meaning |
|----------|---------|
| 1000 | Application crash (faulting module info) |
| 1001 | Windows Error Reporting bucket |
| 1002 | Application hang |
| 11724 | MSI install complete |
| 11707 | MSI install successful |
| 11708 | MSI install failed |

**System Log:**
| Event ID | Meaning |
|----------|---------|
| 1 | BSOD (BugCheck) |
| 41 | Unexpected shutdown (kernel power) |
| 7000-7034 | Service errors (see Section 12) |
| 6005 | Event Log service started (system boot) |
| 6006 | Event Log service stopped (clean shutdown) |
| 6008 | Unexpected shutdown |
| 219 | Driver failed to load |

### Querying Event Viewer from Command Line

```cmd
:: Last 20 application errors
wevtutil qe Application /q:"*[System[(Level=2)]]" /c:20 /f:text

:: PowerShell script blocks in the last hour
wevtutil qe "Microsoft-Windows-PowerShell/Operational" /q:"*[System[TimeCreated[timediff(@SystemTime) <= 3600000]]]" /c:50 /f:text

:: Export a log for analysis
wevtutil epl System C:\temp\system.evtx
```

---



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



<!-- Windows 11 Troubleshooting Bible — Quick Reference Cards — Diagnostic Command Cheat Sheets -->

## 22. Quick Reference Cards

### PowerShell Emergency Commands (from cmd.exe)

| Command | What It Does |
|---------|--------------|
| `pwsh -NoProfile` | Start PS7 without any profile |
| `powershell -NoProfile` | Start PS5.1 without any profile |
| `pwsh -NoProfile -Command "..."` | Run a single command without profile |
| `where pwsh` | Find PS7 install location |
| `where powershell` | Find PS5.1 location |
| `type "%USERPROFILE%\Documents\PowerShell\Microsoft.PowerShell_profile.ps1"` | Read PS7 profile |
| `ren "%USERPROFILE%\Documents\PowerShell\Microsoft.PowerShell_profile.ps1" "profile.bak"` | Disable PS7 profile |

### WSL Emergency Commands

| Command | What It Does |
|---------|--------------|
| `wsl --shutdown` | Kill ALL WSL instances and Vmmem |
| `wsl --list --verbose` | Show all distros with state |
| `wsl --status` | WSL version and kernel info |
| `wsl --update` | Update WSL kernel |
| `wsl --update --web-download` | Force download update |
| `wsl --unregister <name>` | Delete a distro (DESTRUCTIVE) |
| `wsl --install -d Ubuntu` | Install a fresh distro |
| `net stop LxssManager` | Force-stop WSL service |
| `notepad %USERPROFILE%\.wslconfig` | Edit WSL resource limits |

### System Repair Commands

| Command | What It Does |
|---------|--------------|
| `DISM /Online /Cleanup-Image /RestoreHealth` | Repair Windows image |
| `sfc /scannow` | Scan and repair system files |
| `chkdsk /f /r C:` | Check and repair disk |
| `mdsched.exe` | Memory diagnostic |
| `systeminfo` | Full system info |
| `bcdedit /enum` | Boot configuration |

### Network Commands

| Command | What It Does |
|---------|--------------|
| `netsh winsock reset` | Reset network sockets |
| `netsh int ip reset` | Reset TCP/IP stack |
| `ipconfig /flushdns` | Clear DNS cache |
| `ipconfig /all` | All network adapter details |
| `netstat -ano` | All active connections with PIDs |
| `netsh interface ipv4 show excludedportrange protocol=tcp` | Hyper-V reserved ports |

### Sysinternals Quick Start

| Tool | Install | Use |
|------|---------|-----|
| Process Explorer | `procexp64.exe` | Replace Task Manager |
| Process Monitor | `procmon64.exe` | Trace file/registry/network activity |
| Autoruns | `autoruns64.exe` | Audit all autostart locations |
| TCPView | `tcpview64.exe` | Network connections per process |
| Handle | `handle64.exe <file>` | Find who's locking a file |
| PsExec | `psexec64.exe -s cmd` | Run cmd as SYSTEM account |

---



<!-- Windows 11 Troubleshooting Bible — Hardware/Drivers, Audio, Bluetooth, Display/GPU -->

## 23. Hardware, Drivers & Device Manager

### Device Manager Error Codes

Open Device Manager: `devmgmt.msc`

Devices with problems show a yellow triangle (⚠️). Double-click the device → General tab to see the error code.

| Code | Meaning | Fix |
|------|---------|-----|
| 1 | Device not configured | Install/update driver |
| 3 | Driver corrupted | Uninstall device, reboot (Windows reinstalls) |
| 10 | Device cannot start | Hardware failure or driver issue; try rolling back driver |
| 12 | Not enough resources | IRQ/memory conflict; disable conflicting device |
| 18 | Reinstall driver | Uninstall, reboot, reinstall |
| 19 | Registry problem | Uninstall device, reboot |
| 22 | Device disabled | Right-click → Enable device |
| 28 | Driver not installed | Install driver manually |
| 31 | Device not working properly | Update driver or check Windows Update |
| 39 | Driver corrupted or missing | Uninstall, reboot, reinstall |
| 43 | Device stopped (reported problem) | Often GPU overheating or hardware failure |
| 52 | Driver not signed | Disable driver signature enforcement (test mode) or find signed driver |

### Driver Management Commands

```cmd
:: List all drivers
driverquery /v

:: List drivers with sign status
driverquery /si

:: Export driver list
driverquery /v /fo csv > C:\temp\drivers.csv

:: Check for problematic drivers (unsigned)
sigverif
:: Opens File Signature Verification tool

:: Backup all installed drivers
mkdir C:\DriverBackup
dism /online /export-driver /destination:C:\DriverBackup

:: Install a driver from backup
pnputil /add-driver C:\DriverBackup\*.inf /subdirs /install
```

### Rollback a Driver

1. Device Manager → Right-click device → Properties → Driver tab → Roll Back Driver
2. If grayed out, Windows didn't keep the previous driver. You'll need to manually install the old version.

### Force Driver Installation

```cmd
:: Add a driver to the store
pnputil /add-driver driver.inf

:: Remove a driver from the store
pnputil /delete-driver oem42.inf /force

:: Enumerate all OEM drivers
pnputil /enum-drivers
```

### USB Troubleshooting

```cmd
:: Reset USB controllers
:: Device Manager → Universal Serial Bus controllers → Uninstall ALL "USB Root Hub" entries → Reboot

:: Check USB power
:: Device Manager → USB Root Hub → Properties → Power tab → shows per-port power allocation

:: Fix USB selective suspend (causes devices to disconnect)
:: Control Panel → Hardware and Sound → Power Options → Change plan settings →
:: Change advanced power settings → USB settings → USB selective suspend → Disable
```

---

## 24. Audio Troubleshooting

### No Sound — Diagnostic Checklist

```cmd
:: 1. Check audio service
sc query AudioSrv
:: Should show: STATE = RUNNING

:: 2. If not running
net start AudioSrv
net start AudioEndpointBuilder

:: 3. Check audio devices
:: Settings → System → Sound → Output → make sure correct device is selected

:: 4. Check volume mixer
sndvol
:: Verify per-app volumes aren't muted
```

### Realtek Audio Issues (Most Common)

After Windows Update frequently breaks Realtek drivers:

1. **Uninstall and reinstall:**
   - Device Manager → Sound, video and game controllers → Realtek → Uninstall device → check "Delete the driver software" → Reboot
   - Windows will install a generic "High Definition Audio Device" driver
   - Then install the latest Realtek driver from your motherboard manufacturer's website (NOT from Realtek directly — they're often outdated)

2. **Microsoft UAA Bus Driver fix:**
   - Device Manager → System Devices → Microsoft UAA Bus Driver for High Definition Audio
   - Right-click → Disable → Right-click → Enable

3. **Disable audio enhancements:**
   - Right-click speaker icon → Sound settings → Output device → Properties → Disable audio enhancements

### Audio Services

| Service | Purpose |
|---------|---------|
| `AudioSrv` (Windows Audio) | Core audio playback and recording |
| `AudioEndpointBuilder` | Manages audio endpoints (speakers, headphones) |
| `RtkAudioUniversalService` | Realtek driver service |

```cmd
:: Restart all audio services
net stop AudioSrv & net stop AudioEndpointBuilder
net start AudioEndpointBuilder & net start AudioSrv
```

### Hidden Audio Diagnostic

```cmd
:: Run from cmd
msdt.exe /id AudioPlaybackDiagnostic
:: Opens the built-in audio troubleshooter with more options than Settings
```

---

## 25. Bluetooth Troubleshooting

### Bluetooth Disappeared

Three common causes: disabled adapter, crashed driver, or power management turned it off.

```cmd
:: Check Bluetooth service
sc query bthserv
:: Should be RUNNING

:: Start if stopped
net start bthserv

:: Check Bluetooth Support Service
sc query BluetoothUserService_*
```

### Fix Missing Bluetooth

1. **Check Device Manager** for "Bluetooth" category
   - If missing entirely: adapter is dead, disabled in BIOS, or driver is gone
   - If has yellow triangle: driver issue

2. **BIOS/UEFI check:** Some laptops have a hardware switch or BIOS setting for Bluetooth

3. **Airplane mode:** Make sure it's OFF (Settings → Network & internet → Airplane mode)

4. **Driver reinstall:**
   ```cmd
   :: Uninstall from Device Manager, then scan for hardware changes
   :: Or force Windows to rediscover:
   pnputil /scan-devices
   ```

5. **Power management causing disconnects:**
   - Device Manager → Bluetooth adapter → Properties → Power Management tab
   - UNCHECK "Allow the computer to turn off this device to save power"

### Bluetooth Audio Not Working

- Settings → System → Sound → Output → Select your Bluetooth device
- If listed as both "headset" and "headphones": Headphones = A2DP (high quality), Headset = HFP (low quality for calls)
- Remove device and re-pair if audio profile is stuck on HFP

---

## 26. Display & GPU Troubleshooting

### Graphics Stack Emergency Reset

```
Win + Ctrl + Shift + B
```

Screen will briefly flash black. This restarts the graphics driver without rebooting. Use when:
- Screen is frozen but mouse moves
- Display output is garbled
- Second monitor went black

### Black Screen After Update

1. Boot into Safe Mode (see Section 19)
2. Uninstall GPU driver:
   ```cmd
   :: From Safe Mode cmd
   pnputil /enum-drivers | findstr /i "display"
   pnputil /delete-driver oem##.inf /force
   ```
3. Reboot — Windows will use basic display driver
4. Install fresh GPU driver from NVIDIA/AMD/Intel website

### Multi-Monitor Issues

```cmd
:: Check connected displays
:: Settings → System → Display → shows all connected monitors

:: Reset display configuration
:: Delete per-display registry entries:
:: HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\DisplaySettings

:: If external monitor not detected:
:: Win+P → choose display mode (Duplicate, Extend, Second screen only)
```

### Display Scaling on Mixed-DPI Setups

When monitors have different DPI (e.g., 4K laptop + 1080p external):
- Settings → System → Display → select each monitor → Scale → set individually
- Some apps don't respect per-monitor DPI: Right-click app .exe → Properties → Compatibility → Change high DPI settings → Override → Application

### Screen Flickering

To determine if it's a driver or app issue:
1. Open Task Manager (Ctrl+Shift+Esc)
2. If Task Manager **also flickers** → display driver issue
3. If Task Manager **doesn't flicker** → incompatible app issue

---



<!-- Windows 11 Troubleshooting Bible — User Profile Corruption, BitLocker/Encryption, Group Policy -->

## 27. User Profile Corruption

### Symptoms

- "The User Profile Service failed the sign-in"
- Login screen loops back to lock screen
- Temporary profile loaded (no desktop, no files)
- New user accounts can't be created

### Fix Via Registry (SID Repair)

```cmd
:: Boot into Safe Mode with admin account
:: Open Registry Editor
regedit

:: Navigate to:
:: HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList

:: Find the S-1-5-21-... entry for the broken user
:: (Click each one and check ProfileImagePath in the right pane)

:: If you see two entries with the same SID (one ending in .bak):
:: 1. Rename the one WITHOUT .bak to .bak2
:: 2. Rename the one WITH .bak to remove the .bak
:: 3. Click the renamed entry (without .bak now)
:: 4. Set RefCount to 0
:: 5. Set State to 0
:: 6. Reboot
```

### NTUSER.DAT Corruption

The user's registry hive is stored in `C:\Users\<username>\NTUSER.DAT`. If this file is corrupt:

```cmd
:: Copy from default profile
copy "C:\Users\Default\NTUSER.DAT" "C:\Users\<broken_user>\NTUSER.DAT.bak"

:: Or copy from another working user account
:: (This resets all user settings but preserves files)

:: Check if default profile itself is corrupt
:: (prevents new account creation)
dir C:\Users\Default\NTUSER.DAT
:: If 0 bytes or missing → copy from another Windows 11 machine or extract from install.wim
```

### Create a New Profile When All Else Fails

1. Create a new local admin account from Safe Mode:
   ```cmd
   net user TempAdmin P@ssw0rd /add
   net localgroup Administrators TempAdmin /add
   ```
2. Log in as TempAdmin
3. Copy your files from `C:\Users\<old_profile>\` to the new profile
4. Transfer app settings from `C:\Users\<old_profile>\AppData\`

---

## 28. BitLocker & Encryption

### Finding Your Recovery Key

If BitLocker is asking for a recovery key you don't remember:

1. **Microsoft Account:** Go to https://account.microsoft.com/devices/recoverykey
2. **Azure AD:** IT admin can retrieve it from Azure portal
3. **Printed/saved:** Check if you saved a `.txt` or printed it when enabling BitLocker
4. **USB drive:** Check USB drives for `BitLocker Recovery Key *.txt`

### BitLocker Commands

```cmd
:: Check BitLocker status on all drives
manage-bde -status

:: Unlock a drive with recovery key
manage-bde -unlock D: -RecoveryPassword 123456-789012-...

:: Suspend BitLocker (for BIOS updates, etc.)
manage-bde -protectors -disable C:

:: Resume BitLocker
manage-bde -protectors -enable C:

:: Turn off BitLocker entirely
manage-bde -off C:

:: Backup recovery key to Microsoft account
manage-bde -protectors -adbackup C: -id {key-protector-id}
```

### BitLocker Recovery Loop

When Windows keeps asking for the recovery key every boot:

```cmd
:: From WinRE Command Prompt:
:: 1. Unlock the drive
manage-bde -unlock C: -RecoveryPassword YOUR-RECOVERY-KEY

:: 2. Suspend protection
manage-bde -protectors -disable C:

:: 3. Boot normally, then re-enable
manage-bde -protectors -enable C:

:: If TPM is the issue:
:: 4. Clear TPM from admin PowerShell
Clear-Tpm
:: 5. Re-enable BitLocker
```

### TPM Troubleshooting

```cmd
:: Check TPM status
tpm.msc
:: Shows: TPM version, status, manufacturer

:: From cmd
wmic /namespace:\\root\cimv2\security\microsofttpm path win32_tpm get *

:: Clear TPM (CAUTION: may trigger BitLocker recovery)
:: tpm.msc → Actions → Clear TPM
```

---

## 29. Group Policy Troubleshooting

### Essential Commands

```cmd
:: Force Group Policy refresh
gpupdate /force

:: Generate HTML report of all applied GPOs
gpresult /h C:\temp\GPReport.html
:: Open in browser for full details

:: Text summary
gpresult /r

:: For a specific user
gpresult /user <username> /h C:\temp\GPReport.html

:: Verbose with denied GPOs
gpresult /z

:: Resultant Set of Policy (GUI)
rsop.msc
```

### Reading the GPResult Report

The HTML report shows:
- **Applied GPOs:** Which policies are active and in what order
- **Denied GPOs:** Which policies were denied and WHY (security filtering, WMI filter, disabled link)
- **Component Status:** Per-extension processing (Registry, Folder Redirection, etc.)

### Common GPO Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| Policy not applying | Security filtering excludes this user/computer | Check GPO → Delegation tab → Authenticated Users must have Read |
| Conflicting settings | Multiple GPOs set same value | Check precedence in `gpresult /z` |
| GPO slow to apply | WMI filter is complex or network is slow | Check event log: `Microsoft-Windows-GroupPolicy/Operational` |
| "Not Applied (Empty)" | GPO has no settings configured | Check the GPO in GPMC |

### Local Group Policy (Non-Domain)

Even non-domain machines have local policy:

```cmd
:: Edit local policy
gpedit.msc
:: Only available on Pro/Enterprise editions

:: On Home edition, use registry directly:
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\..." /v ValueName /t REG_DWORD /d 1
```

---



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



<!-- Windows 11 Troubleshooting Bible — Windows Performance Toolkit — WPR, WPA, ETW Tracing -->

## 34. Windows Performance Toolkit (WPR / WPA / ETW Tracing)

### What It Is

Windows Performance Toolkit is the **definitive performance analysis toolchain** — it captures Event Tracing for Windows (ETW) events at the kernel level and lets you visualize exactly what's happening during boot, app launch, disk I/O, CPU scheduling, and memory usage. This is what Microsoft's own engineers use internally.

**Components:**
- **WPR (Windows Performance Recorder)** — captures ETW traces into `.etl` files
- **WPA (Windows Performance Analyzer)** — opens `.etl` files for deep visual analysis
- **xperf** — legacy command-line ETW tool (still works, WPR is the modern replacement)

**Install:** Part of the Windows ADK (Assessment and Deployment Kit). Download from [Microsoft Learn](https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install). You only need the "Windows Performance Toolkit" checkbox during install.

### WPR — Recording Traces

```powershell
# Start a general performance trace (CPU + disk + memory + file I/O)
wpr -start GeneralProfile

# Start with specific profiles stacked
wpr -start CPU -start DiskIO -start FileIO

# Stop recording and save
wpr -stop C:\traces\my-trace.etl

# Cancel without saving
wpr -cancel

# Record a boot trace (captures everything from BIOS handoff to desktop)
wpr -start GeneralProfile -start Boot
wpr -boottrace -addboot GeneralProfile
# Reboot, then after login:
wpr -boottrace -stopboot C:\traces\boot-trace.etl
```

**Built-in profiles (use with `-start`):**
| Profile | What It Captures |
|---------|-----------------|
| `GeneralProfile` | CPU, disk, memory, handles — the go-to starting point |
| `CPU` | CPU sampling, context switches, DPC/ISR |
| `DiskIO` | Physical and logical disk operations |
| `FileIO` | File system operations (opens, reads, writes, closes) |
| `Heap` | Heap allocations and frees (heavy, use targeted) |
| `Minifilter` | File system minifilter driver activity |
| `Network` | TCP/IP, Winsock, and DNS activity |
| `Power` | CPU/GPU power state transitions, idle analysis |
| `GPU` | GPU scheduling, VRAM, render queue |
| `Audio` | Audio engine glitching analysis |
| `Video` | Media Foundation video pipeline |
| `Boot` | Full boot path analysis |

### WPA — Analyzing Traces

Open `.etl` files in WPA. Key analysis views:

**CPU analysis:**
- `Computation > CPU Usage (Sampled)` — shows which processes/threads consume CPU
- `Computation > CPU Usage (Precise)` — context switch analysis, shows what's waiting on what
- `Computation > DPC/ISR Duration` — driver interrupt analysis (high DPC = bad driver)

**Disk analysis:**
- `Storage > Disk Usage` — I/O size, duration, queue depth per process
- `Storage > File I/O` — individual file operations with timing

**Memory analysis:**
- `Memory > Virtual Memory Snapshots` — committed vs working set over time
- `Memory > Hard Faults` — page faults requiring disk reads (performance killer)

**Boot analysis (the killer feature):**
- `Boot Phases` graph shows exactly how long each boot phase takes
- `Main Path` shows the critical path — what's actually blocking boot completion
- Drill into `Winlogon`, `Explorer Init`, `Post Boot` phases

### ETW Tracing — Raw Power

```powershell
# List all running ETW sessions
logman query -ets

# List all registered providers
logman query providers

# Find providers for a specific component
logman query providers | findstr /i "storage"

# Start a custom trace session
logman create trace "MyTrace" -ow -o C:\traces\mytrace.etl -p "Microsoft-Windows-Kernel-Process" 0xFFFFFFFF 0xFF -ets

# Stop it
logman stop "MyTrace" -ets

# Capture netsh ETW trace for networking
netsh trace start capture=yes tracefile=C:\traces\net.etl
netsh trace stop

# Convert ETL to CSV for scripting
tracerpt C:\traces\my-trace.etl -o C:\traces\report.csv -of CSV
```

### Common Performance Investigation Workflows

**Slow boot:**
```
wpr -boottrace -addboot GeneralProfile → reboot → wpr -boottrace -stopboot boot.etl → open in WPA → Boot Phases → find bottleneck
```

**App hangs:**
```
wpr -start CPU -start Wait → reproduce hang → wpr -stop hang.etl → WPA → CPU Usage (Precise) → filter to process → check Wait Analysis
```

**Disk thrashing:**
```
wpr -start DiskIO -start FileIO → wpr -stop disk.etl → WPA → Disk Usage → sort by Total Time → identify heavy hitters
```

**Audio glitching:**
```
wpr -start Audio -start DPC → reproduce glitch → wpr -stop audio.etl → WPA → Audio Glitches graph → correlate with DPC/ISR spikes
```

### Tips

- **Always use GeneralProfile first** — it captures enough for 80% of investigations without the overhead of specialized profiles
- **Keep traces short** — 30-60 seconds of the problem behavior. Large traces are slow to analyze
- **Symbols:** For deep stack analysis, configure symbol paths: `set _NT_SYMBOL_PATH=srv*C:\symbols*https://msdl.microsoft.com/download/symbols`
- **Regions of Interest:** In WPA, use `Ctrl+Shift+R` to mark the exact timespan of the problem, then zoom in
- **Compare traces:** WPA can overlay two traces — capture a "good" and "bad" run, then diff them

---



<!-- Windows 11 Troubleshooting Bible — Task Scheduler Troubleshooting -->

## 35. Task Scheduler Troubleshooting

### Architecture

Task Scheduler runs as a Windows service (`Schedule`) with its own engine and persistence store. Tasks are defined as XML and stored in:

```
# File system location
C:\Windows\System32\Tasks\              # Root tasks
C:\Windows\System32\Tasks\Microsoft\    # Microsoft built-in tasks

# Registry location (task registration metadata)
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree\
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tasks\
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Boot\
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Logon\
HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Plain\
```

### Common Error Codes (Last Run Result)

| Exit Code | Hex | Meaning | Fix |
|-----------|-----|---------|-----|
| 0 | 0x0 | Success | — |
| 1 | 0x1 | Incorrect function or general error | Check the action command path and arguments |
| 2 | 0x2 | File not found | The executable/script path in the action is wrong |
| 10 | 0xA | Environment incorrect | Usually wrong user context or missing env vars |
| 267008 | 0x41300 | Task is currently running | Normal — means it hasn't finished yet |
| 267009 | 0x41301 | Task is queued | Waiting for a trigger condition to be met |
| 267010 | 0x41302 | Task has not yet run | Never been triggered since creation |
| 267011 | 0x41303 | No more instances of task | Task reached its repetition limit |
| 267012 | 0x41304 | Task terminated by user | Someone manually stopped it |
| 267014 | 0x41306 | Task was terminated because missed a start | Start-when-available wasn't set and the trigger time passed |
| 2147750671 | 0x8004130F | Credentials became corrupted or expired | Delete and recreate the task, re-enter credentials |
| 2147750687 | 0x8004131F | Unknown — general service error | Restart `Schedule` service, check Event Log |
| 2147942401 | 0x80070001 | Incorrect function | Exe returns non-zero — check the script itself |
| 2147942402 | 0x80070002 | System cannot find the file | Path is wrong or the user context can't see it |
| 2147942405 | 0x80070005 | Access denied | Task user lacks permissions to run the action or write output |
| 2147943645 | 0x800704DD | User not logged in | "Run only when user is logged on" selected but nobody's logged in |
| 2147946720 | 0x80071060 | Task XML parse error | Export task, validate XML, fix and re-import |

### Corrupted Task Cleanup

**Symptoms:** Task Scheduler won't open, shows "The task image is corrupt or has been tampered with," or tasks fail silently.

```powershell
# Step 1: Identify the corrupt task
# Check Event Viewer: Microsoft-Windows-TaskScheduler/Operational
# Event ID 413: "Task registered but with errors"
# Event ID 414: "Engine received message to start task but instance already running"

# Step 2: Find orphaned entries (registry has entry, file system doesn't, or vice versa)
# Export registry tree
reg export "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree" C:\temp\taskcache.reg

# Compare against file system
dir C:\Windows\System32\Tasks /s /b > C:\temp\taskfiles.txt

# Step 3: For a specific corrupt task, remove it manually
# Delete the file
del "C:\Windows\System32\Tasks\<TaskName>"

# Delete registry entries (find the GUID in Tree\<TaskName>, then remove from Tasks\{GUID})
# Use regedit to navigate to:
# HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree\<TaskName>
# Note the "Id" value (a GUID), then delete:
# - The Tree\<TaskName> key
# - The Tasks\{GUID} key
# - Any entry in Boot\, Logon\, or Plain\ with that GUID

# Step 4: Restart the service
net stop Schedule && net start Schedule
```

### PowerShell Task Management

```powershell
# List all tasks with their last result
Get-ScheduledTask | Get-ScheduledTaskInfo |
  Select-Object TaskName, LastRunTime, LastTaskResult, NextRunTime |
  Sort-Object LastRunTime -Descending

# Find all failed tasks (non-zero last result, excluding "running" codes)
Get-ScheduledTask | Get-ScheduledTaskInfo |
  Where-Object { $_.LastTaskResult -ne 0 -and $_.LastTaskResult -ne 267009 } |
  Select-Object TaskName, LastTaskResult, @{N='HexResult';E={'0x{0:X}' -f $_.LastTaskResult}}

# Export a task to XML for inspection
Export-ScheduledTask -TaskName "MyTask" | Out-File C:\temp\task.xml

# Re-register a task from XML
Register-ScheduledTask -TaskName "MyTask" -Xml (Get-Content C:\temp\task.xml -Raw)

# Create a task that runs a script at logon
$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-NoProfile -File C:\Scripts\startup.ps1"
$trigger = New-ScheduledTaskTrigger -AtLogon
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries
Register-ScheduledTask -TaskName "StartupScript" -Action $action -Trigger $trigger -Settings $settings -RunLevel Highest

# Run with SYSTEM account (no credential storage issues)
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest
Register-ScheduledTask -TaskName "SystemTask" -Action $action -Trigger $trigger -Principal $principal -Settings $settings

# Disable all tasks in a folder
Get-ScheduledTask -TaskPath "\Microsoft\Windows\Defrag\" | Disable-ScheduledTask
```

### Common Issues & Fixes

**Task runs but does nothing:** Usually a working directory problem. The task runs in `C:\Windows\System32` by default. Set `-WorkingDirectory` in `New-ScheduledTaskAction` or specify the full path to everything.

**Task runs as wrong user:** `Run whether user is logged on or not` stores credentials encrypted. Password changes = task breaks silently. Use SYSTEM account when possible, or use Group Managed Service Accounts (gMSA) in domain environments.

**Task won't run after Windows Update:** Some updates reset the Task Scheduler service configuration. Check `services.msc` → Task Scheduler → Startup type should be `Automatic`.

**Task Scheduler GUI is blank/won't open:** Delete `C:\Windows\System32\Tasks\Microsoft\Windows\UpdateOrchestrator\Schedule Scan` if it's corrupted (known Windows 11 bug). Or run `sfc /scannow` to repair.

---



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



<!-- Windows 11 Troubleshooting Bible — Intune/MDM Enterprise Device Management -->

## 37. Intune / MDM Enterprise Device Management Troubleshooting

### Device Registration Status

The first diagnostic for any Intune/MDM issue:

```powershell
# Check join status — the single most important command
dsregcmd /status

# Key fields to check in output:
# AzureAdJoined: YES/NO — is the device Azure AD joined?
# DomainJoined: YES/NO — is it hybrid (on-prem AD + Azure AD)?
# WorkplaceJoined: YES/NO — is it BYOD workplace joined?
# DeviceId: {GUID} — unique device ID in Azure AD
# TenantId: {GUID} — which tenant it belongs to
# KeyProvider: Microsoft Software Key Storage Provider (expected)
# TpmProtected: YES — TPM is securing the device keys
# DeviceAuthStatus: SUCCESS — device can authenticate to Azure AD

# Force re-registration if status is bad
dsregcmd /leave
dsregcmd /join

# Debug token acquisition
dsregcmd /status /debug
```

### MDM Enrollment Diagnostics

```powershell
# Check current MDM enrollment
Get-ChildItem "HKLM:\SOFTWARE\Microsoft\Enrollments" |
  ForEach-Object { Get-ItemProperty $_.PSPath } |
  Select-Object EnrollmentType, ProviderID, UPN

# MDM Diagnostics Report (generates comprehensive HTML report)
mdmdiagnosticstool.exe -area DeviceEnrollment -cab C:\temp\mdmdiag.cab

# Collect ALL MDM diagnostics
mdmdiagnosticstool.exe -area DeviceEnrollment;DeviceProvisioning;Autopilot -cab C:\temp\fulldiag.cab

# Check enrollment scheduled task
Get-ScheduledTask -TaskPath "\Microsoft\Windows\EnterpriseMgmt\*"

# Event logs for enrollment
Get-WinEvent -LogName "Microsoft-Windows-DeviceManagement-Enterprise-Diagnostics-Provider/Admin" -MaxEvents 20 |
  Format-Table TimeCreated, Id, Message -Wrap
```

### Common Enrollment Failures

| Scenario | Symptom | Fix |
|----------|---------|-----|
| `0x80180001` — server unreachable | Enrollment times out | Check DNS for `enterpriseenrollment.<domain>.com` CNAME → `enterpriseenrollment-s.manage.microsoft.com` |
| `0x80180014` — device already enrolled | Re-enrollment blocked | Remove existing enrollment: Settings → Accounts → Access work or school → Disconnect. Then re-enroll. |
| `0x80180018` — device cap reached | Too many devices for this user | Increase device limit in Intune → Devices → Enrollment restrictions, or remove stale devices |
| `0x801c0003` — TPM error | Device key attestation failed | `tpm.msc` → check TPM status. Try `Clear TPM` (Settings → Privacy & Security → Windows Security → Device Security → Security Processor → Troubleshoot). Reboot and re-enroll. |
| Hybrid Join stuck | Device shows `AzureAdJoined: NO` despite GPO | Check `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\CDJ\AAD` for errors. Verify Azure AD Connect sync includes the device OU. Check `dsregcmd /status` → Diagnostic Data section for the exact failure. |

### GPO vs. MDM Policy Conflicts

When a device is both domain-joined and Intune-managed:

```
Precedence order (highest wins):
1. MDM policy (Intune)
2. Group Policy (on-prem AD)
3. Local policy

Exception: "MDM Wins Over GP" setting in:
HKLM\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM
"ControlPolicyConflict" → "MDMWinsOverGP" = 1
```

**Diagnosing conflicts:**
```powershell
# See all applied MDM policies
Get-ChildItem "HKLM:\SOFTWARE\Microsoft\PolicyManager\Current\Device" -Recurse |
  Get-ItemProperty | Where-Object { $_.PSChildName -notmatch '^\(default\)$' } |
  Select-Object PSPath, * -ExcludeProperty PS*

# Compare with Group Policy
gpresult /h C:\temp\gpreport.html
# Open gpreport.html and compare with Intune policy report from:
# Intune → Devices → [device] → Device configuration → per-setting status

# Check for conflicting settings
# Common conflicts: BitLocker, Windows Update, Defender, Firewall
# Look in Event Viewer: Applications and Services Logs → Microsoft → Windows → DeviceManagement-Enterprise-Diagnostics-Provider
```

### Autopilot Troubleshooting

```powershell
# Check Autopilot profile assignment
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Provisioning\Diagnostics\Autopilot"

# View Autopilot event logs
Get-WinEvent -LogName "Microsoft-Windows-Provisioning-Diagnostics-Provider/AutoPilot" -MaxEvents 50

# Collect Autopilot diagnostics (during OOBE, press Shift+F10 for cmd)
mdmdiagnosticstool.exe -area Autopilot -cab C:\temp\autopilot.cab

# Check hardware hash registration
# The hash must be uploaded to Intune for Autopilot to work
# Verify: Intune → Devices → Enrollment → Windows Autopilot devices
# If missing, extract hash:
Install-Script -Name Get-WindowsAutoPilotInfo -Force
Get-WindowsAutoPilotInfo -OutputFile C:\temp\autopilot.csv
```

---



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



<!-- Windows 11 Troubleshooting Bible — Keyboard Shortcuts, Glossary, Recommended Reading, Sources -->

## Appendix A: Essential Keyboard Shortcuts for Troubleshooting

| Shortcut | Action |
|----------|--------|
| `Win + X` | Power user menu (Device Manager, Disk Management, Terminal, etc.) |
| `Win + I` | Settings |
| `Win + R` | Run dialog |
| `Win + Ctrl + Shift + B` | Reset graphics driver |
| `Ctrl + Shift + Esc` | Task Manager |
| `Win + Pause` | System info (About page) |
| `Win + V` | Clipboard history |
| `Win + .` | Emoji/symbols picker |
| `Alt + F4` | Close current window |
| `Ctrl + Alt + Del` | Security options (works even when system is hung) |

---

## Appendix B: Glossary

| Term | Meaning |
|------|---------|
| **Vmmem** | Synthetic process representing WSL2/Hyper-V VM memory usage |
| **WSL** | Windows Subsystem for Linux — run Linux binaries natively |
| **WSL2** | WSL version 2 — uses a real Linux kernel in a lightweight VM |
| **ConHost** | Legacy Windows console host (the old black terminal window) |
| **Windows Terminal** | Modern terminal app (tabs, GPU rendering, multiple profiles) |
| **PSReadLine** | PowerShell module that handles keyboard input and line editing |
| **DISM** | Deployment Image Servicing and Management — repairs Windows image |
| **SFC** | System File Checker — repairs individual system files |
| **CBS** | Component Based Servicing — Windows' internal component management |
| **WinRE** | Windows Recovery Environment — boot-time repair tools |
| **WinSxS** | Windows Side-by-Side — component store where all system files live |
| **Dev Drive** | ReFS volume optimized for developer workloads (Win11 23H2+) |
| **MSIX/AppX** | Modern Windows app packaging formats (Microsoft Store apps) |
| **MSI** | Microsoft Installer — traditional Windows installer format |
| **winget** | Windows Package Manager CLI (like apt/brew for Windows) |
| **Hyper-V** | Windows hypervisor — required for WSL2 and Docker |
| **VT-x / AMD-V** | CPU virtualization extensions (must be enabled in BIOS) |
| **BOM** | Byte Order Mark — invisible character at start of text files that can break config parsing |
| **TPM** | Trusted Platform Module — hardware security chip for BitLocker, Secure Boot |
| **NLA** | Network Level Authentication — RDP security requiring pre-authentication |
| **GPO** | Group Policy Object — centralized configuration management |
| **NTUSER.DAT** | Per-user registry hive file storing user settings |
| **SID** | Security Identifier — unique ID for every user/group account |
| **TRIM** | SSD command that tells the drive which blocks are no longer in use |
| **SMART** | Self-Monitoring, Analysis, and Reporting Technology — drive health monitoring |
| **DWM** | Desktop Window Manager — Windows compositor for display rendering |
| **BITS** | Background Intelligent Transfer Service — downloads for Windows Update |
| **DO** | Delivery Optimization — peer-to-peer update download system |
| **CredSSP** | Credential Security Support Provider — RDP authentication protocol |
| **pnputil** | Plug and Play utility — manage driver packages from command line |

---

## Appendix C: Recommended Reading

These are the definitive references for deep Windows troubleshooting. The document you're reading is a condensed operational reference — these books are the full encyclopedias.

**1. "Troubleshooting and Supporting Windows 11" — Mike Halsey (Apress, 2022)**
- 17 chapters covering everything from first principles to advanced debugging
- Available on [SpringerLink](https://link.springer.com/book/10.1007/978-1-4842-8728-6) and Amazon
- The closest thing to a complete handbook for IT support on Windows 11

**2. "Windows Internals, 7th Edition" — Russinovich, Yosifovich, Ionescu, Solomon**
- How Windows actually works under the hood — kernel, processes, memory, I/O, security
- [Part 1](https://www.microsoftpressstore.com/store/windows-internals-part-1-system-architecture-processes-9780735684188) and [Part 2](https://www.microsoftpressstore.com/store/windows-internals-part-2-9780135462331)
- THE bible for understanding WHY things break

**3. "Troubleshooting with the Windows Sysinternals Tools, 2nd Edition" — Russinovich & Margosis**
- 65+ tools, 45 real-world debugging examples
- [O'Reilly](https://www.oreilly.com/library/view/troubleshooting-with-the/9780133986549/) / [Amazon](https://www.amazon.com/Troubleshooting-Windows-Sysinternals-Tools-2nd/dp/0735684448)

**4. [Microsoft Learn — Windows Client Troubleshooting Hub](https://learn.microsoft.com/en-us/troubleshoot/windows-client/welcome-windows-client)**
- Official, constantly updated by the Windows product team
- Categories: Active Directory, Group Policy, Networking, Storage, Virtualization, App Management, Performance, Security, Shell Experience

**5. [Microsoft Learn — Windows 11 Known Issues](https://learn.microsoft.com/en-us/windows/release-health/)**
- Real-time known issues for each Windows 11 build

---

*Compiled from 50+ sources including Microsoft Learn documentation, Sysinternals guides, Windows community forums, and real-world IT troubleshooting experience. This document is a living reference — for the latest fixes, always cross-reference with Microsoft's official documentation.*

**Sources:**
- [Microsoft Learn: about_Profiles](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_profiles)
- [Microsoft Learn: SFC/DISM](https://support.microsoft.com/en-us/topic/use-the-system-file-checker-tool-to-repair-missing-or-corrupted-system-files-79aa86cb-ca52-166a-92a3-966e85d4094e)
- [Microsoft Learn: BitLocker Troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/windows-client/windows-security/bitlocker-issues-troubleshooting)
- [Microsoft Learn: Group Policy Troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/windows-server/group-policy/applying-group-policy-troubleshooting-guidance)
- [Microsoft Learn: Fix Bluetooth](https://support.microsoft.com/en-us/windows/fix-bluetooth-problems-in-windows-723e092f-03fa-858b-5c80-131ec3fba75c)
- [Microsoft Learn: Fix Printer Problems](https://support.microsoft.com/en-us/windows/fix-printer-connection-and-printing-problems-in-windows-fb830bff-7702-6349-33cd-9443fe987f73)
- [Microsoft Learn: Fix Corrupted User Profile](https://support.microsoft.com/en-us/windows/fix-a-corrupted-user-profile-in-windows-1cf41c18-7ce3-12f9-8e1d-95896661c5c9)
- [Microsoft Learn: WinSxS Cleanup](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/clean-up-the-winsxs-folder)
- [Microsoft Learn: chkdsk](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk)
- [Microsoft Support: Screen Flickering](https://support.microsoft.com/en-us/windows/troubleshoot-screen-flickering-in-windows-47d5b0a7-89ea-1321-ec47-dc262675fc7b)
- [Microsoft Learn: Sysinternals Suite](https://learn.microsoft.com/en-us/sysinternals/)
- [WSL Memory Limiting — Aleksandr Hovhannisyan](https://www.aleksandrhovhannisyan.com/blog/limiting-memory-usage-in-wsl-2/)
- [Vmmem Fix — Koskila.net](https://www.koskila.net/how-to-solve-vmmem-consuming-ungodly-amounts-of-ram-when-running-docker-on-wsl/)
- [Windows Central: Device Manager Troubleshooting](https://www.windowscentral.com/software-apps/windows-11/how-to-fix-device-manager-yellow-mark-for-drivers-on-windows-11)
- [Microsoft Learn: Windows Performance Toolkit](https://learn.microsoft.com/en-us/windows-hardware/test/wpt/)
- [Microsoft Learn: DISM Image Management](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/dism-image-management-command-line-options-s14)
- [Microsoft Learn: Windows Update Error Reference](https://learn.microsoft.com/en-us/windows/deployment/update/windows-update-errors)
- [Microsoft Learn: Intune Enrollment Troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/mem/intune/device-enrollment/troubleshoot-windows-enrollment-errors)
- [Microsoft Learn: Task Scheduler](https://learn.microsoft.com/en-us/windows/win32/taskschd/task-scheduler-start-page)
- [Microsoft Learn: CBS Log Analysis](https://learn.microsoft.com/en-us/troubleshoot/windows-client/deployment/cbs-log-file-overview)
- [TSS Toolset Download](https://aka.ms/getTSS)
- GitHub: PowerShell/PSReadLine, oh-my-posh, docker/for-win, microsoft/WSL issue trackers
- ElevenForum, WindowsForum, BleepingComputer community threads


