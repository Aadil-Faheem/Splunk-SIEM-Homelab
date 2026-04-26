# Phase 2 - Setting Up Splunk

## Overview

This phase turns the Windows 10 VM into a fully functioning SIEM. You'll install **Splunk Enterprise** as the indexer and search head, then install the **Splunk Universal Forwarder** on the same machine to ship Windows Event Logs into it. By the end, you'll be able to open Splunk's web UI in a browser, see live log data flowing in, and run your first searches.

**Architecture for this lab:**

```
[ Windows 10 VM ]
        │
        ├── Splunk Enterprise (Indexer + Search Head)
        │     Listens on: :8000 (Web UI), :9997 (receiver)
        │
        └── Splunk Universal Forwarder (runs as a service)
              Monitors: Windows Event Logs (Security, System, Application)
              Sends to:  127.0.0.1:9997  (loopback to Splunk Enterprise)
```

> Both Splunk Enterprise and the Universal Forwarder run on the **same Windows 10 VM** in this lab. The forwarder ships logs to `localhost:9997` where Splunk Enterprise is listening.

---

## Phase 2 Checklist

Work through these in order.

**Splunk Enterprise**

- [ ] Download Splunk Enterprise `.msi` installer
- [ ] Install Splunk Enterprise on Windows 10
- [ ] Complete first-time setup (admin credentials, license)
- [ ] Verify Web UI is accessible at `127.0.0.1:8000 or 192.168.0.123:8000`
- [ ] Enable and configure the receiving port (9997) in Splunk UI
- [ ] Create a dedicated index called `wineventlog`

**Universal Forwarder**

- [ ] Download Splunk Universal Forwarder `.msi` installer
- [ ] Install the Universal Forwarder on Windows 10
- [ ] Configure `outputs.conf` to send to `127.0.0.1:9997`
- [ ] Configure `inputs.conf` to monitor Windows Event Logs
- [ ] Restart the SplunkForwarder service
- [ ] Verify data is flowing into Splunk (run a search)

**Validation**

- [ ] Search `index=wineventlog` and confirm events appear
- [ ] Confirm Security, System, and Application logs are ingesting
- [ ] Access Splunk Web UI from Linux Mint Host browser
- [ ] Take screenshots as evidence

---

## Files in This Section

| File                     | Description                                                          |
| ------------------------ | -------------------------------------------------------------------- |
| `splunk-install.md`      | Download and install Splunk Enterprise on Windows 10                 |
| `universal-forwarder.md` | Install and configure the Splunk Universal Forwarder                 |
| `inputs-conf.md`         | Full `inputs.conf` reference — what to monitor and how               |
| `outputs-conf.md`        | `outputs.conf` reference — how to point the forwarder at the indexer |

---

## Downloads Needed

Get both installers before starting. Download them directly on the Windows 10 VM.

| Component                  | URL                                                            | File Type        |
| -------------------------- | -------------------------------------------------------------- | ---------------- |
| Splunk Enterprise          | https://www.splunk.com/en_us/download/splunk-enterprise.html   | `.msi` (Windows) |
| Splunk Universal Forwarder | https://www.splunk.com/en_us/download/universal-forwarder.html | `.msi` (Windows) |

> Although I have a license for Splunk, the free trial license is sufficient for this lab (500MB/day ingest limit). 
> 
> A free Splunk account is required to download.

---

## Splunk Terminology (Quick Reference)

| Term             | What It Means                                                               |
| ---------------- | --------------------------------------------------------------------------- |
| **Indexer**      | The Splunk component that receives, parses, and stores log data             |
| **Search Head**  | The Splunk component that runs searches and displays the UI                 |
| **Forwarder**    | A lightweight agent that monitors log sources and ships data to the indexer |
| **Index**        | A named data store inside Splunk (like a database table)                    |
| **Sourcetype**   | Tells Splunk what kind of data it's looking at so it can parse it correctly |
| **SPL**          | Search Processing Language - Splunk's query language                        |
| **inputs.conf**  | Config file that tells the forwarder *what* to monitor                      |
| **outputs.conf** | Config file that tells the forwarder *where* to send data                   |

---

## Next Phase

Once all checklist items are complete and data is flowing, proceed to **[Phase 3 - Attack Simulation & Log Analysis](../attack-simulation/checklist.md)**.
