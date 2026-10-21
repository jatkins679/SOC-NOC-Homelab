# Homelab Shutdown and Startup Runbook

## Overview

This runbook documents how I shut down and restart the homelab when maintenance requires taking multiple systems offline.

The lab contains more dependencies than it did when I first started assembling it. Proxmox hosts run infrastructure services, the storage server provides shared backups, OPNsense routes between VLANs, and several Raspberry Pis provide DNS, remote access, and monitoring.

Turning everything off and back on without considering those dependencies can create problems that look like system failures.

The objective is to shut down workloads cleanly, preserve access to storage and network services as long as they're needed, and verify that everything returns to a working state.

This procedure is intended for a planned full-lab shutdown. A normal Proxmox node update or individual service restart doesn't require taking down the entire environment.

---

## 1. Important Dependencies

The main dependencies are:

| System | Role | Dependency |
|---|---|---|
| AT&T gateway | Internet and original LAN gateway | Physical power and fiber connection |
| `sw01`, `sw02` | Managed network switching | Power and uplinks |
| `dns01` | Pi-hole DNS | Network switching |
| `tailscale01` | Remote access | Internet connectivity and local network |
| `util01` | UPS/NUT monitoring | Network and UPS communication |
| `storage01` | Backup storage | Windows, Drobo, SMB, network |
| `pve01`–`pve04` | Virtualization hosts | Power, management network |
| `fw01` | OPNsense inter-VLAN routing | Proxmox host and VLAN networking |
| `dc01` | Active Directory and DNS | Proxmox, network, OPNsense for routed access |
| `docker` | Containerized applications | Proxmox, network |
| `wazuh01`, `nms01`, `nb01` | Monitoring and infrastructure services | Proxmox and routed connectivity |

I need to keep in mind that starting a Proxmox node doesn't automatically mean its guests or the services they provide are ready.

---

## 2. UPS Identification

The UPS naming convention is:

| UPS | Identification |
|---|---|
| `UPS01` | Older large UPS |
| `UPS02` | APC SUA1000 |
| `UPS03` | APC BE850G2 |

The physical power distribution is documented separately.

Before changing connections or performing a full shutdown, I want to identify which systems depend on each UPS rather than relying on the unit's name or assumed capacity.

NUT monitoring through `util01` is part of the power-management arrangement.

---

## 3. Before Shutting Down

I start with the cluster and storage state.

On a Proxmox node:

```bash
pvecm status
pvesh get /nodes
pvesm status
```

I also review which guests are running:

```bash
qm list
pct list
```

Before taking everything offline, I confirm that recent backups exist on `t-20-backup` and that storage isn't currently being written to by an active backup job.

```bash
pvesm list t-20-backup
```

For physical changes, I also photograph the current layout and identify cables and power connections.

My pre-shutdown checklist is:

- [ ] Confirm the maintenance scope.
- [ ] Check cluster quorum and node status.
- [ ] Identify running VMs and containers.
- [ ] Verify relevant backups.
- [ ] Confirm no backup or storage maintenance operation is active.
- [ ] Review any unresolved hardware or service problems.
- [ ] Record physical connections before moving equipment.
- [ ] Keep a local management client available for startup checks.
- [ ] Identify a rollback path if networking will change.

For planned maintenance, I normally leave the AT&T fiber gateway powered unless the work specifically requires shutting it down.

---

## 4. Controlled Shutdown

The shutdown order follows the direction of dependencies:

```text
Applications and test workloads
               |
               v
Monitoring and infrastructure guests
               |
               v
Routing / firewall guests
               |
               v
Proxmox hosts
               |
               v
Storage and support systems
               |
               v
Network equipment being serviced
```

The exact order may vary depending on which systems are being maintained.

### Step 1 — Shut Down Nonessential Guests

I begin with workloads that aren't needed for managing the shutdown.

That includes attack-test systems, vulnerability scanners, and other optional applications.

Guests should be shut down through their operating system or Proxmox rather than abruptly removing host power.

I verify their state using:

```bash
qm list
pct list
```

### Step 2 — Shut Down Monitoring and Application Guests

Next are services such as Wazuh, LibreNMS, NetBox, and Docker applications.

