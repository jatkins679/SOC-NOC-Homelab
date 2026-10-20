# Managed Switching, VLANs, and OPNsense

## Overview

The homelab originally used the same flat network as the rest of my house.

Everything was connected through unmanaged switches and used addresses in `192.168.1.0/24`.

That was sufficient when I was running a handful of virtual machines, but it became less useful as I added security-testing systems, monitoring servers, Windows Active Directory, and more network services.

I wanted to separate different kinds of traffic, control which systems could communicate, and gain practical experience configuring managed switches, VLANs, and a firewall.

I now use a Cisco SG350-10, a UniFi US-8, and OPNsense to support the segmented portion of the lab.

The original LAN remains operational, while selected virtual machines use separate VLANs. The migration is ongoing.

---

## 1. Network Overview

The main network components are:

| Device | IP address | Role |
|---|---|---|
| AT&T BGW320 gateway | `192.168.1.254` | Household Internet gateway |
| `sw-home01` | Unmanaged | TRENDnet TEG-S160G |
| `sw01` | `192.168.1.21` | Cisco SG350-10 managed switch |
| `sw02` | `192.168.1.22` | UniFi US-8 managed switch |
| `unifi01` | `192.168.1.23` | UniFi controller |
| `fw01` | `192.168.1.187` | OPNsense firewall, flat-network interface |

OPNsense also has an internal management interface at `10.10.10.1`.

The Proxmox hosts use two separate networking paths.

**`vmbr0`** carries the existing management and flat-LAN connectivity.

**`vmbr1`** provides VLAN-aware connectivity through the managed switch infrastructure.

This lets me introduce VLANs without immediately moving every device off the original network.

---

## 2. Physical Switching

The Cisco SG350-10 is the main managed switch for the lab.

Its current documented connections are:

| Port | Connection | Configuration |
|---|---|---|
| Gi1 | `pve01` secondary Ethernet | Trunk |
| Gi2 | `pve02` secondary Ethernet | Trunk |
| Gi3 | `pve03` secondary Ethernet | Trunk |
| Gi4 | `pve04` secondary Ethernet | Trunk |
| Gi5 | `sw02` UniFi US-8 | Trunk |
| Gi8 | Existing flat-LAN switch | Uplink |

On the UniFi US-8, port 1 is the trunk uplink to `sw01`.

Ports 2–8 are currently configured as access ports on VLAN 1.

### Trunk and Access Ports

An access port carries traffic for one VLAN and normally presents it untagged to the connected device.

A trunk carries traffic for multiple VLANs using IEEE 802.1Q tags.

The trunks between `sw01` and the Proxmox nodes allow virtual machines assigned to different VLANs to share a physical network connection.

The same principle applies to the trunk connecting `sw01` and `sw02`.

The documented port configuration should be checked against the live switch state before making physical or VLAN changes.

---

## 3. VLAN Design

I divided the lab into six logical networks.

| VLAN | Name | Subnet | Gateway |
|---|---|---|---|
| 10 | Management | `10.10.10.0/24` | `10.10.10.1` |
| 20 | Servers | `10.10.20.0/24` | `10.10.20.1` |
| 30 | Users | `10.10.30.0/24` | `10.10.30.1` |
| 40 | SOC/Security | `10.10.40.0/24` | `10.10.40.1` |
| 50 | Attack/Test | `10.10.50.0/24` | `10.10.50.1` |
| 60 | Guest/Targets | `10.10.60.0/24` | `10.10.60.1` |

The first objective was to separate systems by function.

The second was to make the communication between those networks explicit rather than allowing everything to reach everything else.

### Management — VLAN 10

Used for management interfaces and administrative services.

Examples include the OPNsense management interface and NetBox.

### Servers — VLAN 20

Used for infrastructure servers such as the Active Directory domain controller.

### Users — VLAN 30

Used for Windows and other client systems.

### SOC/Security — VLAN 40

Used for security and monitoring services, including Wazuh and LibreNMS.

### Attack/Test — VLAN 50

Used for Kali Linux and other controlled security-testing systems.

### Guest/Targets — VLAN 60

Used for test targets and systems that should be separated from the rest of the network.

The names describe the intended functions. Actual permitted connectivity is determined by the firewall rules, not by the VLAN names alone.

---

## 4. Guest Network Assignments

Some virtual machines have already been moved onto the segmented network.

The documented assignments include:

| Host | IP address | VLAN |
|---|---|---|
| `dc01` | `10.10.20.10` | 20 |
| `win11-01` | `10.10.30.160` | 30 |
| `nms01` | `10.10.40.10` | 40 |
| `wazuh01` | `10.10.40.20` | 40 |
| `kali01` | `10.10.50.113` | 50 |
| `target01` | `10.10.60.10` | 60 |
| `nb01` | `10.10.10.15` | 10 |

These are the documented lab addresses, rather than a live discovery of every system.

Some hosts may retain legacy interfaces or addresses while the migration is being completed.

### Proxmox Configuration

For a VLAN-assigned guest, Proxmox connects its virtual NIC to `vmbr1` and applies the appropriate VLAN tag.

For example:

```text
win11-01
    |
Proxmox virtual NIC — VLAN 30
    |
vmbr1
    |
sw01 trunk
    |
OPNsense — 10.10.30.1
```

That virtual machine can use its VLAN gateway to reach other permitted networks.

The original `vmbr0` path remains separate from this VLAN-aware guest network.

---

## 5. OPNsense Routing

