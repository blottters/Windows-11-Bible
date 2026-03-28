# Windows 11 Troubleshooting Bible

A comprehensive, Claude-optimized reference for diagnosing and fixing Windows 11 issues. Built as both a standalone knowledge base and an AI-powered IT help desk skill.

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


## Repo Structure

```
├── README.md                              # You are here
├── Windows-11-Troubleshooting-Bible.md    # Full monolith (all 38 sections)
├── SYSTEM-PROMPT.md                       # RISEN+ReAct prompt for Claude
├── HEALTH-CHECK.md                        # Automated 4-hour health check setup
├── chapters/                              # Individual chapter files
│   ├── INDEX.md                           # Problem → Chapter routing table
│   ├── Ch01-PowerShell.md
│   ├── ...
│   ├── Ch18-CBS-DISM-TSS.md
│   └── Appendices.md
└── skill/                                 # Claude Code skill (drop-in ready)
    ├── SKILL.md                           # Skill definition + diagnostic protocol
    └── references/                        # Chapter files loaded on demand
        ├── INDEX.md
        ├── Ch01-PowerShell.md
        └── ...
```

## How to Use

### As a Claude Skill (recommended)
Copy the `skill/` folder into your Claude Code skills directory:
```
~/.claude/skills/windows-it-tech/
```
Claude will automatically activate it when you describe any Windows problem.

### As a System Prompt
Paste the contents of `SYSTEM-PROMPT.md` into Claude's system prompt field. Uses RISEN+ReAct framework for structured diagnosis.

### As a Reference
Read `Windows-11-Troubleshooting-Bible.md` directly, or browse individual chapters in `chapters/`.

### Automated Health Monitoring
See `HEALTH-CHECK.md` for setting up a 4-hour recurring health check via Claude's scheduled tasks.


## Sections Overview

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
- Structured for Claude's context window efficiency (chapter-based lazy loading)
- RISEN+ReAct prompt engineering for systematic diagnosis

## License

MIT
