# Splunk Detection Queries (SPL)

## Overview

This file is your complete detection query library for the lab. Every query is written in **SPL (Search Processing Language)** and can be pasted directly into the Splunk Web search bar. Queries are organized by attack phase - from basic validation to advanced correlation.

**How to use these:**

1. Open Splunk Web at `http://192.168.0.123:8000`
2. Go to **Search & Reporting**
3. Paste any query into the search bar
4. Set the time range to cover your attack window (e.g., "Last 60 minutes")
5. Click the magnifying glass to run

---

## Section 1 - Baseline & Validation Queries

Run these first to confirm data is flowing before you start hunting.

### 1.1 - Confirm Data is Ingesting

```spl
index=wineventlog
| stats count by sourcetype
```

### 1.2 - Most Recent Events

```spl
index=wineventlog
| sort -_time
| head 20
| table _time, EventCode, Account_Name, host, sourcetype
```

### 1.3 - Event Count Over Time

```spl
index=wineventlog
| timechart span=5m count by sourcetype
```

### 1.4 - Top Event IDs by Frequency

```spl
index=wineventlog
| stats count by EventCode
| sort -count
| head 20
| rename EventCode as "Event ID", count as "Occurrences"
```

---

## Section 2 - Reconnaissance Detection

### 2.1 - Port Scan Detection (WFP Connection Events)

```spl
index=wineventlog (EventCode=5156 OR EventCode=5157)
| rex field=Source_Address "(?<src_ip>\d+\.\d+\.\d+\.\d+)"
| stats dc(Dest_Port) as unique_ports, count as total_connections by src_ip
| where unique_ports > 20
| sort -unique_ports
| rename src_ip as "Source IP", unique_ports as "Unique Ports Scanned", total_connections as "Total Connections"
```

**Detection logic:** Any source IP touching more than 20 unique destination ports is very likely performing a port scan.

### 2.2 - Connection Spike from Single Source

```spl
index=wineventlog EventCode=5156
| bin _time span=1m
| stats count as connections by _time, Source_Address
| where connections > 50
| sort -_time
| rename Source_Address as "Source IP", connections as "Connections per Minute"
```

### 2.3 - SMB Enumeration (Share Access)

```spl
index=wineventlog EventCode=5140
| table _time, Account_Name, IpAddress, ShareName, AccessMask
| sort -_time
| rename Account_Name as "Account", IpAddress as "Source IP", ShareName as "Share Accessed"
```

---

## Section 3 - Brute Force Detection

### 3.1 - All Failed Login Attempts

```spl
index=wineventlog EventCode=4625
| table _time, Account_Name, Workstation_Name, Logon_Type, Failure_Reason
| sort -_time
```

### 3.2 - Failed Logins by Source - Rank Attackers

```spl
index=wineventlog EventCode=4625
| stats count as failed_attempts by Account_Name, Workstation_Name
| sort -failed_attempts
| rename Account_Name as "Target Account", Workstation_Name as "Source", failed_attempts as "Failed Attempts"
```

### 3.3 - Brute Force Alert - 5+ Failures in 5 Minutes (Core Detection Rule)

```spl
index=wineventlog EventCode=4625
| bin _time span=5m
| stats count as failures by _time, Account_Name, Workstation_Name
| where failures >= 5
| sort -_time
| rename _time as "Window Start", Account_Name as "Target Account", Workstation_Name as "Source", failures as "Failed Logins in 5min"
```

### 3.4 - RDP Brute Force Only (Logon Type 10)

```spl
index=wineventlog EventCode=4625 Logon_Type=10
| stats count as rdp_failures by Account_Name, Workstation_Name
| sort -rdp_failures
| rename Account_Name as "Target Account", Workstation_Name as "Source", rdp_failures as "RDP Failures"
```

### 3.5 - SMB Brute Force Only (Logon Type 3)

```spl
index=wineventlog EventCode=4625 Logon_Type=3
| stats count as smb_failures by Account_Name, Workstation_Name
| sort -smb_failures
| rename Account_Name as "Target Account", Workstation_Name as "Source", smb_failures as "SMB Failures"
```

### 3.6 - Account Lockouts

```spl
index=wineventlog EventCode=4740
| table _time, TargetUserName, SubjectUserName, SubjectDomainName
| sort -_time
| rename TargetUserName as "Locked Account", SubjectUserName as "Triggered By"
```

### 3.7 - Successful Login After Multiple Failures (Brute Force Success)

```spl
index=wineventlog (EventCode=4625 OR EventCode=4624)
| eval event_type=if(EventCode=4624, "SUCCESS", "FAILURE")
| stats
    count(eval(event_type="FAILURE")) as total_failures,
    count(eval(event_type="SUCCESS")) as total_successes
    by Account_Name
| where total_failures > 3 AND total_successes > 0
| sort -total_failures
| rename Account_Name as "Account", total_failures as "Prior Failures", total_successes as "Successful Logins"
```

### 3.8 - Brute Force Timeline Chart

```spl
index=wineventlog EventCode=4625
| timechart span=1m count as "Failed Login Attempts"
```

---

## Section 4 - Exploitation Detection

### 4.1 - All New Processes Created

```spl
index=wineventlog EventCode=4688
| table _time, Account_Name, New_Process_Name, Creator_Process_Name, Process_Command_Line
| sort -_time
| head 50
```

### 4.2 - Suspicious Shell Spawns (cmd.exe / powershell.exe from Unusual Parent)

```spl
index=wineventlog EventCode=4688
| where (New_Process_Name LIKE "%\\cmd.exe" OR New_Process_Name LIKE "%\\powershell.exe")
  AND NOT (
    Creator_Process_Name LIKE "%\\explorer.exe"
    OR Creator_Process_Name LIKE "%\\cmd.exe"
    OR Creator_Process_Name LIKE "%\\powershell.exe"
    OR Creator_Process_Name LIKE "%\\WindowsTerminal.exe"
  )
| table _time, Account_Name, Creator_Process_Name, New_Process_Name, Process_Command_Line
| sort -_time
```

