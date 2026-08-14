# SOC/NOC Incident and Change Ticket Examples

## Purpose

This document contains realistic incident, service, and change-ticket examples
based on work actually performed in the SOC/NOC homelab.

The objective is to demonstrate how technical work can be communicated in an
operations environment where another analyst, administrator, or manager may need
to understand:

- what was reported;
- what was affected;
- what evidence was collected;
- what was changed;
- whether service was restored;
- what remains to be done.

These examples are written in a concise ticket style rather than as long-form
lab reports.

They are not intended to imply experience with a specific commercial ticketing
platform.

---

# Ticket 1 — Wazuh Installation Failure Due to Disk Allocation

## Type

Incident / deployment failure

## Priority

P3 — Medium

## Affected System

```text
wazuh01
192.168.1.206
```

## Summary

Wazuh installation failed during the final OpenSearch security-user
configuration stage.

## Reported / Observed Symptom

The Wazuh installation assistant deployed the major components but did not
complete successfully.

The dashboard and indexer later showed additional operational problems.

## Initial Checks

Reviewed the installation log:

```bash
sudo tail -n 80 /var/log/wazuh-install.log
```

Checked filesystem and storage allocation:

```bash
df -h /
lsblk
sudo pvs
sudo vgs
sudo lvs
```

## Findings

The Proxmox VM had been assigned a larger virtual disk than the Ubuntu root
logical volume was actually using.

OpenSearch crossed its flood-stage disk watermark and placed the security index
into read-only mode.

The installer then failed while attempting to update internal security
configuration.

## Root Cause

Ubuntu LVM had not allocated all available virtual-disk capacity to the root
logical volume.

## Corrective Action

Expanded the root logical volume:

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

Verified new filesystem capacity:

```bash
df -h /
```

Additional recovery attempts exposed inconsistent credential state after the
interrupted installation.

Because the environment was new and contained no production data or custom
rules, the VM was rebuilt cleanly rather than continuing to repair a partially
configured deployment.

## Validation

After rebuild:

- hostname verified;
- networking verified;
- Pi-hole DNS verified;
- root filesystem capacity verified;
- QEMU guest agent verified;
- Wazuh installation completed;
- Wazuh core services reported active;
- dashboard authentication succeeded.

## Final Status

Resolved.

## Preventive / Follow-Up Action

Added guest-filesystem capacity validation to the Wazuh deployment checklist.

---

# Ticket 2 — Wazuh Indexer Service Failure

## Type

Incident

## Priority

P3 — Medium

## Affected Service

```text
wazuh-indexer
```

## Summary

Wazuh indexer failed to start during recovery of the first Wazuh deployment.

## Investigation

Checked detailed service state:

```bash
sudo systemctl status wazuh-indexer --no-pager -l
```

Reviewed the systemd journal:

```bash
sudo journalctl -xeu wazuh-indexer.service --no-pager
```

## Finding

The service log showed:

```text
java.nio.file.AccessDeniedException:
/etc/wazuh-indexer/backup
```

## Root Cause

Incorrect filesystem permissions on an installer-created backup directory
prevented the indexer from accessing required files.

## Corrective Action

Corrected permissions on the affected directory and restarted the service.

## Validation

Confirmed:

- indexer service started;
- indexer listened on TCP port 9200;
- recovery work could continue.

## Final Status

Resolved as part of the broader Wazuh deployment recovery.

---

# Ticket 3 — Pi-hole Migration / DNS Service Change

## Type

Planned change

## Priority

P4 — Low / planned maintenance

## Change Objective

Move Pi-hole DNS from a Proxmox-hosted LXC container to a dedicated Raspberry Pi
without requiring broad client DNS changes.

## Existing Service Address

```text
192.168.1.20
```

## New Host

```text
dns01
Raspberry Pi 3 Model B+
192.168.1.20
```

## Reason for Change

DNS was a critical home-network dependency hosted inside the virtualization
environment.

