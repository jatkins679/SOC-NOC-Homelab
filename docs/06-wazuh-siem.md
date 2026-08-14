# Wazuh SIEM Deployment

## Objective

The next stage of the SOC/NOC homelab was to deploy a centralized security monitoring platform.

I selected Wazuh because it provides:

- Endpoint monitoring
- Security event collection
- File integrity monitoring
- Log analysis
- Linux Audit integration
- MITRE ATT&CK mappings
- Alert correlation
- A web-based investigation interface

The initial goal was to deploy Wazuh as a dedicated virtual machine in the Proxmox cluster and enroll a Linux test system as the first monitored endpoint.

## Architecture

| System | Role | IP Address |
|---|---|---|
| pve01 | Proxmox host for Wazuh VM | 192.168.1.10 |
| wazuh01 | Wazuh all-in-one server | 192.168.1.206 |
| target01 | Ubuntu monitored endpoint | 192.168.1.238 |
| kali01 | Security testing system | 192.168.1.211 |
| dns01 | Pi-hole DNS | 192.168.1.20 |

`wazuh01` runs as VM 500 on the Proxmox cluster.

## Wazuh VM Configuration

The Wazuh server was provisioned with:

- Ubuntu Server 24.04.4 LTS
- 4 virtual CPUs
- 8 GB RAM
- 100 GB virtual disk
- VirtIO networking
- Proxmox QEMU guest agent
- DNS through `dns01`

The VM uses the Proxmox bridge `vmbr0`.

## Initial VM Creation

The VM was created from the Proxmox command line. The configuration below records the deployment configuration used for VM 500.

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

The virtual disk, ISO and network interface were then attached:

```bash
qm set 500 --scsi0 local-lvm:100,discard=on,iothread=1

qm set 500 \
  --ide2 local:iso/ubuntu-24.04.4-live-server-amd64.iso,media=cdrom

qm set 500 --net0 virtio,bridge=vmbr0
qm set 500 --boot 'order=ide2;scsi0'
```

The VM was then started:

```bash
qm start 500
```

## Ubuntu Configuration

After installing Ubuntu Server, I verified the system identity and network configuration:

```bash
hostname
ip -br addr
ip route
```

The system hostname was:

```text
wazuh01
```

The rebuilt server ultimately received:

```text
192.168.1.206
```

from DHCP.

## DNS Configuration

The Wazuh server was configured to use the physical Pi-hole server rather than DNS supplied by the home gateway.

The Netplan configuration retained DHCP for addressing but disabled DHCP-provided DNS:

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

The configuration was applied with:

```bash
sudo netplan apply
```

DNS was verified with:

```bash
resolvectl status
ping -c 3 192.168.1.20
ping -c 3 google.com
```

This confirmed both connectivity to `dns01` and external name resolution.

## QEMU Guest Agent

The Proxmox guest agent was installed inside Ubuntu:

```bash
sudo apt update
sudo apt install -y qemu-guest-agent
sudo systemctl start qemu-guest-agent
```

The guest service was then verified:

```bash
systemctl is-active qemu-guest-agent
```

## Operating System Update

Before installing Wazuh, the system was fully updated:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

## Storage Validation

An important lesson from the first deployment attempt was to verify the guest filesystem before installing Wazuh.

The following commands were used:

```bash
df -h /
sudo pvs
sudo vgs
sudo lvs
lsblk
```

Ubuntu's default LVM installation did not initially allocate the entire virtual disk to the root logical volume.

The logical volume was expanded using:

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

The result was verified with:

```bash
df -h /
```

This step became part of the deployment checklist before installing Wazuh.

## First Wazuh Installation Attempt

The Wazuh installation assistant was downloaded and started with:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

The installation successfully deployed:

- Wazuh indexer
- Wazuh manager
- Filebeat
- Wazuh dashboard

However, the installer failed during the final security-user configuration stage.

## Troubleshooting the Failed Deployment

The Wazuh installation log was inspected:

```bash
sudo tail -n 80 /var/log/wazuh-install.log
```

The important failure was caused by OpenSearch placing its security index into read-only mode after the filesystem crossed its flood-stage disk watermark.

This occurred because the Ubuntu root logical volume was substantially smaller than the virtual disk assigned by Proxmox.

The disk issue was diagnosed using:

```bash
df -h /
lsblk
sudo pvs
sudo vgs
sudo lvs
```

The underlying virtual disk had free LVM capacity that had not been assigned to the root filesystem.

After expanding the logical volume, filesystem utilization dropped significantly.

## Additional Indexer Recovery

During recovery attempts, the Wazuh indexer later failed to start.

The service was investigated using:

```bash
sudo systemctl status wazuh-indexer --no-pager -l
sudo journalctl -xeu wazuh-indexer.service --no-pager
```

