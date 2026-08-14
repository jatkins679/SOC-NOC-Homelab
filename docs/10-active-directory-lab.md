# Active Directory Lab

## Overview

This lab adds Microsoft Active Directory Domain Services (AD DS) to the SOC/NOC
homelab so Windows identity, authentication, domain administration, and security
monitoring can be practiced in an environment that resembles a small enterprise
network.

The Active Directory environment is integrated with the broader security lab,
including a domain-joined Windows 11 endpoint and Wazuh SIEM monitoring.

## Lab Objectives

The goals of this phase were to:

- Deploy an Active Directory domain controller.
- Configure Active Directory-integrated DNS.
- Create a new forest and domain.
- Join a Windows 11 workstation to the domain.
- Verify DNS-based domain discovery.
- Validate the workstation's secure channel to the domain.
- Inspect and understand the domain password policy.
- Create, disable, and delete test domain accounts.
- Generate Windows Security events associated with account administration.
- Ingest relevant Windows events into Wazuh.
- Validate that account-management activity produces actionable SIEM alerts.
- Document the administrative and troubleshooting workflow.

## Environment

| System | Role | Address / Name |
|---|---|---|
| `dc01` | Active Directory Domain Controller / DNS | `192.168.1.30` |
| `dc01.corp.home.arpa` | Domain controller FQDN | `corp.home.arpa` |
| `win11-01` | Domain-joined Windows 11 workstation | DHCP / lab LAN |
| `wazuh01` | Wazuh SIEM | `192.168.1.206` |
| `CORP` | NetBIOS domain name | `corp.home.arpa` |

The Active Directory DNS domain is:

```text
corp.home.arpa
```

The NetBIOS domain name is:

```text
CORP
```

## Architecture

```text
                         SOC/NOC Homelab
                               |
                        192.168.1.0/24
                               |
               +---------------+---------------+
               |                               |
            dc01                           wazuh01
        192.168.1.30                    192.168.1.206
      Active Directory                    Wazuh
      DNS / Kerberos                       SIEM
               |
               |
          corp.home.arpa
               |
               |
           win11-01
       Windows 11 endpoint
               |
        Wazuh + Sysmon
```

This arrangement allows identity-management actions on the domain to be
observed from both an administrator and a security-operations perspective.

---

## 1. Installing Active Directory Domain Services

The domain controller was configured with the Active Directory Domain Services
role and the associated management tools.

Example PowerShell installation:

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

The installed role can be verified with:

```powershell
Get-WindowsFeature AD-Domain-Services
```

**Purpose:** Confirm that the AD DS server role and management components are
installed before attempting to promote the server to a domain controller.

---

## 2. Creating the Domain

The lab domain was created as:

```text
corp.home.arpa
```

with the NetBIOS name:

```text
CORP
```

The domain controller is:

```text
dc01.corp.home.arpa
```

After promotion, the server was restarted and Active Directory services were
validated.

Useful verification commands include:

```powershell
Get-ADDomain
Get-ADForest
Get-ADDomainController
```

During validation, `Get-ADDomain` / `Get-ADForest` reported the domain and forest
functional levels as:

```text
Windows2025Domain
Windows2025Forest
```

These commands provide information about the domain, forest, domain controller,
FSMO role ownership, and functional levels.

---

## 3. DNS Configuration

Active Directory depends heavily on DNS. The domain controller provides DNS for
the `corp.home.arpa` domain and domain clients use the domain controller for
Active Directory name resolution and service discovery.

The domain controller uses:

```text
192.168.1.30
```

The Windows 11 domain workstation was configured to use:

```text
192.168.1.30
```

as its DNS server.

This is important because Active Directory clients locate services such as LDAP
and Kerberos through DNS SRV records.

Useful validation commands include:

```powershell
Resolve-DnsName corp.home.arpa
Resolve-DnsName dc01.corp.home.arpa
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.corp.home.arpa
Resolve-DnsName -Type SRV _kerberos._tcp.corp.home.arpa
```

**SOC/NOC relevance:** DNS problems can appear to be authentication,
workstation, or domain-controller failures. Verifying DNS is therefore one of
the first troubleshooting steps for Active Directory connectivity.

