# SOC/NOC Skills Matrix

## Overview

This page connects the technical work I've done in my homelab with skills relevant to SOC, NOC, systems administration, and infrastructure-support positions.

Rather than list technologies I've experimented with, I want to show what I've actually configured, tested, troubleshot, and documented.

The links point to the work itself.

I distinguish between completed work, systems I'm still expanding, and projects I haven't started.

---

## 1. Virtualization and Systems Administration

| Skill | Work completed | Evidence |
|---|---|---|
| Proxmox VE | Built and administer a four-node cluster with VMs, LXC containers, and centralized management | [Cluster build](03-proxmox-cluster-build.md) |
| Cluster administration | Validated membership, quorum, node health, and rolling updates | [Node expansion](16-proxmox-node-expansion.md) |
| VM administration | Created, configured, migrated, backed up, and restored virtual machines | [Proxmox build](03-proxmox-cluster-build.md) |
| Linux administration | Configured Ubuntu, Debian, Proxmox, and Raspberry Pi OS systems | [Wazuh deployment](06-wazuh-siem.md) |
| Windows Server | Deployed Windows Server 2025 and configured Active Directory | [Active Directory](10-active-directory-lab.md) |
| Backup and recovery | Configured shared backup storage and performed restore validation | [Restore validation](21-proxmox-backup-restore-validation.md) |
| Storage administration | Worked with CIFS mounts, LVM, Proxmox storage, and backup recovery | [Storage resilience](17-storage-backup-resilience.md) |
| Service troubleshooting | Used systemd, journalctl, logs, and resource checks to diagnose failures | [Wazuh troubleshooting](06-wazuh-siem.md) |

One of the more useful examples is the Wazuh installation failure. I traced the problem from an OpenSearch error back to an Ubuntu root logical volume that wasn't using the available virtual disk capacity.

That investigation involved Linux storage tools, service logs, filesystem permissions, and ultimately a clean rebuild.

---

## 2. Networking and NOC Operations

| Skill | Work completed | Evidence |
|---|---|---|
| Managed switching | Configured and operate Cisco SG350-10 and UniFi US-8 switches | [Network documentation](04-network-vlans.md) |
| VLANs and 802.1Q | Configured trunks and segmented networks for lab workloads | [VLAN configuration](04-network-vlans.md) |
| Routing and firewalling | Deployed OPNsense for inter-VLAN routing and access control | [OPNsense](05-opnsense.md) |
| Network troubleshooting | Diagnosed routing, DNS, return-path, and connectivity problems | [NOC runbook](13-noc-troubleshooting-runbook.md) |
| DNS | Migrated Pi-hole to dedicated hardware and configured Active Directory DNS | [Pi-hole migration](02-pihole-migration.md) |
| SNMP monitoring | Integrated managed-switch monitoring with LibreNMS | [Network documentation](04-network-vlans.md) |
| Availability monitoring | Deployed Uptime Kuma for host and service checks | [Operations](15-homelab-operations-checklist.md) |
| Metrics and dashboards | Deployed Prometheus and Grafana, including monitoring a remote VPS | [Asset inventory](02-asset-inventory.md) |
| Remote access | Configured Tailscale, SSH tunnels, and browser-based administration | [Tailscale](23-tailscale-remote-access.md) |

The `reply-to` problem in OPNsense is a particularly good troubleshooting example.

Traffic from a trusted laptop matched an allow rule, but connections to internal services still failed. The problem involved return-path routing rather than the initial firewall permission.

After identifying the behavior and disabling reply-to on the appropriate rule, I verified the connections.

This required investigating more than simple host reachability or firewall pass logs.

---

## 3. Security Operations

| Skill | Work completed | Evidence |
|---|---|---|
| SIEM deployment | Installed and validated Wazuh manager, indexer, Filebeat, and dashboard | [Wazuh deployment](06-wazuh-siem.md) |
| Endpoint enrollment | Connected Linux and Windows endpoints to Wazuh | [Wazuh](06-wazuh-siem.md) |
| Alert investigation | Generated controlled activity and compared alerts with original logs | [Detection labs](09-attack-detection-labs.md) |
| Authentication monitoring | Detected repeated failed SSH login attempts | [Detection Lab 1](09-attack-detection-labs.md) |
| File Integrity Monitoring | Configured real-time file monitoring with content-change reporting | [Detection Lab 2](09-attack-detection-labs.md) |
| Web log analysis | Detected an SQL-injection-style request in Apache logs | [Detection Lab 3](09-attack-detection-labs.md) |
| Linux Auditd | Monitored root-level command execution and user attribution | [Detection Lab 4](09-attack-detection-labs.md) |
| Account monitoring | Detected local Linux account creation and deletion | [Detection Lab 5](09-attack-detection-labs.md) |
| Windows Sysmon | Collected process-creation telemetry and validated Wazuh alerts | [Windows telemetry](08-windows-telemetry.md) |
| PowerShell logging | Enabled Script Block Logging and detected a test registry modification | [Windows telemetry](08-windows-telemetry.md) |
| Active Directory monitoring | Generated Windows Security Event 4720 and verified Wazuh Rule 60109 | [Active Directory](10-active-directory-lab.md) |
| MITRE ATT&CK | Reviewed mappings associated with controlled detection exercises | [Detection labs](09-attack-detection-labs.md) |

