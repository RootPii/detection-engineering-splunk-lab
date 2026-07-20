# Windows Telemetry Configuration

**Status:** Complete
**Lab:** Splunk Detection Engineering Lab
**Updated:** 2026-07-12

WIN10-01 is set up (`04-windows-setup.md`). This doc covers making it actually generate and forward useful security telemetry: audit policy, PowerShell logging, Sysmon, and the Universal Forwarder — in that order, because there's no point installing a forwarder before there's anything worth forwarding.

## Advanced audit policy

Configured via `secpol.msc → Advanced Audit Policy Configuration`. Deliberately scoped to what detection engineering actually needs instead of turning on every audit category — Object Access, Privilege Use, DS Access, and Global Object Access Auditing were left Not Configured to avoid noise in Phase 1.

![Run dialog, secpol.msc](screenshots/windows-telemetry/01-run-secpol.png)

![Advanced Audit Policy Configuration overview, all categories Not Configured](screenshots/windows-telemetry/02-audit-policy-overview-not-configured.png)

| Category | Subcategory | Setting |
|---|---|---|
| Detailed Tracking | Audit Process Creation | Success |
| Logon/Logoff | Audit Logon | Success and Failure |
| Logon/Logoff | Audit Special Logon | Success |
| Account Logon | Audit Credential Validation | Success and Failure |
| Policy Change | Audit Audit Policy Change | Success |
| System | Audit Security State Change | Success |
| System | Audit Security System Extension | Success |
| System | Audit System Integrity | Success and Failure |

![Audit Process Creation Properties, Success](screenshots/windows-telemetry/03-audit-process-creation.png)

![Audit Logon Properties](screenshots/windows-telemetry/04-audit-logon.png)

![Logon/Logoff category confirmed — Audit Logon and Audit Special Logon both set](screenshots/windows-telemetry/05-audit-logon-and-special-logon-confirmed.png)

![Audit Credential Validation Properties, Success and Failure](screenshots/windows-telemetry/06-audit-credential-validation.png)

![Audit Audit Policy Change Properties, Success](screenshots/windows-telemetry/07-audit-policy-change.png)

![System category, before configuration — all Not Configured](screenshots/windows-telemetry/08-system-category-not-configured.png)

![Audit Security State Change Properties, Success](screenshots/windows-telemetry/09-audit-security-state-change.png)

![Audit Security System Extension Properties, Success](screenshots/windows-telemetry/10-audit-security-system-extension.png)

![System category confirmed — State Change, System Extension, and System Integrity all set](screenshots/windows-telemetry/11-system-category-confirmed.png)

This generates 4688 (process creation), 4624/4625 (logon success/failure), 4672 (special logon).

Ran a verification check with `auditpol /get /category:*` right after `gpupdate /force` — but that check happened *before* the System category subcategories above were finished in the GUI, so its output only reflects partial state (it shows System Integrity as "No Auditing" at that point in time). Not a real gap, just a sequencing mismatch between when the check ran and when configuration finished. Worth re-running `auditpol` once everything's applied and swapping in a final, fully-consistent output.

![Terminal — gpupdate /force and auditpol /get /category:* partial-state output](screenshots/windows-telemetry/12-gpupdate-auditpol-verification.png)

## Command-line logging for 4688

Event 4688 doesn't include the full command line by default — without this, Sigma rules matching on command-line content have nothing to match against.

```
gpedit.msc → Computer Configuration → Administrative Templates → System
→ Audit Process Creation → Include command line in process creation events
→ Enabled
```

![Include command line in process creation events, Enabled](screenshots/windows-telemetry/13-gpedit-include-commandline.png)

## PowerShell logging

```
gpedit.msc → Computer Configuration → Administrative Templates
→ Windows Components → Windows PowerShell
```

**Module Logging** — Enabled, module names `*` (logs execution across all modules, not a hand-picked subset). Generates Event 4103.

![Turn on Module Logging, Enabled](screenshots/windows-telemetry/14-gpedit-module-logging.png)

![Module Names list, wildcard *](screenshots/windows-telemetry/15-gpedit-module-logging-wildcard.png)

**Script Block Logging** — Enabled. Generates Event 4104 — this is the one that actually captures script content, which is what most PowerShell-based detections (encoded commands, download cradles, obfuscation) depend on.

![Turn on PowerShell Script Block Logging, Enabled](screenshots/windows-telemetry/16-gpedit-scriptblock-logging.png)

