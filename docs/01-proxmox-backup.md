# Proxmox Backup and Disaster-Recovery Preparation

## Overview

Before rebuilding the existing Proxmox hosts for the new SOC/NOC homelab, I wanted to make sure that the current virtual machines, LXC containers, host configuration, and locally stored installation media were identified and protected.

The objective:

> **Do not make destructive changes to an existing virtualization host until I know what is running on it, where its data is stored, and whether I can recover it.**

This phase involved:

- Identifying the Proxmox host’s network configuration
- Inventorying virtual machines and LXC containers
- Identifying configured Proxmox storage
- Inspecting locally stored ISO images
- Verifying the remote backup destination
- Backing up Proxmox guests
- Backing up the Proxmox host configuration
- Inspecting backup contents
- Performing recovery/restoration testing
- Verifying backups before proceeding with the homelab rebuild

## Why Back Up the Existing Environment?

The existing Proxmox server was already hosting several services used on my home network:

- Windows VMs
- Kali Linux VM
- Pi-hole LXC
- WatchYourLAN LXC
- Channels DVR LXC
- FreePBX VM

Before repurposing the physical hardware, I needed to make sure that I could recover those services if something went wrong.

This was particularly important because the planned redesign involves:

- Reinstalling Proxmox hosts
- Moving services between physical systems
- Creating a multi-node Proxmox cluster
- Changing network addressing
- Introducing VLANs
- Moving some services from virtual machines or containers to physical hardware

I wanted to confirm that the backups were not only created successfully, but could also be inspected and restored if needed. The process therefore included backup verification and recovery testing before any destructive changes were made.

# 1. Inventory the Existing Proxmox Host

Before changing anything, I collected information about the host.

## Inspect Network Interfaces

    ip -br link

### Purpose

Display a list of the host’s network interfaces and their current state.

This was useful for identifying:

- Physical Ethernet interfaces
- Linux bridges
- Tunnel interfaces
- Virtual interfaces associated with VMs and containers
- Interfaces that were UP or DOWN

### Skill Demonstrated

- Linux networking
- Interface identification
- Infrastructure discovery

## Record IP Addressing

    ip -br addr

### Purpose

Display the IP addresses assigned to each interface.

I wanted a record of the current addressing before making changes to the Proxmox networking configuration.

### Skill Demonstrated

- TCP/IP
- Linux network administration
- Configuration documentation

## Inspect the Proxmox Network Configuration

    cat /etc/network/interfaces

### Purpose

Review the persistent Debian/Proxmox network configuration.

This allowed me to identify:

- Physical interfaces
- Linux bridges
- Static addressing
- Gateway configuration
- Existing Proxmox networking relationships

### Why This Matters

The output of `ip addr` shows the current state of the system, while `/etc/network/interfaces` shows how the host is configured to establish networking after boot.

Both are useful when documenting or rebuilding a Proxmox host.

### Skill Demonstrated

- Debian networking
- Proxmox networking
- Linux bridge configuration
- Infrastructure documentation

## Identify Physical Ethernet Hardware

    lspci | grep -i ethernet

### Purpose

Identify the Ethernet controllers installed in the host.

This helped distinguish physical network hardware from the many virtual interfaces visible on a Proxmox system.

### Skill Demonstrated

- Linux hardware discovery
- NIC identification
- Troubleshooting

## Inspect an Existing Tunnel Interface

    ip addr show tun0

and:

    ip route | grep tun0

### Purpose

Inspect the configuration and routing associated with the existing `tun0` interface before modifying the host.

The first command examined the interface itself, while the second looked for routes associated with it.

### Skill Demonstrated

- Linux routing
- Tunnel interfaces
- Network troubleshooting

# 2. Inventory Proxmox Guests

The next step was to determine exactly what Proxmox was hosting.

## List QEMU/KVM Virtual Machines

    qm list

### Purpose

List all QEMU/KVM virtual machines registered on the Proxmox host.

The command provides information such as:

- VM ID
- VM name
- Status
- Allocated memory
- Disk usage
- Process ID when running

### Why I Used It

I wanted a complete VM inventory before beginning the backup and migration process.

### Skill Demonstrated

- Proxmox VE administration
- QEMU/KVM virtual machine management
- Infrastructure inventory

## List LXC Containers

    pct list

### Purpose

List all LXC containers registered on the Proxmox host.

This was particularly important because several home-network services were running as Proxmox containers.

### Skill Demonstrated

- Proxmox VE administration
- LXC container administration
- Service inventory

# 3. Inspect Host Resources

## Check Memory

    free -h

### Purpose

Display current RAM and swap utilization in human-readable format.

This gave me a quick view of the resources available on the physical Proxmox host before migration and restoration work.

### Skill Demonstrated

