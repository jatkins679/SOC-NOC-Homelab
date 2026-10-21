# Proxmox Fourth-Node Expansion and Workload Rebalancing

## Overview

In August 2026, I expanded the Proxmox cluster from three nodes to four by adding a repurposed Dell Precision 5550 laptop.

The original three nodes were small-form-factor systems with relatively limited memory and storage. The Precision gave me another 32 GB of RAM and a more capable processor, which was useful as I added additional security and infrastructure services.

The expansion involved more than installing Proxmox.

I also standardized the software versions across the cluster, checked hostname resolution, validated quorum, migrated selected virtual machines, and reviewed which services should start automatically.

This is a historical record of the expansion work. Some VM placements and network addresses have changed since then.

---

## 1. Added Hardware

The fourth cluster node is `pve04`.

| Component | Configuration |
|---|---|
| Hardware | Dell Precision 5550 |
| Processor | Intel Core i7-10850H |
| CPU | 6 cores / 12 threads |
| Memory | 32 GiB |
| Storage | 256 GB Samsung NVMe SSD |
| Management IP | `192.168.1.13/24` |
| Default gateway | `192.168.1.254` |
| Network adapter | Realtek RTL8153B USB Gigabit Ethernet |
| Linux driver | `r8152` |
| Management bridge | `vmbr0` |

The laptop doesn't have a built-in RJ45 Ethernet port.

I therefore used a USB Gigabit Ethernet adapter for its original management connection.

A second network connection was subsequently added for the managed-switch and VLAN infrastructure.

---

## 2. Preparing the Host

After installing Proxmox VE, I checked the hardware and operating-system configuration.

The original validation commands included:

```bash
hostnamectl
pveversion
ip -br link
ip -br addr
ip route
lsusb
lscpu | grep -E 'Model name|CPU\(s\)|Core|Thread'
free -h
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL
ethtool -i nic0
pvesm status
```

The hardware checks confirmed the expected CPU, memory, NVMe storage, and USB network adapter.

The Ethernet adapter was recognized by Linux using the `r8152` driver.

Before proceeding, I also checked that the host could reach the management network and shared Proxmox backup storage.

---

## 3. Repository Configuration and Updates

During preparation, the Proxmox enterprise package repositories returned HTTP 401 errors because the lab doesn't have a Proxmox enterprise subscription.

I disabled the subscription-only sources and enabled the no-subscription repository for Debian Trixie.

The repository entry was:

```text
http://download.proxmox.com/debian/pve trixie pve-no-subscription
```

I then updated the cluster nodes individually.

The documented common software state after the update was:

```text
pve-manager/9.2.10
Linux kernel 7.0.14-12-pve
```

I avoided rebooting all of the existing nodes at once.

Updating one node at a time helped preserve cluster quorum during maintenance.

These are the versions recorded during the August expansion, not a statement that the cluster is still running those exact versions.

---

## 4. Hostname Resolution

Before adding `pve04`, I checked hostname resolution across the cluster.

The documented mappings were:

```text
192.168.1.10 pve01.home.arpa pve01
192.168.1.11 pve02.home.arpa pve02
192.168.1.12 pve03.home.arpa pve03
192.168.1.13 pve04.home.arpa pve04
```

Consistent hostname resolution was important for administering the cluster and adding another node.

I wanted the new host to be identified correctly before joining it to the existing cluster.

---

## 5. Adding the Fourth Node

After preparing the laptop, I joined it to the `homelab` Proxmox cluster.

I checked the result with:

```bash
pvecm status
pvecm nodes
```

The recorded cluster state was:

```text
Cluster name:     homelab
Config Version:   4
Nodes:            4
Expected votes:   4
Total votes:      4
Quorum:           3
Quorate:          Yes
Transport:        knet
Secure auth:      on
```

All four nodes were recognized and the cluster had quorum.

### Quorum Implications

With four voting nodes, a majority requires three votes.

The cluster can tolerate one unavailable node and remain quorate.

If two nodes become unavailable, the remaining two don't constitute a majority.

This was an important distinction because adding a fourth node increased compute resources without increasing the number of simultaneous voting-node failures the cluster could tolerate.

It also didn't create automatic VM failover.

The virtual machines primarily use local storage, and I haven't configured Ceph or Proxmox HA for automatic workload recovery on other hosts.

---

## 6. Moving Workloads

After joining `pve04`, I moved selected virtual machines to distribute resource usage more effectively.

I verified backups before the migrations.

The documented moves were:

| VM | VMID | From | To | RAM | Disk |
|---|---:|---|---|---:|---:|
| `vulnscan01` | 320 | `pve03` | `pve04` | 8 GiB | 80 GiB |
| `wazuh01` | 500 | `pve01` | `pve03` | 8 GiB | 100 GiB |

The migrations were performed offline.

I shut down the guests, moved them, and validated them on their destination nodes.

### Guest Validation

For `vulnscan01`:

```bash
qm status 320
qm agent 320 ping
qm guest cmd 320 network-get-interfaces
```

For `wazuh01`:

```bash
qm status 500
qm agent 500 ping
qm guest cmd 500 network-get-interfaces
```

The guest-agent checks confirmed that Proxmox could communicate with the operating systems.

I also checked the virtual network interfaces and guest IP addresses.

At the time, the validated addresses were:

```text
vulnscan01  192.168.1.247
wazuh01     192.168.1.206
```

These reflect the original flat-network configuration.

`wazuh01` was subsequently assigned `10.10.40.20` on VLAN 40.

### Why Offline Migration

Because the guest disks were on local Proxmox storage, the work required moving the guest disks and configuration rather than performing a shared-storage live migration.

The offline approach was appropriate for these lab workloads.

I could schedule the downtime, confirm the backups, and verify each guest before moving on.

---

## 7. Workload Placement After Expansion

Immediately after the August rebalancing, the documented workload placement was:

| Node | Workloads |
|---|---|
| `pve01` | `dc01`, `win11-01`, `docker`, `sqlserver2025`, `apache-guacamole`, `pialert` |
| `pve02` | `kali01`, `target01` |
| `pve03` | `wazuh01` |
| `pve04` | `vulnscan01` |

This table is a historical snapshot.

The cluster has subsequently changed as services were migrated, rebuilt, or added.

For current placement, I use the Proxmox cluster inventory and updated asset documentation rather than relying on this table.

---

## 8. Startup Configuration

I also reviewed which virtual machines and containers should start automatically when their Proxmox host boots.

Some services are foundational to the lab and should come up without manual intervention.

Other workloads, such as attack targets and vulnerability scanners, don't need to run continuously.

The recorded startup configuration included:

| Node | Workload | Setting |
|---|---|---|
| `pve01` | `dc01` | `onboot=1`, order 1, 60-second delay |
| `pve01` | `apache-guacamole` | `onboot=1`, order 2, 20-second delay |
| `pve01` | `pialert` | `onboot=1`, order 3, 20-second delay |
| `pve03` | `wazuh01` | `onboot=1`, order 1, 120-second delay |

These values document the configuration at that stage of the project.

I review startup settings as services move between nodes and the power-recovery process evolves.

---

## 9. Laptop Power Management

Running a laptop as a Proxmox node required changing its normal power-management behavior.

I wanted it to remain running when the lid was closed.

In the systemd-logind configuration, I set:

```ini
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

I also masked the systemd sleep-related targets:

```bash
systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

Afterward, I closed the laptop lid and verified that the system remained reachable.

The laptop isn't designed specifically for continuous server operation, so I also need to pay attention to cooling and physical placement.

---

## 10. Post-Change Validation

After the expansion, I checked the nodes, cluster, storage, and migrated guests.

The command set included:

```bash
pveversion
uname -r
pvecm status
pvecm nodes
pvesh get /nodes --output-format text
pvesh get /cluster/resources --type vm --output-format text
pvesm status
qm status 320
qm status 500
```

The checks confirmed that the fourth node was recognized, quorum was healthy, and the migrated guests were available on their destination nodes.

The shared backup storage was also accessible from the added host.

---

## Result

The fourth-node expansion was successful.

`pve04` joined the cluster, all four voting nodes were validated, and the nodes were standardized on the recorded Proxmox version and kernel.

I moved `vulnscan01` and `wazuh01` to improve workload distribution and verified their guest state afterward.

I also adjusted laptop power-management behavior and reviewed automatic guest startup.

The expansion gave me more capacity for additional services and a chance to practice cluster membership, rolling updates, offline migration, backup-aware changes, and post-change validation.

The four-node cluster remains the foundation for the broader SOC/NOC lab.

## Related Documentation

- [Proxmox Cluster Build](03-proxmox-cluster-build.md)
- [Proxmox Backup and Disaster Recovery](01-proxmox-backup.md)
- [Storage and Backup Resilience](17-storage-backup-resilience.md)
- [Shutdown and Startup Runbook](19-shutdown-startup-runbook.md)
- [Asset Inventory](02-asset-inventory.md)
