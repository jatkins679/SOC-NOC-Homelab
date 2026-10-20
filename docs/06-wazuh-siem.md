# Wazuh SIEM Deployment and Troubleshooting

## Overview

I installed Wazuh as the central security-monitoring platform for my SOC/NOC homelab.

I wanted somewhere to collect logs from Linux and Windows systems, monitor changes to files, investigate security events, and experiment with detection rules.

Wazuh appealed to me because it combines several capabilities I wanted to work with, including endpoint monitoring, log analysis, File Integrity Monitoring, and MITRE ATT&CK mappings.

The installation turned out to be more involved than I expected.

My first attempt failed because Ubuntu had not allocated the entire virtual disk to its root logical volume. This eventually caused OpenSearch to place an index into read-only mode and prevented the Wazuh installer from completing its security configuration.

I spent some time diagnosing and attempting to repair the installation before deciding to rebuild the VM.

This document covers both the successful deployment and the problems encountered along the way.

---

## 1. Lab Architecture

Wazuh runs as an all-in-one deployment, with its main components installed on a dedicated Ubuntu virtual machine.

| System | IP address | Purpose |
|---|---|---|
| `wazuh01` | `10.10.40.20` | Wazuh SIEM |
| `target01` | `10.10.60.10` | Linux monitored endpoint |
| `win11-01` | `10.10.30.160` | Windows monitored endpoint |
| `kali01` | `10.10.50.113` | Security testing |
| `dns01` | `192.168.1.20` | Pi-hole DNS |

The Wazuh VM has the following documented configuration:

| Setting | Value |
|---|---|
| VMID | `500` |
| Hostname | `wazuh01` |
| Operating system | Ubuntu Server 24.04.4 LTS |
| Virtual CPUs | 4 |
| Memory | 8 GB |
| Virtual disk | 100 GB |
| Network | VirtIO |
| Guest agent | QEMU Guest Agent |

The original deployment used the flat `192.168.1.0/24` network.

The service was subsequently moved to VLAN 40, using `10.10.40.20`, although the old flat-network interface was retained temporarily during the network migration.

---

## 2. Creating the Virtual Machine

I created VM 500 using the Proxmox command line.

```bash
qm create 500 \
  --name wazuh01 \
  --memory 8192 \
  --cores 4 \
  --sockets 1 \
  --cpu x86-64-v2-AES \
  --scsihw virtio-scsi-single \
  --agent enabled=1 \
  --ostype l26
```

I then configured the virtual disk, Ubuntu installation ISO, and network interface.

```bash
qm set 500 --scsi0 local-lvm:100,discard=on,iothread=1

qm set 500 \
  --ide2 local:iso/ubuntu-24.04.4-live-server-amd64.iso,media=cdrom

qm set 500 --net0 virtio,bridge=vmbr0
qm set 500 --boot 'order=ide2;scsi0'
```

The VM was started with:

```bash
qm start 500
```

These commands reflect the original flat-network configuration. The later VLAN migration introduced a separate interface connected through `vmbr1`.

---

## 3. Preparing Ubuntu

After installing Ubuntu Server, I checked the hostname, network configuration, and routing table.

```bash
hostname
ip -br addr
ip route
```

### DNS Configuration

I wanted the Wazuh server to use my physical Pi-hole server, `dns01` (`192.168.1.20`), rather than accepting DNS settings from DHCP.

The original Netplan configuration included:

```yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: true
      dhcp4-overrides:
        use-dns: false
      nameservers:
        addresses:
          - 192.168.1.20
```

I applied the configuration:

```bash
sudo netplan apply
```

Then checked DNS and connectivity:

```bash
resolvectl status
ping -c 3 192.168.1.20
ping -c 3 google.com
```

The configuration shown here is from the original deployment and isn't intended to represent the final VLAN 40 interface configuration.

### QEMU Guest Agent

Inside Ubuntu, I installed the guest agent:

```bash
sudo apt update
sudo apt install -y qemu-guest-agent
sudo systemctl start qemu-guest-agent
```

I checked its state:

```bash
systemctl is-active qemu-guest-agent
```

The guest agent provides additional visibility and management capabilities from Proxmox.

### Operating System Updates

Before installing Wazuh, I updated the operating system.

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

---

## 4. First Installation Attempt

