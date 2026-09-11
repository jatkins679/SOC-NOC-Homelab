# VLAN Migration Dependency Checklist

## Purpose

Use this checklist before and after moving any asset from the flat `192.168.1.0/24` network to a routed VLAN. A migration is not complete when the guest has its new address; every consumer, firewall path, monitoring target, name record, and operational document must also be updated and tested.

## Current VLAN and Gateway Matrix

| VLAN | OPNsense interface | Zone | Subnet | Gateway | Current assets |
|---:|---|---|---|---|---|
| 10 | `lan` / LABMGMT | Management | `10.10.10.0/24` | `10.10.10.1` | Reserved for management services |
| 20 | `opt1` / SERVERS | Servers | `10.10.20.0/24` | `10.10.20.1` | `dc01` (`10.10.20.10`) |
| 30 | `opt2` / USERS | Users | `10.10.30.0/24` | `10.10.30.1` | `win11-01` (`10.10.30.160`) |
| 40 | `opt3` / SOCNOC | Monitoring | `10.10.40.0/24` | `10.10.40.1` | `nms01` (`10.10.40.10`), `wazuh01` (`10.10.40.20`) |
| 50 | `opt4` / RED | Attack lab | `10.10.50.0/24` | `10.10.50.1` | `kali01` (`10.10.50.113`) |
| 60 | `opt5` / DMZRANGE | Vulnerable targets | `10.10.60.0/24` | `10.10.60.1` | `target01` (`10.10.60.10`) |

The Proxmox nodes and selected supporting services remain on the flat `192.168.1.0/24` management/home network. OPNsense `fw01` is VM 220 on `pve01`; its flat-side address is `192.168.1.187`, and its internal management address is `10.10.10.1`.

## Required Pre-Migration Audit

- [ ] Record the asset's old address, new static address, VLAN, gateway, DNS servers, Proxmox node, VM/LXC ID, bridge, tag, and MAC address.
- [ ] Search the repository for the old IP and hostname.
- [ ] Search active configuration on monitoring, remote-access, DNS, automation, and application hosts.
- [ ] Inventory inbound and outbound service flows, including source, destination, protocol, port, and required reply path.
- [ ] Create narrow OPNsense rules on the source interface before moving the asset.
- [ ] For trusted traffic entering OPNsense through WAN/flat LAN, enable **Disable reply-to** to prevent asymmetric replies through the AT&T gateway.
- [ ] Create or update stable DNS records before clients are changed to use the new name/address.
- [ ] Export or back up every configuration or database that will be modified.
- [ ] Define explicit success tests and a rollback point.

## Dependency Consumers to Check

| Consumer | Items to inspect |
|---|---|
| OPNsense | Interface rules, aliases, NAT, policy routing, reply-to behavior, logging |
| LibreNMS | `devices.hostname`, SNMP reachability, web-server bind/name, local SNMP bind |
| Wazuh | Agent manager/enrollment addresses, agent keys, manager listeners, local SNMP bind |
| Uptime Kuma | Monitor URL, hostname, port, TLS behavior, firewall path |
| Guacamole | Connection hostname/IP and port; firewall path from `192.168.1.151` |
| DNS | Pi-hole local records, AD-integrated DNS records, forwarders, client resolvers |
| PiAlert | Scan scope and the fact that ARP discovery does not cross routed VLANs |
| Automation | Ansible inventory, scripts, scheduled jobs, SSH config, health checks |
| Documentation | README, inventory, diagrams, runbooks, shutdown/startup checks, lab procedures |

## Required Post-Migration Validation

- [ ] Confirm the guest reports the expected address, route, DNS servers, and interface state.
- [ ] Test the gateway and required destinations from the migrated guest.
- [ ] Test each required flow from its real source; a firewall rule existing is not proof that the flow works.
- [ ] Verify return traffic and inspect OPNsense live logs or packet capture when a TCP handshake fails.
- [ ] Verify LibreNMS polling and alert recovery.
- [ ] Verify Wazuh agents appear Active and their logs show connection to `10.10.40.20:1514`.
- [ ] Verify Uptime Kuma and Guacamole targets after their records are updated.
- [ ] Verify forward and reverse DNS where applicable.
- [ ] Search again for the old address and classify every remaining result as active, historical, backup, cache, or log evidence.
- [ ] Update current-state documentation and link the validation evidence to the tracking issue.
- [ ] Remove the old interface/address only after all dependencies pass.

## Known Audit Work — GitHub Issue #1

The post-migration dependency audit is tracked in GitHub issue #1. At the time of this documentation update:

- `target01` now uses `10.10.40.20` for Wazuh and has successfully connected on TCP 1514.
- LibreNMS device records for `nms01`, `dc01`, and `wazuh01` still require live remediation.
- LibreNMS nginx/SNMP bindings and the Wazuh SNMP binding still require live remediation.
- The Uptime Kuma Wazuh dashboard monitor and six Guacamole connections still require coordinated firewall and target updates.
- Stable Pi-hole records for migrated Linux systems still need to be created and verified.
- PiAlert remains limited to its local flat broadcast domain unless a routed/distributed discovery method is implemented.

Do not migrate another asset until issue #1's active dependency items have been remediated or explicitly accepted as tracked exceptions.
