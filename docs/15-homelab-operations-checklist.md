# Homelab Operations and Health Checklist

## Purpose

This document defines the routine operational checks used to keep the SOC/NOC
homelab healthy after deployment.

The objective is to practice the kind of recurring operational discipline used
in NOC, SOC, systems-administration, and infrastructure-support roles:

- confirm important systems are reachable;
- verify critical services are running;
- identify capacity problems before they become outages;
- confirm backups remain accessible;
- detect missing security telemetry;
- validate Windows/Active Directory health;
- document exceptions and follow-up work.

This checklist is intentionally based on the services that are currently
implemented. The Cisco SG350-10 baseline is now staged and receives limited
configuration/backup checks, while production switch operation, Zabbix, SNMP
monitoring, VLAN segmentation, and OPNsense remain separate until deployment and
validation.

---

# 1. Current Critical Systems

| System | Role | Address |
|---|---|---:|
| `pve01` | Proxmox VE cluster node | `192.168.1.10` |
| `pve02` | Proxmox VE cluster node | `192.168.1.11` |
| `pve03` | Proxmox VE cluster node | `192.168.1.12` |
| `pve04` | Proxmox VE cluster node | `192.168.1.13` |
| `mgmt01` | Independent management / health-monitoring host | `192.168.1.5` |
| `dns01` | Pi-hole DNS | `192.168.1.20` |
| `nms01` | LibreNMS network monitoring | `192.168.1.22` |
| `sw01` | Cisco SG350-10 managed switch (staged) | `192.168.1.21` |
| `dc01` | Active Directory / DNS | `192.168.1.30` |
| `apache-guacamole` | Remote-access gateway | `192.168.1.151` |
| `wazuh01` | Wazuh SIEM | `192.168.1.206` |
| `kali01` | Security-testing system | `192.168.1.211` |
| `target01` | Ubuntu monitored target | `192.168.1.238` |
| `storage01` | File / backup storage | `192.168.1.208` |
| `t-20-backup` | Proxmox shared backup storage | CIFS target |

---

# 2. Daily Health Check

The daily check is designed to be fast.

## Automated Baseline from `mgmt01`

`mgmt01` provides an independent management and monitoring path for the lab.

The primary consolidated health check is `homelab-status`.

A healthy run ends with `OVERALL STATUS: HEALTHY` and returns exit status `0`.

The consolidated check validates:

- network and critical-service reachability;
- Proxmox cluster, quorum, storage, backups, resources, and expected guests;
- Linux infrastructure health on `dns01` and `docker`;
- Pi-hole FTL status;
- Docker Engine and Uptime Kuma health;
- `storage01` ICMP and SMB reachability;
- authenticated access to the `ProxmoxBackups` share and `dump` directory.

Health-check logs are stored under `~/.local/state/homelab/`.

The most recent run is available as `~/.local/state/homelab/latest.log`.

Recent results can be summarized with `homelab-history`.

The same check runs hourly through the user-level `homelab-status.timer`
systemd timer.

Service execution history can be reviewed with:

    journalctl --user -u homelab-status.service --no-pager

The management SSH key remains passphrase protected. After a reboot or login,
load it into the persistent user SSH agent with:

    ssh-add ~/.ssh/id_ed25519

The management workstation can be rebuilt or reconfigured with:

    scripts/setup-mgmt01

Private SSH keys and SMB credentials remain outside the repository.


## 2.1 Proxmox Cluster

From a cluster node:

```bash
pvecm status
```

Healthy state should include:

```text
Nodes:          4
Expected votes: 4
Total votes:    4
Quorum:         3
Quorate:        Yes
```

Then:

```bash
pvesh get /nodes
```

Expected:

```text
pve01   online
pve02   online
pve03   online
pve04   online
```

### If Not Healthy

Check:

```bash
pvecm nodes
ip -br addr
ip route
```

Then test reachability among cluster nodes.

---

## 2.2 Wazuh Core Services

On `wazuh01`:

```bash
sudo systemctl is-active wazuh-indexer
sudo systemctl is-active wazuh-manager
sudo systemctl is-active filebeat
sudo systemctl is-active wazuh-dashboard
```

