# Zabbix Monitoring — Planned

I plan to add Zabbix to the lab to practice another approach to infrastructure monitoring and alerting.

The lab already includes LibreNMS, Uptime Kuma, and Grafana with Prometheus. Adding Zabbix will give me a way to compare how these tools collect information, identify problems, and report recovery.

This page is a placeholder for that work. It does not document a completed Zabbix deployment.

## Initial Scope

I intend to start with a small number of systems and verify the full monitoring process before expanding coverage.

The initial work will include:

- Deploying the Zabbix server and recording its configuration.
- Adding a Linux host and checking that measurements arrive.
- Monitoring service availability and resource usage.
- Configuring an alert for a controlled service interruption.
- Confirming that the alert clears when the service returns.
- Recording the troubleshooting steps and results.

I will choose the host, address, and deployment method when I begin the build.

## What I Will Document

For each monitored system, I will record what is collected, how it reaches Zabbix, and which conditions generate alerts.

I also want to distinguish between a host being reachable and its applications working. A successful ping alone will not be the completion test.

Once the deployment is working, I will replace this plan with the actual configuration, commands, validation results, and any problems encountered.
