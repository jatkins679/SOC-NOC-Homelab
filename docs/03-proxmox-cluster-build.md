# Proxmox Cluster Build, Expansion, and Validation

## Overview

One of my main goals for the homelab rebuild was to bring several independent Proxmox hosts together into a cluster.

I originally had three small-form-factor systems running Proxmox. Rather than continue managing them separately, I wanted a single management interface, consistent configuration, and a place to deploy additional networking, monitoring, and security tools.

I started with three nodes and later added a fourth, using a repurposed Dell Precision laptop.

The resulting cluster is named `homelab`.

This document covers the initial build, validation, and subsequent expansion. Some commands and results are historical; the asset inventory contains the latest documented addressing and system information.

---

## 1. Cluster Architecture

The cluster consists of four Proxmox VE nodes.

| Host | Management IP | Hardware |
|---|---|---|
| `pve01` | `192.168.1.10` | Mini PC |
| `pve02` | `192.168.1.11` | Mini PC |
| `pve03` | `192.168.1.12` | Mini PC |
| `pve04` | `192.168.1.13` | Dell Precision 5550 |

The first three nodes are relatively modest systems, with approximately 16 GB of memory each.

The Dell Precision has 32 GB of RAM and a six-core, twelve-thread processor, giving me additional capacity for more demanding virtual machines.

Several supporting systems operate outside the cluster:

- `dns01` (`192.168.1.20`) — Pi-hole DNS on a Raspberry Pi
- `mgmt01` (`192.168.1.5`) — Independent Linux management workstation
- `storage01` (`192.168.1.208`) — Windows-based storage and backup server

Keeping DNS and management access separate from Proxmox reduces the number of services affected when I'm working on the cluster.

---

## 2. Building the Initial Cluster

The initial cluster used `pve01`, `pve02`, and `pve03`.

Once the nodes were joined, I checked the cluster configuration from `pve01`.

```bash
pvecm status
```

The recorded output included:

```text
Name:             homelab
Config Version:   3
Transport:        knet
Secure auth:      on

Nodes:            3
Expected votes:   3
Total votes:      3
Quorum:           2
Quorate:          Yes
```

I also checked the membership:

```bash
pvecm nodes
```

The output listed all three nodes:

```text
Node 1   pve01
Node 2   pve02
Node 3   pve03
```

This confirmed that the nodes recognized one another and that the cluster had quorum.

### Why quorum matters

Proxmox uses quorum to determine whether enough cluster members are communicating to permit normal clustered management operations.

With three voting nodes, two votes constitute a majority.

That means one node can become unavailable while the remaining two retain quorum.

Quorum is not the same as high availability. My cluster does not automatically restart every failed VM on another node. Most VM disks are stored locally, and I haven't configured Proxmox HA for automatic workload failover.

The cluster gives me centralized management, but recovery still depends on the storage configuration and backup procedures.

---

## 3. Validating the Proxmox Hosts

After building the cluster, I checked the identity, network configuration, storage, and available resources on the nodes.

### Proxmox Version

```bash
pveversion
```

During the initial three-node validation, `pve01` reported:

```text
pve-manager/9.2.2
Kernel: 7.0.2-6-pve
```

These were the versions at that stage of the project, before the later cluster-wide updates.

### Host Identity

```bash
hostname
hostname -f
```

On `pve01`, the recorded values were:

```text
pve01
pve01.home.arpa
```

Consistent hostnames and working name resolution are important because cluster communication and administration depend on stable node identities.

### Network Configuration

```bash
ip -br addr
ip route
```

At the time, `pve01` used:

```text
Interface:       vmbr0
Management IP:   192.168.1.10/24
Default gateway: 192.168.1.254
```

The management network was `192.168.1.0/24`, with the AT&T gateway at `192.168.1.254`.

The network has since evolved to include managed switching, additional Proxmox bridges, VLAN-tagged guests, and OPNsense routing.

Those changes are covered in the [VLAN and managed switching documentation](04-network-vlans.md).

---

## 4. Storage Configuration

I checked the storage recognized by Proxmox:

```bash
pvesm status
```

The configured storage included:

| Storage | Purpose |
|---|---|
| `local` | ISOs, templates, and other local files |
| `local-lvm` | Local VM and container disks |
| `t-20-backup` | Shared backup storage |

The shared backup target is hosted by `storage01`, a Dell PowerEdge T20 running Windows Server.

It is presented to Proxmox using SMB/CIFS and backed by attached storage.

This gives the nodes access to a common backup destination without requiring the VM disks themselves to be shared across the cluster.

I deliberately kept the initial storage design relatively simple.

---

## 5. Checking Workloads and Resources

I used the Proxmox command-line tools to inspect the existing virtual machines and containers.

```bash
qm list
pct list
```

To view all cluster nodes through the Proxmox API:

```bash
pvesh get /nodes
```

The initial results showed all three nodes online.

I also checked the VMs and containers across the cluster:

```bash
pvesh get /cluster/resources --type vm
```

This provided a consolidated view rather than requiring me to inspect each Proxmox host individually.

For local memory information:

```bash
free -h
```

At the time of the initial validation, `pve01` had approximately 15 GiB of RAM, with around 11 GiB available and 8 GiB of configured swap.

The resource checks helped me decide where additional services could run without concentrating everything on one node.

---

## 6. Why I Didn't Deploy Ceph

I considered using Ceph but decided against it.

Ceph is distributed storage software that can store redundant copies or encoded portions of data across multiple systems. In a Proxmox environment, it can provide shared storage for VMs and support more flexible workload placement.

It also requires memory, storage capacity, and network bandwidth.

