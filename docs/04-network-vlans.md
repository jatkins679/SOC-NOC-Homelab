# Cisco Switching, VLANs, and OPNsense

## Current State

The segmented lab network is operational. The original flat `192.168.1.0/24` network remains the Proxmox management and household network, while selected guests use tagged VLANs on `vmbr1`. OPNsense `fw01` routes and filters traffic between the VLANs and the flat network.

| Component | Current role |
|---|---|
| AT&T gateway | `192.168.1.254`; household Internet edge |
| TRENDnet TEG-S160G | Unmanaged flat-LAN switch |
| `sw01` Cisco SG350-10 | Managed VLAN switch at `192.168.1.21/24` |
| `fw01` | OPNsense VM 220 on `pve01`; `192.168.1.187` flat/WAN and `10.10.10.1` LABMGMT |
| Proxmox `vmbr0` | Flat management path on `192.168.1.0/24` |
| Proxmox `vmbr1` | VLAN-aware guest path through `sw01` |

## Physical Port Map

| `sw01` port | Connection | State |
|---|---|---|
| Gi1 | `pve01` secondary Ethernet / `vmbr1` | Up, trunk |
| Gi2 | `pve02` secondary Ethernet / `vmbr1` | Up, trunk |
| Gi3 | `pve03` secondary Ethernet / `vmbr1` | Up, trunk |
| Gi4 | `pve04` secondary Ethernet / `vmbr1` | Up, trunk |
| Gi8 | Upstream to the unmanaged flat-LAN switch | Up, root port |

The switch runs RSTP. On 2026-09-11, Gi8 showed no FCS, collision, carrier, symbol, or pause-frame errors. The exact initiator of that morning's transient Layer-2 disruption could not be proven because the unmanaged switch has no logs and the AT&T gateway had already discarded the relevant history.

## VLAN Matrix

| VLAN | OPNsense interface | Zone | Subnet | Gateway | Current guest |
|---:|---|---|---|---|---|
| 10 | `lan` / LABMGMT | Management | `10.10.10.0/24` | `10.10.10.1` | OPNsense management |
| 20 | `opt1` / SERVERS | Servers | `10.10.20.0/24` | `10.10.20.1` | `dc01` — `10.10.20.10` |
| 30 | `opt2` / USERS | Users | `10.10.30.0/24` | `10.10.30.1` | `win11-01` — `10.10.30.160` |
| 40 | `opt3` / SOCNOC | Monitoring | `10.10.40.0/24` | `10.10.40.1` | `nms01` — `10.10.40.10`; `wazuh01` — `10.10.40.20` |
| 50 | `opt4` / RED | Attack lab | `10.10.50.0/24` | `10.10.50.1` | `kali01` — `10.10.50.113` |
| 60 | `opt5` / DMZRANGE | Vulnerable targets | `10.10.60.0/24` | `10.10.60.1` | `target01` — `10.10.60.10` |

## Proxmox Guest Attachment

| VM | VMID / node | Bridge and tag |
|---|---|---|
| `dc01` | 100 / `pve01` | `vmbr1`, VLAN 20 |
| `win11-01` | 101 / `pve01` | `vmbr1`, VLAN 30 |
| `fw01` | 220 / `pve01` | `vmbr0` plus untagged/trunk `vmbr1` |
| `kali01` | 300 / `pve02` | `vmbr1`, VLAN 50 |
| `target01` | 400 / `pve02` | `vmbr1`, VLAN 60 |
| `nms01` | 102 / `pve03` | `vmbr1`, VLAN 40 |
| `wazuh01` | 500 / `pve03` | legacy `vmbr0` plus `vmbr1`, VLAN 40 |

## Firewall Policy Principles

- Rules are applied on the source interface and permit only documented service flows.
- VLAN 50 and VLAN 60 remain restricted except for explicit attack-lab and telemetry paths.
- Wazuh agents use TCP 1514 to `10.10.40.20`; TCP 1515 is allowed only where enrollment is required.
- Trusted traffic that enters OPNsense through the flat/WAN interface must use **Disable reply-to** when the source's normal gateway is the AT&T router. Without it, OPNsense can force replies toward the wrong gateway and break otherwise permitted TCP sessions.
- Every permitted flow must be tested from its actual source and, when needed, verified with OPNsense live logs or packet capture.

## Management Access

From John's Mac, create the OPNsense GUI tunnel and keep the terminal open:

```bash
ssh -N -L 8443:10.10.10.1:443 root@192.168.1.10
```

Then browse to `https://localhost:8443`.

## Migration Control

The guest address change is only one part of a migration. Monitoring databases, agent configuration, remote-access tools, DNS, automation, firewall rules, and documentation must be audited before the old address is removed. Follow [`24-vlan-migration-dependency-checklist.md`](24-vlan-migration-dependency-checklist.md) for every future migration. Remaining remediation is tracked in GitHub issue #1.

## Historical Baseline

Before segmentation, `sw01` used the default VLAN 1 on all ports and OPNsense was only planned. That state is retained in dated build/ticket records; it is no longer the current operating state.