Expected result for each:

```text
active
```

### If a Service Is Inactive

Run:

```bash
sudo systemctl status <service> --no-pager -l
sudo journalctl -u <service> --since "30 minutes ago" --no-pager
```

Also check:

```bash
df -h /
free -h
```

---

## 2.3 Wazuh Agent Coverage

Verify that expected monitored endpoints still appear active.

Current key endpoints include:

```text
target01
win11-01
dc01
```

Confirm that telemetry is recent rather than merely that the agent exists.

Check for:

- recent Linux events;
- recent Windows Security events;
- recent Sysmon events;
- recent PowerShell telemetry where applicable.

### Warning Condition

A monitored endpoint that shows no recent events should be investigated even if
its agent is technically still enrolled.

---

## LibreNMS / NOC Monitoring

`nms01` provides LibreNMS network monitoring at `192.168.1.22`.

The automated `mgmt01` health checks validate:

- host reachability and LibreNMS HTTP response;
- SSH availability;
- root-filesystem and memory pressure;
- MariaDB;
- nginx;
- PHP-FPM;
- SNMP daemon;
- cron;
- LibreNMS scheduler timer;
- LibreNMS database connectivity;
- active pollers;
- Python poller-wrapper operation.

A normal LibreNMS web check may return HTTP `302`; this is treated as healthy
because the application redirects the initial request.

The detailed Linux check can be run manually with:

    linux-health

or directly through Ansible:

    ansible-playbook ansible/linux-health.yml

LibreNMS application validation can be checked on `nms01` with:

    cd /opt/librenms
    sudo -u librenms ./validate.php

The normal daily LibreNMS maintenance job is scheduled through
`/etc/cron.d/librenms`. A host that is powered off at the scheduled time will
miss that cron execution; traditional cron does not automatically replay the
missed job after boot.

## 2.4 Pi-hole DNS

On `dns01`:

```bash
pihole status
```

Expected:

- FTL operational;
- DNS service available;
- blocking enabled.

From another host:

```bash
ping -c 3 192.168.1.20
```

Then test DNS:

```bash
dig example.com
```

or:

```bash
nslookup example.com
```

### Healthy State

```text
dns01 reachable
DNS resolves public names
Pi-hole service operational
```

---

## 2.5 Internet / Gateway Check

From `mgmt01` or another known-good Linux host:

```bash
ping -c 3 192.168.1.254
```

Then:

```bash
ping -c 3 google.com
```

Interpretation:

```text
gateway fails
→ local network / gateway issue

gateway works, hostname fails
→ likely DNS issue

gateway works, DNS works
→ basic network path healthy
```

---

## 2.6 Critical Storage Check

On a Proxmox node:

```bash
pvesm status
```

Expected active storage:

```text
local
local-lvm
t-20-backup
```

Look for:

- inactive storage;
- unexpectedly high utilization;
- backup storage unavailable.

For `t-20-backup`, Proxmox reports the Drobo's 64 TB logical volume rather than
its physical capacity. Check Drobo Dashboard for real usable space and health.

---

# 3. Weekly Operations Checklist

The weekly check goes deeper than the daily availability check.

## 3.1 Proxmox Resource Review

Run:

```bash
pvesh get /cluster/resources --type vm
```

Review:

- running guests;
- stopped guests that should be running;
- memory allocation;
- placement across nodes.

On each host:

```bash
free -h
df -h
```

Review:

- RAM pressure;
- swap use;
- filesystem capacity.

---

## 3.2 Proxmox Storage Capacity

Run:

```bash
pvesm status
```

Check:

- `local`;
- `local-lvm`;
- `t-20-backup`.

Record any storage approaching an operational threshold.

A useful warning threshold for manual review is:

```text
80% used
```

A critical threshold for immediate attention is:

```text
90% used
```

These are operational review thresholds, not hard product limits.

---

## 3.3 Backup Visibility

Verify Proxmox can list the backup destination contents:

```bash
pvesm list t-20-backup
```

Questions:

- Are recent backups visible?
- Are expected guests represented?
- Is the target reachable?
- Has available capacity changed unexpectedly?

