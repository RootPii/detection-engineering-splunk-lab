# Atomic Red Team Installation

**Status:** Complete
**Lab:** Splunk Detection Engineering Lab
**Updated:** 2026-07-18

Getting WIN10-01 ready to run repeatable ATT&CK simulations. Everything after this — SPL, Sigma rules, coverage tracking — depends on being able to trigger a known technique on demand, so this needs to actually work before any detection work starts.

**Atomic tests (the technique library):** [github.com/redcanaryco/atomic-red-team](https://github.com/redcanaryco/atomic-red-team)
**Invoke-AtomicRedTeam (the execution framework used here):** [github.com/redcanaryco/invoke-atomicredteam](https://github.com/redcanaryco/invoke-atomicredteam)

These are two separate repos working together: `atomic-red-team` holds the actual technique definitions (the "what to run"), and `invoke-atomicredteam` is the PowerShell module that runs them (the "how to run it"). Installing with `-getAtomics` pulls both at once.

## Step 1: Check prerequisites

```powershell
Get-ExecutionPolicy
```
```
RemoteSigned
```

```powershell
$PSVersionTable.PSVersion
```
```
5.1
```

`Get-Module -ListAvailable PowerShellGet` confirmed present before going any further.

## Step 2: First install attempt — blocked by Defender

```powershell
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)
Install-AtomicRedTeam -getAtomics
```

This failed partway through:
```
Operation did not complete successfully because the file contains a virus or potentially unwanted software.
```

![Defender blocking the install](screenshots/atomic-red-team/01-defender-virus-block-error.png)

**Why this happens:** several atomic test files intentionally mimic real malware patterns, so Windows Defender flags and quarantines them on sight during download. This isn't a broken install — it's Defender doing exactly what it's designed to do. The `Invoke-AtomicRedTeam` module itself installed fine; only the atomics repo download got interrupted.

## Step 3: Fix — dedicated folder with a Defender exclusion

```powershell
New-Item -ItemType Directory -Force -Path C:\AtomicRedTeam
Add-MpPreference -ExclusionPath "C:\AtomicRedTeam"
Get-MpPreference | Select-Object -ExpandProperty ExclusionPath
```
```
C:\AtomicRedTeam
C:\Program Files\AccessData\FTK Imager
```

![Defender exclusion added](screenshots/atomic-red-team/02-defender-exclusion-added.png)

(The FTK Imager line was already there from before this session — not something added here, just confirming the new exclusion sits alongside it correctly.)

Reinstalled with `-Force`, since the previous attempt had already partially registered:

```powershell
Install-AtomicRedTeam -getAtomics -Force
```
```
Installation of Invoke-AtomicRedTeam is complete. You can now use the Invoke-AtomicTest function.
```

## Step 4: Verify the install actually worked

```powershell
Get-ChildItem C:\AtomicRedTeam
```
```
atomics
invoke-atomicredteam
```

Hundreds of ATT&CK technique folders confirmed under `atomics` (T1001.002, T1003 and its sub-techniques, T1005 through T1027, and beyond).

```powershell
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
Get-Command Invoke-AtomicTest
Invoke-AtomicTest T1059.001 -ShowDetailsBrief
```

![Module imported, command available, 22 sub-tests listed for T1059.001](screenshots/atomic-red-team/03-install-verified-module-ready.png)

22 sub-tests returned for T1059.001 alone — confirms the module isn't just installed, it's actually ready to run tests.

## How testing will run from here

Instead of installing everything and running a pile of tests before writing any detections, each technique goes through the same loop, start to finish, before moving to the next one:

![Atomic Red Team execution and validation workflow](screenshots/atomic-red-team/workflow-diagram.png)

A quick note on this diagram: it's something I put together myself to keep the loop visible while working through techniques — it isn't from Red Canary's official documentation. In short: prep the environment once, then for every technique — select it, check details, check prerequisites (install any that are missing), run the test, confirm it shows up in Windows telemetry and in Splunk, investigate what was generated, clean up, and confirm the cleanup actually worked before moving to the next technique.

If you want the official, more detailed version of this workflow straight from the source, it's on the Invoke-AtomicRedTeam wiki: [github.com/redcanaryco/invoke-atomicredteam/wiki](https://github.com/redcanaryco/invoke-atomicredteam/wiki)

Slower per technique this way, but every rule that lands in `sigma/` will have a real investigation behind it, not a guess made from memory days later.

## Rehearsal: T1059.001, Test #17

Before trusting the loop above for real technique work, it needed to be proven end to end at least once: run an atomic on WIN10-01, confirm Windows logged it, and confirm it actually made it across the wire to Splunk on SOC01. This section is that rehearsal — not the real T1059 investigation, which belongs in `detections/T1059/investigation.md` once Phase 5 starts.

**Technique used:** T1059.001 (Command and Scripting Interpreter: PowerShell), Atomic Test #17 — "PowerShell Command Execution." It runs an obfuscated, base64-encoded PowerShell command that just decodes to `Hello, from PowerShell!` — deliberately low-risk, so the first end-to-end run wouldn't be complicated by a noisier technique.

### Session setup, test details, prerequisites, and execution

```powershell
powershell.exe -ExecutionPolicy Bypass
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
Get-Command Invoke-AtomicTest

Invoke-AtomicTest T1059.001 -TestNumbers 17 -ShowDetails
Invoke-AtomicTest T1059.001 -TestNumbers 17 -CheckPrereqs
Invoke-AtomicTest T1059.001 -TestNumbers 17
```
```
Prerequisites met: T1059.001-17 PowerShell Command Execution
...
Hello, from PowerShell!
Exit code: 0
```

![Full test sequence: details, prereq check, and execution](screenshots/atomic-red-team/04-test17-full-sequence.png)

A few things worth calling out from this run:

- `-ExecutionPolicy Bypass` only applies to this one PowerShell process — it doesn't touch the machine-wide `RemoteSigned` policy checked in Step 1.
- `-ShowDetails` (not the brief version) surfaced the description, the test GUID, the executor (`command_prompt`, no elevation needed), and the actual obfuscated `powershell.exe -e <base64>` command — all before anything ran.
- Prerequisites were already met, so `-GetPrereqs` wasn't needed. The order to keep for every future technique: `-ShowDetails` → `-CheckPrereqs` → `-GetPrereqs` only if that check fails → then run it.
- Exit code 0 and the expected string printed — the test ran cleanly.

### Confirm Windows logged it

Checked the PowerShell Operational log in Event Viewer for the matching Event ID 4104 (script block logging) — the run showed up as expected, carrying the decoded `Hello, from PowerShell!` content.

### Confirm Splunk on SOC01 received it

Splunk runs on SOC01, not on WIN10-01 — the Windows box only generates the event and forwards it. So this step checks two things at once: did Windows PowerShell logging capture the run, and did the forwarder actually ship it to the indexer.

```spl
index=powershell earliest=-5m
| search "Hello"
```

![Confirmed in Splunk — 41 matching events](screenshots/atomic-red-team/05-splunk-search-hello-confirmed.png)

41 matching events came back, sourced from `WinEventLog:Microsoft-Windows-PowerShell/Operational`, with `host=WIN10-01` tying them back to this exact test run. Getting a result at all confirms both halves of the pipeline at once.

**One thing worth flagging here:** free-text search like `"Hello"` above works fine, but filtering on a specific normalized field name doesn't — at this point in the lab, Splunk hasn't been taught to parse Windows XML into clean fields yet.

```spl
index=powershell
| stats count by HostApplication
```

![Filtering on a specific field returns nothing pre-TA-Windows](screenshots/atomic-red-team/06-splunk-specific-field-no-results.png)

If you hit this same "no results" wall while filtering on fields like `EventCode`, `Image`, or `CommandLine`, that's expected at this stage — it gets fixed in `08-splunk-ta-windows.md`, which installs the add-on that normalizes these fields across every index.

A couple of other queries worth keeping on hand for future tests that aren't as easy to spot as "Hello":

```spl
index=powershell earliest=-5m | search 4104
index=sysmon earliest=-5m Image="*powershell.exe" | table _time Image CommandLine ParentImage User
index=wineventlog earliest=-5m EventCode=4688 NewProcessName="*powershell.exe"
```

Neither of the last two was needed for this rehearsal — the PowerShell log alone was enough — but the real T1059 investigation later will need the process-creation view too, not just the script-block view.

### Clean up

```powershell
Invoke-AtomicTest T1059.001 -TestNumbers 17 -Cleanup
```
```
Executing cleanup for test: T1059.001-17 PowerShell Command Execution
Done executing cleanup for test: T1059.001-17 PowerShell Command Execution
```

![Cleanup completed](screenshots/atomic-red-team/07-test17-cleanup.png)

Cleanup after every atomic test is the right habit to build now, even for a harmless one like this — it stops leftover state from quietly piling up across dozens of future runs.

### Result

The full pipeline checks out: view test details → confirm prerequisites → execute → confirm the event lands in Splunk → clean up. Atomic Red Team, Windows telemetry, and the forwarder into Splunk are all verified working together. This is a rehearsal of the loop, not the real T1059 investigation — that comes in Phase 5, with raw-event analysis, SPL, a Sigma rule, and false-positive tuning.

## Where this leaves things

Atomic Red Team is installed on WIN10-01 and verified functional. The Defender-quarantine issue is understood well enough not to repeat it on a future reinstall — the fix (dedicated directory + exclusion) should be applied *before* the first install attempt, not after hitting the error again.

One gap surfaced during this rehearsal: raw Windows XML events don't have normalized fields like `EventCode`, `Image`, or `CommandLine` ready to search on yet. That gets fixed by installing the Splunk Add-on for Microsoft Windows (TA-Windows) on SOC01 — covered in `08-splunk-ta-windows.md`, since it's a Splunk-side change that affects every future detection, not something specific to Atomic Red Team.

Phase 5 starts from here, one T-ID folder per technique:

```
detections/
├── T1059/
│   └── investigation.md
├── T1218/
│   └── investigation.md
└── ...
```

`detections/T1059/investigation.md` is next: pick up where this rehearsal left off, do the full raw-event investigation, build the SPL, convert it to a Sigma rule, tune false positives, and log it in `coverage/attack-matrix.md`.
