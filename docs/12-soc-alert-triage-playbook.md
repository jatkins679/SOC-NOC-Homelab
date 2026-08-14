# SOC Alert Triage and Investigation Playbook

## Purpose

This playbook documents the investigation workflow I use in the SOC/NOC homelab
when a security alert appears in Wazuh or another monitoring source.

The goal is not to create a fictional enterprise incident-response program. It is
to demonstrate a repeatable analyst process for answering:

1. What happened?
2. Is the alert real?
3. Which system and user are involved?
4. How serious is it?
5. Is the activity expected?
6. Did anything else happen around the same time?
7. What action should be taken?
8. How do I prove the issue is resolved?

The process is intentionally evidence-driven.

---

# 1. Triage Workflow

```text
Alert received
      |
      v
Identify rule / event / severity
      |
      v
Identify host, user, source, destination
      |
      v
Check source telemetry
      |
      v
Validate whether activity actually occurred
      |
      v
Determine expected vs suspicious
      |
      v
Scope related activity
      |
      v
Assign severity / priority
      |
      v
Contain or remediate if needed
      |
      v
Verify normal state
      |
      v
Document findings
```

The most important rule is:

> Do not treat the SIEM alert as the only source of truth.

Whenever possible, I validate the alert against the original endpoint,
application, authentication, or operating-system log.

---

# 2. Initial Triage Questions

When reviewing a new alert, I begin with:

- What is the alert title or rule description?
- What rule ID or event ID triggered?
- What severity or level was assigned?
- Which host generated the event?
- Which user or account was involved?
- What source IP initiated the activity?
- What destination system or service was affected?
- What process or command was involved?
- When did the activity occur?
- Is the event isolated or repeated?
- Is the activity expected in the context of the lab?

These questions establish the minimum context needed before deciding whether the
alert is important.

---

# 3. Validate the Source Event

## Linux Authentication

For SSH-related alerts, check the system journal or authentication logs.

Examples:

```bash
sudo journalctl -u ssh
```

or:

```bash
sudo grep -i "failed" /var/log/auth.log
```

Questions:

- Does the source log show the same timestamp?
- Is the username the same as the Wazuh alert?
- Is the source IP consistent?
- How many failed attempts occurred?
- Was there a successful login afterward?

---

## Apache / Web Logs

For web alerts:

```bash
sudo tail -n 100 /var/log/apache2/access.log
```

or:

```bash
sudo grep -n "SELECT" /var/log/apache2/access.log
```

Questions:

- Is the suspicious request actually present?
- Which source IP sent it?
- Which URI was requested?
- What HTTP status code was returned?
- Was the request repeated?
- Were other suspicious requests sent nearby in time?

---

## File Integrity Monitoring

For FIM alerts:

- Confirm the file path.
- Confirm the action: create, modify, delete.
- Compare hashes if available.
- Review captured file-content differences.
- Confirm whether the file change was expected.

Useful endpoint commands include:

```bash
stat /path/to/file
sha256sum /path/to/file
```

---

## Linux Auditd

For Auditd-related alerts:

```bash
sudo ausearch -ts recent
```

or for a specific executable or user:

```bash
sudo ausearch -x sudo
```

Questions:

- Which user executed the command?
- What UID, EUID, and AUID were recorded?
- Was the command run through `sudo`?
- What executable was launched?
- Was the action expected?

---

## Windows Security Events

For Windows events:

```powershell
Get-WinEvent -LogName Security -MaxEvents 50
```