These exercises are not examples of responding to live production incidents.

They demonstrate the underlying process: generating a known event, finding the source evidence, locating the resulting SIEM alert, and examining what the detection means.

---

## 4. Detection Evidence

The following results were recorded during the lab exercises.

| Detection | Wazuh rule | Level | Related technique |
|---|---:|---:|---|
| SSH brute force | `5712` | 10 | `T1110` |
| File modification | `550` | 7 | `T1565.001` |
| SQL-injection-style request | `31103` | 7 | `T1190` |
| Privileged Linux command | `80792` | 3 | Context-dependent |
| Local account creation | `5902` | 8 | `T1136` |
| Local account deletion | `5903` | 3 | `T1531` |
| Windows PowerShell process | `92027` | 4 | `T1059.001` |
| PowerShell registry modification | `91843` | 3 | `T1059.001`, `T1112` |
| Active Directory account creation | `60109` | 8 | `T1098` |

The recorded rule numbers and levels correspond to the lab configuration used during testing.

Screenshots and supporting evidence are included in the individual detection documents and under [`evidence/wazuh/`](../evidence/wazuh/).

---

## 5. Windows and Identity Administration

Beyond the SIEM exercises, I've worked with several Windows administration tasks.

| Skill | Validation |
|---|---|
| Active Directory Domain Services | Created the `corp.home.arpa` domain |
| Active Directory DNS | Checked domain and LDAP/Kerberos service discovery |
| Domain joining | Joined Windows 11 to Active Directory |
| Secure-channel verification | Used `Test-ComputerSecureChannel` |
| Account administration | Created, inspected, disabled, and deleted a test domain account |
| Password policy | Inspected policy settings and resolved a password-compliance issue |
| Windows Security logs | Queried account-management events through PowerShell |
| Windows service administration | Troubleshot Wazuh agent installation and service permissions |

The supporting commands and results are in the [Active Directory Lab](10-active-directory-lab.md) and [Windows Endpoint Telemetry](08-windows-telemetry.md).

---

## 6. Troubleshooting and Operational Work

Much of the work in the lab involves problems that weren't part of the original installation plan.

Some documented examples include:

| Problem | Investigation and result |
|---|---|
| Wazuh installation failed | Identified insufficient root filesystem allocation and rebuilt successfully |
| Wazuh indexer wouldn't start | Investigated an access-denied error involving an indexer backup directory |
| Windows Wazuh agent missing | Verified the MSI hadn't installed, reinstalled it, and corrected service-control permissions |
| OPNsense allowed traffic but connections failed | Identified and corrected reply-to routing behavior |
| Remote lab access failed while traveling | Investigated overlapping subnets and used alternative Tailscale/SSH access paths |
| Backup reliability needed validation | Restored a Docker VM into an isolated test instance and checked its services |

These examples are particularly useful because they show diagnosis and verification, not simply installation.

---

## 7. Projects Still in Progress or Planned

Some services are operational but still being expanded. Others are planned and shouldn't be presented as completed deployments.

| Project | Status |
|---|---|
| VLAN migration | Operational, with remaining dependencies and address cleanup |
| Default-LAN renumbering | Planned; considering `192.168.11.0/24` |
| OPNsense rule cleanup | In progress |
| SNMPv3 expansion | Planned |
| OpenVAS vulnerability-scanning exercise | Planned/documentation in progress |
| Zabbix monitoring | Planned, not deployed |
| Custom Wazuh rules and Active Response | Planned |
| Additional recovery testing | Ongoing |
| Infrastructure automation | Ongoing development |

I intend to update this matrix when the corresponding implementation and validation work is documented.

---

## 8. Documentation and Evidence

The repository serves as a technical record of the homelab.

My documentation includes:

- System and network configurations
- Commands used during deployments
- Validation checks and actual results
- Failed attempts and troubleshooting
- Recovery procedures
- Detection rules and evidence
- Infrastructure decisions and tradeoffs
- Work that's still planned

I try to distinguish an installed service from a tested service, and a tested service from one I've actually used to investigate a problem.

That distinction matters when describing the experience accurately.

---

## Summary

The strongest areas demonstrated in this repository are Proxmox and Linux administration, managed switching and VLANs, DNS, OPNsense routing, backup and recovery, Wazuh SIEM operations, Windows telemetry, Active Directory, and technical troubleshooting.

The homelab isn't a production SOC or NOC.

It's a working environment where I can build systems, generate controlled problems, observe their behavior, investigate failures, and document the results.

This page is intended to make the supporting evidence easy to find.

## Related Documentation

- [Repository Overview](../README.md)
- [Asset Inventory](02-asset-inventory.md)
- [Proxmox Cluster](03-proxmox-cluster-build.md)
- [Network and VLANs](04-network-vlans.md)
- [Wazuh SIEM](06-wazuh-siem.md)
- [Detection Labs](09-attack-detection-labs.md)
- [Active Directory](10-active-directory-lab.md)
- [Backup Restore Validation](21-proxmox-backup-restore-validation.md)