The failure was traced to:

```text
java.nio.file.AccessDeniedException:
/etc/wazuh-indexer/backup
```

Permissions on installer-created backup directories were corrected before restarting the indexer.

The indexer was then verified as listening on port 9200.

## Credential State After Interrupted Installation

Because the original installer failed while updating internal Wazuh/OpenSearch credentials, the installation was left with inconsistent authentication state between components.

Although individual services could be recovered, the dashboard continued to encounter authentication problems with the indexer.

At that point, continuing to repair a new installation had less value than rebuilding the server in a known-good state.

No production data, agents or custom rules existed yet, so the VM was rebuilt.

## Clean Rebuild

VM 500 was removed and recreated with a larger 100 GB virtual disk.

Before reinstalling Wazuh, the rebuilt system was validated for:

- Correct hostname
- Correct networking
- Pi-hole DNS
- Full LVM allocation
- Approximately 8 GB RAM
- Current Ubuntu packages
- QEMU guest agent operation

Only after those checks passed was the Wazuh installation assistant run again:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

This installation completed successfully.

## Wazuh Service Validation

The four main services were checked:

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

The web dashboard was then accessed at:

```text
https://192.168.1.206
```

and successful authentication confirmed the Wazuh stack was operational.

## Administrator Credential

After verifying the clean installation, the Wazuh `admin` password was changed using the Wazuh password utility.

Credentials are intentionally not stored in this repository.

## Dashboard Credential Recovery

After a later Wazuh server restart, the dashboard was available but the previously retained installation credential no longer authenticated successfully.

The original installer archive was located with:

```bash
sudo find / -name wazuh-install-files.tar 2>/dev/null
```

The archived `admin` credential was recovered for comparison, but it did not match the active credential state.

Rather than reinstalling Wazuh, the supported password-management utility was used to rotate only the `admin` account:

```bash
sudo bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -u admin
```

Dependent services were refreshed and dashboard authentication was verified successfully.

No password or generated credential is stored in this repository.

This recovery demonstrated a useful operational principle: verify whether a stored credential reflects the live authentication state before assuming a service is broken.

## First Agent Enrollment

The first endpoint enrolled into Wazuh was:

```text
target01
192.168.1.238
```

Before installing the agent, connectivity to the Wazuh server was tested:

```bash
nc -zv 192.168.1.206 1514
nc -zv 192.168.1.206 1515
```

Both connections succeeded.

The Wazuh repository was then added to `target01`.

```bash
sudo apt-get install -y gnupg apt-transport-https
```

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | \
sudo gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg \
  --import
```

```bash
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
```

```bash
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | \
sudo tee /etc/apt/sources.list.d/wazuh.list
```

```bash
sudo apt-get update
```

The endpoint was installed and configured to use `wazuh01`:

```bash
sudo env \
WAZUH_MANAGER="192.168.1.206" \
WAZUH_REGISTRATION_SERVER="192.168.1.206" \
WAZUH_AGENT_NAME="target01" \
apt-get install -y wazuh-agent
```

The service was enabled and started:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
```

Agent status was verified with:

```bash
sudo systemctl is-active wazuh-agent
```

The Wazuh dashboard then showed `target01` as an active agent.

## Windows Agent Enrollment and Troubleshooting

A Windows 11 Pro endpoint was added as a second Wazuh agent:

```text
Agent name: win11-01
Address during testing: 192.168.1.167
Agent ID: 002
```

The Windows agent deployment initially failed to create the expected service. The installation state was checked with:

```powershell
Get-Service *wazuh*
Test-Path "C:\Program Files (x86)\ossec-agent"
```

The missing service and `False` path result confirmed that the MSI had not actually installed the agent.

After reinstalling the MSI, the service was present but initially could not be controlled from a non-elevated shell. Reopening PowerShell as Administrator resolved the service-control issue.

Final service state was verified with:

```powershell
Get-Service WazuhSvc
```

and manager connectivity was confirmed with:

```powershell
Test-NetConnection 192.168.1.206 -Port 1514
```

which returned:

```text
TcpTestSucceeded : True
```

The dashboard then showed `win11-01` as an active endpoint.

Additional Windows telemetry configuration is documented in `docs/08-windows-telemetry.md`.

## Current Result

The environment now contains a functioning all-in-one Wazuh SIEM server with actively monitored Linux and Windows endpoints.

The deployment provides:

- Centralized log analysis
- Endpoint telemetry
- File integrity monitoring
- Linux Audit event collection
- Authentication monitoring
- Windows Sysmon telemetry
- PowerShell Script Block Logging
- MITRE ATT&CK mappings
- Security alert correlation
- Threat Hunting capability

