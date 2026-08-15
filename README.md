# SOC/NOC Homelab

A hands-on infrastructure, networking, and security-monitoring lab built from repurposed hardware, small-form-factor systems, Raspberry Pis, Proxmox VE, Linux, and open-source security tools.

The goal of this project is to document what I actually build, the commands I use, the problems I run into, how I troubleshoot them, and what I learn from the process.

This is not intended to represent a production enterprise network. It is a practical learning environment built from hardware I already had available, with a focus on skills relevant to NOC, SOC, systems administration, networking, and entry-level cybersecurity work.

---

## Current Lab Status

The lab currently includes:

- A four-node Proxmox VE cluster
- Independent Linux management host
- Physical Pi-hole DNS server
- Shared Windows-based storage and backup server
- Kali Linux security-testing system
- Ubuntu target/test system
- Wazuh all-in-one SIEM server
- Wazuh Linux endpoint monitoring
- Windows 11 endpoint monitoring with Sysmon and PowerShell Script Block Logging
- Windows Server 2025 Active Directory Domain Services and AD-integrated DNS
- Domain-joined Windows 11 workstation in `corp.home.arpa`
- Wazuh monitoring of Active Directory account-management events
- File Integrity Monitoring
- Linux Auditd integration
- Apache log monitoring
- Controlled SOC detection exercises
- Shared Proxmox backup storage

The Cisco-managed switching, VLAN, OPNsense, cellular backup Internet, and additional monitoring portions of the design are still planned work. The Cisco SG350-10 has been purchased and is awaiting delivery.

---

## Current Architecture

| System | Role | IP Address |
|---|---|---|
| `pve01` | Proxmox cluster node | `192.168.1.10` |
| `pve02` | Proxmox cluster node | `192.168.1.11` |
| `pve03` | Proxmox cluster node | `192.168.1.12` |
| `pve04` | Dell Precision 5550 Proxmox cluster node | `192.168.1.13` |
| `mgmt01` | Independent Linux management host | `192.168.1.5` |
| `dns01` | Physical Pi-hole DNS server | `192.168.1.20` |
| `storage01` | Windows Server 2025 storage/media/backup server | `192.168.1.208` |
| `wazuh01` | Wazuh SIEM server | `192.168.1.206` |
| `target01` | Ubuntu monitored/test endpoint | `192.168.1.238` |
| `kali01` | Security-testing system | `192.168.1.211` |
| `vulnscan01` | Vulnerability-scanning system | `192.168.1.247` |
| `pialert` | Network-presence monitoring service | DHCP (`192.168.1.225` observed) |
| `dc01` | Windows Server 2025 AD DS / DNS domain controller | `192.168.1.30` |
| `win11-01` | Windows 11 Pro domain-joined monitored endpoint | DHCP (`192.168.1.167` during validation) |

The Proxmox cluster is named:

```text
homelab
```

and currently consists of:

```text
pve01
pve02
pve03
pve04
```

with four-node quorum verified. The cluster requires three votes for quorum.

---

## Project Documentation

### Reference Documentation

#### [Asset Inventory](docs/02-asset-inventory.md)

Provides the current working inventory of physical systems, Proxmox nodes,
virtual machines and containers, infrastructure services, security systems,
storage, monitoring coverage, IP addressing, and planned assets.

The inventory intentionally distinguishes between operational, DHCP-addressed,
supporting, historical, and planned assets so future-state systems are not
presented as already deployed.

---

#### [SOC/NOC Skills Matrix](docs/11-soc-noc-skills-matrix.md)

Maps SOC, NOC, systems, networking, identity, security-monitoring, backup,
troubleshooting, and documentation competencies to specific hands-on lab work
and repository evidence.

The matrix separates demonstrated skills from supporting and planned work so the
portfolio does not claim experience that has not yet been implemented and
validated.

---

#### [SOC Alert Triage and Investigation Playbook](docs/12-soc-alert-triage-playbook.md)

