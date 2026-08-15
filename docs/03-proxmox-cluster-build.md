# Proxmox Cluster Build, Expansion, and Validation

## Objective

The initial stage of the homelab rebuild consolidated three Proxmox VE hosts into a single manageable cluster. The cluster was later expanded to four nodes with the addition of `pve04`.

The goals were to:

- Manage all three Proxmox hosts from one interface.
- Provide quorum for normal cluster operation.
- Make VM and container placement easier to manage.
- Establish a platform for SOC/NOC services such as Wazuh, network monitoring, vulnerability scanning, and isolated security-testing systems.
- Validate the cluster before making additional infrastructure changes.

This environment is a small home lab built from inexpensive mini PCs and repurposed hardware, so the design emphasizes practical administration and learning rather than enterprise-scale high availability.

## Cluster Architecture

The resulting Proxmox cluster is named:

```text
homelab
```

It was initially built with three nodes and now contains four:

| Node | Management IP |
|---|---|
| pve01 | 192.168.1.10 |
| pve02 | 192.168.1.11 |
| pve03 | 192.168.1.12 |
| pve04 | 192.168.1.13 |

An independent Linux management system, `mgmt01`, is kept outside the cluster for administrative access.

DNS is provided by the physical Raspberry Pi system `dns01` at `192.168.1.20`.

Shared backup storage is provided through the Dell PowerEdge T20 storage system.

## Cluster Validation

After the cluster was assembled, I verified membership and quorum from `pve01`.

```bash
pvecm status
```

The cluster reported:

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

This confirmed that all three nodes were participating and that the cluster had quorum.

I then checked individual cluster membership:

```bash
pvecm nodes
```

The three nodes were present:

```text
Node 1   pve01
Node 2   pve02
Node 3   pve03
```

## Why Quorum Matters

With three voting nodes and a quorum requirement of two votes, the cluster can lose one node and still retain a majority. This is one reason a three-node design is more useful for cluster administration practice than a two-node design.

Quorum protects the cluster control plane from conflicting decisions when nodes cannot communicate reliably. In this lab, quorum validation demonstrates that `pve01`, `pve02`, and `pve03` agree on cluster membership and that normal clustered management operations can continue while a majority of votes remains available.

Quorum should not be confused with workload high availability. This lab does not currently use Ceph or Proxmox HA for automatic workload failover; VM and container recovery is handled through local storage, shared backups, and documented restore procedures.

## Proxmox Version

The Proxmox version was checked with:

```bash
pveversion
```

At validation time, `pve01` was running:

```text
pve-manager/9.2.2
```

with kernel `7.0.2-6-pve`.

## Host Identity

The node hostname and fully qualified hostname were verified:

```bash
hostname
hostname -f
```

For `pve01`:

```text
pve01
pve01.home.arpa
```

Using consistent node names is important because Proxmox cluster membership, certificates, storage configuration, and management all depend on stable host identity.

## Network Validation

The active interfaces were checked with:

```bash
ip -br addr
```

On `pve01`, the Proxmox bridge `vmbr0` carried the management address:

```text
192.168.1.10/24
```

The routing table was then verified:

```bash
ip route
```

The default route was:

```text
default via 192.168.1.254 dev vmbr0
```

and the local network was:

```text
192.168.1.0/24
```

This confirmed that the management bridge and default gateway were operating normally.

## Storage Validation

Cluster storage was checked using:

```bash
pvesm status
```

The active storage on `pve01` included:

```text
local
local-lvm
t-20-backup
```

`local` is directory-based storage used for files such as ISO images and templates. `local-lvm` is LVM-thin storage used primarily for VM and container virtual disks.

`t-20-backup` is CIFS-backed shared storage hosted by the Dell PowerEdge T20 and is used for Proxmox backups.

At the time of validation, all three storage entries reported an active state.

## Existing Workloads

Local virtual machines were inspected with:

```bash
qm list
```

Containers were inspected with:

```bash
pct list
```

Existing workloads on the lab included Windows/Linux virtual machines and containers supporting services such as:

- Docker
- Apache Guacamole
- SQL Server
- network/security testing
- vulnerability scanning

The objective was not to eliminate existing useful workloads during the cluster rebuild, but to establish a cleaner management platform around them.

## Cluster Resource Check

I used the Proxmox API command-line interface to confirm the status and resource utilization of all three nodes:

```bash
pvesh get /nodes
```

The results showed:

```text
pve01   online
pve02   online
pve03   online
```

All three nodes had approximately 15.4 GiB of RAM and four logical CPUs available to Proxmox.

Cluster-wide VM and container resource utilization was also checked:

```bash
pvesh get /cluster/resources --type vm
```

This provided a single view of running QEMU virtual machines and LXC containers across the cluster.

## Memory Validation

Memory usage on the host was checked with:

```bash
free -h
```

At the time of validation, `pve01` had approximately:

```text
15 GiB total RAM
11 GiB available
8 GiB swap
```

This provided sufficient headroom for additional lightweight SOC/NOC services while still leaving capacity for the existing workloads.

## Design Decision: No Ceph

I chose not to deploy Ceph on this cluster.

Although Proxmox supports Ceph for distributed storage and high availability, these nodes are small N95/N97-class systems with limited CPU, memory, network interfaces, and local storage.

Adding Ceph would consume resources without providing enough benefit for this lab.

Instead:

- VM disks remain on local Proxmox storage.
- Backups are stored on the shared T20 storage server.
- Recovery is handled through tested Proxmox backup and restore procedures.

This is more appropriate for the scale and purpose of the environment.


## Four-Node Expansion - August 2026

