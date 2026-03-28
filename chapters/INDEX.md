# Windows 11 Troubleshooting Bible — Chapter Index

**Usage:** Load this file first. Based on the user's problem, load ONLY the relevant chapter(s). Do NOT load all chapters at once.

---

## Problem → Chapter Routing

| User's Problem | Load These Chapters |
|---|---|
| PowerShell frozen, can't type, profile broken, PSReadLine, PS update broke things | `Ch01-PowerShell.md` |
| Vmmem high RAM, WSL2 issues, Docker install stalls/fails, Hyper-V conflicts | `Ch02-WSL2-Docker-HyperV.md` |
| App won't install, MSIX/MSI errors, Store download stuck, winget issues | `Ch03-App-Install-Winget.md` |
| SFC fails, DISM errors, component store corrupt, service won't start/stop | `Ch04-System-Repair-Services.md` + `Ch18-CBS-DISM-TSS.md` |
| No internet, DNS issues, Wi-Fi drops, VPN problems, slow network, firewall | `Ch05-Networking.md` |
| Blue screen (BSOD), crash dumps, need Process Monitor/Explorer, Autoruns | `Ch06-BSOD-Sysinternals-EventViewer.md` |
| PATH broken, environment variable issues, registry corruption | `Ch07-Environment-Registry.md` |
| Need Safe Mode, WinRE won't boot, reset PC, in-place upgrade, fresh install | `Ch08-Recovery-Reset.md` |
| Quick command lookup, diagnostic one-liners | `Ch09-Quick-Reference.md` |
| Hardware not detected, driver issues, audio problems, Bluetooth drops, display/GPU glitches, screen flicker | `Ch10-Hardware-Audio-Display.md` |
| User profile corrupt, can't log in, BitLocker recovery, Group Policy not applying | `Ch11-UserProfiles-BitLocker-GPO.md` |
| Printer not working, disk errors, chkdsk, storage full, RDP can't connect | `Ch12-Print-Disk-RDP.md` |
| Windows Update stuck, update fails with error code | `Ch13-Windows-Update.md` + `Ch16-Update-Error-Codes.md` |
| Slow boot, slow app launch, performance analysis, need ETW trace | `Ch14-Performance-Tracing.md` |
| Scheduled task fails, Task Scheduler won't open, task error codes | `Ch15-Task-Scheduler.md` |
| Windows Update error code lookup (0x80070005, 0x800F0831, etc.) | `Ch16-Update-Error-Codes.md` |
| Intune enrollment fails, MDM policy conflicts, Autopilot, dsregcmd | `Ch17-Intune-MDM.md` |
| CBS.log analysis, DISM full reference, TSS diagnostic collection, pending.xml | `Ch18-CBS-DISM-TSS.md` |
| Keyboard shortcuts, glossary, book recommendations | `Appendices.md` |

## Chapter Manifest

| File | Lines | Sections | Topics |
|---|---|---|---|
| `Ch01-PowerShell.md` | 546 | 1–5 | Profile system, frozen shell fix, PSReadLine, PS5.1 vs PS7 conflicts, PS debugging/forensics |
| `Ch02-WSL2-Docker-HyperV.md` | 362 | 6–8 | Vmmem RAM fix, .wslconfig, WSL networking, Docker cleanup/reinstall, Hyper-V toggle |
| `Ch03-App-Install-Winget.md` | 125 | 9–10 | MSIX/MSI/EXE install failures, Store reset, winget troubleshooting |
| `Ch04-System-Repair-Services.md` | 172 | 11–12 | SFC/DISM repair chain, component store, service debugging |
| `Ch05-Networking.md` | 92 | 13 | TCP/IP reset, DNS, Wi-Fi, firewall, Winsock, proxy |
| `Ch06-BSOD-Sysinternals-EventViewer.md` | 209 | 14–16 | Crash dump analysis, WinDbg, Process Monitor, Autoruns, Event IDs |
| `Ch07-Environment-Registry.md` | 126 | 17–18 | PATH repair, env var management, registry backup/restore |
| `Ch08-Recovery-Reset.md` | 238 | 19–21 | Safe Mode entry, WinRE, developer features, in-place upgrade, clean install, YOUR SPECIFIC FIX |
| `Ch09-Quick-Reference.md` | 63 | 22 | Diagnostic command cheat sheets |
| `Ch10-Hardware-Audio-Display.md` | 243 | 23–26 | Device Manager codes, driver management, Realtek audio, Bluetooth profiles, GPU reset, DPI |
| `Ch11-UserProfiles-BitLocker-GPO.md` | 187 | 27–29 | Profile SID repair, NTUSER.DAT, BitLocker manage-bde, TPM, GPO precedence |
| `Ch12-Print-Disk-RDP.md` | 231 | 30–32 | Spooler reset, driver isolation, chkdsk SSD vs HDD, SMART, TRIM, RDP/NLA/CredSSP |
| `Ch13-Windows-Update.md` | 80 | 33 | Update pipeline internals, Delivery Optimization, manual update install |
| `Ch14-Performance-Tracing.md` | 131 | 34 | WPR profiles, WPA analysis views, ETW tracing, boot/hang/disk/audio workflows |
| `Ch15-Task-Scheduler.md` | 119 | 35 | Error code table, corrupted task cleanup, PowerShell task management |
| `Ch16-Update-Error-Codes.md` | 127 | 36 | 17 error codes with fixes, nuclear WU reset procedure, manual MSU install |
| `Ch17-Intune-MDM.md` | 114 | 37 | dsregcmd, enrollment diagnostics, GPO vs MDM, Autopilot |
| `Ch18-CBS-DISM-TSS.md` | 222 | 38 | TSS toolset, CBS.log patterns, DISM full parameter reference, pending.xml |
| `Appendices.md` | 107 | A–C | Keyboard shortcuts, glossary, recommended books, all sources |

**Total: 3,546 lines across 19 files**
