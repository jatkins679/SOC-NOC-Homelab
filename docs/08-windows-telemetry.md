# Windows Endpoint Telemetry

## Objective

Add a Windows 11 endpoint to the SOC/NOC homelab and configure it to produce security telemetry useful for SIEM investigation.

The goal was to build a Windows endpoint that could be monitored with Wazuh and provide higher-fidelity process and PowerShell visibility through Sysmon and PowerShell Script Block Logging.

## Lab System

| System | Role | Address during testing |
|---|---|---|
| `win11-01` | Windows 11 Pro monitored endpoint | `192.168.1.167/24` (DHCP) |
| `wazuh01` | Wazuh SIEM server | `192.168.1.206` |
| `dns01` | Pi-hole DNS | `192.168.1.20` |

`win11-01` was created as a Proxmox VM and initially configured with a local administrator account. It remains a standalone workstation pending the Active Directory phase of the lab.

## Proxmox Guest Integration

The Windows VM uses the VirtIO driver ISO and Proxmox QEMU Guest Agent.

The QEMU Guest Agent was installed from the VirtIO ISO using the 64-bit Windows installer in the `guest-agent` directory. Service status was verified with:

```powershell
Get-Service QEMU-GA
```

The hostname was verified as:

```text
win11-01
```

After the guest agent was operational, Proxmox reported the guest IPv4 address correctly.

## Wazuh Agent Deployment

The Windows Wazuh agent was deployed with `wazuh01` as the manager and `win11-01` as the agent name.

Post-installation validation included:

```powershell
Get-Service WazuhSvc
```

and network connectivity to the Wazuh manager:

```powershell
Test-NetConnection 192.168.1.206 -Port 1514
```

The connectivity test returned:

```text
TcpTestSucceeded : True
```

The Wazuh dashboard subsequently showed `win11-01` as an active monitored endpoint.

## Wazuh Agent Installation Troubleshooting

The first deployment attempt did not create the Windows service.

The absence of the agent was confirmed with:

```powershell
Get-Service *wazuh*
Test-Path "C:\Program Files (x86)\ossec-agent"
```

The path check returned `False`, confirming that the agent had not been installed rather than merely failing to start.

The MSI was then installed successfully and the service became available. An additional startup failure was traced to the PowerShell session not being elevated. After reopening PowerShell as Administrator, the service started successfully.

This troubleshooting sequence reinforced the difference between:

```text
installer downloaded
        ↓
agent installed
        ↓
Windows service created
        ↓
service running
        ↓
manager reachable
        ↓
agent active in SIEM
```

## Sysmon Installation

Sysmon was added to increase Windows endpoint visibility beyond the default event logs.

A working directory was created:

```powershell
New-Item -ItemType Directory -Path C:\Tools\Sysmon -Force
cd C:\Tools\Sysmon
```

The Microsoft Sysinternals package was downloaded and extracted, and Sysmon was installed with a security-focused XML configuration:

```powershell
.\Sysmon64.exe -accepteula -i .\sysmonconfig.xml
```

The service and event channel were verified with:

```powershell
Get-Service *sysmon*
```

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5
```

## Wazuh Sysmon Collection

Before editing the Wazuh configuration, the original file was backed up:

```powershell
Copy-Item "C:\Program Files (x86)\ossec-agent\ossec.conf" `
          "C:\Program Files (x86)\ossec-agent\ossec.conf.pre-sysmon"
```

The following event channel was added to `ossec.conf`:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

The Wazuh service was restarted:

```powershell
Restart-Service WazuhSvc
```

## Sysmon Detection Validation

Controlled PowerShell activity was generated:

```powershell
powershell.exe -NoProfile -Command "Get-Date"
```

Sysmon recorded the activity as Event ID `1` (Process Create).

Wazuh generated:

```text
Rule ID:       92027
Rule Level:    4
Description:   Powershell process spawned powershell instance
```

MITRE ATT&CK mapping:

```text
T1059.001 - PowerShell
Tactic: Execution
```

The alert preserved useful endpoint evidence including the PowerShell image, command line, parent process, user context, integrity level, process identifiers, and executable hashes.

Evidence:

![Windows Sysmon PowerShell detection - Rule 92027](../evidence/wazuh/08-windows-sysmon-powershell-rule-92027.png)

## PowerShell Script Block Logging

PowerShell Script Block Logging was enabled through Local Group Policy:

```text
Computer Configuration
  → Administrative Templates
  → Windows Components
  → Windows PowerShell
  → Turn on PowerShell Script Block Logging
```

The policy state was verified directly:

```powershell
Get-ItemProperty `
  "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
```

Expected result:

```text
EnableScriptBlockLogging : 1
```

The Wazuh agent was also configured to collect the PowerShell Operational event channel:

```xml
<localfile>
  <location>Microsoft-Windows-PowerShell/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

## Script Block Detection Validation

A controlled registry modification was generated from PowerShell:

```powershell
New-ItemProperty `
  -Path "HKLM:\Software\Microsoft\ADs" `
  -Name "NoofAlerts" `
  -Value 2
```

Windows recorded the command as PowerShell Operational Event ID `4104`.

The event was verified locally with:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Microsoft-Windows-PowerShell/Operational'
    Id=4104
} -MaxEvents 10
```

Wazuh then generated:

```text
Rule ID:       91843
Rule Level:    3
Description:   Powershell executed "New-ItemProperty -Path".
               Possible addition of new item to registry
```

MITRE ATT&CK mapping:

```text
T1059.001 - PowerShell
T1112     - Modify Registry

Tactics:
Execution
Defense Evasion
```

Evidence:

![PowerShell registry modification detection - Rule 91843](../evidence/wazuh/09-powershell-registry-rule-91843.png)

The test value was removed after validation:

```powershell
Remove-ItemProperty `
  -Path "HKLM:\Software\Microsoft\ADs" `
  -Name "NoofAlerts"
```

## Telemetry Path Validated

```text
Windows activity
      ↓
Sysmon / PowerShell Operational logs
      ↓
Wazuh agent on win11-01
      ↓
wazuh01
      ↓
Wazuh detection rule
      ↓
MITRE ATT&CK mapping
      ↓
Threat Hunting investigation
```

## Skills Demonstrated

- Windows 11 endpoint administration
- Proxmox Windows guest integration
- VirtIO drivers and QEMU Guest Agent
- Windows service troubleshooting
- Wazuh Windows-agent deployment
- Endpoint-to-SIEM connectivity validation
- Sysmon deployment and validation
- Sysmon Event ID 1 process analysis
- PowerShell Script Block Logging
- Windows Event ID 4104 analysis
- Wazuh event-channel configuration
- MITRE ATT&CK interpretation
- SOC alert investigation
- Evidence capture and validation
- Troubleshooting and root-cause isolation

## Current Status

- [x] Windows 11 Pro VM created
- [x] Hostname set to `win11-01`
- [x] QEMU Guest Agent installed and running
- [x] Wazuh Windows agent installed and active
- [x] Wazuh manager connectivity on TCP 1514 verified
- [x] Sysmon installed and producing events
- [x] Sysmon Operational channel collected by Wazuh
- [x] Wazuh Rule 92027 validated
- [x] PowerShell Script Block Logging enabled
- [x] Event ID 4104 validated locally
- [x] PowerShell Operational channel collected by Wazuh
- [x] Wazuh Rule 91843 validated
- [ ] Join `win11-01` to Active Directory after `dc01` is promoted
