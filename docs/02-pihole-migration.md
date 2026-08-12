# Pi-hole Migration to a Dedicated Raspberry Pi

## Overview

As part of the SOC/NOC homelab redesign, I decided to move Pi-hole from an LXC container running on a Proxmox host to a dedicated Raspberry Pi.

The existing Pi-hole service worked, but hosting DNS inside the virtualization environment created an unnecessary dependency: maintenance or failure of the Proxmox host could also interrupt DNS service for the home network. I also had an old Raspberry Pi B+ sitting around unused, so this seemed like a good way to repurpose old hardware.

Moving Pi-hole to separate physical hardware allows DNS to remain available while Proxmox hosts are being rebuilt and reclustered.

This phase involved:

- Selecting a dedicated Raspberry Pi for DNS
- Preparing the Raspberry Pi operating system
- Renaming the host
- Updating the operating system
- Installing Pi-hole
- Verifying the Pi-hole DNS service
- Completing the migration away from the existing Proxmox LXC
- Cutting DNS service over to the Raspberry Pi at 192.168.1.20 and confirming that DNS was independent of the virtualization environment

## Why Move Pi-hole Off Proxmox?

The original Pi-hole instance was running as an LXC container on one of the existing Proxmox hosts.

That configuration worked, but it meant several systems were dependent on each other:

    Home clients
         |
         v
    Pi-hole LXC
         |
         v
    Proxmox host
         |
         v
    Physical mini PC

If the Proxmox host was shut down for maintenance, reinstalled, or had a networking problem, the Pi-hole service would also become unavailable.

The planned homelab redesign includes significant changes to the Proxmox environment, including:

- Reinstalling hosts
- Creating a multi-node cluster
- Changing hostnames and addressing
- Reworking network interfaces
- Adding VLANs
- Deploying additional virtual infrastructure

I therefore moved DNS onto a Raspberry Pi that could remain operational independently of the Proxmox environment.

The revised relationship is:

    Home clients
         |
         v
    Dedicated Raspberry Pi
         |
         v
    Pi-hole DNS

The Proxmox cluster can now be taken offline without automatically taking Pi-hole down with it.

## 1. Select the Physical DNS Host

A Raspberry Pi Model B+ was selected to become the dedicated Pi-hole server.

Its new infrastructure role is:

    Hostname: dns01
    Role:     DNS / Pi-hole
    Platform: Raspberry Pi Model B+
    IP:       192.168.1.20
    OS:       Raspberry Pi OS

Using a dedicated hostname makes the function of the system immediately identifiable when viewing logs, SSH sessions, monitoring dashboards, or network documentation.

## 2. Rename the Raspberry Pi

The Raspberry Pi previously used a generic hostname.

I renamed the system:

    sudo hostnamectl set-hostname dns01

### Purpose

`hostnamectl` changes the system hostname using systemd’s hostname management interface.

The new name:

    dns01

identifies the system by its infrastructure role rather than by its physical appearance or previous use.

### Skill Demonstrated

- Linux system administration
- Host naming conventions
- Infrastructure organization

## 3. Review and Update `/etc/hosts`

After changing the hostname, I checked the local hosts file:

    cat /etc/hosts

### Purpose

Review static hostname mappings stored on the Raspberry Pi.

Because the old hostname still appeared in `/etc/hosts`, I replaced it with the new name:

    sudo sed -i 's/blue-rp/dns01/g' /etc/hosts

### Command Breakdown

### `sed`

`sed` is a stream editor commonly used to modify text from the command line.

### `-i`

Edit the file in place.

### `s/blue-rp/dns01/g`

Replace occurrences of:

    blue-rp

with:

    dns01

### `/etc/hosts`

The file being modified.

### Why This Was Necessary

Changing the system hostname with `hostnamectl` does not necessarily update every static reference to the old hostname.

### Skill Demonstrated

- Linux configuration files
- `sed`
- Text replacement
- Hostname resolution
- CLI administration

## 4. Reboot the Raspberry Pi

After the hostname configuration was changed, I rebooted the system:

    sudo reboot

### Purpose

Restart the Raspberry Pi so the hostname and related system configuration were applied cleanly.

After reconnecting, the system could be managed using its new identity:

    dns01

### Skill Demonstrated

- Linux host administration
- Configuration change validation

