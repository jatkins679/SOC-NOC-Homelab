# Zabbix Monitoring — Planned

## Overview

I'm considering adding Zabbix to the homelab as another way to experiment with infrastructure monitoring and alerting.

I already have several monitoring systems running:

- **LibreNMS** for network monitoring
- **Uptime Kuma** for service availability
- **Prometheus** for metrics collection
- **Grafana** for dashboards and visualization

Those services cover many of my current needs, so I'm not adding Zabbix simply because the lab needs another monitoring application.

I'm interested in comparing how Zabbix handles host monitoring, service checks, triggers, alerts, and recovery events.

This document is a plan. Zabbix has not yet been deployed.

---

## 1. What I Want to Test

The initial deployment will be deliberately small.

Rather than immediately monitor the entire homelab, I want to connect a few representative systems and work through the complete monitoring process.

That includes collecting data, creating a trigger, generating an alert, investigating the event, and confirming that the alert clears when the problem is resolved.

The initial tests will cover:

1. Linux host availability
2. CPU, memory, and disk utilization
3. Service availability
4. Trigger thresholds
5. Alert generation and recovery
6. Monitoring-agent communication

The objective is to understand how the monitoring system behaves, not just demonstrate that its dashboard loads.

---

## 2. Proposed Lab Systems

Some candidates for initial monitoring include:

| Host | Documented address | Potential monitoring |
|---|---|---|
| `docker` | `192.168.1.174` | Linux metrics and services |
| `dns01` | `192.168.1.20` | Pi-hole DNS availability |
| `pve01` | `192.168.1.10` | Host resources |
| `util01` | `192.168.1.243` | Linux resources and utility services |

These are possible targets, not confirmed Zabbix agent installations.

I'll choose the actual systems and deployment method when I start the project.

I also need to account for the VLAN configuration and firewall rules when deciding where to place the Zabbix server.

---

## 3. Monitoring and Alert Testing

One experiment I want to perform is a controlled service interruption.

The process will be:

1. Establish normal service operation.
2. Confirm that Zabbix is collecting the expected data.
3. Configure a trigger for a specific failure.
4. Stop the selected test service.
5. Confirm that Zabbix detects the condition.
6. Review the event and any generated notification.
7. Restart the service.
8. Confirm the recovery event.
9. Document the results.

I want to distinguish between a host being reachable and an application actually working.

A successful ping doesn't tell me whether DNS, HTTP, or another application is functioning correctly.

---

## 4. Comparing Monitoring Platforms

After the initial tests, I want to compare Zabbix with the tools already running in the lab.

| Platform | Primary area of comparison |
|---|---|
| Zabbix | Host monitoring, triggers, event management |
| LibreNMS | Network devices and interface monitoring |
| Prometheus | Time-series metrics and collection |
| Grafana | Visualization and dashboards |
| Uptime Kuma | Availability checks and notifications |

These tools overlap in some areas, and several can perform functions beyond those listed here.

The experiment should help me understand where each tool is useful and whether Zabbix adds anything meaningful to the existing monitoring stack.

---

## 5. What I'll Document

When I deploy Zabbix, I'll record the actual installation and configuration rather than filling this page with untested commands.

The documentation will include:

- Deployment location and system specifications
- Network configuration
- Installation commands
- Monitored hosts and services
- Agent configuration where applicable
- Trigger and alert definitions
- Controlled failure and recovery results
- Troubleshooting
- Any decisions about keeping or removing the service

If I encounter problems during installation or testing, I'll document those as well.

---

## Status

**Planned — not deployed.**

The next step is to select the Zabbix server's deployment location and carry out the initial monitoring experiment.

## Related Documentation

- [Asset Inventory](02-asset-inventory.md)
- [Homelab Operations Checklist](15-homelab-operations-checklist.md)
- [Managed Switching and VLANs](04-network-vlans.md)
- [NOC Troubleshooting Runbook](13-noc-troubleshooting-runbook.md)
