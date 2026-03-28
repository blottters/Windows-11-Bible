<!-- Windows 11 Troubleshooting Bible — Hardware/Drivers, Audio, Bluetooth, Display/GPU -->

## 23. Hardware, Drivers & Device Manager

### Device Manager Error Codes

Open Device Manager: `devmgmt.msc`

Devices with problems show a yellow triangle (⚠️). Double-click the device → General tab to see the error code.

| Code | Meaning | Fix |
|------|---------|-----|
| 1 | Device not configured | Install/update driver |
| 3 | Driver corrupted | Uninstall device, reboot (Windows reinstalls) |
| 10 | Device cannot start | Hardware failure or driver issue; try rolling back driver |
| 12 | Not enough resources | IRQ/memory conflict; disable conflicting device |
| 18 | Reinstall driver | Uninstall, reboot, reinstall |
| 19 | Registry problem | Uninstall device, reboot |
| 22 | Device disabled | Right-click → Enable device |
| 28 | Driver not installed | Install driver manually |
| 31 | Device not working properly | Update driver or check Windows Update |
| 39 | Driver corrupted or missing | Uninstall, reboot, reinstall |
| 43 | Device stopped (reported problem) | Often GPU overheating or hardware failure |
| 52 | Driver not signed | Disable driver signature enforcement (test mode) or find signed driver |

### Driver Management Commands

```cmd
:: List all drivers
driverquery /v

:: List drivers with sign status
driverquery /si

:: Export driver list
driverquery /v /fo csv > C:\temp\drivers.csv

:: Check for problematic drivers (unsigned)
sigverif
:: Opens File Signature Verification tool

:: Backup all installed drivers
mkdir C:\DriverBackup
dism /online /export-driver /destination:C:\DriverBackup

:: Install a driver from backup
pnputil /add-driver C:\DriverBackup\*.inf /subdirs /install
```

### Rollback a Driver

1. Device Manager → Right-click device → Properties → Driver tab → Roll Back Driver
2. If grayed out, Windows didn't keep the previous driver. You'll need to manually install the old version.

### Force Driver Installation

```cmd
:: Add a driver to the store
pnputil /add-driver driver.inf

:: Remove a driver from the store
pnputil /delete-driver oem42.inf /force

:: Enumerate all OEM drivers
pnputil /enum-drivers
```

### USB Troubleshooting

```cmd
:: Reset USB controllers
:: Device Manager → Universal Serial Bus controllers → Uninstall ALL "USB Root Hub" entries → Reboot

:: Check USB power
:: Device Manager → USB Root Hub → Properties → Power tab → shows per-port power allocation

:: Fix USB selective suspend (causes devices to disconnect)
:: Control Panel → Hardware and Sound → Power Options → Change plan settings →
:: Change advanced power settings → USB settings → USB selective suspend → Disable
```

---

## 24. Audio Troubleshooting

### No Sound — Diagnostic Checklist

```cmd
:: 1. Check audio service
sc query AudioSrv
:: Should show: STATE = RUNNING

:: 2. If not running
net start AudioSrv
net start AudioEndpointBuilder

:: 3. Check audio devices
:: Settings → System → Sound → Output → make sure correct device is selected

:: 4. Check volume mixer
sndvol
:: Verify per-app volumes aren't muted
```

### Realtek Audio Issues (Most Common)

After Windows Update frequently breaks Realtek drivers:

1. **Uninstall and reinstall:**
   - Device Manager → Sound, video and game controllers → Realtek → Uninstall device → check "Delete the driver software" → Reboot
   - Windows will install a generic "High Definition Audio Device" driver
   - Then install the latest Realtek driver from your motherboard manufacturer's website (NOT from Realtek directly — they're often outdated)

2. **Microsoft UAA Bus Driver fix:**
   - Device Manager → System Devices → Microsoft UAA Bus Driver for High Definition Audio
   - Right-click → Disable → Right-click → Enable

3. **Disable audio enhancements:**
   - Right-click speaker icon → Sound settings → Output device → Properties → Disable audio enhancements

### Audio Services

| Service | Purpose |
|---------|---------|
| `AudioSrv` (Windows Audio) | Core audio playback and recording |
| `AudioEndpointBuilder` | Manages audio endpoints (speakers, headphones) |
| `RtkAudioUniversalService` | Realtek driver service |

```cmd
:: Restart all audio services
net stop AudioSrv & net stop AudioEndpointBuilder
net start AudioEndpointBuilder & net start AudioSrv
```

### Hidden Audio Diagnostic

```cmd
:: Run from cmd
msdt.exe /id AudioPlaybackDiagnostic
:: Opens the built-in audio troubleshooter with more options than Settings
```

---

## 25. Bluetooth Troubleshooting

### Bluetooth Disappeared

Three common causes: disabled adapter, crashed driver, or power management turned it off.

```cmd
:: Check Bluetooth service
sc query bthserv
:: Should be RUNNING

:: Start if stopped
net start bthserv

:: Check Bluetooth Support Service
sc query BluetoothUserService_*
```

### Fix Missing Bluetooth

1. **Check Device Manager** for "Bluetooth" category
   - If missing entirely: adapter is dead, disabled in BIOS, or driver is gone
   - If has yellow triangle: driver issue

2. **BIOS/UEFI check:** Some laptops have a hardware switch or BIOS setting for Bluetooth

3. **Airplane mode:** Make sure it's OFF (Settings → Network & internet → Airplane mode)

4. **Driver reinstall:**
   ```cmd
   :: Uninstall from Device Manager, then scan for hardware changes
   :: Or force Windows to rediscover:
   pnputil /scan-devices
   ```

5. **Power management causing disconnects:**
   - Device Manager → Bluetooth adapter → Properties → Power Management tab
   - UNCHECK "Allow the computer to turn off this device to save power"

### Bluetooth Audio Not Working

- Settings → System → Sound → Output → Select your Bluetooth device
- If listed as both "headset" and "headphones": Headphones = A2DP (high quality), Headset = HFP (low quality for calls)
- Remove device and re-pair if audio profile is stuck on HFP

---

## 26. Display & GPU Troubleshooting

### Graphics Stack Emergency Reset

```
Win + Ctrl + Shift + B
```

Screen will briefly flash black. This restarts the graphics driver without rebooting. Use when:
- Screen is frozen but mouse moves
- Display output is garbled
- Second monitor went black

### Black Screen After Update

1. Boot into Safe Mode (see Section 19)
2. Uninstall GPU driver:
   ```cmd
   :: From Safe Mode cmd
   pnputil /enum-drivers | findstr /i "display"
   pnputil /delete-driver oem##.inf /force
   ```
3. Reboot — Windows will use basic display driver
4. Install fresh GPU driver from NVIDIA/AMD/Intel website

### Multi-Monitor Issues

```cmd
:: Check connected displays
:: Settings → System → Display → shows all connected monitors

:: Reset display configuration
:: Delete per-display registry entries:
:: HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\DisplaySettings

:: If external monitor not detected:
:: Win+P → choose display mode (Duplicate, Extend, Second screen only)
```

### Display Scaling on Mixed-DPI Setups

When monitors have different DPI (e.g., 4K laptop + 1080p external):
- Settings → System → Display → select each monitor → Scale → set individually
- Some apps don't respect per-monitor DPI: Right-click app .exe → Properties → Compatibility → Change high DPI settings → Override → Application

### Screen Flickering

To determine if it's a driver or app issue:
1. Open Task Manager (Ctrl+Shift+Esc)
2. If Task Manager **also flickers** → display driver issue
3. If Task Manager **doesn't flicker** → incompatible app issue

---

