# Splunk Enterprise - Installation & Initial Configuration

## Overview

Splunk Enterprise is the core of this lab. It acts as both the **indexer** (stores and parses log data) and the **search head** (provides the web UI for searching). It runs as a Windows service on the Windows 10 VM.

![splunkinstall.png](/home/lzy/Documents/GitHub/Splunk-SIEM-Homelab/screenshots/phase2/splunkinstall.png)

---

## Step 1 - Download Splunk Enterprise

On the **Windows 10 VM**, open a browser and go to:

```
https://www.splunk.com/en_us/download/splunk-enterprise.html
```

- Sign in or create a free Splunk account
- Select **Windows** as the platform
- Download the `.msi` installer (e.g., `splunk-9.x.x-windows-x64.msi`)

> As of 2026, Splunk Enterprise free trial includes a **500MB/day** indexing limit. This lab will use well under that.

---

## Step 2 - Install Splunk Enterprise

### GUI Installer

1. Double-click the downloaded `.msi` file
2. Accept the License Agreement
3. Choose **Customize Options** when prompted (don't use defaults)
4. Set the install path: `C:\Program Files\Splunk` (default is fine)
5. When asked to create an admin account:
   - Username: `admin`
   - Password: *(choose something you'll remember - write it down)*
6. Leave **"Start Splunk Enterprise at login"** checked
7. Click **Install**
8. When complete, click **Finish** - Splunk will attempt to open in your browser

---

## Step 3 - First Login & License Setup

1. Open a browser on the Windows 10 VM and go to:
   
   ```
   http://127.0.0.1:8000
   ```

2. Log in with `admin` and the password you set

3. On first login you'll see the **License** screen:
   
   - Select **"Start a free trial"** (60-day Enterprise trial, then reverts to free license)
   - OR select **"Use the free license"** if you don't need the trial features

4. Click through the setup wizard - defaults are fine for this lab

> To access Splunk from your Linux Mint Host browser, navigate to:
> 
> ```
> http://192.168.0.123:8000
> ```
> 
> Replace `192.168.0.123` with your Windows 10 VM's static IP.

---

## Step 4 - Enable the Receiving Port (9997)

This tells Splunk Enterprise to **listen** for incoming log data from the Universal Forwarder.

### Via Splunk Web UI:

1. Log into Splunk Web (`http://127.0.0.1:8000`)
2. Go to **Settings** → **Forwarding and Receiving**
3. Under **Receive Data**, click **"Configure receiving"**
4. Click **"New Receiving Port"**
5. Enter port: `9997`
6. Click **Save**

You should see port `9997` listed as an active receiving port.

### Verify via PowerShell (after enabling):

```powershell
# Confirm Splunk is listening on 9997
netstat -ano | findstr :9997

# Expected output (LISTENING):
# TCP    0.0.0.0:9997    0.0.0.0:0    LISTENING    XXXX
```

---

## Step 5 - Create a Dedicated Index

Creating a separate index keeps Windows Event Logs organized and makes searching faster.

### Via Splunk Web UI:

1. Go to **Settings** → **Indexes**
2. Click **"New Index"**
3. Fill in:
   - Index Name: `wineventlog`
   - Index Data Type: `Events`
   - Leave all other settings at defaults
4. Click **Save**

---

## Step 6 - Manage Splunk as a Windows Service

Splunk installs itself as a Windows service called `SplunkD`.

```powershell
# Check service status
Get-Service -Name "SplunkD"

# Start Splunk
Start-Service -Name "SplunkD"

# Stop Splunk
Stop-Service -Name "SplunkD"

# Restart Splunk
Restart-Service -Name "SplunkD"
```

Via Splunk CLI:

```powershell
cd "C:\Program Files\Splunk\bin"

# Start
.\splunk.exe start

# Stop
.\splunk.exe stop

# Restart
.\splunk.exe restart

# Check status
.\splunk.exe status
```

---

## Step 7 — Verify Splunk is Running Correctly

```powershell
# Check that Splunk web and management ports are open
netstat -ano | findstr ":8000"
netstat -ano | findstr ":8089"
netstat -ano | findstr ":9997"
```

Expected output for each:

```
TCP    0.0.0.0:8000    0.0.0.0:0    LISTENING    XXXX
TCP    0.0.0.0:8089    0.0.0.0:0    LISTENING    XXXX
TCP    0.0.0.0:9997    0.0.0.0:0    LISTENING    XXXX
```

### From Kali Linux - Test Web UI is reachable:

```bash
# Replace with your Windows 10 IP
curl -I http://192.168.0.123:8000

# Expected: HTTP/1.1 303 See Other  (Splunk redirects to login)
```

---

## Key File Paths

| Path                                        | Description                    |
| ------------------------------------------- | ------------------------------ |
| `C:\Program Files\Splunk\`                  | Splunk Enterprise install root |
| `C:\Program Files\Splunk\bin\splunk.exe`    | Splunk CLI                     |
| `C:\Program Files\Splunk\etc\system\local\` | Local config overrides         |
| `C:\Program Files\Splunk\var\log\splunk\`   | Splunk internal logs           |
| `C:\Program Files\Splunk\var\lib\splunk\`   | Index data storage             |

---

## Troubleshooting

| Problem                               | Fix                                                                        |
| ------------------------------------- | -------------------------------------------------------------------------- |
| Can't reach `:8000` from host browser | Check Phase 1 firewall rule for port 8000                                  |
| Login page loads but login fails      | Run `.\splunk.exe edit user admin --password NEWPASS --auth admin:OLDPASS` |
| Port 9997 not listening               | Go to Settings → Forwarding and Receiving and re-add it                    |
| Splunk service won't start            | Check `C:\Program Files\Splunk\var\log\splunk\splunkd.log` for errors      |
| `SplunkD` service missing             | Re-run installer or run `.\splunk.exe enable boot-start`                   |

---

## Next Step

Splunk Enterprise is now running and ready to receive data.
Proceed to **[universal-forwarder.md](./universal-forwarder.md)** to install the log shipping agent.
