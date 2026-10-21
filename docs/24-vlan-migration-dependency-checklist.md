# VLAN Migration Dependency Checklist

## Overview

I've been moving parts of my homelab from the original flat `192.168.1.0/24` network into separate VLANs.

The purpose is to separate management, servers, users, monitoring, and security-testing systems while controlling traffic between them through OPNsense.

The work has also demonstrated that changing a virtual machine's IP address is only one part of a network migration.

Monitoring targets, DNS records, firewall rules, remote-access connections, scripts, and other services may still depend on the old address.

During my October 2026 trip, I encountered another reason to reconsider the original addressing scheme: the Wi-Fi network where I was staying used the same `192.168.1.0/24` subnet as my home network, causing a routing conflict.

I'm therefore considering moving the remaining default LAN to `192.168.11.0/24`.

**That renumbering is planned, not completed.**

This checklist is intended to help me make incremental changes without losing access to the systems needed to recover from a failed migration.

---

## 1. VLAN Structure

The documented lab VLANs are:

| VLAN | Purpose | Subnet | Gateway |
|---|---|---|---|
| 10 | Management | `10.10.10.0/24` | `10.10.10.1` |
| 20 | Servers | `10.10.20.0/24` | `10.10.20.1` |
| 30 | Users | `10.10.30.0/24` | `10.10.30.1` |
| 40 | SOC / Security | `10.10.40.0/24` | `10.10.40.1` |
| 50 | Attack / Test | `10.10.50.0/24` | `10.10.50.1` |
| 60 | Guest / Test range | `10.10.60.0/24` | `10.10.60.1` |

The VLAN interface labels recorded in OPNsense include `LABMGMT`, `SERVERS`, `USERS`, `SOCNOC`, `RED`, and `DMZRANGE`.

Some labels reflect the original configuration and don't exactly match the current descriptive names. I'll reconcile those when reviewing the firewall.

`fw01` is OPNsense VMID 220 on `pve01`.

Its documented addresses include:

```text
Flat LAN / WAN: 192.168.1.187
Management:     10.10.10.1
```

The original `192.168.1.0/24` LAN continues to host Proxmox nodes, switches, storage, DNS, and other equipment.

---

## 2. Why I Want to Change the Default LAN

The original LAN is a common consumer-router subnet.

While traveling, my Mac received an address on a remote `192.168.1.0/24` network.

When I attempted to reach my Windows storage server at `192.168.1.208`, macOS selected the local Wi-Fi route instead of the Tailscale subnet route.

That was confirmed through the routing table.

The proposed replacement is:

```text
192.168.11.0/24
```

A less commonly used subnet would reduce the likelihood of encountering the same overlap.

It wouldn't guarantee that no future remote network would use that range.

Before selecting the final network, I want to check for conflicts with existing VLANs, VPN routes, and other private subnets used by the lab.

---

## 3. Systems Affected by Default-LAN Renumbering

The default-LAN change would affect considerably more than the AT&T gateway.

Examples of documented addresses include:

| System | Existing address |
|---|---|
| AT&T gateway | `192.168.1.254` |
| `pve01` | `192.168.1.10` |
| `pve02` | `192.168.1.11` |
| `pve03` | `192.168.1.12` |
| `pve04` | `192.168.1.13` |
| `dns01` | `192.168.1.20` |
| `sw01` | `192.168.1.21` |
| `sw02` | `192.168.1.22` |
| `docker` | `192.168.1.174` |
| `fw01` WAN | `192.168.1.187` |
| `storage01` | `192.168.1.208` |
| `tailscale01` | Previously `192.168.1.185` |
| `util01` | `192.168.1.243` |

These are documented addresses rather than a newly verified inventory.

Before renumbering anything, I need to distinguish DHCP leases, DHCP reservations, and manually configured static addresses.

The goal is to avoid overlooking equipment simply because it isn't in the Proxmox inventory.

---

## 4. Pre-Migration Inventory

Before moving a guest or changing an infrastructure address, I record:

- Hostname and existing address
- Intended VLAN and new address
- Gateway and DNS settings
- VM/LXC ID and Proxmox node
- Bridge and VLAN tag
- Physical switch port, where applicable
- Inbound and outbound network dependencies
- Monitoring targets
- Remote administration method
- Configuration backup or rollback method

For a Proxmox guest, I check its network configuration before changing it.

For example:

```bash
qm config 500
```

For containers:

```bash
pct config 200
```

The actual VMID depends on the system being migrated.

I also search the repository and active configuration files for the existing IP address and hostname.

Documentation search alone is insufficient because dependencies may exist in applications, databases, scheduled tasks, or monitoring configuration.

---

## 5. Firewall and Routing Dependencies

Before migrating a service to another VLAN, I identify the communication it needs.

For example, a Wazuh agent must be able to communicate with the Wazuh manager at `10.10.40.20`, normally using TCP 1514.

A domain-joined Windows workstation requires access to domain-controller services on `dc01` (`10.10.20.10`).

LibreNMS needs network access to the systems and devices it monitors.

For each required connection, I record:

```text
Source system:
Source VLAN:
Destination system:
Destination VLAN:
Protocol:
Destination port:
Expected route:
Applicable firewall rule:
Validation test:
```

### OPNsense Reply-To

I encountered a problem with reply-to behavior on OPNsense rules permitting trusted access from the original LAN.

A rule could permit incoming traffic while replies returned through an unintended gateway.

Disabling reply-to on the appropriate rule corrected the path.

This is a specific concern when the OPNsense WAN interface is also connected to the original home LAN.

I check both directions of the traffic before concluding that a connection is working.

---

## 6. Service Dependencies

An address change can affect many services beyond the migrated host.

| Service | What needs checking |
|---|---|
| Pi-hole | Local records, conditional forwarding, client resolver configuration |
| Active Directory DNS | A and SRV records, client DNS settings, domain-controller discovery |
| LibreNMS | Device records, SNMP targets and service reachability |
| Wazuh | Manager addresses, enrollment and agent connectivity |
| Uptime Kuma | Hostnames, monitor IPs, ports and URLs |
| Grafana / Prometheus | Scrape targets and dashboard queries |
| NetBox | Device interfaces, IP assignments and VLAN records |
| Guacamole | SSH/RDP connection destinations |
| Tailscale | Advertised subnet routes and reachable management addresses |
| Nginx Proxy Manager | Proxy upstream targets and access lists |
| Automation | Ansible inventory, scripts, SSH configuration and scheduled tasks |
| PiAlert | ARP discovery scope, which does not cross routed VLANs |

This is why I want to migrate one system or group of closely related systems at a time.

---

## 7. Migrating an Individual Guest

The general procedure is:

1. Confirm the guest and its dependencies.
2. Record its existing address, route, DNS and VLAN configuration.
3. Back up or export the relevant configuration.
4. Prepare the necessary OPNsense rules.
5. Prepare new DNS and monitoring records.
6. Change the guest's VLAN tag or network interface configuration.
7. Assign and verify the new IP address.
8. Test the default gateway.
9. Test required DNS and application connections.
10. Verify access from real client systems.
11. Update monitoring and remote-access targets.
12. Remove obsolete records only after validation.
13. Update documentation.

A guest successfully pinging its new gateway isn't enough to declare the migration complete.

I also want to confirm the services that depend on it continue working.

---

## 8. Default-LAN Migration Plan

The proposed default-LAN renumbering requires more preparation because it affects several foundational services simultaneously.

### Stage 1 — Verify the Current Network

Before making changes, I want to verify:

- AT&T gateway configuration and DHCP range
- Static addresses and DHCP reservations
- Proxmox management interfaces
- Switch management IPs
- Pi-hole DNS settings
- Storage-server SMB access
- Tailscale subnet advertising
- OPNsense WAN configuration and routes
- Firewall rules and aliases
- Monitoring and automation targets

I also want to identify which clients can provide local administrative access if routing or DNS stops working.

### Stage 2 — Prepare the New Address Plan

The candidate is `192.168.11.0/24`.

A straightforward starting point would preserve host numbers where practical—for example, `pve01` could move from `192.168.1.10` to `192.168.11.10`.

That is a proposal, not an assigned or implemented address.

I would check reservations and conflicts before finalizing each destination.

The gateway address, DHCP scope, static assignments, DNS resolvers, and management connections need to be coordinated.

### Stage 3 — Preserve an Administrative Recovery Path

Before changing the gateway or switch management addresses, I want an independent way to reach the equipment.

That may involve a directly connected laptop, temporary static addressing, or another verified management path.

I don't want the only available management session to depend on the subnet being changed.

### Stage 4 — Migrate in Controlled Groups

