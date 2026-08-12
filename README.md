# SOC/NOC Homelab

This is a hands-on home laboratory designed to develop, practice, and
document the infrastructure, networking, monitoring, security, and
troubleshooting skills used in **Security Operations Center (SOC)** and
**Network Operations Center (NOC)** environments.

This repository documents the design and construction of my homelab from
the physical hardware layer upward. The goal is to show **what I built,
the commands I executed, the problems I encountered, how I diagnosed
them, and how I resolved them**.

The lab is built primarily around **Proxmox VE**, Linux, Windows,
network segmentation, centralized monitoring, logging, security
telemetry, and controlled attack/detection exercises.

The original idea was to use all the cobbled-together pieces of hardware
(old laptops, Raspberry Pi's, a Windows server, unmanaged and managed
switches) and see what I could accomplish in my garage, behind my home
AT&T fiber connection.

## Project Goals

The primary goals of this project are to gain and demonstrate practical
experience with:

- Linux system administration
- Proxmox VE administration
- KVM virtual machines and LXC containers
- Multi-host virtualization
- Infrastructure backup and recovery
- TCP/IP networking
- DNS and DHCP
- VLANs and network segmentation
- Firewall configuration
- Routing
- Network monitoring
- SNMP
- Centralized logging
- SIEM administration
- Windows and Linux endpoint telemetry
- Vulnerability scanning
- Packet analysis
- Incident detection and investigation
- Controlled offensive-security testing
- Troubleshooting and root-cause analysis
- Infrastructure documentation
- Bash and Linux command-line administration

A major objective is to build an environment that resembles a small
enterprise network rather than a collection of unrelated home servers.

But the over-riding objective is to simply see what I can do, teaching
myself networking and administration skills in the process, and seeing
what works.

## Lab Architecture

The completed environment will use multiple physical systems with
clearly defined infrastructure roles.

                             Internet
                                |
                          Home Gateway
                                |
                         Managed Switch
                                |
                 +--------------+--------------+
                 |                             |
          Home / Production                SOC/NOC Lab
                 |                             |
                 |                       Proxmox Cluster
                 |                             |
                 |                 +-----------+-----------+
                 |                 |           |           |
                 |               pve01       pve02       pve03
                 |                 |           |           |
                 |                 +-----------+-----------+
                 |                             |
                 |                         OPNsense
                 |                             |
                 |           +---------+-------+-------+---------+
                 |           |         |       |       |         |
                 |         MGMT      SERVERS  USERS    SOC      ATTACK
                 |           |         |       |       |         |
                 |        Proxmox    Linux   Windows  Wazuh    Kali
                 |                   Hosts            Zabbix   Test VMs
                 |
           Physical Services
                 |
           +-----+------+----------------+
           |            |                |
         Pi-hole     Storage          Monitoring
          DNS        / Backup           Probe

The design intentionally keeps critical home services separate from
experimental security infrastructure wherever practical. I don't want my
home services going down if I mess something up on the learning side of
my lab.

## Core Technologies

### Virtualization

- Proxmox VE
- KVM/QEMU
- LXC containers
- Proxmox clustering
- Proxmox storage management
- VM/LXC backup and restoration
- Proxmox Datacenter Manager

### Networking

- TCP/IP
- Ethernet
- Linux bridges
- VLANs
- DNS
- DHCP
- Routing
- NAT
- Firewall rules
- OPNsense
- SNMP
- Network troubleshooting

### SOC / Security

- Wazuh
- Windows Event Logs
- Sysmon
- Linux logging
- OpenVAS / vulnerability scanning
- Wireshark
- Nmap
- Kali Linux
- Metasploitable
- MITRE ATT&CK mapping
- Security alert investigation

### NOC / Monitoring

- Zabbix
- Availability monitoring
- SNMP monitoring
- Host resource monitoring
- Network latency monitoring
- Service monitoring
- Centralized alerting
- Performance troubleshooting

### Supporting Infrastructure

- Pi-hole
- Raspberry Pi
- Windows systems
- Linux systems
- SMB/network storage
- Backup storage
- Apache Guacamole
- SSH
- Git/GitHub

## Physical Infrastructure

The lab uses a mixture of dedicated mini PCs, Raspberry Pis, a Dell
server, and networking equipment.

The primary virtualization platform is being consolidated into a
**three-node Proxmox cluster** using similar Intel N95/N97 mini PCs.

Planned primary roles include:

| System                | Role                                         |
|-----------------------|----------------------------------------------|
| `pve01`               | Proxmox VE cluster node                      |
| `pve02`               | Proxmox VE cluster node                      |
| `pve03`               | Proxmox VE cluster node                      |
| `pve-mgmt` / `mgmt01` | Independent management system                |
| `dns01`               | Physical Pi-hole DNS server                  |
| `storage01`           | File storage and backup server               |
| `probe01`             | Independent network/service monitoring probe |

Additional physical and virtual endpoints are used for monitoring,
security telemetry, testing, and controlled attack simulation.

