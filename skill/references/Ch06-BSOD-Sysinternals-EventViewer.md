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

