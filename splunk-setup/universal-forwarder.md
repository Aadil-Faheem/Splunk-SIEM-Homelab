# Splunk Universal Forwarder - Installation & Configuration

## Overview

The **Splunk Universal Forwarder (UF)** is a lightweight agent that runs as a Windows service. Its only job is to monitor log sources and forward data to a Splunk indexer. It has no UI of its own - it's configured entirely through `.conf` files.

In this lab, the Universal Forwarder and Splunk Enterprise both run on the same Windows 10 VM. The forwarder ships logs to `127.0.0.1:9997` (localhost) where Splunk Enterprise is listening.

```
[ Windows 10 VM ]
  ┌────────────────────────────────────────┐
  │                                        │
  │  Universal Forwarder                   │
  │  (monitors Windows Event Logs)         │
  │          │                             │
  │          │ sends to 127.0.0.1:9997     │
  │          ▼                             │
  │  Splunk Enterprise                     │
  │  (indexes & stores data)               │
  │  Web UI: localhost:8000                │
  │                                        │
  └────────────────────────────────────────┘
```

---

## Step 1 — Download the Universal Forwarder

On the **Windows 10 VM**, go to:

```
https://www.splunk.com/en_us/download/universal-forwarder.html
```

- Select **Windows** → **64-bit**
- Download the `.msi` file (e.g., `splunkforwarder-9.x.x-windows-x64.msi`)

> Use the same Splunk account you created for the Enterprise download.

---

## Step 2 - Install the Universal Forwarder

### GUI Installer

1. Double-click the `.msi` file
2. Accept the License Agreement - check **"An on-premises Splunk Enterprise instance"**
3. Set credentials:
   - Username: `admin`
   - Password: *(can be same as or different from Splunk Enterprise password)*
4. On the **"Deployment Server"** screen - **leave blank** (not needed for this lab)
5. On the **"Receiving Indexer"** screen:
   - Host/IP: `127.0.0.1`
   - Port: `9997`
6. Click **Install** → **Finish**

---

## Step 3 - Verify the Service is Running

```powershell
# Check the forwarder service status
Get-Service -Name "SplunkForwarder"

# Expected:
# Status   Name               DisplayName
# ------   ----               -----------
# Running  SplunkForwarder    SplunkForwarder

# If not running, start it
Start-Service -Name "SplunkForwarder"
```

---

## Step 4 - Configure outputs.conf

`outputs.conf` tells the forwarder **where to send data**. Even if you set the indexer during install, it's best practice to verify and set this manually.

**File path:**

```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\outputs.conf
```

Open **Notepad as Administrator** and navigate to the path above. If the file doesn't exist, create it.

**Contents of `outputs.conf`:**

```ini
[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = 127.0.0.1:9997

[tcpout-server://127.0.0.1:9997]
```

> See **[outputs-conf.md](./outputs-conf.md)** for a full explanation of every field and alternative configurations.

---

## Step 5 - Configure inputs.conf

`inputs.conf` tells the forwarder **what to monitor**. This is where you define the Windows Event Log channels to collect.

**File path:**

```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```

Open **Notepad as Administrator** and navigate to the path above. Create or edit the file with the following:

```ini
[WinEventLog://Security]
index = wineventlog
sourcetype = WinEventLog:Security
disabled = false
start_from = oldest
current_only = false

[WinEventLog://System]
index = wineventlog
sourcetype = WinEventLog:System
disabled = false
start_from = oldest
current_only = false

[WinEventLog://Application]
index = wineventlog
sourcetype = WinEventLog:Application
disabled = false
start_from = oldest
current_only = false
```

> See **[inputs-conf.md](./inputs-conf.md)** for the full breakdown of every option, additional log sources, and how to add Sysmon.

---

## Step 6 - Restart the Forwarder to Apply Config

Any time you change `.conf` files, you must restart the forwarder service.

```powershell
# Restart via PowerShell
Restart-Service -Name "SplunkForwarder"

# OR via Splunk CLI (run as Administrator)
cd "C:\Program Files\SplunkUniversalForwarder\bin"
.\splunk.exe restart
```

---

## Step 7 - Verify Data is Flowing into Splunk

### In Splunk Web UI:

1. Open `http://127.0.0.1:8000` and log in
2. Go to **Search & Reporting**
3. Run this search in the search bar:

```spl
index=wineventlog
| head 20
```

4. Change the time range to **"Last 15 minutes"** or **"All time"**
5. You should see Windows Event Log entries appearing

If events appear - **the pipeline is working**.

### Check by Sourcetype:

```spl
index=wineventlog sourcetype="WinEventLog:Security"
| head 10
```

```spl
index=wineventlog sourcetype="WinEventLog:System"
| head 10
```

### Check Event Count Over Time:

```spl
index=wineventlog
| timechart count by sourcetype
```

### Confirm All Three Log Channels are Ingesting:

```spl
index=wineventlog
| stats count by sourcetype
```

Expected output:

| sourcetype              | count |
| ----------------------- | ----- |
| WinEventLog:Application | XXXX  |
| WinEventLog:Security    | XXXX  |
| WinEventLog:System      | XXXX  |

---

## Key File Paths

| Path                                                                      | Description             |
| ------------------------------------------------------------------------- | ----------------------- |
| `C:\Program Files\SplunkUniversalForwarder\`                              | UF install root         |
| `C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe`                | UF CLI                  |
| `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`  | What to monitor         |
| `C:\Program Files\SplunkUniversalForwarder\etc\system\local\outputs.conf` | Where to send data      |
| `C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log`    | Forwarder internal logs |

---

## Useful UF CLI Commands

Run these from `C:\Program Files\SplunkUniversalForwarder\bin\` in **PowerShell as Administrator**:

```powershell
# Check forwarder status
.\splunk.exe status

# List configured monitors
.\splunk.exe list monitor

# Add a monitor on the fly (alternative to editing inputs.conf)
.\splunk.exe add monitor "C:\Windows\System32\winevt\Logs\Security.evtx" -index wineventlog

# List forward-servers (confirm outputs)
.\splunk.exe list forward-server

# View internal forwarder log
Get-Content "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" -Tail 50
```

---

## Troubleshooting

| Problem                                              | Fix                                                                                               |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| No data in `index=wineventlog`                       | Check `SplunkForwarder` service is Running; check `splunkd.log`                                   |
| Events show in `main` index instead of `wineventlog` | Verify `index = wineventlog` is set in `inputs.conf` and restart forwarder                        |
| `list forward-server` shows no servers               | Check `outputs.conf` exists at the correct path and has no typos                                  |
| Security log is empty but others work                | Run Splunk/forwarder **as Administrator** - Security logs require elevated access                 |
| `splunkd.log` shows `Connection refused`             | Confirm Splunk Enterprise is running and port 9997 is listening (`netstat -ano \| findstr :9997`) |
| Config changes not taking effect                     | Always restart the forwarder after editing any `.conf` file                                       |

---

## Next Step

Data is now flowing from Windows Event Logs → Universal Forwarder → Splunk Enterprise.

Proceed to **[inputs-conf.md](./inputs-conf.md)** for a deep dive into monitoring options, or jump to **[Phase 3](../attack-simulation/checklist.md)** if your logs are flowing and you're ready to start attacking.
