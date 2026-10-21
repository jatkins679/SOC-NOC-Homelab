# NOC Network and Service Troubleshooting Runbook

## Overview

This is the troubleshooting reference I use for network and service problems in my homelab.

The environment includes Proxmox, Linux and Windows servers, managed switches, VLANs, OPNsense, DNS, shared storage, monitoring applications, and remote-access services.

With that many components, an outage can have several possible causes.

I try to establish what's actually failing before making configuration changes.

The general approach is to collect the current state, work through the relevant dependencies, make one controlled change, and verify the result from the system that originally experienced the problem.

---

## 1. Initial Assessment

When something stops working, I start with a few questions:

1. What service or system is affected?
2. Is the problem limited to one host, one VLAN, or multiple systems?
3. When did the problem start?
4. What changed shortly before the failure?
5. Can I reproduce it from another client?

The answers determine which part of the system needs attention.

For example, if a laptop can reach its local gateway but can't reach an external IP address, I investigate routing or VPN behavior before concentrating on DNS.

If DNS resolution fails but Internet IP connectivity works, the resolver configuration and DNS service become more likely causes.

I prefer read-only commands and observations before changing configuration.

---

## 2. Core Infrastructure Reference

These are documented addresses used throughout the runbook.

| Host | IP address | Purpose |
|---|---|---|
| AT&T gateway | `192.168.1.254` | Internet gateway |
| `pve01` | `192.168.1.10` | Proxmox |
| `pve02` | `192.168.1.11` | Proxmox |
| `pve03` | `192.168.1.12` | Proxmox |
| `pve04` | `192.168.1.13` | Proxmox |
| `dns01` | `192.168.1.20` | Pi-hole |
| `sw01` | `192.168.1.21` | Cisco SG350-10 |
| `sw02` | `192.168.1.22` | UniFi US-8 |
| `docker` | `192.168.1.174` | Docker services |
| `storage01` | `192.168.1.208` | SMB backup storage |
| `dc01` | `10.10.20.10` | Active Directory and DNS |
| `wazuh01` | `10.10.40.20` | Wazuh SIEM |
| `nms01` | `10.10.40.10` | LibreNMS |
| `fw01` | `192.168.1.187` / `10.10.10.1` | OPNsense |

These are documented values, not the result of a fresh discovery.

The original `192.168.1.0/24` network is still in use, alongside the segmented VLANs.

The planned move to another default-LAN subnet has not yet been completed.

---

## 3. Host or Service Unreachable

The first distinction is whether the host is unreachable or only one service is unavailable.

### Linux Checks

On the affected host:

```bash
ip -br link
ip -br addr
ip route
```

These show whether the interface is up, whether it has the expected address, and where traffic will be routed.

From a known-good client:

```bash
ping -c 4 192.168.1.174
```

This example tests `docker` (`192.168.1.174`).

If the host responds, I then test the application port.

For example:

```bash
nc -vz 192.168.1.174 3001
```

Port 3001 is used by the Uptime Kuma instance on `docker`.

A failed ping doesn't conclusively mean a host is down because ICMP may be filtered.

A successful TCP connection doesn't prove that the application is functioning correctly.

### What to Check Next

If the host is reachable but the application isn't, I inspect the service state, listening sockets, and logs.

```bash
systemctl --failed
sudo ss -lntp
```

For a systemd-managed service:

```bash
sudo systemctl status SERVICE --no-pager -l
sudo journalctl -u SERVICE --since "30 minutes ago" --no-pager
```

`SERVICE` is a placeholder for the actual service name.

For Docker workloads, I would inspect the container status and logs instead.

---

## 4. Internet Connectivity Failure

When a system can communicate on the local network but can't reach Internet services, I separate gateway connectivity, external routing, DNS, and VPN behavior.

On Linux:

```bash
ip -br addr
ip route
ping -c 3 192.168.1.254
```

I then test an external IP address and a hostname.

For example:

```bash
ping -c 3 1.1.1.1
dig example.com
```

If external IP connectivity works but name resolution fails, DNS becomes the primary focus.

If neither works, I examine the route and any VPN-related interfaces or policies.