Documents the repeatable analyst workflow used to validate, classify, scope,
prioritize, investigate, remediate, and verify security alerts.

Includes worked examples for SSH brute force, file-integrity changes,
SQL-injection-style requests, and Active Directory account creation, plus a
reusable investigation-notes template.

---

#### [NOC Network and Service Troubleshooting Runbook](docs/13-noc-troubleshooting-runbook.md)

Documents a repeatable infrastructure troubleshooting workflow for host-down,
DNS, Internet connectivity, Proxmox cluster, VM/LXC, storage, disk-capacity,
Linux service, endpoint-monitoring, and Windows domain-connectivity incidents.

Includes incident-priority guidance, dependency mapping, escalation/handoff
notes, service-restoration checks, and reusable incident templates.

---

#### [SOC/NOC Incident and Change Ticket Examples](docs/14-soc-noc-ticket-examples.md)

Provides concise operations-style ticket examples based on actual homelab work,
including Wazuh failures, DNS migration, Proxmox cluster validation, Active
Directory events, detection alerts, storage checks, and escalation notes.

Shows how technical troubleshooting is translated into clear incident,
change-control, validation, handoff, and closure documentation.

---

#### [Homelab Operations and Health Checklist](docs/15-homelab-operations-checklist.md)

Defines daily, weekly, monthly, and periodic operational checks for Proxmox,
Wazuh, Pi-hole DNS, backups, storage, Windows endpoints, Active Directory,
capacity, telemetry health, maintenance readiness, and recovery testing.

Planned Cisco, Zabbix, OPNsense, and Suricata checks remain explicitly separated
until those systems are deployed and validated.

---

### Completed

#### [01 - Proxmox Backup and Recovery](docs/01-proxmox-backup.md)

Documents the pre-rebuild backup process, Proxmox storage validation, VM/LXC inventory, backup verification, and recovery-oriented checks performed before destructive infrastructure changes.

Topics include:

- Proxmox storage inspection
- VM and LXC inventory
- Shared backup storage
- `vzdump`
- Backup verification
- Restore-oriented validation
- Host configuration backup

---

#### [02 - Pi-hole Migration](docs/02-pihole-migration.md)

Documents the migration of DNS service from a Proxmox-hosted Pi-hole container to a dedicated Raspberry Pi.

The physical Pi-hole server is:

```text
dns01
192.168.1.20
```

Topics include:

- Raspberry Pi hostname configuration
- Raspberry Pi OS updates
- Pi-hole installation
- DNS-service validation
- Retaining the existing DNS service address
- Migration planning designed to minimize client-side changes

---

#### [03 - Proxmox Cluster Build and Validation](docs/03-proxmox-cluster-build.md)

Documents the initial three-node build and the current four-node Proxmox cluster:

```text
pve01  192.168.1.10
pve02  192.168.1.11
pve03  192.168.1.12
pve04  192.168.1.13
```

Topics include:

- Cluster membership
- Quorum
- Node identity
- Networking
- Shared storage
- VM and LXC visibility
- Cluster resource validation
- Capacity review
- Decision not to deploy Ceph

---

#### [16 - Proxmox Node Expansion and Workload Rebalancing](docs/16-proxmox-node-expansion.md)

Documents the addition of `pve04`, repository and version standardization,
four-node quorum validation, workload rebalancing, and startup-policy review.

---

#### [06 - Wazuh SIEM Deployment](docs/06-wazuh-siem.md)

Documents deployment of the Wazuh SIEM server as a dedicated Proxmox virtual machine.

Topics include:

- Ubuntu Server deployment
- Proxmox VM configuration
- Pi-hole DNS integration
- QEMU guest agent
- LVM expansion
- Wazuh all-in-one installation
- OpenSearch troubleshooting
- Disk flood-stage diagnosis
- Failed-install recovery
- Clean VM rebuild
- Wazuh service validation
- Linux agent enrollment

