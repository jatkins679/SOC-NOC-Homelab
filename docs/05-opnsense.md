# OPNsense Routing and Firewalling

## Overview

I deployed OPNsense to provide routing and firewalling between the segmented networks in my homelab.

Before introducing VLANs, most systems shared the same `192.168.1.0/24` network. That was easy to manage but offered little separation between infrastructure services, client systems, security tools, and testing targets.

I wanted to control traffic between those systems without replacing the AT&T gateway used by the rest of the household.

OPNsense now provides routing between the lab VLANs while the AT&T gateway continues handling the primary home Internet connection.

This document covers the firewall configuration, the rules I've tested, and a routing problem I encountered during deployment.

---

## 1. OPNsense Deployment

OPNsense runs as a virtual machine on the Proxmox cluster.

| Setting | Value |
|---|---|
| Hostname | `fw01` |
| VMID | `220` |
| Proxmox node | `pve01` |
| Flat-network interface | `192.168.1.187/24` |
| Internal management address | `10.10.10.1` |
| Upstream gateway | `192.168.1.254` |

The flat-network interface is designated WAN in OPNsense, although it connects to my private home LAN rather than directly to a public Internet connection.

This distinction matters when configuring firewall rules and troubleshooting routes.

The AT&T gateway remains upstream, while OPNsense manages the segmented lab network.

---

## 2. Firewall Interfaces

Each lab VLAN has a separate logical interface in OPNsense.

| Interface | Name | Network | Gateway address |
|---|---|---|---|
| WAN | Flat network | `192.168.1.0/24` | OPNsense `192.168.1.187` |
| LAN | LABMGMT | `10.10.10.0/24` | `10.10.10.1` |
| OPT1 | SERVERS | `10.10.20.0/24` | `10.10.20.1` |
| OPT2 | USERS | `10.10.30.0/24` | `10.10.30.1` |
| OPT3 | SOCNOC | `10.10.40.0/24` | `10.10.40.1` |
| OPT4 | RED | `10.10.50.0/24` | `10.10.50.1` |
| OPT5 | DMZRANGE | `10.10.60.0/24` | `10.10.60.1` |

OPNsense routes between these networks and filters the traffic according to its firewall rules.

For example, a Windows endpoint on VLAN 30 communicates with the Wazuh server on VLAN 40 through OPNsense.

The rules determine whether that communication is permitted.

---

## 3. How the Firewall Rules Work

One of the important concepts I worked through was that OPNsense normally evaluates rules on the interface where a packet enters the firewall.

If a connection starts from VLAN 30 and is destined for VLAN 40, the rule normally belongs on the USERS interface.

The rule needs to account for:

- Source address or network
- Destination address or network
- Protocol
- Destination port
- Direction in which the connection starts

Because OPNsense uses stateful filtering, permitted return traffic is normally handled by the connection state.

That doesn't guarantee the session will work, however. The routing path still needs to be correct in both directions.

### Example: Windows Endpoint to Wazuh

A Windows client on the USERS VLAN communicates with the Wazuh manager on the SOC/Security VLAN.

```text
win11-01 — 10.10.30.160
          |
       VLAN 30
          |
       OPNsense
          |
       VLAN 40
          |
wazuh01 — 10.10.40.20
```

The documented Wazuh connection uses:

- TCP 1514 for agent communication
- TCP 1515 for agent enrollment when required

The corresponding rule permits the necessary traffic rather than opening unrestricted connectivity between the two VLANs.

---

## 4. Validated Firewall Rules

Some of the documented connections I've tested include:

| Source | Destination | Service |
|---|---|---|
| `win11-01` `10.10.30.160` | `wazuh01` `10.10.40.20` | TCP 1514–1515 |
| `target01` `10.10.60.10` | `wazuh01` `10.10.40.20` | TCP 1514–1515 |
| Trusted Mac `192.168.1.77` | `target01` `10.10.60.10` | SSH TCP 22 |
| VLAN 50 RED | Authorized VLAN 60 targets | Controlled test traffic |

These are examples of verified connectivity, not an exhaustive export of the current firewall rulebase.

The actual firewall configuration is maintained in OPNsense.

As the lab grows, I want rules to be specific enough that I can understand why a particular service is allowed through the firewall.

---

## 5. Troubleshooting the Reply-To Problem

One of the more frustrating networking problems during the build involved connecting to systems behind OPNsense from a trusted laptop on the original home LAN.

The firewall rule appeared correct, but connections still failed.

For example, my Mac on `192.168.1.77` needed to reach services on the segmented network.

The connection entered OPNsense through the interface designated WAN.

