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

The VM was created from the Proxmox command line.

Example configuration used during deployment:

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