For a specific event:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id      = 4720
} -MaxEvents 20
```

Questions:

- Which account performed the action?
- Which account was affected?
- What system generated the event?
- Was the change expected?
- Was there related login, process, or PowerShell activity?

---

# 4. Determine Whether the Alert Is a True Positive

An alert is not automatically malicious.

I classify the alert into one of these categories:

| Classification | Meaning |
|---|---|
| **True Positive — Malicious / Suspicious** | Alert accurately represents activity that requires investigation or action |
| **True Positive — Expected** | Alert accurately detected activity, but the action was authorized or part of testing |
| **False Positive** | Alert logic matched something benign that does not represent the intended behavior |
| **Needs More Context** | Insufficient evidence to classify confidently |

In the homelab, most controlled exercises are:

```text
True Positive — Expected
```

because the activity was deliberately generated to validate monitoring.

The analytical value comes from proving that the alert accurately reflects the
underlying event.

---

# 5. Scope the Activity

After confirming the event occurred, determine whether it was isolated.

## Search by Host

Look for other events from the same system around the same time.

Examples:

- authentication failures
- process creation
- PowerShell activity
- account changes
- file modifications
- service changes
- network connections

## Search by User

Determine whether the same user or account appears in:

- logon events
- privileged commands
- account administration
- PowerShell execution
- process creation
- authentication failures

## Search by Source IP

Determine whether the source system generated:

- additional authentication attempts
- scans
- web requests
- connections to other targets

## Search by Time Window

A practical investigation window is:

```text
5–15 minutes before the event
through
5–15 minutes after the event
```

Expand the window if related events are found.

---

# 6. Severity and Priority

Severity should reflect both the technical event and the environment.

I use the following simple lab prioritization model:

| Priority | Example |
|---|---|
| **P1 — Critical** | Confirmed compromise, destructive activity, privileged persistence, widespread impact |
| **P2 — High** | Suspicious privileged account creation, repeated successful unauthorized access, serious endpoint activity |
| **P3 — Medium** | Repeated failed authentication, suspicious process, web attack attempt, unexpected file change |
| **P4 — Low** | Informational event, isolated benign activity, expected administrative change |
| **P5 — Testing / Informational** | Controlled lab event generated intentionally |

A Wazuh rule level is useful context, but it is not the only factor in priority.

---

# 7. Investigation Example — SSH Brute Force

## Alert

```text
Wazuh Rule: 5712
Level: 10
MITRE ATT&CK: T1110 - Brute Force
```

## Triage

Questions:

- Which endpoint reported the event?
- What source IP generated the attempts?
- Which username was targeted?
- How many failed attempts occurred?
- Was any authentication successful?

## Validation

Check the SSH journal on `target01`:

```bash
sudo journalctl -u ssh
```

Compare:

- timestamp
- source IP
- username
- failure count

## Classification

For the controlled exercise:

```text
True Positive — Expected
```

because the activity was intentionally generated from `kali01`.

## Analyst Conclusion

Wazuh successfully detected repeated SSH authentication failures and the
underlying endpoint logs independently confirmed the event.

---

# 8. Investigation Example — File Integrity Change

## Alert

```text
Wazuh Rule: 550
Level: 7
MITRE ATT&CK: T1565.001 - Stored Data Manipulation
```

## Triage

Identify:

- monitored file
- modification timestamp
- before/after hashes
- content difference
- agent reporting the event

## Validation

Confirm that the file was modified during the test.

Review:

- file metadata
- current hash
- Wazuh `syscheck.diff`
- recorded before/after values

## Classification

```text
True Positive — Expected
```

## Analyst Conclusion

The SIEM captured both the integrity change and useful forensic context,
including hashes and file-content differences.

---

# 9. Investigation Example — SQL-Injection-Style Request

## Alert

```text
Wazuh Rule: 31103
Level: 7
MITRE ATT&CK: T1190
```

## Triage

Identify:

- source IP
- destination
- URI
- HTTP method
- status code
- repetition

## Validation

Check:

```bash
grep -n "SELECT" /var/log/apache2/access.log
```

Compare the Wazuh alert to the Apache source log.

## Important Interpretation

An alert for SQL-injection-style syntax does not prove that exploitation
succeeded.

In the lab exercise, Apache returned:

```text
HTTP 404
```

because the requested application path did not exist.

The correct conclusion is:

> Suspicious request detected; successful exploitation not demonstrated.

This distinction is important in SOC analysis.

---

# 10. Investigation Example — Active Directory Account Creation

## Alert

```text
Windows Event ID: 4720
Wazuh Rule:       60109
Level:            8
MITRE ATT&CK:     T1098 - Account Manipulation
```

## Triage Questions

- Who created the account?
- What account was created?
- When was it created?
- Was the administrator expected to perform the action?
- Was the account added to privileged groups?
- Did the account log on?
- Were other accounts modified around the same time?

## Validate on the Domain Controller

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id      = 4720
} -MaxEvents 20
```

Inspect the account:

```powershell
Get-ADUser soc-lab-user -Properties *
```

## Scope Related Events

Look for:

```text
4720 — user created
4725 — user disabled
4726 — user deleted
```

Also review nearby:

- logon activity
- PowerShell execution
- process creation
- group-membership changes

## Classification

For the controlled lab:

```text
True Positive — Expected
```

## Analyst Conclusion

The administrative action was recorded by Windows Security logging, collected
by Wazuh, correlated to rule `60109`, and mapped to an identity-related ATT&CK
technique.

---

# 11. Containment and Remediation Decision

Containment should follow evidence.

