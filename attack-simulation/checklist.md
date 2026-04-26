# Phase 3 - Attack Simulation & Log Analysis

## Overview

This phase is the payoff of the entire lab. You'll use Kali Linux to run real attacks against the Windows 10 VM and then hunt for that activity inside Splunk. This demonstrates the full blue team workflow: an attack happens, logs are generated, you detect it, and you build detections to catch it automatically.

**The attack chain used in this lab:**

```
[ Kali Linux - Attacker ]
        │
        │  1. Reconnaissance  (Nmap scan)
        │  2. Brute Force     (Hydra against RDP/SMB)
        │  3. Exploitation    (Metasploit - optional)
        │
        ▼
[ Windows 10 VM - Victim ]
        │
        │  Windows Event Logs generated
        │
        ▼
[ Splunk Enterprise ]
        │
        │  You hunt for the events, build SPL queries,
        │  create dashboards and alerts
        ▼
[ Detection & Report ]
```

---

## ✅ Phase 3 Checklist

**Pre-Attack Setup**

- [ ] Confirm Kali → Windows 10 ping works
- [ ] Confirm Splunk is running and `index=wineventlog` has data
- [ ] Enable RDP on Windows 10 VM (needed for brute force target)
- [ ] Create a weak test user on Windows 10 (for brute force demo)
- [ ] Open a live Splunk search before attacking so you can watch events come in

**Attack Execution**

- [ ] Run Nmap reconnaissance scan from Kali
- [ ] Run Hydra brute force against RDP (port 3389)
- [ ] Run Hydra brute force against SMB (port 445)
- [ ] (Optional) Run Metasploit exploit

**Log Analysis & Detection**

- [ ] Find Nmap scan evidence in Splunk
- [ ] Find failed login events (Event ID 4625) in Splunk
- [ ] Find successful login events (Event ID 4624) if brute force succeeds
- [ ] Find account lockout events (Event ID 4740) if lockout policy is set
- [ ] Build SPL detection queries for each attack type

**Documentation**

- [ ] Complete `findings-report.md`

---

## Files in This Section

| File                    | Description                                                  |
| ----------------------- | ------------------------------------------------------------ |
| `nmap-scan.md`          | Nmap reconnaissance from Kali and how to detect it in Splunk |
| `brute-force-hydra.md`  | Brute force attack with Hydra and detecting Event ID 4625    |
| `metasploit-exploit.md` | Optional exploitation phase with Metasploit                  |

**Log Analysis files** (in `../log-analysis/`):

| File                     | Description                                   |
| ------------------------ | --------------------------------------------- |
| `event-ids-reference.md` | Key Windows Event ID cheat sheet for this lab |
| `splunk-queries.md`      | All SPL detection queries used in this lab    |

---

## Pre-Attack Setup on Windows 10

Before running any attacks, set up the environment on the Windows 10 VM.

### Enable RDP (Required for Brute Force Target)

Run in **PowerShell as Administrator** on Windows 10:

```powershell
# Enable Remote Desktop
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' `
  -Name "fDenyTSConnections" -Value 0

# Allow RDP through Windows Firewall
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

# Verify RDP is enabled
Get-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' `
  -Name "fDenyTSConnections"
# Should return: fDenyTSConnections : 0
```

### Create a Weak Test User (Brute Force Target)

```powershell
# Create a test user with a weak/known password
# This user exists ONLY to be the brute force target
New-LocalUser -Name "testuser" -Password (ConvertTo-SecureString "Password123" -AsPlainText -Force) `
  -FullName "Test User" -Description "Lab brute force target"

# Add to Remote Desktop Users group so RDP brute force works
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "testuser"

# Verify user was created
Get-LocalUser -Name "testuser"
```

### Set an Account Lockout Policy (Generates Event ID 4740)

```powershell
# Set lockout after 5 bad attempts — generates rich audit events
net accounts /lockoutthreshold:5 /lockoutduration:30 /lockoutwindow:30

# Verify policy
net accounts
```

### Enable Detailed Audit Policy on Windows 10

Windows 10 doesn't log everything by default. Run these to enable the audit policies that matter:

```powershell
# Enable logon/logoff auditing
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Logoff" /success:enable /failure:enable

# Enable account logon events
auditpol /set /subcategory:"Credential Validation" /success:enable /failure:enable

# Enable process creation (needed to see what ran on the system)
auditpol /set /subcategory:"Process Creation" /success:enable /failure:enable

# Enable account management (lockouts, user creation)
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable

# Verify current policy
auditpol /get /category:*
```

---

## Key Windows Event IDs for This Lab

| Event ID | Description                     | Triggered By                   |
| -------- | ------------------------------- | ------------------------------ |
| `4625`   | Failed logon attempt            | Brute force, bad password      |
| `4624`   | Successful logon                | Successful login               |
| `4740`   | Account locked out              | Too many failed attempts       |
| `4648`   | Logon with explicit credentials | Pass-the-hash, runas           |
| `4688`   | New process created             | Malware, exploit, cmd.exe      |
| `4698`   | Scheduled task created          | Persistence                    |
| `4720`   | User account created            | Attacker creates backdoor user |
| `7045`   | New service installed           | Metasploit persistence         |

> Full reference: `/log-analysis/event-ids-reference.md`

---

## Live Monitoring During Attacks

Before launching any attack from Kali, open these Splunk searches in your browser so you can watch events come in real-time.

**Watch all incoming events:**

```spl
index=wineventlog
| tail 20
```

**Watch for failed logins specifically:**

```spl
index=wineventlog EventCode=4625
| table _time, Account_Name, Workstation_Name, src_ip, Failure_Reason
```

Set the time picker to **"Real-time" → "30 second window"** to see events as they arrive.
