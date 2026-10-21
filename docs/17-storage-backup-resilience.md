# Storage and Backup Resilience

## Overview

My Proxmox cluster uses shared backup storage hosted on a Dell PowerEdge T20 running Windows.

The storage server is connected to a Drobo over USB, and the backup directory is made available to Proxmox through SMB/CIFS.

This arrangement works for the lab, but it has also produced some problems that taught me a great deal about backup recovery and storage dependencies.

One incident involved accidentally deleting the directory that contained the Proxmox backup archives. Another occurred after a power outage, when Windows firewall behavior prevented Proxmox from reaching a perfectly healthy SMB share.

I also encountered destination-side write errors when several cluster nodes backed up concurrently.

This document covers the storage configuration, those incidents, the recovery process, and the operational changes made afterward.

---

## 1. Storage Server

The storage host is `storage01` (`192.168.1.208`).

| Component | Configuration |
|---|---|
| Hardware | Dell PowerEdge T20 |
| Processor | Intel Xeon E3-1225 v3, 3.20 GHz |
| Memory | 32 GB |
| BIOS | A20, dated August 19, 2019 |
| Network address | `192.168.1.208` |
| Main function | File storage and Proxmox backups |

The server uses several local and external disks.

The documented layout includes:

| Volume | Storage | Role |
|---|---|---|
| `C:` | 1 TB-class SATA disk | Windows system disk |
| `E:` | Partition on 5 TB-class USB disk | Movies |
| Unmounted | HFS+ partition on same disk | Preserved Apple-formatted data |
| `F:` | 1.5 TB-class USB disk | Secondary storage |
| `G:` | Drobo Gen3 | Shared files and Proxmox backups |

The Drobo volume is the backup destination used by Proxmox.

---

## 2. Drobo Capacity

The Drobo contains two 6 TB drives configured with single-disk protection.

The recorded storage details were:

| Item | Value |
|---|---|
| Raw installed capacity | 12 TB |
| Protected usable capacity | 5.34 TB |
| Logical NTFS volume | 64 TB |
| Firmware | 4.2.3 |
| Connection | USB |

The distinction between logical and physical capacity is important.

The Drobo exposes a large thin-provisioned NTFS volume to Windows.

Windows and Proxmox can therefore report a capacity that greatly exceeds the space physically available on the drives.

For actual capacity planning, I use Drobo Dashboard rather than relying solely on the filesystem size displayed by Windows or Proxmox.

---

## 3. Proxmox Backup Configuration

The Proxmox storage identifier is:

`t-20-backup`

The SMB share is:

`ProxmoxBackups`

The documented connection is:

```text
Proxmox storage: t-20-backup
Windows server:  storage01
Server address:  192.168.1.208
SMB share:       ProxmoxBackups
Proxmox mount:   /mnt/pve/t-20-backup
```

From a Proxmox node, I can check the target with:

```bash
pvesm status
pvesm list t-20-backup
```

These commands tell me whether Proxmox recognizes the target and can access the available backup content.

That is more meaningful than simply confirming that the Windows server is reachable.

---

## 4. Incident: Backup Directory Deleted

During storage cleanup, the directory backing the `ProxmoxBackups` share was permanently removed.

The Proxmox storage configuration still existed, but the backup target was no longer usable.

The deleted directory contained historical Proxmox backup archives.

The running VMs and containers were unaffected.

### Establishing the Scope

Before attempting file recovery, I checked the Proxmox task history across the cluster.

I wanted to know which guests had been backed up recently and whether any workloads existed only inside the deleted backup set.

The latest recorded backup coverage was:

```text
pve01: 100 101 105 200 201 210
pve02: 300 400
pve03: 500
pve04: 320
```

All ten expected guests were still present in the cluster.

An older backup record for VM 320 represented the same guest before it was moved to another node, not an additional unique workload.

The loss was therefore historical recovery points rather than the only copy of a running system.

That didn't make the deletion harmless, but it changed the recovery decision.

I decided to rebuild the backup set rather than attempt deleted-file recovery on the Drobo.

---

## 5. Recreating the SMB Share

On `storage01`, I created a dedicated backup directory on the Drobo volume.

The recovery commands used this pattern in an elevated PowerShell session:

```powershell
$backupPath = 'G:\ProxmoxBackups'
$backupUser = 'storage01\<backup-account>'

New-Item -Path $backupPath -ItemType Directory -Force | Out-Null

icacls $backupPath /inheritance:r

icacls $backupPath /grant:r `
    'NT AUTHORITY\SYSTEM:(OI)(CI)F' `
    'BUILTIN\Administrators:(OI)(CI)F' `
    "\${backupUser}:(OI)(CI)M"

New-SmbShare `
    -Name 'ProxmoxBackups' `
    -Path $backupPath `
    -Description 'Proxmox VE backups' `
    -FullAccess 'BUILTIN\Administrators' `
    -ChangeAccess $backupUser
```

The backup account is represented by a placeholder rather than publishing its actual name or credentials.

The new configuration shares the dedicated backup directory rather than the root of the Drobo volume.

---

## 6. Clearing Stale CIFS Mounts

Recreating the SMB share didn't immediately restore connectivity from Proxmox.

The Linux clients still had stale mount state associated with the original deleted share.

The errors included:

```text
Stale file handle
CIFS: VFS: reconnect tcon failed
```

On the affected Proxmox nodes, I checked the mount state:

```bash
findmnt /mnt/pve/t-20-backup
```

After confirming there were no active backup operations using the target, I detached the stale mount:

```bash
umount /mnt/pve/t-20-backup
```

Then forced Proxmox to access the configured storage again:

```bash
pvesm list t-20-backup
pvesm status
```

I repeated the checks on the relevant nodes because the cluster-wide storage configuration doesn't mean every node shares the same Linux kernel mount state.

After recovery, Proxmox reported the storage as active.

The initial backup list was empty, as expected after recreating the directory.

---

## 7. Rebuilding the Backup Set

I started the cluster-wide backup job manually to create a new recovery set.

Nine guests completed successfully.

The remaining guest, VM 100, failed at the end of its compressed backup stream.

The error was:

```text
zstd: /*stdout*: Resource temporarily unavailable
ERROR: Backup of VM 100 failed
```

Kernel logs also showed CIFS write errors while several nodes were writing concurrently to the storage server.

The recorded error included:

```text
CIFS: VFS: \\192.168.1.208 Error -14 sending data on socket to server
```

Because the target was still active and the other backup jobs had succeeded, I retried only the failed guest.

```bash
vzdump 100 \
    --storage t-20-backup \
    --mode snapshot \
    --compress zstd \
    --notes-template '{{guestname}}'
```

The retry completed successfully.

### Verification

I confirmed that the new backup set contained an archive for each expected guest.

```text
VMID 100  — 1 archive
VMID 101  — 1 archive
VMID 105  — 1 archive
CTID 200  — 1 archive
CTID 201  — 1 archive
VMID 210  — 1 archive
VMID 300  — 1 archive
VMID 320  — 1 archive
VMID 400  — 1 archive
VMID 500  — 1 archive
```

This was the complete recovery set at the time of the incident.

It isn't intended to represent the current guest inventory.

---

## 8. Backup Retention and Scheduling

After the incident, I changed the backup configuration to reduce unnecessary growth and avoid multiple nodes writing to the storage system simultaneously.

### Retention

The recorded retention policy became:

```text
keep-last=7
keep-weekly=4
keep-monthly=3
```

This replaced the earlier indefinite retention arrangement.

The goal was to preserve useful recovery history without allowing backups to grow without limit against the Drobo's finite physical capacity.

### Staggered Jobs

I replaced the all-node schedule with staggered per-node backups.

| Node | Daily start |
|---|---|
| `pve01` | 02:00 |
| `pve02` | 04:00 |
| `pve03` | 05:00 |
| `pve04` | 06:00 |

The spacing reduces simultaneous CIFS and USB write pressure.

It doesn't guarantee that a backup destination can never become overloaded, but it removes the obvious overlap that contributed to the earlier write problems.

---

## 9. Proxmox Thin-Pool Maintenance

While investigating backup problems, I also encountered LVM-thin warnings on `pve01`.

The recorded starting state was:

| Item | Value |
|---|---|
| Thin-pool data utilization | 33.79% |
| Metadata utilization | 1.85% |
| Free volume-group space | 16 GiB |
| LXC 201 root filesystem | 85% used |

I expanded the root filesystem for LXC 201 from 3 GiB to 6 GiB.

That reduced its filesystem usage to approximately 42%.

I also removed obsolete Proxmox VM snapshots.

Afterward, the recorded thin-pool utilization was:

```text
Data:     22.78%
Metadata: 1.18%
```

### Auto-Extension Guard

The documented thin-pool monitoring settings were:

```text
thin_pool_autoextend_threshold=80
thin_pool_autoextend_percent=4
```

The limited volume-group free space constrained how much automatic growth was possible.

I consider those settings an additional safeguard rather than a replacement for capacity reviews.

---

## 10. Incident: SMB Unavailable After Power Outage

A later power outage caused another backup-storage problem.

When `storage01` restarted, the Drobo volume returned normally.

The backup directory still existed, and the SMB service was running.

However, Proxmox reported `t-20-backup` as unavailable.

Uptime Kuma also reported that TCP port 445 on `storage01` was unreachable.

### Local Windows Checks

On the storage server, I verified that:

- The Drobo was mounted as `G:`.
- The `ProxmoxBackups` folder existed.
- The SMB share was still configured.
- `LanmanServer` was running.
- Windows was listening on TCP 445.

The useful PowerShell checks included:

```powershell
Get-Service LanmanServer
Get-SmbShare
Get-NetTCPConnection -LocalPort 445 -State Listen
Get-NetConnectionProfile
```

The SMB configuration appeared healthy locally.

### Finding the Cause

The active Windows Ethernet network had been classified as:

```text
NetworkCategory : Public
```

The firewall policy associated with the Public profile prevented inbound SMB access from the lab network.

This explained why the server and share were functioning locally while remote clients couldn't connect.

### Resolution

I corrected the trusted Ethernet interface to use the Private profile.

The recorded command was:

```powershell
Set-NetConnectionProfile -InterfaceIndex 20 -NetworkCategory Private
```

The interface index was the observed value on `storage01` at that time. It shouldn't be assumed for another machine or configuration.

### Verification

After the change:

- TCP 445 became reachable.
- Uptime Kuma's SMB availability check recovered.
- Proxmox recognized `t-20-backup` again.
- The existing share and storage definition remained intact.

No Drobo or CIFS storage rebuild was required.

This was a network and firewall-profile problem, not a disk failure.

---

## 11. Restore Validation

Rebuilding backups wasn't the end of the recovery work.

On August 26, 2026, I tested an actual restore of the Docker VM.

The source was VMID 210, and the temporary restored guest was VMID 910.

I kept the restored VM isolated from the production network to avoid duplicate IP addresses or other conflicts.

I then checked that Debian booted, the QEMU Guest Agent responded, Docker worked, and the Uptime Kuma container was available.

After verifying the service, I removed the temporary VM and its disks.

The full procedure is recorded in [Proxmox Backup and Restore Validation](21-proxmox-backup-restore-validation.md).

That exercise demonstrated recoverability more convincingly than checking that an archive existed.

---

## 12. Operational Checks

For routine backup and storage monitoring, I focus on a few questions:

1. Can Proxmox access the backup target?
2. Did the scheduled backups actually complete?
3. Are the expected guests represented?
4. Is the physical Drobo capacity sufficient?
5. Are there kernel or CIFS errors?
6. Are local Proxmox thin pools approaching capacity?
7. Have I tested a restore recently?

Useful commands include:

```bash
pvesm status
pvesm list t-20-backup
lvs -a -o lv_name,lv_size,data_percent,metadata_percent
```

When the storage target is inaccessible, I also check the SMB server, Windows network profile, TCP 445 reachability, and mount state before modifying the storage definition.

---

## Result

The storage incidents were frustrating, but they provided useful troubleshooting and recovery experience.

The deleted backup directory required me to establish exactly what had been lost, recreate the share, clear stale mounts, rebuild backups, and verify the new recovery set.

The concurrent write failures led to staggered schedules.

The power-outage incident showed how a Windows network-profile change could interrupt SMB access even when the disks, filesystem, share, and server service were healthy.

Finally, the isolated Docker VM restore demonstrated that an archive could be used to recover a working application.

The overall backup design still has limitations, particularly its dependence on a single Windows server and USB-attached storage. Those are tradeoffs I need to account for as the lab grows.

## Related Documentation

- [Proxmox Backup and Disaster Recovery](01-proxmox-backup.md)
- [Proxmox Cluster Build](03-proxmox-cluster-build.md)
- [Homelab Operations Checklist](15-homelab-operations-checklist.md)
- [NOC Troubleshooting Runbook](13-noc-troubleshooting-runbook.md)
- [Shutdown and Startup Runbook](19-shutdown-startup-runbook.md)
- [Backup and Restore Validation](21-proxmox-backup-restore-validation.md)