---

## 4. Joining `win11-01` to the Domain

The Windows 11 test workstation was joined to:

```text
corp.home.arpa
```

After the join and reboot, domain membership was validated.

Examples:

```powershell
Get-CimInstance Win32_ComputerSystem |
    Select-Object Name, Domain, PartOfDomain
```

The secure channel between the workstation and the domain was tested with:

```powershell
Test-ComputerSecureChannel
```

Expected result:

```text
True
```

A successful secure-channel test demonstrates that the workstation can
authenticate to the domain and that the computer account relationship is
healthy.

Additional domain discovery can be tested with:

```powershell
nltest /dsgetdc:corp.home.arpa
nltest /sc_verify:corp.home.arpa
```

These commands help diagnose domain-controller discovery and workstation trust
problems.

---

## 5. Inspecting the Domain Password Policy

Before creating a test account, the domain password policy was inspected:

```powershell
Get-ADDefaultDomainPasswordPolicy |
Select-Object MinPasswordLength,
              ComplexityEnabled,
              PasswordHistoryCount,
              MinPasswordAge,
              MaxPasswordAge
```

Observed policy:

```text
MinPasswordLength    : 7
ComplexityEnabled    : True
PasswordHistoryCount : 24
MinPasswordAge       : 1.00:00:00
MaxPasswordAge       : 42.00:00:00
```

The first attempt to create a test account encountered password-policy
enforcement. A new compliant password was supplied and the account password was
reset successfully.

Example:

```powershell
$pw = Read-Host "Enter a NEW compliant password for soc-lab-user" -AsSecureString

Set-ADAccountPassword `
    -Identity "soc-lab-user" `
    -Reset `
    -NewPassword $pw
```

**Why this matters:** Password policies are an important identity-security
control. Administrators and security analysts need to understand both how the
policy is configured and how failed administrative actions appear in system
logs.

---

## 6. Creating a Test Domain Account

A test account named:

```text
soc-lab-user
```

was used to generate controlled Active Directory account-management telemetry.

The purpose was not simply to create a user. The exercise connected an
administrative action to:

1. Active Directory state
2. Windows Security logging
3. Wazuh ingestion
4. SIEM rule matching
5. MITRE ATT&CK context

The account can be inspected with:

```powershell
Get-ADUser soc-lab-user -Properties *
```

A more concise query is:

```powershell
Get-ADUser soc-lab-user |
Select-Object Name, SamAccountName, Enabled, DistinguishedName
```

---

## 7. Active Directory Security Event: User Creation

Creating the test account generated Windows Security Event ID:

```text
4720
```

Event 4720 means:

```text
A user account was created.
```

This is a useful security event because unexpected account creation can indicate
unauthorized persistence, compromised administrator credentials, or malicious
identity manipulation.

The event can be queried directly on the domain controller:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id      = 4720
} -MaxEvents 10
```

A more targeted search can inspect the rendered event text:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id      = 4720
} -MaxEvents 20 |
Where-Object { $_.Message -match 'soc-lab-user' } |
Select-Object TimeCreated, Id, ProviderName, Message
```

---

## 8. Wazuh Detection of the AD Account Creation

The Windows account-creation event was successfully ingested and detected by
Wazuh.

Observed detection:

```text
Windows Security Event ID : 4720
Wazuh rule                : 60109
Rule level                : 8
MITRE ATT&CK              : T1098 - Account Manipulation
Tactic                    : Persistence
```

This validates the complete telemetry path:

```text
Active Directory administrative action
               |
               v
Windows Security Event Log
               |
               v
Wazuh agent / Windows event collection
               |
               v
Wazuh manager
               |
               v
Rule 60109
               |
               v
