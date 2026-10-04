# Homelab Shutdown and Startup Runbook

## Purpose

This runbook defines the controlled shutdown and startup process used when the
homelab is intentionally taken fully offline for physical rearrangement,
recabling, switch work, or other infrastructure maintenance.

For the current physical rebuild, the lab is powered down while the **AT&T fiber
box/gateway remains powered**.

The objective is to avoid treating a planned outage like an uncontrolled power
loss and to verify dependencies in a deliberate order during recovery.

---

# 1. Change Scope

## UPS Naming Convention

```text
UPS01  older of the two large UPS units
UPS02  newer of the two large UPS units - APC SMC1000
UPS03  small UPS / surge protector - APC BE850G2
```

These names are the canonical identifiers for power-distribution and future NUT
shutdown policy. Earlier references that used `UPS02` for the small UPS are
superseded; the small UPS is now `UPS03`.

---

Typical work covered by this runbook:

- moving equipment between shelves;
- replacing or reorganizing Ethernet cables;
- connecting the Cisco SG350-10 (`sw01`);
- moving power loads between `UPS01`, `UPS02`, and `UPS03` outputs;
- securing small devices;
- reorganizing storage peripherals;
- documenting final physical layout.

---

# 2. Pre-Shutdown Checks

Before powering anything down:

- [ ] Confirm the planned outage window and scope.
- [ ] Take current photographs of shelves, cable paths, and power connections.
- [ ] Confirm both ends of Ethernet cables are labeled or can be positively identified.
- [ ] Confirm the Proxmox cluster is quorate.
- [ ] Confirm all expected VMs/LXCs are accounted for.
- [ ] Confirm recent Proxmox backups exist on `t-20-backup`.
- [ ] Confirm `storage01` / Drobo health and backup-target reachability.
- [ ] Confirm the `sw01` baseline configuration backup is available.
- [ ] Record any temporary DHCP addresses or unusual service state.
- [ ] Keep the management Mac available for startup validation.

Useful Proxmox checks:

```bash
pvecm status
pvesh get /nodes
qm list
pct list
pvesm status
```

Expected cluster baseline:

```text
Nodes:          4
Expected votes: 4
Total votes:    4
Quorum:         3
Quorate:        Yes
```

---

# 3. Controlled Shutdown Order

The guiding rule is:

```text
Applications / guests
        ↓
Compute hosts
        ↓
Storage / support services
        ↓
Network equipment being moved
```

## Step 1 — Stop or Shut Down Workloads

Gracefully shut down VMs and LXCs that will not remain online.

Verify they are stopped rather than assuming the shutdown request completed.

```bash
qm list
pct list
```

## Step 2 — Shut Down Proxmox Nodes

After guests are down, shut down the cluster nodes cleanly in the documented maintenance order:

1. `pve04` — `192.168.1.13`
2. `pve03` — `192.168.1.12`
3. `pve02` — `192.168.1.11`
4. `pve01` — `192.168.1.10`

Do not simply remove UPS/power-strip power from running Proxmox hosts.

## Step 3 — Shut Down Storage

After compute workloads no longer depend on shared storage:

- shut down `storage01` cleanly;
- allow Windows to complete shutdown;
- power down attached storage only after host I/O has stopped.

## Step 4 — Shut Down Remaining Lab Support Devices

After compute and storage dependencies are down, shut down the remaining support hosts that are part of the physical work:

1. `dns01` — `192.168.1.20`
2. `tailscale01` — `192.168.1.185`
3. `util01` — `192.168.1.243` — shut down last among the support hosts because it provides UPS/NUT monitoring

The resulting household DNS and homelab remote-access outage is expected during this planned full-lab maintenance window.

## Step 5 — Power Down Switches / Lab Power as Needed

Power down the switching and UPS-fed lab devices that must be moved or recabled.

**Leave the AT&T fiber box/gateway powered.**

---

# 4. Physical Work Window

With the lab safely down:

1. move equipment into the NETWORK / COMPUTE / STORAGE / POWER zones;
2. attach prepared cable labels;
3. route short same-shelf patch cables;
4. connect `sw01` according to the current port/cable record;
5. secure lightweight devices with removable hook-and-loop mounting where
   appropriate;
6. confirm no vents are blocked;
7. confirm no cable is carrying device weight;
8. photograph the completed physical arrangement before applying power.

Avoid making undocumented logical/VLAN changes during a physical-rebuild step.
The first recovery target is the known flat `192.168.1.0/24` network.

---

# 5. Controlled Startup Order

The startup order follows dependencies from the network outward:

```text
Network path
    ↓
DNS / storage support
    ↓
Proxmox hosts
    ↓
Cluster quorum
    ↓
VMs / LXCs
    ↓
Monitoring and application validation
```

## Step 1 — Restore Switching

Power the required switch infrastructure first.

For the current Cisco/OPNsense deployment:

- confirm `sw01` powers normally;
- confirm expected link lights;
- confirm management reachability at `192.168.1.21` when the management path is available;
- verify Gi1-Gi4 trunks, Gi8 upstream, and the configured VLAN membership.

`fw01` is a Proxmox guest, so its service validation occurs after the Proxmox hosts are available. Do not expect routed VLAN guests to be reachable until `fw01` has started.

## Step 2 — Start DNS and Storage Dependencies

Power and validate:

- `dns01` (`192.168.1.20`);
- `tailscale01` (`192.168.1.185`);
- `util01` (`192.168.1.243`);
- `storage01` (`192.168.1.208`);
- Drobo / attached storage.

