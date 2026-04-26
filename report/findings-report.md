# Findings Report - SIEM Splunk Home Lab

**Lab Environment:** Linux Mint (Host) / Kali Linux VM (Attacker) / Windows 10 VM (Victim + Splunk SIEM)

---

## Summary

This report documents the findings from a simulated attack-and-detect exercise conducted in an isolated virtual lab environment. Using Kali Linux as the attacker machine and Windows 10 as the victim, three attack phases were executed: network reconnaissance, credential brute forcing, and system exploitation. All activity was captured and detected using Splunk Enterprise as the SIEM.

---

## Lab Environment

| Component     | Details                                          |
| ------------- | ------------------------------------------------ |
| SIEM Platform | Splunk Enterprise                                |
| Attacker OS   | Kali Linux - IP: `192.168.0.119`                 |
| Victim OS     | Windows 10 - IP: `192.168.0.123`                 |
| Network       | VirtualBox Bridged Adapter - `192.168.0.0/24`    |
| Log Sources   | Windows Security, System, Application Event Logs |
| Optional      | Sysmon with SwiftOnSecurity config               |

---

## Phase 1 - Network Reconnaissance

### Attack Performed

**Tool:** Nmap  
**Command used:**

```bash
sudo nmap -A -T4 192.168.0.123
```

**Open ports discovered:**

| Port | Service | Version                     |
| ---- | ------- | --------------------------- |
| 135  | MSRPC   | Windows RPC                 |
| 445  | SMB     | Windows 10                  |
| 3389 | RDP     | Microsoft Terminal Services |
| 8000 | HTTP    | Splunk Web                  |

### Detection in Splunk

**Query used:**

```spl
index=wineventlog (EventCode=5156 OR EventCode=5157)
| rex field=Source_Address "(?<src_ip>\d+\.\d+\.\d+\.\d+)"
| stats dc(Dest_Port) as unique_ports, count as total_conn by src_ip
| where unique_ports > 20
| sort -unique_ports
```

---

## Phase 2 - Brute Force Attack

### Attack Performed

**Tool:** Hydra  
**Targets:** RDP (port 3389), SMB (port 445)  
**Commands used:**

```bash
hydra -l testuser -P ~/lab-passwords.txt -t 4 -V rdp://192.168.0.123
hydra -l testuser -P ~/lab-passwords.txt -t 4 -V smb://192.168.0.123
```

**Outcome:** Valid credentials found - `testuser:Password123`

### Events Generated

| Event ID | Count | Description                            |
| -------- | ----- | -------------------------------------- |
| `4625`   | 12    | Failed login attempts                  |
| `4740`   | 1     | Account lockout events                 |
| `4624`   | 2     | Successful login (brute force success) |

### Detection in Splunk

**Brute force detection query:**

```spl
index=wineventlog EventCode=4625
| bin _time span=5m
| stats count as failures by _time, Account_Name, Workstation_Name
| where failures >= 5
| sort -_time
```

**Result:** `testuser` accumulated `12` failed login attempts within a 5-minute window from `KALI`, triggering the brute force detection threshold. Account lockout (Event ID 4740) confirmed the lockout policy fired. Successful brute force confirmed via Event ID 4624 at with Logon Type `10` (RDP).

---

## Phase 3 - Exploitation & Post-Exploitation

### Attack Performed

**Tool:** Metasploit Framework + msfvenom  
**Payload:** `windows/x64/meterpreter/reverse_tcp`  
**Listener:** `192.168.0.119:4444`

**Steps executed:**

1. Generated reverse shell payload with msfvenom
2. Delivered payload via Python HTTP server
3. Executed payload on Windows 10 VM
4. Established Meterpreter session
5. Ran post-exploitation enumeration (`whoami`, `net user`, `netstat`)
6. Installed persistence via scheduled task (Event ID 4698)
7. Created backdoor user account (Event ID 4720)

### Events Generated