## Why Proxmox?

Proxmox is being used as more than simply a way to run virtual machines.

The environment is intended to provide hands-on experience with:

- Hypervisor installation and administration
- VM and container provisioning
- Virtual networking
- Storage management
- Resource allocation
- Backup and disaster recovery
- Host migration
- Cluster administration
- Infrastructure monitoring
- Troubleshooting
- Service availability
- Centralized management

The Proxmox environment provides the foundation on which the SOC and NOC
infrastructure is being built.

## Repository Structure

    SOC-NOC-Homelab/
    │
    ├── README.md
    │
    ├── docs/
    │   ├── 01-proxmox-backup.md
    │   ├── 02-pihole-migration.md
    │   ├── 03-proxmox-cluster-build.md
    │   ├── 04-network-vlans.md
    │   ├── 05-opnsense.md
    │   ├── 06-wazuh-siem.md
    │   ├── 07-zabbix-monitoring.md
    │   ├── 08-windows-telemetry.md
    │   └── 09-attack-detection-labs.md
    │
    ├── diagrams/
    │
    ├── configs/
    │
    └── scripts/

Each major implementation phase is documented separately.

## Documentation Philosophy

This repository is intentionally more detailed than a typical homelab
overview.

For significant configuration tasks, I document:

1.  **What I was trying to accomplish**
2.  **Why the change was necessary**
3.  **Commands executed**
4.  **Configuration changes**
5.  **Expected results**
6.  **Actual results**
7.  **Errors encountered**
8.  **Troubleshooting performed**
9.  **Final resolution**
10. **Skills demonstrated**

For example:

    qm list

**Purpose:** Enumerate QEMU/KVM virtual machines on a Proxmox host
before performing backup or migration work.

    pct list

**Purpose:** Inventory LXC containers before modifying the
virtualization environment.

    pvesm status

**Purpose:** Verify Proxmox storage availability and confirm backup
storage is accessible.

The goal is not to present a collection of copied commands. The
documentation explains **why each command was used and what information
or result it provided**.

## Project Phases

### Phase 1 — Existing Infrastructure Discovery

- Inventory physical hardware
- Identify network interfaces
- Record existing addressing
- Inventory Proxmox VMs and containers
- Identify storage
- Document existing services
- Verify backup locations

**Status:** In progress / substantially completed

### Phase 2 — Proxmox Backup and Disaster-Recovery Preparation

Before rebuilding existing Proxmox systems, the current environment is
being backed up and validated.

Work includes:

- VM inventory
- LXC inventory
- Storage verification
- Host configuration backup
- VM/LXC backup
- Backup inspection
- Restoration testing
- Verification before destructive changes

Documentation:

[`docs/01-proxmox-backup.md`](docs/01-proxmox-backup.md)

### Phase 3 — Pi-hole Migration

Pi-hole is being moved from a Proxmox LXC container to a dedicated
Raspberry Pi.

This separates a critical DNS service from the virtualization
environment so DNS remains available during Proxmox maintenance or
cluster outages.

Work includes:

- Raspberry Pi configuration
- Linux hostname configuration
- Package updates
- Pi-hole installation
- DNS configuration
- Service verification
- Migration from the previous virtualized Pi-hole instance

Documentation:

[`docs/02-pihole-migration.md`](docs/02-pihole-migration.md)

### Phase 4 — Proxmox Cluster

Three similar mini PCs will form the primary virtualization cluster.

Planned work includes:

- Clean Proxmox installations
- Static management addressing
- Linux bridge configuration
- Cluster creation
- Cluster node joining
- Storage configuration
- VM/LXC migration
- Backup configuration
- Centralized Proxmox management

Documentation:

[`docs/03-proxmox-cluster-build.md`](docs/03-proxmox-cluster-build.md)

### Phase 5 — Network Segmentation

The lab will be divided into logical security zones using VLANs.

Planned segments include:

    Management
    Servers
    User endpoints
    SOC/security infrastructure
    Attack/testing environment

This allows firewall policy, routing, monitoring, and security controls
to be tested in an environment resembling an enterprise network.

Documentation:

[`docs/04-network-vlans.md`](docs/04-network-vlans.md)

### Phase 6 — OPNsense

OPNsense will provide routing and firewalling for the lab environment.

Planned work includes:

- Interface configuration
- VLAN interfaces
- DHCP
- DNS forwarding
- NAT
- Inter-VLAN firewall policy
- Logging
- Traffic analysis

Documentation:

[`docs/05-opnsense.md`](docs/05-opnsense.md)

### Phase 7 — Wazuh SIEM

Wazuh will provide centralized security monitoring.

Planned monitored systems include:

- Windows endpoints
- Linux endpoints
- Infrastructure servers
- Selected physical systems

Planned security exercises include:

- Failed authentication detection
- SSH brute-force activity
- Port scanning
- Windows account changes
- Suspicious process execution
- PowerShell activity
- File-integrity changes
- Vulnerability detection