Do not assume a scheduled backup succeeded because the schedule exists.

If `t-20-backup` becomes unavailable after a reboot or power interruption,
check the storage-server dependency chain before modifying the Proxmox storage
definition:

```text
Drobo / attached storage
        ↓
Windows volume
        ↓
SMB share and LanmanServer
        ↓
Windows network profile / firewall
        ↓
TCP 445 reachability
        ↓
Proxmox CIFS storage
```

Useful checks on `storage01`:

```powershell
Get-Service LanmanServer
Get-SmbShare
Get-NetConnectionProfile
```

The trusted home-LAN Ethernet connection should normally be classified as
`Private`.

From Proxmox:

```bash
nc -nvz -w 3 192.168.1.208 445
pvesm status
pvesm list t-20-backup
```

Current expected coverage is one recent archive for each guest ID:

```text
100 101 105 200 201 210 300 320 400 500
```

The daily jobs are intentionally staggered:

| Node | Schedule |
|---|---:|
| `pve01` | `02:00` |
| `pve02` | `04:00` |
| `pve03` | `05:00` |
| `pve04` | `06:00` |

Storage-level retention should remain:

```text
keep-last=7,keep-weekly=4,keep-monthly=3
```

Verify the current job and storage configuration with:

```bash
pvesh get /cluster/backup --output-format yaml
pvesh get /storage/t-20-backup --output-format yaml
```

## 3.3.1 Proxmox Thin-Pool Review

On each Proxmox node, periodically inspect LVM-thin data and metadata usage:

```bash
lvs -a --units g \
    -o vg_name,lv_name,lv_size,data_percent,metadata_percent,seg_monitor

vgs --units g \
    -o vg_name,vg_size,vg_free
```

On `pve01`, LVM monitoring is enabled and the current auto-extension guard is:

```text
thin_pool_autoextend_threshold=80
thin_pool_autoextend_percent=4
```

The small percentage reflects the limited free space remaining in the volume
group. It provides a guard, not unlimited growth. Continue monitoring actual
thin-pool consumption and plan added capacity before the reserve is exhausted.

---

## 3.4 Backup Spot Check

Select at least one backup periodically and confirm that Proxmox can read its
configuration metadata.

Example workflow:

```text
List backup
   ↓
Select recent guest
   ↓
Inspect configuration
   ↓
Confirm archive is readable
```

Where appropriate, use:

```bash
pvesm extractconfig ...
```

Do not present reconstructed commands as original execution history.

---

## 3.5 Linux Service Review

On key Linux systems, review failed systemd units:

```bash
systemctl --failed
```

For important services:

```bash
systemctl is-active <service>
```

Investigate anything unexpectedly failed.

---

## 3.6 Wazuh Alert Review

Review recent alerts for:

- repeated authentication failures;
- unexpected account changes;
- file integrity changes;
- suspicious process activity;
- Windows Security events;
- PowerShell activity;
- Auditd events.

The goal is not simply to clear alerts.

Ask:

```text
Is this expected?
Is it repeated?
Is it new?
Does it require follow-up?
```

---

## 3.7 Windows Endpoint Telemetry

On `win11-01`, confirm:

- Wazuh agent is running;
- Sysmon telemetry is present;
- Windows Security events are arriving;
- PowerShell logging is still visible.

Example PowerShell service check:

```powershell
Get-Service WazuhSvc
```

Review recent event activity where needed.

---

## 3.8 Active Directory Health

On `dc01`, verify the domain is responding.

Useful checks:

```powershell
Get-ADDomain
Get-ADForest
Get-ADDomainController
```

From `win11-01`:

```powershell
Test-ComputerSecureChannel -Verbose
```

Healthy result:

```text
True
```

Also verify DNS resolution for:

```text
dc01.corp.home.arpa
```

---

## 3.7 Cisco Switch Staging Check

Until `sw01` becomes the active homelab switching path, perform only the checks
that match its actual staged state:

- confirm `192.168.1.21` is still the intended unique management address;
- confirm the stored baseline configuration backup remains available;
- record firmware if it changes;
- use `show vlan` to confirm that no unplanned segmentation was introduced;
- keep the physical port/cable map current as devices are connected.

