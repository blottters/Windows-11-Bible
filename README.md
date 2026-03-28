# Windows 11 Troubleshooting Bible

A comprehensive troubleshooting reference built for **Claude Cowork** — Anthropic's desktop tool for non-developers to automate file and task management. Designed as a drop-in skill that turns Claude into a fully capable Windows 11 IT help desk technician, powered by Windows-MCP for live diagnosis.

## What's Inside

**3,500+ lines** across **38 sections** covering every major Windows 11 troubleshooting domain:

| Chapter | Topics |
|---------|--------|
| Ch01 | PowerShell fundamentals, profiles, remoting, JEA |
| Ch02 | WSL2, Docker Desktop, Hyper-V |
| Ch03 | App installation, Winget, MSIX, Store repair |
| Ch04 | System repair (SFC, DISM), services management |
| Ch05 | Networking (Wi-Fi, DNS, VPN, firewall) |
| Ch06 | BSOD analysis, Sysinternals, Event Viewer |
| Ch07 | Environment variables, registry editing |
| Ch08 | Recovery, system restore, reset |
| Ch09 | Quick reference commands |
| Ch10 | Hardware, audio, display troubleshooting |
| Ch11 | User profiles, BitLocker, Group Policy |
| Ch12 | Printing, disk management, RDP |
| Ch13 | Windows Update troubleshooting |
| Ch14 | Performance tracing (WPR/WPA) |
| Ch15 | Task Scheduler |
| Ch16 | Windows Update error code reference |
| Ch17 | Intune/MDM enrollment and policy |
| Ch18 | CBS logs, DISM internals, TSS diagnostic toolkit |


## How It Works with Cowork

This repo is designed around Claude Cowork's skill system and MCP (Model Context Protocol) architecture:

- **Skill auto-triggers** when you describe any Windows problem in a Cowork session
- **Windows-MCP integration** lets Claude run PowerShell, inspect processes, read the registry, and capture screenshots directly on your machine — no copy-pasting commands
- **Chapter-based lazy loading** keeps context window efficient — Claude reads INDEX.md first, then loads only the 1-3 chapters relevant to your problem
- **Scheduled health checks** run automatically every 4 hours via Cowork's scheduled tasks, logging results and sending Windows desktop notifications
- **Desktop Commander** handles file operations when Windows-MCP needs support

## Repo Structure

```
├── README.md                              # You are here
├── Windows-11-Troubleshooting-Bible.md    # Full monolith (all 38 sections)
├── SYSTEM-PROMPT.md                       # RISEN+ReAct prompt for Claude
├── HEALTH-CHECK.md                        # Automated 4-hour health check setup
├── chapters/                              # Individual chapter files
│   ├── INDEX.md                           # Problem → Chapter routing table
│   ├── Ch01-PowerShell.md ... Ch18        # 18 focused chapters
│   └── Appendices.md                      # Shortcuts, glossary, sources
└── skill/                                 # Cowork skill (drop-in ready)
    ├── SKILL.md                           # Skill definition + diagnostic protocol
    └── references/                        # Chapter files loaded on demand
        ├── INDEX.md
        ├── Ch01 ... Ch18
        └── Appendices.md
```


## Setup

### Cowork Skill (recommended)

The skill is already installed if you cloned this from the original Cowork session. To install manually:

1. Copy the `skill/` folder contents into your Cowork skills directory:
   ```
   %USERPROFILE%\.claude\skills\windows-it-tech\
   ```
2. The folder should contain `SKILL.md` and a `references/` subdirectory with all chapter files
3. Claude will automatically activate the skill when you describe any Windows problem

### Required MCP: Windows-MCP

For live diagnosis (Claude runs commands directly instead of telling you what to type), you need the [Windows-MCP](https://github.com/nicholasrubright/windows-mcp) server connected to your Cowork session. This provides:

- `mcp__Windows-MCP__PowerShell` — run PowerShell commands
- `mcp__Windows-MCP__Process` — inspect running processes
- `mcp__Windows-MCP__Registry` — read/write registry keys
- `mcp__Windows-MCP__FileSystem` — read/write/check files
- `mcp__Windows-MCP__Screenshot` — capture screen state
- `mcp__Windows-MCP__Notification` — send desktop notifications
- Plus: App, Clipboard, Shortcut, Snapshot, Click, Type, Scrape, Wait

### Optional: Desktop Commander MCP

[Desktop Commander](https://github.com/nicholasrubright/desktop-commander) adds process management, file search, and directory operations that complement Windows-MCP. Used by the health check system for logging.

### Scheduled Health Check

See `HEALTH-CHECK.md` for setting up the 4-hour recurring health check via Cowork's `mcp__scheduled-tasks__create_scheduled_task`. Monitors RAM, disk, updates, services, and event log errors — logs to `%USERPROFILE%\Documents\Claude\Health-Logs\`.


## Alternative Usage

### System Prompt (Claude.ai or API)

If you're not using Cowork, paste the contents of `SYSTEM-PROMPT.md` as a system prompt and attach the chapter files as project knowledge. The RISEN+ReAct framework still works — you just won't have live MCP diagnosis.

### Standalone Reference

Read `Windows-11-Troubleshooting-Bible.md` directly or browse individual chapters in `chapters/`. Works as a plain knowledge base even without Claude.

## All 38 Sections

1. PowerShell Profiles & Execution Policy
2. PowerShell Remoting & WinRM
3. PowerShell Module Management
4. PowerShell Error Handling & Debugging
5. Just Enough Administration (JEA)
6. WSL2 Installation & Configuration
7. Docker Desktop on Windows 11
8. Hyper-V Management
9. App Installation & Winget
10. MSIX & Store Repair
11. SFC & DISM System Repair
12. Windows Services Management
13. Startup Programs & Scheduled Tasks
14. Wi-Fi & Network Adapters
15. DNS & Name Resolution
16. VPN Troubleshooting
17. Windows Firewall
18. Network Performance & Diagnostics
19. BSOD Analysis & Minidumps
20. Sysinternals Suite
21. Event Viewer Deep Dive
22. Environment Variables & PATH
23. Registry Editing & Repair
24. System Restore & Recovery
25. Windows Reset & Reinstall
26. Keyboard Shortcuts & Quick Commands
27. Audio & Sound Troubleshooting
28. Display, GPU & Multi-Monitor
29. Bluetooth & Peripheral Devices
30. User Profiles & Account Repair
31. BitLocker Drive Encryption
32. Group Policy for Home & Pro
33. Printer Troubleshooting
34. Disk Management & Storage Spaces
35. Remote Desktop (RDP)
36. Performance Tracing (WPR/WPA)
37. Task Scheduler Deep Dive
38. Windows Update Error Code Reference

**Appendices:** Intune/MDM, CBS/DISM/TSS Toolkit, Keyboard Shortcuts, Glossary, Sources

## Built With

- Researched via Microsoft Docs MCP and hands-on Windows 11 troubleshooting
- Built in Claude Cowork with Windows-MCP + Desktop Commander for live testing
- Structured for chapter-based lazy loading (context window efficiency)
- RISEN+ReAct prompt engineering for systematic diagnosis

## License

MIT
