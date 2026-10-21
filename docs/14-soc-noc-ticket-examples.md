# SOC/NOC Incident and Change Ticket Examples

## Overview

This page contains incident and change records based on work I've carried out in my homelab.

I use a ticket-style format to practice documenting operational problems: what happened, what I checked, what I found, what I changed, and whether the result was verified.

These aren't tickets exported from a professional service-management system. They're records I've written from my own troubleshooting and lab notes.

Some describe completed incidents or planned changes that were successfully implemented. A separate section contains example handoffs and templates for future use.

Addresses in historical tickets reflect the systems at the time and may differ from their current VLAN assignments.

---

## Ticket 1 — Wazuh Installation Failure

**Type:** Deployment incident  
**Priority:** P3  
**Historical address:** `wazuh01` — `192.168.1.206`  
**Status:** Resolved

### Problem

The Wazuh installation assistant failed while updating OpenSearch security-user configuration.

The initial error suggested an authentication problem.

### Investigation

I reviewed the installer log:

```bash
sudo tail -n 80 /var/log/wazuh-install.log
```

Then checked filesystem capacity and LVM:

```bash
df -h /
lsblk
sudo pvs
sudo vgs
sudo lvs
```

The Ubuntu root logical volume wasn't using all the space available on the virtual disk.

OpenSearch had crossed its flood-stage disk watermark and placed its security index into read-only mode.

### Resolution

I expanded the root logical volume:

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

The filesystem problem was corrected, but the incomplete installation had left authentication settings inconsistent.

Because the VM contained no production data or custom Wazuh rules, I rebuilt it cleanly.

### Verification

The rebuilt Wazuh installation completed, the four main services were active, and dashboard authentication succeeded.

**Follow-up:** Check guest filesystem capacity before installing storage-intensive applications.

---

## Ticket 2 — Wazuh Indexer Access-Denied Error

**Type:** Service incident  
**Priority:** P3  
**Service:** `wazuh-indexer`  
**Status:** Resolved during deployment recovery

### Problem

The Wazuh indexer failed to start during recovery of the first installation.

### Investigation

```bash
sudo systemctl status wazuh-indexer --no-pager -l
sudo journalctl -xeu wazuh-indexer.service --no-pager
```

The service journal showed:

```text
java.nio.file.AccessDeniedException:
/etc/wazuh-indexer/backup
```

### Resolution

I corrected the affected backup-directory permissions and restarted the service.

### Verification

The indexer started and was listening on TCP port 9200.

Further recovery work showed that the overall installation still had credential problems, leading to the clean rebuild described in Ticket 1.

---

## Ticket 3 — Pi-hole Migration

**Type:** Planned infrastructure change  
**Priority:** P4  
**Service:** DNS  
**Status:** Completed

### Objective

Move Pi-hole from a Proxmox LXC container to a physical Raspberry Pi so Proxmox maintenance wouldn't interrupt household DNS.

### Change

I prepared a Raspberry Pi, installed Pi-hole, and assigned the existing DNS service address to the new system.

```text
dns01
192.168.1.20
```

The original Pi-hole LXC was removed from DNS service after the replacement was working.

### Verification

I confirmed the Pi-hole service was active, filtering was enabled, and clients could continue resolving hostnames.

Keeping the same IP address avoided a broad client DNS reconfiguration.

**Result:** Successful change.

---

## Ticket 4 — Initial Proxmox Cluster Validation

**Type:** Infrastructure change  
**Priority:** P4  
**Cluster:** `homelab`  
**Status:** Completed

### Objective

Create and validate the initial three-node Proxmox cluster.

| Node | IP address |
|---|---|
| `pve01` | `192.168.1.10` |
| `pve02` | `192.168.1.11` |
| `pve03` | `192.168.1.12` |

### Validation

I checked cluster membership, quorum, node state, and storage.

```bash
pvecm status
pvecm nodes
pvesh get /nodes
pvesm status
```

The recorded cluster state was:

```text
Nodes:          3
Expected votes: 3
Total votes:    3
Quorum:         2
Quorate:        Yes
```

### Result

All three nodes were recognized and the cluster was quorate.

The cluster was later expanded to four nodes, as documented separately.

---

## Ticket 5 — Windows 11 Domain Join

**Type:** Infrastructure change  
**Priority:** P4  
**System:** `win11-01`  
**Domain:** `corp.home.arpa`  
**Status:** Completed

### Objective

Join the Windows endpoint to Active Directory and verify domain connectivity.

At the time of the initial domain work, `dc01` was recorded at `192.168.1.30`. Its later VLAN 20 address is `10.10.20.10`.

