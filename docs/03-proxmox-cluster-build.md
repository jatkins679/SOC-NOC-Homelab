# Proxmox Three-Node Cluster Build and Validation

## Objective

The next stage of the homelab rebuild was to consolidate three Proxmox VE hosts into a single manageable cluster.

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

It contains three nodes:

| Node | Management IP |
|---|---|
| pve01 | 192.168.1.10 |
| pve02 | 192.168.1.11 |
| pve03 | 192.168.1.12 |

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

## Current Result

The lab now has a functioning three-node Proxmox cluster with:

- Three online nodes.
- Working quorum.
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

- [x] Three Proxmox nodes installed and online
- [x] Cluster `homelab` created
- [x] `pve01`, `pve02`, and `pve03` joined to the cluster
- [x] Three-node quorum verified
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

- Screenshot of the three-node Proxmox cluster
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
