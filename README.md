# Windows Built-in Apps, Edge & WebView2 Complete Repair Guide

> **Use case:** Your PC is missing built-in Windows apps, Edge and WebView2 are stuck on old versions, and nothing seems to install or update correctly — especially after using a privacy/blocker tool like O&O ShutUp10, WPD, or privacy.sexy.

---

## Part 1 — Restoring Missing Built-in Windows Apps

### Step 1 — Re-register all installed apps
```powershell
Get-AppxPackage -AllUsers | Where-Object {$_.InstallLocation -ne $null -and $_.InstallLocation -ne ""} | Foreach {
    Add-AppxPackage -DisableDevelopmentMode -Register "$($_.InstallLocation)\AppXManifest.xml" -ErrorAction SilentlyContinue
}
```
**Expected:** A flood of errors is normal. Ignore:
- `Cannot find path 'C:\AppXManifest.xml'` — ghost entries, safe to ignore
- `0x80073D06` higher version already installed — safe to ignore
- `Microsoft.Wallet` end of life — safe to ignore

### Step 2 — Scan and repair system files
Run these **in order** (DISM must come before SFC):
```powershell
DISM /Online /Cleanup-Image /RestoreHealth
sfc /scannow
chkdsk C: /f /r
```
> `chkdsk` will ask to schedule on next reboot — type `Y` and restart.

---

## Part 2 — Installing AppX/MSIX Packages Manually

### Error: 0x80073CF9 — File locked by another process
```powershell
Stop-Service wuauserv -Force
Stop-Service msiserver -Force
Stop-Service AppXSvc -Force

Add-AppxPackage -Path "C:\Path\To\yourfile.MSIX"

Start-Service wuauserv
Start-Service msiserver
Start-Service AppXSvc
```

### Error: 0x80073D02 — App must be closed first (e.g. Microsoft Store)
```powershell
Get-Process | Where-Object {$_.Name -like "*Store*" -or $_.Name -like "*wsappx*"} | Stop-Process -Force
Stop-Service AppXSvc -Force
Add-AppxPackage -Path "C:\Path\To\yourfile.APPX" -ForceUpdateFromAnyVersion
Start-Service AppXSvc
```

### Error: 0x80073D06 — Higher version already installed
**Nothing to fix.** A newer version is already on your system. Skip this package.

---

## Part 3 — Fixing Edge & WebView2 Stuck on Old Version

This is the most complex part. Privacy tools block Edge/WebView2 installation through multiple mechanisms.

### Step 1 — Remove policy blocks (set by privacy tools)
```powershell
Remove-Item "HKLM:\SOFTWARE\Policies\Microsoft\EdgeUpdate" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "HKLM:\SOFTWARE\WOW6432Node\Policies\Microsoft\EdgeUpdate" -Recurse -Force -ErrorAction SilentlyContinue
```

### Step 2 — Fix permissions on EdgeUpdate folder
This is often the root cause — the folder exists but has wrong permissions:
```powershell
takeown /F "C:\Program Files (x86)\Microsoft\EdgeUpdate" /R /D Y
icacls "C:\Program Files (x86)\Microsoft\EdgeUpdate" /grant Administrators:F /T
icacls "C:\Program Files (x86)\Microsoft\EdgeUpdate" /grant SYSTEM:F /T
```

### Step 3 — Clear ghost WebView2 registry entries
Check for ghost entries (installed in registry but no files on disk):
```powershell
Get-ChildItem "HKLM:\SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\Clients" | Get-ItemProperty | Select PSChildName, pv
```
Remove ghost WebView2 entries (version 122 with no files):
```powershell
Remove-Item "HKLM:\SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\Clients\{56EB18F8-B008-4CBD-B6D2-8C97FE7E9062}" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "HKLM:\SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}" -Recurse -Force -ErrorAction SilentlyContinue
```
Also clear from Uninstall registry:
```powershell
Get-ChildItem "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall" | 
Get-ItemProperty | Where-Object {$_.DisplayName -like "*WebView2*"} | 
ForEach-Object { Remove-Item $_.PSPath -Recurse -Force }
```

### Step 4 — Install Edge using the Business installer
> **Important:** Use the **Edge Business** installer, NOT the Enterprise MSI (winget uses Enterprise MSI which causes issues). Download directly from:
> `https://www.microsoft.com/en-us/edge/business/download`

Run the installer normally (GUI). After installation:
- Open Edge → `edge://settings/help`
- It should show "Microsoft Edge is up to date" or start checking for updates
- "Managed by your organization" should NOT appear at the bottom