The original three-node validation above is retained as build history. The
current cluster was expanded by installing Proxmox VE on a repurposed Dell
Precision 5550 and joining it as `pve04`.

`pve04` provides substantially more compute capacity than the three small-form-
factor nodes:

| Item | Verified value |
|---|---|
| Model | Dell Precision 5550 |
| Processor | Intel Core i7-10850H; 6 cores / 12 threads |
| Memory | 32 GiB |
| Local storage | 256 GB Samsung NVMe |
| Management address | `192.168.1.13/24` |
| Network adapter | Realtek RTL8153B USB Gigabit Ethernet using the Linux `r8152` driver |

The mobile-workstation chassis does not provide a built-in RJ45 port, so the
current management bridge uses a USB Gigabit Ethernet adapter. A second adapter
may later separate management and lab traffic after the managed Cisco switch is
installed. That future design is not represented as operational yet.

Before the node joined the cluster, hostnames were made resolvable on every node
through consistent `/etc/hosts` entries. The Proxmox repositories were also
changed from the subscription-only enterprise repositories to the supported
no-subscription repository used by this lab.

All four nodes were then updated and rebooted one at a time. The verified common
software state was:

```text
pve-manager/9.2.10
kernel 7.0.14-12-pve
```

Rolling maintenance preserved cluster quorum while avoiding a simultaneous
outage of all hosts.

After `pve04` joined, cluster validation reported:

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

With four voting nodes, the cluster requires three votes. It can therefore
tolerate one unavailable node while retaining quorum. Quorum still does not
provide automatic VM failover because the lab uses local VM disks and does not
run Proxmox HA with shared or distributed production storage.

Two workloads were moved offline using verified backups as recovery protection:

| Workload | VM ID | Previous node | Current node | Result |
|---|---:|---|---|---|
| `vulnscan01` | 320 | `pve03` | `pve04` | Guest agent responded; address remained `192.168.1.247` |
| `wazuh01` | 500 | `pve01` | `pve03` | Guest agent responded; address remained `192.168.1.206` |

This placement moves vulnerability-scanning demand to the higher-capacity
Precision host and reduces workload concentration on `pve01`.

Startup policy was reviewed after migration. Core services start automatically,
while interactive testing and resource-intensive lab workloads remain manual:

| Node | Workload | Startup policy |
|---|---|---|
| `pve01` | `dc01` | Automatic; order 1 |
| `pve01` | `apache-guacamole` | Automatic; order 2 |
| `pve01` | `pialert` | Automatic; order 3 |
| `pve03` | `wazuh01` | Automatic; order 1 |
| Various | Kali, target, vulnerability scanner, Windows endpoint, Docker, SQL Server | Manual unless needed |

Because `pve04` is a laptop-form-factor host, systemd lid actions were set to
`ignore` and sleep, suspend, hibernate, and hybrid-sleep targets were masked.
A closed-lid validation confirmed that the host remained reachable and retained
cluster membership.

Detailed commands, validation output, and the final placement record are in
[`16-proxmox-node-expansion.md`](16-proxmox-node-expansion.md).

## Current Result

The lab now has a functioning four-node Proxmox cluster with:

- Four online nodes.
- Three-vote quorum verified.
- Centralized Proxmox management.
- Shared backup storage.
- Existing VMs and containers retained.
- Independent DNS and management systems.
- Capacity for additional SOC/NOC workloads.

The cluster now provides the virtualization foundation for the active Wazuh SIEM environment and for remaining work such as network monitoring, network segmentation, and additional controlled security-testing systems.

## Commands Used for Validation

```bash
pvecm status
pvecm nodes

pveversion

hostname
hostname -f

ip -br addr
ip route

pvesm status

qm list
pct list

free -h

pvesh get /nodes
pvesh get /cluster/resources --type vm
```

## Skills Demonstrated

- Proxmox VE administration
- Multi-node cluster management
- Cluster quorum validation
- Linux networking
- Linux command-line administration
- Virtual machine and LXC management
- Shared storage administration
- Capacity planning
- Backup-oriented infrastructure design
- Infrastructure documentation

## Status

- [x] Initial three Proxmox nodes installed and online
- [x] Cluster `homelab` created
- [x] `pve01`, `pve02`, and `pve03` joined to the initial cluster
- [x] `pve04` installed and joined as the fourth node
- [x] Four-node quorum verified
- [x] All nodes standardized on Proxmox VE 9.2.10 and kernel 7.0.14-12-pve
- [x] `vulnscan01` and `wazuh01` migrated and validated
- [x] Essential-service startup policy reviewed
- [x] Cluster management verified from the Proxmox interface
- [x] Management addressing verified
- [x] Shared `t-20-backup` storage verified
- [x] Existing VMs and containers visible across the cluster
- [x] Cluster resource utilization reviewed
- [x] Backup strategy retained
- [x] Ceph intentionally excluded from the design

## Evidence / Follow-Up

The cluster is operational and validated.

Additional portfolio evidence can be added later if useful:

- Screenshot of the four-node Proxmox cluster
- Screenshot of cluster storage status
- Screenshot of cluster resource view
- Network diagram showing `pve01`, `pve02`, `pve03`, `mgmt01`, `dns01`, and `storage01`

These items are supporting evidence rather than blockers for completion.

## Next Phase

The Proxmox cluster now serves as the virtualization foundation for the SOC/NOC environment.

Wazuh SIEM is already deployed and operational; its implementation is documented separately in `docs/06-wazuh-siem.md`.

The next major infrastructure work includes:

- Network monitoring
- Cisco managed-switch integration
- VLAN segmentation
- OPNsense routing/firewalling
- Additional monitored endpoints and security-testing systems