Moving DNS to a physical Raspberry Pi reduces the risk that Proxmox host
maintenance or cluster changes disrupt DNS for the rest of the network.

## Pre-Change Checks

- Raspberry Pi configured and updated.
- Hostname changed to `dns01`.
- Pi-hole installed.
- DNS service verified.
- Port 53 confirmed available.
- Pi-hole filtering confirmed enabled.

## Change Performed

The dedicated Raspberry Pi was assigned the existing Pi-hole service address:

```text
192.168.1.20
```

Clients were cut over only after the replacement service was operational.

The old Pi-hole LXC was removed from DNS service afterward.

## Validation

Confirmed:

- `dns01` reachable;
- Pi-hole running;
- filtering enabled;
- normal DNS resolution continued;
- Internet access continued;
- client-side DNS address changes were not required.

## Rollback Plan

If the new Pi-hole failed during cutover, restore DNS service to the previous
known-good platform before retiring the old instance.

## Final Status

Successful.

---

# Ticket 4 — Proxmox Three-Node Cluster Validation

## Type

Infrastructure change / implementation

## Priority

P4 — Planned

## Change Objective

Create and validate the primary three-node Proxmox VE cluster.

## Cluster

```text
homelab
```

## Nodes

```text
pve01   192.168.1.10
pve02   192.168.1.11
pve03   192.168.1.12
```

## Validation Performed

Checked cluster status:

```bash
pvecm status
```

Checked membership:

```bash
pvecm nodes
```

Checked node state:

```bash
pvesh get /nodes
```

Checked storage:

```bash
pvesm status
```

## Result

Verified:

```text
Nodes:          3
Expected votes: 3
Total votes:    3
Quorum:         2
Quorate:        Yes
Transport:      knet
Secure auth:    on
```

All three nodes appeared online.

## Storage State

Validated:

```text
local
local-lvm
t-20-backup
```

## Risk / Design Note

Ceph and Proxmox HA automatic workload failover were intentionally not deployed
during this phase.

The goal was first to establish a stable, understandable cluster with local
storage and verified shared backups.

## Final Status

Successful.

---

# Ticket 5 — Windows Endpoint Domain Join

## Type

Service request / infrastructure change

## Priority

P4 — Low / planned

## Affected Endpoint

```text
win11-01
```

## Target Domain

```text
corp.home.arpa
```

## Domain Controller

```text
dc01
192.168.1.30
```

## Objective

Join the Windows 11 endpoint to the lab Active Directory domain and validate
domain connectivity.

## Pre-Change Validation

Verified:

- endpoint networking;
- DNS configuration;
- domain-controller resolution;
- Active Directory DNS availability;
- domain-controller discovery.

## Change

Joined `win11-01` to:

```text
corp.home.arpa
```

## Post-Change Validation

Verified the computer secure channel:

```powershell
Test-ComputerSecureChannel -Verbose
```

## Result

Secure-channel validation succeeded.

## Final Status

Successful.

---

# Ticket 6 — Active Directory Test User Creation Alert

## Type

Security alert

## Priority

P5 — Controlled lab test

## Alert

```text
Windows Event ID: 4720
Wazuh Rule:       60109
Level:            8
MITRE ATT&CK:     T1098 - Account Manipulation
```

## Affected System

```text
dc01
```

## Affected Account

```text
soc-lab-user
```

## Summary

Wazuh generated an identity-related security alert after a controlled Active
Directory user account was created.

## Validation

Queried Windows Security logs:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id      = 4720
} -MaxEvents 20
```

Inspected Active Directory account state:

```powershell
Get-ADUser soc-lab-user -Properties *
```

## Findings

The Windows Security event and Active Directory account state matched the Wazuh
alert.

## Classification

```text
True Positive — Expected
```

The account creation was intentionally generated as part of the lab.

## Analyst Assessment

The test validated the complete telemetry path:

```text
AD administrative action
        ↓
Windows Security Event
        ↓
Wazuh collection
        ↓
Rule 60109
        ↓
