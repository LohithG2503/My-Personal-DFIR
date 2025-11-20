# Setup and Standard Operating Procedures (SOP)

This document summarizes the build phases, isolation steps, logging pipeline configuration, and the start/reset SOP for the DFIR lab.

## Phase 1 — Prepare ISOs and Host-Only Network
- Create VirtualBox Host-Only network as documented in `docs/lab-architecture.md`.
- Download ISOs / OVAs:
  - SIFT Workstation OVA (SANS)
  - Windows 11 / Windows 10 LTSC Evaluation ISO
  - Ubuntu Server ISO

## Phase 2 — Build VMs (Internet phase)
Build each VM with NAT networking first to allow updates and package installation.

ANALYST-VM (SIFT)
- Import `sift-22.04-*.ova` via **File > Import Appliance**.
- Add Optical Drive before first boot to allow Guest Additions install.
- Boot, install Guest Additions (Devices > Insert Guest Additions CD Image).
- Update system: `sudo apt update && sudo apt upgrade -y`.
- Take snapshot: `Clean Install (Internet)`.

LOGGER-VM (Ubuntu Server)
- Create VM with NAT, install Ubuntu Server, include OpenSSH during installation.
- Install `syslog-ng` and remove `rsyslog` to avoid conflicts.
- Take snapshot: `Clean Install (Internet)`.

VICTIM-VM (Windows LTSC)
- Create VM with NAT, install Windows LTSC, use an offline/local account if desired.
- Install Guest Additions and Sysinternals Suite.
- Install Sysmon (configure desired manifest) for deep telemetry.
- Take snapshot: `Clean Install (Internet)`.

## Phase 2b — Lab Isolation (Host-Only)
1. Change each VM's network adapter from NAT to **Host-Only Adapter** (vboxnet0).
2. Boot each VM and configure static IPs according to `docs/lab-architecture.md`.

Example: Windows GUI static IP via `ncpa.cpl`.

Example netplan for Ubuntu (Logger): `/etc/netplan/01-netcfg.yaml` (apply with `sudo netplan apply`).

Take a snapshot after lab isolation: `Isolated Lab (Static IP)`.

## Phase 3 — Logging Pipeline

LOGGER-VM (syslog-ng)
- Create `/etc/syslog-ng/conf.d/net-listen.conf` with the following:

```text
source s_network {
    network(ip("0.0.0.0") port(514) transport("tcp"));
    network(ip("0.0.0.0") port(514) transport("udp"));
};

destination d_remote_logs {
    file("/var/log/remote/$HOST/$PROGRAM.log"
        owner("root") group("root") perm(0600) create_dirs(yes)
    );
};

log {
    source(s_network);
    destination(d_remote_logs);
    flags(final);
};
```

- Create the remote log directory: `sudo mkdir -p /var/log/remote`.
- Restart syslog-ng: `sudo systemctl restart syslog-ng`.
- Verify listener: `sudo ss -tulpn | grep 514`.

VICTIM-VM (NXLog)
- Install NXLog Community Edition for Windows on the Victim VM.
- Configure `C:\Program Files\nxlog\conf\nxlog.conf` to forward Sysmon and Windows Event Log data to `192.168.56.103` (Logger) on port 514 (or a configured port/proto).
- Start the `nxlog` service.

Verify pipeline by generating activity on the Victim (e.g., `whoami`) and on the Logger run:

```bash
sudo tail -f /var/log/remote/192.168.56.102/nxlog.log
```

You should see Sysmon Process Create events (Event ID 1) streaming in.

## Standard Operating Procedures (SOP)

Start-Up Routine
- Start VirtualBox.
- Start `LOGGER-VM` first.
- Start `VICTIM-VM` second.
- Start `ANALYST-VM` last.
- Verify logger stream: `sudo tail -f /var/log/remote/192.168.56.102/nxlog.log`.

Reset Routine (restore a clean baseline)
1. Shutdown all VMs.
2. For each VM, open the Snapshots tab in VirtualBox.
3. Right-click the `Isolated Lab (Static IP)` snapshot → **Restore**.
4. Uncheck **Create a snapshot of the current machine state** and click **Restore**.

This ensures every session starts from a mathematically identical baseline.

## Security & Ethics
- Use this lab only for authorized, legal, and educational research.
- Do not connect malware to the internet; keep detonation snapshots isolated.

## Troubleshooting tips
- If syslog-ng isn't receiving logs, confirm firewall rules and that NXLog is sending to the correct IP/port/protocol.
- On Windows, allow inbound ICMP or other required traffic for discovery and testing as needed.
