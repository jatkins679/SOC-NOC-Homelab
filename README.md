# SOC/NOC Homelab

A working homelab for networking, systems administration, virtualization, security monitoring, and troubleshooting.

This project started with a collection of old computers, Raspberry Pis, switches, and other equipment sitting in my garage. I wanted to see what I could build with the hardware I already had.

What started as a few experiments has grown into a four-node Proxmox cluster with managed switching, VLANs, Windows Active Directory, Linux and Windows servers, security monitoring, and a growing collection of network services.

This repository documents what I've built, how I've configured it, problems I've encountered, and how I've worked through them.

It isn't a production enterprise network. It's a place to experiment, break things, fix them, and gain practical experience.

---

## Start Here

If you're new to this repository, these three examples provide a good introduction to the kind of work I've been doing.

### 1. Wazuh Deployment and Troubleshooting

My initial Wazuh installation failed because the Ubuntu VM's root logical volume wasn't using all of the disk space available to it.

I worked through the storage configuration, identified why OpenSearch had placed its index into read-only mode, and rebuilt the installation.

[Wazuh deployment and troubleshooting](docs/06-wazuh-siem.md)

### 2. Testing a Proxmox Backup

A successful backup doesn't necessarily mean the backup can be restored.

I restored my Docker VM into an isolated temporary VM and checked that the operating system, Docker, and Uptime Kuma worked before removing the test instance.

[Proxmox backup and restore validation](docs/21-proxmox-backup-restore-validation.md)

### 3. Troubleshooting Remote Access

While traveling in Europe, I discovered that the local network and my home network were using overlapping IP address ranges.

I investigated the routing problem, used SSH tunneling as a workaround, and tested direct Tailscale connectivity to reach my systems remotely.

The experience also led me to reconsider the addressing scheme for my home network.

[Tailscale remote access and troubleshooting](docs/23-tailscale-remote-access.md)

---

## Lab Architecture

The lab runs primarily on repurposed hardware, including mini PCs, a Dell laptop, a Windows storage server, and Raspberry Pis.

### Virtualization

Four Proxmox VE nodes form the `homelab` cluster:

| Host | IP Address |
|---|---|
| pve01 | 192.168.1.10 |
| pve02 | 192.168.1.11 |
| pve03 | 192.168.1.12 |
| pve04 | 192.168.1.13 |

The cluster hosts Linux and Windows virtual machines, security tools, networking services, monitoring systems, and Docker applications.

### Networking

The network includes:

- Cisco SG350-10 managed switch (`sw01`)
- UniFi US-8 managed switch (`sw02`)
- OPNsense firewall/router (`fw01`)
- Pi-hole DNS server (`dns01`)
- Tailscale subnet routing and remote access
- Segmented VLANs for management, servers, users, security monitoring, testing, and guests

The original flat LAN uses `192.168.1.0/24`. Some services have already been moved to VLANs, while other systems remain on the original network.

I'm also evaluating a change to `192.168.11.0/24` for the default LAN to reduce the likelihood of overlapping subnets when accessing the lab remotely.

[Managed switching and VLAN documentation](docs/04-network-vlans.md)

### Systems and Services

| System | Purpose |
|---|---|
| Windows Server 2025 | Active Directory and DNS |
| Wazuh | SIEM and endpoint monitoring |
| LibreNMS | Network monitoring |
| NetBox | IP address and infrastructure management |
| Grafana / Prometheus | Metrics collection and visualization |
| Uptime Kuma | Availability monitoring |
| Nginx Proxy Manager | Reverse proxy |
| Homarr | Service dashboard |
| Apache Guacamole | Browser-based remote administration |
| Tailscale | Remote access |
| Kali Linux | Security testing |
| Ubuntu target | Detection and vulnerability-testing exercises |

I also operate a separate Linux VPS connected to the lab through Tailscale. It provides an opportunity to experiment with administration and monitoring outside the home network.

[Asset inventory](docs/02-asset-inventory.md)

---

## Security Monitoring and Detection

Wazuh is the main security-monitoring platform.

I've configured Linux and Windows endpoints to generate telemetry and used controlled exercises to test how those events appear in the SIEM.

Exercises include:

- SSH brute-force detection
- File Integrity Monitoring
- Apache log analysis and SQL-injection-style detection
- Linux Auditd and privileged-command monitoring
- Local Linux account creation and deletion
- Windows Sysmon and PowerShell logging
- Active Directory account-management monitoring

The exercises document both the activity that generated the event and how I verified the resulting alert.

