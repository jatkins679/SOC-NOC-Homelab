# Ethernet Cabling and Labeling Standard

## Overview

As the homelab has grown, I've ended up with enough Ethernet cables that identifying connections by sight or memory is no longer practical.

The cabling includes connections between Proxmox nodes, managed switches, storage systems, Raspberry Pis, the AT&T gateway, and other devices.

I want to be able to disconnect a cable, identify its purpose, and reconnect it correctly without tracing the entire run.

This document describes the labeling, color, and cable-record conventions I'm using.

The physical labels identify the cables. The documentation records where they're connected.

---

## 1. Cable Identification

I use a simple numbered identification system:

```text
C01
C02
C03
C04
...
```

Each cable gets a unique identifier.

The number identifies the physical cable, not its VLAN or switch port.

For example, if cable `C03` is moved from one switch port to another, it remains `C03`.

I update the connection record rather than assigning a new cable number.

This makes it possible to track physical changes without constantly relabeling equipment.

---

## 2. Labeling Both Ends

Each Ethernet cable should have a label at both ends.

A label includes its cable ID and, where useful, the connected device.

For example:

```text
C02 — PVE01
```

At the switch end, the port information can also be recorded:

```text
C02 — PVE01
SW01 Gi1
```

I prefer labels that are readable without removing the cable from the switch.

A label should also be placed far enough from the RJ45 connector that it doesn't interfere with inserting or removing it.

The two ends should be identifiable independently.

---

## 3. Cable Color Convention

I'm using different cable colors to help identify their intended functions.

| Cable color | Purpose |
|---|---|
| Blue | LAN and compute |
| Green | Storage |
| Yellow | Network management |
| Red | WAN |

This is a visual convention, not a technical requirement.

A blue cable might carry several tagged VLANs over a trunk connection, for example.

The actual network configuration is determined by the switch, interface, and VLAN settings—not the cable color.

Cable labels and port records remain the authoritative way to identify a connection.

---

## 4. Prepared Cable Labels

The initial physical-rebuild plan established these cable identifiers.

| Cable | Endpoint | Original planned connection |
|---|---|---|
| `C01` | AT&T BGW320 | `sw01` Gi8 |
| `C02` | `pve01` | `sw01` Gi1 |
| `C03` | `pve02` | `sw01` Gi2 |
| `C04` | `pve03` | `sw01` Gi3 |
| `C05` | `pve04` | `sw01` Gi4 |
| `C06` | `storage01` | To be recorded |
| `C07` | `mgmt01` | To be recorded |
| `C08` | `dns01` | To be recorded |

These are the labels prepared for the original physical rebuild.

They should not automatically be treated as a verified inventory of every cable currently installed.

In particular, additional network equipment and connections have been added since the initial plan.

I want to confirm each physical cable assignment before treating the record as current.

---

## 5. Current Managed-Switch Connections

The documented logical connections for `sw01`, the Cisco SG350-10 at `192.168.1.21`, include:

| Switch port | Connected system | Network role |
|---|---|---|
| Gi1 | `pve01` | Proxmox VLAN trunk |
| Gi2 | `pve02` | Proxmox VLAN trunk |
| Gi3 | `pve03` | Proxmox VLAN trunk |
| Gi4 | `pve04` | Proxmox VLAN trunk |
| Gi5 | `sw02` | UniFi switch uplink/trunk |
| Gi8 | Upstream LAN | Flat-network uplink |

`sw02` is the UniFi US-8 at `192.168.1.22`.

This table describes the documented connections and roles. The actual physical cable IDs should be confirmed and entered when inspecting the installed wiring.

That is particularly important because the Proxmox systems may have more than one Ethernet connection.

I don't want a management connection confused with a VLAN trunk simply because both cables lead to the same physical computer.

---

## 6. Labeling Network Interfaces

Where a device has multiple Ethernet adapters, I also want to identify the interface itself.

Useful interface-role labels include:

```text
MGMT
LAB
TRUNK
```

For example, a Proxmox node may have one connection for the original management LAN and another for the managed VLAN network.

