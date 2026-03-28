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

