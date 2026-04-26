# inputs.conf - Configuration Reference

## Overview

`inputs.conf` tells the Splunk Universal Forwarder **what data to collect**. Every log source you want to monitor must have an entry here. This file lives in the forwarder's `local` directory so it overrides any defaults.

**File location:**

```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```

> Always edit files in `local\` - never edit files in `default\`. The `local\` directory is for your custom overrides and survives Splunk upgrades.

---

## Core Concepts

Each stanza in `inputs.conf` starts with a **stanza header** in square brackets that defines the input type and source. Common Windows stanza types:

| Stanza Type                   | Purpose                                        |
| ----------------------------- | ---------------------------------------------- |
| `[WinEventLog://ChannelName]` | Monitor a Windows Event Log channel            |
| `[monitor://path\to\file]`    | Monitor a flat file or directory for new lines |
| `[script://path\to\script]`   | Run a script and index its output              |

---

## Full inputs.conf for This Lab

This is the complete `inputs.conf` for the lab - copy this directly into your file. You can uncomment optional sections as needed.

```ini
# =============================================================
# Splunk Universal Forwarder — inputs.conf
# Windows 10 Victim / SIEM Lab
# =============================================================

# -------------------------------------------------------
# REQUIRED: Core Windows Event Logs
# -------------------------------------------------------

[WinEventLog://Security]
index = wineventlog
sourcetype = WinEventLog:Security
disabled = false
start_from = oldest
current_only = false
checkpointInterval = 5

[WinEventLog://System]
index = wineventlog
sourcetype = WinEventLog:System
disabled = false
start_from = oldest
current_only = false
checkpointInterval = 5

[WinEventLog://Application]
index = wineventlog
sourcetype = WinEventLog:Application
disabled = false
start_from = oldest
current_only = false
checkpointInterval = 5

# -------------------------------------------------------
# OPTIONAL: Sysmon (install Sysmon first - see note below)
# Sysmon provides deep process, network, and file telemetry
# -------------------------------------------------------

#[WinEventLog://Microsoft-Windows-Sysmon/Operational]
#index = wineventlog
#sourcetype = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
#disabled = false
#renderXml = true
#start_from = oldest

# -------------------------------------------------------
# OPTIONAL: Windows PowerShell Logging
# -------------------------------------------------------

#[WinEventLog://Microsoft-Windows-PowerShell/Operational]
#index = wineventlog
#sourcetype = WinEventLog:PowerShell
#disabled = false
#start_from = oldest

# -------------------------------------------------------
# OPTIONAL: Windows Defender Logs
# -------------------------------------------------------

#[WinEventLog://Microsoft-Windows-Windows Defender/Operational]
#index = wineventlog
#sourcetype = WinEventLog:WindowsDefender
#disabled = false
#start_from = oldest

# -------------------------------------------------------
# OPTIONAL: RDP / Terminal Services
# -------------------------------------------------------

#[WinEventLog://Microsoft-Windows-TerminalServices-RemoteConnectionManager/Operational]
#index = wineventlog
#sourcetype = WinEventLog:RDP
#disabled = false
#start_from = oldest

# -------------------------------------------------------
# OPTIONAL: Monitor a specific log file on disk
# -------------------------------------------------------

#[monitor://C:\Windows\System32\winevt\Logs\Security.evtx]
#index = wineventlog
#sourcetype = WinEventLog:Security
#disabled = false
```

---

## Field Reference - WinEventLog Stanzas

| Field                | Description                                                     | Recommended Value           |
| -------------------- | --------------------------------------------------------------- | --------------------------- |
| `index`              | Which Splunk index to store events in                           | `wineventlog`               |
| `sourcetype`         | Labels the data type for Splunk parsing                         | `WinEventLog:Security` etc. |
| `disabled`           | Set to `false` to enable this input                             | `false`                     |
| `start_from`         | `oldest` ingests all existing events; `newest` only new ones    | `oldest` for initial setup  |
| `current_only`       | `false` = catch up on existing events; `true` = only real-time  | `false`                     |
| `checkpointInterval` | How often (seconds) the forwarder saves its position in the log | `5`                         |
| `renderXml`          | For Sysmon - renders the full XML event format                  | `true` (Sysmon only)        |

---

## Adding Sysmon (Highly Recommended for Phase 3)

Sysmon is a free Microsoft Sysinternals tool that dramatically improves Windows telemetry. It logs process creation, network connections, file creation, registry changes, and more - with far more detail than standard Windows Event Logs alone.

### Install Sysmon on Windows 10:

1. Download Sysmon from Microsoft:
   
   ```
   https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
   ```

2. Download a community Sysmon config (SwiftOnSecurity is the standard):
   
   ```
   https://github.com/SwiftOnSecurity/sysmon-config
   ```
   
   Download `sysmonconfig-export.xml`

3. Install Sysmon with the config (run in **PowerShell as Administrator**):
   
   ```powershell
   # Navigate to where you downloaded sysmon
   cd C:\Users\YourUser\Downloads\
   
   # Install with config
   .\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
   
   # Verify it's running
   Get-Service -Name "Sysmon64"
   ```

4. Uncomment the Sysmon stanza in `inputs.conf` (shown above)

5. Restart the forwarder:
   
   ```powershell
   Restart-Service -Name "SplunkForwarder"
   ```

6. Verify Sysmon data in Splunk:
   
   ```spl
   index=wineventlog sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational"
   | head 20
   ```

---

## Enable PowerShell Script Block Logging (Optional but Useful)

PowerShell logging captures every PowerShell command executed on the system - very useful for detecting attacker activity.

Run in **PowerShell as Administrator** on Windows 10:

```powershell
# Enable PowerShell Module Logging
$basePath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging"
New-Item -Path $basePath -Force
New-ItemProperty -Path $basePath -Name "EnableModuleLogging" -Value 1 -PropertyType DWord -Force

# Enable Script Block Logging
$sbPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
New-Item -Path $sbPath -Force
New-ItemProperty -Path $sbPath -Name "EnableScriptBlockLogging" -Value 1 -PropertyType DWord -Force
```

After enabling, uncomment the PowerShell stanza in `inputs.conf` and restart the forwarder.

---

## Verify What the Forwarder is Monitoring

After restarting the forwarder, confirm what inputs are active:

```powershell
cd "C:\Program Files\SplunkUniversalForwarder\bin"

# List all active monitors
.\splunk.exe list monitor

# Expected output includes:
# Monitored: WinEventLog://Security
# Monitored: WinEventLog://System
# Monitored: WinEventLog://Application
```

---

## Apply Changes

**Every time you edit `inputs.conf`, restart the forwarder:**

```powershell
Restart-Service -Name "SplunkForwarder"
```

---

## Validate in Splunk

Run these searches in Splunk Web (`http://localhost:8000`) to confirm each source is flowing:

```spl
# Count events per source type
index=wineventlog
| stats count by sourcetype
```

```spl
# See the most recent events from Security log
index=wineventlog sourcetype="WinEventLog:Security"
| sort -_time
| head 10
| table _time, EventCode, Message, host
```

```spl
# Confirm Sysmon data (if installed)
index=wineventlog sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational"
| stats count by EventID
```