I downloaded the Wazuh 4.14 installation assistant:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
```

Then started the all-in-one installation:

```bash
sudo bash ./wazuh-install.sh -a
```

The installation proceeded through the main components:

- Wazuh indexer
- Wazuh manager
- Filebeat
- Wazuh dashboard

However, it failed during the final security-user configuration stage.

Initially, this looked like an authentication or installer problem.

The underlying cause turned out to be storage.

---

## 5. Investigating the Failure

I started by examining the installation log.

```bash
sudo tail -n 80 /var/log/wazuh-install.log
```

The important error involved OpenSearch placing its security index into read-only mode.

OpenSearch has disk-watermark thresholds intended to prevent the filesystem from filling completely.

When the filesystem crosses the flood-stage threshold, OpenSearch can block writes to affected indices.

That prevented the Wazuh installation assistant from updating the internal security configuration.

### Checking Disk Utilization

I examined the filesystem:

```bash
df -h /
```

Then checked how Ubuntu had allocated the virtual disk.

```bash
lsblk
sudo pvs
sudo vgs
sudo lvs
```

The VM had a large virtual disk assigned in Proxmox, but Ubuntu's root logical volume was using only part of it.

The remaining capacity was available within LVM but had not been allocated to the root filesystem.

This was the problem I had overlooked.

### Expanding the Root Logical Volume

I expanded the logical volume to use the remaining free extents:

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

The `-l +100%FREE` option uses all remaining free extents in the volume group.

The `-r` option resizes the filesystem along with the logical volume.

I then checked the result:

```bash
df -h /
```

Filesystem utilization dropped substantially.

This corrected the original storage problem, but the interrupted Wazuh installation still had other issues.

---

## 6. OpenSearch Indexer Recovery

During the recovery attempts, the Wazuh indexer subsequently failed to start.

I checked the service state:

```bash
sudo systemctl status wazuh-indexer --no-pager -l
```

Then examined the systemd journal:

```bash
sudo journalctl -xeu wazuh-indexer.service --no-pager
```

The relevant error was:

```text
java.nio.file.AccessDeniedException:
/etc/wazuh-indexer/backup
```

The indexer could not access an installer-created backup directory.

I investigated the filesystem permissions and corrected the directory access problem.

After restarting the indexer, I confirmed that it was listening on TCP port 9200.

However, recovering the indexer service did not completely repair the installation.

---

## 7. The Credential Problem

The original installation had failed while updating internal Wazuh and OpenSearch credentials.

That left the components with inconsistent authentication settings.

The indexer could be started, but the dashboard continued to encounter authentication problems.

By this point, I had corrected the storage problem and investigated the permissions failure, but the new installation still wasn't operating normally.

I considered continuing the recovery work.

Since this was a brand-new SIEM with no important data or custom rules to preserve, I decided that rebuilding the virtual machine would be a better use of time.

The investigation had already identified the original configuration mistake.

I could avoid it during the rebuild.

---

## 8. Rebuilding Wazuh

I removed and recreated VM 500 with a 100 GB virtual disk.

This time, I checked the filesystem and LVM configuration before attempting the Wazuh installation.

The checks included:

```bash
df -h /
lsblk
sudo pvs
sudo vgs
sudo lvs
```

I also verified the hostname, networking, DNS, available memory, operating system updates, and QEMU Guest Agent.

Once those checks passed, I ran the installation assistant again.

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

The installation completed successfully.

### Service Validation

I checked all four main services:

```bash
sudo systemctl is-active wazuh-indexer
sudo systemctl is-active wazuh-manager
sudo systemctl is-active filebeat
sudo systemctl is-active wazuh-dashboard
```

All four returned:

```text
active
```

I then opened the Wazuh dashboard and verified that I could authenticate successfully.

The documented service address is:

`https://10.10.40.20`

The rebuilt Wazuh server was operational.

---

## 9. Recovering Dashboard Access

I later encountered a separate problem after restarting the Wazuh server.

The dashboard was available, but the saved administrator password no longer worked.

I located the original installer archive:

```bash
sudo find / -name wazuh-install-files.tar 2>/dev/null
```

The archived credential did not match the current authentication state.

Rather than reinstalling Wazuh, I used its password-management utility to rotate the administrator password:

```bash
sudo bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -u admin
```

After refreshing the dependent services, I verified dashboard access.

This was a different problem from the original storage-related installation failure, and it didn't require rebuilding the VM.

I don't include passwords or installer-generated credentials in this repository.

---

## 10. Enrolling the First Linux Agent

Once Wazuh was working, I installed an agent on `target01` (`10.10.60.10`).

Before installation, I tested connectivity to the Wazuh manager:

```bash
nc -zv 10.10.40.20 1514
nc -zv 10.10.40.20 1515
```

Both TCP connections succeeded.

### Installing the Agent

On `target01`, I installed the necessary package-management tools:

```bash
sudo apt-get install -y gnupg apt-transport-https
```

Then imported the Wazuh repository key:

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | \
sudo gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg \
  --import

sudo chmod 644 /usr/share/keyrings/wazuh.gpg
```

I added the Wazuh package repository:

```bash
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | \
sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt-get update
```

The agent was installed using the Wazuh manager address and the intended endpoint name:

```bash
sudo env \
WAZUH_MANAGER="10.10.40.20" \
WAZUH_REGISTRATION_SERVER="10.10.40.20" \
WAZUH_AGENT_NAME="target01" \
apt-get install -y wazuh-agent
```

I enabled and started the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
```

