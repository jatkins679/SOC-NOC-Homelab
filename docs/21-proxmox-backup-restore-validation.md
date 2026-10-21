# Proxmox Backup and Restore Validation

## Overview

A successful backup job doesn't necessarily mean the backup can be restored.

I wanted to test that distinction in my Proxmox environment, particularly because the backups are stored on a Windows server using SMB/CIFS.

On August 26, 2026, I restored a backup of my Debian Docker VM into a separate temporary virtual machine.

I kept the restored copy disconnected from the production network, verified that the operating system booted, checked Docker, and confirmed that the restored Uptime Kuma application responded correctly.

After the test, I removed the temporary VM and verified that the original production machine had not been affected.

**The restore test passed.**

---

## 1. Tested Environment

The exercise used these systems and resources:

| Item | Value |
|---|---|
| Test date | August 26, 2026 |
| Proxmox node | `pve01` |
| Production VM | VMID 210 — `docker` |
| Temporary restored VM | VMID 910 |
| Backup storage | `t-20-backup` |
| Restore destination | `local-lvm` |
| Guest OS | Debian 13 |
| Container runtime | Docker |
| Application | Uptime Kuma |
| Production address | `192.168.1.174` |
| Result | PASS |

The exact backup archive was:

```text
t-20-backup:backup/vzdump-qemu-210-2026_08_26-15_55_47.vma.zst
```

The production VM used `vmbr0` and DHCP.

This page records the commands and observations from that specific test. Any future restore should use the correct current VM configuration and backup archive rather than assuming those historical values still apply.

---

## 2. Why I Isolated the Restored VM

Restoring a copy of a working server creates the possibility of duplicate network identities.

The backup can contain the original hostname, MAC address, IP configuration, and application data.

Starting that copy on the same network as the production VM could cause conflicts or unintended behavior.

For this test, I took three precautions:

1. Generated a new virtual MAC address during restore.
2. Disabled automatic startup.
3. Placed the restored virtual NIC into a link-down state before booting.

I could then check the guest using the Proxmox console and QEMU Guest Agent rather than connecting it to the production LAN.

---

## 3. Checking the Backup

On `pve01`, I confirmed that Proxmox could list the backup:

```bash
pvesm list t-20-backup --vmid 210
```

I also inspected the production VM configuration:

```bash
qm config 210
```

The recorded configuration included:

```text
name: docker
bios: ovmf
memory: 4096
cores: 2
ipconfig0: ip=dhcp
net0: virtio=02:1F:08:D6:E8:C4,bridge=vmbr0
onboot: 1
scsi0: local-lvm:vm-210-disk-1,discard=on,size=10G,ssd=1
```

I wanted to know the original networking and disk configuration before restoring it.

I also checked the available Proxmox storage:

```bash
pvesm status
```

The temporary VMID was 910.

Before using it, I checked that no existing VM had that ID:

```bash
qm status 910
```

The expected result was that the VM configuration did not exist.

---

## 4. Restoring the Backup

Before restoring, I checked whether the installed `qmrestore` supported generating a unique network identity.

```bash
qmrestore help | grep -A2 -B2 unique
```

The relevant option was:

```text
--unique <boolean>
    Assign a unique random ethernet address.
```

I then restored the archive:

```bash
qmrestore \
  t-20-backup:backup/vzdump-qemu-210-2026_08_26-15_55_47.vma.zst \
  910 \
  --storage local-lvm \
  --unique 1
```

I did not use `--start`.

The restored VM remained stopped after the restore operation completed.

The restore recreated the VM configuration, EFI disk, cloud-init disk, and system disk.

---

## 5. Verifying Network Isolation

Before booting the restored machine, I disabled automatic startup:

```bash
qm set 910 --onboot 0
```

I inspected the restored configuration:

```bash
qm config 910 | egrep '^(name|net0|onboot|ipconfig0|scsi0|efidisk0):'
```

The restored VM had a new MAC address.

| VM | MAC address recorded during test |
|---|---|
| Production VM 210 | `02:1F:08:D6:E8:C4` |
| Restored VM 910 | `BC:24:11:18:BF:60` |

I then placed the restored network interface in a disconnected state:

```bash
qm set 910 --net0 "virtio=BC:24:11:18:BF:60,bridge=vmbr0,link_down=1"
```

The configuration check showed:

```text
net0: virtio=BC:24:11:18:BF:60,bridge=vmbr0,link_down=1
```

This allowed the VM to boot without communicating over the production network.

---

## 6. Booting the Restored Guest

I started the temporary VM:

```bash
qm start 910
```

Then checked its status:

```bash
qm status 910
```

The VM reached the running state and Debian booted successfully.

Because the virtual network link was intentionally disabled, cloud-init couldn't reach DNS or package repositories during startup.

Those network-related messages were expected under the test conditions.

They weren't evidence of a failed restore.

---

## 7. Checking the Operating System

The guest already had the QEMU Guest Agent installed.

That allowed me to run commands inside the restored VM from the Proxmox host, without needing SSH connectivity.

I checked its hostname:

```bash
qm guest exec 910 -- hostname
```

The result was:

```text
docker
```

That confirmed that the guest operating system was running and that communication through the QEMU Guest Agent worked.

It also confirmed the restored VM retained the original hostname, which reinforced why I had isolated it from the production network.

---

## 8. Checking Docker and Uptime Kuma

The next step was to determine whether the application recovered—not merely whether Debian had booted.

From the Proxmox host:

```bash
qm guest exec 910 -- docker ps
```

Docker reported that the restored Uptime Kuma container was running and healthy.

The container used the image:

```text
louislam/uptime-kuma:2
```

and exposed port 3001.

I then tested the application locally inside the restored VM:

```bash
qm guest exec 910 -- curl -I http://localhost:3001
```

The recorded response was:

```text
HTTP/1.1 302 Found
Location: /dashboard
```

The redirect demonstrated that the restored Uptime Kuma web service was responding.

Because the VM was isolated, this test did not establish external network accessibility or notification delivery. Those weren't part of this exercise.

It did establish that the restored operating system, Docker runtime, and application could start and respond.

---

## 9. Test Result

**Restore validation: PASS**

August 26, 2026 — VMID 210 restored as temporary VMID 910.

Verified: VM configuration, restored disks, Debian startup, QEMU Guest Agent, Docker container health, and Uptime Kuma HTTP response.

This gave me evidence that the archive was usable and that the application could recover from it.

The test was more meaningful than confirming that the Proxmox backup task had completed successfully.

---

## 10. Cleaning Up

Once I'd finished validating the restored guest, I shut it down:

```bash
qm shutdown 910
```

I checked that it had stopped:

```bash
qm status 910
```

Then removed the temporary VM and its associated storage:

```bash
qm destroy 910 --purge 1
```

The recorded cleanup removed the restored EFI, cloud-init, and system disks.

I checked again that VMID 910 no longer existed.

Production VM 210 remained untouched throughout the exercise.

---

## 11. Reusing the Procedure

For another restore test, the same general procedure applies:

1. Select a recent backup.
2. Verify that the archive is accessible.
3. Inspect the original VM configuration.
4. Select an unused temporary VMID.
5. Confirm that restore storage has enough capacity.
6. Restore without automatically starting the VM.
7. Ensure the restored network identity cannot conflict with production.
8. Boot the temporary VM.
9. Verify the guest operating system.
10. Verify the application or service.
11. Record the result.
12. Remove the temporary environment when finished.

The exact isolation method may vary by guest and application.

Some services require testing with an isolated network rather than a disconnected NIC. In those cases, I would use a dedicated test VLAN or isolated virtual network and confirm that it could not reach the production environment.

---

## Result

The August 26 restore test demonstrated that a backup of the Docker VM could be restored into a separate machine and used to run its original operating system and application.

I verified the complete path from the Proxmox archive through guest startup, Docker, and an actual Uptime Kuma HTTP response.

The test also gave me a repeatable procedure for future recovery exercises.

I want backup monitoring and occasional restore testing to remain part of the ongoing operation of the lab.

## Related Documentation

- [Proxmox Backup and Disaster Recovery](01-proxmox-backup.md)
- [Storage and Backup Resilience](17-storage-backup-resilience.md)
- [Homelab Operations Checklist](15-homelab-operations-checklist.md)
- [Proxmox Cluster](03-proxmox-cluster-build.md)
