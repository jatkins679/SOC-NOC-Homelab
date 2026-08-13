# Current Homelab Architecture

This diagram reflects the currently implemented lab before the planned Cisco managed-switch, VLAN, and OPNsense phase.

```mermaid
flowchart TB
    WAN[AT&T Fiber / Gateway\n192.168.1.254] --> SW[TRENDnet TEG-S160G\nCurrent unmanaged switch]

    SW --> PVE1[pve01\nProxmox VE\n192.168.1.10]
    SW --> PVE2[pve02\nProxmox VE\n192.168.1.11]
    SW --> PVE3[pve03\nProxmox VE\n192.168.1.12]
    SW --> MGMT[mgmt01\nIndependent management host\n192.168.1.5]
    SW --> DNS[dns01\nPi-hole / DNS\n192.168.1.20]
    SW --> STORE[storage01\nDell PowerEdge T20\nMedia / CIFS backup]

    PVE1 --> WAZUH[wazuh01 - VM 500\nWazuh SIEM\n192.168.1.206]
    PVE1 --> DOCKER[docker - VM 210]
    PVE1 --> SQL[sqlserver2025 - LXC 105]
    PVE1 --> GUAC[apache-guacamole - LXC 200]
    PVE1 --> PIALERT[pialert - LXC 201]

    PVE2 --> KALI[kali01\nSecurity testing\n192.168.1.211]
    PVE2 --> TARGET[target01\nUbuntu monitored endpoint\n192.168.1.238]

    PVE3 --> VULN[vulnscan01\nVulnerability scanning]

    STORE -. CIFS backup .-> PVE1
    STORE -. CIFS backup .-> PVE2
    STORE -. CIFS backup .-> PVE3

    TARGET -- Wazuh agent telemetry --> WAZUH
    KALI -- Controlled test activity --> TARGET
    DNS -. DNS .-> WAZUH
    DNS -. DNS .-> TARGET
```

## Current State

- `pve01`, `pve02`, and `pve03` form the `homelab` Proxmox cluster.
- `mgmt01` remains independent of the cluster for administrative access.
- `dns01` provides DNS using Pi-hole.
- `storage01` provides media/storage services and shared Proxmox backup storage.
- `wazuh01` provides SIEM and endpoint monitoring.
- `target01` is the first actively monitored Linux endpoint.
- `kali01` is used for controlled security-testing activity against lab-owned targets.

## Planned Network Phase

The current unmanaged switch will later be supplemented/replaced in the lab design by a managed Cisco switch. VLAN segmentation and OPNsense will be documented only after those components are actually implemented.