OPNsense runs as VM 220 (`fw01`) on `pve01`.

It connects the VLANs and applies the firewall rules controlling traffic between them.

The original flat-network interface uses:

`192.168.1.187`

The internal management interface uses:

`10.10.10.1`

Each segmented network has an OPNsense gateway address.

For example, traffic from a client on VLAN 30 to a service on VLAN 40 normally passes through OPNsense.

That gives me a place to control and inspect the communication.

### Firewall Policy

The general approach is to permit the traffic a system needs rather than simply allowing unrestricted communication between VLANs.

Examples include:

- Allowing clients to reach required DNS and domain services
- Allowing Wazuh agents to reach the Wazuh manager
- Allowing trusted management systems to reach administrative interfaces
- Restricting communication from attack and test networks
- Checking firewall logs when expected traffic fails

Wazuh agent traffic normally uses TCP port 1514.

Agent enrollment may also require TCP port 1515.

Rules must account for the direction in which traffic begins, since OPNsense evaluates rules on the interface where traffic enters the firewall.

---

## 6. Troubleshooting an OPNsense Routing Problem

One of the more useful problems I encountered involved reaching internal services from a trusted laptop on the original LAN.

The firewall rule appeared to allow the connection, but the TCP session still failed.

I eventually traced the problem to OPNsense's `reply-to` behavior.

The traffic entered OPNsense through its flat-network interface, while the originating laptop normally used the AT&T gateway.

The automatically applied reply-to behavior could force response traffic toward an inappropriate gateway.

Disabling reply-to on the relevant trusted-access rule corrected the return path.

This resolved access problems involving internal services, including NetBox.

The problem was a good example of why checking only whether a firewall rule permits a packet isn't enough. The return path also has to work.

---

## 7. Management Access

OPNsense's internal GUI is available at `10.10.10.1`.

When direct access isn't convenient, I can use an SSH tunnel through `pve01` (`192.168.1.10`).

From a Mac terminal:

```bash
ssh -N -L 8443:10.10.10.1:443 root@192.168.1.10
```

The tunnel forwards the Mac's local port 8443 to the internal OPNsense HTTPS interface.

With the tunnel running, I open:

`https://localhost:8443`

This provides access to the OPNsense GUI without requiring the management interface to be directly reachable from every network.

---

## 8. Network Incident and Diagnostics

During the network build, I also encountered a transient Layer-2 disruption on September 11, 2026.

I investigated switch connectivity, interface counters, and spanning-tree behavior.

The Cisco switch runs Rapid Spanning Tree Protocol (RSTP).

The documented inspection of Gi8 showed no FCS, collision, carrier, symbol, or pause-frame errors.

I couldn't conclusively determine the original cause of the disruption because the upstream unmanaged switch had no event logs and the AT&T gateway no longer had the relevant historical information.

I retained the incident in the documentation rather than assigning a cause that couldn't be demonstrated.

---

## 9. Planned Default-LAN Address Change

The original home LAN uses:

`192.168.1.0/24`

While traveling, I encountered a network using the same address range.

That created a routing conflict when I tried to reach the home lab remotely.

I used SSH tunneling and Tailscale to work around the immediate problem, but the experience made me reconsider the default addressing scheme.

One candidate is:

`192.168.11.0/24`

This change is **planned, not implemented**.

It is separate from the existing lab VLANs using the `10.10.x.0/24` networks.

Changing the default LAN will require accounting for more than just the AT&T gateway.

The affected systems and services include:

- Proxmox management addresses and cluster communications
- Switch management addresses
- Pi-hole and other statically addressed infrastructure
- OPNsense routing and firewall rules
- Tailscale subnet route advertisements
- DNS records and forwarding configuration
- Monitoring tools and service integrations
- Client DHCP and default gateway configuration
- Remote administration and rollback access

I want to document and validate the dependencies before beginning the renumbering.

The [VLAN Migration Dependency Checklist](24-vlan-migration-dependency-checklist.md) provides the framework for that work.

---

## 10. Migration and Validation Approach

I've been moving services onto the VLANs incrementally rather than trying to change the entire network at once.

For each migration, the basic process is:

1. Record the original interface and address.
2. Confirm the intended VLAN and gateway.
3. Check switch port and Proxmox bridge configuration.
4. Review the required OPNsense rules.
5. Move or add the network interface.
6. Verify addressing, gateway access, and DNS.
7. Test the actual application or monitoring traffic.
8. Check logs and monitoring systems.
9. Update inventory and documentation.
10. Remove the old path only after the replacement works.

This is particularly important for systems used to manage or monitor the rest of the lab.

A service can appear healthy locally while still being unreachable from the systems that depend on it.

---

## 11. Current Result

The lab now has operational managed switching, several VLAN-backed virtual machines, and OPNsense routing between the segmented networks.

The original flat network is still in use, and the migration is not complete.

The next steps are to finish auditing remaining service dependencies, improve documentation of the physical and logical network, and plan the eventual default-LAN address change.

The larger objective is to have a network that's easier to understand, troubleshoot, monitor, and use for security experiments without unnecessarily affecting the rest of the household.

## Related Documentation

- [OPNsense Configuration](05-opnsense.md)
- [Asset Inventory](02-asset-inventory.md)
- [VLAN Migration Dependency Checklist](24-vlan-migration-dependency-checklist.md)
- [Tailscale Remote Access](23-tailscale-remote-access.md)
- [NOC Troubleshooting Runbook](13-noc-troubleshooting-runbook.md)
