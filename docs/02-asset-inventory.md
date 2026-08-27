# SOC/NOC Homelab Asset Inventory

## Purpose

This document is the working asset inventory for the SOC/NOC homelab.

Its purpose is to provide a single reference for the systems that currently make
up the environment, including:

- Physical infrastructure
- Proxmox cluster nodes
- Virtual machines and LXC containers
- Core network and infrastructure services
- Security-monitoring systems
- Test and attack systems
- Storage and backup resources
- Planned assets that are not yet operational

The inventory intentionally distinguishes between **verified current assets**,
**transitional or DHCP-addressed assets**, and **planned assets**. A planned
system is not presented as though it is already deployed.

---

## Inventory Status Legend

| Status | Meaning |
|---|---|
| **Operational** | Deployed and verified in the current lab |
| **Operational / DHCP** | Deployed, but the observed address may change |
| **Supporting infrastructure** | Active equipment supporting the lab or home network |
| **Staged** | Configured and validated for deployment but not yet carrying its intended live role |
| **Planned** | Intended for a later phase but not yet documented as operational |
| **Historical / legacy** | Previously present and retained only for rebuild/history context |
| **Verify** | Asset is known, but a current operational detail still needs to be revalidated |

---

# 1. Network and Core Infrastructure

| Asset | Platform / Hardware | Address | Role | Status |
|---|---|---:|---|---|
| AT&T gateway | AT&T residential fiber gateway | `192.168.1.254` | Default gateway / Internet edge | Supporting infrastructure |
| `dns01` | Raspberry Pi 3 Model B+ | `192.168.1.20` | Pi-hole DNS / ad blocking | **Operational** |
| `mgmt01` | Lenovo IdeaPad 330S / Linux | `192.168.1.5` | Independent management host; SSH/Ansible administration, health monitoring, and operational tooling | **Operational** |
| `storage01` | Dell PowerEdge T20 / Windows Server 2025 + attached storage | `192.168.1.208` | SMB/file storage and Proxmox backup support | **Operational** |
| `sw-home01` | TRENDnet TEG-S160G, unmanaged, 16-port | Not applicable; unmanaged | Current home-network switching | Supporting infrastructure |
| `sw01` | Cisco SG350-10 managed switch | `192.168.1.21/24` | Managed homelab switching; future VLANs, SNMP, syslog, and SPAN | **Staged** |
| `cellular-wan01` | GL.iNet GL-A1300 travel router + compatible USB LTE modem/SIM | Planned | Backup Internet connectivity and WAN-failover testing | **Planned** |
| `fw01` | OPNsense virtual firewall | Planned | Lab routing, firewalling, inter-VLAN policy | **Planned** |
| `zabbix01` | Linux VM | Planned | NOC monitoring / availability / SNMP | **Planned** |

`mgmt01` provides an administration path that is independent of the Proxmox
cluster. It is used for SSH access, Ansible health checks, service reachability
testing, authenticated SMB validation, and scheduled homelab status checks.
The management tooling is maintained in this repository, while credentials
and SSH private keys remain outside the repository.

The current management network remains:

```text
192.168.1.0/24
```

with the default route through:

```text
192.168.1.254
```

The Cisco switch baseline is now staged and documented separately. VLAN and
OPNsense design remains planned and is not treated as operational in this inventory.

---

# 2. Proxmox Cluster

The primary virtualization platform is the four-node Proxmox VE cluster:

```text
homelab
```

| Asset | Hardware | Management IP | Role | Status |
|---|---|---:|---|---|
| `pve01` | Acemagic S1, Intel N97, 16 GB RAM, ~512 GB storage | `192.168.1.10` | Proxmox VE cluster node | **Operational** |
| `pve02` | Acemagic S1, Intel N95, 16 GB RAM, ~512 GB storage | `192.168.1.11` | Proxmox VE cluster node | **Operational** |
| `pve03` | Ace Magician T8PLUS, Intel N95, 16 GB RAM, ~512 GB storage | `192.168.1.12` | Proxmox VE cluster node | **Operational** |
| `pve04` | Dell Precision 5550, Intel Core i7-10850H, 32 GB RAM, 256 GB NVMe | `192.168.1.13` | Proxmox VE cluster node; USB Gigabit Ethernet | **Operational** |

Verified cluster state:

```text
Cluster name:     homelab
Nodes:            4
Expected votes:   4
Total votes:      4
Quorum:           3
Quorate:          Yes
Transport:        knet
Secure auth:      on
```

The cluster currently uses local node storage for VM/LXC disks and shared remote
backup storage rather than Ceph.

## Current Proxmox Workload Placement

The following placement was verified after the fourth node was added and the
selected workloads were migrated:

| Node | Current workloads |
|---|---|
| `pve01` | `dc01`, `win11-01`, `docker`, `sqlserver2025`, `apache-guacamole`, `pialert` |
| `pve02` | `kali01`, `target01` |
| `pve03` | `nms01`, `wazuh01` |
| `pve04` | `vulnscan01` |

VM disks remain on each node's local storage. The shared `t-20-backup` CIFS
target provides backup-based recovery; the cluster does not use Ceph or claim
automatic workload failover.


---

# 3. Core Security and Windows Systems

| Asset | Platform | Address | Role | Monitoring / Integration | Status |
|---|---|---:|---|---|---|
| `wazuh01` | Linux | `192.168.1.206` | Wazuh SIEM / manager / dashboard | Central security monitoring | **Operational** |
| `dc01` | Windows Server 2025 | `192.168.1.30` | Active Directory Domain Services / DNS | Windows Security events monitored through Wazuh | **Operational** |
| `win11-01` | Windows 11 Pro | DHCP; `192.168.1.167` observed during validation | Domain-joined Windows endpoint | Wazuh agent, Sysmon, PowerShell telemetry | **Operational / DHCP** |

Active Directory domain:

```text
corp.home.arpa
```

NetBIOS domain:

```text
CORP
```

The domain controller is:

```text
dc01.corp.home.arpa
```

`win11-01` has been joined to the domain and its secure channel has been
validated.

---

# 4. Security Testing and Monitored Targets

| Asset | Platform | Address | Role | Monitoring / Use | Status |
|---|---|---:|---|---|---|
| `kali01` | Kali Linux | `192.168.1.211` | Controlled security-testing workstation | Nmap, attack simulation, packet/security testing | **Operational** |
| `target01` | Ubuntu Linux | `192.168.1.238` | Monitored Linux / Apache target | Wazuh, Apache logging, controlled attack detection | **Operational** |
| `vulnscan01` | Linux VM | `192.168.1.247` | Vulnerability-scanning platform | Authorized scanning of lab-owned systems | **Operational** |

`target01` has been used to generate and observe activity including:

- Web requests
- Nmap/NSE probes
- SQL-injection-style test traffic
- Linux authentication activity
- File-integrity changes
- Auditd activity
- Local account lifecycle events

These systems form the core attack/detection path:

```text
kali01
   |
   | controlled test activity
   v
target01 / Windows targets
   |
   | logs and telemetry
   v
wazuh01
   |
   v
Detection / investigation / evidence
```

---

# 5. Additional Active Virtualized Services

| Asset | Type / Platform | Address | Role | Status |
|---|---|---:|---|---|
| `apache-guacamole` | Proxmox LXC | `192.168.1.151` | Browser-based remote access gateway | **Operational** |
| `sqlserver2025` | Ubuntu LXC / Microsoft SQL Server | DHCP; `192.168.1.165` observed | SQL Server learning / application service | **Operational / DHCP** |
| `pialert` | Proxmox LXC | DHCP; `192.168.1.225` observed | Network-presence and device-awareness service | **Operational / DHCP** |

The Guacamole service is used for browser-based remote access to lab systems.

The SQL Server container is retained as an additional server workload within the
virtualization environment and provides another platform for systems,
application, and security-administration practice.

---

# 6. Storage and Backup Assets

| Asset | Type | Role | Status |
|---|---|---|---|
| `storage01` | Dell PowerEdge T20 | File server / storage host | Supporting infrastructure |
| Attached Drobo storage | Direct-attached storage | Bulk file/media storage | Supporting infrastructure |
| `t-20-backup` | Proxmox CIFS storage target | VM/LXC backup destination | **Operational** |
| `local` | Proxmox directory storage | ISOs, templates, local files | **Operational** |
| `local-lvm` | Proxmox LVM-thin storage | VM/LXC virtual disks | **Operational** |

The Proxmox cluster has verified access to the shared `t-20-backup` target.

## Verified `storage01` Platform

