# Storage and Backup Resilience

## Purpose

This document records the verified `storage01` platform and attached-storage
layout, a real Proxmox backup-target incident, the evidence used to determine
scope, the recovery steps, and the operational changes made afterward.

The incident demonstrates an important recovery principle:

> Restore service from verified evidence, validate the complete recovery set,
> and change the design so the same failure is less likely to recur.

No passwords, device serial numbers, or personal directory contents are
included.

---

# 1. Verified Storage Host

| Component | Verified value |
|---|---|
| Hardware | Dell PowerEdge T20 |
| CPU | Intel Xeon E3-1225 v3 at 3.20 GHz |
| RAM | 32 GB |
| BIOS | A20, dated 2019-08-19 |
| TPM | Not present |
| Secure Boot | Supported by UEFI but disabled during inspection |
| Address | `192.168.1.208` |
| Current role | SMB media/file storage and Proxmox backup target |

---

# 2. Attached Storage

| Volume | Device / layout | Filesystem | Role |
|---|---|---|---|
| `C:` | 1 TB-class SATA disk; 0.91 TiB visible | NTFS | Current system disk |
| `E:` | 2.27 TiB partition on a 5 TB-class USB disk | NTFS | `MOVIES` |
| Hidden | 2.27 TiB partition on the same USB disk | HFS+ | Preserved Apple-format data partition |
| `F:` | 1.5 TB-class USB disk; 1.36 TiB visible | NTFS | Secondary external storage |
| `G:` | Drobo Gen3 | NTFS | `T20-NAS` and Proxmox backup storage |

The Drobo contains two 6 TB drives with single-disk protection. Its 64 TB NTFS
volume is thin provisioned and does not represent installed capacity.

```text
Installed raw capacity:       12 TB (10.91 TiB actual)
Protected usable capacity:   5.34 TB
Used after backup rebuild:   232.77 GB
Health:                      Good
Firmware:                    4.2.3
Interface:                   USB
```

Drobo Dashboard is the authoritative capacity source. Windows and Proxmox see
the logical 64 TB volume and can therefore report misleading free-space values.

---

# 3. Incident Summary

The directory backing the `ProxmoxBackups` SMB share was permanently removed
during storage cleanup. Proxmox still retained the `t-20-backup` configuration,
but the target became unavailable:

```text
Storage: t-20-backup
Server:  192.168.1.208
Share:   ProxmoxBackups
Path:    /mnt/pve/t-20-backup
Type:    cifs
```

The lost directory contained backup history. The live guests remained intact.

## Scope Determination

Before attempting deleted-file recovery or creating new data on the Drobo,
Proxmox task history was reviewed across all four nodes. The most recent job
logs showed successful backups for the ten guests that still existed in the
cluster:

```text
pve01: 100 101 105 200 201 210
pve02: 300 400
pve03: 500
pve04: 320
```

An older backup history entry for VM 320 corresponded to the same guest before
its move to `pve04`, not a deleted unique guest. Because every backed-up guest
still existed and was operational, deleted-file recovery was not pursued. The
accepted loss was historical restore points, not unique workload data.

---

# 4. Windows Share Recovery

A new restricted directory was created on the Drobo volume. In elevated
PowerShell, the recovery pattern was:

```powershell
$backupPath = 'G:\ProxmoxBackups'
$backupUser = 'storage01\<backup-account>'

New-Item -Path $backupPath -ItemType Directory -Force | Out-Null

icacls $backupPath /inheritance:r

icacls $backupPath /grant:r `
    'NT AUTHORITY\SYSTEM:(OI)(CI)F' `
    'BUILTIN\Administrators:(OI)(CI)F' `
    "${backupUser}:(OI)(CI)M"

New-SmbShare `
    -Name 'ProxmoxBackups' `
    -Path $backupPath `
    -Description 'Proxmox VE backups' `
    -FullAccess 'BUILTIN\Administrators' `
    -ChangeAccess $backupUser
```

This avoids sharing the entire Drobo root and limits the backup account to
modify access on the backup directory.

---

# 5. Clearing Stale CIFS Mounts

Recreating the Windows share did not immediately restore Proxmox access. The
Linux kernel still held the deleted share as a stale CIFS mount:

```text
ls: cannot access '/mnt/pve/t-20-backup': Stale file handle
CIFS: VFS: reconnect tcon failed
```

The stale mount was detached on each node, then storage access was forced:

```bash
umount /mnt/pve/t-20-backup
pvesm list t-20-backup
pvesm status
```

Each node subsequently reported `t-20-backup` as active and returned an empty
backup list, which was expected after recreating the directory.

---

# 6. Rebuilding and Verifying Backups

The cluster-wide job was started manually to create a new recovery set. Nine
guests completed successfully. VM 100 reached the end of its stream but failed
while finalizing the compressed archive:

```text
zstd: /*stdout*: Resource temporarily unavailable
ERROR: Backup of VM 100 failed
```

The kernel log showed destination-side CIFS write errors during the period when
four nodes were writing concurrently:

```text
CIFS: VFS: \\192.168.1.208 Error -14 sending data on socket to server
```

Because subsequent backups succeeded and the target remained active, only VM
100 was retried:

```bash
vzdump 100 \
    --storage t-20-backup \
    --mode snapshot \
    --compress zstd \
    --notes-template '{{guestname}}'
```

The retry completed successfully. Final verification showed exactly one current
archive for every expected guest:

```text
1 100
1 101
1 105
1 200
1 201
1 210
1 300
1 320
1 400
1 500
```

---

# 7. Preventive Changes

## Finite Retention

The original storage policy retained everything indefinitely. It was replaced
with:

```text
keep-last=7
keep-weekly=4
keep-monthly=3
```

The policy preserves recent, weekly, and monthly recovery points without
allowing unbounded growth on a target whose logical capacity exceeds its
physical capacity.

## Staggered Jobs

The all-node job was replaced with per-node jobs:

| Node | Daily schedule |
|---|---:|
| `pve01` | `02:00` |
| `pve02` | `04:00` |
| `pve03` | `05:00` |
| `pve04` | `06:00` |

Staggering reduces concurrent CIFS and USB write pressure on `storage01` and the
Drobo.

## Proxmox Thin-Pool Review

The backup logs also exposed LVM-thin warnings on `pve01`. Investigation found:

```text
Initial thin-pool data use:       33.79%
Initial metadata use:             1.85%
Initial VG free space:            16.00 GiB
LXC 201 root filesystem use:      85%
```

LXC 201 was expanded from 3 GiB to 6 GiB, reducing filesystem utilization to
42%. Obsolete VM snapshots were removed through Proxmox, reducing thin-pool data
use to 22.78% and metadata use to 1.18%.

LVM monitoring remained active. A limited auto-extension guard was enabled:

```text
thin_pool_autoextend_threshold=80
thin_pool_autoextend_percent=4
```

The 4% value fits within the current 16 GiB VG reserve. It is a safety guard,
not a substitute for monitoring or future capacity expansion.

---

# 8. Operational Lessons

## Post-Power-Loss SMB Availability Incident

A later power outage produced a second storage-related failure mode.

After `storage01` rebooted:

- the Drobo returned normally as `G:`;
- `G:\ProxmoxBackups` remained intact;
- the `ProxmoxBackups` SMB share still existed;
- the Windows `LanmanServer` service was running;
- TCP port 445 was listening locally;
- Proxmox nevertheless reported `t-20-backup` as unavailable;
- Uptime Kuma independently reported TCP 445 on `storage01` as down.

The failure was therefore not caused by the Drobo, the SMB share, or the
Proxmox storage definition.

The active Windows Ethernet connection had been classified as:

```text
NetworkCategory : Public
```

Because Windows Firewall applies more restrictive policy to Public networks,
remote SMB access from the LAN was blocked.

The profile was corrected with:

```powershell
Set-NetConnectionProfile -InterfaceIndex 20 -NetworkCategory Private
```

SMB access and Proxmox backup-storage availability returned without recreating
the CIFS storage definition.

This incident reinforced the need to troubleshoot storage dependencies in
layers:

```text
attached storage
    ↓
Windows filesystem
    ↓
SMB share
    ↓
SMB server/listener
    ↓
Windows firewall/network profile
    ↓
network path
    ↓
Proxmox CIFS storage
```

A Proxmox storage alarm should therefore not be treated immediately as a
storage-device or CIFS-configuration failure. Host networking and Windows
firewall/profile state must also be validated after an unexpected reboot or
power interruption.

- A configured backup job is not proof that recoverable archives still exist.
- Task logs can establish the scope and recency of a lost backup set.
- Recreating a deleted SMB share may leave stale CIFS mounts on every client.
- Cluster-wide parallel writes can expose limits in a Windows/USB-backed target.
- Thin-provisioned logical capacity must not be confused with physical capacity.
- Retention must be finite and aligned with the destination's real capacity.
- Backup verification should account for every expected VM/LXC ID.
- Hypervisor-capacity warnings can reveal separate guest-filesystem risks.

---

# 9. Skills Demonstrated

- Windows SMB share and NTFS permission administration
- Proxmox CIFS storage troubleshooting
- Linux mount and kernel-log analysis
- Proxmox task-history and backup-scope validation
- Targeted `vzdump` recovery
- Backup retention and scheduling design
- Drobo protected-capacity interpretation
- LVM-thin capacity analysis and monitoring
- Incident documentation and corrective-action planning
