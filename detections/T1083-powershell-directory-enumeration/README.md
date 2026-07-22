# T1083 -- File and Directory Discovery

| | |
|---|---|
| **MITRE ATT&CK** | [T1083](https://attack.mitre.org/techniques/T1083/) -- File and Directory Discovery |
| **Tactic** | Discovery |
| **Atomic Test** | T1083-5 -- Simulating MAZE Directory Enumeration |
| **Status** | ✅ Validated -- Tuned |
| **Sigma Rule** | [`sigma/T1083_powershell_directory_enumeration_scriptblock.yml`](sigma/T1083_powershell_directory_enumeration_scriptblock.yml) |
| **Sigma CLI Setup** | [`docs/07-sigma-cli-setup.md`](../../docs/07-sigma-cli-setup.md) (one-time environment setup, referenced not repeated) |
| **Log Sources** | PowerShell Operational (4104/4103), Windows Security (EventCode 4688), Sysmon (EventID 1) |
| **Part of Campaign** | Stage 2 (Discovery), following on from [T1059.001](../T1059.001-encoded-powershell-via-cmd/README.md) |

## Contents

Files are numbered in the order you should read them -- GitHub lists them in that same order.
The **Layer** column shows which part of the technique each artifact belongs to.

| # | File | Layer | Purpose |
|---|---|---|---|
| 1 | [`01-hypothesis.md`](01-hypothesis.md) | 🎯 Threat / Objective | Why this technique, what we expect to see, what "done" looks like |
| 2 | [`02-atomic-test.md`](02-atomic-test.md) | ⚔️ Attack Execution | Atomic Red Team test review, prerequisite checks, execution record |
| 3 | [`03-telemetry-validation.md`](03-telemetry-validation.md) | 🔍 Detection (Windows) | Raw Windows Event Log / Sysmon evidence (Event IDs 4104, 4103, 4688, and Sysmon EventID 1) |
| 4 | [`04-splunk-validation.md`](04-splunk-validation.md) | 🔍 Detection (Splunk) | SPL hunt queries used to confirm ingestion, with evidence |
| 5 | [`05-evidence.md`](05-evidence.md) | 🛡️ Sigma / Rule | Sigma rule design, Sigma → SPL conversion, tuning, final verification |
| -- | [`sigma/`](sigma/) | 🛡️ Sigma / Rule | The Sigma detection rule itself (platform-independent) |
| -- | [`spl/`](spl/) | 🔍 Detection (Splunk) | Generated (raw) and environment-tuned SPL, as standalone files |
| -- | [`screenshots/`](screenshots/) | ⚔️ / 🔍 Evidence | Technique-specific evidence screenshots (01--14), in write-up order. General Sigma CLI installation evidence lives in [`docs/screenshots/sigma-cli-setup/`](../../docs/screenshots/sigma-cli-setup/) instead, since that's environment setup, not evidence for this technique. |

## Summary

The MAZE-style directory enumeration test drives a PowerShell script that walks a fixed list
of user and system folders with `Get-ChildItem`, swallowing errors and appending everything it
finds to a temp file. That activity shows up cleanly across the PowerShell Operational log,
Windows Security, and Sysmon, and ingests correctly into the `powershell`, `wineventlog`, and
`sysmon` Splunk indexes. Rather than detecting on `Get-ChildItem` alone -- far too common a
cmdlet to be useful by itself -- the Sigma rule looks for the specific combination of a `foreach`
loop, silent error handling, and appended `Out-File` output inside a single PowerShell script
block. That combination was converted from Sigma to SPL using the official Sigma CLI, tuned for
this lab's index naming, and validated end-to-end against the Atomic Red Team simulation.

See [`05-evidence.md`](05-evidence.md) for the full detection engineering narrative, or jump straight to
the final tuned query: [`spl/tuned_splunk.spl`](spl/tuned_splunk.spl).