This document also records an important deployment failure in which Ubuntu had not allocated the full virtual disk to the root logical volume. OpenSearch crossed its disk flood-stage threshold, placed its security index into read-only mode, and caused the Wazuh installer to fail during internal-user configuration.

Rather than hiding the failed deployment, the troubleshooting process is documented because it demonstrates the diagnostic workflow that led to the root cause and recovery decision.

---

#### [08 - Windows Endpoint Telemetry](docs/08-windows-telemetry.md)

Documents deployment and validation of Windows security telemetry on `win11-01`.

Topics include:

- Windows Wazuh agent deployment and troubleshooting
- Sysmon installation and Event ID 1 process telemetry
- PowerShell Script Block Logging and Event ID 4104
- Wazuh Rule 92027
- Wazuh Rule 91843
- MITRE ATT&CK mapping for PowerShell and registry modification

---

#### [09 - Wazuh Attack Detection Labs](docs/09-attack-detection-labs.md)

Documents controlled SOC exercises performed after the Wazuh deployment was operational.

Completed detections include:

| Lab | Detection | Wazuh Rule | Level | MITRE ATT&CK |
|---|---|---:|---:|---|
| 1 | SSH brute force | `5712` | 10 | `T1110` |
| 2 | File integrity modification | `550` | 7 | `T1565.001` |
| 3 | SQL injection attempt | `31103` | 7 | `T1190` |
| 4 | Root command execution | `80792` | 3 | — |
| 5a | Local account creation | `5902` | 8 | `T1136` |
| 5b | Local account deletion | `5903` | 3 | `T1531` |

These exercises validate the telemetry path:

```text
Controlled activity
      ↓
Linux / application logs
      ↓
Wazuh agent
      ↓
Wazuh manager / indexer
      ↓
Wazuh rule correlation
      ↓
Threat Hunting
      ↓
Analyst validation
```

---

#### [10 - Active Directory Lab](docs/10-active-directory-lab.md)

Documents deployment and validation of the Windows Active Directory environment and its integration with the SOC monitoring stack.

The lab includes:

```text
Domain: corp.home.arpa
NetBIOS: CORP
Domain controller: dc01
Domain-joined endpoint: win11-01
```

Topics include:

- Active Directory Domain Services
- AD-integrated DNS
- Domain and forest validation
- Windows 11 domain join
- Secure-channel verification
- Domain password-policy inspection
- Active Directory user lifecycle administration
- Windows Security Event IDs 4720, 4725, and 4726
- Wazuh detection of domain-account creation
- Wazuh Rule 60109
- MITRE ATT&CK T1098 - Account Manipulation

The account-creation exercise validated an end-to-end detection path from an Active Directory administrative action through the Windows Security log and Wazuh to a SIEM alert.

---

## Detection Work Completed

### SSH Brute-Force Detection

Repeated failed SSH attempts were generated from `kali01` against a nonexistent account on `target01`.

Wazuh generated:

```text
Rule 5712
Level 10
MITRE T1110 - Brute Force
```

The Wazuh alert was independently validated against the original SSH journal on the endpoint.

---

### File Integrity Monitoring

Wazuh FIM was configured to monitor a controlled test directory in real time.

A configuration file was:

1. Created
2. Modified
3. Deleted

Wazuh captured:

- File-size changes
- Modification time
- MD5
- SHA1
- SHA256
- File-content difference

The modification generated:

```text
Rule 550
Level 7
MITRE T1565.001 - Stored Data Manipulation
```

---

### Web / SQL Injection Detection

Apache access logs from `target01` were added to Wazuh log collection.

A controlled request containing SQL-like syntax was generated from `kali01`.

Wazuh generated:

```text
Rule 31103
Level 7
MITRE T1190 - Exploit Public-Facing Application
```

The request returned HTTP 404, but the attempt was still correctly identified from the Apache access log.

---

### Privileged Command Monitoring

Linux Auditd was configured to record `execve` events executed with effective UID 0.

