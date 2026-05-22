# FIELD FIXES OF A MIGRATED OS, So You Don't Have To!
### PowerShell · CMD · Bcdedit · DISM · SFC · Registry · Drivers · Linux
> Field Notes — Jon Merriman / SFE LLC
> Last Updated: 2026-05-19

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
13. [Linux Networking — Manual IP / NIC Recovery](#linuxnet)
14. [Event Log Troubleshooting](#eventlog)
15. [Archive & Compression](#archives)

---

## 1. SYSTEM INFO & DIAGNOSTICS <a name="system-info"></a>

### PowerShell
```powershell
# Full Windows version info
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion" | Select-Object ProductName, CurrentBuild, DisplayVersion, ReleaseId

# Check Windows edition — opens GUI popup
winver

# Full system info dump
Get-ComputerInfo

# Quick OS version
[System.Environment]::OSVersion

# Check if running as Administrator (returns True/False)
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
:: Full system info
systeminfo

:: Windows version string
ver

:: All environment variables
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

## 4. BOOT CONFIGURATION (Bcdedit) <a name="bcdedit"></a>

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

# Pause updates for 35 days via registry
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

# Start / Stop / Restart a service
Start-Service -Name "ServiceName"
Stop-Service -Name "ServiceName" -Force
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

# Clear temp files
Remove-Item -Path "$env:TEMP\*" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "C:\Windows\Temp\*" -Recurse -Force -ErrorAction SilentlyContinue

# Restart explorer (fixes frozen taskbar/desktop)
Stop-Process -Name explorer -Force; Start-Process explorer
```

---

## 🔑 12. COMMON REGISTRY PATHS <a name="regpaths"></a>

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
| File Type Associations | `HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\FileExts` |
| Context Menu Entries | `HKCR:\*\shell` |
| Power Settings | `HKLM:\SYSTEM\CurrentControlSet\Control\Power` |
| Network Adapters | `HKLM:\SYSTEM\CurrentControlSet\Control\Network` |

---

## 🐧 13. LINUX NETWORKING — MANUAL IP / NIC RECOVERY <a name="linuxnet"></a>

> ⚠️ Works on any Linux: Proxmox, Debian, Ubuntu, Kali, etc. Use when a NIC shows DOWN or no IP assigned — common after hardware swaps or moving drives to a new motherboard.

### Check NIC Status
```bash
# Show all network interfaces and their state
ip a

# Quick view — interface names and UP/DOWN state
ip link show

# Check a specific interface
ip link show enp1s0
```

### Bring a NIC Up Manually (Temporary — Resets on Reboot)
```bash
# Step 1 — Bring the interface up
ip link set enp1s0 up

# Step 2 — Assign a static IP with subnet
ip addr add 192.168.100.24/24 dev enp1s0

# Step 3 — Add default gateway
# (skip if you get "File exists" — route already there)
ip route add default via 192.168.100.1

# Verify — ping your router
ping 192.168.100.1
```

### Common Interface Names
```
enp1s0, enp2s0  — PCIe ethernet (desktops/servers)
eth0, eth1      — older naming convention
ens18, ens19    — common in VMs (Proxmox/VMware)
lo              — loopback, ignore this one
```

### Make It Permanent — Proxmox / Debian
```bash
# Edit the network config
nano /etc/network/interfaces

# Add/update your interface block:
# auto enp1s0
# iface enp1s0 inet static
#     address 192.168.100.24
#     netmask 255.255.255.0
#     gateway 192.168.100.1

# Apply without reboot:
systemctl restart networking
```

### Make It Permanent — Ubuntu / Netplan
```bash
# Edit netplan config (Ubuntu 18.04+)
nano /etc/netplan/00-installer-config.yaml

# Example config:
# network:
#   version: 2
#   ethernets:
#     enp1s0:
#       addresses: [192.168.100.24/24]
#       gateway4: 192.168.100.1
#       nameservers:
#         addresses: [8.8.8.8, 1.1.1.1]

# Apply changes
netplan apply
```

### DNS & Connectivity Checks
```bash
# Test gateway
ping 192.168.100.1

# Test internet — no DNS involved
ping 8.8.8.8

# Test DNS resolution
ping google.com

# Check current DNS servers
cat /etc/resolv.conf

# Flush DNS cache (systemd)
systemd-resolve --flush-caches

# Manual DNS lookup
nslookup google.com
dig google.com
```

### Useful Network Diagnostics
```bash
# Show routing table
ip route show

# Show ARP table — who's on your network
arp -n

# Show open ports and listening services
ss -tuln

# Same but with process names
ss -tulnp

# Trace route to a host
traceroute google.com

# Check interface stats — errors, drops
ip -s link show enp1s0
```

---

## 📋 14. EVENT LOG TROUBLESHOOTING <a name="eventlog"></a>

> ⚠️ Don't stare at the error — look at the event BEFORE it. That tells you what Windows was trying to do when it failed.

### Root Cause Analysis Steps
```
1. Find the ERROR event in Event Viewer
2. Look at the event BEFORE it — that's what Windows was trying to do when it failed
3. Follow the breadcrumbs: Event 400 (driver selected) → Event 411 (driver failed)
4. The failure point isn't where the problem lives — it's where it surfaces
5. Trace upstream to find the actual corruption
```

### Key Kernel-PnP Events
| Event ID | Level | Meaning |
|----------|-------|---------|
| 400 | Info | Driver selected for device — shows driver name, version, rank |
| 401 | Info | Device configured successfully |
| 410 | Warning | Driver install failed — device not configured |
| 411 | Error | Device failed to start — includes Problem code and Status |
| 420 | Warning | Device requires further installation |
| 430 | Info | Device removed |

### Common Problem Codes (Event 411)
| Code | Hex | Meaning | Fix |
|------|-----|---------|-----|
| Code 1 | 0x01 | Device not configured | Reinstall driver |
| Code 10 | 0x0A | Device cannot start | Update/rollback driver |
| Code 19 | 0x13 | Registry config incomplete/damaged | Check UpperFilters/LowerFilters |
| Code 22 | 0x16 | Device is disabled | Enable in Device Manager |
| Code 28 | 0x1C | No driver installed | Install correct driver |
| Code 31 | 0x1F | Device not working properly | Remove and re-detect |
| Code 43 | 0x2B | Device reported a problem | Hardware failing or driver crash |

### Common NTSTATUS Codes (Event 411 Status field)
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

# Get recent driver selections (Event 400 — what driver was picked)
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Kernel-PnP/Configuration'; Id=400} -MaxEvents 20 | Format-List

# Get all PnP config events from last 24 hours
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Kernel-PnP/Configuration'; StartTime=(Get-Date).AddDays(-1)} | Format-List

# Search System log for device errors
Get-WinEvent -FilterHashtable @{LogName='System'; Level=2} -MaxEvents 50 | Where-Object { $_.ProviderName -like "*PnP*" -or $_.ProviderName -like "*Driver*" }

# Export PnP events to CSV for analysis
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Kernel-PnP/Configuration'} -MaxEvents 100 | Export-Csv "C:\pnp-events.csv" -NoTypeInformation
```

### PowerShell — Code 19 Fix (UpperFilters / LowerFilters)
```powershell
# Check for stale filters in a device class (replace GUID)
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{CLASS-GUID}" | Select-Object UpperFilters, LowerFilters

# Common Class GUIDs:
# Keyboard:  {4D36E96B-E325-11CE-BFC1-08002BE10318}
# Mouse:     {4D36E96F-E325-11CE-BFC1-08002BE10318}
# USB:       {36FC9E60-C465-11CF-8056-444553540000}
# Display:   {4D36E968-E325-11CE-BFC1-08002BE10318}
# Audio:     {4D36E96C-E325-11CE-BFC1-08002BE10318}
# DVD/CD:    {4D36E965-E325-11CE-BFC1-08002BE10318}
# Disk:      {4D36E967-E325-11CE-BFC1-08002BE10318}
# Network:   {4D36E972-E325-11CE-BFC1-08002BE10318}

# Remove a stale UpperFilter (example: remove SynTP from keyboard)
$val = (Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4D36E96B-E325-11CE-BFC1-08002BE10318}").UpperFilters
$val = $val | Where-Object { $_ -ne "SynTP" }
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4D36E96B-E325-11CE-BFC1-08002BE10318}" -Name UpperFilters -Value $val

# Nuclear option — remove UpperFilters entirely
Remove-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4D36E96B-E325-11CE-BFC1-08002BE10318}" -Name UpperFilters
```

### PowerShell — Other Common Event Sources
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

### PowerShell — OS Migration Cleanup (Orphaned Drivers & Filters)
```powershell
# List ALL UpperFilters and LowerFilters across every device class
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

## 📦 15. ARCHIVE & COMPRESSION <a name="archives"></a>

### PowerShell / CMD — Windows
```powershell
:: Extract a .zip file (built into modern Windows)
tar -xf archive.zip

:: Extract .zip to a specific folder
tar -xf archive.zip -C C:\extracted

# PowerShell — extract a zip
Expand-Archive archive.zip -DestinationPath C:\extracted

# PowerShell — create a zip from a folder
Compress-Archive -Path C:\MyFolder\* -DestinationPath C:\backup.zip

# PowerShell — add files to existing zip
Compress-Archive -Path C:\newfile.txt -DestinationPath C:\backup.zip -Update

:: Extract .tar.gz with Windows tar
tar -xzf archive.tar.gz

:: Extract .tar.bz2
tar -xjf archive.tar.bz2

:: List contents of a tar/zip without extracting
tar -tf archive.tar.gz
```

### Linux — zip / unzip
```bash
# Install unzip if missing (Debian/Ubuntu/Proxmox)
sudo apt install unzip

# Install unzip on Fedora/CentOS
sudo dnf install unzip

# Extract a .zip file
unzip archive.zip

# Extract to a specific directory
unzip archive.zip -d /home/user/extracted/

# List contents without extracting
unzip -l archive.zip

# Test if archive is valid
unzip -t archive.zip

# Overwrite existing files without prompting
unzip -o archive.zip

# Create a zip file
zip -r backup.zip /home/user/myfolder/
```

### Linux — tar (tarballs)
```bash
# Extract .tar (uncompressed)
tar -xf archive.tar

# Extract .tar.gz or .tgz (gzip compressed)
tar -xzf archive.tar.gz

# Extract .tar.bz2 (bzip2 compressed)
tar -xjf archive.tar.bz2

# Extract .tar.xz (xz compressed)
tar -xJf archive.tar.xz

# Extract to a specific directory
tar -xf archive.tar.gz -C /home/user/extracted/

# List contents without extracting
tar -tf archive.tar.gz

# Verbose — see files as they extract
tar -xvf archive.tar.gz

# Create a .tar.gz archive from a folder
tar -czf backup.tar.gz /home/user/myfolder/

# Create a .tar.bz2 archive (better compression, slower)
tar -cjf backup.tar.bz2 /home/user/myfolder/
```

### Linux — 7z (7-Zip)
```bash
# Install 7zip (Debian/Ubuntu/Proxmox)
sudo apt install p7zip-full

# Install 7zip (Fedora/CentOS)
sudo dnf install p7zip p7zip-plugins

# Extract a .7z file
7z x archive.7z

# Extract to a specific directory
7z x archive.7z -o/home/user/extracted/

# List contents without extracting
7z l archive.7z

# Create a .7z archive
7z a backup.7z /home/user/myfolder/

# 7z can also handle .zip, .tar.gz, .rar, .iso, etc.
7z x archive.zip
7z x archive.rar
```

### Quick Reference — tar Flags
| Flag | What It Does |
|------|-------------|
| -x | Extract files from archive |
| -c | Create a new archive |
| -f | Specify the archive filename |
| -v | Verbose — show file names during operation |
| -z | Filter through gzip (.tar.gz) |
| -j | Filter through bzip2 (.tar.bz2) |
| -J | Filter through xz (.tar.xz) |
| -t | List contents without extracting |
| -C | Extract to a specific directory |

---

© 2025-2026 Jon Merriman / Sync-First Essentials LLC — All Rights Reserved
*"Remember Chaos is Peace, Peace is Chaos!!"*
🌐 [syncfirstessentials.com](https://syncfirstessentials.com) | ✉️ jon@syncfirstessentials.com
Created by: Jon Merriman / Juggalospsyco420 / GhostDragon420