**Detection logic:** `cmd.exe` or `powershell.exe` spawned by anything other than `explorer.exe` (the normal desktop shell) or another shell is suspicious. This catches shells opened by malware or Meterpreter.

### 4.3 - Suspicious Recon Commands Run

```spl
index=wineventlog EventCode=4688
| where Process_Command_Line LIKE "%whoami%"
  OR Process_Command_Line LIKE "%net user%"
  OR Process_Command_Line LIKE "%net localgroup%"
  OR Process_Command_Line LIKE "%ipconfig%"
  OR Process_Command_Line LIKE "%tasklist%"
  OR Process_Command_Line LIKE "%netstat%"
  OR Process_Command_Line LIKE "%systeminfo%"
| table _time, Account_Name, New_Process_Name, Process_Command_Line
| sort -_time
```

### 4.4 - Reverse Shell Network Connections (Sysmon Event ID 3)

```spl
index=wineventlog sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventID=3
| where DestinationIp="192.168.0.119"
| table _time, Image, User, SourceIp, SourcePort, DestinationIp, DestinationPort, Protocol
| sort -_time
```

### 4.5 - Unusual Outbound Connection on Non-Standard Port

```spl
index=wineventlog sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventID=3
| where NOT (DestinationPort=80 OR DestinationPort=443 OR DestinationPort=53)
  AND NOT (DestinationIp LIKE "127.%")
| table _time, Image, SourceIp, DestinationIp, DestinationPort, Protocol
| sort -_time
```

---

## Section 5 - Persistence Detection

### 5.1 - New Service Installed (Event ID 7045)

```spl
index=wineventlog EventCode=7045
| table _time, ServiceName, ServiceFileName, ServiceType, ServiceStartType, AccountName
| sort -_time
| rename ServiceName as "Service Name", ServiceFileName as "Executable Path"
```

### 5.2 - Scheduled Task Created (Event ID 4698)

```spl
index=wineventlog EventCode=4698
| table _time, SubjectUserName, TaskName, TaskContent
| sort -_time
| rename SubjectUserName as "Created By", TaskName as "Task Name"
```

### 5.3 - New User Account Created (Event ID 4720)

```spl
index=wineventlog EventCode=4720
| table _time, TargetUserName, SubjectUserName, SubjectDomainName
| sort -_time
| rename TargetUserName as "New Account", SubjectUserName as "Created By"
```

### 5.4 - User Added to Administrators Group (Event ID 4732)

```spl
index=wineventlog EventCode=4732
| where TargetUserName="Administrators" OR GroupName="Administrators"
| table _time, MemberName, SubjectUserName, TargetUserName
| sort -_time
| rename MemberName as "User Added", SubjectUserName as "Added By", TargetUserName as "Group"
```

---

## Section 6 - Advanced Correlation

### 6.1 - Full Attack Kill Chain Timeline

```spl
index=wineventlog (EventCode=4625 OR EventCode=4624 OR EventCode=4688 OR EventCode=7045 OR EventCode=4720 OR EventCode=4698 OR EventCode=4740 OR EventCode=4732)
| eval Event_Description=case(
    EventCode=4625, "Failed Login",
    EventCode=4624, "Successful Login",
    EventCode=4688, "Process Created",
    EventCode=7045, "Service Installed",
    EventCode=4720, "User Account Created",
    EventCode=4698, "Scheduled Task Created",
    EventCode=4740, "Account Locked Out",
    EventCode=4732, "⬆️ Added to Admin Group",
    true(), "Other"
  )
| table _time, EventCode, Event_Description, Account_Name, New_Process_Name, host
| sort _time
```

### 6.2 - Account Compromise Score

```spl
index=wineventlog
| eval score=case(
    EventCode=4625, 1,
    EventCode=4740, 5,
    EventCode=4624 AND Logon_Type=10, 2,
    EventCode=4688, 2,
    EventCode=4720, 10,
    EventCode=7045, 10,
    EventCode=4698, 8,
    true(), 0
  )
| stats sum(score) as risk_score by Account_Name
| where risk_score > 10
| sort -risk_score
| rename Account_Name as "Account", risk_score as "Risk Score"
```

### 6.3 - Activity from a Specific IP (Attacker-Centric View)

```spl
index=wineventlog
| eval kali_involved=if(
    Workstation_Name="KALI" OR IpAddress="192.168.0.119" OR Source_Address="192.168.0.119",
    "YES", "NO")
| where kali_involved="YES"
| table _time, EventCode, Account_Name, Workstation_Name, IpAddress, Message
| sort _time
```

---

## Section 7 - Log Clearing Detection (Covering Tracks)

```spl
index=wineventlog (EventCode=1102 OR EventCode=104)
| table _time, host, EventCode, SubjectUserName, Message
| sort -_time
```

> This is one of the most important queries to always have saved. Log clearing is rare in normal operations and almost always indicates malicious intent.

---

## Saving Queries as Alerts in Splunk

For any query you want to run automatically:

1. Run the query in Splunk Web
2. Click **Save As** → **Alert**
3. Set:
   - **Alert type:** Scheduled (every 5 or 15 minutes) or Real-time
   - **Trigger condition:** Number of results is greater than `0`
   - **Actions:** Add to Triggered Alerts, or send an email if configured
4. Save

**Recommended alerts to save:**

- `3.3` - Brute Force Detection (5+ failures in 5 min)
- `5.1` - New Service Installed
- `5.3` - New User Created
- `7.0` - Log Clearing Detected
