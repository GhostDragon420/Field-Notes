# Field Fixes of a Migrated OS, So You Don't Have To!
### PowerShell · CMD · Bcdedit · DISM · SFC · Registry · Drivers
> Field Notes — Jon Merriman / SFE LLC
> Last Updated: 2026-05-10

---

## 📋 TABLE OF CONTENTS
1. [System Info & Diagnostics](#system-info)
2. [Registry Commands](#registry)
3. [Driver Management](#drivers)
4. [Boot Configuration (Bcdedit)](#bcdedit)
5. [System File & Image Repair (SFC / DISM)](#repair)
6. [Windows Update Control](#windows-update)
7. [Process & Service Management](#processes)
8. [Network Commands](#network)
9. [Disk & Storage](#disk)
10. [Security & Permissions](#security)
11. [Useful One-Liners](#oneliners)
12. [Common Registry Paths](#regpaths)
13. [Linux Networking](#linuxnet)
14. [Event Log Troubleshooting](#eventlog)

---

## 1. SYSTEM INFO & DIAGNOSTICS <a name="system-info"></a>

### PowerShell
```powershell
# Full Windows version info
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion" | Select-Object ProductName, CurrentBuild, DisplayVersion, ReleaseId

# Check Windows edition and build
winver

# System info dump
Get-ComputerInfo

# Quick OS version check
[System.Environment]::OSVersion

# Check if running as Administrator
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)

# Uptime
(Get-Date) - (gcim Win32_OperatingSystem).LastBootUpTime

# Hardware info
Get-CimInstance Win32_ComputerSystem
Get-CimInstance Win32_Processor
Get-CimInstance Win32_PhysicalMemory
```

### CMD
```cmd
:: System info
systeminfo

:: Windows version
ver

:: Environment variables
set
```

---

## 2. REGISTRY COMMANDS <a name="registry"></a>

### PowerShell
```powershell
# Read a registry value
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion" -Name "ProductName"

# Change a registry value (example: fix ProductName to Windows 11)
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion" -Name "ProductName" -Value "Windows 11 Home"

# Create a new registry key
New-Item -Path "HKLM:\SOFTWARE\MyKey"

# Create a new registry value
New-ItemProperty -Path "HKLM:\SOFTWARE\MyKey" -Name "MyValue" -Value "Data" -PropertyType String

# Delete a registry value
Remove-ItemProperty -Path "HKLM:\SOFTWARE\MyKey" -Name "MyValue"

# Delete a registry key
Remove-Item -Path "HKLM:\SOFTWARE\MyKey" -Recurse

# List all values in a key
Get-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion"

# Search registry for a value
Get-ChildItem -Path "HKLM:\SOFTWARE" -Recurse | Where-Object { $_.Name -like "*SearchTerm*" }
```

### Registry Hive Shortcuts
| Shorthand | Full Path |
|-----------|-----------|
| HKLM | HKEY_LOCAL_MACHINE |
| HKCU | HKEY_CURRENT_USER |
| HKCR | HKEY_CLASSES_ROOT |
| HKU | HKEY_USERS |
| HKCC | HKEY_CURRENT_CONFIG |

---

## 3. DRIVER MANAGEMENT <a name="drivers"></a>

### PowerShell
```powershell
# List all installed drivers
Get-WindowsDriver -Online -All

# List 3rd party drivers only
Get-WindowsDriver -Online -All | Where-Object { $_.OriginalFileName -notlike "*system32*" }

# Get driver info for a specific device
Get-PnpDevice | Where-Object { $_.FriendlyName -like "*YourDevice*" }

# List all PnP devices
Get-PnpDevice

# List problem devices (error state)
Get-PnpDevice | Where-Object { $_.Status -eq "Error" }

# Disable a device
Disable-PnpDevice -InstanceId "PCI\VEN_XXXX..."

# Enable a device
Enable-PnpDevice -InstanceId "PCI\VEN_XXXX..."

# Export all driver info to file
Get-WindowsDriver -Online -All | Export-Csv "C:\drivers-list.csv" -NoTypeInformation

# Install a driver from .inf file
pnputil /add-driver "C:\path\to\driver.inf" /install

# Remove a driver
pnputil /delete-driver oem##.inf /uninstall /force
```

### CMD
```cmd
:: List all drivers
driverquery

:: List drivers with details
driverquery /v

:: List drivers in CSV format
driverquery /fo csv > C:\drivers.csv

:: Add driver via PnPUtil
pnputil /add-driver driver.inf /install

:: List all OEM drivers
pnputil /enum-drivers
```

### Driver Signature Enforcement
```powershell
# Check current secure boot state
Confirm-SecureBootUEFI

# Check BitLocker status before touching secure boot
manage-bde -status

# Enable test signing mode (requires Secure Boot OFF)
bcdedit /set testsigning on

# Disable test signing mode
bcdedit /set testsigning off

# Check current test signing state
bcdedit /enum | findstr testsigning
```

---

## 4. BOOT CONFIGURATION (BCDedit) <a name="bcdedit"></a>

> ⚠️ These commands require Admin. Secure Boot must be OFF to modify most of these.

```cmd
:: View full boot config
bcdedit /enum all

:: View current boot entry only
bcdedit

:: Enable test signing (allows unsigned drivers)
bcdedit /set testsigning on

:: Disable test signing
bcdedit /set testsigning off

:: Enable debug mode
bcdedit /set debug on

:: Disable driver integrity checks (use carefully)
bcdedit /set nointegritychecks on

:: Re-enable driver integrity checks
bcdedit /set nointegritychecks off

:: Set timeout for boot menu (seconds)
bcdedit /timeout 10

:: Set default boot entry
bcdedit /default {identifier}

:: Add safe mode boot entry
bcdedit /copy {current} /d "Safe Mode Entry"
bcdedit /set {newguid} safeboot minimal

:: Delete a boot entry
bcdedit /delete {identifier}
```

---

## 5. SYSTEM FILE & IMAGE REPAIR (SFC / DISM) <a name="repair"></a>

### SFC — System File Checker
```cmd
:: Scan and repair system files
sfc /scannow

:: Scan only, no repair
sfc /verifyonly

:: Scan specific file
sfc /scanfile=C:\Windows\System32\filename.dll

:: Run offline (for when Windows won't boot properly)
sfc /scannow /offbootdir=C:\ /offwindir=C:\Windows
```

### DISM — Deployment Image Servicing
```cmd
:: Check image health
DISM /Online /Cleanup-Image /CheckHealth

:: Scan image health
DISM /Online /Cleanup-Image /ScanHealth

:: Repair image (downloads from Windows Update)
DISM /Online /Cleanup-Image /RestoreHealth

:: Repair using local source (ISO/WIM)
DISM /Online /Cleanup-Image /RestoreHealth /Source:D:\Sources\install.wim

:: Get OS info
DISM /Online /Get-CurrentEdition

:: List available editions
DISM /Online /Get-TargetEditions

:: Remove an installed update
DISM /Online /Remove-Package /PackageName:Package_for_KB######~####

:: Clean up component store (free space)
DISM /Online /Cleanup-Image /StartComponentCleanup
```

---

## 6. WINDOWS UPDATE CONTROL <a name="windows-update"></a>

### PowerShell
```powershell
# Check Windows Update service status
Get-Service wuauserv

# Stop Windows Update service
Stop-Service wuauserv -Force

# Start Windows Update service
Start-Service wuauserv

# Disable Windows Update service (prevents auto updates)
Set-Service wuauserv -StartupType Disabled

# Re-enable Windows Update service
Set-Service wuauserv -StartupType Manual

# List installed updates
Get-HotFix

# List updates sorted by date
Get-HotFix | Sort-Object InstalledOn -Descending

# Remove a specific update
wusa /uninstall /kb:XXXXXXX /quiet /norestart
```

### Pause Updates via Registry
```powershell
# Pause updates for 35 days
$date = (Get-Date).AddDays(35).ToString("yyyy-MM-dd") + "T00:00:00Z"
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\WindowsUpdate\UX\Settings" -Name "PauseUpdatesExpiryTime" -Value $date

# Disable driver updates via Windows Update
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Name "ExcludeWUDriversInQualityUpdate" -Value 1 -Type DWord
```

---

## 7. PROCESS & SERVICE MANAGEMENT <a name="processes"></a>

### PowerShell
```powershell
# List all running processes
Get-Process

# Find a specific process
Get-Process -Name "chrome"

# Kill a process by name
Stop-Process -Name "chrome" -Force

# Kill a process by ID
Stop-Process -Id 1234 -Force

# List all services
Get-Service

# Find a specific service
Get-Service -Name "wuauserv"

# Start a service
Start-Service -Name "ServiceName"

# Stop a service
Stop-Service -Name "ServiceName" -Force

# Restart a service
Restart-Service -Name "ServiceName"

# Set service startup type
Set-Service -Name "ServiceName" -StartupType Disabled  # or Manual, Automatic

# List services set to auto-start
Get-Service | Where-Object { $_.StartType -eq "Automatic" }
```

---

## 8. NETWORK COMMANDS <a name="network"></a>

### PowerShell
```powershell
# Get IP address info
Get-NetIPAddress

# Get network adapters
Get-NetAdapter

# Test connection (like ping but better)
Test-NetConnection google.com
Test-NetConnection google.com -Port 443

# Flush DNS cache
Clear-DnsClientCache

# View DNS cache
Get-DnsClientCache

# Get routing table
Get-NetRoute

# Reset network stack
netsh winsock reset
netsh int ip reset
```

### CMD
```cmd
:: Ping
ping google.com

:: Trace route
tracert google.com

:: DNS lookup
nslookup google.com

:: Show all connections
netstat -ano

:: Show open ports
netstat -an | findstr LISTENING

:: Release/renew IP
ipconfig /release
ipconfig /renew

:: Flush DNS
ipconfig /flushdns

:: Full network info
ipconfig /all
```

---

## 9. DISK & STORAGE <a name="disk"></a>

### PowerShell
```powershell
# List all disks
Get-Disk

# List all partitions
Get-Partition

# List all volumes
Get-Volume

# Check disk for errors
Repair-Volume -DriveLetter C -Scan

# Repair disk errors
Repair-Volume -DriveLetter C -OfflineScanAndFix

# Get disk space usage
Get-PSDrive -PSProvider FileSystem

# Find large files
Get-ChildItem -Path C:\ -Recurse -ErrorAction SilentlyContinue | Sort-Object Length -Descending | Select-Object -First 20 FullName, Length
```

### CMD
```cmd
:: Check disk (schedule on next reboot)
chkdsk C: /f /r

:: Disk cleanup
cleanmgr

:: Disk partitioning tool
diskpart

:: List volumes in diskpart
list volume

:: List disks in diskpart
list disk
```

---

## 10. SECURITY & PERMISSIONS <a name="security"></a>

### PowerShell
```powershell
# Check execution policy
Get-ExecutionPolicy

# Set execution policy (allows local scripts)
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser

# Check Secure Boot status
Confirm-SecureBootUEFI

# Check BitLocker status
manage-bde -status

# List local users
Get-LocalUser

# List local groups
Get-LocalGroup

# Add user to local admin group
Add-LocalGroupMember -Group "Administrators" -Member "Username"

# Check current user privileges
whoami /priv

# Check group memberships
whoami /groups

# Take ownership of a file
takeown /f "C:\path\to\file" /r /d y

# Grant full permissions to a file
icacls "C:\path\to\file" /grant Administrators:F
```

### Defender
```powershell
# Check Defender status
Get-MpComputerStatus

# Run quick scan
Start-MpScan -ScanType QuickScan

# Run full scan
Start-MpScan -ScanType FullScan

# Add exclusion path (stops Defender deleting your tools)
Add-MpPreference -ExclusionPath "C:\YourToolsFolder"

# Add exclusion by process name
Add-MpPreference -ExclusionProcess "winaero.exe"

# List current exclusions
Get-MpPreference | Select-Object ExclusionPath, ExclusionProcess

# Disable real-time protection (temporary, resets)
Set-MpPreference -DisableRealtimeMonitoring $true
```

---

## 11. USEFUL ONE-LINERS <a name="oneliners"></a>

```powershell
# Fix Windows reporting as Windows 10 instead of 11
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion" -Name "ProductName" -Value "Windows 11 Home"

# Check what Windows thinks it is
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion" | Select-Object ProductName, CurrentBuild, DisplayVersion, ReleaseId

# Find what's eating your CPU right now
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10 Name, CPU, WorkingSet

# Find what's eating your RAM right now
Get-Process | Sort-Object WorkingSet -Descending | Select-Object -First 10 Name, WorkingSet

# List all startup programs
Get-CimInstance Win32_StartupCommand | Select-Object Name, Command, Location

# Check for failed services
Get-Service | Where-Object { $_.Status -eq "Stopped" -and $_.StartType -eq "Automatic" }

# Export list of all installed programs
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* | Select-Object DisplayName, DisplayVersion, Publisher | Export-Csv "C:\installed-programs.csv" -NoTypeInformation

# Check Windows license status
slmgr /xpr

# Activate Windows (if needed)
slmgr /ato

# Generate full system health report
powercfg /energy
powercfg /batteryreport  # laptops only

# Clear temp files
Remove-Item -Path "$env:TEMP\*" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "C:\Windows\Temp\*" -Recurse -Force -ErrorAction SilentlyContinue

# Restart explorer (fixes frozen taskbar/desktop)
Stop-Process -Name explorer -Force; Start-Process explorer
```

---

## 12. EVENT LOG TROUBLESHOOTING <a name="eventlog"></a>

### Methodology — Root Cause Analysis
```
1. Find the ERROR event in Event Viewer
2. Look at the event BEFORE it — that's what Windows was trying to do when it failed
3. Follow the breadcrumbs: Event 400 (driver selected) → Event 411 (driver failed)
4. The failure point isn't where the problem lives — it's where it surfaces
5. Trace upstream to find the actual corruption
```

### Key Kernel-PnP Events (Microsoft-Windows-Kernel-PnP/Configuration)
| Event ID | Level | Meaning |
|----------|-------|---------|
| 400 | Info | Driver selected for a device — shows driver name, version, rank |
| 401 | Info | Device configured successfully |
| 410 | Warning | Driver install failed — device not configured |
| 411 | Error | Device failed to start — includes Problem code and Status |
| 420 | Warning | Device requires further installation |
| 430 | Info | Device removed |

### Common Problem Codes (from Event 411)
| Problem Code | Hex | Meaning | Typical Fix |
|-------------|-----|---------|-------------|
| Code 1 | 0x01 | Device not configured | Reinstall driver |
| Code 10 | 0x0A | Device cannot start | Update/rollback driver |
| Code 19 | 0x13 | Registry config incomplete/damaged | Check UpperFilters/LowerFilters in Class key |
| Code 22 | 0x16 | Device is disabled | Enable in Device Manager |
| Code 28 | 0x1C | No driver installed | Install correct driver |
| Code 31 | 0x1F | Device not working properly | Remove and re-detect device |
| Code 43 | 0x2B | Device reported a problem — stopped | Hardware failing or driver crash |

### Common NTSTATUS Codes (from Event 411 Status field)
| Status | Meaning |
|--------|---------|
| 0xc0000034 | STATUS_OBJECT_NAME_NOT_FOUND — registry key or file missing |
| 0xc0000010 | STATUS_INVALID_DEVICE_REQUEST — driver rejected the request |
| 0xc00000e5 | STATUS_INTERNAL_ERROR — driver internal failure |
| 0xc0000001 | STATUS_UNSUCCESSFUL — generic failure |
| 0xc000009a | STATUS_INSUFFICIENT_RESOURCES — out of memory/resources |

### PowerShell — Event Log Queries
```powershell
# List all devices currently in a problem state
pnputil /enum-devices /problem

# Get recent Kernel-PnP errors (Event 411 — device failed to start)
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Kernel-PnP/Configuration'; Id=411} -MaxEvents 20 | Format-List

# Get recent Kernel-PnP driver selections (Event 400 — what driver was picked)
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Kernel-PnP/Configuration'; Id=400} -MaxEvents 20 | Format-List

# Get all PnP config events from last 24 hours
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Kernel-PnP/Configuration'; StartTime=(Get-Date).AddDays(-1)} | Format-List

# Search System log for device errors
Get-WinEvent -FilterHashtable @{LogName='System'; Level=2} -MaxEvents 50 | Where-Object { $_.ProviderName -like "*PnP*" -or $_.ProviderName -like "*Driver*" }

# Export PnP events to CSV for analysis
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Kernel-PnP/Configuration'} -MaxEvents 100 | Export-Csv "C:\pnp-events.csv" -NoTypeInformation
```

### Code 19 Fix — UpperFilters / LowerFilters
```powershell
# Check for stale filters in a device class (replace GUID with target class)
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{CLASS-GUID}" | Select-Object UpperFilters, LowerFilters

# Common Class GUIDs
# Keyboard:  {4D36E96B-E325-11CE-BFC1-08002BE10318}
# Mouse:     {4D36E96F-E325-11CE-BFC1-08002BE10318}
# USB:       {36FC9E60-C465-11CF-8056-444553540000}
# Display:   {4D36E968-E325-11CE-BFC1-08002BE10318}
# Audio:     {4D36E96C-E325-11CE-BFC1-08002BE10318}
# DVD/CD:    {4D36E965-E325-11CE-BFC1-08002BE10318}
# Disk:      {4D36E967-E325-11CE-BFC1-08002BE10318}
# Network:   {4D36E972-E325-11CE-BFC1-08002BE10318}

# Remove a stale UpperFilter (example: remove SynTP from keyboard class)
$val = (Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4D36E96B-E325-11CE-BFC1-08002BE10318}").UpperFilters
$val = $val | Where-Object { $_ -ne "SynTP" }
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4D36E96B-E325-11CE-BFC1-08002BE10318}" -Name UpperFilters -Value $val

# Nuclear option — remove UpperFilters entirely (device will use default stack)
Remove-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4D36E96B-E325-11CE-BFC1-08002BE10318}" -Name UpperFilters
```

### Other Common Event Log Sources
```powershell
# DCOM errors (Event 10016) — usually permissions issues
Get-WinEvent -FilterHashtable @{LogName='System'; Id=10016} -MaxEvents 10 | Format-List

# CAPI2 certificate errors
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-CAPI2/Operational'} -MaxEvents 20 | Format-List

# Windows Update errors
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-WindowsUpdateClient/Operational'; Level=2} -MaxEvents 20 | Format-List

# Disk/Storage errors
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='disk'} -MaxEvents 20 | Format-List

# Blue Screen / Bugcheck events
Get-WinEvent -FilterHashtable @{LogName='System'; Id=1001} -MaxEvents 10 | Format-List

# GPU / Display driver events
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Display'} -MaxEvents 20 | Format-List
```

### OS Migration Cleanup — Finding Orphaned Drivers & Filters
```powershell
# List all UpperFilters and LowerFilters across ALL device classes
Get-ChildItem "HKLM:\SYSTEM\CurrentControlSet\Control\Class" | ForEach-Object {
    $props = Get-ItemProperty $_.PSPath
    if ($props.UpperFilters -or $props.LowerFilters) {
        [PSCustomObject]@{
            Class = $props.Class
            GUID = $_.PSChildName
            UpperFilters = ($props.UpperFilters -join ', ')
            LowerFilters = ($props.LowerFilters -join ', ')
        }
    }
} | Format-Table -AutoSize

# Find ghost/non-present devices still in registry
pnputil /enum-devices /disconnected

# Remove a specific ghost device by instance ID
pnputil /remove-device "HID\VID_1C4F&PID_0084&MI_01&Col03\7&c5963c1&0&0002"
```

---

## 🔑 QUICK REFERENCE — COMMON REGISTRY PATHS

| What | Path |
|------|------|
| Windows Version Info | `HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion` |
| Startup Programs (Machine) | `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` |
| Startup Programs (User) | `HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` |
| Installed Programs | `HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall` |
| Windows Update Settings | `HKLM:\SOFTWARE\Microsoft\WindowsUpdate\UX\Settings` |
| Defender Exclusions | `HKLM:\SOFTWARE\Microsoft\Windows Defender\Exclusions` |
| Environment Variables | `HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Environment` |
| Services | `HKLM:\SYSTEM\CurrentControlSet\Services` |

---

---

© 2025-2026 Sync-First Essentials LLC
Remember Chaos is Peace, Peace is Chaos!

*Field Notes — Jon Merriman / SFE LLC | Updated 2026-05-10*