### Change and Validation

I joined the workstation to `corp.home.arpa` and restarted it.

I then checked the domain secure channel:

```powershell
Test-ComputerSecureChannel -Verbose
```

The secure-channel check succeeded.

**Result:** Successful domain join.

---

## Ticket 6 — Active Directory Account Creation Alert

**Type:** Security detection exercise  
**Priority:** P5 — Controlled test  
**System:** `dc01`  
**Status:** Closed — expected activity

### Event

I created a test Active Directory user named `soc-lab-user`.

Windows generated Security Event ID `4720`.

Wazuh generated:

| Field | Value |
|---|---|
| Rule | `60109` |
| Level | 8 |
| MITRE ATT&CK | `T1098` |

### Validation

On the domain controller:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4720
} -MaxEvents 20
```

I also inspected the test account:

```powershell
Get-ADUser soc-lab-user -Properties *
```

The Active Directory event matched the Wazuh detection.

**Classification:** True positive — expected.

---

## Ticket 7 — Active Directory Password-Policy Failure

**Type:** Administrative issue  
**Priority:** P4  
**Status:** Resolved

### Problem

A test-account password operation failed because the supplied password did not satisfy the domain policy.

### Investigation

```powershell
Get-ADDefaultDomainPasswordPolicy |
Select-Object MinPasswordLength,
              ComplexityEnabled,
              PasswordHistoryCount,
              MinPasswordAge,
              MaxPasswordAge
```

The policy required password complexity, tracked 24 previous passwords, and specified minimum and maximum ages of one and 42 days.

### Resolution

I entered a compliant password as a SecureString and reset the account password.

```powershell
$pw = Read-Host "Enter a NEW compliant password for soc-lab-user" -AsSecureString

Set-ADAccountPassword `
    -Identity "soc-lab-user" `
    -Reset `
    -NewPassword $pw
```

The command completed successfully.

**Result:** Resolved.

---

## Tickets 8–10 — Controlled Wazuh Detection Exercises

These three tickets represent separate tests. Their complete commands and evidence are retained in the [Wazuh Detection Labs](09-attack-detection-labs.md).

| Ticket | Activity | Detection | Result |
|---|---|---|---|
| 8 | Eight failed SSH logins from `kali01` to `target01` | Wazuh `5712`, Level 10 | Verified against SSH journal |
| 9 | Configuration-file modification on `target01` | Wazuh `550`, Level 7 | Hashes and content changes captured |
| 10 | SQL-injection-style request to Apache | Wazuh `31103`, Level 7 | Access-log entry verified; HTTP 404 |

All three were classified as **true positives involving expected lab activity**.

The web request did not demonstrate successful exploitation.

---

## Ticket 11 — Wazuh Agent Connectivity

**Type:** Diagnostic template  
**Priority:** Assigned according to impact  
**Status:** Template — not a separately verified incident

If an endpoint stops appearing in Wazuh, the initial checks include:

```bash
systemctl is-active wazuh-agent
nc -zv 10.10.40.20 1514
nc -zv 10.10.40.20 1515
```

I would then check the manager address, firewall path, enrollment state, agent logs, and whether fresh telemetry is arriving.

The original document included this as a ticket example. I'm keeping it as a reusable diagnostic template rather than presenting it as an incident with an established resolution.

---

## Ticket 12 — Proxmox Backup Target Validation

**Type:** Maintenance validation  
**Priority:** P4  
**Storage:** `t-20-backup`  
**Status:** Completed

### Objective

Confirm that the shared Proxmox backup destination was available before rebuild work.

### Checks

```bash
pvesm status
pvesm list t-20-backup
```

The backup target was active, and Proxmox could list its contents.

The backup and restore process was later tested more extensively, including restoring a Docker VM into an isolated temporary guest.

**Result:** Shared backup target verified.

---

## Ticket 13 — Guacamole RDP Connectivity

**Type:** Application-connectivity incident  
**Priority:** P4  
**Target:** Windows RDP — `192.168.1.189:3389`  
**Status:** Resolved

### Problem

Apache Guacamole initially couldn't establish an RDP session.

### Investigation

From the Guacamole system:

```bash
nc -vz 192.168.1.189 3389
```

The TCP port was reachable.

That suggested basic network connectivity wasn't the cause.

### Result

The Guacamole RDP connection was subsequently working.

The original notes don't preserve the exact application-level setting that corrected the problem, so I won't assign a more specific root cause.

---

## Ticket 14 — Storage Server SMB Outage After Power Loss

