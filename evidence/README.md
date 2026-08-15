# Homelab Evidence

This directory contains screenshots and other supporting evidence from completed SOC/NOC homelab work.

The documentation in `/docs` explains the configuration, commands, troubleshooting, and analysis. The files here provide visual confirmation of selected results.

## Wazuh Detection Evidence

| File | Detection | Rule |
|---|---|---:|
| `01-ssh-bruteforce-rule-5712.png` | SSH brute force | 5712 |
| `02-fim-rule-550.png` | File integrity modification | 550 |
| `03-sql-injection-rule-31103.png` | SQL injection attempt | 31103 |
| `04-audit-sudo-rule-80792.png` | Privileged `sudo` execution | 80792 |
| `05-audit-id-rule-80792.png` | Privileged `id` execution | 80792 |
| `06-user-created-rule-5902.png` | Local account creation | 5902 |
| `07-user-deleted-rule-5903.png` | Local account deletion | 5903 |
| `08-windows-sysmon-powershell-rule-92027.png` | Windows Sysmon PowerShell process | 92027 |
| `09-powershell-registry-rule-91843.png` | PowerShell registry modification | 91843 |

These images were exported from the Wazuh Threat Hunting event details generated during controlled lab exercises.

Evidence is intentionally limited to information appropriate for a public repository. Passwords, authentication tokens, generated credentials, and other secrets are excluded.

## Proxmox Cluster Evidence

| File | Evidence |
|---|---|
| `proxmox/01-four-node-quorum.txt` | Sanitized four-node membership and quorum result |
| `proxmox/02-workload-placement.txt` | Sanitized workload placement after rebalancing |

The text evidence omits hardware serial numbers, MAC addresses, SSH
fingerprints, credentials, and authentication material.
