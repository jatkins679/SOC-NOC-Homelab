# Homelab Operations Scripts

This directory contains small operational utilities used to administer and
troubleshoot the SOC/NOC homelab.

## homelab-health

`homelab-health` is an independent management-plane health check intended to run
from `mgmt01`.

It checks selected infrastructure dependencies from outside the Proxmox
environment and distinguishes between:

- a host that is unreachable;
- a host that is reachable but has an expected service unavailable;
- a healthy host/service combination.

Current checks include:

- AT&T gateway reachability;
- Proxmox VE web/API availability on `pve01` through `pve04`;
- Pi-hole DNS resolution;
- Cisco `sw01` SSH availability;
- `dc01` reachability;
- Uptime Kuma availability;
- Wazuh HTTPS availability;
- `storage01` SMB availability on TCP 445.

Example:

```bash
homelab-health

## mgmt01 Management and Health Commands

The independent `mgmt01` management host provides the operational command
environment for the homelab.

Primary commands:

    homelab-health      Network and service reachability
    proxmox-health      Proxmox cluster and backup health
    linux-health        Linux infrastructure health
    storage-health      Authenticated storage01 / SMB health
    homelab-status      Consolidated health check
    homelab-history     Historical health-check results

The management workstation can be configured or rebuilt with:

    scripts/setup-mgmt01

The setup script installs command links, user-level systemd units, the persistent
SSH-agent configuration, and the hourly health-check timer.

Generated health logs are stored outside the repository under:

    ~/.local/state/homelab/

Secrets, SSH private keys, and the local `storage01` SMB credential file are not
stored in Git.