### Mac VPN Troubleshooting Example

During a trip, one of my MacBooks connected successfully to Wi-Fi and could ping the local gateway, but couldn't access Internet addresses or hostnames.

The problem occurred on more than one Wi-Fi network.

It turned out that Surfshark VPN was still connected.

Disconnecting Surfshark restored Internet connectivity.

The important distinction was that Wi-Fi association and local gateway reachability did not prove that Internet routing was working.

This also gives me a reason to include active VPN configuration in future Mac connectivity checks rather than focusing only on Wi-Fi and DNS.

---

## 5. DNS Troubleshooting

My primary general-purpose DNS server is `dns01` (`192.168.1.20`), running Pi-hole on a physical Raspberry Pi.

I check the client's resolver configuration first.

On a system using systemd-resolved:

```bash
resolvectl status
```

Then I test Pi-hole directly:

```bash
dig @192.168.1.20 example.com
```

On `dns01`, I check the service:

```bash
pihole status
```

The troubleshooting path is:

```text
Client DNS settings
         |
         v
Network route to dns01
         |
         v
Pi-hole service / port 53
         |
         v
Upstream DNS
```

I also check whether the problem affects all clients or only a particular VLAN.

### Active Directory DNS

Domain-joined Windows clients use `dc01` (`10.10.20.10`) for domain DNS and service discovery.

Useful Windows checks include:

```powershell
ipconfig /all
Resolve-DnsName dc01.corp.home.arpa
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.corp.home.arpa
Test-ComputerSecureChannel
```

A failed domain login or trust operation may be caused by incorrect DNS, unavailable domain-controller services, routing, or an actual secure-channel problem.

---

## 6. VLAN and Firewall Connectivity

With VLAN segmentation, a host can have a valid IP address and still be unable to reach a service on another network.

For a connection between VLANs, I check:

1. Guest interface and VLAN tag
2. Proxmox bridge assignment
3. Switch trunk configuration
4. OPNsense gateway
5. Source-interface firewall rules
6. Return routing
7. Destination service and port

For example, `win11-01` on VLAN 30 needs to reach Wazuh on VLAN 40.

From Windows:

```powershell
Test-NetConnection 10.10.40.20 -Port 1514
```

I compare the test against OPNsense firewall logs.

A pass entry in the firewall log confirms a rule matched, but it doesn't necessarily establish that the TCP session completed.

### The OPNsense Reply-To Problem

I encountered this when attempting to reach internal lab services from a trusted Mac on the flat LAN.

The rule permitted the incoming traffic, but OPNsense's reply-to behavior could send responses along the wrong route.

Disabling reply-to on the appropriate trusted-access WAN rule corrected the return path.

The full incident is covered in [OPNsense Routing and Firewalling](05-opnsense.md).

---

## 7. Proxmox Cluster Troubleshooting

When a Proxmox node becomes unreachable or cluster operations behave unexpectedly, I start with the node's own connectivity and then check the cluster state.

On a Proxmox node:

```bash
hostname
hostname -f
ip -br addr
ip route
```

Then:

```bash
pvecm status
pvecm nodes
pvesh get /nodes
```

The four-node cluster normally has four voting members and requires three votes for quorum.

One node can become unavailable without necessarily losing quorum.

Two unavailable voting nodes would leave the cluster without a majority under the documented configuration.

### Quorum Versus VM Availability

Quorum governs cluster management and coordinated operation.

It doesn't mean every VM automatically restarts on another node.

The lab primarily uses local VM disks and shared backup storage rather than Ceph-backed automatic workload failover.

I therefore check guest state, storage, and recovery options separately from quorum.

---

## 8. VM or LXC Unavailable

On the relevant Proxmox node, I inspect the guest inventory.

Virtual machines:

```bash
qm list
```

Containers:

```bash
pct list
```

Then I inspect the specific configuration:

```bash
qm config 500
pct config 200
```

The examples refer to Wazuh VM 500 and Guacamole LXC 200.

Before attempting to start or restart a guest, I check whether it is already running and whether its storage is available.

After a guest starts, I verify networking and the application, not just its Proxmox status.

For a container:

```bash
pct exec 200 -- ip -br addr
```

For a VM, I can use the guest agent where available or connect to the guest console.

---

## 9. Shared Storage and Backup Problems

The Proxmox backup destination is provided through SMB/CIFS on `storage01` (`192.168.1.208`).

The Proxmox storage identifier is:

`t-20-backup`

### Initial Checks

On a Proxmox node:

```bash
pvesm status
pvesm list t-20-backup
```

I also inspect the mount and network connectivity:

```bash
findmnt /mnt/pve/t-20-backup
nc -nvz -w 3 192.168.1.208 445
```

A storage entry appearing in the Proxmox configuration doesn't prove that the underlying share is accessible.

### Windows SMB Network Profile Problem

I encountered a situation where the Windows storage server was running and the SMB share existed, but Proxmox couldn't reach it.

After a restart, Windows had classified the Ethernet connection as a Public network.

The Windows firewall policy then interfered with SMB connectivity.

On `storage01`, the useful PowerShell checks include:

```powershell
Get-Service LanmanServer
Get-SmbShare
Get-NetTCPConnection -LocalPort 445 -State Listen
Get-NetConnectionProfile
```

After confirming which trusted Ethernet interface was affected, correcting its network category to Private restored the expected SMB access.

The command used for that type of change is:

```powershell
Set-NetConnectionProfile -InterfaceIndex <index> -NetworkCategory Private
```

The interface index must be taken from the current system configuration.

I then retest TCP 445 and Proxmox storage access.

This was a case where the storage hardware and share were healthy, but a Windows network setting prevented clients from reaching them.

### Stale CIFS Mounts

Another problem occurred after a Windows share was removed and recreated under the same name.

A Proxmox node retained a stale CIFS mount.

The error included:

```text
Stale file handle
CIFS: VFS: reconnect tcon failed
```

I checked the mount:

```bash
findmnt /mnt/pve/t-20-backup
```

Once I had confirmed that no backup task was using the share, the stale mount could be detached:

```bash
sudo umount /mnt/pve/t-20-backup
```

I then checked whether Proxmox could access the configured storage again:

```bash
pvesm list t-20-backup
pvesm status
```

Because each Proxmox host maintains its own mount state, an issue on one node doesn't necessarily mean the other nodes have the same problem.

### Backup Write Failures

A backup can also fail when the destination becomes temporarily unavailable during a write.

One observed symptom involved a `zstd` write failure.

Useful kernel-log checks include:

```bash
journalctl -k --since "1 hour ago" --no-pager |
    grep -Ei 'cifs|smb|error|disconnect|reconnect'
```

I check whether other backups completed and which particular guest archive is missing before rerunning jobs.

The per-node backup schedules are staggered to reduce simultaneous writes against the storage system.

---

## 10. Filesystem and Capacity Problems

One of my Wazuh installation attempts failed because Ubuntu's root filesystem didn't have the capacity I expected.

The virtual disk was large enough in Proxmox, but the operating system's LVM configuration hadn't allocated all of the available capacity.

The initial checks were:

```bash
df -h /
lsblk
sudo pvs
sudo vgs
sudo lvs
```

The error ultimately caused OpenSearch to cross its flood-stage disk threshold and put an index into read-only mode.

Where unused extents were available in the volume group, I expanded the logical volume:

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

Then checked the filesystem again:

```bash
df -h /
```

The path and volume-group name are specific to that Ubuntu installation.

The broader point is to check the capacity available to the actual filesystem rather than relying on the virtual disk size shown in Proxmox.

---

## 11. Linux Service Failures

For systemd services, I start with their current state and logs.

```bash
systemctl is-active SERVICE
sudo systemctl status SERVICE --no-pager -l
sudo journalctl -xeu SERVICE --no-pager
```

For example, during a Wazuh indexer recovery attempt:

```bash
sudo systemctl status wazuh-indexer --no-pager -l
sudo journalctl -xeu wazuh-indexer.service --no-pager
```

The journal showed:

```text
java.nio.file.AccessDeniedException:
/etc/wazuh-indexer/backup
```

That was a permissions problem involving an indexer backup directory.