- Linux resource monitoring
- Capacity assessment
- Systems administration

# 4. Inventory Proxmox Storage

## Check Configured Storage

    pvesm status

### Purpose

Display the storage resources configured in Proxmox and their current status.

This helped identify:

- Local storage
- Backup storage
- Storage availability
- Capacity
- Utilization

### Why This Matters

Before creating backups, I needed to make sure the intended destination was actually available to Proxmox.

### Skill Demonstrated

- Proxmox storage administration
- Capacity management
- Backup preparation

## Inspect Mounted Proxmox Storage

    ls -lah /mnt/pve/

### Purpose

Inspect storage mounted beneath Proxmox’s `/mnt/pve` directory.

This provided another way to verify that the expected storage was mounted and accessible from the host operating system.

### Skill Demonstrated

- Linux filesystem administration
- Storage troubleshooting
- Proxmox storage verification

# 5. Inventory Installation Media

## List Local ISO Images

    ls -lh /var/lib/vz/template/iso/

### Purpose

Identify ISO images stored locally on the Proxmox host.

These files may be needed again when rebuilding virtual machines after the Proxmox host is reconfigured.

### Skill Demonstrated

- Linux filesystem navigation
- Proxmox storage layout
- Migration preparation

# 6. Verify the Remote Backup Destination

A remote backup target named:

    t20-backup

was being used for Proxmox backups.

## List Backups on the Storage Target

    pvesm list t20-backup

### Purpose

Ask Proxmox to enumerate the content stored on the `t20-backup` storage target.

This was more useful than simply proving that a directory existed because it verified that Proxmox itself recognized and could access the backup content.

### Skill Demonstrated

- Proxmox storage administration
- Remote backup verification
- Disaster-recovery preparation

# 7. Back Up the Proxmox Guests

Proxmox provides the `vzdump` utility for creating backups of virtual machines and LXC containers.

The guest IDs identified for backup during this phase included:

    100
    101
    102
    103
    104
    105
    109

## Reconstructed Backup Command

Based on the documented guest IDs, backup destination, compression method, and snapshot mode, the command used for this backup was reconstructed as follows:

    vzdump 100 101 102 103 104 105 109 \
      --storage t20-backup \
      --compress zstd \
      --mode snapshot

## Options Used

### `--storage t20-backup`

Store the backups on the configured `t20-backup` target rather than on the local Proxmox disk.

### `--compress zstd`

Use Zstandard compression for the backup archives.

### `--mode snapshot`

Create backups using Proxmox snapshot mode where supported.

This allows a running guest to be backed up with less interruption than a stop-mode backup.

# 8. Back Up the Proxmox Host Configuration

In addition to the VMs and containers, I created a separate archive of the Proxmox host configuration.

This is important because guest backups do not by themselves preserve every useful piece of information about the physical Proxmox host.

Configuration worth preserving includes items such as:

- Network configuration
- Storage configuration
- Proxmox configuration files
- Host-specific settings
- Cluster-related configuration where applicable

## Verify the Host Configuration Archive

