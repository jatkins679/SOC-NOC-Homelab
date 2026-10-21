# Homelab Operations and Health Checklist

## Overview

Building services in the homelab is one thing. Keeping them running, checking their health, and noticing when something changes is a different part of the work.

I've gradually added several monitoring systems and administrative tools to help with that.

These include LibreNMS, Uptime Kuma, Prometheus, Grafana, Wazuh, and a set of health-check scripts and Ansible playbooks.

This document records the recurring checks I use or have configured, along with the manual checks I want to perform during regular maintenance.

Not every check is automated. Some are periodic review tasks, while others are part of the lab's existing monitoring configuration.

---

## 1. Main Operational Systems

| System | Documented address | Role |
|---|---|---|
| `mgmt01` | `192.168.1.5` | Management and health checks |
| `pve01` | `192.168.1.10` | Proxmox |
| `pve02` | `192.168.1.11` | Proxmox |
| `pve03` | `192.168.1.12` | Proxmox |
| `pve04` | `192.168.1.13` | Proxmox |
| `dns01` | `192.168.1.20` | Pi-hole |
| `sw01` | `192.168.1.21` | Cisco switch |
| `sw02` | `192.168.1.22` | UniFi switch |
| `docker` | `192.168.1.174` | Containerized applications |
| `storage01` | `192.168.1.208` | Backup storage |
| `dc01` | `10.10.20.10` | Active Directory |
| `nms01` | `10.10.40.10` | LibreNMS |
| `wazuh01` | `10.10.40.20` | Wazuh |
| `nb01` | `10.10.10.15` | NetBox |
| `fw01` | `192.168.1.187` / `10.10.10.1` | OPNsense |

The addresses reflect documented lab configuration rather than a live network inventory.

The original LAN is still in use alongside the segmented VLANs.

---

## 2. Automated Health Monitoring

### Management Host Checks

I use `mgmt01` as an independent management and health-check system.

The main consolidated check is:

```bash
homelab-status
```

A healthy result ends with:

```text
OVERALL STATUS: HEALTHY
```

and returns exit status `0`.

The checks cover the Proxmox cluster, key hosts, backup storage, service availability, and supporting applications.

Health-check logs are stored under:

```text
~/.local/state/homelab/
```

The most recent log is:

```text
~/.local/state/homelab/latest.log
```

Recent results can be summarized with:

```bash
homelab-history
```

The consolidated check also runs hourly using a user-level systemd timer.

To review execution history:

```bash
journalctl --user -u homelab-status.service --no-pager
```

The management host uses a passphrase-protected SSH key. When necessary, I load it into the SSH agent:

```bash
ssh-add ~/.ssh/id_ed25519
```

The management environment can be configured using the repository script:

```bash
scripts/setup-mgmt01
```

Private keys and other credentials aren't stored in the public repository.

### Other Monitoring Systems

Different tools provide different parts of the overall picture.

| Tool | Current use |
|---|---|
| Uptime Kuma | Host, port, and HTTP availability checks |
| LibreNMS | Network monitoring and SNMP telemetry |
| Prometheus | Metrics collection |
| Grafana | Metrics dashboards |
| Wazuh | Security-event and endpoint monitoring |
| NetBox | Infrastructure and IP address inventory |

These tools overlap in some areas, but none is a substitute for checking whether important applications actually work.

---

## 3. Daily Health Review

The daily review is intended to be brief.

I generally begin with the automated health results and follow up on exceptions rather than manually checking every system from scratch.

### Proxmox Cluster

From a cluster node:

```bash
pvecm status
pvesh get /nodes
```

Under the documented four-node configuration, a fully healthy cluster has four votes, a quorum requirement of three, and all nodes online.

If a node is missing, I investigate its connectivity and cluster state before making changes.

### DNS

On `dns01`:

```bash
pihole status
```

From another host:

```bash
dig @192.168.1.20 example.com
```

The second test matters because it checks that a client can receive an actual DNS response.

### Wazuh

On `wazuh01`, I can check:

```bash
sudo systemctl is-active wazuh-indexer
sudo systemctl is-active wazuh-manager
sudo systemctl is-active filebeat
sudo systemctl is-active wazuh-dashboard
```

The management-host health checks also inspect required Wazuh processes and application endpoints.

An HTTP redirect from the dashboard, or an HTTP 401 from an unauthenticated API request, can be a healthy response when the service is listening and enforcing its expected access policy.

I also check whether monitored endpoints are reporting recent telemetry.

### LibreNMS

The existing health checks cover the LibreNMS web interface and important services such as MariaDB, nginx, PHP-FPM, SNMP, and pollers.

For a more detailed application check on `nms01`:

```bash
cd /opt/librenms
sudo -u librenms ./validate.php
```

### Backup Storage

From Proxmox:

```bash
pvesm status
pvesm list t-20-backup
```

I want to know whether the backup destination is available, not merely whether the Windows storage server is powered on.

### Monitoring Exceptions

Uptime Kuma provides a quick view of whether monitored services are reachable.

For an outage, I check the affected service and its dependencies rather than assuming the availability monitor identifies the root cause.

---

## 4. Ansible Health Checks

I also use Ansible for Linux infrastructure checks.

The documented playbook is:

```bash
ansible-playbook ansible/linux-health.yml
```

The associated `linux-health` command provides another way to run the checks.

These cover host resources and selected services, including Wazuh and LibreNMS.

The Ansible checks help standardize what I inspect across systems and make the results easier to compare.

---

## 5. Weekly Proxmox Review

Once a week, I take a closer look at the cluster.

### Guest Inventory and Resources

```bash
pvesh get /cluster/resources --type vm
```

I review which guests are running, which are intentionally stopped, and whether any node is carrying more workload than expected.

On individual nodes:

```bash
free -h
df -h
```

### Storage Capacity

```bash
pvesm status
```

I review `local`, `local-lvm`, and `t-20-backup`.

As a working guideline, I investigate filesystems approaching 80% utilization and give higher priority to those approaching 90%.

Those are operational review thresholds, not universal product limits.

### LVM-Thin Pools

Proxmox local storage also requires attention to thin-pool data and metadata consumption.

```bash
lvs -a --units g \
  -o vg_name,lv_name,lv_size,data_percent,metadata_percent,seg_monitor
```

```bash
vgs --units g \
  -o vg_name,vg_size,vg_free
```

The documented `pve01` thin-pool auto-extension settings are:

```text
thin_pool_autoextend_threshold=80
thin_pool_autoextend_percent=4
```

These settings provide some protection against running out of capacity, but they aren't a replacement for monitoring actual storage consumption.

---

## 6. Backup Review

The Proxmox backup jobs use shared storage named:

`t-20-backup`

The documented backup schedule is:

| Node | Daily time |
|---|---|
| `pve01` | 02:00 |
| `pve02` | 04:00 |
| `pve03` | 05:00 |
| `pve04` | 06:00 |

The jobs are staggered to avoid placing unnecessary simultaneous load on the shared Windows/USB-backed storage.

The recorded retention policy is:

```text
keep-last=7,keep-weekly=4,keep-monthly=3
```

I can review the configured jobs and storage settings with:

```bash
pvesh get /cluster/backup --output-format yaml
pvesh get /storage/t-20-backup --output-format yaml
```

### What I Review

I check that recent backups exist for the guests that are currently expected to be protected.

I also look for failed jobs, missing archives, unexpected storage growth, and changes in retention.

The original checklist recorded guest IDs:

```text
100 101 105 200 201 210 300 320 400 500
```

That is a historical coverage list, not a guarantee that every listed VMID remains active or should still have a current backup.

### Storage Capacity

The backup storage uses an attached Drobo volume.

Proxmox may report the volume's large logical capacity rather than the actual usable physical capacity.

For capacity planning, I also check the storage hardware's reported health and real available capacity.

---

## 7. Security Monitoring Review

### Wazuh Alerts

I review security events involving:

- Repeated authentication failures
- Unexpected account changes
- File Integrity Monitoring
- Linux privileged commands
- Windows process execution
- PowerShell events
- Active Directory account administration

The question isn't simply whether Wazuh generated alerts.

I want to know whether the activity was expected and whether it needs investigation.

### Endpoint Coverage

Important monitored endpoints include:

`target01`, `win11-01`, and `dc01`.

I check that events are recent, not just that the agents are enrolled.

### Windows and Active Directory

On `dc01`:

```powershell
Get-ADDomain
Get-ADForest
Get-ADDomainController
```

From the domain-joined Windows endpoint:

```powershell
Test-ComputerSecureChannel -Verbose
```

I also verify Active Directory DNS when domain discovery or authentication appears abnormal.

---

## 8. Network Monitoring Review

With the Cisco and UniFi switches deployed, the regular network review includes switch availability, trunk status, interface errors, and VLAN connectivity.

I use LibreNMS and direct switch inspection as appropriate.

For OPNsense, I check interface state, expected routed connectivity, and firewall logs when something fails.

I also want to identify monitoring gaps created by migrating systems from the original flat network to VLANs.

A service address change may require updates to SNMP monitoring, dashboards, alerts, DNS, and remote-access rules.

The default-LAN renumbering remains planned, not completed.

---

## 9. Monthly Maintenance

The monthly review focuses on updates, recovery readiness, account management, and documentation.