The system has already been used to successfully detect several controlled security events. Those exercises are documented separately in `docs/09-attack-detection-labs.md`.

## Lessons Learned

### Verify guest storage before installing data-intensive services

A virtual disk being 60 GB or 100 GB in Proxmox does not guarantee that the guest operating system is using all of it.

`lsblk`, `vgs`, `lvs`, and `df` should be checked before deploying storage-intensive applications.

### Investigate the first failure instead of repeatedly rerunning installers

The initial Wazuh error appeared to be a credential problem, but log analysis showed that the original cause was disk exhaustion.

The chain was:

```text
undersized root filesystem
        ↓
OpenSearch flood-stage watermark
        ↓
security index becomes read-only
        ↓
internal-user update fails
        ↓
Wazuh installer exits incomplete
        ↓
component credential state becomes inconsistent
```

### Rebuilding can be the correct recovery decision

Because this was a brand-new SIEM with no production data, rebuilding the VM was lower risk and faster than continuing to repair an installation that had failed midway through security configuration.

This preserved the troubleshooting lesson while restoring the environment to a known-good state.

### Do not publish credentials

Installer-generated passwords and other security credentials are excluded from the public repository.

## Commands Used During Deployment and Troubleshooting

```bash
hostname
ip -br addr
ip route
resolvectl status

df -h /
lsblk
sudo pvs
sudo vgs
sudo lvs
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv

sudo apt update
sudo apt full-upgrade -y

sudo apt install -y qemu-guest-agent
sudo systemctl start qemu-guest-agent

curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a

sudo systemctl is-active wazuh-indexer
sudo systemctl is-active wazuh-manager
sudo systemctl is-active filebeat
sudo systemctl is-active wazuh-dashboard

sudo tail -n 80 /var/log/wazuh-install.log
sudo systemctl status wazuh-indexer --no-pager -l
sudo journalctl -xeu wazuh-indexer.service --no-pager

sudo ss -tlnp

nc -zv 192.168.1.206 1514
nc -zv 192.168.1.206 1515

sudo systemctl is-active wazuh-agent
```

## Skills Demonstrated

- Proxmox virtual machine provisioning
- Ubuntu Server administration
- Linux networking
- Netplan configuration
- DNS administration
- LVM troubleshooting and expansion
- Linux service management with systemd
- Wazuh SIEM deployment
- OpenSearch troubleshooting
- Log analysis
- Authentication troubleshooting
- Wazuh endpoint enrollment
- Windows endpoint enrollment
- Windows service troubleshooting
- Sysmon integration
- PowerShell telemetry integration
- Incident troubleshooting
- Root-cause analysis
- Recovery decision making
- Security documentation

## Deployment Status

- [x] Dedicated Wazuh VM created
- [x] Ubuntu Server installed
- [x] Pi-hole DNS configured
- [x] QEMU guest agent installed
- [x] Root filesystem/LVM capacity validated
- [x] Failed first installation investigated
- [x] OpenSearch flood-stage root cause identified
- [x] Rebuild completed with larger virtual disk
- [x] Wazuh all-in-one installation completed successfully
- [x] Wazuh indexer active
- [x] Wazuh manager active
- [x] Filebeat active
- [x] Wazuh dashboard active
- [x] Web dashboard login verified
- [x] Administrator password changed
- [x] `target01` enrolled as first monitored endpoint
- [x] `win11-01` enrolled as Windows monitored endpoint
- [x] Sysmon telemetry received from Windows
- [x] PowerShell Script Block telemetry received from Windows
- [x] Agent communication verified
- [x] Threat Hunting validated with real lab alerts

## Detection Validation

The completed Wazuh deployment has successfully detected:

- SSH brute-force activity
- File-integrity changes
- SQL-injection-style web requests
- Commands executed with root privileges
- Local account creation
- Local account deletion
- PowerShell process execution on Windows
- PowerShell registry modification on Windows

Those exercises are documented separately in:

`docs/09-attack-detection-labs.md`

## Evidence / Follow-Up

Supporting evidence available from the build includes:

- Wazuh service status checks
- Wazuh dashboard screenshots
- Agent enrollment confirmation
- Installer troubleshooting logs
- LVM/storage diagnostics
- Wazuh alert exports from completed detection labs

Credentials, generated passwords, and other secrets are intentionally excluded from the public repository.

## Next Phase

The Wazuh platform is now operational and ready for additional endpoint and detection work.

Planned future work includes:

- Additional Linux agents
- Active Directory and domain-joined Windows telemetry
- SSH `authorized_keys` persistence monitoring
- Custom Wazuh rules
- Alert tuning
- Active Response testing
- Additional MITRE ATT&CK-aligned exercises
- Integration with the future segmented/VLAN lab design