Security alert / investigation
```

This is a key SOC exercise because the lab is not only generating events; it is
demonstrating that endpoint/domain telemetry reaches a SIEM and produces a
security-relevant alert.

---

## 9. Disabling and Deleting the Test Account

Additional identity lifecycle actions were performed against the test user.

The account can be disabled with:

```powershell
Disable-ADAccount -Identity "soc-lab-user"
```

The account state can be verified with:

```powershell
Get-ADUser soc-lab-user |
Select-Object Name, SamAccountName, Enabled
```

Disabling a user account generates Windows Security Event ID:

```text
4725
```

The test account can later be removed with:

```powershell
Remove-ADUser -Identity "soc-lab-user"
```

Deletion of a user account generates Windows Security Event ID:

```text
4726
```

These events are useful for identity lifecycle monitoring and for investigating
unexpected administrative changes.

---

## 10. Event IDs Used in the Lab

| Event ID | Meaning | Security relevance |
|---|---|---|
| `4720` | User account created | Possible unauthorized account creation / persistence |
| `4725` | User account disabled | Administrative or containment action; potentially malicious disruption |
| `4726` | User account deleted | Account lifecycle event; possible evidence destruction or unauthorized change |

Account-management events should be interpreted in context. A legitimate
administrator may generate exactly the same Windows event IDs as an attacker.
The analyst's task is to determine whether the action is expected and
authorized.

---

## 11. Investigation Workflow

The lab follows a simple SOC investigation process:

```text
Alert
  |
  v
Identify event ID and rule
  |
  v
Identify affected account
  |
  v
Identify actor / administrator
  |
  v
Confirm source system and timestamp
  |
  v
Determine whether change was authorized
  |
  v
Review related authentication/process activity
  |
  v
Contain or remediate if necessary
  |
  v
Document findings
```

For an unexpected Event 4720, useful questions include:

- Who created the account?
- When was it created?
- From which system was the action performed?
- Was the administrator expected to perform the action?
- Was the new account added to any privileged groups?
- Was there suspicious authentication activity immediately before the event?
- Did PowerShell or another administrative tool launch near the same time?
- Did the account subsequently log on?
- Were other accounts created, modified, disabled, or deleted?

---

## 12. Why Active Directory Matters to a SOC

Active Directory is a high-value security-monitoring source because domain
identity is central to many Windows enterprise environments.

An attacker with control of an Active Directory account may be able to:

- Access additional systems.
- Create persistence.
- Change group memberships.
- Reset passwords.
- Disable legitimate users.
- Create additional accounts.
- Abuse administrative privileges.
- Move laterally between systems.

For this reason, security monitoring commonly includes:

- Account creation
- Account deletion
- Account disablement
- Password resets
- Group-membership changes
- Privileged logons
- Failed logons
- Kerberos activity
- Domain-controller events
- PowerShell and process telemetry

The lab provides a controlled environment in which these behaviors can be
generated and investigated without affecting a production organization.

---

## 13. Skills Demonstrated

This phase demonstrates hands-on experience with:

### Windows / Active Directory

- Active Directory Domain Services
- Active Directory-integrated DNS
- Domain controller administration
- Domain and forest validation
- Windows domain joins
- LDAP/Kerberos service discovery
- Computer secure-channel verification
- Active Directory user administration
- Domain password-policy inspection
- PowerShell Active Directory cmdlets

### Security Operations

- Windows Security Event Logs
- Identity and account-management telemetry
- SIEM ingestion
- Wazuh alert investigation
- Security rule interpretation
- MITRE ATT&CK mapping
- Account Manipulation (`T1098`)
- Security-event correlation
- Detection validation

### Troubleshooting

- DNS verification
- Domain-controller discovery
- Domain trust validation
- Password-policy troubleshooting
- Event-log verification
- End-to-end telemetry validation

---

## 14. Result

The Active Directory lab successfully established a functioning Windows domain
and connected domain administration to the SOC monitoring stack.

Validated results include:

- `corp.home.arpa` domain operational.
- `dc01.corp.home.arpa` functioning as the domain controller.
- Active Directory DNS functioning.
- `win11-01` successfully joined to the domain.
- Workstation secure channel validated.
- Domain password policy inspected and enforced.
- Test account lifecycle activity generated.
- Event 4720 generated for domain-user creation.
- Active Directory account creation ingested by Wazuh.
- Wazuh rule `60109` generated a level 8 alert.
- Detection mapped to MITRE ATT&CK `T1098` Account Manipulation.

The next phase is to expand Active Directory monitoring to additional
authentication and privilege-management events and correlate those events with
Sysmon and other Windows telemetry.
