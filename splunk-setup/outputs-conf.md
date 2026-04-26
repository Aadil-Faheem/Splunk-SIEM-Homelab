# outputs.conf - Configuration Reference

## Overview

`outputs.conf` tells the Splunk Universal Forwarder **where to send data**. It defines the destination indexer (Splunk Enterprise) and controls how the forwarder manages the connection - buffering, load balancing, SSL, and more.

**File location:**

```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\outputs.conf
```

---

## Minimum Required Configuration (This Lab)

For this lab, the Universal Forwarder and Splunk Enterprise both run on the same machine, so the destination is `127.0.0.1:9997` (localhost).

```ini
# =============================================================
# Splunk Universal Forwarder — outputs.conf
# Windows 10 Victim / SIEM Lab
# =============================================================

[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = 127.0.0.1:9997

[tcpout-server://127.0.0.1:9997]
```

That's all you need for a single local indexer. The sections below explain each field and cover more advanced configurations.

---

## Field Reference

### `[tcpout]` - Global TCP Output Settings

| Field             | Description                                                          | Value                     |
| ----------------- | -------------------------------------------------------------------- | ------------------------- |
| `defaultGroup`    | The output group to use by default for all forwarded data            | Name of your group stanza |
| `disabled`        | Disable all TCP output (set `false` to enable)                       | `false`                   |
| `indexAndForward` | If `true`, the forwarder also indexes data locally (not needed here) | `false`                   |

### `[tcpout:groupName]` - Output Group

| Field             | Description                                                 | Example          |
| ----------------- | ----------------------------------------------------------- | ---------------- |
| `server`          | The indexer(s) to send data to — `host:port`                | `127.0.0.1:9997` |
| `autoLBFrequency` | Seconds between auto load balancing across multiple servers | `30`             |
| `autoLBVolume`    | Bytes before switching to next server in the group          | `10485760`       |
| `compressed`      | Compress data before sending (reduces bandwidth)            | `true`           |

### `[tcpout-server://host:port]` - Per-Server Settings

This is optional for basic configs but required if you add SSL or per-server overrides.

---

## Alternative Configurations

### If Splunk Enterprise Were on a Separate Machine

If your indexer was on a different VM (e.g., a dedicated server at `192.168.0.200`), you would change the server address:

```ini
[tcpout]
defaultGroup = lab-indexer-group

[tcpout:lab-indexer-group]
server = 192.168.0.200:9997

[tcpout-server://192.168.0.200:9997]
```

### Multiple Indexers (Load Balancing)

The forwarder can automatically distribute data across multiple indexers:

```ini
[tcpout]
defaultGroup = indexer-pool

[tcpout:indexer-pool]
server = 192.168.0.200:9997, 192.168.0.201:9997
autoLBFrequency = 30
```

### With Compression Enabled

Reduces bandwidth - useful if forwarding over a slow link:

```ini
[tcpout:default-autolb-group]
server = 127.0.0.1:9997
compressed = true
```

### With SSL/TLS (Production Hardening - Optional)

For a hardened production environment you would encrypt the forwarder-to-indexer channel. Not required for this lab but documented for reference:

```ini
[tcpout:secure-group]
server = 127.0.0.1:9997
sslCertPath = $SPLUNK_HOME\etc\certs\forwarder.pem
sslRootCAPath = $SPLUNK_HOME\etc\certs\ca.pem
sslVerifyServerCert = true
```

---

## Verify outputs.conf is Working

### Via Splunk CLI:

```powershell
cd "C:\Program Files\SplunkUniversalForwarder\bin"

# List configured forward servers
.\splunk.exe list forward-server

# Expected output:
# Active forwards:
#   127.0.0.1:9997
# Configured but inactive forwards:
#   (none)
```

### Check the Forwarder Log for Connection Status:

```powershell
# View last 50 lines of forwarder log
Get-Content "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" -Tail 50

# Filter for output-related messages only
Select-String -Path "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" `
  -Pattern "tcpout|forward|connect" | Select-Object -Last 20
```

**Look for lines like:**

```
Connected to idx=127.0.0.1:9997
```

**Red flags to look for:**

```
Connection refused            → Splunk Enterprise not running or port 9997 not listening
Failed to connect             → Wrong IP/port in outputs.conf
SSL handshake failed          → SSL config mismatch
```

### Confirm Data is Reaching Splunk Enterprise:

```powershell
# Check that Splunk Enterprise is listening on 9997
netstat -ano | findstr :9997

# Expected:
# TCP    0.0.0.0:9997    0.0.0.0:0    LISTENING
```

Then in **Splunk Web UI**, run:

```spl
index=wineventlog
| stats count
```

If `count > 0` - data is flowing successfully.

---

## Apply Changes

**Always restart the forwarder after editing `outputs.conf`:**

```powershell
Restart-Service -Name "SplunkForwarder"
```

---

## Summary - The Full Pipeline

```
Windows Event Logs
  (Security, System, Application)
          │
          │  [inputs.conf defines what to collect]
          ▼
Splunk Universal Forwarder
  (SplunkForwarder service)
          │
          │  [outputs.conf defines where to send]
          │  127.0.0.1:9997
          ▼
Splunk Enterprise Indexer
  (SplunkD service)
  Stores in index: wineventlog
          │
          ▼
Splunk Web UI — http://localhost:8000
  Search, dashboards, alerts
```
