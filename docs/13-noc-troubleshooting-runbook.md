# NOC Network and Service Troubleshooting Runbook

## Purpose

This runbook documents the troubleshooting process I use in the SOC/NOC
homelab for infrastructure and service incidents.

The goal is to use a repeatable operations workflow rather than making several
changes at once and hoping one fixes the problem.

Typical incidents covered here include:

- Host unreachable
- Service unavailable
- DNS failure
- Internet connectivity failure
- Proxmox node or cluster problem
- VM or LXC unavailable
- Shared-storage / backup failure
- Disk-capacity or filesystem problem
- Wazuh or another Linux service failure
- Windows endpoint or domain-connectivity problem

The core principle is:

> Collect state first, isolate the failing layer, change one thing, and verify
> the result.

---

# 1. Standard Troubleshooting Workflow

```text
Alert / user report
        |
        v
Identify affected service and scope
        |
        v
Confirm the problem independently
        |
        v
Check physical / link state
        |
        v
Check IP addressing
        |
        v
Check routing
        |
        v
Check DNS
        |
        v
Check port / service reachability
        |
        v
Check service state and logs
        |
        v
Check storage / CPU / memory
        |
        v
Check dependencies
        |
        v
Make one controlled change
        |
        v
Retest
        |
        v
Verify service restoration
        |
        v
Document cause and resolution
```

This sequence is adjusted depending on the symptom, but the general method
remains the same.

---

# 2. First Five Questions

Before changing anything, determine:

1. **What is affected?**
2. **Is the issue one host, one service, one subnet, or the whole environment?**
3. **When did it start?**
4. **What changed immediately before it started?**
5. **Can I reproduce the problem from another system?**

Examples:

- If one VM cannot reach the Internet but other systems can, the ISP is probably
  not the first place to investigate.
- If several clients can ping `192.168.1.20` but cannot resolve names, focus on
  DNS rather than Ethernet.
- If the Proxmox GUI is unavailable but the host answers SSH, focus on Proxmox
  services rather than host power.

---

# 3. Current Core Infrastructure Reference

| System | Role | Address |
|---|---|---:|
| AT&T gateway | Internet edge / default gateway | `192.168.1.254` |
| `mgmt01` | Independent Linux management host | `192.168.1.5` |
| `pve01` | Proxmox cluster node | `192.168.1.10` |
| `pve02` | Proxmox cluster node | `192.168.1.11` |
| `pve03` | Proxmox cluster node | `192.168.1.12` |
| `pve04` | Proxmox cluster node | `192.168.1.13` |
| `dns01` | Pi-hole DNS | `192.168.1.20` |
| `dc01` | Active Directory / DNS | `192.168.1.30` |
| `apache-guacamole` | Remote-access gateway | `192.168.1.151` |
| `wazuh01` | Wazuh SIEM | `192.168.1.206` |
| `kali01` | Security testing | `192.168.1.211` |
| `target01` | Ubuntu monitored target | `192.168.1.238` |
| `t-20-backup` | Proxmox CIFS backup storage | Shared storage target |

The primary management network is currently:

```text
192.168.1.0/24
```

with default gateway:

```text
192.168.1.254
```

---

# 4. Incident Priority

A simple lab priority model:

| Priority | Example |
|---|---|
| **P1 — Critical** | Broad network outage, DNS unavailable to most clients, storage failure threatening data |
| **P2 — High** | Proxmox cluster degraded, important service unavailable, multiple systems affected |
| **P3 — Medium** | One host or application unavailable, backup target inaccessible |
| **P4 — Low** | Intermittent issue, noncritical lab workload, minor configuration problem |
| **P5 — Maintenance / Test** | Planned outage, controlled troubleshooting exercise |

Priority should be based on impact, not merely how interesting the technical
problem is.

---

# 5. Host Unreachable Runbook

## Symptom

A host cannot be reached by ping, SSH, RDP, HTTP, or another expected protocol.

## Step 1 — Confirm the Address

Check the asset inventory and, when applicable, DHCP state.

Do not assume a DHCP-observed address is permanent.

## Step 2 — Test Basic Reachability

From `mgmt01` or another known-good host:

```bash
ping -c 4 192.168.1.X
```

Interpretation:

- Reply received: Layer 3 path exists.
- No reply: host may be down, filtered, incorrectly addressed, or unreachable.

Remember that ICMP can be blocked, so ping failure alone does not prove the host
is down.

## Step 3 — Test the Required Port

Examples:

```bash
nc -vz 192.168.1.X 22
nc -vz 192.168.1.X 443
nc -vz 192.168.1.X 3389
```

This separates:

```text
host reachable
```

from:

```text
required application port reachable
```

## Step 4 — Check Local Interface State

On Linux:

```bash
ip -br addr
ip -br link
```

Look for:

- expected interface
- `UP` state
- expected IPv4 address
- unexpected additional or missing interfaces

## Step 5 — Check Routing

```bash
ip route
```

Verify:

- local subnet route
- default route
- correct interface
- correct gateway

---

# 6. Internet Connectivity Runbook

## Symptom

A host can reach local systems but cannot reach Internet services.

Work outward from the host.

## Test 1 — Local Interface

```bash
ip -br addr
```

## Test 2 — Gateway

```bash
ping -c 3 192.168.1.254
```

If the gateway cannot be reached, investigate local addressing, link state,
bridge configuration, switch connection, or host firewall.

## Test 3 — External IP

A known external IP test can distinguish routing from DNS.

If external IP connectivity works but names fail, investigate DNS.

## Test 4 — DNS Name

```bash
ping -c 3 google.com
```

Interpretation:

```text
gateway works
external IP works
hostname fails
```

strongly points toward DNS.

---

# 7. DNS Failure Runbook

DNS failures can look like general network outages.

The lab intentionally separates DNS from Proxmox by hosting Pi-hole on the
physical `dns01` Raspberry Pi.

## Client Checks

### Inspect Resolver State

```bash
resolvectl status
```

Confirm the client is using the expected resolver.

For systems intended to use Pi-hole directly:

```text
192.168.1.20
```

### Test the DNS Host

```bash
ping -c 3 192.168.1.20
```

### Test Resolution

Use an available DNS tool such as:

```bash
dig example.com
```

or:

```bash
nslookup example.com
```

## Server Checks on `dns01`

```bash
pihole status
```

Confirm:

- DNS service running
- FTL operational
- filtering enabled

Port 53 should be available for DNS.

A useful layered diagnosis is:

```text
Client
  |
  v
Client resolver configuration
  |
  v
Network path to 192.168.1.20
  |
  v
Pi-hole / FTL
  |
  v
Upstream DNS
```

Do not change all layers at once.

---

# 8. Active Directory DNS / Domain Connectivity

Windows domain problems are frequently DNS problems.

## On the Windows Endpoint

Inspect configured DNS:

```powershell
ipconfig /all
```

Verify domain discovery:

```powershell
nslookup dc01.corp.home.arpa
```

Verify the secure channel:

```powershell
Test-ComputerSecureChannel -Verbose
```

Useful questions:

- Is the endpoint using the AD DNS server when required?
- Does `dc01.corp.home.arpa` resolve?
- Are AD locator records available?
- Is the workstation still joined to `corp.home.arpa`?
- Does the secure channel validate?

Avoid treating an AD authentication failure as a password problem until DNS and
domain discovery have been checked.

---

# 9. Proxmox Node Troubleshooting

## Symptom

A Proxmox node appears offline, cannot be administered, or disappears from the
cluster.

## Check Host Identity

```bash
hostname
hostname -f
```

Stable hostnames matter to cluster membership and certificates.

## Check Networking

```bash
ip -br addr
ip route
```

Verify the management address and default route.

Expected cluster node addresses are:

```text
pve01  192.168.1.10
pve02  192.168.1.11
pve03  192.168.1.12
pve04  192.168.1.13
```

## Check Cluster State

```bash
pvecm status
```

Then:

```bash
pvecm nodes
```

Questions:

- Are all four nodes listed?
- Is the cluster quorate?
- How many votes are expected?
- How many votes are present?

Normal lab state is:

```text
Nodes:          4
Expected votes: 4
Total votes:    4
Quorum:         3
Quorate:        Yes
```

## Check Cluster-Wide Node State

```bash
pvesh get /nodes
```

Expected healthy result includes:

```text
pve01   online
pve02   online
pve03   online
pve04   online
```

---