MITRE T1098 mapping
        ↓
Analyst investigation
```

## Final Status

Closed — expected test activity.

---

# Ticket 7 — Active Directory Password Policy Failure

## Type

Service request / administrative issue

## Priority

P4 — Low

## Summary

Initial test-account password operation did not complete because the supplied
password did not satisfy domain policy.

## Investigation

Reviewed the default domain password policy:

```powershell
Get-ADDefaultDomainPasswordPolicy |
Select-Object MinPasswordLength,
              ComplexityEnabled,
              PasswordHistoryCount,
              MinPasswordAge,
              MaxPasswordAge
```

## Observed Policy

```text
MinPasswordLength    : 7
ComplexityEnabled    : True
PasswordHistoryCount : 24
MinPasswordAge       : 1 day
MaxPasswordAge       : 42 days
```

## Root Cause

The supplied test password did not meet domain password requirements.

## Corrective Action

Entered a new compliant password securely:

```powershell
$pw = Read-Host "Enter a NEW compliant password for soc-lab-user" -AsSecureString
```

Reset the password:

```powershell
Set-ADAccountPassword `
    -Identity "soc-lab-user" `
    -Reset `
    -NewPassword $pw
```

## Validation

The command completed successfully and the account remained manageable in
Active Directory.

## Final Status

Resolved.

---

# Ticket 8 — SSH Brute-Force Detection

## Type

Security alert

## Priority

P5 — Controlled test

## Source

```text
kali01
```

## Target

```text
target01
192.168.1.238
```

## Alert

```text
Wazuh Rule: 5712
Level:      10
MITRE:      T1110 - Brute Force
```

## Summary

Repeated failed SSH authentication attempts generated a Wazuh brute-force
alert.

## Source Validation

Reviewed the endpoint SSH log:

```bash
sudo journalctl -u ssh
```

## Findings

The underlying SSH events matched the SIEM alert.

## Classification

```text
True Positive — Expected
```

## Analyst Assessment

The test demonstrated that Wazuh detected repeated authentication failures and
that the alert could be independently verified against the source endpoint log.

## Final Status

Closed — controlled test successful.

---

# Ticket 9 — File Integrity Monitoring Alert

## Type

Security alert

## Priority

P5 — Controlled test

## Target

```text
target01
```

## Alert

```text
Wazuh Rule: 550
Level:      7
MITRE:      T1565.001 - Stored Data Manipulation
```

## Summary

A monitored configuration file was modified during a controlled FIM exercise.

## Test Path

```text
/opt/wazuh-lab/test.conf
```

## Validation

Wazuh recorded:

- file modification;
- size change;
- modification time;
- MD5 hash;
- SHA1 hash;
- SHA256 hash;
- content difference.

## Findings

The FIM event accurately reflected the controlled file modification.

## Classification

```text
True Positive — Expected
```

## Final Status

Closed — detection validated.

---

# Ticket 10 — SQL-Injection-Style Web Request Alert

## Type

Security alert

## Priority

P5 — Controlled test

## Source

```text
kali01
```

## Target

```text
target01
Apache
```

## Alert

```text
Wazuh Rule: 31103
Level:      7
MITRE:      T1190
```

## Test Activity

A controlled request containing SQL-style syntax was sent to the Apache server.

## Source Validation

Checked:

```bash
/var/log/apache2/access.log
```

## Findings

The suspicious request was present in the source Apache log.

The server returned:

```text
HTTP 404
```

because the requested application path did not exist.

## Analyst Assessment

Suspicious request detected.

Successful exploitation was **not** demonstrated.

## Classification

```text
True Positive — Expected
```

## Final Status

Closed — detection validated.

---

# Ticket 11 — Wazuh Agent Connectivity Check

## Type

Incident / monitoring issue

## Priority

P3 — Medium

## Symptom

Endpoint does not appear active in Wazuh or telemetry stops arriving.

## Initial Checks

On the Linux endpoint:

```bash
systemctl is-active wazuh-agent
```

Check manager ports:

```bash
nc -zv 192.168.1.206 1514
nc -zv 192.168.1.206 1515
```

## Investigation Areas

- agent service state;
- network connectivity;
- manager IP;
- enrollment;
- firewall path;
- Wazuh manager availability;
- endpoint time;
- DNS if hostnames are used.

## Escalation Condition

Escalate or broaden investigation if:

- service is active;
- required ports are reachable;
- configuration is correct;
- but telemetry still does not arrive.

## Final Status

Template for future monitoring incidents.

---

# Ticket 12 — Proxmox Backup Target Verification

## Type

Maintenance / backup validation

## Priority

P4 — Planned

## Objective

Verify the remote Proxmox backup target before performing destructive rebuild
work.

## Storage

```text
t-20-backup
```

## Checks

```bash
pvesm status
```

Then:

```bash
pvesm list t-20-backup
```

## Purpose

Confirm that Proxmox itself can:

- see the storage;
- report it active;
- enumerate existing backup content.

## Validation

Backup content was visible through Proxmox storage tools.

Restore testing was also performed during the backup/recovery phase.

## Final Status

Successful.

---

# Ticket 13 — Guacamole RDP Connectivity Issue

## Type

Incident

## Priority

P4 — Low

## Symptom

Apache Guacamole could not initially connect to a Windows 11 RDP target.

## Target

```text
192.168.1.189
TCP 3389
```

## Validation

From the Guacamole host:

```bash
nc -vz 192.168.1.189 3389
```

## Result

TCP 3389 was reachable.

## Interpretation

The successful TCP test showed that:

- the network path was working;
- the destination was reachable;
- RDP was listening.

Therefore, the remaining issue was above basic network connectivity and needed
to be investigated in the Guacamole/RDP configuration.

## Final Status

Resolved; Guacamole RDP connection later worked.

## Lesson

Port reachability is a useful layer test but does not by itself prove the
application session will succeed.

---

# Ticket 14 — Example NOC Escalation Note

## Incident

Proxmox node unavailable from management network.

## Priority

P2 — High

## Affected System

```text
pve02
192.168.1.11
```

## User / Service Impact

One virtualization node unavailable. Cluster workload impact not yet confirmed.

## Checks Completed

- Ping to node failed.
- Other cluster nodes reachable.
- Default gateway reachable.
- `pvecm status` checked from surviving node.
- Cluster quorum checked.
- `pvecm nodes` used to inspect membership.
- Storage state reviewed.

## Current Finding

Issue appears isolated to `pve02`; broader network remains functional.

## Current Cluster State

Record:

```text
Nodes visible:
Votes present:
Quorate:
Affected guests:
```

## Recommended Next Action

Check:

- physical power;
- Ethernet link;
- switch port;
- host console;
- Proxmox boot state;
- NIC configuration.

## Handoff Quality Standard

The receiving administrator should not need to repeat the basic network and
cluster-state checks already completed.

---

# Ticket 15 — Example SOC Escalation Note

## Incident

Unexpected Active Directory account creation.

## Priority

P2 — High

## Alert

```text
Event ID: 4720
Wazuh Rule: 60109
```

## Affected System

```text
dc01
```

## Initial Findings

A new domain account was created outside the expected maintenance window.

## Analyst Checks

- Confirmed Event 4720 on domain controller.
- Identified creating administrator account.
- Identified created account.
- Checked creation timestamp.
- Checked nearby authentication activity.
- Checked for additional account-management events.
- Checked PowerShell/process telemetry where available.

## Escalation Reason

Unauthorized account creation may indicate:

- compromised administrative credentials;
- persistence;
- identity abuse;
- privilege escalation.

## Recommended Next Action

- Disable suspicious account if authorized by procedure.
- Review privileged group membership.
- Review creating administrator's logons.
- Check endpoint/process telemetry.
- Search for related account events.
- Preserve evidence and escalate to incident response.

---

