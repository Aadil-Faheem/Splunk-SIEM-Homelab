# SIEM Splunk Home Lab

A fully documented cybersecurity home lab simulating real-world attack and detection scenarios using Splunk as the SIEM. Built to demonstrate skills in log ingestion, threat detection, and incident analysis.

**You can use this for Setting up your own lab - It is structured in a tutorial format**

---

## Project Goals

- Deploy and configure Splunk Enterprise as a functional SIEM
- Simulate common attacks (reconnaissance, brute force, exploitation) from a Kali Linux attacker machine
- Ingest and analyze Windows Event Logs to detect malicious activity
- Build Splunk dashboards and detection queries (SPL) to surface threats
- Document the entire process as a reproducible security lab

---

## Lab Environment

| Machine | OS         | Role                   | IP Address    |
| ------- | ---------- | ---------------------- | ------------- |
| Host    | Linux Mint | Hypervisor             | 192.168.0.109 |
| VM1     | Kali Linux | Attacker               | 192.168.0.119 |
| VM2     | Windows 10 | Victim + Splunk Server | 192.168.0.123 |

**Virtualization:** VirtualBox - both VMs use **Bridged Adapter** so they share the same subnet as the host.

---

## Repository Structure

```
splunk-siem-homelab/
│
├── README.md                        # You are here
│
├── network-setup/
│   ├── checklist.md                 # Phase 1 checklist
│   ├── static-ip-config.md          # Static IP configuration for VMs
│   ├── firewall-rules.md            # Windows firewall rules for Splunk
│   └── network-topology.md          # Network diagram
│
├── splunk-setup/
│   ├── checklist.md                 # Phase 2 checklist
│   ├── splunk-install.md            # Splunk install on Windows 10
│   ├── universal-forwarder.md       # Universal Forwarder setup
│   ├── inputs-conf.md               # inputs.conf configuration
│   └── outputs-conf.md              # outputs.conf forwarder-to-indexer
│
├── attack-simulation/
│   ├── checklist.md                 # Phase 3 checklist
│   ├── nmap-scan.md                 # Reconnaissance with Nmap
│   ├── brute-force-hydra.md         # Brute force with Hydra
│   └── metasploit-exploit.md        # Exploitation with Metasploit
│
├── log-analysis/
│   ├── event-ids-reference.md       # Windows Event ID cheat sheet
│   ├── splunk-queries.md            # SPL detection queries
│   └── dashboards/
│       └── failed-logins.xml        # Exported Splunk dashboard
│
├── screenshots/
│   └── (screenshots)
│
└── report/
    └── findings-report.md           # Attack findings
```

---

## Project Phases

### [Phase 1 - Network Setup](./network-setup/checklist.md)

Configure the virtual network, assign static IPs, verify connectivity, and document the topology.

### [Phase 2 - Splunk Setup](./splunk-setup/checklist.md)

Install Splunk Enterprise on Windows 10, configure the Universal Forwarder, and verify log ingestion.

### ./attack-simulation/checklist.md

Run attacks from Kali Linux, detect them in Splunk, and build dashboards.

---

## Tools & Technologies

- **Splunk Enterprise** - SIEM platform
- **Splunk Universal Forwarder** - Log shipping agent
- **Kali Linux** - Attacker OS (Nmap, Hydra, Metasploit)
- **Windows 10** - Victim machine (Windows Event Logging)
- **VirtualBox** - Hypervisor
- **SPL (Search Processing Language)** - Splunk query language

---

## Skills Demonstrated

- SIEM deployment and configuration
- Log ingestion pipeline (Forwarder => Indexer)
- Windows Event Log analysis
- Attack simulation and detection
- Threat hunting with SPL queries
- Dashboard creation in Splunk

---

*This lab is for educational purposes only. All attacks are performed in an isolated virtual environment.*
