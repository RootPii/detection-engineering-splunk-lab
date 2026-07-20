# Splunk Add-on for Microsoft Windows (TA-Windows)

**Status:** Complete
**Lab:** Splunk Detection Engineering Lab
**Updated:** 2026-07-18

While rehearsing the atomic test loop in `06-atomic-red-team.md`, filtering Splunk searches on specific field names came up empty — no `EventCode`, `Image`, or `CommandLine` to search on, just free text against raw event XML. That's not sustainable across dozens of future detections, so this install happens now, before Phase 5 detection work starts.

## Why this add-on, and why now

Most real Splunk environments run the **Splunk Add-on for Microsoft Windows (TA-Windows)** to normalize Windows event data and map it to Splunk's Common Information Model (CIM) — the same field names (`EventCode`, `Image`, `CommandLine`, `ParentImage`, `User`, `Computer`, `ProcessGuid`) that most community Sigma rules already assume exist.

Installing it now, before any real detection is written, means every future SPL and Sigma rule in this lab can rely on those fields being there — instead of retrofitting field extractions into rules that already work.

## Where it needs to be installed

```
WIN10-01 → Universal Forwarder → SOC01 (Splunk Enterprise, all-in-one)
```

SOC01 runs Splunk Enterprise as a single instance — search head and indexer combined. That means TA-Windows only needs to go on SOC01. The Universal Forwarder on WIN10-01 doesn't need it; it just ships raw data, it doesn't parse anything.

## Step 1: Download

Downloaded the latest **Splunk Add-on for Microsoft Windows** from Splunkbase (version 10.1.1 at the time of this install).

**Download:** [splunkbase.splunk.com/app/742](https://splunkbase.splunk.com/app/742)

![Splunkbase — Splunk Add-on for Microsoft Windows](screenshots/ta-windows/01-splunkbase-download-page.png)

## Step 2: Install on SOC01

**Navigation:**
```
Settings → Apps → Manage Apps → Install app from file
```

Uploaded `splunk-add-on-for-microsoft-windows_1011.spl` directly through the web UI.

![Installing the app from file](screenshots/ta-windows/02-install-app-from-file.png)

## Step 3: Restart Splunk

```bash
sudo /opt/splunk/bin/splunk restart
```

![Splunk restart output](screenshots/ta-windows/03-splunk-restart-output.png)

## Step 4: Verify the normalized fields actually work

```spl
index=wineventlog
| stats count by EventCode
```

![EventCode field working across wineventlog](screenshots/ta-windows/04-verify-eventcode-stats.png)

```spl
index=powershell
| stats count by EventID
```

![EventID field working across powershell, including 4103/4104](screenshots/ta-windows/05-verify-powershell-eventid-stats.png)

Both queries return clean, structured breakdowns instead of raw XML dumps — including the 4103/4104 script-block events and the standard logon/process codes (4624, 4672, 4688, and others). This confirms the add-on is doing its job: turning Windows event XML into fields Splunk, and every future Sigma rule, can search on directly.

## Where this leaves things

TA-Windows is installed and verified on SOC01. Every future SPL search and Sigma rule in this lab can now use standard CIM-style field names instead of writing regex against raw event XML — which matters because most community Sigma rules already assume these fields exist. This closes the field-normalization gap surfaced during the Atomic Red Team rehearsal in `06-atomic-red-team.md`, and Phase 5 detection work (`detections/T1059/investigation.md` onward) starts from here.