### Step 5 — Recreate EdgeUpdate Clients registry entries
After installing Edge, recreate the registry entries so EdgeUpdate knows what to manage:
```powershell
# Edge
$edgePath = "HKLM:\SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\Clients\{56EB18F8-B008-4CBD-B6D2-8C97FE7E9062}"
New-Item -Path $edgePath -Force
Set-ItemProperty -Path $edgePath -Name "pv" -Value "148.0.3967.70"
Set-ItemProperty -Path $edgePath -Name "ap" -Value "stable-arch_x64"
Set-ItemProperty -Path $edgePath -Name "brand" -Value "GGLS"
Set-ItemProperty -Path $edgePath -Name "name" -Value "Microsoft Edge"

# WebView2
$wv2Path = "HKLM:\SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}"
New-Item -Path $wv2Path -Force
Set-ItemProperty -Path $wv2Path -Name "pv" -Value "148.0.3967.70"
Set-ItemProperty -Path $wv2Path -Name "ap" -Value "stable-arch_x64"
Set-ItemProperty -Path $wv2Path -Name "brand" -Value "INBX"
Set-ItemProperty -Path $wv2Path -Name "name" -Value "Microsoft Edge WebView2 Runtime"
```

### Step 6 — Trigger EdgeUpdate to check and update
```powershell
Start-ScheduledTask -TaskName "MicrosoftEdgeUpdateTaskMachineUA{YOURGUID}"
Start-Sleep -Seconds 45
Get-ChildItem "HKLM:\SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\ClientState" | Get-ItemProperty | Select PSChildName, pv
```
> To find your GUID: `Get-ScheduledTask | Where-Object {$_.TaskName -like "*EdgeUpdateTaskMachineUA*"}`

---

## Part 4 — "Managed by Your Organization" Blocking Updates

If Edge shows "Managed by your organization" at the bottom of Settings, a privacy tool left behind policy registry keys.

### Check for leftover policies:
```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Edge" -ErrorAction SilentlyContinue
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\EdgeUpdate" -ErrorAction SilentlyContinue
```

### Remove all Edge policies (this fixes the "Managed by organization" message):
```powershell
Remove-Item "HKLM:\SOFTWARE\Policies\Microsoft\Edge" -Recurse -Force
Remove-Item "HKLM:\SOFTWARE\Policies\Microsoft\EdgeUpdate" -Recurse -Force -ErrorAction SilentlyContinue
```
> ⚠️ This removes all privacy tool settings for Edge. You'll need to re-apply any privacy preferences manually afterward.

---

## Part 5 — Verify Everything Is Working

```powershell
# Check Edge and WebView2 versions
winget list Microsoft.Edge
winget list Microsoft.EdgeWebView2Runtime

# Check EdgeUpdate registry
Get-ChildItem "HKLM:\SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\ClientState" | Get-ItemProperty | Select PSChildName, pv

# Confirm no policy blocks remain
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\EdgeUpdate" -ErrorAction SilentlyContinue
```

**Expected healthy output:**
- Edge: `148.x.x.x` (or latest)
- WebView2: same version as Edge
- No output from the policy check

---

## Quick Reference — Common Error Codes

| Error Code | Meaning | Fix |
|-----------|---------|-----|
| `0x80073CF9` | File locked by another process | Stop services, retry |
| `0x80073D02` | App is running, must be closed | Kill the process, retry |
| `0x80073D06` | Higher version already installed | Skip, nothing to do |
| `0x80073CFD` | App is end of life | Skip, can't install |
| `0x80040C01` | WebView2 install failed | Clear ghost registry entries |
| `0x8004081E` | "Hardware requirements" (misleading) | Remove Edge policy keys, use Business installer |
| `1603` | MSI fatal error | Fix EdgeUpdate folder permissions |

---

## Root Causes Summary

If you used a privacy tool (O&O ShutUp10, WPD, privacy.sexy, etc.) and Edge/WebView2 broke, here's what happened and how we fixed it:

1. **Policy blocks** — The tool set `InstallDefault=0` and `UpdateDefault=0` in `HKLM:\SOFTWARE\Policies\Microsoft\EdgeUpdate` → **Fixed by deleting the policy key**
2. **Ghost registry entries** — WebView2 version 122 existed in registry with no files on disk → **Fixed by deleting the Clients and Uninstall registry entries**
3. **Wrong folder permissions** — EdgeUpdate folder had permissions that blocked installers → **Fixed with `takeown` and `icacls`**
4. **Enterprise MSI vs Business installer** — winget uses Enterprise MSI which doesn't self-update properly → **Fixed by using the Edge Business installer instead**
5. **Missing Clients registry entries** — We deleted them during cleanup and EdgeUpdate stopped working → **Fixed by recreating them manually**
6. **"Managed by organization"** — Leftover Edge policy keys made Edge think it was enterprise-managed → **Fixed by deleting `HKLM:\SOFTWARE\Policies\Microsoft\Edge`**

---

*Guide written based on a real troubleshooting session on Windows 11 (Build 26200) — May 2026*