### Linux Updates

On an Ubuntu or Debian system:

```bash
sudo apt update
apt list --upgradable
```

I review the impact of updates before applying them to critical systems.

### Proxmox Maintenance

For cluster updates, I prefer working through one node at a time.

I verify backups and cluster health before beginning, then validate each node after its maintenance.

### Active Directory Accounts

On `dc01`:

```powershell
Get-ADUser -Filter * |
Select-Object Name, SamAccountName, Enabled
```

I review whether test accounts, disabled accounts, and administrative memberships are still appropriate.

### Documentation and Inventory

I compare the asset inventory and configuration documents against the systems actually in use.

Particular attention goes to IP addresses, VM placement, planned versus completed work, and changes to service dependencies.

---

## 10. Restore Testing

A backup job completing successfully is useful, but it doesn't establish that a system can be restored.

I periodically want to test a restore and verify the recovered application.

In August 2026, I restored the Docker VM from VMID 210 to a temporary VMID 910.

The restored VM used an isolated network interface.

I confirmed that the guest booted, Docker worked, and Uptime Kuma was operational before removing the temporary VM.

That exercise is documented in [Proxmox Backup and Restore Validation](21-proxmox-backup-restore-validation.md).

Future exercises should continue testing application functionality and documenting the results rather than only confirming that a restore command succeeded.

---

## 11. UPS and Power Recovery

The lab includes several UPS units and uses NUT monitoring through `util01`.

Power events affect more than the hypervisors. They can also affect storage, switching, DNS, and remote administration.

I want to maintain an orderly shutdown sequence and a predictable startup procedure.

Regular review includes UPS communications, battery and load information, shutdown dependencies, and whether critical systems come back correctly after a power interruption.

The procedures are documented in [Shutdown and Startup Runbook](19-shutdown-startup-runbook.md).

---

## 12. Before and After Changes

For changes affecting shared services or network infrastructure, I follow a short checklist.

Before the change:

- Record the current state and affected systems.
- Confirm relevant backups or configuration exports.
- Establish the intended result.
- Identify a rollback path.
- Preserve a management connection that doesn't depend on the change succeeding.

After the change:

- Verify basic connectivity.
- Test the actual service.
- Check monitoring and logging.
- Confirm dependent systems still function.
- Update the inventory and documentation.
- Record any unresolved issues.

This is particularly important for firewall, VLAN, DNS, and storage changes because a successful configuration change on one system can still interrupt something that depends on it.

---

## 13. Daily and Weekly Review Template

```text
Date:

DAILY
[ ] Review automated health results
[ ] Check Proxmox cluster status
[ ] Confirm DNS resolution
[ ] Review Uptime Kuma exceptions
[ ] Confirm critical services
[ ] Confirm backup storage availability
[ ] Check Wazuh/LibreNMS health
[ ] Review significant monitoring alerts

WEEKLY
[ ] Review Proxmox guest placement
[ ] Review node resources
[ ] Review storage and thin pools
[ ] Confirm recent backups
[ ] Review Wazuh agent coverage
[ ] Review network-monitoring exceptions
[ ] Check Windows/AD health
[ ] Review unresolved incidents

Exceptions:

Actions taken:

Verification:

Follow-up:
```

The template is a reference for manual reviews. It doesn't imply that every item has been automated or completed on every date.

---

## 14. Planned Additions

Some future work will require additional monitoring coverage.

That includes expanding SNMPv3 usage, reviewing OPNsense firewall rules, testing more recovery scenarios, and adding checks as new services are deployed.

Zabbix is still a possible future experiment, not an operational dependency.

If I deploy it, I'll first validate a small set of hosts and alerts before incorporating it into regular health checks.

---

## Result

The homelab now has several overlapping monitoring systems, automated health checks, and documented maintenance procedures.

I use them to detect failures, check resource capacity, verify backups, and identify services that stop behaving as expected.

The operational work is ongoing. I expect the checklist to change as I add systems, migrate network services, and improve automation.

The main objective is to make it easier to identify problems early, understand their impact, and verify that the lab is working after maintenance or recovery.

## Related Documentation

- [Asset Inventory](02-asset-inventory.md)
- [Proxmox Cluster](03-proxmox-cluster-build.md)
- [Managed Switching and VLANs](04-network-vlans.md)
- [Wazuh SIEM](06-wazuh-siem.md)
- [NOC Troubleshooting Runbook](13-noc-troubleshooting-runbook.md)
- [Storage and Backup Resilience](17-storage-backup-resilience.md)
- [Shutdown and Startup Runbook](19-shutdown-startup-runbook.md)
- [Backup and Restore Validation](21-proxmox-backup-restore-validation.md)