My original Proxmox nodes were small, low-power mini PCs with limited local disks and network connections.

For this lab, the additional complexity and resource consumption weren't justified.

Instead, I use:

- Local storage for running VM and container disks
- Shared storage for backups
- Staggered backup schedules
- Restore testing to verify recovery procedures

This doesn't provide the same availability as a properly designed Ceph environment, but it's a more reasonable fit for the hardware and my current objectives.

---

## 7. Expanding to Four Nodes

In August 2026, I added a Dell Precision 5550 laptop as `pve04`.

The laptop was available for repurposing and had substantially more compute capacity than the existing mini PCs.

| Component | Configuration |
|---|---|
| Processor | Intel Core i7-10850H |
| CPU | 6 cores / 12 threads |
| Memory | 32 GiB |
| Storage | 256 GB NVMe SSD |
| Network | USB Gigabit Ethernet, Realtek RTL8153B |
| Proxmox IP | `192.168.1.13` |

### Preparing the New Host

Before adding `pve04`, I checked hostname resolution across the existing nodes and standardized the repository configuration.

The cluster uses the Proxmox no-subscription repository.

I then updated the hosts and rebooted them individually rather than taking the entire cluster offline.

The recorded common software state following these updates was:

```text
pve-manager/9.2.10
Kernel: 7.0.14-12-pve
```

### Joining and Validating `pve04`

After joining the fourth node, I checked cluster status again.

The recorded result included:

```text
Name:             homelab
Config Version:   4
Transport:        knet
Secure auth:      on

Nodes:            4
Expected votes:   4
Total votes:      4
Quorum:           3
Quorate:          Yes
```

With four voting nodes, quorum requires three votes.

The cluster can still tolerate one unavailable node, but losing two nodes would leave it without quorum.

Adding the fourth node increased available compute capacity, not the number of node failures the cluster could tolerate while maintaining a majority.

---

## 8. Rebalancing Workloads

After `pve04` joined, I moved selected workloads to make better use of the available hardware.

Two of the documented migrations were:

| VM | VMID | From | To |
|---|---:|---|---|
| `vulnscan01` | 320 | `pve03` | `pve04` |
| `wazuh01` | 500 | `pve01` | `pve03` |

The migrations used offline moves with backups available as recovery protection.

Afterward, I checked guest-agent connectivity and confirmed that the guests retained their expected addresses.

These placements reflect the August expansion work, not necessarily the latest VM placement or operational state.

The detailed migration record is available in:

[Proxmox Node Expansion and Workload Rebalancing](16-proxmox-node-expansion.md)

---

## 9. Running a Laptop as a Proxmox Node

Using a laptop as a virtualization host required a few additional changes.

The Precision 5550 doesn't have a built-in RJ45 Ethernet port, so I use a USB Gigabit Ethernet adapter.

I also needed to prevent the machine from suspending when its lid was closed.

I configured systemd to ignore lid-close actions and masked the sleep, suspend, hibernate, and hybrid-sleep targets.

After making those changes, I tested the system with the lid closed.

It remained accessible and retained cluster membership.

The laptop isn't intended to be a highly available enterprise server, but it provides useful compute capacity for a homelab.

---

## 10. Startup and Recovery

I reviewed which guests should start automatically when their Proxmox host boots.

Services that support the rest of the environment, such as the domain controller and security monitoring, are more useful when they start automatically.

Testing systems, attack targets, and other resource-intensive workloads can remain stopped until needed.

This helps keep resource usage under control and makes recovery after a power interruption more predictable.

Because the lab relies heavily on local VM storage, the backup and recovery process remains important even with multiple cluster nodes.

Related documentation:

- [Proxmox Backup and Disaster-Recovery Preparation](01-proxmox-backup.md)
- [Proxmox Backup and Restore Validation](21-proxmox-backup-restore-validation.md)
- [Homelab Shutdown and Startup Runbook](19-shutdown-startup-runbook.md)

---

## 11. Commands Reference

The principal commands used to inspect and validate the cluster were:

| Command | Purpose |
|---|---|
| `pvecm status` | Check cluster quorum and membership |
| `pvecm nodes` | List cluster members |
| `pveversion` | Check the installed Proxmox version |
| `hostname` | Show the short hostname |
| `hostname -f` | Show the fully qualified hostname |
| `ip -br addr` | Inspect addresses and interfaces |
| `ip route` | Inspect the routing table |
| `pvesm status` | Check configured Proxmox storage |
| `qm list` | List virtual machines |
| `pct list` | List LXC containers |
| `pvesh get /nodes` | Inspect cluster node resources |
| `pvesh get /cluster/resources --type vm` | Inspect cluster-wide VMs and containers |
| `free -h` | Check local memory usage |

---

## 12. Result

The homelab now has a four-node Proxmox cluster providing a common management platform for my virtual machines and containers.

The cluster has been validated with four voting nodes and a three-vote quorum requirement.

DNS, management, and shared backup storage are provided by systems outside the cluster.

The design does not include Ceph or automatic VM high availability, but it provides enough flexibility for the networking, security monitoring, and systems-administration work I'm doing.

The cluster has become the foundation for the rest of the lab, including Active Directory, Wazuh, Docker, LibreNMS, OPNsense, and the security-testing environment.

The next phase of the project involves expanding the network configuration, managed switching, VLAN segmentation, and additional monitoring services.

## Related Documentation

- [Asset Inventory](02-asset-inventory.md)
- [Managed Switching and VLAN Design](04-network-vlans.md)
- [Proxmox Node Expansion](16-proxmox-node-expansion.md)
- [Storage and Backup Resilience](17-storage-backup-resilience.md)