## 5. Update Raspberry Pi OS

Before installing Pi-hole, I refreshed the operating system’s package information:

    sudo apt update

### Purpose

Download current package metadata from the configured software repositories.

I then upgraded the installed packages:

    sudo apt full-upgrade -y

### Purpose

Bring the Raspberry Pi OS installation up to date before deploying the DNS service.

### `full-upgrade`

Allows APT to resolve package dependency changes that may require installing or removing packages.

### `-y`

Automatically answers yes to package-management confirmation prompts.

### Skill Demonstrated

- Debian package management
- Linux patching
- System preparation
- Dependency management

## 6. Install Pi-hole

Pi-hole was installed using its installation script:

    curl -sSL https://install.pi-hole.net | bash

### Command Breakdown

### `curl`

Retrieves data from a remote URL.

### `-s`

Silent mode.

### `-S`

Display errors even when silent mode is enabled.

### `-L`

Follow HTTP redirects.

### `| bash`

Pass the downloaded installation script to the Bash shell for execution.

The interactive installation process was then used to configure the Raspberry Pi as the dedicated Pi-hole DNS server.

### Skill Demonstrated

- Linux command-line administration
- Shell pipelines
- Application deployment
- DNS infrastructure
- Raspberry Pi administration

## 7. Verify Pi-hole

After installation, I checked the Pi-hole service:

    pihole status

The status output confirmed that Pi-hole’s FTL service was listening for DNS traffic on port 53 using:

- UDP over IPv4
- TCP over IPv4
- UDP over IPv6
- TCP over IPv6

The output also showed:

    Pi-hole blocking: enabled

### Why This Verification Matters

Need to confirm that the service is operational.

Running:

    pihole status

confirmed that the DNS components were running and that Pi-hole filtering was enabled.

### Skill Demonstrated

- DNS service verification
- Linux service validation
- Post-installation testing

## 8. Separate DNS From the Hypervisor

This change altered the infrastructure dependency model of the homelab.

### Previous Design

    Client
       |
       v
    Pi-hole
       |
       v
    LXC
       |
       v
    Proxmox
       |
       v
    Physical host

### New Design

    Client
       |
       v
    Pi-hole @ 192.168.1.20
       |
       v
    Dedicated Raspberry Pi Model B+

This means Proxmox maintenance no longer inherently requires shutting down the Pi-hole host.

## 9. DNS Migration and Cutover

After the physical Pi-hole installation was configured and verified, I completed the network cutover and moved the DNS service to the Raspberry Pi at 192.168.1.20.

The completed migration process was:

    Prepare Raspberry Pi
            ↓
    Install Pi-hole
            ↓
    Verify DNS service
            ↓
    Configure network settings
            ↓
    Assign 192.168.1.20 to the Raspberry Pi
            ↓
    Cut clients/network over to the new DNS host
            ↓
    Test DNS resolution and filtering
            ↓
    Take the old Pi-hole LXC out of DNS service

The previous Pi-hole LXC was taken out of DNS service only after the physical replacement was running successfully at the established Pi-hole address.

### Final Configuration

The migration was completed successfully. The dedicated Raspberry Pi Model B+ is now running Pi-hole at 192.168.1.20, the same DNS service address previously used by the Pi-hole LXC.

    Before

    192.168.1.20
         |
    Pi-hole LXC
         |
    Proxmox host
         |
    Mini PC


    After

    192.168.1.20
         |
    Pi-hole
         |
    Raspberry Pi Model B+

Retaining the 192.168.1.20 service address meant that clients and network configuration already pointing to that DNS address did not need to be changed simply because the underlying Pi-hole platform moved from an LXC container to physical hardware.

The previous Pi-hole LXC is no longer providing DNS service. DNS is now hosted independently of the Proxmox environment, allowing the virtualization hosts to be restarted, rebuilt, or reconfigured without automatically taking Pi-hole offline.

## 10. Validation

The cutover was considered complete after the dedicated Pi-hole host was reachable at 192.168.1.20, Pi-hole was running, and normal DNS service continued through the new physical host.

### Server Validation

- Pi-hole service running
- DNS port 53 listening
- Filtering enabled
- Raspberry Pi reachable on the network
- Hostname correctly configured

### DNS Validation

Functional DNS validation focused on confirming that client systems could:

- Resolve public DNS names
- Reach normal Internet services
- Resolve consistently through the Pi-hole server
- Generate queries visible in Pi-hole

### Infrastructure Validation

The architectural goal is for DNS to remain available when:

- A Proxmox guest is stopped
- A Proxmox host is restarted
- The Proxmox cluster is undergoing maintenance

The completed cutover removes the direct DNS dependency on Proxmox. A deliberate Proxmox host-restart maintenance test remains on the project checklist.

## 11. Troubleshooting Approach

If DNS stops working during the migration, I want to determine whether the problem exists at the:

    Client
      ↓
    DNS configuration
      ↓
    Network
      ↓
    Raspberry Pi
      ↓
    Pi-hole
      ↓
    Upstream DNS

rather than immediately changing multiple configurations at once.

Useful diagnostic areas include:

- Raspberry Pi interface status
- IP addressing
- Routing
- DNS server configuration
- Pi-hole service status
- Client DNS configuration
- Network connectivity
- Upstream DNS reachability

## Commands Reference

| Command | Purpose |
|---|---|
| `hostnamectl set-hostname` | Change Linux system hostname |
| `cat /etc/hosts` | Review local hostname mappings |
| `sed -i` | Modify hostname references in a configuration file |
| `reboot` | Restart the system after configuration changes |
| `apt update` | Refresh package repository information |
| `apt full-upgrade` | Upgrade installed packages |
| `curl` | Retrieve the Pi-hole installer |
| `bash` | Execute the Pi-hole installation script |
| `pihole status` | Verify Pi-hole DNS and filtering status |

## Commands Executed

    sudo hostnamectl set-hostname dns01

    cat /etc/hosts

    sudo sed -i 's/blue-rp/dns01/g' /etc/hosts

    sudo reboot

    sudo apt update

    sudo apt full-upgrade -y

    curl -sSL https://install.pi-hole.net | bash

    pihole status

## Skills Demonstrated

### Linux Administration

- Raspberry Pi OS administration
- Hostname configuration
- `/etc/hosts` management
- Package management
- Operating system updates
- Reboot and configuration validation

### Command-Line Administration

- `hostnamectl`
- `cat`
- `sed`
- `apt`
- `curl`
- Bash pipelines
- Pi-hole CLI

### Networking

- DNS infrastructure
- Host addressing
- Service availability
- Network dependency analysis
- DNS troubleshooting methodology

### Infrastructure Design

- Separating critical services from virtualization infrastructure
- Reducing service dependencies
- Assigning infrastructure roles to physical systems
- Planning service migration and cutover
- Preserving a service IP while changing the underlying hosting platform

### Operational Practice

- Preparing a replacement service before retiring the original
- Post-installation verification
- Incremental migration
- Service validation
- Change documentation

## Status

- [x] Select dedicated Raspberry Pi
- [x] Assign `dns01` infrastructure role
- [x] Rename Linux host
- [x] Update `/etc/hosts`
- [x] Update Raspberry Pi OS
- [x] Install Pi-hole
- [x] Verify Pi-hole service
- [x] Confirm DNS service listening on port 53
- [x] Confirm Pi-hole filtering enabled
- [x] Complete network DNS cutover
- [x] Move Pi-hole service to `192.168.1.20`
- [x] Remove DNS dependency on Proxmox
- [ ] Validate DNS from multiple client systems
- [ ] Verify operation during Proxmox maintenance
- [ ] Retire previous Pi-hole LXC
- [x] Document final migration result

## Result

The Pi-hole migration is complete. DNS service is now provided by a dedicated Raspberry Pi Model B+ at 192.168.1.20, replacing the previous Pi-hole LXC while retaining the same service IP address.

Retaining 192.168.1.20 allowed existing clients and network configuration that already referenced that address to continue using Pi-hole without a broad DNS-address change.

The previous Pi-hole LXC is no longer providing DNS service. DNS is now independent of the Proxmox environment, so the virtualization rebuild and cluster work can continue without unnecessarily disrupting DNS for the rest of the home network.

## Previous Phase

[`01-proxmox-backup.md`](01-proxmox-backup.md)

## Next Phase

The next major infrastructure phase is the standardized Proxmox cluster build:

[`03-proxmox-cluster-build.md`](03-proxmox-cluster-build.md)
