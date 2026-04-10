# Windows App Installer & winget Recovery Playbook

A comprehensive guide and repository of scripts for repairing corrupted Windows App Installer environments. This playbook addresses critical failures where winget, the Microsoft Store, and MSIX package handling become unresponsive or throw dependency errors.

### Problem Statement
When the Windows AppX database or the App Installer dependency chain becomes corrupted, standard repair tools like sfc /scannow often fail to detect the issue. This results in:

* winget failing to initialize or reporting missing runtimes.

* Microsoft Store stuck on "Starting download" or "Failed to install."

* Error codes such as 0x80073CF6 (Package could not be registered) or dependency chain breaks.

### The Recovery Solution
This repository documents a 5-step recovery process to force-reset the package ecosystem and restore functionality.

1. Clear Broken Registrations
First, we must remove the stale or corrupted package entries that block new installations.

```PowerShell
# Run as Administrator
Get-AppxPackage *AppInstaller* -AllUsers | Remove-AppxPackage -AllUsers -ErrorAction SilentlyContinue
Get-AppxProvisionedPackage -Online | Where-Object {$_.DisplayName -like "*AppInstaller*"} | Remove-AppxProvisionedPackage -Online
```
2. Deep Component Cleanup
Force a refresh of the Windows Component Store to ensure no "ghost" dependencies remain.

```PowerShell
DISM /Online /Cleanup-Image /StartComponentCleanup
DISM /Online /Cleanup-Image /RestoreHealth
```
3. Global AppX Re-registration
This restores the underlying frameworks (VCLibs, etc.) that the App Installer relies on.

```PowerShell
Get-AppxPackage -AllUsers | Foreach {Add-AppxPackage -DisableDevelopmentMode -Register "$($_.InstallLocation)\AppXManifest.xml"}
```
4. Manual Reinstallation
Download the latest MSIX bundle from the Official Release and install it manually:

```PowerShell
Add-AppxPackage .\Microsoft.DesktopAppInstaller_8wekyb3d8bbwe.msixbundle
```
### Root Cause Analysis
Through testing, it was determined that the failure point is rarely the winget.exe itself, but rather a break in the MSIX dependency chain. When a core framework package is partially updated or incorrectly unregistered, the entire App Installer service enters a "broken" state that prevents it from updating itself through the Store.

### Lessons Learned
DISM isn't a silver bullet: It manages the image health, but not necessarily the live AppX registration database.

Sequence Matters: You must unprovision the package before attempting a re-registration of the manifest.

Manual Intervention: In cases of deep corruption, the Microsoft Store cannot "self-heal" the App Installer.

### Repository Contents
`playbook.md` - The full technical breakdown of the incident.

`scripts/` - (Optional) PowerShell scripts to automate the recovery steps.

`logs/` - Templates for analyzing DISM and AppX deployment logs.

### Author
Shawn - IT & Systems Specialist

### License
This project is licensed under the MIT License - see the LICENSE file for details.

### Pro-Tip
If you are still seeing errors after Step 5, check the AppX Deployment Server 
```code
logs:C:\Windows\Logs\AppXDeploymentServer\
```
