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

