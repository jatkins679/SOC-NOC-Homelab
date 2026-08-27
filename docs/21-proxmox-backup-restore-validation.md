# Proxmox Backup and Restore Validation

## Purpose

This runbook documents how to validate a Proxmox VE VM backup by restoring it as a temporary, isolated VM and verifying that the guest operating system and application services are recoverable.

A successful backup job alone does not prove recoverability. This procedure tests the complete recovery path without disrupting the production VM.

---

## Tested Environment

Validation performed **August 26, 2026**.

| Item                   | Value            |
| ---------------------- | ---------------- |
| Proxmox node           | `pve01`          |
| Production VM          | `210` (`docker`) |
| Temporary restore VMID | `910`            |
| Backup storage         | `t-20-backup`    |
| Restore storage        | `local-lvm`      |
| Guest OS               | Debian 13        |
| Application            | Uptime Kuma      |
| Container runtime      | Docker           |
| Production networking  | `vmbr0`, DHCP    |
| Production VM IP       | `192.168.1.174`  |
| Validation result      | **PASS**         |

The backup tested was:

```text
t-20-backup:backup/vzdump-qemu-210-2026_08_26-15_55_47.vma.zst
```

---

## Safety Considerations

Do not boot a restored copy of a production VM directly onto the production LAN without first reviewing its network configuration.

A restored VM can contain the original:

* MAC address
* hostname
* DHCP configuration
* static IP configuration
* application identity
* service configuration

This can create duplicate-IP, duplicate-hostname, or application conflicts.

For this test:

1. The backup was restored using `--unique 1` to generate a new MAC address.
2. Automatic startup was disabled.
3. The restored NIC was placed in a link-down state before the first boot.
4. Application testing was performed locally inside the restored VM.

---

## 1. Locate the Backup

Run on the Proxmox host:

```bash
pvesm list t-20-backup --vmid 210
```

Confirm that the desired backup appears and that Proxmox can read the backup storage.

---

## 2. Inspect the Production VM

```bash
qm config 210
```

Important settings observed during this validation included:

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

The DHCP configuration and existing MAC address made network isolation particularly important.

---

## 3. Select an Unused Temporary VMID

VMID `910` was selected for the test.

Verify that it is unused:

```bash
qm status 910
```

Expected result:

```text
Configuration file 'nodes/pve01/qemu-server/910.conf' does not exist
```

Check storage capacity:

```bash
pvesm status
```

Confirm that the destination storage has sufficient free space.

---

## 4. Confirm Restore Options

Verify that the installed `qmrestore` supports unique network identity generation:

```bash
qmrestore help | grep -A2 -B2 unique
```

The relevant option is:

```text
--unique <boolean>
    Assign a unique random ethernet address.
```

---

## 5. Restore the Backup

Restore the production VM backup to temporary VMID `910`:

```bash
qmrestore \
  t-20-backup:backup/vzdump-qemu-210-2026_08_26-15_55_47.vma.zst \
  910 \
  --storage local-lvm \
  --unique 1
```

Do not use `--start`.

By default, `qmrestore` leaves the restored VM stopped.

The restore successfully recreated:

* EFI disk
* cloud-init disk
* 10 GB system disk
* VM configuration

---

## 6. Disable Automatic Startup

Before booting:

```bash
qm set 910 --onboot 0
```

Verify:

```bash
qm config 910 | egrep '^(name|net0|onboot|ipconfig0|scsi0|efidisk0):'
```

During the August 26 test, the restored VM received a new MAC address:

```text
BC:24:11:18:BF:60
```

This differed from production VM 210:

```text
02:1F:08:D6:E8:C4
```

---

## 7. Isolate the Restored NIC

Keep the virtual NIC installed but place its link in the down state:

```bash
qm set 910 --net0 "virtio=BC:24:11:18:BF:60,bridge=vmbr0,link_down=1"
```

Verify:

```bash
qm config 910 | grep '^net0'
```

Expected:

```text
net0: virtio=BC:24:11:18:BF:60,bridge=vmbr0,link_down=1
```

This permits normal guest booting while preventing communication with the production LAN.

---

## 8. Boot the Restored VM

```bash
qm start 910
```

Verify:

```bash
qm status 910
```

Expected:

```text
status: running
```

The Debian guest successfully reached multi-user mode.

Because the NIC was intentionally disconnected, cloud-init reported network/DNS failures while attempting package updates. These were expected and did not indicate a failed restore.

---

## 9. Verify the Guest Through QEMU Guest Agent

The restored VM had the QEMU Guest Agent installed and running.

This allowed validation without SSH or the guest root password.

From the Proxmox host:

```bash
qm guest exec 910 -- hostname
```

Successful result:

```text
docker
```

This confirmed that the restored operating system was operational and that Proxmox could communicate with the guest agent.

---

## 10. Verify Docker and Application Recovery

Check running containers:

```bash
qm guest exec 910 -- docker ps
```

The restored VM reported the Uptime Kuma container as:

```text
Up ... (healthy)
```

The restored container used:

```text
louislam/uptime-kuma:2
```

and exposed TCP port `3001`.

Next, test the application locally inside the restored VM:

```bash
qm guest exec 910 -- curl -I http://localhost:3001
```

The restored Uptime Kuma instance returned:

```text
HTTP/1.1 302 Found
Location: /dashboard
```

This confirmed that the application was responding successfully.

---

## 11. Validation Result

**PASS**

The test demonstrated that the backup could successfully recover:

* Proxmox VM configuration
* EFI configuration
* virtual disks
* Debian operating system
* QEMU Guest Agent
* Docker runtime
* Uptime Kuma container
* persistent application state
* Uptime Kuma web service

This provides stronger evidence of recoverability than a successful backup-job status alone.

---

## 12. Remove the Temporary Restore

Shut down the validation VM:

```bash
qm shutdown 910
```

Verify:

```bash
qm status 910
```

Expected:

```text
status: stopped
```

Delete the temporary VM and its associated storage:

```bash
qm destroy 910 --purge 1
```

Verify removal:

```bash
qm status 910
```

Expected:

```text
Configuration file 'nodes/pve01/qemu-server/910.conf' does not exist
```

During the August 26 validation, Proxmox successfully removed:

```text
vm-910-cloudinit
vm-910-disk-1
vm-910-disk-0
```

Production VM `210` remained untouched throughout the exercise.

---

## Operational Lesson

A backup should not be considered fully validated merely because the backup job completed successfully.

There are two separate questions:

1. **Did the backup complete?**
2. **Can the system actually be recovered from that backup?**

Periodic isolated restore testing provides evidence for the second question.

For important Proxmox workloads, restore validation should be incorporated into routine backup and disaster-recovery procedures.