[Wazuh detection labs](docs/09-attack-detection-labs.md)

[Windows endpoint telemetry](docs/08-windows-telemetry.md)

[Active Directory lab](docs/10-active-directory-lab.md)

### Vulnerability Scanning

OpenVAS/Greenbone is planned as a separate vulnerability-assessment exercise.

The goal is to scan a lab target, review the findings, investigate selected vulnerabilities, make configuration changes, and rescan to verify the results.

[OpenVAS vulnerability-scanning plan](docs/25-openvas-vulnerability-scanning.md)

---

## Operations, Backups, and Recovery

A substantial part of the project involves maintaining the lab rather than simply installing new applications.

That includes:

- Proxmox cluster and VM administration
- Backup scheduling and restore testing
- Monitoring availability and resource usage
- DNS and connectivity troubleshooting
- Firewall and VLAN configuration
- UPS monitoring and shutdown planning
- Linux service troubleshooting
- System inventory and IP address management

Shared backup storage is hosted on a Windows server, with staggered Proxmox backup jobs and finite retention.

I've also documented why I decided not to use Ceph. Although Ceph can provide distributed, redundant storage across cluster nodes, the additional complexity and resource requirements aren't justified for my current equipment.

[Storage and backup resilience](docs/17-storage-backup-resilience.md)

[Operations checklist](docs/15-homelab-operations-checklist.md)

[Shutdown and startup runbook](docs/19-shutdown-startup-runbook.md)

---

## Other Projects

### ADS-B Aircraft Tracking

I operate an ADS-B receiver that listens to aircraft broadcasts, decodes position and flight information, and displays nearby aircraft on a local map.

The receiver has been migrated from a Raspberry Pi to a Debian VM running on Proxmox.

It also feeds aircraft tracking services.

[ADS-B receiver documentation](docs/22-adsb-flight-tracking.md)

### Remote VPS

I've added a Linux VPS outside my home network, connected through Tailscale.

It runs Docker, Uptime Kuma, and Prometheus Node Exporter. Its metrics are collected by my home Prometheus instance.

This provides an additional place to practice Linux administration, remote connectivity, firewall configuration, and monitoring.

---

## Documentation

The detailed documentation is organized by subject.

| Area | Documentation |
|---|---|
| Hardware and systems | [Asset inventory](docs/02-asset-inventory.md) |
| Proxmox | [Cluster build](docs/03-proxmox-cluster-build.md) |
| Networking | [Switching and VLANs](docs/04-network-vlans.md) |
| Firewall | [OPNsense](docs/05-opnsense.md) |
| Security monitoring | [Wazuh](docs/06-wazuh-siem.md) |
| Detection exercises | [Attack detection labs](docs/09-attack-detection-labs.md) |
| Windows identity | [Active Directory](docs/10-active-directory-lab.md) |
| SOC operations | [Alert triage playbook](docs/12-soc-alert-triage-playbook.md) |
| NOC operations | [Troubleshooting runbook](docs/13-noc-troubleshooting-runbook.md) |
| Operational examples | [Incident and change tickets](docs/14-soc-noc-ticket-examples.md) |
| Recovery | [Backup restore validation](docs/21-proxmox-backup-restore-validation.md) |
| Remote access | [Tailscale](docs/23-tailscale-remote-access.md) |
| Network migration | [VLAN dependency checklist](docs/24-vlan-migration-dependency-checklist.md) |
| Vulnerability scanning | [OpenVAS plan](docs/25-openvas-vulnerability-scanning.md) |

The [SOC/NOC Skills Matrix](docs/11-soc-noc-skills-matrix.md) connects specific skills to documented work in the lab.

---

## Current Work

Some of the work I'm continuing includes:

- Completing the VLAN migration and related service updates
- Reconsidering the default LAN addressing scheme
- Adding OpenVAS vulnerability scanning
- Expanding Wazuh detection exercises
- Improving network and system monitoring
- Testing additional recovery scenarios
- Developing automation and infrastructure-management skills
- Adding screenshots and diagrams to the documentation

This list will change as the lab develops.

---

## Documentation Approach

I try to document the things I actually do.

That includes configurations, commands, testing, mistakes, troubleshooting, and the decisions that follow.

Not every experiment works the first time, and I don't see much value in pretending otherwise. In many cases, figuring out why something failed is more interesting than the original installation.

I also try to distinguish between work I've completed, work that's in progress, and things I'm only considering.

The goal is to keep the repository useful as both a technical reference for myself and a record of the practical work I've done.
