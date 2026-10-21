# Active Directory Lab

## Overview

I added Active Directory to the homelab to get more experience with Windows domain administration, authentication, DNS, and security monitoring.

Until this point, most of my work had involved Linux systems, Proxmox, networking, and Wazuh. I wanted to add a Windows domain and see how identity-management events could be collected and investigated through the SIEM.

I deployed Windows Server 2025 as a domain controller, created the `corp.home.arpa` domain, joined a Windows 11 workstation, and used test account-management activity to generate security events.

I then verified that Wazuh detected the creation of an Active Directory account.

---

## 1. Lab Environment

| System | IP address | Purpose |
|---|---|---|
| `dc01` | `10.10.20.10` | Domain controller and DNS |
| `win11-01` | `10.10.30.160` | Domain-joined Windows 11 endpoint |
| `wazuh01` | `10.10.40.20` | Security monitoring |

The domain configuration is:

```text
DNS domain:        corp.home.arpa
NetBIOS domain:    CORP
Domain controller: dc01.corp.home.arpa
Operating system:  Windows Server 2025
```

`dc01` runs on VLAN 20, while `win11-01` is on VLAN 30.

OPNsense handles permitted traffic between the networks.

---

## 2. Installing Active Directory Domain Services

I installed the Active Directory Domain Services role and the associated administration tools on `dc01`.

The PowerShell command for that configuration is:

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

The installed role can be checked with:

```powershell
Get-WindowsFeature AD-Domain-Services
```

After the role was installed, I promoted the server to a domain controller and created a new forest.

The resulting domain was:

`corp.home.arpa`

With the NetBIOS name:

`CORP`

The original documentation doesn't preserve the exact promotion command, so I'm not including a reconstructed command as though it came from the deployment session.

---

## 3. Validating the Domain

After promotion and restart, I checked the Active Directory configuration using:

```powershell
Get-ADDomain
Get-ADForest
Get-ADDomainController
```

The results confirmed that the domain and forest existed and that the domain controller was available.

The recorded functional levels were:

```text
Domain functional level: Windows2025Domain
Forest functional level: Windows2025Forest
```

These commands also provide information about domain identity, forest configuration, and FSMO role ownership.

At this stage, I had a functioning domain controller and could begin connecting Windows clients.

---

## 4. Active Directory DNS

DNS is central to Active Directory operation.

Windows clients need to find services such as domain controllers, LDAP, and Kerberos.

I configured the Windows 11 workstation to use `dc01` (`10.10.20.10`) for Active Directory DNS.

Useful checks include:

```powershell
Resolve-DnsName corp.home.arpa
Resolve-DnsName dc01.corp.home.arpa
```

To check domain-controller service discovery:

```powershell
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.corp.home.arpa
Resolve-DnsName -Type SRV _kerberos._tcp.corp.home.arpa
```

The SRV records help clients locate the services required for domain operations.

This is an important distinction from the Pi-hole server I use for general home-network DNS.

For the domain-joined Windows systems, Active Directory DNS must be available for normal domain discovery and authentication.

A DNS problem can easily appear to be a domain-controller or Windows authentication problem.

---

## 5. Joining Windows 11 to the Domain

I joined `win11-01` to:

`corp.home.arpa`

After restarting the workstation, I checked its domain membership:

```powershell
Get-CimInstance Win32_ComputerSystem |
    Select-Object Name, Domain, PartOfDomain
```

I also checked the secure channel:

```powershell
Test-ComputerSecureChannel
```

The test returned:

```text
True
```

This confirmed that the workstation's trust relationship with the domain was functioning.

Additional diagnostic commands include:

```powershell
nltest /dsgetdc:corp.home.arpa
nltest /sc_verify:corp.home.arpa
```

These are useful when investigating domain-controller discovery and secure-channel problems.

With the workstation joined, I had a Windows endpoint capable of producing both local security events and domain-related activity.

---

## 6. Inspecting the Password Policy

Before creating test accounts, I checked the domain's default password policy.

```powershell
Get-ADDefaultDomainPasswordPolicy |
Select-Object MinPasswordLength,
              ComplexityEnabled,
              PasswordHistoryCount,
              MinPasswordAge,
              MaxPasswordAge
```

The recorded policy was:

| Setting | Observed value |
|---|---|
| Minimum password length | 7 |
| Complexity enabled | True |
| Password history | 24 |
| Minimum password age | 1 day |
| Maximum password age | 42 days |

These were the observed lab settings, not recommendations for a new production password policy.

### Password-Policy Troubleshooting

My first attempt to configure the test account's password ran into password-policy enforcement.

I supplied a new compliant password and reset the account password.

The documented PowerShell commands were:

```powershell
$pw = Read-Host "Enter a NEW compliant password for soc-lab-user" -AsSecureString

Set-ADAccountPassword `
    -Identity "soc-lab-user" `
    -Reset `
    -NewPassword $pw
```

The process was useful because it showed the practical effect of the domain password policy during account administration.

It also provided another example of why checking configuration and understanding the actual error is better than repeatedly attempting the same operation.

---

## 7. Creating a Test Domain Account

I used a test account named:

`soc-lab-user`

The purpose was to generate account-management activity and follow the resulting security event through Windows and Wazuh.

I checked the account using:

```powershell
Get-ADUser soc-lab-user -Properties *
```

For a shorter view of the relevant attributes:

```powershell
Get-ADUser soc-lab-user |
Select-Object Name, SamAccountName, Enabled, DistinguishedName
```

These commands provide information about the account's identity, enabled state, and location in Active Directory.

---

## 8. Windows Security Event 4720

Creating the test user generated Windows Security Event ID `4720`.

The event indicates that a user account was created.

I checked the domain controller's Security log:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id      = 4720
} -MaxEvents 10
```

To find events associated with the test account:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id      = 4720
} -MaxEvents 20 |
Where-Object { $_.Message -match 'soc-lab-user' } |
Select-Object TimeCreated, Id, ProviderName, Message
```

This allowed me to examine the original Windows evidence rather than relying entirely on what Wazuh displayed.

---

## 9. Wazuh Detection

The account-creation event was successfully collected and detected by Wazuh.

The recorded alert included:

| Field | Value |
|---|---|
| Windows Event ID | `4720` |
| Wazuh rule | `60109` |
| Rule level | 8 |
| MITRE ATT&CK | `T1098` — Account Manipulation |
| Tactic | Persistence |

The telemetry path was:

```text
Active Directory account creation
              |
              v
Windows Security Event 4720
              |
              v
Wazuh Windows event collection
              |
              v
Wazuh manager — wazuh01
              |
              v
Rule 60109
              |
              v
Security alert
```

This was the part I was particularly interested in.

I had performed a known administrative action, confirmed the event in Windows, and then located the corresponding Wazuh detection.

The alert showed that domain account-management events could be collected and used for SOC investigation.

---

## 10. Disabling and Deleting Accounts

I also worked through additional account lifecycle operations.

### Disable an Account

```powershell
Disable-ADAccount -Identity "soc-lab-user"
```

The account's state can be checked with:

```powershell
Get-ADUser soc-lab-user |
Select-Object Name, SamAccountName, Enabled
```

Disabling an account can generate Windows Security Event ID `4725`.

### Delete an Account

```powershell
Remove-ADUser -Identity "soc-lab-user"
```

Account deletion can generate Windows Security Event ID `4726`.

The key account-management events are:

| Event ID | Meaning |
|---|---|
| `4720` | User account created |
| `4725` | User account disabled |
| `4726` | User account deleted |

The documented Wazuh detection was the account-creation event. I'm not treating the other two event IDs as separately verified Wazuh alerts without matching evidence.

---

## 11. Investigating an Account-Creation Alert

A new account isn't automatically suspicious.

An administrator may create accounts as part of normal operations.

The important question is whether the activity was expected and authorized.

For an unexpected account-creation event, I would investigate:

- Which administrator created the account
- When the account was created
- Where the administrative activity originated
- Whether the account was added to privileged groups
- Whether it subsequently authenticated
- Whether related accounts were created or modified
- Whether suspicious PowerShell or other process activity occurred around the same time

The Windows Security event establishes that an account-management action occurred. Other evidence is needed to determine its significance.

This is the same general approach I used in the earlier Wazuh detection labs: verify the original event, examine the alert, and determine what else would be needed to understand the activity.

---

## 12. Result

The Active Directory lab established a functioning Windows Server 2025 domain with a domain-joined Windows 11 workstation.

I verified domain and forest configuration, Active Directory DNS, and the workstation's secure channel.

I also inspected password-policy enforcement and performed test account-management operations.

The main security-monitoring result was detecting Windows Security Event 4720 through Wazuh Rule 60109.

This connected Active Directory administration with the broader SOC monitoring environment.

The next steps are to expand Windows identity monitoring to additional authentication and privilege-management events, and correlate that activity with Sysmon and PowerShell telemetry.

## Related Documentation

- [Windows Endpoint Telemetry](08-windows-telemetry.md)
- [Wazuh SIEM Deployment](06-wazuh-siem.md)
- [Wazuh Attack Detection Labs](09-attack-detection-labs.md)
- [SOC Alert Triage Playbook](12-soc-alert-triage-playbook.md)
- [Managed Switching and VLANs](04-network-vlans.md)