PowerShell Transcription left disabled on purpose — this project is built around Windows Event Logs, not transcript text files on disk.

## Apply and verify

```
gpupdate /force
```
```
Computer Policy update has completed successfully.
User Policy update has completed successfully.
```

![Terminal — gpupdate /force and gpresult /r](screenshots/windows-telemetry/17-gpupdate-gpresult.png)

**Process creation (4688):** ran `notepad.exe`, confirmed the event in Event Viewer — `New Process Name: C:\Windows\System32\notepad.exe`, full command line present.

![Event Viewer, Event 4688 — notepad.exe process creation detail](screenshots/windows-telemetry/18-eventviewer-4688-notepad.png)

**PowerShell (4103/4104):** ran `Get-Process | Select-Object -First 5`, confirmed both event IDs generated in the PowerShell Operational log.

Every native telemetry source needed before touching Sysmon or the forwarder was verified independently, rather than assuming Group Policy applied correctly just because `gpupdate` reported success. That verification step is what makes the rest of this document trustworthy: Sysmon and the Universal Forwarder are only worth installing once Windows itself is actually generating the telemetry they depend on.

---

# Sysmon

Used Olaf Hartong's [sysmon-modular](https://github.com/olafhartong/sysmon-modular) pre-built config instead of Microsoft's default or SwiftOnSecurity's — ATT&CK-tagged, better maintained, and the goal here is detection engineering, not Sysmon config authorship.

**Sysmon itself (Microsoft Sysinternals):** [learn.microsoft.com/en-us/sysinternals/downloads/sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)

## Download and layout

Used a dedicated lab directory instead of dumping everything in Downloads:

```powershell
New-Item -ItemType Directory C:\Lab\Sysmon
```

![Downloads folder — sysmonconfig and Sysmon zip](screenshots/windows-telemetry/19-sysmon-downloads-folder.png)

![Extract Compressed (Zipped) Folders, destination C:\Lab\Sysmon](screenshots/windows-telemetry/20-sysmon-extract-to-clab.png)

```
C:\Lab\Sysmon
├── Eula.txt
├── Sysmon.exe
├── Sysmon64.exe
├── Sysmon64a.exe
└── sysmonconfig.xml
```

![C:\Lab\Sysmon folder contents](screenshots/windows-telemetry/21-sysmon-folder-contents.png)

`sysmonconfig.xml` pulled directly from the sysmon-modular repo. Verified it was the right file before installing — it opens with `<Sysmon schemaVersion="4.90">`, matching the repo's current build.

![PowerShell dir confirming Sysmon folder layout](screenshots/windows-telemetry/22-sysmon-dir-confirmation.png)

## Install

```powershell
cd C:\Lab\Sysmon
.\Sysmon64.exe -accepteula -i .\sysmonconfig.xml
```

```
Loading configuration file with schema version 4.90
Sysmon schema version: 4.91
Configuration file validated.
SysmonDrv installed.
Starting SysmonDrv.
SysmonDrv started.
Sysmon64 started.
```

Config hash confirmed via `Sysmon64.exe -c`, later re-verified directly against the deployed file (an earlier transcription of this hash from a terminal screenshot had a few misread characters — corrected below):
```
Config file:  c:\Lab\Sysmon\sysmonconfig.xml
Config hash:  SHA256=4516404FA30EE87CEA558567820CDC78863CC4AB07889519E49EAC3CCA92E0D2
```

![Sysmon install output and config hash](screenshots/windows-telemetry/23-sysmon-install-and-hash.png)

## Verify

```powershell
Get-Service Sysmon64
Status: Running   Name: Sysmon64

sc.exe query Sysmon64
STATE: 4 RUNNING
```

![sc.exe query and Get-Service confirming Sysmon64 Running](screenshots/windows-telemetry/24-sysmon-service-running.png)

Event Viewer → Applications and Services Logs → Microsoft → Windows → Sysmon → Operational — events generating: Process Create (1), Image Load (7), Process Access (10), File Create (11), Registry Value Set (13), DNS Query (22).

## Log size

Increased Sysmon Operational log from default to 1 GB, same rationale as the Windows-native logs in `04-windows-setup.md` — default rolls over too fast during repeated Atomic testing.

---

# Splunk Universal Forwarder

## Install

Splunk Universal Forwarder 10.4.1, Windows x64 MSI, installed via "Customize Options" rather than the default path — needed the manual control over data collection, not the installer's auto-generated inputs.conf.