# 10. Proxmox Quorum Incident

## Symptom

One or more nodes are unavailable or cluster operations are restricted.

A four-node cluster requires a majority.

With this lab:

```text
4 voting nodes
quorum = 3
```

Losing one node can leave three votes and preserve quorum.

Losing two voting nodes leaves two votes and the cluster no longer has a
majority.

## Investigation

Check:

```bash
pvecm status
pvecm nodes
```

Then validate the network path among the surviving nodes.

Do not confuse:

```text
cluster quorum
```

with:

```text
automatic VM high availability
```

This lab does not currently claim Proxmox HA/Ceph automatic workload failover.

---

# 11. VM or LXC Unavailable Runbook

## Identify Guest State

For VMs:

```bash
qm list
```

For containers:

```bash
pct list
```

## Inspect Configuration

Examples:

```bash
qm config <VMID>
pct config <CTID>
```

## Start the Guest

VM:

```bash
qm start <VMID>
```

LXC:

```bash
pct start <CTID>
```

## Test From Inside

For a container:

```bash
pct exec <CTID> -- ip -br addr
```

For a VM with QEMU Guest Agent:

```bash
qm guest exec <VMID> -- ip addr
```

The troubleshooting sequence is:

```text
Guest registered?
      |
      v
Guest running?
      |
      v
Virtual NIC attached?
      |
      v
Guest has IP?
      |
      v
Guest has route?
      |
      v
DNS works?
      |
      v
Application works?
```

---

# 12. Proxmox Storage / Backup Failure

## Symptom

Backup fails, storage appears inactive, or a backup target cannot be reached.

## Check Proxmox Storage

```bash
pvesm status
```

Current expected storage includes:

```text
local
local-lvm
t-20-backup
```

## Inspect Backup Target

```bash
pvesm list t-20-backup
```

This is more useful than simply seeing a mount point because it confirms
Proxmox can enumerate content through its storage configuration.

## Inspect Mounts

```bash
ls -lah /mnt/pve/
```

## Questions

- Does Proxmox report the storage as active?
- Can the host reach the storage server?
- Is the CIFS mount present?
- Are credentials still valid?
- Is the remote filesystem full?
- Can Proxmox list existing backups?

Do not start deleting backups merely because a job failed. Establish the cause
first.

---

# 13. Disk Space / Filesystem Incident

## Symptom

Application errors, read-only indices, failed installations, or unexplained
service failures.

This is based on a real failure encountered during the Wazuh deployment.

## Check Filesystem Utilization

```bash
df -h
df -h /
```

## Check Block Devices

```bash
lsblk
```

## If LVM Is Used

```bash
sudo pvs
sudo vgs
sudo lvs
```

A virtual disk can be large in the hypervisor while the guest root logical
volume is much smaller.

That distinction caused the first Wazuh deployment to cross OpenSearch's
flood-stage disk threshold.

## Corrective Example

When free extents are available in the volume group:

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

Then verify:

```bash
df -h /
```

The lesson is:

> Verify capacity at the application-visible filesystem layer, not only at the
> hypervisor virtual-disk layer.

---

# 14. Linux Service Failure Runbook

## Check Whether the Service Is Active

```bash
systemctl is-active <service>
```

## Inspect Detailed Status

```bash
sudo systemctl status <service> --no-pager -l
```

## Inspect Recent Journal Entries

```bash
sudo journalctl -u <service> --since "30 minutes ago" --no-pager
```

For more detailed failure context:

```bash
sudo journalctl -xeu <service> --no-pager
```

## Example — Wazuh Indexer

During the Wazuh build:

```bash
sudo systemctl status wazuh-indexer --no-pager -l
sudo journalctl -xeu wazuh-indexer.service --no-pager
```

revealed a filesystem-permission problem involving:

```text
/etc/wazuh-indexer/backup
```

The service log—not repeated restarts—identified the actionable error.

---

# 15. Wazuh Service Health Runbook

The main Wazuh services are:

```text
wazuh-indexer
wazuh-manager
filebeat
wazuh-dashboard
```

Check them:

```bash
sudo systemctl is-active wazuh-indexer
sudo systemctl is-active wazuh-manager
sudo systemctl is-active filebeat
sudo systemctl is-active wazuh-dashboard
```

If one fails:

1. inspect `systemctl status`;
2. inspect that service's journal;
3. check disk space;
4. check memory;
5. check dependent services;
6. check expected listening ports;
7. make one change;
8. restart only what is necessary;
9. validate dashboard and endpoint ingestion.

---

# 16. Wazuh Agent / Endpoint Monitoring Failure

## Symptom

Endpoint disappears from Wazuh, shows inactive, or stops sending telemetry.

On Linux:

```bash
systemctl is-active wazuh-agent
```

If inactive:

```bash
sudo systemctl status wazuh-agent --no-pager -l
sudo journalctl -u wazuh-agent --since "30 minutes ago" --no-pager
```

Check connectivity to the manager:

```bash
nc -zv 192.168.1.206 1514
nc -zv 192.168.1.206 1515
```

Then verify:

- agent service state;
- network path;
- DNS if hostname-based configuration is used;
- manager address;
- enrollment state;
- endpoint clock/time;
- manager service state.

---

# 17. Memory / Capacity Checks

A slow or failing service can be caused by resource pressure.

## Memory

```bash
free -h
```

## Disk

```bash
df -h
```

## Proxmox Storage

```bash
pvesm status
```

## Cluster Resource View

```bash
pvesh get /cluster/resources --type vm
```

Questions:

- Is memory exhausted?
- Is swap heavily used?
- Is a filesystem nearly full?
- Is local-lvm near capacity?
- Is one node carrying too many active guests?
- Did an application fail because a resource threshold was crossed?

---

# 18. Port and Service Reachability

Use `nc` to separate a network problem from an application-port problem.

Examples:

```bash
nc -vz 192.168.1.206 443
nc -vz 192.168.1.206 1514
nc -vz 192.168.1.206 1515
nc -vz 192.168.1.189 3389
```

A successful TCP connection indicates that:

- the destination is reachable;
- the path permits the connection;
- something is accepting the port.

It does **not** necessarily prove the application is functioning correctly.

For example:

```text
TCP 443 reachable
```

does not prove successful web authentication or correct application behavior.

---

# 19. Dependency Mapping

When a service fails, identify what it depends on.

Example:

```text
Wazuh dashboard
      |
      +--> wazuh-indexer
      |
      +--> wazuh-manager / data pipeline
      |
      +--> filesystem capacity
      |
      +--> network
      |
      +--> DNS
```

Example:

```text
Domain login
      |
      +--> Windows endpoint networking
      |
      +--> DNS
      |
      +--> dc01
      |
      +--> AD DS
      |
      +--> Kerberos / LDAP
      |
      +--> computer trust
```

Example:

```text
Proxmox backup
      |
      +--> Proxmox host
      |
      +--> guest state
      |
      +--> network path
      |
      +--> CIFS storage
      |
      +--> remote capacity
```

Dependency thinking prevents wasting time on unrelated components.

---

# 20. Change One Thing

Avoid troubleshooting by changing:

- DNS,
- gateway,
- firewall,
- service configuration,
- package versions,
- and host networking

all at once.

A better workflow is:

```text
Observe
  ↓
Form hypothesis
  ↓
Test hypothesis
  ↓
Make one change
  ↓
Retest
```

If the change does not solve the problem, record that result before moving on.

Negative results still narrow the problem.

---

# 21. Service Restoration Verification

A service is not considered restored simply because a daemon starts.

Verification should occur at multiple layers.

Example for DNS:

```text
pihole service active
        +
port 53 listening
        +
client can reach dns01
        +
client can resolve names
        +
normal Internet service works
```

Example for Wazuh:

```text
services active
        +
dashboard loads
        +
authentication succeeds
        +
agent active
        +
new telemetry arrives
```

Example for Proxmox:

```text
host reachable
        +
cluster membership correct
        +
quorum healthy
        +
storage active
        +
guest visible/running
```

---

# 22. Escalation / Handoff Template

In a professional NOC, escalation should include useful evidence.

```text
Incident:
Priority:
Start Time:
Affected Service:
Affected Systems:
User / Business Impact:

What Was Confirmed:

Checks Performed:

Commands / Tests:

Key Findings:

Changes Made:

Current State:

Remaining Problem:

Recommended Next Action:

Evidence / Logs:
```

The objective is for the next person to continue the investigation without
repeating the entire discovery process.