I would start with low-dependency systems, verify connectivity, and then address shared infrastructure.

Foundational services such as DNS, storage, Proxmox management, and OPNsense need specific transition plans rather than arbitrary changes.

I would avoid simultaneously changing the AT&T gateway, Proxmox interfaces, switch management, and Pi-hole.

### Stage 5 — Update Dependencies

As addresses change, I would update DNS records, reservations, monitoring, proxy targets, scripts, Tailscale subnet advertisements, and documentation.

For the Tailscale router, that includes updating the subnet route and verifying it has been approved and accepted by remote clients.

The old and new routes should be handled deliberately to avoid leaving an obsolete remote-access path in place.

### Stage 6 — Confirm the New Network

The migration is successful only when:

- The gateway and switch infrastructure are accessible.
- Proxmox nodes and guests are reachable.
- DNS works from ordinary clients.
- OPNsense routes traffic as intended.
- Storage and backups are usable.
- Monitoring targets have recovered.
- Remote access works from outside the home LAN.
- No unexplained dependencies remain on the old network.

---

## 9. Rollback Criteria

I would stop or reverse a migration if it causes broad loss of management access, DNS, storage, routing, or other essential services.

The specific rollback must be prepared before making the change.

For an individual guest, that usually means restoring its original VLAN tag, address, gateway, and DNS settings.

For the default LAN, a rollback may require restoring the previous gateway subnet, DHCP configuration, static addresses, switch management settings, and dependent routing.

I would verify that the original `192.168.1.0/24` network is functional again before resuming.

A rollback should restore a known working configuration rather than introduce another set of simultaneous changes.

---

## 10. Post-Migration Verification

After moving a guest, I check the result from several locations.

On the guest:

```bash
ip -br addr
ip route
resolvectl status
```

For Windows systems:

```powershell
ipconfig /all
route print
```

I also test the service from an actual client and confirm the connection in the relevant application.

For Wazuh, that means receiving fresh telemetry.

For LibreNMS, I check successful polling.

For Guacamole, I test an actual remote session.

For Proxmox storage, I confirm the backup share is accessible and archives can be listed.

I then search for the old address again.

Remaining references are classified as current configuration, historical records, backup data, or old logs.

Historical IP addresses don't need to be erased from incident documentation, but active configuration shouldn't silently continue pointing at retired addresses.

---

## 11. Existing Dependency Audit

A previous migration audit was recorded in GitHub issue #1.

At the time of that audit, items requiring additional work included LibreNMS device records and bindings, Wazuh-related settings, Uptime Kuma targets, Guacamole connections, Pi-hole records, and PiAlert's discovery limitations.

The audit also recorded that `target01` had successfully connected to the Wazuh manager on TCP 1514.

These are historical observations.

Before another migration, I'll review the issue against the current live configuration and determine which items remain open.

I don't want to close an issue simply because a document was updated, or retain an obsolete warning after the underlying problem has been resolved.

---

## 12. Migration Record Template

```text
Change date:
System:
VMID / CTID:
Proxmox node:

Original VLAN:
Original IP:
Original gateway:
Original DNS:

New VLAN:
New IP:
New gateway:
New DNS:

Network dependencies:
Firewall changes:
DNS changes:
Monitoring changes:
Remote-access changes:

Pre-change backup:
Rollback procedure:

Gateway test:
DNS test:
Application test:
Client-side test:
Monitoring verification:

Result:
Remaining issues:
Documentation updated:
```

This gives me a way to record both the changes and the evidence that the migrated system still works.

---

## Result

The VLAN migration is an ongoing project.

Several services have already been moved into segmented networks, but the original flat LAN still hosts important infrastructure.

The next major change under consideration is renumbering the default LAN to reduce address overlap during remote access.

I want to approach that as a coordinated infrastructure change rather than a collection of isolated IP edits.

The main requirement is to preserve management access and verify every dependent service before considering a migration complete.

## Related Documentation

- [Managed Switching and VLANs](04-network-vlans.md)
- [OPNsense Routing and Firewalling](05-opnsense.md)
- [Tailscale Remote Access](23-tailscale-remote-access.md)
- [Asset Inventory](02-asset-inventory.md)
- [NOC Troubleshooting Runbook](13-noc-troubleshooting-runbook.md)
- [Shutdown and Startup Runbook](19-shutdown-startup-runbook.md)