Possible actions include:

## Account Activity

- Disable the affected account.
- Reset credentials.
- Review group memberships.
- Revoke sessions where appropriate.
- Investigate related authentication.

## Endpoint Activity

- Isolate the endpoint.
- Stop a malicious process.
- Disable a service.
- Remove persistence.
- Restore modified configuration.

## Network Activity

- Block a source IP.
- Restrict a port or service.
- Apply firewall policy.
- Segment the affected system.

In the homelab, containment actions are performed only when they support the
exercise and can be safely reversed.

---

# 12. Verification After Remediation

A remediation is not complete until the environment is rechecked.

Verification may include:

- Confirm account disabled or removed.
- Confirm service state.
- Confirm file restored.
- Confirm process no longer running.
- Confirm suspicious connections stopped.
- Confirm endpoint still reports telemetry.
- Confirm Wazuh agent is active.
- Confirm normal application functionality.
- Confirm no related alerts continue unexpectedly.

Example Linux service check:

```bash
systemctl is-active wazuh-agent
```

Example AD check:

```powershell
Get-ADUser soc-lab-user |
Select-Object Name, SamAccountName, Enabled
```

---

# 13. Investigation Notes Template

Use the following format for future labs:

```text
Alert:
Rule / Event ID:
Severity:
MITRE ATT&CK:
Timestamp:

Affected Host:
Affected User:
Source IP:
Destination:
Process / Command:

Initial Observation:

Source-Log Validation:

Related Events:

Classification:
[ ] True Positive — Suspicious
[ ] True Positive — Expected
[ ] False Positive
[ ] Needs More Context

Impact:

Containment / Remediation:

Verification:

Final Assessment:

Evidence:
```

---

# 14. Escalation Criteria

In a real SOC environment, I would escalate when:

- privileged account compromise is suspected;
- successful unauthorized access is confirmed;
- multiple systems appear affected;
- malware or persistence is suspected;
- data access or exfiltration may have occurred;
- destructive activity is observed;
- investigation requires permissions beyond analyst scope;
- the event affects a critical production system;
- evidence is incomplete but risk is high.

The purpose of escalation is not to abandon the investigation. It is to provide
the next analyst or responder with the evidence already collected.

---

# 15. Evidence Preservation

Useful evidence includes:

- SIEM alert details
- source log excerpts
- timestamps
- hostnames
- usernames
- source/destination IPs
- process paths
- command lines
- event IDs
- rule IDs
- file hashes
- screenshots
- configuration snippets
- remediation commands
- post-remediation validation

Sensitive values such as passwords, tokens, and private credentials should not
be stored in the public repository.

---

# 16. Skills Demonstrated

This playbook reflects practical experience with:

## Security Operations

- Alert triage
- Event validation
- True-positive / false-positive classification
- Severity assessment
- Event correlation
- Threat hunting
- MITRE ATT&CK interpretation
- Endpoint log validation
- Investigation scoping
- Containment decision-making
- Remediation verification

## Windows / Identity

- Windows Security Event analysis
- Active Directory account events
- PowerShell event queries
- User lifecycle investigation
- Identity-security monitoring

## Linux

- SSH authentication logs
- systemd journal analysis
- Apache access logs
- Auditd
- File Integrity Monitoring
- process and service validation

## Analyst Operations

- Evidence collection
- Root-cause analysis
- Clear investigation notes
- Escalation readiness
- Post-remediation validation
- Technical documentation

---

# 17. Related Repository Evidence

| Investigation Area | Documentation |
|---|---|
| Wazuh deployment | [`06-wazuh-siem.md`](06-wazuh-siem.md) |
| Windows telemetry | [`08-windows-telemetry.md`](08-windows-telemetry.md) |
| Attack / detection labs | [`09-attack-detection-labs.md`](09-attack-detection-labs.md) |
| Active Directory monitoring | [`10-active-directory-lab.md`](10-active-directory-lab.md) |
| Skills / evidence matrix | [`11-soc-noc-skills-matrix.md`](11-soc-noc-skills-matrix.md) |
| Detection screenshots | [`../evidence/wazuh/`](../evidence/wazuh/) |

---

# 18. Summary

The analyst workflow used throughout the lab is:

```text
Alert
  ↓
Validate
  ↓
Correlate
  ↓
Classify
  ↓
Scope
  ↓
Prioritize
  ↓
Contain / remediate
  ↓
Verify
  ↓
Document
```

The objective is to make each investigation reproducible and defensible rather
than relying only on the fact that a SIEM generated an alert.
