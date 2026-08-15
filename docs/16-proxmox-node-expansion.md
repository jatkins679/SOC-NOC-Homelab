# Proxmox Fourth-Node Expansion and Workload Rebalancing

## Objective

This change expanded the `homelab` Proxmox VE cluster from three nodes to four
by repurposing a Dell Precision 5550 as `pve04`. The work also standardized
Proxmox versions, validated quorum, rebalanced selected workloads, and reviewed
automatic-start behavior for essential services.

The change was performed without adding Ceph or claiming automatic high
availability. VM disks remain on local node storage, with recovery protection
provided by backups on the shared `t-20-backup` CIFS target.

## Added Host

| Item | Verified value |
|---|---|
| Hostname | `pve04` |
| Hardware | Dell Precision 5550 |
| CPU | Intel Core i7-10850H; 6 cores / 12 threads |
| Memory | 32 GiB |
| Storage | 256 GB Samsung NVMe |
| Management address | `192.168.1.13/24` |
| Default gateway | `192.168.1.254` |
| Ethernet | Realtek RTL8153B USB Gigabit adapter; Linux `r8152` driver |
| Proxmox bridge | `vmbr0` |

The Precision 5550 has no built-in RJ45 Ethernet port. Its current Proxmox
management connection therefore uses a USB Ethernet adapter. A second adapter
is reserved for a later managed-switch and segmented-network phase.

## Host Preparation

After installing Proxmox VE, the host identity, network, CPU, memory, storage,
and USB Ethernet driver were checked:

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

The host reported 12 logical CPUs, approximately 31 GiB of usable memory, and
an RTL8153 adapter using the `r8152` driver.

## Repository and Version Standardization

The subscription-only enterprise sources returned HTTP 401 responses because
this lab does not use a Proxmox subscription. They were disabled, and the
Proxmox VE no-subscription repository for Debian Trixie was enabled:

```text
http://download.proxmox.com/debian/pve trixie pve-no-subscription
```

Each cluster node was updated and rebooted individually. The verified target
state was:

```text
pve-manager/9.2.10
Linux kernel 7.0.14-12-pve
```

Rolling one node at a time kept a majority of cluster votes online during the
maintenance window.

## Name Resolution

Before `pve04` joined, the following host mappings were made consistent on all
nodes:

```text
192.168.1.10 pve01.home.arpa pve01
192.168.1.11 pve02.home.arpa pve02
192.168.1.12 pve03.home.arpa pve03
192.168.1.13 pve04.home.arpa pve04
```

This provided stable short-name and FQDN resolution for cluster administration.

## Cluster Join and Quorum Validation

After the fourth node joined, membership was checked with:

```bash
pvecm status
pvecm nodes
```

The verified result was:

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

Four voting nodes require three votes for a majority. The cluster can lose one
node and remain quorate, but losing two nodes removes quorum. This is control-
plane protection and is not the same as automatic workload failover.

## Workload Rebalancing

Two selected VMs were shut down and migrated between nodes. Existing backups
were verified before the moves.

| VM | ID | Source | Destination | Memory | Disk | Address after validation |
|---|---:|---|---|---:|---:|---:|
| `vulnscan01` | 320 | `pve03` | `pve04` | 8 GiB | 80 GiB | `192.168.1.247` |
| `wazuh01` | 500 | `pve01` | `pve03` | 8 GiB | 100 GiB | `192.168.1.206` |

Validation included VM state, QEMU guest-agent response, and guest interface
addresses:

```bash
qm status 320
qm agent 320 ping
qm guest cmd 320 network-get-interfaces

qm status 500
qm agent 500 ping
qm guest cmd 500 network-get-interfaces
```

The moves preserved each VM's virtual NIC configuration and IPv4 address.

## Current Placement

| Node | Current workloads |
|---|---|
| `pve01` | `dc01`, `win11-01`, `docker`, `sqlserver2025`, `apache-guacamole`, `pialert` |
| `pve02` | `kali01`, `target01` |
| `pve03` | `wazuh01` |
| `pve04` | `vulnscan01` |

This is placement for capacity and administrative convenience. It does not imply
live migration or HA because the virtual disks reside on local storage.

## Startup Policy

Essential services were configured to start automatically in a controlled
order. Interactive lab and scanning systems remain manual to avoid unnecessary
resource use and unintended test activity.

| Node | Workload | Startup setting |
|---|---|---|
| `pve01` | `dc01` | `onboot=1`, order 1, 60-second delay |
| `pve01` | `apache-guacamole` | `onboot=1`, order 2, 20-second delay |
| `pve01` | `pialert` | `onboot=1`, order 3, 20-second delay |
| `pve03` | `wazuh01` | `onboot=1`, order 1, 120-second delay |

## Laptop Power Behavior

To keep `pve04` available with its lid closed, logind lid actions were set to
ignore and all sleep-related systemd targets were masked:

```ini
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

```bash
systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

The host remained reachable after the lid was closed. This setting does not
replace normal thermal monitoring; vents must remain unobstructed during
continuous operation.

## Validation Commands

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

## Result

- [x] `pve04` installed and validated
- [x] USB Gigabit Ethernet recognized with the `r8152` driver
- [x] Sleep and lid-close shutdown behavior disabled
- [x] Four nodes standardized on Proxmox VE 9.2.10
- [x] Four nodes running kernel 7.0.14-12-pve
- [x] Four-node membership and three-vote quorum verified
- [x] Shared backup storage active on the added node
- [x] `vulnscan01` migrated to `pve04` and validated
- [x] `wazuh01` migrated to `pve03` and validated
- [x] Essential-service autostart reviewed and configured
- [x] Current architecture and asset inventory updated

## Skills Demonstrated

- Proxmox VE installation and cluster expansion
- Linux repository management and rolling patching
- Cluster quorum analysis
- USB Ethernet validation and Linux driver identification
- Static management addressing and hostname resolution
- Offline VM migration and recovery planning
- QEMU guest-agent validation
- Workload placement and capacity planning
- Service startup sequencing
- Laptop power-management configuration for server use
- Change documentation and evidence sanitization
