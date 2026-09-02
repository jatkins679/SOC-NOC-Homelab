# SOC/NOC Skills Matrix

## Purpose

This document maps job-relevant SOC, NOC, systems, networking, and security
skills to work that has actually been implemented and validated in this
homelab.

It is intended to answer a practical hiring-manager question:

> What can I point to in this repository that demonstrates the skill?

The matrix distinguishes between:

- **Demonstrated** — implemented and validated in the current lab
- **Supporting** — used as part of a demonstrated workflow
- **In progress / planned** — part of the target architecture but not yet claimed as completed experience

The goal is evidence-backed skills, not a list of product names.

---

# 1. Core Skills Matrix

| Skill / Competency | Status | Hands-on Implementation | Validation / Evidence | Repository Documentation |
|---|---|---|---|---|
| Proxmox VE administration | **Demonstrated** | Built and administer a four-node Proxmox VE cluster; manage VMs, LXC containers, storage, networking, and resources | Four nodes online; cluster membership, workloads, and resources validated | [`03-proxmox-cluster-build.md`](03-proxmox-cluster-build.md), [`16-proxmox-node-expansion.md`](16-proxmox-node-expansion.md) |
| Multi-node virtualization | **Demonstrated** | `pve01`, `pve02`, `pve03`, and `pve04` operate as cluster `homelab` | `pvecm status`, `pvecm nodes`, `pvesh get /nodes` | [`03-proxmox-cluster-build.md`](03-proxmox-cluster-build.md), [`16-proxmox-node-expansion.md`](16-proxmox-node-expansion.md) |
| Cluster quorum | **Demonstrated** | Four voting Proxmox nodes using Corosync/knet | 4 votes, quorum 3, `Quorate: Yes` | [`03-proxmox-cluster-build.md`](03-proxmox-cluster-build.md), [`16-proxmox-node-expansion.md`](16-proxmox-node-expansion.md) |
| VM / LXC administration | **Demonstrated** | Inventory, backup, restore, start, inspect, and manage QEMU VMs and LXC containers | `qm`, `pct`, QEMU guest-agent and restore workflows | [`01-proxmox-backup.md`](01-proxmox-backup.md), [`03-proxmox-cluster-build.md`](03-proxmox-cluster-build.md) |
| Backup and disaster recovery | **Demonstrated** | Created Proxmox guest backups and host-configuration backup; inspected and restored workloads | Backup existence, configuration extraction, LXC restoration and post-restore checks | [`01-proxmox-backup.md`](01-proxmox-backup.md) |
| Shared storage / CIFS | **Demonstrated** | Integrated and recovered `t-20-backup` as remote Proxmox backup storage | `pvesm status`, `pvesm list`, stale-mount recovery, backup access and restore workflow | [`01-proxmox-backup.md`](01-proxmox-backup.md), [`02-asset-inventory.md`](02-asset-inventory.md), [`17-storage-backup-resilience.md`](17-storage-backup-resilience.md) |
| Linux administration | **Demonstrated** | Ubuntu, Raspberry Pi OS, Proxmox/Debian administration; services, packages, filesystems, networking, logging | Repeated CLI-based build, validation, troubleshooting, and recovery work | Multiple documents |
| Linux networking | **Demonstrated** | Interface inspection, IP addressing, routing, bridges, DNS client configuration | `ip -br addr`, `ip route`, `/etc/network/interfaces`, resolver checks | [`01-proxmox-backup.md`](01-proxmox-backup.md), [`03-proxmox-cluster-build.md`](03-proxmox-cluster-build.md) |
| DNS administration | **Demonstrated** | Migrated Pi-hole DNS to dedicated Raspberry Pi; configured and validated AD-integrated DNS | Port 53/service checks, client resolution, domain discovery, DNS locator records | [`02-pihole-migration.md`](02-pihole-migration.md), [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| Infrastructure dependency reduction | **Demonstrated** | Removed household DNS dependency from the Proxmox virtualization layer | Pi-hole moved from LXC to dedicated `dns01` at `192.168.1.20` | [`02-pihole-migration.md`](02-pihole-migration.md) |
| Asset inventory / configuration management | **Demonstrated** | Maintain current inventory of physical, virtual, service, storage, IP, and monitoring assets | Operational vs DHCP vs planned vs historical state documented | [`02-asset-inventory.md`](02-asset-inventory.md) |
| Technical documentation | **Demonstrated** | Document commands, expected results, actual results, failures, diagnosis, fixes, and validation | Public repository structure and implementation records | Entire repository |
| Change / recovery discipline | **Demonstrated** | Baseline before changes, verify backup destination, validate recovery before destructive rebuild | Backup and restore criteria documented and exercised | [`01-proxmox-backup.md`](01-proxmox-backup.md) |

---

# 2. SOC / Security Operations Skills

| Skill / Competency | Status | Hands-on Implementation | Validation / Evidence | Repository Documentation |
|---|---|---|---|---|
| SIEM deployment | **Demonstrated** | Deployed Wazuh all-in-one SIEM on `wazuh01` | Indexer, manager, Filebeat, dashboard all validated active | [`06-wazuh-siem.md`](06-wazuh-siem.md) |
| SIEM endpoint enrollment | **Demonstrated** | Enrolled Linux and Windows systems into Wazuh | Active agent validation and telemetry observed in dashboard | [`06-wazuh-siem.md`](06-wazuh-siem.md), [`08-windows-telemetry.md`](08-windows-telemetry.md) |
| Centralized log analysis | **Demonstrated** | Collected endpoint, authentication, application, Auditd, file-integrity, and Windows security telemetry | Controlled events produced searchable Wazuh alerts | [`06-wazuh-siem.md`](06-wazuh-siem.md), [`09-attack-detection-labs.md`](09-attack-detection-labs.md) |
| Alert triage and validation | **Demonstrated** | Compared Wazuh detections with source endpoint/application logs | SSH, Apache, FIM, Auditd, account and Windows events validated against source activity | [`09-attack-detection-labs.md`](09-attack-detection-labs.md), [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| Threat hunting | **Demonstrated** | Used Wazuh event data to locate and inspect controlled security activity | Rule IDs, event details, agents, timestamps, users and source logs correlated | [`06-wazuh-siem.md`](06-wazuh-siem.md), [`09-attack-detection-labs.md`](09-attack-detection-labs.md) |
| MITRE ATT&CK mapping | **Demonstrated** | Mapped controlled detections to ATT&CK techniques | Examples include T1110, T1565.001, T1190, T1136, T1531, T1098 | [`09-attack-detection-labs.md`](09-attack-detection-labs.md), [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| File Integrity Monitoring | **Demonstrated** | Configured Wazuh realtime monitoring with content-change reporting | Rule `550`; size, mtime, MD5, SHA1, SHA256 and content diff captured | [`09-attack-detection-labs.md`](09-attack-detection-labs.md) |
| Linux Auditd monitoring | **Demonstrated** | Collected privileged-command and process execution activity through Auditd/Wazuh | Rule `80792` exercised in controlled labs | [`09-attack-detection-labs.md`](09-attack-detection-labs.md) |
| Authentication monitoring | **Demonstrated** | Generated repeated failed SSH authentication attempts | Wazuh rule `5712`, level 10, MITRE T1110 | [`09-attack-detection-labs.md`](09-attack-detection-labs.md) |
| Web/application log monitoring | **Demonstrated** | Collected Apache access logs and generated SQL-injection-style request | Wazuh rule `31103`, level 7, MITRE T1190 | [`09-attack-detection-labs.md`](09-attack-detection-labs.md) |
| Linux account lifecycle monitoring | **Demonstrated** | Created and deleted test local accounts | Wazuh rules `5902` and `5903` | [`09-attack-detection-labs.md`](09-attack-detection-labs.md) |
| Windows endpoint telemetry | **Demonstrated** | Configured Wazuh agent, Sysmon, Windows event collection, and PowerShell telemetry on `win11-01` | Windows security/process activity reached Wazuh | [`08-windows-telemetry.md`](08-windows-telemetry.md) |
| Sysmon | **Demonstrated** | Installed and exercised Sysmon on Windows 11 endpoint | Process-creation telemetry observed in Wazuh | [`08-windows-telemetry.md`](08-windows-telemetry.md) |
| PowerShell logging | **Demonstrated** | Enabled PowerShell Script Block Logging and generated test activity | PowerShell telemetry collected and validated | [`08-windows-telemetry.md`](08-windows-telemetry.md) |
| Identity monitoring | **Demonstrated** | Generated Active Directory user lifecycle events on `dc01` | Event IDs 4720, 4725, 4726; Wazuh account-creation alert | [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| Detection-path validation | **Demonstrated** | Followed administrative/attack action through source log, agent, SIEM rule and analyst review | Multiple end-to-end exercises documented | [`09-attack-detection-labs.md`](09-attack-detection-labs.md), [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| Controlled offensive testing | **Supporting** | Used Kali Linux, Nmap, curl and controlled attack traffic against owned lab targets | Activity generated known telemetry without affecting production systems | [`09-attack-detection-labs.md`](09-attack-detection-labs.md) |

---

# 3. Detection Evidence Matrix

| Exercise | Source / Target | Detection | Rule / Event | Severity / Level | MITRE ATT&CK |
|---|---|---|---|---:|---|
| SSH brute force | `kali01` → `target01` | Repeated failed SSH authentication | Wazuh `5712` | 10 | `T1110` Brute Force |
| File modification | `target01` | Realtime FIM integrity change | Wazuh `550` | 7 | `T1565.001` Stored Data Manipulation |
| SQL-injection-style request | `kali01` → Apache on `target01` | Suspicious SQL syntax in access log | Wazuh `31103` | 7 | `T1190` Exploit Public-Facing Application |
| Privileged Linux command | `target01` | Auditd privileged-command activity | Wazuh `80792` | 3 | Context-dependent |
| Local account creation | Linux target | New local account | Wazuh `5902` | 8 | `T1136` Create Account |
| Local account deletion | Linux target | Local account removed | Wazuh `5903` | 3 | `T1531` Account Access Removal |
| AD account creation | `dc01` | Domain user created | Windows `4720` / Wazuh `60109` | 8 | `T1098` Account Manipulation |
| AD account disable | `dc01` | Domain user disabled | Windows `4725` | — | Identity lifecycle monitoring |
| AD account deletion | `dc01` | Domain user deleted | Windows `4726` | — | Identity lifecycle monitoring |

Existing sanitized screenshots are stored under:

```text
evidence/wazuh/
```

Current documented Linux/Wazuh evidence includes:

```text
01-ssh-bruteforce-rule-5712.png
02-fim-rule-550.png
03-sql-injection-rule-31103.png
04-audit-sudo-rule-80792.png
05-audit-id-rule-80792.png
06-user-created-rule-5902.png
07-user-deleted-rule-5903.png
```

---

# 4. Windows / Identity Skills

| Skill / Competency | Status | Hands-on Implementation | Validation | Documentation |
|---|---|---|---|---|
| Active Directory Domain Services | **Demonstrated** | Deployed `dc01` as domain controller for `corp.home.arpa` | Domain/forest/DC queries validated | [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| AD-integrated DNS | **Demonstrated** | Domain DNS supports AD discovery and Windows domain operations | DNS and service-discovery validation | [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| Domain join | **Demonstrated** | Joined `win11-01` to `corp.home.arpa` | Computer secure channel returned valid | [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| LDAP / Kerberos discovery | **Demonstrated** | Validated DNS-based domain-controller/service discovery | AD locator/service records used during troubleshooting | [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| AD user administration | **Demonstrated** | Created, inspected, disabled, password-reset and deleted controlled test account | AD state and corresponding security events validated | [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| Domain password policy | **Demonstrated** | Inspected and worked through enforced password policy | Complexity, history, min/max age values observed | [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| PowerShell administration | **Demonstrated** | Used Windows Server and AD PowerShell cmdlets for administration and validation | `Get-AD*`, account management, event queries | [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| Windows Security Event analysis | **Demonstrated** | Queried and interpreted account-management events | 4720, 4725 and 4726 exercised | [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| Windows process telemetry | **Demonstrated** | Sysmon and PowerShell logging on Windows endpoint | Process execution telemetry ingested by Wazuh | [`08-windows-telemetry.md`](08-windows-telemetry.md) |

---

# 5. NOC / Infrastructure Operations Skills

The lab already demonstrates several NOC-adjacent operational skills even though
the dedicated Zabbix/SNMP phase is not yet complete.

| Skill / Competency | Status | Hands-on Implementation | Validation / Evidence | Documentation |
|---|---|---|---|---|
| Service availability validation | **Demonstrated** | Verified DNS, Wazuh, Proxmox nodes, storage and agents before/after changes | Service-state, connectivity and functional checks | Multiple documents |
| Resource monitoring | **Demonstrated** | Checked memory, storage utilization and cluster-wide resources | `free -h`, `pvesm status`, `pvesh` resource views | [`03-proxmox-cluster-build.md`](03-proxmox-cluster-build.md) |
| Network connectivity troubleshooting | **Demonstrated** | Validated IP, routes, DNS, ports and service reachability | `ip`, resolver tools, `nc`, host/service tests | Multiple documents |
| Infrastructure health checks | **Demonstrated** | Verified cluster membership, quorum, storage availability and endpoint agents | Repeated before/after validation workflows | [`03-proxmox-cluster-build.md`](03-proxmox-cluster-build.md), [`06-wazuh-siem.md`](06-wazuh-siem.md) |
| DNS service operations | **Demonstrated** | Physical Pi-hole migration, cutover and service validation | Port 53, filtering, resolution and client operation | [`02-pihole-migration.md`](02-pihole-migration.md) |
| Backup operations | **Demonstrated** | Remote backup verification and restore testing | Proxmox backup/restore workflow | [`01-proxmox-backup.md`](01-proxmox-backup.md) |
| Capacity awareness | **Demonstrated** | Reviewed RAM, storage and guest allocations before deploying workloads | Proxmox/Linux resource checks | [`03-proxmox-cluster-build.md`](03-proxmox-cluster-build.md), [`06-wazuh-siem.md`](06-wazuh-siem.md) |
| Root-cause analysis | **Demonstrated** | Diagnosed Wazuh/OpenSearch installation failure caused by guest storage allocation | Failure → storage diagnosis → rebuild/recovery → service validation | [`06-wazuh-siem.md`](06-wazuh-siem.md) |
| Asset management | **Demonstrated** | Current-state asset, IP, role and monitoring inventory | Operational/planned distinction maintained | [`02-asset-inventory.md`](02-asset-inventory.md) |
| Network monitoring with Zabbix | **Planned** | Dedicated NOC monitoring server and agents | Not yet claimed as operational | `07-zabbix-monitoring.md` |
| Cisco managed-switch baseline / CLI | **Demonstrated** | Bench-configured Cisco SG350-10 as `sw01`; assigned management address, validated firmware and VLAN state, and exported a baseline configuration | `show version`, `show vlan`, configuration backup, staging record | [`04-network-vlans.md`](04-network-vlans.md) |
| **SNMP** monitoring | **Operational / expanding** | Cisco switch and infrastructure telemetry | `sw01` successfully monitored by LibreNMS via SNMP; broader SNMPv3 rollout remains future work | LibreNMS / network-monitoring phase |
| Managed-switch production operation | **In progress / planned** | Move homelab links to `sw01`, validate port/link state, then operate it as the lab switching platform | Physical deployment and post-move validation still pending | [`04-network-vlans.md`](04-network-vlans.md) |
| VLAN segmentation | **Planned** | Management/server/user/SOC/attack networks | Not yet claimed as operational | `04-network-vlans.md` |
| OPNsense firewalling / routing | **Planned** | Inter-VLAN routing, firewall policy, NAT and telemetry | Not yet claimed as operational | `05-opnsense.md` |

---

# 6. Troubleshooting and Operational Method

A recurring workflow across the repository is:

```text
Identify the system
        |
        v
Collect current state
        |
        v
Check addressing / routes / DNS
        |
        v
Check service state
        |
        v
Check storage / resources
        |
        v
Check source logs / telemetry
        |
        v
Change one thing
        |
        v
Retest
        |
        v
Document result
```

This method has been applied to:

- Proxmox host and cluster rebuilds
- VM/LXC backup and restoration
- DNS migration
- Wazuh installation failures
- Linux-agent enrollment
- Windows Wazuh-agent setup
- Sysmon and PowerShell telemetry
- Active Directory DNS/domain discovery
- Password-policy errors
- Security-event validation

This is relevant to both SOC and NOC work because effective operations depend on
collecting evidence before changing systems and verifying the result afterward.

---

# 7. Evidence-Backed Interview Examples

## "Tell me about a SIEM you have worked with."

I deployed Wazuh in my homelab, validated its core services, enrolled Linux and
Windows endpoints, and used it for controlled detection exercises. I have
followed activity from the endpoint or application log through Wazuh ingestion,
rule correlation, MITRE ATT&CK mapping, and analyst validation.

See:

- [`06-wazuh-siem.md`](06-wazuh-siem.md)
- [`09-attack-detection-labs.md`](09-attack-detection-labs.md)

## "Tell me about a problem you troubleshot."

During a Wazuh deployment, the virtual disk size visible in Proxmox did not
match the space available to the Ubuntu root filesystem. OpenSearch reached its
disk flood-stage threshold, placed an internal security index into read-only
mode, and the installer failed. I traced the failure through disk and LVM state,
identified the storage-allocation problem, and documented the recovery/rebuild
decision and final validation.

See:

- [`06-wazuh-siem.md`](06-wazuh-siem.md)

## "What Active Directory experience do you have?"

I built a lab Active Directory domain, configured AD-integrated DNS, joined a
Windows 11 workstation, validated domain discovery and the computer secure
channel, administered a test user through its lifecycle, and correlated Windows
Security account-management events into Wazuh.

See:

- [`10-active-directory-lab.md`](10-active-directory-lab.md)

## "What virtualization experience do you have?"

I administer a four-node Proxmox VE cluster using KVM/QEMU and LXC. I have
worked with cluster membership, quorum, rolling node updates, workload
migration, local and shared storage, guest inventory, backup/restoration,
resource checks, Linux networking and centralized cluster management.

See:

- [`01-proxmox-backup.md`](01-proxmox-backup.md)
- [`03-proxmox-cluster-build.md`](03-proxmox-cluster-build.md)
- [`16-proxmox-node-expansion.md`](16-proxmox-node-expansion.md)

## "How do you validate a security alert?"

For controlled labs I compare the SIEM alert with the source of truth on the
endpoint or application. For example, an SSH brute-force alert is compared with
the SSH journal, a SQL-injection-style alert with the Apache access log, a file
integrity alert with the actual file operation and hashes, and an AD account
alert with the Windows Security event and AD account state.

See:

- [`09-attack-detection-labs.md`](09-attack-detection-labs.md)
- [`10-active-directory-lab.md`](10-active-directory-lab.md)

---

# 8. Current Skills Coverage

## Strongest Demonstrated Areas

- Proxmox VE administration
- KVM/QEMU and LXC
- Linux administration
- DNS operations
- Backup and restoration
- Wazuh SIEM deployment and operation
- Linux security telemetry
- Windows endpoint telemetry
- Sysmon
- PowerShell logging
- Active Directory and AD-integrated DNS
- Windows Security Event analysis
- File Integrity Monitoring
- Auditd
- Alert investigation
- MITRE ATT&CK mapping
- Controlled attack/detection validation
- Troubleshooting and root-cause analysis
- Asset inventory
- Technical documentation

## Next Skills to Convert From Planned to Demonstrated

- Cisco managed-switch production cutover and port operations
- VLAN implementation
- 802.1Q trunking
- SNMPv3
- Switch syslog
- SPAN / port mirroring
- OPNsense routing and firewalling
- Inter-VLAN policy
- Zabbix infrastructure monitoring
- Availability / latency / packet-loss monitoring
- Network and service alerting
- Suricata network-security telemetry

These planned items are intentionally separated from demonstrated skills until
the implementation and validation evidence exists.

---

# 9. Repository Evidence Index

| Area | Primary Documentation |
|---|---|
| Backup / disaster recovery | [`01-proxmox-backup.md`](01-proxmox-backup.md) |
| Asset inventory | [`02-asset-inventory.md`](02-asset-inventory.md) |
| DNS / Pi-hole | [`02-pihole-migration.md`](02-pihole-migration.md) |
| Proxmox cluster | [`03-proxmox-cluster-build.md`](03-proxmox-cluster-build.md), [`16-proxmox-node-expansion.md`](16-proxmox-node-expansion.md) |
| Cisco managed switching | [`04-network-vlans.md`](04-network-vlans.md) |
| Wazuh SIEM | [`06-wazuh-siem.md`](06-wazuh-siem.md) |
| Windows telemetry | [`08-windows-telemetry.md`](08-windows-telemetry.md) |
| Detection exercises | [`09-attack-detection-labs.md`](09-attack-detection-labs.md) |
| Active Directory | [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| Detection screenshots | [`../evidence/wazuh/`](../evidence/wazuh/) |
| Current architecture | [`../diagrams/current-architecture.md`](../diagrams/current-architecture.md) |

---

# 10. Summary

The value of this homelab is not simply that the technologies are installed.

The repository demonstrates a repeatable operational cycle:

```text
Build
  ↓
Configure
  ↓
Validate
  ↓
Generate activity
  ↓
Observe telemetry
  ↓
Investigate
  ↓
Troubleshoot
  ↓
Recover / remediate
  ↓
Verify
  ↓
Document
```

That cycle is directly relevant to entry-level and junior work in:

- Security Operations Centers
- Network Operations Centers
- Security analyst roles
- Systems administration
- Infrastructure support
- Endpoint monitoring
- Technical operations