| Component | Verified state |
|---|---|
| System | Dell PowerEdge T20 |
| Processor | Intel Xeon E3-1225 v3 at 3.20 GHz |
| Memory | 32 GB |
| BIOS | A20, dated 2019-08-19 |
| TPM | Not present |
| Secure Boot | UEFI-capable but disabled during inspection |

## Verified Attached Storage

| Windows volume | Device / layout | Filesystem | Verified role / state |
|---|---|---|---|
| `C:` | 1 TB-class WDC SATA disk; 0.91 TiB visible | NTFS | Current system disk |
| `E:` | 5 TB-class Seagate Expansion; 2.27 TiB Windows partition | NTFS | `MOVIES`; approximately 0.56 TiB free during inspection |
| Hidden partition on same Seagate disk | 2.27 TiB Apple HFS/HFS+ partition | HFS+ | Preserved; do not initialize, format, or assign a Windows drive letter |
| `F:` | 1.5 TB-class WD Elements; 1.36 TiB visible | NTFS | Secondary external storage; largely free during inspection |
| `G:` | Drobo Gen3 logical volume | NTFS | `T20-NAS`; Proxmox backup target and bulk storage |

The Drobo contains two 6 TB disks with single-disk protection. Drobo Dashboard
reported:

```text
Installed raw capacity:       12 TB (10.91 TiB actual)
Protected usable capacity:   5.34 TB
Logical NTFS volume:         64 TB
Used after backup rebuild:   232.77 GB
Health:                      Good
Firmware:                    4.2.3
Interface:                   USB
```

The 64 TB value is the Drobo's thin-provisioned maximum logical volume, not the
installed raw or usable physical capacity. Physical utilization must therefore
be checked in Drobo Dashboard rather than inferred from Windows or Proxmox.

## Current Backup Policy

The SMB share `ProxmoxBackups` maps to a restricted directory on `G:` and is
configured in Proxmox as `t-20-backup`. Current retention is:

```text
keep-last=7
keep-weekly=4
keep-monthly=3
```

Backup jobs are staggered to avoid four simultaneous write streams to the
USB-attached Drobo:

| Node | Daily schedule |
|---|---:|
| `pve01` | `02:00` |
| `pve02` | `04:00` |
| `pve03` | `05:00` |
| `pve04` | `06:00` |

The complete current backup set contains one archive for each VM/LXC ID:

```text
100 101 105 200 201 210 300 320 400 500
```

The storage design intentionally favors:

```text
Local VM/LXC storage
        +
External/shared backup storage
```

rather than distributed Ceph storage on the small cluster nodes.

---

# 7. Power Infrastructure

| Asset | Role | Planned / Current Use |
|---|---|---|
| APC UPS 600VA + USB | Network / lightweight infrastructure UPS | Gateway, switching, DNS and other lightweight network devices |
| APC Smart-UPS 1000 | Compute/storage UPS | Proxmox nodes and storage equipment |

Exact power distribution remains subject to load and runtime testing as the
physical rebuild continues.

---

# 8. Planned Infrastructure

The following assets or services are part of the target architecture but are not
listed as operational until they have been configured and validated. `sw01` no
longer appears in this table because its baseline configuration has been completed
and it is now tracked as **Staged** in the current network inventory.

| Asset | Planned Role | State |
|---|---|---|
| `cellular-wan01` | Backup Internet via GL.iNet GL-A1300, compatible USB LTE modem, and SIM | Planned / compatibility and failover testing required |
| `fw01` | OPNsense firewall/router | Planned |
| `zabbix01` | Zabbix NOC monitoring server | Planned |
| VLAN 10 | Management | Planned |
| VLAN 20 | Infrastructure / servers | Planned |
| VLAN 30 | User / endpoint systems | Planned |
| VLAN 40 | SOC / monitoring | Planned |
| VLAN 50 | Attack / testing | Planned |
| VLAN 60 | Isolated / vulnerable systems | Planned |
| Suricata telemetry | Network IDS / security telemetry | Planned |
| Additional monitoring probe | Independent availability/network monitoring | Planned |

These items should move into the operational inventory only after their
configuration and validation are documented.

---

# 9. Historical / Rebuild Context

Earlier versions of the homelab included additional VMs, containers, hostnames,
and services that were inventoried during backup and rebuild preparation.

Examples included:

- Older Kali Linux VMs
- CentOS test systems
- OpenVAS test systems
- Pi-hole in an LXC container
- WatchYourLAN
- FreePBX
- Other temporary or experimental guests

