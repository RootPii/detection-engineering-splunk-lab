# Threat Hypothesis -- T1083 (File and Directory Discovery)

**MITRE ATT&CK:** T1083 -- File and Directory Discovery
**Stage:** Discovery
**Atomic Test:** T1083-5 -- Simulating MAZE Directory Enumeration

## Objective

Validate that command-line-driven file and directory enumeration generates the expected
endpoint telemetry, and that it can be reliably detected using Windows Event Logs, Sysmon,
and Splunk.

## Real-World Context

Once an adversary has code execution on a host, one of the first things they usually do is
look around. File and directory discovery is how that looks in practice: walking user
folders, checking installed software, hunting for configuration files, scripts, or anything
left lying around that hints at credentials. None of this requires dropping new malware --
built-in Windows utilities and PowerShell cmdlets are more than enough, and that's exactly
why it's easy to miss. The output of this stage regularly feeds directly into the next one:
privilege escalation, credential access, persistence, or staging data for exfiltration.

This is deliberately written in general terms rather than around a specific threat actor,
because the goal here is the detection engineering, not the attribution.

The specific atomic test used in this write-up (T1083-5) simulates the directory-walking
behavior associated with MAZE ransomware intrusions -- a script that loops over common user
and system folders and records what it finds. It's a good stand-in for the broader technique
because the pattern (loop + enumerate + silently continue on errors + append the output
somewhere) is common to a lot of real discovery scripts, not just this one strain.

## Detection Objectives

For this technique we want to answer four questions:

1. **Did Windows generate the expected telemetry?**
   Expected log sources: PowerShell Operational Log, Security Log, Sysmon.

2. **Did Splunk ingest the telemetry correctly?**
   Expected indexes: `powershell`, `sysmon`, `wineventlog`.

3. **Can we identify the execution?**
   Evidence should include: Process Image, Parent Process, Command Line, User, Host,
   Timestamp, and the PowerShell Script Block content itself.

4. **Can we create a reliable detection?**
   Deliverables: SPL Hunt, Sigma Rule, Detection Logic, Validation, Tuning Notes.

## Data Sources

The detection is validated against:

- Sysmon Event ID 1 (Process Creation)
- Windows Security Event ID 4688 (Process Creation)
- PowerShell Operational Logs -- Event ID 4104 (Script Block Logging) and 4103 (Module Logging)
- Splunk, for confirming ingestion into the correct indexes

## Workflow

This technique follows the standard nine-step detection engineering workflow used across
every technique in this lab (see [`docs/00-methodology.md`](../../docs/00-methodology.md)):

`Atomic Test → Observe Telemetry → Ingest & Validate → Investigate → SPL Hunt → Sigma Rule → Test & Tune → Document → Commit`

Answers to each objective, and the corresponding evidence, are recorded in
[`03-telemetry-validation.md`](03-telemetry-validation.md), [`04-splunk-validation.md`](04-splunk-validation.md),
and [`05-evidence.md`](05-evidence.md).
