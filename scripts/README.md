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
