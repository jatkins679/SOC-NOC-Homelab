# Physical Infrastructure and Lab Layout

## Overview

The homelab has grown from a collection of spare computers and networking equipment into a fairly complicated installation.

It includes four Proxmox nodes, managed switches, Raspberry Pis, a Windows storage server, external disk enclosures, UPS units, and various small supporting devices.

Organizing the equipment physically has become as important as documenting the network configuration.

I want to be able to identify a device, trace its cables, disconnect it, and put it back without having to disturb several other systems.

This document describes the physical layout and the standards I'm using for equipment placement, cabling, power, and maintenance.

Some of the arrangements began as a rebuild plan. Hardware placement and port assignments should be checked against the actual installation when changes are made.

---

## 1. Physical Space

The main equipment area uses a shelving unit with approximately these dimensions:

| Measurement | Size |
|---|---|
| Width | 48 inches |
| Depth | 18 inches |
| Clearance between shelves | About 17 inches |

The AT&T fiber and power connection enters near the wall to the left of the window.

That location influences where the gateway can be positioned and how the upstream networking and power cables are routed.

The goal is to keep the equipment accessible rather than pack as many devices as possible into the available space.

---

## 2. Equipment Layout

The three-shelf arrangement groups equipment by function.

### Top Shelf — Network and Compute

The network and compute area accommodates:

- `pve01`, `pve02`, and `pve03` mini PCs
- `pve04`, the Dell Precision 5550 laptop
- `sw01`, Cisco SG350-10
- `sw02`, UniFi US-8
- Raspberry Pi systems, including `dns01` and `tailscale01`
- Small network adapters and supporting equipment

I group the Proxmox nodes together and label them individually.

The switches need enough clearance that I can see port assignments and move a patch cable without disturbing adjacent equipment.

### Middle Shelf — Storage and WAN

The storage area includes:

- AT&T BGW320 gateway
- `storage01`, Dell PowerEdge T20
- Drobo storage array
- Other external disks connected to the Windows server

Keeping the Windows server and its attached disks together makes it easier to identify USB and power connections.

It also makes storage troubleshooting easier when I need to determine whether a failure involves the disk, server, SMB service, or network path.

### Bottom Shelf — Power

Power equipment is grouped separately so that UPS connections and power adapters can be traced.

I want the power distribution to be as understandable as the Ethernet cabling.

---

## 3. UPS Identification

I standardized the UPS naming to avoid confusion between the two larger units and the smaller backup-power device.

| Name | Identification |
|---|---|
| `UPS01` | Older of the two large UPS units |
| `UPS02` | APC SUA1000 |
| `UPS03` | APC BE850G2 |

Earlier notes sometimes used different UPS numbers or identified the wrong model for UPS02.

The table above reflects the corrected naming.

Device-to-UPS assignments should be based on measured load, runtime, and service dependencies rather than assumed from the UPS model alone.

A primary consideration is keeping essential networking and management services available long enough for an orderly shutdown.

Detailed shutdown procedures are covered in the [Shutdown and Startup Runbook](19-shutdown-startup-runbook.md).

---

## 4. Ventilation and Physical Mounting

The small systems make it tempting to stack everything tightly or attach devices wherever space is available.

I'm using a few practical rules to avoid making the installation difficult to maintain.

- Keep intake and exhaust vents clear.
- Avoid placing heavier equipment on adhesive mounts.
- Use reusable hook-and-loop straps for lightweight devices.
- Keep power bricks identifiable.
- Don't suspend devices by their power or data cables.
- Leave enough cable slack to remove one device without disconnecting others.

Lightweight Raspberry Pi cases and small adapters may be suitable for removable mounting.

The storage server and heavier drives should rest on stable surfaces.

For `pve04`, the Dell laptop, I also need to account for cooling while operating with the lid closed.

---

## 5. Ethernet Cable Organization

I've adopted a color convention for different cable functions.

