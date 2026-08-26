# Ansible Management

`mgmt01` is used as the independent Ansible control node for the homelab.

## Current Managed Hosts

The initial inventory contains the four Proxmox VE nodes:

- `pve01` — `192.168.1.10`
- `pve02` — `192.168.1.11`
- `pve03` — `192.168.1.12`
- `pve04` — `192.168.1.13`

SSH key-based authentication is used from `mgmt01`.

No passwords or private SSH keys are stored in this repository.

## Inventory

```text
ansible/inventory.ini