After the physical migration, move link-state, interface-counter, VLAN, trunk,
SNMP, syslog, and SPAN checks into the regular active operations schedule.

---

# 4. Monthly Operations Checklist

The monthly checklist focuses on maintenance, recovery readiness, and trend
review.

## 4.1 Operating System Updates

Review pending updates on Linux systems.

Example:

```bash
sudo apt update
apt list --upgradable
```

Apply updates according to the role and maintenance risk of the system.

For critical infrastructure such as DNS or Wazuh, confirm backup/recovery options
before major updates.

---

## 4.2 Proxmox Update Review

Review available Proxmox updates and release notes before applying significant
changes.

Operational approach:

```text
Verify backups
   ↓
Check cluster healthy
   ↓
Update one node
   ↓
Validate node
   ↓
Proceed to next node
```

Avoid unnecessary simultaneous maintenance on all cluster nodes.

---

## 4.3 Backup Restore Test

At least periodically, perform a controlled restore test.

A useful sequence is:

```text
Select backup
   ↓
Restore to temporary guest ID
   ↓
Start guest
   ↓
Validate networking
   ↓
Validate service
   ↓
Document result
   ↓
Remove temporary test if no longer needed
```

Backup creation without restore testing is incomplete disaster-recovery
validation.

---

## 4.4 Asset Inventory Review

Review:

```text
docs/02-asset-inventory.md
```

Confirm:

- hostnames still accurate;
- IP addresses still accurate;
- DHCP-observed systems documented correctly;
- retired assets removed from current state;
- new assets added;
- planned systems are not incorrectly marked operational.

---

## 4.5 Documentation Drift Review

Check whether implementation documents still match reality.

Examples:

- a service marked planned is now deployed;
- an IP changed;
- a hostname changed;
- a VM moved;
- a backup target changed;
- monitoring coverage expanded;
- an old troubleshooting note is now misleading.

Documentation should be treated as operational state, not a historical artifact
that is never revisited.

---

## 4.6 Password / Credential Hygiene

Review administrative credentials according to actual service requirements.

Do not store credentials in the public repository.

Check for:

- unused administrative accounts;
- stale test accounts;
- default credentials;
- unnecessary local accounts;
- credentials embedded in scripts or configuration examples.

---

## 4.7 Active Directory Account Review

Review test and administrative accounts.

Example:

```powershell
Get-ADUser -Filter * |
Select-Object Name, SamAccountName, Enabled
```

Questions:

- Does each account still have a purpose?
- Are temporary lab accounts still required?
- Are disabled accounts intentionally retained?
- Are privileged memberships appropriate?

---

# 5. Quarterly Recovery and Failure Testing

A less frequent test can intentionally validate failure handling.

Examples:

## DNS Independence Test

Verify DNS remains available during controlled Proxmox maintenance.

Expected architectural result:

```text
Proxmox host maintenance
        |
        X
        |
dns01 remains available
```

## Single Proxmox Node Failure Test

During a controlled maintenance window:

- shut down or isolate one cluster node;
- verify surviving nodes retain quorum;
- verify management remains available;
- document affected workloads.

This tests cluster control-plane behavior, not automatic HA.

## Backup Restore Exercise

Restore a selected VM/LXC and validate real service functionality.

## Wazuh Telemetry Test

Generate a known benign test event and verify the full collection path.

---

# 6. Security Monitoring Health Checks

Monitoring can fail silently.

Routine validation should include generating known test activity.

Examples:

## Linux

Generate a benign authentication or file-integrity test in the lab.

Confirm:

```text
activity
  ↓
endpoint log
  ↓
Wazuh agent
  ↓
Wazuh manager
  ↓
alert/search result
```

## Windows

Generate a benign PowerShell action:

```powershell
Get-Date
```

Confirm recent process / PowerShell telemetry appears as expected.

## Active Directory

When appropriate, use a controlled test account to verify Windows Security and
Wazuh identity-event collection.

Do not repeatedly create noisy events merely to populate the dashboard.

---

# 7. Capacity Threshold Checklist

## Filesystems

Review when:

```text
>= 80% used → investigate trend
>= 90% used → prioritize corrective action
```

