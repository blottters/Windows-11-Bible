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

