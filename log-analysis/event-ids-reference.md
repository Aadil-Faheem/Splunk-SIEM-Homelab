# Windows Event ID Reference

## Overview

Windows Event IDs are the core of what you'll be hunting in Splunk. This cheat sheet covers every Event ID relevant to this lab - organized by category so you can quickly find what you're looking for during an investigation.

**How to search for any Event ID in Splunk:**

```spl
index=wineventlog EventCode=XXXX
| table _time, Account_Name, Message
| sort -_time
```

---

## Authentication & Logon Events

These are the most important events for detecting brute force, credential attacks, and unauthorized access.

| Event ID | Log      | Description                          | Attack Relevance                           |
| -------- | -------- | ------------------------------------ | ------------------------------------------ |
| `4624`   | Security | Successful logon                     | Brute force success, lateral movement      |
| `4625`   | Security | Failed logon attempt                 | Brute force in progress                    |
| `4634`   | Security | Account logoff                       | Session tracking                           |
| `4647`   | Security | User initiated logoff                | Session tracking                           |
| `4648`   | Security | Logon using explicit credentials     | Pass-the-hash, runas, RDP with saved creds |
| `4649`   | Security | Replay attack detected               | Kerberos replay                            |
| `4672`   | Security | Special privileges assigned at logon | Admin/SYSTEM logon                         |
| `4740`   | Security | User account locked out              | Brute force threshold reached              |
| `4776`   | Security | NTLM credential validation           | NTLM-based attacks                         |

### Logon Type Values (for 4624 and 4625)

| Type | Name              | Meaning                        |
| ---- | ----------------- | ------------------------------ |
| `2`  | Interactive       | Local keyboard/console login   |
| `3`  | Network           | SMB, net use, file share       |
| `4`  | Batch             | Scheduled tasks                |
| `5`  | Service           | Service account login          |
| `7`  | Unlock            | Workstation unlock             |
| `8`  | NetworkCleartext  | IIS basic auth, WinRM          |
| `10` | RemoteInteractive | RDP session                    |
| `11` | CachedInteractive | Cached domain credential login |

---

## Process & Execution Events

Critical for detecting malware execution, lateral movement tools, and attacker commands.

| Event ID | Log      | Description                       | Attack Relevance                 |
| -------- | -------- | --------------------------------- | -------------------------------- |
| `4688`   | Security | New process created               | Malware, cmd.exe, powershell.exe |
| `4689`   | Security | Process exited                    | Process lifecycle                |
| `4696`   | Security | Primary token assigned to process | Privilege changes                |

