# Phase 1 - Setting Up the Network

## Overview

This phase ensures that all three machines (Host, Kali VM, Windows 10 VM) can communicate with each other on the same subnet, and that the Windows 10 VM is prepared to receive Splunk traffic.

**Both VMs use VirtualBox Bridged Adapter**

---

## Phase 1 Checklist

- [ ] Verify VirtualBox Bridged Adapter is set on both VMs
- [ ] Boot both VMs and confirm they receive an IP address
- [ ] Set static IP on Kali Linux VM
- [ ] Set static IP on Windows 10 VM
- [ ] Confirm Kali => Windows 10 ping works
- [ ] Confirm Windows 10 => Kali ping works
- [ ] Open required Splunk ports in Windows Firewall (8000, 9997, 8088, 8089)
- [ ] Document final IP addresses in the topology file
- [ ] Screenshots of pings

---

## 📄 Files in This Section

| File                  | Description                                                    |
| --------------------- | -------------------------------------------------------------- |
| `static-ip-config.md` | Step-by-step commands to set static IPs on Kali and Windows 10 |
| `firewall-rules.md`   | Windows Firewall rules required for Splunk to function         |
| `network-topology.md` | Network diagram and IP address reference table                 |

---

## 🌐 Network Overview

![network_diagram.png](../screenshots/phase1/network_diagram.png)



> Your actual subnet may differ (e.g., 10.0.0.0/24). Run `ip a` on your host to confirm.

---

## ⚠️ Before You Start

- Know your home network subnet. Run `ip a` on your Linux Mint host and look at the IP of your main network interface (usually `eth0` or `wlan0`). Your VMs should be on the same subnet.
- Make sure both VMs are powered off before changing their network adapter settings in VirtualBox.
- VirtualBox → VM Settings → Network → Adapter 1 → Attached to: **Bridged Adapter** → Select your host's active NIC.

---

## Next Phase

Once all checklist items above are complete, proceed to **[Phase 2 - Setting Up Splunk](../splunk-setup/checklist.md)**.
