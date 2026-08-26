# Cisco Managed Switching and VLAN Design

## Purpose

This document tracks the managed-switch phase of the SOC/NOC homelab.

It intentionally separates three different states:

1. **Completed switch staging** — the Cisco SG350-10 has been received, identified,
   given a unique management address, checked from the CLI, and backed up.
2. **Physical homelab deployment** — moving lab Ethernet connections to `sw01`
   while preserving the TRENDnet switch for the existing home network.
3. **Future segmentation** — VLANs, trunks, OPNsense routing, SNMP, syslog, and
   SPAN are documented as planned until they are implemented and validated.

The goal is to show actual network-engineering work without presenting a future
architecture as though it already exists.

---

# 1. Current Network State

The existing flat management/home network remains:

```text
192.168.1.0/24
Gateway: 192.168.1.254
```

The current home-network switch is:

```text
sw-home01
TRENDnet TEG-S160G
16-port unmanaged Ethernet switch
```

The Cisco switch is the managed **homelab** switch. It is not intended to replace
`sw-home01` as the general household switch.

---

# 2. Cisco SG350-10 Baseline

| Item | Verified / Current State |
|---|---|
| Hostname | `sw01` |
| Model | Cisco SG350-10 |
| Management address | `192.168.1.21/24` |
| Active firmware image | `2.5.9.55` |
| Inactive firmware image | `2.5.0.83` |
| Configuration backup | Baseline startup configuration downloaded |
| VLAN state | Default VLAN 1; segmentation not yet implemented |
| Deployment state | Staged; physical homelab migration pending |

The fixed management address was assigned before the switch was introduced into
the live LAN so that it would not conflict with another device.

---

# 3. Firmware Validation

The switch firmware was checked from the CLI with:

```text
show version
```

Observed image state:

```text
Active-image: flash://system/images/image_tesla_hybrid_2.5.9.55_release_cisco_signed.bin
Version: 2.5.9.55
Date: 20-Jan-2026

Inactive-image: flash://system/images/image1.bin
Version: 2.5.0.83
Date: 18-Jun-2019
```

This establishes a configuration/firmware baseline before the switch begins
carrying homelab traffic.

---

# 4. VLAN Baseline

The VLAN database was checked with:

```text
show vlan
```

Observed baseline:

```text
Vlan  Name  Tagged Ports  UnTagged Ports  Created by
----  ----  ------------  --------------  ----------
1     1                   gi1-10,Po1-8    DV
```

This is important because it confirms that the switch is still in a default flat
Layer-2 state. The repository therefore does **not** claim that VLAN segmentation
is already working.

---

# 5. Initial Physical Port Convention

The currently accepted initial convention is:

| Switch port | Intended connection | State |
|---|---|---|
| `Gi1` | `pve01` | Planned physical connection |
| `Gi2` | `pve02` | Planned physical connection |
| `Gi3` | `pve03` | Planned physical connection |
| `Gi4` | `pve04` | Planned physical connection |
| `Gi8` | Upstream / BGW320 | Planned physical connection |
| Other ports | `storage01`, `mgmt01`, `dns01`, future devices | Record during rebuild |

The cable labels deliberately keep the switch-port field visible so the final
port assignment can be recorded at the time of connection rather than recalled
later from memory.

---

# 6. Physical Deployment Sequence

Before VLAN work begins, the first deployment objective is deliberately simple:

```text
Move homelab Ethernet links to sw01
        ↓
Remain on the existing flat/default VLAN
        ↓
Verify management reachability and services
        ↓
Only then begin segmentation work
```

This reduces the number of variables changed at one time.

Post-move validation should include:

- `sw01` management reachability at `192.168.1.21`;
- `pve01` through `pve04` management reachability;
- four-node Proxmox quorum;
- `dns01` name resolution;
- `storage01` and `t-20-backup` reachability;
- Wazuh and monitored endpoint connectivity;
- Internet access through the existing AT&T gateway.

---

# 7. Planned VLAN Design

The current target VLAN set is:

| VLAN | Intended role | Status |
|---:|---|---|
| 10 | Management | Planned |
| 20 | Infrastructure / servers | Planned |
| 30 | User / endpoint systems | Planned |
| 40 | SOC / monitoring | Planned |
| 50 | Attack / testing | Planned |
| 60 | Isolated / vulnerable systems | Planned |

These IDs and roles are design targets, not claims of an operational segmented
network.

---

# 8. Planned Routing and Security Phase

Future work includes:

- 802.1Q trunking;
- VLAN-aware Proxmox bridges where required;
- OPNsense as `fw01`;
- inter-VLAN routing;
- firewall policy between security zones;
- explicit isolation of attack/vulnerable networks from the home LAN;
- DHCP/DNS decisions per VLAN;
- validation of permitted and denied paths.

A useful milestone will be a test endpoint on VLAN 30 that can reach its intended
gateway and Internet path while VLAN 50/60 systems remain restricted from the
home network.

---

# 9. Planned Observability

After switching and VLANs are stable, `sw01` can become an infrastructure
telemetry source through:

- SNMPv3 for NOC monitoring;
- syslog for centralized event review;
- interface/error counters;
- spanning-tree state;
- MAC-address table review;
- SPAN / port mirroring for selected packet-analysis exercises.

These features remain planned until their configuration and collected evidence
are documented.

---

# 10. Change and Rollback Strategy

The first physical move should have a simple rollback:

1. preserve the known-good `sw01` baseline;
2. move only the intended lab links;
3. verify the flat network before changing VLANs;
4. if broad connectivity fails, reconnect affected lab links to the prior
   TRENDnet path;
5. restore `192.168.1.0/24` service first;
6. review cable labels, switch ports, and VLAN state before retrying.

The TRENDnet switch remains available as the known-good home-network path during
this phase.

---

# 11. Evidence to Add as Work Continues

Useful future evidence includes:

- sanitized `show interfaces status` output;
- `show vlan` after each segmentation phase;
- trunk configuration and validation;
- switch-port/cable mapping;
- interface counter review;
- MAC-table and spanning-tree inspection;
- SNMPv3 monitoring screenshots;
- syslog evidence;
- SPAN capture exercise;
- before/after physical photographs.

Sensitive values, passwords, secrets, and private configuration data should not
be committed.

---

# 12. Skills Demonstrated So Far

The staging work already demonstrates:

- managed-switch console/CLI access;
- hostname and management-address planning;
- firmware-state validation;
- VLAN database inspection;
- configuration backup;
- planned port mapping;
- pre-change baseline capture;
- rollback thinking;
- separation of implemented state from future design.

VLAN trunking, inter-VLAN routing, SNMPv3, syslog, and SPAN will move from
planned to demonstrated only after they are implemented and validated.