> **Note:** To capture `Process_Command_Line` in Event 4688, you must enable command line auditing:
> 
> ```powershell
> # Enable command line logging in process creation events
> Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" `
>   -Name "ProcessCreationIncludeCmdLine_Enabled" -Value 1 -Type DWord -Force
> ```

---

## Account Management Events

Detects attackers creating backdoor accounts or modifying privileges.

| Event ID | Log      | Description                   | Attack Relevance               |
| -------- | -------- | ----------------------------- | ------------------------------ |
| `4720`   | Security | User account created          | Attacker adds backdoor user    |
| `4722`   | Security | User account enabled          | Activating disabled account    |
| `4723`   | Security | Password change attempted     | Credential manipulation        |
| `4724`   | Security | Password reset attempted      | Privilege escalation           |
| `4725`   | Security | User account disabled         | Denial of service              |
| `4726`   | Security | User account deleted          | Covering tracks                |
| `4728`   | Security | User added to global group    | Privilege escalation           |
| `4732`   | Security | User added to local group     | Adding to Administrators group |
| `4738`   | Security | User account changed          | Account modification           |
| `4756`   | Security | User added to universal group | Privilege escalation           |

---

## Service & Persistence Events

Attackers use services and scheduled tasks to survive reboots.

| Event ID | Log      | Description                     | Attack Relevance                         |
| -------- | -------- | ------------------------------- | ---------------------------------------- |
| `7045`   | System   | New service installed           | Meterpreter persistence, malware install |
| `7036`   | System   | Service state changed           | Service start/stop                       |
| `7040`   | System   | Service start type changed      | Persistence modification                 |
| `4697`   | Security | Service installed in the system | Duplicate of 7045 with more detail       |
| `4698`   | Security | Scheduled task created          | Attacker persistence mechanism           |
| `4699`   | Security | Scheduled task deleted          | Covering tracks                          |
| `4700`   | Security | Scheduled task enabled          | Activating persistence                   |
| `4702`   | Security | Scheduled task updated          | Modifying persistence                    |

---

## Policy & Audit Events

| Event ID | Log      | Description                 | Attack Relevance                     |
| -------- | -------- | --------------------------- | ------------------------------------ |
| `4719`   | Security | System audit policy changed | Attacker disabling logging           |
| `4703`   | Security | Token right adjusted        | Privilege manipulation               |
| `4704`   | Security | User right assigned         | Privilege escalation                 |
| `1102`   | Security | Audit log cleared           | Attacker covering tracks (critical!) |
| `104`    | System   | System log cleared          | Attacker covering tracks             |

> **Event ID 1102 and 104 are critical.** If you see these, someone cleared the event logs - that's a major red flag.

```spl
index=wineventlog (EventCode=1102 OR EventCode=104)
| table _time, host, Message
| sort -_time
```

---

## Network & Firewall Events

| Event ID | Log      | Description                       | Attack Relevance              |
| -------- | -------- | --------------------------------- | ----------------------------- |
| `5156`   | Security | WFP allowed a connection          | Port scan evidence            |
| `5157`   | Security | WFP blocked a connection          | Port scan blocked traffic     |
| `5158`   | Security | WFP permitted a bind              | Listening socket              |
| `5140`   | Security | Network share accessed            | SMB enumeration               |
| `5145`   | Security | Network share object access check | Lateral movement, file access |

---

## Sysmon Event IDs (If Sysmon is Installed)

Sysmon generates far richer data than standard Windows event logs.

| Event ID | Description                                         | Attack Relevance                  |
| -------- | --------------------------------------------------- | --------------------------------- |
| `1`      | Process creation with full command line             | Every process launched            |
| `2`      | Process changed a file creation time (timestomping) | Anti-forensics                    |
| `3`      | Network connection                                  | Reverse shells, C2 traffic        |
| `5`      | Process terminated                                  | Process lifecycle                 |
| `6`      | Driver loaded                                       | Rootkit detection                 |
| `7`      | Image/DLL loaded                                    | DLL injection                     |
| `8`      | CreateRemoteThread                                  | Code injection                    |
| `10`     | Process accessed (OpenProcess)                      | Credential dumping (LSASS access) |
| `11`     | File created                                        | Dropped payloads, malware writes  |
| `12`     | Registry key/value created or deleted               | Persistence via registry          |
| `13`     | Registry value set                                  | Registry-based persistence        |
| `15`     | FileCreateStreamHash                                | ADS (alternate data stream)       |
| `17`     | Pipe created                                        | Named pipe (Metasploit comms)     |
| `22`     | DNS query                                           | C2 domain resolution              |

---

## Quick Reference - This Lab's Key Event IDs

| Phase               | Event ID        | What It Means                    |
| ------------------- | --------------- | -------------------------------- |
| Reconnaissance      | `5156` / `5157` | Port scan connection attempts    |
| Brute Force         | `4625`          | Failed login (each bad password) |
| Brute Force         | `4740`          | Account locked out               |
| Brute Force Success | `4624`          | Successful login after failures  |
| Exploitation        | `4688`          | Malicious process created        |
| Exploitation        | Sysmon `3`      | Reverse shell network connection |
| Persistence         | `7045`          | New service installed            |
| Persistence         | `4698`          | Scheduled task created           |
| Persistence         | `4720`          | Backdoor user account created    |
| Covering Tracks     | `1102` / `104`  | Event logs cleared               |

---

## Splunk Field Names for Common Event IDs

When searching in Splunk, Windows Event Log fields are extracted automatically. These are the most useful field names:

| Splunk Field           | Maps To                                     |
| ---------------------- | ------------------------------------------- |
| `EventCode`            | Windows Event ID number                     |
| `Account_Name`         | The account involved in the event           |
| `Workstation_Name`     | Source machine name                         |
| `Logon_Type`           | Type of logon (2=interactive, 10=RDP, etc.) |
| `Failure_Reason`       | Why a logon failed (4625 only)              |
| `New_Process_Name`     | Full path of new process (4688)             |
| `Creator_Process_Name` | Parent process that launched it (4688)      |
| `Process_Command_Line` | Full command line used (4688, if enabled)   |
| `ServiceName`          | Name of installed service (7045)            |
| `ServiceFileName`      | Path to service executable (7045)           |
| `TargetUserName`       | Account being acted on (4720, 4740)         |
| `SubjectUserName`      | Account that performed the action           |
| `TaskName`             | Scheduled task name (4698)                  |
