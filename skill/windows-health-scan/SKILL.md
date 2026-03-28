---
name: windows-health-scan
description: >
  Use when the user says "health check", "health scan", "system scan", "check my system",
  "how's my PC", "system health", "run diagnostics", "quick scan", "deep scan", or any
  request to assess the overall state of their Windows machine.
argument-hint: "[quick|standard|deep]"
allowed-tools: mcp__Windows-MCP__PowerShell, mcp__Windows-MCP__Notification
---

# Windows 11 Health Scanner

Run a tiered health scan on the user's Windows 11 machine using Windows-MCP tools.

## Tier Selection

If `$ARGUMENTS` specifies a tier (quick, standard, deep), use that tier directly.
Otherwise, present the three tiers and ask the user to pick one:

**Tier 1 — Quick Scan** (~30 seconds, 6 checks)
Catches immediate threats: runaway processes, low disk, dead services, stale reboots, Defender gaps.

**Tier 2 — Standard Scan** (~2 minutes, 12 checks)
Everything in Quick, plus: Windows Update, event log errors, WSL/Docker, startup bloat, bad drivers, disk health.

**Tier 3 — Deep Scan** (~5 minutes, 20 checks)
Full system audit. Everything in Standard, plus: temp bloat, firewall, failed tasks, network, pagefile, reliability score, expired certs, GPU crashes.

Use AskUserQuestion with these three options. If invoked by a scheduled task, default to Tier 2 (Standard).

## Execution

Run all checks for the selected tier sequentially via `mcp__Windows-MCP__PowerShell`. Read each output before proceeding to the next check. Flag anything abnormal.

Load `references/health-checks.md` for the exact PowerShell commands and flag thresholds for all 20 checks.

## After All Checks

1. **Build a report** with sections for each check, flagged issues, and an overall status
2. **Log the report** — append to `C:\Users\gavin\Documents\Claude\Health-Logs\health-check.log` and overwrite `latest.txt`
3. **Send a notification** via `mcp__Windows-MCP__Notification`:
   - All clear: "Health Scan — All systems nominal"
   - Warnings: "Health Scan — [count] warning(s): [summary]"
   - Critical: "Health Scan — [count] critical issue(s): [summary]"
4. **Return the full report** as your response

If any individual check fails (permission denied, command error), note it and move on — don't let one failure block the scan.

## Report Format

```
========================================
HEALTH SCAN — [TIER NAME] — [timestamp]
========================================

[CHECK NAME]
<output>
[STATUS: OK / WARNING / CRITICAL]
<flag reason if not OK>

... repeat for each check ...

========================================
SUMMARY
========================================
Tier: [Quick/Standard/Deep]
Checks run: [count]
Issues found: [count]
  Critical: [list]
  Warning: [list]
Status: [ALL CLEAR / WARNINGS / CRITICAL]
========================================
```

## Logging

Ensure the log directory exists before writing:
```powershell
New-Item -ItemType Directory -Path "C:\Users\gavin\Documents\Claude\Health-Logs" -Force | Out-Null
```

Append to rolling log:
```powershell
Add-Content -Path "C:\Users\gavin\Documents\Claude\Health-Logs\health-check.log" -Value $report
```

Overwrite latest:
```powershell
Set-Content -Path "C:\Users\gavin\Documents\Claude\Health-Logs\latest.txt" -Value $report
```