The goal is to stop application workloads before the infrastructure they depend on.

### Step 3 — Shut Down Identity and Routing Guests

I shut down `dc01` after the applications and services that rely on Active Directory are stopped.

I keep `fw01` running while I still need inter-VLAN access for shutdown or administration.

Because OPNsense is virtualized, shutting down its Proxmox host or the firewall guest itself will interrupt routed VLAN access.

I therefore finish the necessary remote or cross-VLAN work before stopping it.

### Step 4 — Shut Down the Proxmox Nodes

Once the guests have been stopped, I shut down the hypervisors.

The documented order is:

1. `pve04` — `192.168.1.13`
2. `pve03` — `192.168.1.12`
3. `pve02` — `192.168.1.11`
4. `pve01` — `192.168.1.10`

I confirm that each host has shut down cleanly.

The order is a working procedure, not a requirement imposed by Proxmox. It can be adjusted to account for where critical guests are running.

### Step 5 — Shut Down Storage

After Proxmox no longer needs access to the backup share, I shut down `storage01` cleanly.

I let Windows complete its shutdown before removing power from attached storage devices.

The Drobo shouldn't be disconnected while the server is still performing writes.

### Step 6 — Shut Down Remaining Support Devices

For a full physical shutdown, I then stop the remaining support hosts:

1. `dns01` — `192.168.1.20`
2. `tailscale01` — documented LAN address `192.168.1.185`
3. `util01` — `192.168.1.243`

I leave `util01` until last among these systems because it provides UPS/NUT monitoring.

Shutting down DNS and Tailscale will affect household name resolution and remote access, so I only include them when the maintenance actually requires it.

### Step 7 — Network and Power Equipment

Finally, I power down the switches and other equipment that needs to be moved or rewired.

I leave the AT&T gateway running when possible.

---

## 5. Physical Maintenance

Once the affected systems are down, I can work on cabling, power distribution, equipment placement, or switch connections.

Before restoring power, I check that:

- Every device has the intended power connection.
- Ethernet cables connect to the recorded switch ports.
- USB storage connections are secure.
- No ventilation openings are obstructed.
- Power adapters and cables aren't under strain.
- No equipment is supported by its cables.

For network changes, I avoid combining unrelated firewall, VLAN, and physical-cabling modifications into one difficult-to-reverse operation.

---

## 6. Controlled Startup

Startup reverses the broad dependency order, with networking and storage available before the applications that use them.

```text
Network switching
        |
        v
DNS, storage and support systems
        |
        v
Proxmox hosts
        |
        v
OPNsense and network routing
        |
        v
Active Directory and core services
        |
        v
Monitoring and applications
        |
        v
Functional verification
```

### Step 1 — Restore Network Switching

I power on the switches and verify the intended uplinks.

The main switch addresses are:

| System | Address |
|---|---|
| `sw01` | `192.168.1.21` |
| `sw02` | `192.168.1.22` |

I confirm that relevant link ports are active and that the management network is reachable.

OPNsense routing won't be available until its Proxmox host and VM have started.

### Step 2 — Start DNS and Storage

I bring up the independent support systems and the storage server.

The primary systems are:

| System | Address |
|---|---|
| `dns01` | `192.168.1.20` |
| `tailscale01` | `192.168.1.185` — previously documented address |
| `util01` | `192.168.1.243` |
| `storage01` | `192.168.1.208` |

I confirm DNS service and that the Drobo and other disks are mounted before depending on shared storage.

On `storage01`, I can check:

```powershell
Get-Service LanmanServer
Get-SmbShare
Get-NetConnectionProfile
```

After a previous power outage, Windows incorrectly classified the Ethernet connection as Public, blocking inbound SMB access.

That is now one of the first things I check if storage is unavailable after a reboot.

### Step 3 — Start Proxmox

The documented startup order is:

1. `pve01` — `192.168.1.10`
2. `pve02` — `192.168.1.11`
3. `pve03` — `192.168.1.12`
4. `pve04` — `192.168.1.13`

I allow time for the hosts and cluster services to initialize.

Then check:

```bash
pvecm status
pvecm nodes
pvesh get /nodes
pvesm status
```