Investigating the actual exception was more useful than repeatedly restarting the service without understanding why it failed.

---

## 12. Wazuh Service and Agent Health

The main Wazuh services are:

- `wazuh-indexer`
- `wazuh-manager`
- `filebeat`
- `wazuh-dashboard`

On `wazuh01`, I can check them with:

```bash
sudo systemctl is-active wazuh-indexer
sudo systemctl is-active wazuh-manager
sudo systemctl is-active filebeat
sudo systemctl is-active wazuh-dashboard
```

On a Linux endpoint:

```bash
systemctl is-active wazuh-agent
```

If the agent is running but appears inactive in Wazuh, I check connectivity to the manager.

```bash
nc -zv 10.10.40.20 1514
nc -zv 10.10.40.20 1515
```

Port 1514 is used for agent communication; 1515 is used for enrollment when needed.

I then verify agent status and the arrival of fresh telemetry through the dashboard.

A running agent and open TCP port don't necessarily mean the monitoring pipeline is healthy.

---

## 13. Cisco Switch Troubleshooting

`sw01` is the Cisco SG350-10 at `192.168.1.21`.

The documented trunk connections include:

| Port | Connection |
|---|---|
| Gi1 | `pve01` |
| Gi2 | `pve02` |
| Gi3 | `pve03` |
| Gi4 | `pve04` |
| Gi5 | UniFi `sw02` |
| Gi8 | Flat-LAN uplink |

When a VLAN guest loses connectivity, I check the Proxmox bridge and tag, switch port state, and VLAN membership.

Useful Cisco commands include:

```text
show version
show vlan
```

For link problems, I also inspect interface status and error counters.

If several hosts lose connectivity together, I look at shared components—particularly the uplink, switch, and upstream gateway—before troubleshooting each host independently.

The managed network has evolved since the original flat-LAN switch migration, so historical port and VLAN configurations shouldn't be used as live state without checking.

---

## 14. Service Restoration

I don't consider an incident resolved simply because a process started.

I try to verify the complete function that originally failed.

For DNS, that means more than `pihole status`. A client also needs to resolve names successfully.

For Wazuh, the dashboard must work, agents must be connected, and fresh events must arrive.

For Proxmox, I need to confirm node connectivity, quorum, storage, and the relevant guests.

For a restored SMB backup share, I need to confirm that Proxmox can list or use the backup storage—not merely that the Windows file server is powered on.

This is especially important for services with several dependencies.

---

## 15. Incident Notes Template

```text
Date / time:
Affected system:
Affected service:
Reported symptom:

Expected behavior:
Observed behavior:

Scope:
Recent changes:

Network checks:
- Interface:
- Address:
- Gateway / route:
- DNS:
- Port:

Service checks:
- Service state:
- Logs:

Storage and resources:
- CPU:
- Memory:
- Filesystem:
- Backup / storage:

Findings:

Change made:

Validation:

Root cause:
Final status:
Follow-up:
```

If the root cause isn't established, I record that rather than substituting a guess.

For escalation or handoff, I also include the commands already run, relevant evidence, and what still needs investigation.

---

## Result

This runbook brings together troubleshooting procedures I've used while building and operating the homelab.

The main recurring problems have involved networking, DNS, storage, Proxmox, Windows services, and Linux applications.

The most useful habit has been separating symptoms from causes.

A running host doesn't guarantee a working application. A permitted firewall rule doesn't guarantee a successful TCP session. An accessible SMB server doesn't guarantee that Proxmox can use the share.

The objective is to verify the service from the perspective of the system that depends on it, and to document the evidence supporting the resolution.

## Related Documentation

- [Asset Inventory](02-asset-inventory.md)
- [Pi-hole Migration](02-pihole-migration.md)
- [Proxmox Cluster](03-proxmox-cluster-build.md)
- [Managed Switching and VLANs](04-network-vlans.md)
- [OPNsense Routing and Firewalling](05-opnsense.md)
- [Wazuh Deployment](06-wazuh-siem.md)
- [Storage and Backup Resilience](17-storage-backup-resilience.md)
- [Shutdown and Startup Runbook](19-shutdown-startup-runbook.md)