Confirm basic DNS and SMB/CIFS availability before relying on them from the
cluster.

After an unexpected power loss or reboot, validate the Windows storage host
before assuming the Proxmox CIFS configuration has failed.

On `storage01`:

```powershell
Get-Service LanmanServer
Get-SmbShare
Get-NetConnectionProfile
```

Confirm:

- the Drobo volume is mounted;
- the `ProxmoxBackups` share exists;
- `LanmanServer` is running;
- the active Ethernet connection is classified as `Private`.

From a Proxmox node, verify SMB reachability:

```bash
nc -nvz -w 3 192.168.1.208 445
```

A Windows Ethernet profile that changes to `Public` can allow the local SMB
service to remain healthy while Windows Firewall blocks remote access.

## Step 3 — Start Proxmox Nodes

Power up the Proxmox nodes in this order:

1. `pve01` — `192.168.1.10`
2. `pve02` — `192.168.1.11`
3. `pve03` — `192.168.1.12`
4. `pve04` — `192.168.1.13`

Allow each node to complete boot before interpreting cluster state.

## Step 4 — Validate Cluster Health

From a cluster node:

```bash
pvecm status
pvesh get /nodes
pvesm status
```

Healthy target:

```text
Nodes:          4
Total votes:    4
Quorum:         3
Quorate:        Yes
```

Confirm the shared backup target is visible.

## Step 5 — Start / Verify Workloads

Check the expected guest list and startup state:

```bash
qm list
pct list
```

Start or validate intentionally manual-start workloads in dependency order:

1. `fw01` — OPNsense routing/firewall
2. `dc01` — Active Directory / DNS
3. `docker` — Docker application host at `192.168.1.174`
4. `nb01` — NetBox at `10.10.10.15`
5. `nms01` — LibreNMS at `10.10.40.10`
6. `wazuh01` — Wazuh SIEM
7. `apache-guacamole` — remote-access gateway at `192.168.1.151`

Then start or verify remaining application/test workloads as needed.

---

# 6. Post-Startup Validation

Do not declare the change successful merely because devices have power.

Validate at least:

## Network

- [ ] AT&T gateway reachable at `192.168.1.254`.
- [ ] `sw01` reachable at `192.168.1.21` when connected.
- [ ] `pve01`-`pve04` reachable.
- [ ] Internet connectivity works from a known client.

## DNS / Identity

- [ ] `dns01` answers expected DNS queries.
- [ ] `dc01` reachable at `10.10.20.10`.
- [ ] `corp.home.arpa` resolution works.
- [ ] `win11-01` retains domain connectivity.

## Storage / Backups

- [ ] `storage01` reachable at `192.168.1.208`.
- [ ] Drobo / attached storage mounted normally.
- [ ] Windows Ethernet profile on `storage01` is `Private`.
- [ ] SMB TCP 445 reachable from a Proxmox node.
- [ ] `t-20-backup` active in Proxmox.
- [ ] `pvesm list t-20-backup` returns expected backup contents.
- [ ] No storage device reports an unexpected fault.

## Proxmox

- [ ] Four nodes visible.
- [ ] Cluster quorate.
- [ ] Expected VMs/LXCs visible.
- [ ] Workloads start as intended.

## Security / Monitoring

- [ ] `wazuh01` reachable.
- [ ] Wazuh services healthy.
- [ ] `docker` reachable at `192.168.1.174` and key containers are healthy.
- [ ] Uptime Kuma, Nginx Proxy Manager, Homarr, Grafana, and Prometheus respond as expected.
- [ ] `nb01` reachable at `10.10.10.15`.
- [ ] `nms01` reachable at `10.10.40.10` and LibreNMS polling resumes.
- [ ] `apache-guacamole` reachable at `192.168.1.151`.
- [ ] `tailscale01` reconnects and advertises the intended subnet route.
- [ ] Monitored endpoints reconnect.
- [ ] No important telemetry path was lost during recabling.

---

# 7. Rollback Criteria

Rollback is appropriate when the physical/network change creates broad impact
that cannot be isolated quickly and safely.

Examples:

- multiple lab systems lose network connectivity after the Cisco move;
- `sw01` cannot be managed and port/VLAN state is uncertain;
- DNS or storage dependencies cannot be restored over the new physical path;
- cabling cannot be confidently mapped.

Rollback procedure:

1. stop additional configuration changes;
2. preserve logs/configuration state where useful;
3. reconnect affected lab systems to the previously known-good TRENDnet path;
4. restore the flat `192.168.1.0/24` network;
5. validate DNS, storage, Proxmox quorum, and key workloads;
6. document what failed before scheduling another attempt.

The TRENDnet remains the home-network switch, so this rollback concerns homelab
connections rather than replacing household switching.

---

# 8. Change Closure

After successful startup:

- [ ] Photograph the final physical arrangement.
- [ ] Record the final `C##` cable-to-port mapping.
- [ ] Update `02-asset-inventory.md` if addresses/roles changed.
- [ ] Update `diagrams/current-architecture.md` if the live switch path changed.
- [ ] Update `04-network-vlans.md` with the actual post-move switch state.
- [ ] Record any unexpected issue as a ticket/change example.
- [ ] Run `git diff --check` before committing documentation changes.

---

# 9. Skills Demonstrated

This runbook demonstrates:

- planned outage management;
- dependency-aware shutdown/startup sequencing;
- graceful virtualization shutdown;
- storage dependency handling;
- network-first recovery;
- cluster/quorum validation;
- service restoration checks;
- physical change management;
- rollback criteria;
- post-change documentation.
