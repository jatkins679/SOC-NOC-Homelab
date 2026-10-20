# Proxmox Backup and Disaster-Recovery Preparation

## Overview

Before rebuilding my Proxmox hosts for the SOC/NOC homelab, I needed to know what was running on the existing systems, where the data was stored, and whether I could recover it if something went wrong.

The equipment was already hosting services I used around the house, including Windows VMs, Pi-hole, Kali Linux, Channels DVR, WatchYourLAN, and FreePBX.

I didn't want to start reinstalling Proxmox or moving workloads until I had a reasonably complete inventory and usable backups.

This document covers the preparation and initial recovery work. A separate document covers a later, more structured restore test of my Docker VM.

[Proxmox Backup and Restore Validation](21-proxmox-backup-restore-validation.md)

---

## 1. Inventory the Existing Hosts

I started by checking the host's network interfaces, addressing, hardware, and persistent configuration.

### Network Configuration

```bash
ip -br link
ip -br addr
cat /etc/network/interfaces
```

The first two commands provided a quick look at active interfaces and assigned IP addresses. The third showed the persistent Debian/Proxmox network configuration, including bridges, physical interface assignments, and gateway settings.

I wanted to preserve that information because the rebuild would involve changing physical connections, adding cluster nodes, and eventually introducing VLANs.

To identify physical Ethernet hardware:

```bash
lspci | grep -i ethernet
```

This was useful because the Proxmox hosts have multiple virtual and physical network interfaces.

I also inspected an existing tunnel interface:

```bash
ip addr show tun0
ip route | grep tun0
```

These commands helped identify the interface's current address and any routes depending on it.

### Virtual Machines and Containers

```bash
qm list
pct list
```

These gave me separate inventories of the QEMU/KVM virtual machines and LXC containers.

I needed to know which guests existed, their VMIDs, and whether they were running before deciding what to back up or relocate.

### Host Resources

```bash
free -h
```

I checked available memory and swap as part of understanding the existing host's resource usage.

---

## 2. Inspect Storage

Before starting backups, I checked which storage targets Proxmox recognized and whether they were available.

```bash
pvesm status
ls -lah /mnt/pve/
```

The first command showed storage configuration, status, capacity, and utilization.

The second let me inspect the mounted storage paths directly.

I also checked the locally stored installation media:

```bash
ls -lh /var/lib/vz/template/iso/
```

Some of those ISO files could be useful when rebuilding or recovering the virtual machines.

### Remote Backup Storage

The backup target was the Windows-based storage system associated with my Dell PowerEdge T20.

At the time, the target was named:

```text
t20-backup
```

I checked whether Proxmox could enumerate its contents:

```bash
pvesm list t20-backup
```

This was an important distinction. Seeing a mounted directory isn't quite the same as confirming that Proxmox recognizes the storage target and can read its registered backup content.

The storage configuration has evolved since this initial work. The later shared backup target is named `t-20-backup`.

---

## 3. Back Up the Guests

The documented VM and container IDs identified for backup included:

```text
100 101 102 103 104 105 109
```

Proxmox uses `vzdump` for backing up virtual machines and LXC containers.

The following is a reconstruction based on my notes, not a preserved copy of the original shell command:

```bash
vzdump 100 101 102 103 104 105 109 \
  --storage t20-backup \
  --compress zstd \
  --mode snapshot
```

The options specify the backup destination, Zstandard compression, and snapshot backup mode.

Snapshot mode can reduce interruption to running guests, although its behavior and consistency depend on the guest and storage configuration.

The key objective was to get backups onto storage separate from the physical host I intended to rebuild.

---

## 4. Back Up the Host Configuration

VM and container backups don't preserve everything needed to rebuild a Proxmox host.

I therefore created a separate archive of the host configuration, including information useful for reconstructing network, storage, and Proxmox settings.

I verified that the archive existed using:

```bash
ls -lh /root/*host-config-2026-08-11.tar.gz
```