| Event ID | Description                  | Notes                                    |
| -------- | ---------------------------- | ---------------------------------------- |
| `4624`   | Successful login             | Shell connection established             |
| `4688`   | Process created              | `shell.exe`, `cmd.exe`, `powershell.exe` |
| `4698`   | Scheduled task created       | Meterpreter persistence                  |
| `4720`   | User account created         | Backdoor user `backdoor` added           |
| `4732`   | User added to Administrators | Backdoor user got admin rights           |
| Sysmon 3 | Network connection           | Reverse shell callback to Kali           |

### Detection in Splunk

**Kill chain timeline query:**

```spl
index=wineventlog (EventCode=4625 OR EventCode=4624 OR EventCode=4688 OR EventCode=7045 OR EventCode=4720 OR EventCode=4698 OR EventCode=4740)
| eval Event_Description=case(
    EventCode=4625, "Failed Login",
    EventCode=4624, "Successful Login",
    EventCode=4688, "Process Created",
    EventCode=7045, "Service Installed",
    EventCode=4720, "User Created",
    EventCode=4698, "Scheduled Task",
    EventCode=4740, "Account Lockout",
    true(), "Other")
| table _time, EventCode, Event_Description, Account_Name, host
| sort _time
```

**Findings:**

- `shell.exe` process creation (Event 4688) observed - anomalous parent-child process chain
- Reverse shell network connection (Sysmon Event 3) from `shell.exe` 
- Backdoor user created (Event 4720) and added to Administrators (Event 4732)
- Scheduled task persistence created (Event 4698)

---

## Detections Built

| Detection Name            | Event IDs   | Query Reference          | Type         |
| ------------------------- | ----------- | ------------------------ | ------------ |
| Port Scan Detection       | 5156, 5157  | `splunk-queries.md §2.1` | Saved Search |
| Brute Force - 5+ Failures | 4625        | `splunk-queries.md §3.3` | Alert        |
| Successful Brute Force    | 4624 + 4625 | `splunk-queries.md §3.7` | Alert        |
| Account Lockout           | 4740        | `splunk-queries.md §3.6` | Alert        |
| Suspicious Shell Spawn    | 4688        | `splunk-queries.md §4.2` | Alert        |
| New Service Installed     | 7045        | `splunk-queries.md §5.1` | Alert        |
| New User Created          | 4720        | `splunk-queries.md §5.3` | Alert        |
| Log Clearing              | 1102, 104   | `splunk-queries.md §7`   | Alert        |

---

## Key Takeaways

1. **Reconnaissance is detectable** - Port scanning generates a statistically abnormal number of WFP connection events from a single source IP. A threshold-based alert on unique port count catches this within minutes.

2. **Brute force leaves a loud signature** - Every failed password attempt generates Event ID 4625. The volume spike from Hydra is impossible to miss in Splunk. A 5-failure-in-5-minute window is a practical and effective detection threshold.

3. **Exploitation creates an anomalous process chain** - A reverse shell spawns `cmd.exe` or `powershell.exe` from an unusual parent process. Monitoring parent-child process relationships via Event 4688 is highly effective.

4. **Persistence is extremely noisy** - Service installation (7045), scheduled task creation (4698), and new user accounts (4720) are all rare in normal operation. These make low-false-positive alert candidates.

5. **Sysmon dramatically improves visibility** - Standard Windows Event Logs miss network connections entirely. Sysmon Event 3 was the key to detecting the reverse shell callback.

6. **Audit policy is a prerequisite** - Without enabling detailed audit policies via `auditpol`, many critical events (4688, 4698) are never generated. Audit configuration must come before SIEM deployment.

---

## Recommendations

- Enable detailed audit logging on all endpoints (`auditpol /set ...`)
- Deploy Sysmon with SwiftOnSecurity baseline config across all endpoints
- Disable RDP where not required; restrict to admin subnets where it is
- Enforce account lockout policy (5 failures / 30 min lockout)
- Alert on Event ID 1102 (log clearing) - always suspicious, near-zero false positives
- Implement network segmentation to limit lateral movement blast radius
- Schedule regular Splunk alert reviews to tune thresholds over time

---

*This lab was conducted in an isolated virtual environment for educational purposes only.*