The device label should make that distinction clear.

When documenting the interface, I also record its Linux device name or Proxmox bridge assignment where relevant.

Those software identifiers can change after hardware or adapter changes, so I verify them rather than relying exclusively on an old diagram.

---

## 7. Cable Length and Routing

I use cables long enough to reach comfortably without putting tension on the connectors.

I try to avoid unnecessary coils of cable, particularly around switches and small systems.

The main routing guidelines are:

- Use short patch cables for nearby equipment.
- Leave enough slack to remove or service a device.
- Avoid tight bends and tension at connectors.
- Keep large power bundles separate from data cabling where practical.
- Use reusable hook-and-loop ties rather than tightly cinched plastic ties.
- Keep switch ports and cable labels visible.

I also avoid using the cables themselves to support equipment or adapters.

The objective is to make future maintenance easier, not simply to make the installation look tidy.

---

## 8. Maintaining the Port Record

A cable record needs to describe both the physical connection and the intended network role.

I use the following format:

| Cable ID | Endpoint A | Endpoint B | Network role | Verified |
|---|---|---|---|---|
| `C02` | `pve01` | `sw01` Gi1 | VLAN trunk | Pending physical confirmation |
| `C03` | `pve02` | `sw01` Gi2 | VLAN trunk | Pending physical confirmation |
| `C04` | `pve03` | `sw01` Gi3 | VLAN trunk | Pending physical confirmation |
| `C05` | `pve04` | `sw01` Gi4 | VLAN trunk | Pending physical confirmation |

The switch endpoints and roles are documented, but the association with these original cable IDs still needs physical confirmation.

Additional cables should be entered as their installed endpoints are verified.

For the full network configuration, I use [Managed Switching and VLANs](04-network-vlans.md).

---

## 9. Moving or Replacing a Cable

When changing a physical connection, I use a short procedure:

1. Identify the cable and both endpoints.
2. Record its current switch port.
3. Check whether the port is an access port or trunk.
4. Disconnect only the intended cable.
5. Connect it to the new port.
6. Verify link state.
7. Test the expected network connectivity.
8. Update the cable and port records.

The logical port configuration matters as much as the physical connection.

Moving a cable from a trunk port to an access port, for example, can leave a Proxmox host reachable through one interface while disconnecting its VLAN guests.

I therefore avoid assuming that two physically similar switch ports have equivalent configurations.

---

## 10. Troubleshooting Physical Connections

When a system unexpectedly loses network connectivity, I start by verifying the relevant cable and port.

I check whether the intended cable is connected, whether the switch shows a link, and whether the physical port matches the documented network role.

For multiple simultaneous failures, I look for a shared uplink or switch dependency before replacing individual cables.

On `sw01`, useful commands include:

```text
show vlan
show version
```

I also inspect interface status and error counters as appropriate.

A lit link indicator confirms physical connectivity at the Ethernet layer, but it doesn't establish that VLAN tagging, routing, or applications are functioning correctly.

---

## 11. Documentation and Photographs

Photographs can help confirm the physical installation and make future changes easier.

Useful records include:

- A readable example of the cable-label format
- Switch ports with connected cable IDs
- Device interface labels
- Overall cable routing
- Before-and-after photographs when equipment is moved

I want to keep the connection records current as the network changes.

An accurate cable record is more useful than a polished diagram that no longer matches the hardware.

---

## Result

The cabling standard provides a consistent way to identify the physical connections in the homelab.

Cable numbers identify individual cables, colors indicate broad functions, and the port record describes how the devices are actually connected.

The remaining work is to verify and document all physical cable assignments, particularly where the managed-switch infrastructure has changed since the original rebuild plan.

## Related Documentation

- [Physical Infrastructure](18-physical-infrastructure.md)
- [Managed Switching and VLANs](04-network-vlans.md)
- [Shutdown and Startup Runbook](19-shutdown-startup-runbook.md)
- [Asset Inventory](02-asset-inventory.md)
- [NOC Troubleshooting Runbook](13-noc-troubleshooting-runbook.md)