## Proxmox Storage

Watch:

- `local`;
- `local-lvm`;
- backup storage.

## Memory

Investigate:

- sustained low available memory;
- heavy swap use;
- guest allocation growth.

## Backup Storage

Track:

- available capacity;
- backup growth;
- retention.

The Wazuh deployment demonstrated why capacity checks matter: a virtual disk can
appear adequately sized at the hypervisor while the guest filesystem has not
actually been allocated that capacity.

---

# 8. Change Readiness Checklist

Before a significant infrastructure change:

- [ ] Identify affected systems.
- [ ] Confirm current health.
- [ ] Confirm current IP/hostname.
- [ ] Confirm backups exist where applicable.
- [ ] Confirm rollback path.
- [ ] Record current configuration.
- [ ] Define expected result.
- [ ] Make one controlled change.
- [ ] Validate service afterward.
- [ ] Update documentation.
- [ ] Commit relevant repository changes.
- [ ] For physical/network changes, photograph or record the pre-change cable/port state.
- [ ] Verify both ends of affected Ethernet cables are labeled before disconnecting them.

---

# 9. Incident Follow-Up Checklist

After an outage or service failure:

- [ ] Was the root cause identified?
- [ ] Was service restored?
- [ ] Was restoration independently verified?
- [ ] Were temporary workarounds removed?
- [ ] Is monitoring functioning?
- [ ] Does the issue need a preventive action?
- [ ] Does the asset inventory need updating?
- [ ] Does a runbook need updating?
- [ ] Was evidence preserved?
- [ ] Was the incident/ticket closed with useful notes?

---

# 10. Daily Checklist Template

```text
Date:

PROXMOX
[ ] Cluster quorate
[ ] pve01 online
[ ] pve02 online
[ ] pve03 online
[ ] pve04 online
[ ] Storage active

DNS
[ ] dns01 reachable
[ ] Pi-hole active
[ ] DNS resolution successful

WAZUH
[ ] Indexer active
[ ] Manager active
[ ] Filebeat active
[ ] Dashboard active
[ ] Key agents reporting

WINDOWS / AD
[ ] dc01 reachable
[ ] win11-01 telemetry recent
[ ] AD/DNS normal

CONNECTIVITY
[ ] Gateway reachable
[ ] Internet name resolution works

EXCEPTIONS / NOTES:
```

---

# 11. Weekly Checklist Template

```text
Week of:

PROXMOX
[ ] Review cluster resources
[ ] Review node RAM
[ ] Review filesystem usage
[ ] Review local-lvm capacity
[ ] Review shared storage capacity

BACKUPS
[ ] Recent backups visible
[ ] Backup target accessible
[ ] Spot-check backup metadata
[ ] Confirm all expected guest IDs are represented
[ ] Confirm Drobo health and physical capacity in Drobo Dashboard

WAZUH
[ ] Review recent alerts
[ ] Confirm Linux telemetry
[ ] Confirm Windows telemetry
[ ] Confirm AD security telemetry

DNS
[ ] Pi-hole healthy
[ ] Resolution normal

WINDOWS / AD
[ ] Domain responsive
[ ] Secure channel valid
[ ] No unexpected account changes

LINUX SERVICES
[ ] Review failed units

FOLLOW-UP:
```

---

# 12. Monthly Checklist Template

```text
Month:

MAINTENANCE
[ ] Review Linux updates
[ ] Review Proxmox updates
[ ] Schedule maintenance if needed

RECOVERY
[ ] Perform/confirm restore test
[ ] Verify backup capacity
[ ] Review rollback readiness

DOCUMENTATION
[ ] Review asset inventory
[ ] Review current architecture
[ ] Update changed IPs/roles/status
[ ] Remove stale planned language

IDENTITY
[ ] Review AD accounts
[ ] Review test accounts
[ ] Review administrative accounts

SECURITY
[ ] Confirm telemetry test path
[ ] Review credential hygiene

CAPACITY
[ ] Review disk trends
[ ] Review memory trends
[ ] Review backup growth
[ ] Review LVM-thin data and metadata percentages

NOTES / ACTION ITEMS:
```

