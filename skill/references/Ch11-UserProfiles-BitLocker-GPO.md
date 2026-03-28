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