This confirmed that a matching archive file was present. It did not, by itself, prove the archive was complete or usable.

The exact command originally used to produce the archive wasn't preserved in my notes, so I haven't reconstructed it as though it were copied from the terminal.

---

## 5. Inspect the Backups

After producing the backups, I wanted to confirm that Proxmox could do more than simply list the resulting files.

I used `pvesm extractconfig` to inspect guest configuration information contained in backup archives.

The command takes a backup archive reference, for example:

```bash
pvesm extractconfig <backup-archive>
```

This allowed me to check whether the guest configuration metadata could be read.

I also experimented with shell pipelines, `awk`, and loops to process multiple backup entries.

The full original inspection script wasn't retained, so I'm not including an invented version of it here.

Being able to extract the configuration is useful evidence, but it isn't a substitute for restoring the actual workload.

---

## 6. Restoration Testing

The next step was recovery testing.

For LXC containers, the commands involved included:

```bash
pct restore
pct set
pct start
pct exec
```

These tools allow a container to be restored, its configuration adjusted, the container started, and commands run inside it.

For QEMU/KVM guests with the QEMU Guest Agent available, I also used:

```bash
qm guest exec
```

The work helped establish a basic recovery process rather than relying entirely on whether a backup job reported success.

The original notes do not preserve a complete end-to-end transcript for every guest restored during this phase.

A later test, performed on August 26, 2026, provides more complete evidence: I restored Docker VM 210 to temporary VMID 910, isolated it from the network, and verified its services before removing the temporary guest.

[Read the restore test](21-proxmox-backup-restore-validation.md)

---

## 7. Backup Validation

The process established several checks I continue to use when evaluating backups.

1. Confirm the backup job completed successfully.
2. Verify the resulting archive exists on the intended storage target.
3. Confirm the backup isn't stored exclusively on the system being rebuilt.
4. Check whether Proxmox can recognize and read the backup.
5. Inspect the guest configuration when applicable.
6. Restore a guest into a suitable test environment.
7. Verify that the restored operating system and required services function.

The last two steps are particularly important. A backup archive is useful only if I can recover the workload it contains.

---

## 8. Commands Reference

| Command | Purpose |
|---|---|
| `ip -br link` | List network interfaces |
| `ip -br addr` | Inspect IP addresses |
| `cat /etc/network/interfaces` | Review persistent network configuration |
| `lspci \| grep -i ethernet` | Identify Ethernet controllers |
| `ip addr show tun0` | Inspect a tunnel interface |
| `ip route \| grep tun0` | Inspect tunnel-related routes |
| `qm list` | Inventory virtual machines |
| `pct list` | Inventory LXC containers |
| `free -h` | Check memory usage |
| `pvesm status` | Inspect Proxmox storage |
| `ls -lah /mnt/pve/` | Inspect mounted storage |
| `ls -lh /var/lib/vz/template/iso/` | Inventory installation media |
| `pvesm list t20-backup` | List registered backup content |
| `vzdump` | Back up guests |
| `pvesm extractconfig` | Extract guest configuration from backup |
| `pct restore` | Restore a container |
| `pct set` | Change container configuration |
| `pct start` | Start a container |
| `pct exec` | Execute a command in a container |
| `qm guest exec` | Run a command through the QEMU Guest Agent |

---

## 9. Outcome

This initial work established an inventory of the existing Proxmox environment and a basic backup and recovery procedure before I began repurposing the hardware.

I documented the guest IDs, storage targets, host configuration, and relevant commands. I also performed recovery work and started verifying that backups could be inspected and restored.

There are still a few gaps in the original notes, particularly the exact historical commands used for some operations.

I'm keeping those gaps visible rather than replacing them with commands I can't verify.

The later isolated Docker VM restore test gave me a better, repeatable method for validating backups.

---

## Next Phase

With the initial backup preparation completed, the next project was to move Pi-hole DNS off Proxmox and onto a dedicated Raspberry Pi.

[Pi-hole Migration](02-pihole-migration.md)