The resulting configuration archive was verified with:

    ls -lh /root/*host-config-2026-08-11.tar.gz

### Purpose

Confirm that the host-configuration archive existed and had been written to disk.

### Skill Demonstrated

- Linux backup administration
- File verification
- Disaster-recovery planning

# 9. Inspect Backup Configuration

Creating a backup file is only one part of a backup process.

I also used Proxmox tools to inspect configuration information contained in the backups.

One of the commands used during this work was:

    pvesm extractconfig ...

### Purpose

Extract guest configuration information from an existing Proxmox backup.

This provides an additional verification step by showing that Proxmox can read metadata from the backup rather than treating it merely as an opaque file.

### Skill Demonstrated

- Proxmox backup inspection
- Backup validation
- Disaster-recovery troubleshooting

## Automating Backup Inspection

During the verification process, I also used shell tools including:

    pvesm list
    awk

and a shell loop of the form:

    while read ...

to process multiple backup entries and inspect their configurations.

### Skill Demonstrated

- Bash shell scripting
- Pipelines
- Text processing
- `awk`
- Automation
- Proxmox CLI administration

# 10. Restoration Testing

Backups are much more useful if I know I can restore from them.

During this phase I performed recovery work using Proxmox LXC commands including:

    pct restore ...

### Purpose

Restore an LXC container from a Proxmox backup.

Additional container-management commands used during the recovery/testing process included:

    pct set ...
    pct start ...
    pct exec ...

These commands were used to modify restored container settings, start containers, and execute commands inside them as part of validating the restored environment.

For virtual machines, guest interaction also included:

    qm guest exec ...

### Skill Demonstrated

- LXC disaster recovery
- Proxmox guest administration
- Post-restore configuration
- Service validation
- Troubleshooting

# 11. Backup Verification Strategy

The backup process followed this general workflow:

    Inventory
       ↓
    Identify guests and storage
       ↓
    Verify backup destination
       ↓
    Back up guests
       ↓
    Back up host configuration
       ↓
    List resulting backups
       ↓
    Inspect backup configuration
       ↓
    Restore/test selected workloads
       ↓
    Verify operation
       ↓
    Proceed with rebuild

# 12. Backup Validation Criteria

I used the following questions as validation criteria before proceeding with the rebuild:

1.  Did the backup command complete?
2.  Does the backup file actually exist?
3.  Is the backup stored somewhere other than the system being rebuilt?
4.  Can Proxmox read the backup?
5.  Can configuration information be extracted from it?
6.  Can the workload be restored?
7.  Does the restored workload actually start and function?

# 13. Commands Reference

The principal commands used during this phase are summarized below.

| **Command**                                  | **Purpose**                                                                                                                            |
|----------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| `ip -br link` | List network interfaces and their current state.                                                                                       |
| `ip -br addr` | Display IP addressing assigned to each interface.                                                                                      |
| `cat /etc/network/interfaces` | Inspect the persistent Debian/Proxmox network configuration.                                                                           |
| `lspci \` | grep -i ethernet                    | Identify physical Ethernet controllers.                                                                                                |
| `ip addr show tun0` | Inspect the existing tun0 interface.                                                                                                   |
| `ip route \` | grep tun0                        | Inspect routes associated with tun0.                                                                                                   |
| `qm list` | Inventory QEMU/KVM virtual machines.                                                                                                   |
| `pct list` | Inventory LXC containers.                                                                                                              |
| `pvesm status` | Review configured Proxmox storage, availability, capacity, and utilization.                                                            |
| `free -h` | Review RAM and swap utilization.                                                                                                       |
| `ls -lh /var/lib/vz/template/iso/` | Inventory locally stored ISO images.                                                                                                   |
| `ls -lah /mnt/pve/` | Inspect storage mounted beneath /mnt/pve.                                                                                              |
| `pvesm list t20-backup` | List content available on the remote Proxmox backup target.                                                                            |
| `vzdump 100 101 102 103 104 105 109 ...` | Create guest backups using the documented remote storage, compression, and snapshot options; command reconstructed from project notes. |
| `ls -lh /root/\*host-config-2026-08-11.tar.gz` | Verify that the host-configuration archive exists.                                                                                     |
| `pvesm extractconfig ...` | Inspect guest configuration information contained in a backup.                                                                         |
| `pct restore ...` | Restore an LXC container from backup.                                                                                                  |
| `pct set ...` | Modify configuration for a restored LXC container.                                                                                     |
| `pct start ...` | Start an LXC container.                                                                                                                |
| `pct exec ...` | Execute a command inside an LXC container.                                                                                             |
| `qm guest exec ...` | Execute a command inside a VM through the QEMU Guest Agent.                                                                            |

# 14. Skills Demonstrated

This phase of the project provided hands-on experience with:

## Proxmox VE

- VM inventory
- LXC inventory
- Storage administration
- Guest backup
- Backup inspection
- LXC restoration
- VM/LXC command-line management

## Linux Administration

- Network interface inspection
- Routing inspection
- PCI hardware identification
- Filesystem navigation
- File verification
- Memory/resource inspection
- Shell pipelines
- Text processing
- Bash loops

## Networking

- Interface discovery
- IP addressing
- Linux bridges
- Routing
- Tunnel-interface inspection

## Backup and Disaster Recovery

- Pre-change inventory
- Remote backup storage
- Host configuration backup
- Guest backup
- Backup verification
- Restoration testing
- Post-restore validation

## Operational Practice

- Change preparation
- Configuration documentation
- Troubleshooting
- Validation before destructive work
- Recovery planning

# 15. Status

- [x] Inventory existing Proxmox networking
- [x] Inventory VMs
- [x] Inventory LXC containers
- [x] Inspect Proxmox storage
- [x] Verify remote backup storage
- [x] Inventory local ISO media
- [x] Create Proxmox guest backups
- [x] Create host configuration backup
- [x] Verify host configuration archive
- [x] Inspect backup information
- [x] Perform restoration/recovery work
- [ ] Recover and document the exact original `vzdump` invocation
- [ ] Recover the exact host-configuration archive command
- [ ] Recover the exact backup-inspection pipeline
- [ ] Add final sanitized command output/screenshots

## Next Phase

The next phase moves a critical network service — **Pi-hole DNS** — away from the Proxmox environment and onto a dedicated Raspberry Pi.

See:

[`02-pihole-migration.md`](02-pihole-migration.md)