Wazuh generated:

```text
Rule 80792
```

for root-level command activity.

The audit records demonstrated the difference between:

```text
AUID = original authenticated user
EUID = effective execution identity
```

This allowed a root-level command to be attributed back to the original user session.

---

### Local Account Creation and Deletion

A controlled test account was created and later removed.

Account creation generated:

```text
Rule 5902
Level 8
MITRE T1136 - Create Account
```

Account deletion generated:

```text
Rule 5903
Level 3
MITRE T1531 - Account Access Removal
```

---

## Hardware

The lab is intentionally built from modest and repurposed hardware rather than enterprise server equipment.

Current systems include:

- Acemagic mini PCs used as Proxmox nodes
- Ace Magician mini PC
- Lenovo laptop used as an independent management host
- Dell PowerEdge T20 used for storage, media, and backup
- Raspberry Pi used for DNS
- Additional Raspberry Pi hardware
- Existing unmanaged Ethernet switching
- GL.iNet hardware for network experimentation / backup connectivity

A Cisco managed switch is planned for the next networking phase.

---

## Storage and Backup

Proxmox currently uses:

```text
local
local-lvm
t-20-backup
```

`local` is directory-based Proxmox storage used for files such as ISOs and templates.

`local-lvm` is LVM-thin storage used primarily for VM and container disks.

`t-20-backup` is CIFS-backed shared storage hosted by the Dell PowerEdge T20 and is used for Proxmox backups.

Ceph is intentionally not used. The current mini-PC hardware and lab scale do not justify the additional resource and storage overhead.

---

## Current Security Stack

### Wazuh

Used for:

- SIEM
- Endpoint monitoring
- Log collection
- Threat Hunting
- File Integrity Monitoring
- Auditd event analysis
- Authentication monitoring
- MITRE ATT&CK mappings
- Security alert correlation

### Active Directory

The Windows identity lab uses `dc01` as the Active Directory Domain Services and DNS server for:

```text
corp.home.arpa
```

The environment is used to practice:

- Domain administration
- Active Directory DNS
- Windows domain joins
- Password-policy enforcement
- User account lifecycle management
- Windows Security event analysis
- Identity-related SIEM detection
- MITRE ATT&CK mapping for account manipulation

### Kali Linux

Used as a controlled security-testing system for generating activity against systems owned and operated within the lab.

### Ubuntu Target System

`target01` is used as a monitored Linux endpoint and controlled security-testing target.

It currently provides telemetry from:

- SSH
- journald
- Apache
- Wazuh Syscheck / FIM
- Linux Auditd
- Linux account-management events

---

## Planned Network Design

The next major network phase will introduce managed switching and segmentation.

The planned design includes separate logical areas for:

- Management
- Servers
- User systems
- SOC/security systems
- Attack/testing systems
- Guest access

The exact VLAN implementation will be documented after the managed Cisco switch is installed.

I am intentionally not documenting planned VLANs as though they already exist.

---

## Roadmap

### Completed

- [x] Inventory existing lab hardware
- [x] Back up Proxmox workloads
- [x] Validate Proxmox recovery process
- [x] Move Pi-hole to physical Raspberry Pi
- [x] Establish `mgmt01`
- [x] Move storage/media role to `storage01`
- [x] Build initial three-node Proxmox cluster
- [x] Add `pve04` as the fourth cluster node
- [x] Standardize all nodes on Proxmox VE 9.2.10 and kernel 7.0.14-12-pve
- [x] Verify four-node cluster quorum
- [x] Rebalance `wazuh01` and `vulnscan01` across the expanded cluster
- [x] Verify shared backup storage
- [x] Deploy Wazuh SIEM
- [x] Enroll first Linux Wazuh agent
- [x] Enroll Windows Wazuh agent
- [x] Configure Windows telemetry
- [x] Validate Sysmon and PowerShell detections
- [x] Validate SSH brute-force detection
- [x] Validate File Integrity Monitoring
- [x] Validate Apache / SQL-injection-style detection
- [x] Integrate Linux Auditd
- [x] Validate privileged-command monitoring
- [x] Validate local account creation/deletion monitoring
- [x] Deploy Windows Server 2025 Active Directory Domain Services
- [x] Create the `corp.home.arpa` Active Directory domain
- [x] Join `win11-01` to the domain and verify the secure channel
- [x] Validate Active Directory account-creation detection in Wazuh