**Type:** Infrastructure incident  
**Priority:** P3  
**System:** `storage01` — `192.168.1.208`  
**Service:** Proxmox backup storage  
**Status:** Resolved

### Problem

After a power outage, `storage01` rebooted and its attached Drobo storage returned normally.

However, Proxmox couldn't reach the CIFS backup target, and Uptime Kuma reported TCP port 445 as unavailable.

### Investigation

On `storage01`, I confirmed that the storage volume was mounted, the `ProxmoxBackups` SMB share existed, `LanmanServer` was running, and Windows was listening on TCP 445.

From `pve01`, I tested network connectivity:

```bash
ping -c 3 192.168.1.208
nc -nvz -w 3 192.168.1.208 445
```

The Proxmox host couldn't reach the SMB service.

I then checked the Windows network profile:

```powershell
Get-NetConnectionProfile
```

The Ethernet connection had been classified as Public.

### Root Cause

The Windows Public-network firewall profile was blocking inbound access to SMB after the reboot.

### Resolution

I changed the trusted Ethernet connection back to Private.

The recorded command was:

```powershell
Set-NetConnectionProfile -InterfaceIndex 20 -NetworkCategory Private
```

The interface index was specific to the server at that time.

### Verification

SMB connectivity returned, Uptime Kuma recovered, and Proxmox recognized `t-20-backup` again.

No storage definition or SMB share needed to be rebuilt.

**Result:** Resolved.

---

## Ticket 15 — Cisco SG350-10 Preparation and Deployment

**Type:** Planned infrastructure change  
**Priority:** P4  
**System:** `sw01` — `192.168.1.21`  
**Status:** Initial deployment completed; VLAN configuration expanded afterward

### Objective

Prepare the Cisco SG350-10 for use as the managed switch in the homelab.

### Initial Work

I assigned a management address, configured the hostname, inspected firmware, checked the original VLAN database, and saved a baseline configuration.

The recorded firmware state was:

```text
Active:   2.5.9.55
Inactive: 2.5.0.83
```

Useful CLI checks included:

```text
show version
show vlan
```

The original switch configuration used VLAN 1.

### Subsequent Changes

The switch was later connected to the Proxmox secondary network interfaces, and trunk ports were configured for the lab VLANs.

The current documented topology also includes the UniFi US-8 connected through `sw01`.

The original staging note is retained as historical evidence. The managed-switch deployment is no longer awaiting physical cutover.

[Current switching configuration](04-network-vlans.md)

---

## Example Escalation Notes

The following two cases are **illustrative handoff examples**, not claims of additional completed incidents.

### Example A — Proxmox Node Unavailable

**Priority:** P2  
**Affected system:** `pve02` (`192.168.1.11`)

Reported problem: Node unreachable from the management network.

An appropriate handoff would document host reachability, gateway tests, switch-port state, Proxmox cluster membership, quorum, and affected workloads.

The next administrator should be able to continue from the evidence already collected.

### Example B — Unexpected Domain Account Creation

**Priority:** P2  
**System:** `dc01`

A Windows Security Event 4720 appears outside an expected maintenance period.

An appropriate investigation would identify the creating administrator, new account, timestamp, privileged group membership, related logons, and PowerShell or process activity.

If evidence suggested unauthorized account creation, the case would be escalated under the applicable incident-response procedures.

---

## Ticket Writing Template

```text
Ticket ID:
Type:
Priority:
Status:
Date / time:

Affected system:
Affected service:
Impact:

Reported symptom:

Initial observations:

Checks performed:

Evidence:

Finding / root cause:

Change made:

Validation:

Rollback (if applicable):

Follow-up:

Final status:
```

I want each ticket to leave enough information for someone else to understand what happened without reproducing the entire investigation.

Where the root cause is unknown, I record that explicitly rather than inventing a resolution.

---

## Result

The ticket examples cover several kinds of work I've carried out: infrastructure changes, Linux service failures, storage outages, Windows administration, network troubleshooting, and security detection exercises.

The most useful entries are those where I can show a complete sequence from symptom to investigation, corrective action, and successful validation.

The tickets complement the longer technical documents rather than replace them.

## Related Documentation

- [Wazuh SIEM Deployment](06-wazuh-siem.md)
- [Detection Labs](09-attack-detection-labs.md)
- [Active Directory Lab](10-active-directory-lab.md)
- [NOC Troubleshooting Runbook](13-noc-troubleshooting-runbook.md)
- [Managed Switching and VLANs](04-network-vlans.md)
- [Backup and Restore Validation](21-proxmox-backup-restore-validation.md)