**Download:** [splunk.com/en_us/download/universal-forwarder.html](https://www.splunk.com/en_us/download/universal-forwarder.html)

| Setting | Choice | Reason |
|---|---|---|
| Service account | Virtual Account | Splunk's recommended modern option — gives the forwarder what it needs without the broader access Local System carries |
| SSL certificate | Left blank | TLS on the forwarder-to-indexer channel is planned as manual config later, not installer defaults |
| Windows Event Logs (installer checkboxes) | All left unchecked | Installer can auto-generate `inputs.conf`, but this lab deliberately uses a hand-built one instead — see `03-splunk-installation.md` design decisions |
| Deployment Server | Left blank | No deployment server in this lab — manual config only |
| Receiving Indexer | `192.168.56.10 : 9997` | Points at SOC01 |

![UniversalForwarder Setup — service account, Virtual Account selected](screenshots/windows-telemetry/25-uf-virtual-account.png)

![UniversalForwarder Setup — Receiving Indexer, 192.168.56.10:9997](screenshots/windows-telemetry/26-uf-receiving-indexer-ip-port.png)

Windows privileges granted during install: `SeBackupPrivilege`, `SeSecurityPrivilege`, Performance Monitor Users. These let the forwarder read Security logs and protected files — this list matters, see the troubleshooting case study below, because it's *not* actually sufficient on its own.

## Verify

```powershell
Get-Service SplunkForwarder
Running

& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" status
SplunkForwarder: Running
```

`list forward-server` — checked twice, a few seconds apart:

First check:
```
Configured but inactive forwards:
        192.168.56.10:9997
```

Second check:
```
Active forwards:
        192.168.56.10:9997
```

Just a TCP session establishment delay, not an error — the UF hadn't finished connecting when first checked.

Also confirmed the port is actually reachable from WIN10-01, not just configured:
```powershell
Test-NetConnection 192.168.56.10 -Port 9997
TcpTestSucceeded : True
```

![Test-NetConnection succeeded, list forward-server showing Active forwards](screenshots/windows-telemetry/29-uf-testnetconnection-forward-server.png)

**Note on the Forwarder Management console:** SOC01's web UI (Settings → Forwarder Management → Agent management) showed "No forwarders to display" even after the connection was active. That page tracks deployment-server-managed forwarders specifically — since this lab intentionally has no deployment server, an empty list there is expected and doesn't mean the forwarder isn't sending data. Worth noting so it doesn't look like a red flag to someone reading this later.

`outputs.conf` generated by the installer, matches what was expected — no manual edits needed:
```ini
[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = 192.168.56.10:9997

[tcpout-server://192.168.56.10:9997]
```

## Deploying inputs.conf

The hand-built `inputs.conf` (documented separately, see `configs/splunk/inputs.conf` and the design-decisions writeup in `03-splunk-installation.md`) was placed at:

```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```

![etc\system\local — authentication.conf, outputs.conf, server.conf](screenshots/windows-telemetry/27-uf-etc-system-local-files.png)

Forwarder restarted, monitored inputs confirmed with `splunk.exe list monitor` and `splunk.exe list eventlog`.

---

# Troubleshooting case study — Sysmon events not reaching Splunk

First full validation check, `index=* | stats count by index`, showed only `wineventlog` and `powershell` — zero events in `sysmon`, despite Sysmon clearly running and generating events locally (`Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational"` returned results fine).

![index=* | stats count by index — only wineventlog and powershell, sysmon missing](screenshots/windows-telemetry/30-stats-count-by-index-problem.png)

![index=sysmon | head 5 — 0 events](screenshots/windows-telemetry/31-index-sysmon-zero-events.png)

![index=wineventlog | head 5 — 5 events, proving the pipeline itself works](screenshots/windows-telemetry/32-index-wineventlog-5-events.png)

![index=powershell | head 5 — 5 events, same confirmation](screenshots/windows-telemetry/33-index-powershell-5-events.png)

Also tried hunting for the Sysmon data under different field names, in case it was landing somewhere unexpected — no luck either way:

![sourcetype=XmlWinEventLog:Microsoft-Windows-Sysmon/Operational — 0 events](screenshots/windows-telemetry/34-sourcetype-search-zero-events.png)

![source=WinEventLog:Microsoft-Windows-Sysmon/Operational — 0 events](screenshots/windows-telemetry/35-source-search-zero-events.png)

Walked it front to back instead of guessing:

**1. Confirm the index exists.**
```
./splunk list index
```
`sysmon` was present — not an index problem.

**2. Confirm the UF is actually configured to monitor the Sysmon channel.**
```
splunk.exe list eventlog
```
```
Microsoft-Windows-Sysmon/Operational
    disabled = 0
    index = sysmon
    renderXml = true
```
Config looked correct on paper.

**3. Confirm that config is the one actually loaded, not overridden by another app or a stale default.**
```
splunk.exe btool inputs list WinEventLog://Microsoft-Windows-Sysmon/Operational --debug
```
Confirmed it was loading from `etc/system/local/inputs.conf` — the file I actually edited, not shadowed by anything else.

**4. Confirm Sysmon itself is fine, independent of Splunk entirely.**
```
Get-WinEvent -ListLog *Sysmon*
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 20
```
Events 1, 3, 10, 11, 13, 22 all present. Sysmon side was never the problem.

**5. Check the forwarder's own log instead of assuming.**
```powershell
Select-String -Path "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" `
  -Pattern "Sysmon","subscribeToEvtChannel","errorCode","WinEventLogChannel"
```
This was the actual answer:
```
WinEventLogChannel::subscribeToEvtChannel
Could not subscribe to Microsoft-Windows-Sysmon/Operational
errorCode=5
```
Windows error 5 = Access Denied.

**Root cause:** the Universal Forwarder's Virtual Account (`NT SERVICE\SplunkForwarder`) didn't have permission to subscribe to the Sysmon Operational channel specifically. The three privileges granted during install (`SeBackupPrivilege`, `SeSecurityPrivilege`, Performance Monitor Users) cover the Security log and file access, but Sysmon's channel needs separate membership in **Event Log Readers** — the installer's default privilege set doesn't include this by default for non-Security channels.

**Fix:**
```powershell
net localgroup "Event Log Readers" "NT SERVICE\SplunkForwarder" /add
Restart-Service SplunkForwarder
```

![Restart-Service SplunkForwarder](screenshots/windows-telemetry/28-uf-restart-service.png)

**Confirmed fixed** — re-checked `splunkd.log` for `errorCode=5` and only the original entry from before the fix remained, nothing new after the restart.

**Validated with fresh telemetry, not just historical data** — ran `notepad.exe` and `cmd.exe` to generate new Sysmon events post-fix, then confirmed in Splunk:
```
index=sysmon earliest=-15m
118 events
```

![index=sysmon earliest=-15m — 118 events, fix confirmed](screenshots/windows-telemetry/36-index-sysmon-118-events-fixed.png)

Including Event ID 13 (Registry Value Set), `source = WinEventLog:Microsoft-Windows-Sysmon/Operational`, `sourcetype = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational`.

## Lessons learned

1. Verify the data source first — confirm Windows itself has the events before assuming Splunk is broken.
2. `splunk.exe list eventlog` and `btool` confirm the forwarder is actually monitoring the right channel with the config you think is loaded — don't assume the UI reflects reality.
3. `splunkd.log` should be an early stop, not a last resort. `errorCode=5` pointed straight at a permissions issue instead of a config or indexing problem.
4. The Virtual Account's default privileges are not automatically sufficient for every event channel — Sysmon specifically needed explicit Event Log Readers membership.
5. Validate with fresh events, not just historical ones still sitting in the log — that's the only way to confirm the fix actually applies going forward, not just that old cached data happens to be visible.

---

## Closing state

By the end of this phase, WIN10-01 has native Windows telemetry (audit policy, command-line logging, PowerShell logging), Sysmon running with the Olaf Hartong modular config, and the Universal Forwarder actively sending data to SOC01 — all three indexes (`sysmon`, `wineventlog`, `powershell`) confirmed receiving events after the Event Log Readers fix above.

The forwarder-to-indexer channel on port 9997 is still plaintext. TLS is planned as a manual configuration step rather than something handled by installer defaults, and it stays an open item rather than something to gloss over — a real SOC deployment wouldn't forward Security and PowerShell telemetry unencrypted even on an internal segment, and this lab shouldn't pretend otherwise until it's actually done.

With telemetry confirmed live across all three indexes, the next phase is Atomic Red Team testing (`06-atomic-red-team.md`) and detection validation against real data instead of assumptions.