### In Progress / Planned

- [ ] Add screenshots and diagrams to completed documentation
- [ ] Install Cisco managed switch
- [ ] Design and implement VLANs
- [ ] Deploy OPNsense
- [ ] Validate cellular backup Internet using the GL.iNet GL-A1300 travel router, a compatible USB LTE modem, and a SIM
- [ ] Add additional Linux agents
- [ ] Monitor SSH `authorized_keys`
- [ ] Create custom Wazuh rules
- [ ] Test Wazuh Active Response
- [ ] Tune alerts and reduce false positives
- [ ] Add network monitoring
- [ ] Add Zabbix
- [ ] Expand attack/detection exercises
- [ ] Document final physical and logical network design

---

## Documentation Status

The documentation numbering is intentionally organized around major project phases.

| Document | Topic | Status |
|---|---|---|
| `01-proxmox-backup.md` | Backup and recovery preparation | Complete |
| `02-asset-inventory.md` | Current physical, virtual, service, and monitoring inventory | Active reference |
| `02-pihole-migration.md` | Physical Pi-hole migration | Complete |
| `03-proxmox-cluster-build.md` | Proxmox cluster build and current four-node validation | Complete |
| `04-network-vlans.md` | Managed switching and VLAN design | Planned |
| `05-opnsense.md` | OPNsense routing/firewalling | Planned |
| `06-wazuh-siem.md` | Wazuh deployment | Complete |
| `07-zabbix-monitoring.md` | Infrastructure monitoring | Planned |
| `08-windows-telemetry.md` | Windows endpoint telemetry | Complete |
| `09-attack-detection-labs.md` | SOC detection exercises | Active / documented |
| `10-active-directory-lab.md` | Active Directory and identity monitoring | Complete |
| `11-soc-noc-skills-matrix.md` | SOC/NOC skills matrix | Active reference |
| `12-soc-alert-triage-playbook.md` | SOC alert triage and investigation | Active reference |
| `13-noc-troubleshooting-runbook.md` | NOC troubleshooting runbook | Active reference |
| `14-soc-noc-ticket-examples.md` | Incident and change ticket examples | Active reference |
| `15-homelab-operations-checklist.md` | Homelab operations and health checklist | Active reference |
| `16-proxmox-node-expansion.md` | Fourth-node expansion and workload rebalancing | Complete |

---

## Documentation Approach

This repository is intended to show the actual operational process rather than a polished fictional build.

That means the documentation includes:

- Commands that were actually used
- Validation steps
- Configuration decisions
- Useful failures
- Troubleshooting
- Root-cause analysis
- Recovery decisions
- Security-event evidence
- Skills demonstrated

Where an exact historical command was not preserved, I avoid presenting a reconstructed command as though it came directly from shell history.

Passwords, tokens, generated credentials, and other sensitive values are excluded from the repository.

---

## Why I Built This

Most of this equipment started as a collection of old laptops, Raspberry Pis, mini PCs, Windows systems, switches, and other hardware in my garage behind an AT&T fiber connection.

Rather than replace everything with new enterprise hardware, I wanted to see how far I could take the equipment I already had.

The project gives me a place to:

- Practice Linux and systems administration
- Learn network design and troubleshooting
- Work with virtualization and clustering
- Generate and investigate security telemetry
- Practice SOC-style alert analysis
- Test monitoring tools
- Document problems and their resolution
- Build practical experience that can be shown to prospective employers

The lab will continue to evolve as additional networking, monitoring, and security components are added.