The expected normal cluster state is four nodes, four votes, and a quorum requirement of three.

### Step 4 — Verify OPNsense

I confirm that `fw01` has started before expecting routed VLAN workloads to be reachable.

The relevant documented OPNsense interfaces include:

```text
WAN / original LAN: 192.168.1.187
Management VLAN:    10.10.10.1
```

I check the firewall's interface state and routed connectivity between the networks that depend on it.

A Proxmox guest can be running while it remains unreachable because OPNsense hasn't finished starting.

### Step 5 — Start Core Services

After networking and routing are available, I confirm the main infrastructure services.

These include:

- `dc01` — Active Directory and domain DNS
- `docker` — containerized services
- `nb01` — NetBox
- `nms01` — LibreNMS
- `wazuh01` — Wazuh
- Apache Guacamole for remote management

Some guests are configured to start automatically.

I verify their state rather than starting duplicate instances or assuming that the configured boot order completed successfully.

Optional test systems can be started afterward as needed.

---

## 7. Post-Startup Verification

The maintenance isn't finished simply because all four Proxmox nodes are online.

I check the actual services.

### Network

- [ ] AT&T gateway reachable at `192.168.1.254`.
- [ ] Managed switches reachable.
- [ ] All four Proxmox nodes reachable.
- [ ] OPNsense interfaces and routing working.
- [ ] Expected VLAN connectivity restored.
- [ ] Internet access works from an appropriate client.

### DNS and Identity

- [ ] `dns01` answers DNS queries.
- [ ] `dc01` is available.
- [ ] `corp.home.arpa` resolves.
- [ ] Domain authentication works where applicable.

### Storage

- [ ] `storage01` and the Drobo are available.
- [ ] SMB port 445 is reachable.
- [ ] `t-20-backup` is active.
- [ ] Proxmox can list existing backups.

### Virtualization and Applications

- [ ] Cluster has quorum.
- [ ] Required VMs and containers are running.
- [ ] Docker applications respond.
- [ ] Wazuh services and agent telemetry return.
- [ ] LibreNMS polling resumes.
- [ ] Uptime Kuma monitoring recovers.
- [ ] Remote-access services work as expected.

If something isn't working, I document the failure and check its dependencies rather than restarting unrelated systems.

---

## 8. Rollback and Recovery

For physical network changes, I want a known-good configuration to return to if the new cabling or switching arrangement doesn't work.

If a network change interrupts several systems, I stop further modifications, inspect the switch and physical connections, and restore the known-good path when necessary.

The original physical-rebuild plan included moving lab connections back to the existing TRENDnet switch if the initial Cisco migration failed.

That remains useful historical rollback context, although the current network now uses managed VLANs and OPNsense routing.

A rollback today must account for those dependencies rather than assuming every device still belongs on a flat untagged LAN.

I document what failed and what remained working before trying the change again.

---

## 9. Maintenance Record

```text
Date:
Reason for shutdown:
Affected systems:

Pre-shutdown checks:
Backup status:
Cluster state:
Storage status:

Shutdown completed:
Physical or configuration changes:

Startup completed:
Cluster verification:
Network verification:
Storage verification:
Application verification:

Problems encountered:
Corrective actions:
Remaining issues:
Final status:
```

I want the record to distinguish the planned change, any unexpected failures, and the actual outcome.

---

## Result

The homelab has enough interdependent systems that a full shutdown needs a defined order.

The main considerations are graceful guest shutdown, preserving routing and management access until it's no longer needed, bringing storage and network services online before dependent workloads, and validating applications afterward.

The process will continue to evolve as I change VLANs, improve UPS integration, and move services between Proxmox nodes.

## Related Documentation

- [Physical Infrastructure](18-physical-infrastructure.md)
- [Cabling Standard](20-cabling-standard.md)
- [Proxmox Cluster](03-proxmox-cluster-build.md)
- [Storage and Backup Resilience](17-storage-backup-resilience.md)
- [NOC Troubleshooting Runbook](13-noc-troubleshooting-runbook.md)
- [Homelab Operations Checklist](15-homelab-operations-checklist.md)