| Color | Purpose |
|---|---|
| Blue | LAN and compute |
| Green | Storage |
| Yellow | Network management |
| Red | WAN |

The color helps identify a cable's intended function, but it doesn't replace a label identifying the actual endpoints.

Both ends of a cable should be labeled.

For example, a cable connecting `pve02` to `sw01` should be identifiable at the node and the switch.

The detailed cable-labeling and routing conventions are documented in [Cabling Standard](20-cabling-standard.md).

### Physical Routing

I aim to:

- Use appropriately sized patch cables between nearby devices.
- Avoid unnecessary cable loops.
- Keep network cables separated from large AC power bundles where practical.
- Avoid sharp bends and strain at connectors.
- Use reusable cable ties.
- Keep switch-port labels visible.
- Leave adequate slack for servicing equipment.

This is primarily about making troubleshooting and equipment replacement easier.

---

## 6. Managed Switch Connections

The Cisco SG350-10, `sw01` (`192.168.1.21`), is the main managed switch for the lab.

The documented switch connections include:

| Port | Connection |
|---|---|
| Gi1 | `pve01` |
| Gi2 | `pve02` |
| Gi3 | `pve03` |
| Gi4 | `pve04` |
| Gi5 | `sw02` UniFi US-8 |
| Gi8 | Upstream flat-network connection |

The Proxmox links and the link to `sw02` are used for the managed VLAN infrastructure.

These assignments are a documentation reference; physical ports should be checked before disconnecting or reconnecting anything.

The switch configuration and VLAN details are maintained separately in [Managed Switching and VLANs](04-network-vlans.md).

---

## 7. Before Moving Equipment

Before a significant physical change, I record the current installation.

That includes photographs, cable labels, switch-port connections, and power assignments.

I also verify the state of services that might be affected.

For example, before relocating a Proxmox node or storage device, I want to know which guests it hosts and whether its backup and recovery options are available.

A useful sequence is:

1. Record the current physical layout.
2. Identify the systems affected.
3. Verify the necessary backups.
4. Label both ends of the cables.
5. Record the current port connections.
6. Establish a rollback path.
7. Make the physical change.
8. Restore power and network connections.
9. Verify the systems and services.
10. Update the documentation.

For larger changes, I use the shutdown and startup procedures rather than disconnecting systems arbitrarily.

---

## 8. Serviceability Checks

A successful layout should make routine maintenance straightforward.

I want to be able to remove one Proxmox node, replace a network cable, reach a switch management connection, or service an external disk without dismantling the entire installation.

I also want to identify which UPS powers a device and where its Ethernet connection terminates.

If accessing one system requires moving or disconnecting several unrelated systems, the arrangement needs improvement.

That standard applies to future hardware additions as well.

---

## 9. Physical Documentation

Photographs are particularly useful for changes involving hardware, cabling, and power.

Before-and-after photos can show what changed and help reconstruct a configuration if a connection was made incorrectly.

I want the photos and cabling records to complement the logical network diagrams and asset inventory.

Useful records include:

- Equipment placement
- Labels and hostnames
- Switch ports
- Power connections
- Cable routes
- UPS assignments
- Changes made during a rebuild

This page documents the physical standards and known arrangements. It doesn't claim that every proposed placement has been independently photographed and verified.

---

## Result

The physical organization of the homelab is intended to make the equipment easier to operate and troubleshoot.

The goal isn't to build a miniature data center or create an elaborate rack installation.

It's to take a growing collection of repurposed equipment and organize it so that network, storage, power, and service dependencies are easier to understand.

As the lab changes, the physical documentation needs to change with it.

## Related Documentation

- [Asset Inventory](02-asset-inventory.md)
- [Managed Switching and VLANs](04-network-vlans.md)
- [Storage and Backup Resilience](17-storage-backup-resilience.md)
- [Shutdown and Startup Runbook](19-shutdown-startup-runbook.md)
- [Cabling Standard](20-cabling-standard.md)