Documentation:

[`docs/06-wazuh-siem.md`](docs/06-wazuh-siem.md)

### Phase 8 — Zabbix NOC Monitoring

Zabbix will provide infrastructure and network monitoring.

Monitoring targets will include:

- Proxmox hosts
- Linux servers
- Windows systems
- DNS
- Network equipment
- Storage
- CPU utilization
- Memory utilization
- Disk capacity
- Network latency
- Packet loss
- Service availability

Documentation:

[`docs/07-zabbix-monitoring.md`](docs/07-zabbix-monitoring.md)

### Phase 9 — Windows Telemetry

Windows systems will be configured to produce useful security telemetry.

Planned technologies include:

- Windows Event Logging
- Sysmon
- Wazuh agent
- Zabbix agent
- PowerShell logging
- Authentication auditing

Documentation:

[`docs/08-windows-telemetry.md`](docs/08-windows-telemetry.md)

### Phase 10 — Attack and Detection Labs

Controlled attacks will be generated against intentionally vulnerable
lab systems.

Example workflow:

    Kali Linux
         |
         | Nmap / controlled attack
         v
    Target System
         |
         | logs / telemetry
         v
    Wazuh
         |
         | security alert
         v
    Investigation
         |
         v
    Containment / Remediation

Exercises will be documented from both the offensive and defensive
perspectives.

Each investigation will attempt to record:

- Source
- Destination
- Detection
- Relevant logs
- Network evidence
- Security alert
- MITRE ATT&CK technique
- Investigation process
- Remediation
- Verification

Documentation:

[`docs/09-attack-detection-labs.md`](docs/09-attack-detection-labs.md)

## Current Progress

Completed or underway:

- [x] Physical hardware inventory
- [x] Existing Proxmox guest inventory
- [x] Existing Proxmox storage inspection
- [x] Proxmox host configuration backup
- [x] Backup storage verification
- [x] Proxmox backup/recovery investigation
- [x] Dedicated Raspberry Pi selected for DNS
- [x] Pi-hole installed
- [x] Pi-hole DNS service verified
- [ ] Complete Pi-hole migration
- [ ] Build standardized Proxmox nodes
- [ ] Create Proxmox cluster
- [ ] Deploy managed VLAN infrastructure
- [ ] Deploy OPNsense
- [ ] Deploy Wazuh
- [ ] Deploy Zabbix
- [ ] Configure Windows telemetry
- [ ] Build attack/detection exercises

This list will be updated as the lab evolves.

## Troubleshooting Is Part of the Project

Failed commands and configuration errors are not automatically removed
from the documentation.

When they provide useful technical information, troubleshooting attempts
are documented along with the eventual solution.

A NOC or SOC analyst rarely works only with systems that are functioning
correctly.

For that reason, this project emphasizes:

    Observe
       ↓
    Identify symptoms
       ↓
    Collect information
       ↓
    Form a hypothesis
       ↓
    Test
       ↓
    Correct
       ↓
    Verify
       ↓
    Document

The troubleshooting process is considered part of the technical work
rather than something to hide from the final project.

## Security and Sanitization

This repository documents a real home network.

Before publishing configurations or logs, sensitive information will be
removed or replaced, including:

- Passwords
- API keys
- Authentication tokens
- Private keys
- VPN credentials
- Public IP addresses where appropriate
- Personally identifying information
- Sensitive internal configuration
- Device identifiers when unnecessary

Private RFC1918 addresses may also be generalized in public-facing
diagrams and documentation where exposing the actual addressing provides
no technical value.

## Skills Demonstrated

This project is intended to demonstrate practical ability in several
overlapping areas.

### Systems Administration

- Linux administration
- Windows administration
- Package management
- Services
- Networking
- Storage
- SSH
- CLI troubleshooting

### Virtualization

- Proxmox VE
- KVM
- LXC
- Virtual networking
- Cluster administration
- Backup and recovery
- Resource management

### Network Operations

- TCP/IP
- DNS
- VLANs
- Routing
- Firewalls
- SNMP
- Availability monitoring
- Performance monitoring
- Network troubleshooting

### Security Operations

- SIEM
- Endpoint telemetry
- Log analysis
- Alert investigation
- Vulnerability scanning
- Packet analysis
- Threat detection
- Incident-response workflow

### Documentation

- Change documentation
- Command logging
- Network diagrams
- Troubleshooting notes
- Incident reports
- Configuration documentation
- Git/GitHub

## About This Repository

This is an evolving lab.

Configurations, architecture, and documentation will change as the
environment becomes more mature.

The objective is not to claim that a homelab is equivalent to operating
a production enterprise SOC or NOC. The objective is to **practice the
same underlying technologies, administrative tasks, troubleshooting
methods, and analytical processes in a controlled environment and
document that work clearly**.

The result should make it possible for someone reviewing this repository
to see not only the technologies I have studied, but also evidence of
how I have actually used them.