# Ticket Writing Guidelines

A useful ticket should answer:

```text
What happened?
What was affected?
What did I check?
What did I find?
What did I change?
Did it work?
What remains?
```

Avoid entries such as:

```text
"Restarted server. Fixed."
```

A better entry is:

```text
Wazuh indexer failed to start. systemctl/journal review showed
AccessDeniedException for /etc/wazuh-indexer/backup. Corrected directory
permissions, restarted wazuh-indexer, and verified the service active and
listening on TCP 9200.
```

The second note preserves the diagnostic value of the work.

---

# Common Ticket Fields

A generic operational ticket can use:

```text
Ticket Type:
Priority:
Status:
Opened:
Resolved:

Affected Service:
Affected Systems:
User / Business Impact:

Summary:

Observed Behavior:

Expected Behavior:

Checks Performed:

Evidence:

Root Cause:

Corrective Action:

Validation:

Rollback:

Follow-Up:

Final Resolution:
```

---

# Status Vocabulary

Useful status values include:

| Status | Meaning |
|---|---|
| Open | Work has not started |
| Investigating | Evidence collection / diagnosis in progress |
| Monitoring | Change made; watching for recurrence |
| Pending | Waiting on dependency, approval, hardware, vendor, or user |
| Resolved | Service restored / problem corrected |
| Closed | Work complete and documented |
| Planned | Approved or intended change not yet started |

---

# Priority Vocabulary

| Priority | Typical Meaning |
|---|---|
| P1 | Critical / broad outage / severe security impact |
| P2 | High / important service or security risk |
| P3 | Medium / limited service impact |
| P4 | Low / noncritical issue or planned operational work |
| P5 | Lab test / informational |

---

# Skills Demonstrated

These ticket examples demonstrate:

## SOC

- Alert documentation
- Source-event validation
- Security-event classification
- MITRE ATT&CK context
- Identity-event investigation
- Detection verification
- Escalation notes

## NOC

- Incident documentation
- Service-impact assessment
- DNS troubleshooting
- Cluster validation
- Storage validation
- Service restoration
- Operational handoff

## Systems Administration

- Linux service troubleshooting
- LVM/storage diagnosis
- Proxmox administration
- Active Directory administration
- Windows event-log investigation
- Backup validation

## Professional Operations

- Clear technical writing
- Root-cause documentation
- Change control
- Validation
- Rollback thinking
- Escalation readiness
- Evidence preservation

---

# Related Repository Documentation

| Topic | Document |
|---|---|
| Backup / recovery | [`01-proxmox-backup.md`](01-proxmox-backup.md) |
| Asset inventory | [`02-asset-inventory.md`](02-asset-inventory.md) |
| Pi-hole migration | [`02-pihole-migration.md`](02-pihole-migration.md) |
| Proxmox cluster | [`03-proxmox-cluster-build.md`](03-proxmox-cluster-build.md) |
| Wazuh deployment | [`06-wazuh-siem.md`](06-wazuh-siem.md) |
| Windows telemetry | [`08-windows-telemetry.md`](08-windows-telemetry.md) |
| Attack / detection labs | [`09-attack-detection-labs.md`](09-attack-detection-labs.md) |
| Active Directory | [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| SOC/NOC skills matrix | [`11-soc-noc-skills-matrix.md`](11-soc-noc-skills-matrix.md) |
| SOC triage playbook | [`12-soc-alert-triage-playbook.md`](12-soc-alert-triage-playbook.md) |
| NOC troubleshooting runbook | [`13-noc-troubleshooting-runbook.md`](13-noc-troubleshooting-runbook.md) |

---

# Summary

The technical work in a SOC or NOC is only part of the job.

A useful operator must also be able to leave behind a clear record of:

```text
symptom
  ↓
evidence
  ↓
diagnosis
  ↓
action
  ↓
validation
  ↓
handoff / closure
```

These examples demonstrate how the homelab work can be translated into concise,
operations-style ticket documentation.