OPNsense automatically applies `reply-to` behavior in some WAN rule configurations. This can force response traffic toward a specified gateway instead of allowing the normal routing table to determine the return path.

In my configuration, the AT&T router at `192.168.1.254` was the upstream gateway.

That created a problem for traffic originating from another device on the same flat network.

### The problem

```text
Mac — 192.168.1.77
          |
          v
fw01 — 192.168.1.187
          |
          v
Internal VLAN service
```

The incoming connection could match an allow rule, but the return path could be directed incorrectly.

### The solution

I enabled **Disable reply-to** on the relevant trusted-access WAN firewall rule.

That allowed the response to follow the appropriate route back to the originating client.

After the change, I verified access to internal services, including NetBox.

The important point was that a firewall rule permitting a connection did not prove that the complete TCP session would work.

The return path mattered just as much.

---

## 6. Troubleshooting Firewall Connections

When a connection fails, I generally want to establish whether the problem involves addressing, routing, firewall policy, or the destination service.

Useful checks include:

1. Can the source reach its VLAN gateway?
2. Is the destination using the correct address?
3. Is the application listening on the expected port?
4. Does the firewall have a rule permitting the traffic?
5. Does the connection reach the destination?
6. Does the response return through the expected path?

### Connection Testing

For example, from a Linux or macOS client:

```bash
nc -vz 10.10.40.20 1514
```

This tests whether a TCP connection to the Wazuh agent port can be established.

It doesn't verify that the Wazuh application protocol is functioning correctly, but it is useful for checking basic TCP reachability.

On a Linux destination, I can check listening sockets:

```bash
sudo ss -lntp
```

OPNsense provides live firewall logs and packet capture tools that help identify whether traffic is blocked or whether packets are taking an unexpected route.

For TCP problems, I can examine the SYN, SYN-ACK, and ACK exchange.

A firewall log showing an allowed SYN is not proof that the connection completed successfully.

These are general troubleshooting methods; the particular commands are not all preserved as historical commands from the original deployment.

---

## 7. Accessing the OPNsense GUI

OPNsense's management GUI is reachable through the internal address:

`https://10.10.10.1`

When direct routing to that address isn't available from my laptop, I use SSH port forwarding through `pve01`.

From a terminal on my Mac:

```bash
ssh -N -L 8443:10.10.10.1:443 root@192.168.1.10
```

This establishes a tunnel to `pve01` and forwards local port 8443 to the OPNsense HTTPS interface.

I then open:

`https://localhost:8443`

The SSH session needs to remain running while I'm using the tunnel.

This has been a useful alternative when firewall or routing changes temporarily affect direct management access.

---

## 8. Firewall Configuration and Backups

I use OPNsense's configuration backup and export capabilities to retain copies of the firewall configuration.

The rulebase includes both manually configured rules and rules generated automatically by OPNsense.

That distinction matters when comparing counts between exported rule data and the total number of rules displayed in the interface.

I've also used CSV-based rule management during the build.

A CSV import doesn't necessarily represent every rule visible in the firewall interface.

For ongoing maintenance, I want configuration changes to be documented along with the reason for the change and the connectivity tests used to verify it.

---

## 9. Remaining Firewall Work

The firewall is operational, but the rulebase still needs some cleanup as the VLAN migration continues.

Remaining work includes:

- Reviewing duplicate or overlapping rules
- Removing unnecessarily broad permissions
- Checking DNS dependencies between networks
- Confirming monitoring traffic after address changes
- Updating remote-access rules and routes
- Removing legacy addresses once they're no longer needed
- Testing services from their actual source VLANs

These items are tracked in GitHub issue #1 and the VLAN migration dependency checklist.

[Migration Dependency Checklist](24-vlan-migration-dependency-checklist.md)

---

## 10. Result

OPNsense now provides routing and firewalling for the segmented lab.

The setup has given me practical experience with interface assignments, firewall rule direction, stateful connections, VLAN routing, and diagnosing asymmetric return paths.

The `reply-to` problem was particularly useful because the firewall initially appeared to permit the traffic, yet the application connections failed.

It reinforced the importance of verifying an entire connection rather than relying on one test or a single firewall log entry.

The firewall configuration will continue to evolve as additional systems move onto the VLANs and the default LAN is eventually renumbered.

## Related Documentation

- [Managed Switching and VLANs](04-network-vlans.md)
- [VLAN Migration Dependency Checklist](24-vlan-migration-dependency-checklist.md)
- [Tailscale Remote Access](23-tailscale-remote-access.md)
- [NOC Troubleshooting Runbook](13-noc-troubleshooting-runbook.md)
