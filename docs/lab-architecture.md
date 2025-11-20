# Lab Architecture

## Overview
This document provides the VM breakdown, network schema, and resource allocation used in the My Personal DFIR & Blue Team Lab.

## Virtual Network
- Host-Only Network: `192.168.56.0/24`
- Host/VirtualBox adapter (vboxnet0): `192.168.56.1`

Static IP assignments:
- `192.168.56.101` — Analyst VM (SIFT Workstation)
- `192.168.56.102` — Victim VM (Windows 10/11 LTSC)
- `192.168.56.103` — Logger VM (Ubuntu Server / syslog-ng)

## VM Inventory

| VM Name | OS | Purpose | vCPUs | RAM | Disk | IP |
|---|---:|---|---:|---:|---:|---:|
| ANALYST-VM | SIFT Workstation (Ubuntu-based) | Forensic analysis, Wireshark, Volatility | 2–4 | 4–8 GB | 80 GB | `192.168.56.101` |
| VICTIM-VM | Windows 11 LTSC (Evaluation) | Target machine for analysis (Sysmon, NXLog) | 2 | 2–4 GB | 60 GB | `192.168.56.102` |
| LOGGER-VM | Ubuntu Server 22.04/24.04 | Central log collection (syslog-ng) | 2 | 2–4 GB | 20 GB | `192.168.56.103` |

## Notes on Resource Allocation
- Tailor RAM and vCPUs to available host capacity. The documented settings are intended for a laptop-class host (e.g., ThinkPad X13 with 16 GB RAM).

## Host-Only Network Setup (VirtualBox)
1. In VirtualBox: **File > Host Network Manager**
2. Click **Create** to add a new adapter (e.g., `vboxnet0`).
3. Set IPv4 address to `192.168.56.1` and netmask to `255.255.255.0`.
4. On the DHCP Server tab, uncheck **Enable Server** to use static IPs.

## Netplan example (Ubuntu VMs)
Save as `/etc/netplan/01-netcfg.yaml` (adjust `enp0s3` to the VM's adapter name):

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: no
      dhcp6: no
      addresses:
        - 192.168.56.103/24
      gateway4: 192.168.56.1
      nameservers:
        addresses: [1.1.1.1]
```

Adjust the `addresses:` line for each VM's intended static IP.
