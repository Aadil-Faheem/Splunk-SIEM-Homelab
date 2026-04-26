# Nmap - Reconnaissance & Detection

## Overview

Nmap is almost always the first tool an attacker uses. Before exploiting anything, they need to know what's running - open ports, services, OS version. This phase simulates that reconnaissance from Kali and shows how to detect it in Splunk.

**What this generates in logs:**

- Windows Firewall logs (if enabled) - inbound connection attempts
- Sysmon Event ID 3 (network connections) - if Sysmon is installed
- Windows Event ID 5156 / 5157 - Windows Filtering Platform connection events

---

## Step 1 - Enable Windows Filtering Platform Audit Logging

By default, Windows 10 does **not** log every inbound network connection. Enable it first.

Run in **PowerShell as Administrator** on Windows 10:

```powershell
# Enable Windows Filtering Platform connection auditing
auditpol /set /subcategory:"Filtering Platform Connection" /success:enable /failure:enable

# Enable Windows Filtering Platform packet drop auditing
auditpol /set /subcategory:"Filtering Platform Packet Drop" /success:enable /failure:enable

# Verify
auditpol /get /subcategory:"Filtering Platform Connection"
```

> ⚠️ Warning: These generate a lot of events. For this lab they're useful during the attack phase. You may want to disable them after to reduce noise:
> `auditpol /set /subcategory:"Filtering Platform Connection" /success:disable /failure:disable`

---

## Step 2 - Run Nmap Scans from Kali

All commands below are run on **Kali Linux**.

### Scan 1 - Basic Host Discovery (Is the target alive?)

```bash
# Ping sweep — confirm target is up
nmap -sn 192.168.0.123

# Expected output:
# Host is up (0.00050s latency).
# MAC Address: XX:XX:XX:XX:XX:XX (Oracle VirtualBox)
```

### Scan 2 - Top 1000 Ports (SYN Scan)

```bash
# Fast SYN scan of most common ports
# -sS = SYN scan (stealthy, doesn't complete TCP handshake)
# -T4 = aggressive timing
sudo nmap -sS -T4 192.168.0.123
```

### Scan 3 - Full Port Scan

```bash
# Scan all 65535 ports
sudo nmap -sS -T4 -p- 192.168.0.123
```

### Scan 4 - Service & Version Detection

```bash
# Identify service versions running on open ports
# -sV = version detection
# -sC = run default scripts
# -O  = OS detection
sudo nmap -sV -sC -O -T4 192.168.0.123
```

### Scan 5 - Aggressive Scan (Makes the Most Noise - Best for Detection Demo)

```bash
# -A = enables OS detection, version detection, script scanning, traceroute
# This is the loudest scan — generates the most log events
sudo nmap -A -T4 192.168.0.123
```

### Scan 6 - Targeted Splunk & RDP Ports

```bash
# Check specifically for Splunk and RDP ports
nmap -p 3389,8000,8088,8089,9997,445,139 192.168.0.123 -sV
```

### Save Scan Output to a File (Good for Your Repo)

```bash
# Save results in all formats (.nmap, .xml, .gnmap)
sudo nmap -A -T4 192.168.0.123 -oA ~/nmap_results/windows10_scan

# View the results
cat ~/nmap_results/windows10_scan.nmap
```

---

## Step 3 - Detect the Scan in Splunk

After running the scans, go to Splunk Web and run the following searches.

### Find Windows Filtering Platform Events (Port Scan Evidence)

```spl
index=wineventlog (EventCode=5156 OR EventCode=5157)
| table _time, EventCode, Application_Name, Direction, Source_Address, Source_Port, Dest_Address, Dest_Port, Protocol
| sort -_time
```

Event ID meanings:

- `5156` - The Windows Filtering Platform permitted a connection
- `5157` - The Windows Filtering Platform blocked a connection

### Detect Port Scan Pattern (Many Ports, Same Source IP)

```spl
index=wineventlog EventCode=5156
| rex field=Source_Address "(?<src_ip>\d+\.\d+\.\d+\.\d+)"
| stats dc(Dest_Port) as unique_ports_hit, count by src_ip, host
| where unique_ports_hit > 20
| sort -unique_ports_hit
| rename src_ip as "Source IP", unique_ports_hit as "Unique Ports Scanned", count as "Total Connections"
```

> This query counts how many **unique destination ports** a single source IP touched. More than 20 unique ports is a very strong indicator of a port scan.

### If Sysmon is Installed - Network Connection Events (Event ID 3)

```spl
index=wineventlog sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventID=3
| table _time, SourceIp, SourcePort, DestinationIp, DestinationPort, Image, Protocol
| where SourceIp="192.168.0.119"
| sort -_time
```

### Find the Kali Machine's Activity Over Time

```spl
index=wineventlog
| rex field=_raw "(?<kali_ip>192\.168\.0\.119)"
| where isnotnull(kali_ip)
| timechart count span=1m
```

---

## Step 4 - Build a Scan Detection Alert

In Splunk, you can save this as a **scheduled alert** that fires when a port scan is detected.

1. Run the port scan detection query above in Splunk Web
2. Click **Save As** → **Alert**
3. Configure:
   - Title: `Port Scan Detected`
   - Alert type: `Scheduled` → every 5 minutes
   - Trigger condition: Number of results is **greater than 0**
   - Trigger actions: `Add to Triggered Alerts`
4. Click **Save**

---

## Expected Nmap Scan Results Against Windows 10

When you run `nmap -A -T4` against a default Windows 10 VM, you'll typically see:

```
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds  Windows 10 microsoft-ds
3389/tcp open  ms-wbt-server Microsoft Terminal Services (RDP)
8000/tcp open  http          Splunk httpd
...
OS: Windows 10 / Windows Server 2019
```

---

## Notes for Your Report:

- The attacker identified **open ports and services** before any exploitation
- Ports `445` and `3389` signal SMB and RDP - both high-value brute force targets
- Port `8000` exposed Splunk's web UI to the network - in a real environment this would be a finding
- The reconnaissance was detectable via **Windows Filtering Platform logs** and **Sysmon network events**
- A port scan detection alert could have flagged this within minutes

---

# 
