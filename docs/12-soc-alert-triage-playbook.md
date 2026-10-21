# SOC Alert Triage and Investigation Playbook

## Overview

This is the process I use to investigate security alerts generated in my homelab.

Most of the examples come from controlled Wazuh exercises, where I already know what activity was generated. That gives me an opportunity to check whether the SIEM accurately represents what happened on the endpoint.

The process is intended to be useful beyond those exercises. When an alert appears, I want to identify the affected systems, confirm the underlying activity, determine its significance, and document what I found.

I don't want to assume an alert is malicious simply because Wazuh assigned it a high rule level.

I also don't want to assume a firewall pass log, successful ping, or functioning agent proves that the entire system is working correctly.

The evidence needs to support the conclusion.

---

## 1. Initial Triage

When an alert first appears, I record the basic details.

| Field | What I'm looking for |
|---|---|
| Timestamp | When the activity occurred |
| Rule or event ID | What triggered the alert |
| Severity | Wazuh level and initial priority |
| Host | Which endpoint generated the event |
| User | Account associated with the activity |
| Source | Originating IP or system |
| Destination | Target system or service |
| Process or command | What actually executed |
| Event count | Single occurrence or repeated activity |

My first objective is to understand what the alert is claiming.

I then look for the event in the original logs.

### Investigation Sequence

```text
Alert received
       |
       v
Identify event and affected systems
       |
       v
Verify the original source log
       |
       v
Determine whether activity is expected
       |
       v
Search for related events
       |
       v
Assess impact and priority
       |
       v
Take action if required
       |
       v
Verify the result
       |
       v
Document findings
```

I try not to make changes until I understand enough of the situation to know what I'm changing and why.

---

## 2. Checking the Source Logs

The original endpoint or application log is often the best place to verify what the SIEM reported.

### SSH Authentication

On a Linux endpoint:

```bash
sudo journalctl -u ssh --no-pager
```

To narrow the results to a known source:

```bash
sudo journalctl -u ssh --no-pager | grep '10.10.50.113'
```

I compare the timestamps, usernames, source addresses, and failed-login counts.

I also check whether successful authentication occurred after the failures.

### Apache Access Logs

```bash
sudo tail -n 100 /var/log/apache2/access.log
```

For the SQL-injection-style test:

```bash
sudo grep -n "SELECT" /var/log/apache2/access.log
```

I look at the requested path, source IP, HTTP method, response code, and nearby requests.

A suspicious request does not automatically mean the application was compromised.

### File Integrity Monitoring

For an unexpected file change, I check the path, modification time, and current file contents when the file is still available.

```bash
stat /path/to/file
sha256sum /path/to/file
```

I compare those results with the Wazuh FIM event, including its recorded hashes and content differences.

The current file hash may have changed again since the alert, so timestamps and event history also matter.

### Linux Auditd

```bash
sudo ausearch -ts recent
```

To look for a specific executable:

```bash
sudo ausearch -x sudo
```

I pay particular attention to AUID, UID, and EUID.

Those fields can distinguish the original login identity from the privileges used to execute a command.

### Windows Security Events

From an administrative PowerShell session:

```powershell
Get-WinEvent -LogName Security -MaxEvents 50
```