---

# 23. Incident Notes Template

```text
Date / Time:
System:
Service:
Reported Symptom:

Expected State:

Observed State:

Scope:
[ ] One host
[ ] Multiple hosts
[ ] One service
[ ] Network-wide
[ ] Unknown

Network Checks:
- Link:
- IP:
- Gateway:
- DNS:
- Port reachability:

Service Checks:
- Service state:
- Relevant logs:

Resource Checks:
- CPU:
- Memory:
- Disk:
- Storage:

Dependencies Checked:

Root Cause:

Corrective Action:

Validation:

Final Status:

Follow-Up:
```

---

# 24. Useful Command Reference

| Command | Purpose |
|---|---|
| `ip -br link` | Show interface/link state |
| `ip -br addr` | Show addresses concisely |
| `ip route` | Inspect routing/default gateway |
| `ping` | Basic IP reachability test |
| `nc -vz host port` | Test TCP port reachability |
| `resolvectl status` | Inspect Linux resolver state |
| `dig` / `nslookup` | Test DNS resolution |
| `pihole status` | Check Pi-hole state |
| `systemctl is-active` | Quick service-state check |
| `systemctl status --no-pager -l` | Detailed service status |
| `journalctl -u` | Service logs |
| `journalctl -xeu` | Detailed systemd failure logs |
| `free -h` | Memory and swap |
| `df -h` | Filesystem usage |
| `lsblk` | Block-device layout |
| `pvs` / `vgs` / `lvs` | LVM state |
| `qm list` | Proxmox VM inventory |
| `pct list` | Proxmox LXC inventory |
| `pvesm status` | Proxmox storage state |
| `pvesm list` | Proxmox storage contents |
| `pvecm status` | Cluster/quorum status |
| `pvecm nodes` | Cluster membership |
| `pvesh get /nodes` | Cluster node state |
| `Test-ComputerSecureChannel` | Windows domain trust validation |
| `Get-WinEvent` | Windows event-log investigation |

---

# 25. Skills Demonstrated

This runbook supports hands-on experience with:

## Network Operations

- Host availability troubleshooting
- Service availability validation
- IP addressing
- Routing
- DNS troubleshooting
- TCP port validation
- Dependency mapping
- Incident prioritization
- Operational escalation

## Systems Administration

- Linux service management
- systemd
- journal analysis
- filesystem capacity
- LVM
- VM and LXC administration
- storage troubleshooting
- Windows domain connectivity

## Proxmox

- Node health
- Cluster membership
- Quorum
- VM/LXC state
- Local and shared storage
- Cluster resource inspection
- Backup infrastructure

## Troubleshooting

- Problem scoping
- Layered diagnostics
- Root-cause analysis
- Controlled change
- Service restoration
- Post-change verification
- Incident documentation

---

# 26. Related Repository Documentation

| Area | Document |
|---|---|
| Backup / recovery | [`01-proxmox-backup.md`](01-proxmox-backup.md) |
| Asset / IP reference | [`02-asset-inventory.md`](02-asset-inventory.md) |
| DNS migration | [`02-pihole-migration.md`](02-pihole-migration.md) |
| Proxmox cluster | [`03-proxmox-cluster-build.md`](03-proxmox-cluster-build.md) |
| Wazuh deployment / troubleshooting | [`06-wazuh-siem.md`](06-wazuh-siem.md) |
| Windows telemetry | [`08-windows-telemetry.md`](08-windows-telemetry.md) |
| Active Directory | [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| SOC/NOC skills matrix | [`11-soc-noc-skills-matrix.md`](11-soc-noc-skills-matrix.md) |
| SOC triage playbook | [`12-soc-alert-triage-playbook.md`](12-soc-alert-triage-playbook.md) |

---

# 27. Summary

The operational troubleshooting model used in this lab is:

```text
Confirm
  ↓
Scope
  ↓
Inspect
  ↓
Isolate
  ↓
Diagnose
  ↓
Change
  ↓
Retest
  ↓
Restore
  ↓
Verify
  ↓
Document
```

This approach is intended to resemble Tier 1 / junior NOC and infrastructure
operations work: identify the affected layer quickly, preserve useful evidence,
restore service safely, and leave clear notes for follow-up or escalation.
