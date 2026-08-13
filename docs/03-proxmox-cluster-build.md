# Proxmox Three-Node Cluster Build and Validation

## Objective

The next stage of the homelab rebuild was to consolidate three Proxmox VE hosts into a single manageable cluster.

The goals were to:

- Manage all three Proxmox hosts from one interface.
- Provide quorum for normal cluster operation.
- Make VM and container placement easier to manage.
- Establish a platform for later SOC/NOC services such as Wazuh, network monitoring, vulnerability scanning, and isolated security-testing systems.
- Validate the cluster before making additional infrastructure changes.

This environment is a small home lab built from inexpensive mini PCs and repurposed hardware, so the design emphasizes practical administration and learning rather than enterprise-scale high availability.

## Cluster Architecture

The resulting Proxmox cluster is named:

```text
homelab