For account creation:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4720
} -MaxEvents 20
```

I look for the account that performed the action, the affected account, the time, and any related authentication activity.

For PowerShell or process events, I also examine the appropriate Operational event channels.

---

## 3. Classifying the Alert

An alert can accurately identify an event without the event being malicious.

I use four basic classifications.

| Classification | Meaning |
|---|---|
| True positive — suspicious | The intended behavior was detected and warrants investigation |
| True positive — expected | The intended behavior occurred, but it was authorized or part of testing |
| False positive | The alert incorrectly identifies behavior as matching its intended detection |
| Needs more information | There isn't enough evidence to classify the event confidently |

Most of my controlled Wazuh exercises are **true positives involving expected activity**.

For example, an SSH brute-force alert may accurately describe repeated failed authentication attempts, even when I deliberately generated them from Kali Linux.

I want to keep detection accuracy and malicious intent as separate questions.

---

## 4. Scoping Related Activity

After confirming the alert, I check whether the event was isolated.

I normally start with a short window around the original timestamp, such as 15 minutes before and after the event.

Depending on what I find, I expand the search.

I look for relationships involving:

- The same host
- The same user or account
- The same source IP
- The same process or parent process
- Similar commands or file changes
- Additional authentication activity
- Events on other systems

For an SSH alert, I would look for successful logins after the failures.

For an Active Directory account-creation alert, I would look for group-membership changes, privileged logons, or subsequent activity by the new account.

The objective is to determine whether the alert represents an isolated event or part of a larger sequence.

---

## 5. Assigning Priority

A Wazuh rule level provides useful information, but priority also depends on the affected system, the behavior, and the evidence.

For the homelab, I use a simple model.

| Priority | Example |
|---|---|
| P1 — Critical | Confirmed destructive activity or serious compromise affecting important systems |
| P2 — High | Suspected unauthorized privileged access or persistence |
| P3 — Medium | Repeated authentication failures, suspicious commands, or unexpected file changes |
| P4 — Low | Minor event requiring review, with no apparent impact |
| P5 — Testing | Known controlled activity generated for a lab exercise |

These are local working priorities, not a production SOC's official incident-severity standard.

A controlled test might produce a Wazuh Level 10 alert but still receive P5 for operational handling because I generated the activity intentionally.

---

## 6. Investigation Example: SSH Brute Force

In one exercise, I generated eight failed SSH authentication attempts from `kali01` (`10.10.50.113`) against `target01` (`10.10.60.10`).

Wazuh produced:

| Field | Result |
|---|---|
| Rule | `5712` |
| Level | 10 |
| Technique | `T1110` — Brute Force |
| Targeted account | `baduser` |
| Correlated failures | 8 |

I checked the SSH journal on `target01` and verified the source IP and failed authentication records.

The alert accurately reflected the test.

**Assessment:** True positive — expected.

The same alert outside a known test would require additional investigation, particularly whether the source attempted other usernames or subsequently authenticated successfully.

[Full SSH detection exercise](09-attack-detection-labs.md)

---

## 7. Investigation Example: File Integrity Change

I configured Wazuh to monitor `/opt/wazuh-lab` on `target01`.

After changing a test configuration file, Wazuh generated:

| Field | Result |
|---|---|
| Rule | `550` |
| Level | 7 |
| Event | File modified |
| Technique | `T1565.001` |

The alert included before-and-after hashes and the added text.

I compared those details with the actual modification performed during the test.

**Assessment:** True positive — expected.

In an unexpected case, I'd also check who changed the file, whether the modification was authorized, and whether it affected a service or security control.

[Full FIM exercise](09-attack-detection-labs.md)

---

## 8. Investigation Example: Suspicious Web Request

I sent an HTTP request containing SQL-like syntax from `kali01` to Apache on `target01`.

Wazuh reported:

| Field | Result |
|---|---|
| Rule | `31103` |
| Level | 7 |
| Technique | `T1190` |
| HTTP response | 404 |

I verified the request in the Apache access log.

The requested application path didn't exist.

**Assessment:** True positive — expected; suspicious request detected, successful exploitation not demonstrated.

This distinction is important. An alert indicating an SQL-injection attempt doesn't prove that a database was accessed or modified.

[Full web detection exercise](09-attack-detection-labs.md)

---

## 9. Investigation Example: Active Directory Account Creation

I created the test domain account `soc-lab-user` on `dc01` (`10.10.20.10`).

Windows recorded Security Event ID `4720`.

Wazuh generated:

| Field | Result |
|---|---|
| Windows event | `4720` |
| Wazuh rule | `60109` |
| Level | 8 |
| Technique | `T1098` — Account Manipulation |

I checked the Windows Security event and the corresponding Wazuh alert.

**Assessment:** True positive — expected.

For an unexpected account, I would determine who created it, whether it had privileged memberships, and whether it had been used to authenticate.

[Active Directory investigation](10-active-directory-lab.md)

---

## 10. Containment and Remediation

Actions should depend on the evidence and the impact.

Examples include disabling a compromised account, restricting a network connection, stopping a malicious process, or restoring a modified configuration file.

In the lab, I generally prefer controlled and reversible actions that let me verify the effect of the change.

I don't want to disable a service or modify firewall rules simply because an alert was generated.

Before taking action, I want to know which system is affected and whether the action might interrupt other services.

---

## 11. Verifying the Result

After remediation, I check whether the original problem was resolved and whether the system still performs its intended function.

For example, after investigating a Wazuh agent issue:

```bash
systemctl is-active wazuh-agent
```

I would also verify that new telemetry appears in Wazuh.

A running service alone isn't sufficient evidence that the complete monitoring path works.

For an account issue, I can inspect the account state:

```powershell
Get-ADUser soc-lab-user |
Select-Object Name, SamAccountName, Enabled
```

For a file change, I check the restored file and its contents or hashes.

The goal is to validate the outcome, not simply confirm that a remediation command completed.

---

## 12. Investigation Notes Template

I use the following structure when documenting security investigations.

```text
Alert / Rule:
Event ID:
Time:
Severity / Priority:

Affected host:
User or account:
Source:
Destination:
Process / command:

Initial observation:

Source-log evidence:

Related events:

Classification:
[ ] True positive — suspicious
[ ] True positive — expected
[ ] False positive
[ ] Needs more information

Impact:

Action taken:

Post-action verification:

Final assessment:

Evidence references:
```

The notes should make it possible for someone else to understand what happened and how I reached the conclusion.

For a live incident, I would also record escalation and handoff details as appropriate.

---

## 13. Escalation

In a real SOC, escalation depends on the organization's procedures and the analyst's authority.

Events that would generally warrant escalation include confirmed unauthorized privileged access, evidence of malware or persistence, activity affecting multiple systems, suspected data theft, or destructive behavior.

An investigation may also need escalation when the evidence is incomplete but the potential impact is significant.

A useful handoff should include the original alert, affected systems, relevant timestamps, supporting logs, preliminary assessment, and actions already taken.

---

## 14. Evidence Handling

For lab investigations, I retain useful technical evidence such as:

- Wazuh rule and event details
- Original endpoint or application log entries
- Timestamps and hostnames
- Usernames and relevant process information
- Source and destination addresses
- File hashes
- Configuration snippets
- Screenshots
- Verification commands and results

I exclude passwords, credentials, and other sensitive information from the public repository.

Where a command or result wasn't preserved during the original exercise, I distinguish a later diagnostic example from historical evidence.

---

## Result

This playbook reflects the approach I've developed through the Wazuh detection labs and related systems troubleshooting.

The recurring process is straightforward: identify the alert, verify the event at its source, understand the context, investigate related activity, determine whether action is necessary, and check the result.

I expect the playbook to change as I add new detection exercises and gain more experience with security monitoring.

## Related Documentation

- [Wazuh SIEM Deployment](06-wazuh-siem.md)
- [Windows Endpoint Telemetry](08-windows-telemetry.md)
- [Wazuh Attack Detection Labs](09-attack-detection-labs.md)
- [Active Directory Lab](10-active-directory-lab.md)
- [SOC/NOC Skills Matrix](11-soc-noc-skills-matrix.md)