Then checked its status:

```bash
sudo systemctl is-active wazuh-agent
```

The Wazuh dashboard showed `target01` as an active agent.

That established the first working endpoint-to-SIEM telemetry path.

---

## 11. Adding a Windows Endpoint

I subsequently enrolled a Windows 11 Pro workstation.

```text
Hostname: win11-01
IP:       10.10.30.160
Agent ID: 002
```

The Windows installation didn't work on the first attempt either.

### Troubleshooting the MSI Installation

I checked whether the expected Windows service existed:

```powershell
Get-Service *wazuh*
```

Then checked the installation directory:

```powershell
Test-Path "C:\Program Files (x86)\ossec-agent"
```

The path check returned `False`, and the service wasn't present.

That confirmed the MSI hadn't installed the agent successfully.

After reinstalling the MSI, the service appeared.

I then ran into a service-control permissions issue because PowerShell wasn't elevated.

Opening PowerShell as Administrator resolved that problem.

### Windows Agent Validation

I checked the installed service:

```powershell
Get-Service WazuhSvc
```

Then tested connectivity to the Wazuh manager:

```powershell
Test-NetConnection 10.10.40.20 -Port 1514
```

The result included:

```text
TcpTestSucceeded : True
```

The Wazuh dashboard subsequently showed `win11-01` as active.

I later added Sysmon and PowerShell Script Block Logging to generate additional Windows security telemetry.

That work is documented separately in [Windows Endpoint Telemetry](08-windows-telemetry.md).

---

## 12. Detection Testing

With Linux and Windows agents connected, I began generating controlled activity and verifying the results in Wazuh.

The completed exercises include:

| Exercise | Detection |
|---|---|
| Repeated failed SSH logins | Brute-force alert |
| File creation and modification | File Integrity Monitoring |
| SQL-injection-style HTTP request | Apache log detection |
| Privileged command execution | Linux Auditd |
| Local Linux account creation | Account-management alert |
| Local Linux account deletion | Account-management alert |
| Windows PowerShell execution | Process and script telemetry |
| Windows registry changes | Sysmon and Wazuh events |
| Active Directory account creation | Windows security event monitoring |

The purpose wasn't simply to generate alerts.

I wanted to understand where the original events came from, how Wazuh interpreted them, and how to verify the activity using the source logs.

The specific rule IDs, alert levels, and test procedures are recorded in the [Wazuh Attack Detection Labs](09-attack-detection-labs.md).

---

## 13. What the Failed Installation Revealed

The original deployment failure involved several related problems.

```text
Root filesystem smaller than virtual disk
                 |
                 v
High filesystem utilization
                 |
                 v
OpenSearch flood-stage disk watermark
                 |
                 v
Security index placed into read-only mode
                 |
                 v
Wazuh internal-user configuration fails
                 |
                 v
Installation remains incomplete
                 |
                 v
Authentication problems between components
```

The original error appeared during credential configuration, but the underlying cause was an undersized filesystem.

I wouldn't have found that by concentrating only on passwords or repeatedly running the installer.

Examining the installation log, checking filesystem utilization, and inspecting LVM eventually exposed the problem.

The later permissions failure and dashboard credential issue were separate symptoms encountered while recovering or operating the installation.

The experience also changed how I prepare virtual machines for applications with substantial storage requirements.

I now pay closer attention to the difference between a virtual disk's size in Proxmox and the actual filesystem capacity available inside the guest.

---

## 14. Current Result

Wazuh is operational as the primary SIEM in the lab.

The deployment supports:

- Linux and Windows endpoint monitoring
- Centralized security-event analysis
- File Integrity Monitoring
- Linux Auditd integration
- Windows Sysmon telemetry
- PowerShell Script Block Logging
- Authentication and account-management monitoring
- MITRE ATT&CK-aligned detections
- Controlled security-testing exercises

The installation failure, repair attempts, rebuild, and subsequent endpoint enrollment are all part of the project's documented history.

Future work includes additional Linux agents, custom detection rules, alert tuning, Active Response testing, and completing the removal of legacy network dependencies.

The biggest value of this deployment has been having a working platform where I can generate activity on an endpoint, follow the resulting logs into Wazuh, and investigate what the system actually reports.

## Related Documentation

- [Windows Endpoint Telemetry](08-windows-telemetry.md)
- [Wazuh Attack Detection Labs](09-attack-detection-labs.md)
- [Active Directory Lab](10-active-directory-lab.md)
- [SOC Alert Triage Playbook](12-soc-alert-triage-playbook.md)
- [VLAN Migration Dependency Checklist](24-vlan-migration-dependency-checklist.md)
