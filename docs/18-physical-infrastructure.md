# Physical Infrastructure and Lab Layout

## Purpose

This document records the physical design and serviceability standards used for
the SOC/NOC homelab rebuild.

Physical organization is treated as part of infrastructure operations because
poor placement, airflow, power distribution, and cabling can create outages that
look like software or network failures.

---

# 1. Physical Constraints

The primary shelving unit provides approximately:

```text
Shelf depth:              18 inches
Vertical clearance:       ~15 inches between shelves
Top-shelf clearance:      up to ~24 inches
Usable shelf width:       ~30 linear inches per level
```

The AT&T fiber/power source is on the wall to the left of the window, which
influences upstream cable and power routing.

---

# 2. Equipment Zones

The rebuild uses functional zones rather than placing devices wherever space is
available:

```text
NETWORK
COMPUTE
STORAGE
POWER / CABLE MANAGEMENT
```

The important operational principle is that related equipment can be identified,
reached, powered, disconnected, and reconnected without moving unrelated
systems.

## Network Zone

Examples:

- `sw01` Cisco SG350-10;
- `sw-home01` TRENDnet home-network switch where physically appropriate;
- `dns01`;
- other small network/support devices.

## Compute Zone

Keep the Proxmox nodes visibly grouped and individually labeled:

```text
pve01
pve02
pve03
pve04
```

## Storage Zone

Keep `storage01` and its attached storage together so power, USB, and Ethernet
relationships are visually obvious.

---

# 3. Power Separation

The power design separates lightweight network infrastructure from heavier
compute/storage loads.

The earlier rebuild plan assigns the APC 600VA unit to lightweight/network
infrastructure and the APC Smart-UPS 1000 to compute/storage, subject to actual
load and runtime testing.

Operational goals:

- keep network devices together on the appropriate UPS;
- keep compute/storage loads on the higher-capacity UPS;
- avoid hidden daisy chains and overloaded power strips;
- make device power bricks traceable;
- preserve enough physical slack to service one device without unplugging
  another.

Any final allocation should be recorded after load/runtime validation rather than
assumed from UPS nameplate capacity alone.

---

# 4. Ventilation and Mounting

Small systems may be secured with removable hook-and-loop/Velcro mounting where
it improves serviceability.

Suitable examples include:

- Raspberry Pi enclosures;
- small adapters;
- SSD-sized USB enclosures;
- lightweight support devices.

Mounting rules:

- do not cover intake/exhaust vents;
- do not place adhesive over service labels or removable covers;
- do not use data or power cables to support device weight;
- leave enough slack to disconnect the device without pulling adjacent cables;
- keep heavier equipment resting on the shelf rather than suspended by adhesive;
- use reusable straps where a device may be moved frequently.

The two SSD-sized USB-drive enclosures associated with `storage01` can be secured
to the top/side of the storage system if airflow, ports, and service access remain
clear.

---

# 5. Cable Routing

Ethernet cabling follows [`20-cabling-standard.md`](20-cabling-standard.md).

Physical routing goals:

- label both ends before disconnecting old cabling;
- use short patch cables for same-shelf connections;
- separate data cabling from large bundles of AC power where practical;
- avoid tight bends or strain at RJ45 connectors;
- bundle related cables with reusable hook-and-loop ties rather than permanent
  tight plastic ties;
- leave service loops only where useful;
- keep switch-port labels visible.

---

# 6. Before/After Evidence

Before moving equipment:

1. photograph the existing shelf and cable layout;
2. capture enough angles to reconstruct a connection if needed;
3. record cable IDs and endpoint names;
4. record any adapters that could be confused after removal.

After the rebuild:

1. photograph each equipment zone;
2. photograph `sw01` and visible port labeling;
3. photograph representative cable labels at both ends;
4. update the logical/current architecture diagram;
5. update the asset inventory if device placement or addressing changed.

The before/after images provide useful portfolio evidence because they show an
operational improvement rather than only a polished final shelf.

---

# 7. Serviceability Standard

A successful physical rebuild should make these tasks possible without major
disassembly:

- remove one Proxmox node;
- replace one Ethernet cable;
- reach the Cisco console/management path;
- service `storage01` or an external drive;
- identify the power source for one device;
- trace a cable from endpoint to switch;
- reboot or replace one Raspberry Pi;
- inspect UPS state.

If servicing one device requires unplugging several unrelated systems, the
layout should be improved.

---

# 8. Change Control

The physical rebuild is a planned infrastructure change and uses the controlled
shutdown/startup sequence in:

[`19-shutdown-startup-runbook.md`](19-shutdown-startup-runbook.md)

Pre-change controls include:

- current backups verified;
- Cisco baseline configuration backed up;
- current photos captured;
- labels prepared;
- rollback path known;
- AT&T fiber equipment intentionally left powered during the lab shutdown.

---

# 9. Skills Demonstrated

This physical-infrastructure work demonstrates:

- rack/shelf organization;
- power-dependency planning;
- UPS-aware device grouping;
- physical-layer documentation;
- airflow and serviceability planning;
- cable management;
- asset labeling;
- planned outage/change control;
- rollback preparation;
- before/after operational evidence.