---

# 13. Operations Log Template

For recurring checks:

```text
Date:
Operator:

Check Performed:
System:
Expected Result:
Actual Result:

Status:
[ ] Healthy
[ ] Warning
[ ] Failed
[ ] Follow-up required

Action Taken:

Verification:

Ticket / Documentation Reference:
```

---

# 14. Planned Future Additions

When the following systems become operational, add them to the active
operations checklist:

## Cisco Managed Switch — After Physical Deployment

Baseline staging is complete. After `sw01` begins carrying homelab traffic, add:

- switch reachability;
- interfaces up/down;
- error counters;
- VLAN membership;
- trunk state;
- configuration-backup freshness;
- SNMPv3;
- syslog;
- SPAN configuration where used.

Do not mark VLAN, trunk, SNMP, syslog, or SPAN checks complete before those
features are actually configured and validated.

## Zabbix

Future checks:

- Zabbix server health;
- agent availability;
- trigger state;
- unreachable hosts;
- alert queue;
- monitored CPU/memory/disk;
- latency and packet loss.

## OPNsense

Future checks:

- interface state;
- routing;
- DHCP;
- DNS forwarding;
- firewall logs;
- blocked traffic;
- gateway state;
- inter-VLAN connectivity.

## Suricata

Future checks:

- engine state;
- interface capture;
- rule updates;
- alert volume;
- false-positive review;
- Wazuh/SIEM integration.

These should not be marked complete until deployed and validated.

---

# 15. Skills Demonstrated

This operational checklist demonstrates understanding of:

## NOC Operations

- Routine health checks
- Service availability
- Capacity review
- Infrastructure monitoring
- Escalation readiness
- Maintenance planning
- Operational documentation

## SOC Operations

- Telemetry health
- Security alert review
- Endpoint monitoring
- Detection-path validation
- Identity-monitoring checks

## Systems Administration

- Linux services
- Proxmox clustering
- Storage
- DNS
- Active Directory
- Windows endpoints
- Backup/recovery

## Operational Discipline

- Preventive checks
- Change readiness
- Failure validation
- Rollback planning
- Restore testing
- Documentation maintenance
- Post-incident follow-up

---

# 16. Related Documentation

| Area | Document |
|---|---|
| Backup / disaster recovery | [`01-proxmox-backup.md`](01-proxmox-backup.md) |
| Asset inventory | [`02-asset-inventory.md`](02-asset-inventory.md) |
| Pi-hole DNS | [`02-pihole-migration.md`](02-pihole-migration.md) |
| Proxmox cluster | [`03-proxmox-cluster-build.md`](03-proxmox-cluster-build.md) |
| Cisco managed switching | [`04-network-vlans.md`](04-network-vlans.md) |
| Physical infrastructure | [`18-physical-infrastructure.md`](18-physical-infrastructure.md) |
| Shutdown / startup | [`19-shutdown-startup-runbook.md`](19-shutdown-startup-runbook.md) |
| Cabling standard | [`20-cabling-standard.md`](20-cabling-standard.md) |
| Wazuh SIEM | [`06-wazuh-siem.md`](06-wazuh-siem.md) |
| Windows telemetry | [`08-windows-telemetry.md`](08-windows-telemetry.md) |
| Active Directory | [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| Skills matrix | [`11-soc-noc-skills-matrix.md`](11-soc-noc-skills-matrix.md) |
| SOC triage | [`12-soc-alert-triage-playbook.md`](12-soc-alert-triage-playbook.md) |
| NOC troubleshooting | [`13-noc-troubleshooting-runbook.md`](13-noc-troubleshooting-runbook.md) |
| Ticket examples | [`14-soc-noc-ticket-examples.md`](14-soc-noc-ticket-examples.md) |

---

# 17. Summary

The lab is treated as an environment that must be operated, not merely built.

The recurring operational cycle is:

```text
Check
  ↓
Measure
  ↓
Detect exception
  ↓
Investigate
  ↓
Correct
  ↓
Verify
  ↓
Document
  ↓
Repeat
```

That cycle supports practical experience relevant to NOC, SOC, systems
administration, infrastructure support, and junior operations roles.
