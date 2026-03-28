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