These are not automatically considered part of the current operational
inventory.

The purpose of retaining this historical context is to document the migration
from a collection of independent services into a more deliberate SOC/NOC lab
architecture.

---

# 10. Monitoring Coverage

| Asset | Wazuh | Sysmon | Windows Security Logs | Linux Audit / Logs | NOC Monitoring |
|---|---|---|---|---|---|
| `wazuh01` | SIEM server | N/A | N/A | Local/service logs | Zabbix planned |
| `dc01` | **Yes**; agent running automatically | Not installed | **Yes** | N/A | Planned |
| `win11-01` | **Yes** | **Yes** | **Yes** | N/A | Planned |
| `target01` | **Yes** | N/A | N/A | **Yes** | Planned |
| `kali01` | Testing role | N/A | N/A | Local logs | Planned |
| `pve01` | Additional monitoring possible | N/A | N/A | Linux/Proxmox logs | Planned |
| `pve02` | Additional monitoring possible | N/A | N/A | Linux/Proxmox logs | Planned |
| `pve03` | Additional monitoring possible | N/A | N/A | Linux/Proxmox logs | Planned |
| `pve04` | Additional monitoring possible | N/A | N/A | Linux/Proxmox logs | Planned |
| `dns01` | Additional monitoring possible | N/A | N/A | Pi-hole/Linux logs | Planned |
| `storage01` | Not currently enrolled | Not verified | Available locally; not currently forwarded | Windows/service logs | Planned |

This table is intentionally conservative: a system is only marked as monitored
where that monitoring has already been exercised or documented.

---

# 11. Address Summary

## Static / Infrastructure Addresses

| Address | Asset |
|---:|---|
| `192.168.1.5` | `mgmt01` |
| `192.168.1.10` | `pve01` |
| `192.168.1.11` | `pve02` |
| `192.168.1.12` | `pve03` |
| `192.168.1.13` | `pve04` |
| `192.168.1.20` | `dns01` |
| `192.168.1.21` | `sw01` (staged management address) |
| `192.168.1.30` | `dc01` |
| `192.168.1.151` | `apache-guacamole` |
| `192.168.1.206` | `wazuh01` |
| `192.168.1.208` | `storage01` |
| `192.168.1.211` | `kali01` |
| `192.168.1.238` | `target01` |
| `192.168.1.247` | `vulnscan01` |
| `192.168.1.254` | AT&T gateway |

## DHCP / Observed Addresses

| Observed Address | Asset | Note |
|---:|---|---|
| `192.168.1.165` | `sqlserver2025` | DHCP address observed during validation |
| `192.168.1.167` | `win11-01` | DHCP address observed during Windows/Wazuh validation |
| `192.168.1.225` | `pialert` | DHCP address observed during inventory validation |

DHCP-observed addresses should not be treated as permanent reservations unless
they are later explicitly reserved or converted to static assignments.

---

# 12. Asset Management Practices Demonstrated

Maintaining this inventory demonstrates several operational practices relevant to
SOC, NOC, and infrastructure roles:

- Hardware and software asset identification
- Hostname standardization
- IP-address documentation
- Service ownership and role identification
- Physical versus virtual asset classification
- Monitoring coverage tracking
- Configuration-state awareness
- Planned-versus-operational status tracking
- Backup dependency identification
- Infrastructure dependency mapping
- Change documentation
- Avoiding unsupported assumptions about system state

An accurate asset inventory is important because effective monitoring,
incident response, troubleshooting, vulnerability management, and change
management all depend on knowing which systems exist and what they are supposed
to be doing.

---

# 13. Current Inventory Summary

The currently verified lab includes:

```text
Physical / infrastructure
├── AT&T gateway
├── mgmt01
├── dns01
├── storage01
├── sw01 (staged)
├── pve01
├── pve02
├── pve03
└── pve04

Proxmox cluster: homelab
├── pve01
├── pve02
├── pve03
└── pve04

Security / identity
├── wazuh01
├── dc01
└── win11-01

Attack / detection
├── kali01
├── target01
└── vulnscan01

Additional services
├── apache-guacamole
├── pialert
└── sqlserver2025
```

The next major inventory changes are expected when homelab connections are moved
to the staged Cisco switch and validated, followed by VLAN segmentation,
OPNsense, and the NOC monitoring stack.
