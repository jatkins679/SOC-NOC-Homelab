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
| **Planned** | Intended for a later phase but not yet documented as operational |
| **Historical / legacy** | Previously present and retained only for rebuild/history context |
| **Verify** | Asset is known, but a current operational detail still needs to be revalidated |

---

# 1. Network and Core Infrastructure

| Asset | Platform / Hardware | Address | Role | Status |
|---|---|---:|---|---|
| AT&T gateway | AT&T residential fiber gateway | `192.168.1.254` | Default gateway / Internet edge | Supporting infrastructure |
| `dns01` | Raspberry Pi 3 Model B+ | `192.168.1.20` | Pi-hole DNS / ad blocking | **Operational** |
| `mgmt01` | Lenovo IdeaPad 330S / Linux | `192.168.1.5` | Independent administration host | **Operational** |
| `storage01` | Dell PowerEdge T20 + attached storage | Verify current address | SMB/file storage and Proxmox backup support | Supporting infrastructure |
| `sw-home01` | Existing home switch | Not applicable / management not documented | Existing home-network switching | Supporting infrastructure |
| `sw01` | Cisco SG350-10 managed switch | Planned | Managed lab switching, VLANs, SNMP, CLI, SPAN | **Planned** |
| `fw01` | OPNsense virtual firewall | Planned | Lab routing, firewalling, inter-VLAN policy | **Planned** |
| `zabbix01` | Linux VM | Planned | NOC monitoring / availability / SNMP | **Planned** |

The current management network remains:

```text
192.168.1.0/24
```

with the default route through:

```text
192.168.1.254
```

The planned VLAN and OPNsense design is documented separately and is not treated
as operational in this inventory.

---

# 2. Proxmox Cluster

The primary virtualization platform is the three-node Proxmox VE cluster:

```text
homelab
```

| Asset | Hardware | Management IP | Role | Status |
|---|---|---:|---|---|
| `pve01` | Acemagic S1, Intel N97, 16 GB RAM, ~512 GB storage | `192.168.1.10` | Proxmox VE cluster node | **Operational** |
| `pve02` | Acemagic S1, Intel N95, 16 GB RAM, ~512 GB storage | `192.168.1.11` | Proxmox VE cluster node | **Operational** |
| `pve03` | Ace Magician T8PLUS, Intel N95, 16 GB RAM, ~512 GB storage | `192.168.1.12` | Proxmox VE cluster node | **Operational** |

Verified cluster state:

```text
Cluster name:     homelab
Nodes:            3
Expected votes:   3
Total votes:      3
Quorum:           2
Quorate:          Yes
Transport:        knet
Secure auth:      on
```

The cluster currently uses local node storage for VM/LXC disks and shared remote
backup storage rather than Ceph.

---

# 3. Core Security and Windows Systems

| Asset | Platform | Address | Role | Monitoring / Integration | Status |
|---|---|---:|---|---|---|
| `wazuh01` | Linux | `192.168.1.206` | Wazuh SIEM / manager / dashboard | Central security monitoring | **Operational** |
| `dc01` | Windows Server 2022 | `192.168.1.30` | Active Directory Domain Services / DNS | Windows Security events monitored through Wazuh | **Operational** |
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
listed as operational until they have been configured and validated.

| Asset | Planned Role | State |
|---|---|---|
| `sw01` | Cisco managed lab switch | Planned / pending deployment |
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
- PiAlert
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
| `dc01` | Yes / Windows telemetry | Not documented as primary source | **Yes** | N/A | Planned |
| `win11-01` | **Yes** | **Yes** | **Yes** | N/A | Planned |
| `target01` | **Yes** | N/A | N/A | **Yes** | Planned |
| `kali01` | Testing role | N/A | N/A | Local logs | Planned |
| `pve01` | Additional monitoring possible | N/A | N/A | Linux/Proxmox logs | Planned |
| `pve02` | Additional monitoring possible | N/A | N/A | Linux/Proxmox logs | Planned |
| `pve03` | Additional monitoring possible | N/A | N/A | Linux/Proxmox logs | Planned |
| `dns01` | Additional monitoring possible | N/A | N/A | Pi-hole/Linux logs | Planned |
| `storage01` | Additional monitoring possible | N/A | Depends on host OS | Host/service logs | Planned |

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
| `192.168.1.20` | `dns01` |
| `192.168.1.30` | `dc01` |
| `192.168.1.151` | `apache-guacamole` |
| `192.168.1.206` | `wazuh01` |
| `192.168.1.211` | `kali01` |
| `192.168.1.238` | `target01` |
| `192.168.1.254` | AT&T gateway |

## DHCP / Observed Addresses

| Observed Address | Asset | Note |
|---:|---|---|
| `192.168.1.165` | `sqlserver2025` | DHCP address observed during validation |
| `192.168.1.167` | `win11-01` | DHCP address observed during Windows/Wazuh validation |

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
├── pve01
├── pve02
└── pve03

Proxmox cluster: homelab
├── pve01
├── pve02
└── pve03

Security / identity
├── wazuh01
├── dc01
└── win11-01

Attack / detection
├── kali01
└── target01

Additional services
├── apache-guacamole
└── sqlserver2025
```

The next major inventory changes are expected when the managed Cisco switch,
VLAN segmentation, OPNsense, and NOC monitoring stack are deployed and
validated.
