# INT 4.8 AUX Live Monitor — LAT/BE Business Rules and Engine Specification v4

**Document status:** v4 pre-implementation specification candidate  
**Project:** INT 4.8 AUX Live Monitor  
**Primary artifact:** Full business-rules and implementation-control specification  
**Language:** English  
**Target implementation:** One standalone offline HTML file  
**Purpose:** Define the final LAT/BE interval engine rules before implementation, with all v3 audit gaps resolved or explicitly bounded.

---

# 0. Executive Decision Summary

INT 4.8 already has a frozen AUX snapshot parser and operational display layer. This v4 specification defines the next layer: a LAT/BE interval engine that tracks live time-weighted performance across 30-minute intervals from 3:00 PM to 7:00 AM, with stale protection, end-of-day reset, Auto Req schedule persistence, manual LAT/BE correction, and temporary Manual IN what-if display.

The critical design principle is:

> **Official calculations must remain stable, auditable, and protected from temporary UI experiments.**

Therefore:

- Snapshot paste drives official IN/OUT.
- LAT/BE calculations use official IN/OUT only.
- Manual LAT/BE corrections are explicit user-entered overrides.
- Manual IN +/- is only a temporary what-if view and never enters official history.
- Auto Req schedules must persist locally, because the tool is offline and may ask the user to refresh.
- Stale/frozen/locked states must never look live.

---

# 1. New in v4 vs v3

This section lists what v4 adds, fixes, or clarifies compared with v3.

## 1.1 Auto Req persistence is now technically defined

v3 required Auto Req schedules and configured Req values to survive Refresh / Start Over and 7:15 AM lock/reset, but did not define how this is possible in a single-file offline HTML tool.

v4 defines the persistence mechanism:

- Auto Req schedules must be stored in `localStorage`.
- Configured Req values intended for the next operating window must be stored in `localStorage`.
- On page load, the tool must restore Auto Req schedules and configured Req values from `localStorage`.
- In-memory state alone is not sufficient.

---

## 1.2 Expected LAT% formula is now explicit

v3 described Expected LAT% as a projection but did not provide a computable formula.

v4 defines:

```text
Expected LAT% =
(closedLatWeightedSeconds + currentREQ92PctRaw × remainingSeconds)
/
1800
```

Where:

- `closedLatWeightedSeconds` is the accumulated weighted LAT contribution from elapsed closed segments in the current interval.
- `currentREQ92PctRaw` is the current raw REQ92% value.
- `remainingSeconds` is the number of seconds remaining until the current interval ends.
- `1800` is the full interval length in seconds.

---

## 1.3 Expected BE% formula is now explicit

v3 named Expected BE% but did not fully define it.

v4 defines:

```text
Expected BE% =
(closedBeWeightedSeconds + currentBEPctRaw × remainingSeconds)
/
1800
```

Where:

- `closedBeWeightedSeconds` is the accumulated weighted BE contribution from elapsed closed BE segments in the current interval.
- `currentBEPctRaw` is the current raw BE% value.
- `remainingSeconds` is the number of seconds remaining until the current interval ends.
- `1800` is the full interval length in seconds.

---

## 1.4 Manual LAT/BE blending is now mathematically defined

v3 stated that manual values are highest source of truth, but did not define exactly how they blend with later live samples.

v4 defines:

- Manual LAT% or Manual BE% is the user-entered value.
- It applies from interval start to the Apply timestamp.
- After the Apply timestamp, live tracking resumes for the remaining interval.
- A second manual edit in the same interval replaces the previous manual edit for that metric.
- Manual segments do not stack.

---

## 1.5 Req change replay semantics are now explicit

v3 said Req changes recalculate from interval start but did not define whether past values should be replayed or kept.

v4 defines:

- LAT Req changes retroactively rebuild all non-manual LAT and BE portions of the current interval using the latest LAT Req.
- BE Req changes retroactively rebuild all non-manual BE portions of the current interval using the latest BE Req.
- Manual LAT segments remain locked.
- Manual BE segments remain locked.
- Auto Req schedule values apply from their scheduled interval boundary and continue until the next scheduled Req takes over.

---

## 1.6 Duplicate snapshot behavior is now safer

v3 allowed and calculated duplicate snapshots.

v4 keeps that rule but adds:

- Exact duplicate snapshots are allowed and calculated.
- Exact duplicate snapshots show a Duplicate Snapshot alert.
- Exact duplicate snapshots do **not** reset the stale timer.
- Non-duplicate valid snapshots do reset the stale timer.

This prevents an operator from repeatedly pasting the exact same old snapshot to make stale data look live.

---

## 1.7 Parse-health equality is now exact

v3 used approximate equality language.

v4 defines:

```text
agent count = AUX/status count = timer count
```

If this is not true, show a parse-health warning.

---

## 1.8 Partial-invalid paste vs fully-invalid paste is now defined

v4 defines:

- If a paste produces at least one valid agent record, process valid records and report excluded invalid/noise records.
- If a paste produces zero valid agent records, reject the paste as fully invalid.
- A fully invalid paste must not update official IN/OUT, LAT/BE samples, or stale timer.

---

## 1.9 Manual LAT% UI is now mandatory

v3 described Manual LAT rules but did not clearly require the UI control.

v4 requires:

- Manual LAT% input/control must be visible for each LOB during live interval operation.
- Manual BE% input/control must appear when BE mode is active.

---

## 1.10 BE HEAD NEED/OVER UI is now mandatory

v3 defined BE headcount math but did not clearly require display.

v4 requires:

- When BE mode is active, show BE HEAD NEED/OVER using `requiredInBE`.

---

## 1.11 Temporary Manual IN hard timer is now exploit-proof

v4 defines:

- `temporaryManualInStartedAt` is set only when Manual IN delta transitions from exactly `0` to a non-zero value.
- It must not refresh on later changes while the value remains non-zero.
- The 30-second hard reset uses this original start timestamp.
- This prevents indefinite extension by repeatedly clicking.

---

## 1.12 New valid paste clears temporary Manual IN what-if

v4 defines:

- If a new valid paste arrives while temporary Manual IN what-if is active, reset Manual IN delta to 0 immediately and hide alternate values.

---

## 1.13 7:00–7:15 AM gap is now explicit

v4 defines:

- No Interval 33 exists.
- From 7:00 AM to 7:15 AM, the tool must not start a new official LAT/BE interval.
- It may hold final Interval 32/end-of-window state until 7:15 AM lock/reset.

---

## 1.14 Frozen interval indicator is now explicit

v4 defines:

- When frozen/stale, the interval indicator must remain pinned to the interval where freeze occurred.
- It must not continue advancing with wall-clock time while frozen.

---

## 1.15 Outside-window mode label added

v4 adds:

```text
OUTSIDE_WINDOW
```

to the official mode-label list.

---

# 2. Scope and Non-Scope

## 2.1 In Scope

This specification covers:

- LAT Req input.
- BE Req input.
- Auto Req schedule persistence.
- 30-minute interval engine.
- LAT%.
- Expected LAT%.
- BE%.
- BE LAT%.
- Expected BE%.
- Manual LAT% correction.
- Manual BE% correction.
- Temporary Manual IN +/- what-if display.
- 60-minute stale freeze.
- Refresh / Start Over behavior.
- 7:15 AM hard lock/reset.
- 7:00–7:15 AM end-of-window holding behavior.
- Duplicate snapshot detection.
- Parse-health validation.
- UI metric requirements.
- Persistence schema.
- State model.
- Freeze criteria.

## 2.2 Out of Scope

This specification does not authorize:

- Changing frozen parser business logic.
- Changing LOB aliases.
- Changing known IN/OUT status classification.
- Adding network calls.
- Adding external libraries.
- Adding external fonts.
- Splitting the tool into multiple required runtime files.
- Replacing the existing frozen operational AUX display.
- Building historical reports unless explicitly required by finalized interval storage.

---

# 3. Frozen Baseline Protection

The LAT/BE engine must be implemented as an added layer.

Do not rewrite the parser unless a specific patch is required by this specification.

Frozen behavior to preserve:

- LOB extraction.
- Agent extraction.
- Known IN statuses.
- Known OUT statuses.
- Unknown AUX fallback behavior.
- Longest Available display logic.
- Per-LOB IN/OUT list-side memory.
- Existing offline single-file requirement.
- No external dependencies.

The LAT/BE engine consumes parser outputs. It must not become the parser.

---

# 4. Supported LOBs and State Isolation

Supported LOBs:

| LOB | Short Label |
|---|---|
| CSA T2 | T2 |
| CSA SP | SP |
| CSA T1 | T1 |

Each LOB must have fully independent state.

No LAT/BE state may be global if it affects only one LOB.

Per-LOB independent state includes:

- Official IN.
- Official OUT.
- LAT Req.
- BE Req.
- REQ92%.
- BE%.
- LAT%.
- Expected LAT%.
- BE LAT%.
- Expected BE%.
- Manual LAT% state.
- Manual BE% state.
- Temporary Manual IN what-if state.
- Current interval.
- Interval samples.
- Stale state.
- Frozen state.
- Duplicate snapshot signature.
- List-side UI state.
- Parse-health warnings.

---

# 5. Time Model

## 5.1 Timestamp Source

All event timestamps use local PC time.

Valid event timestamp sources:

- Paste processed time.
- Manual LAT Apply time.
- Manual BE Apply time.
- LAT Req change time.
- BE Req change time.
- Auto Req schedule boundary time.
- Tick time.
- 7:15 AM lock time.

## 5.2 Snapshot Paste Time

Snapshot timestamp is the local PC time at the moment the paste is processed successfully.

The tool does not attempt to infer the snapshot's original generation time.

This is accepted.

## 5.3 Event Time Precision

Use actual event time, not tick-quantized time, for:

- paste,
- Apply button,
- Req change,
- Auto Req activation.

Recommended implementation:

```text
Date.now()
```

Use the live tick only for periodic recalculation and UI refresh.

---

# 6. Operating Window

## 6.1 Main Window

Official LAT/BE tracking window:

```text
3:00 PM to 7:00 AM next day
```

## 6.2 Interval Count

Normal window length:

```text
16 hours
```

Interval length:

```text
30 minutes
```

Normal interval count:

```text
32 intervals
```

## 6.3 Interval Table

| Interval | Start | End |
|---:|---|---|
| 1 | 3:00 PM | 3:30 PM |
| 2 | 3:30 PM | 4:00 PM |
| 3 | 4:00 PM | 4:30 PM |
| ... | ... | ... |
| 31 | 6:00 AM | 6:30 AM |
| 32 | 6:30 AM | 7:00 AM |

## 6.4 No Interval 33

No official Interval 33 exists.

From:

```text
7:00 AM to 7:15 AM
```

the tool must not create a new official LAT/BE interval.

The tool may hold end-of-window data until the 7:15 AM hard lock.

## 6.5 Outside Operating Window

Outside 3:00 PM–7:00 AM:

- The tool may parse and show operational IN/OUT.
- Official LAT/BE live tracking must not start.
- Show mode:

```text
OUTSIDE_WINDOW
```

---

# 7. 7:15 AM Hard Lock / Reset

## 7.1 Trigger

At exactly:

```text
07:15:00 AM local PC time
```

any open tool session must enter end-of-day lock/reset.

## 7.2 Recommended UX

Show a lock screen:

```text
Session ended. Please refresh the page to start a new day.
```

The user action should be simple:

```text
Refresh the page
```

## 7.3 What Must Be Cleared or Locked

At 7:15 AM, the live operational session must be cleared or locked:

- pasted snapshot state,
- agent rows,
- Official IN/OUT live data,
- current LAT/BE interval state,
- LAT samples,
- BE samples,
- manual LAT segment,
- manual BE segment,
- stale state,
- frozen state,
- duplicate snapshot memory,
- temporary Manual IN what-if state,
- current live interval indicator.

## 7.4 What Must Persist

The following must not be lost:

- Auto Req schedules for the next operating window.
- configured LAT Req values intended for the next operating window.
- configured BE Req values intended for the next operating window.
- any persistent settings required to start the next day correctly.

## 7.5 Why Persistence Is Required

Because the recommended lock screen asks the user to refresh, browser memory will be lost.

Therefore, persistent data must be stored in local browser storage.

Required mechanism:

```text
localStorage
```

In-memory variables alone do not satisfy this requirement.

---

# 8. Persistence Model

## 8.1 Required Persistent Storage

Use `localStorage` for Auto Req schedules and configured Req values.

## 8.2 Required localStorage Key

Recommended key:

```text
INT48_LAT_BE_REQ_SCHEDULE_V4
```

## 8.3 Required Stored Payload

Recommended JSON schema:

```json
{
  "schemaVersion": "INT48_LAT_BE_REQ_SCHEDULE_V4",
  "savedAt": "ISO timestamp",
  "operatingWindowDate": "YYYY-MM-DD",
  "lobs": {
    "T2": {
      "latReqSchedule": [
        {
          "intervalId": 1,
          "startLocal": "15:00",
          "latReq": 0
        }
      ],
      "beReqSchedule": [
        {
          "intervalId": 1,
          "startLocal": "15:00",
          "beReq": 0
        }
      ]
    },
    "SP": {
      "latReqSchedule": [],
      "beReqSchedule": []
    },
    "T1": {
      "latReqSchedule": [],
      "beReqSchedule": []
    }
  }
}
```

## 8.4 Load Behavior

On page load:

1. Read `INT48_LAT_BE_REQ_SCHEDULE_V4` from `localStorage`.
2. Validate `schemaVersion`.
3. Validate LOB keys.
4. Validate LAT Req values.
5. Validate BE Req values.
6. Restore schedules if valid.
7. Ignore invalid entries with visible warning.
8. If storage is missing or corrupt, do not crash, and display a visible warning that the Auto Req schedule could not be restored and must be re-entered or re-confirmed.

## 8.5 Save Behavior

Save persistent schedule data when:

- Auto Req schedule is entered.
- Auto Req schedule is edited.
- LAT Req scheduled values change.
- BE Req scheduled values change.
- user imports/pastes schedule data.
- before entering 7:15 AM lock where possible.

## 8.6 What Must Not Be Persisted as Next-Day Truth

Do not persist these as active next-day live data:

- current agents,
- current Official IN/OUT from yesterday,
- current LAT samples,
- current BE samples,
- current manual LAT/BE segments,
- stale state,
- frozen state,
- duplicate snapshot signature,
- temporary Manual IN what-if state.

## 8.7 operatingWindowDate Mismatch

If the stored `operatingWindowDate` does not match the date of the next upcoming 3:00 PM operating window, the stored schedule must be treated as stale/unconfirmed.

Do not auto-apply a stale/unconfirmed schedule to the live window.

Required behavior:

- show a visible warning that the saved Auto Req schedule belongs to a different operating window,
- require the user to confirm reuse or re-enter the schedule,
- keep the stored data available for review,
- do not silently apply it as current-day truth.

If the user confirms reuse, update `operatingWindowDate` to the current upcoming 3:00 PM operating window and save the confirmed schedule back to `localStorage`.

---

# 9. Auto Req Schedule Rules

## 9.1 Source Timing

Req schedules are sent at:

```text
6:00 AM
```

for the next operating window starting:

```text
3:00 PM
```

## 9.2 Auto Req Effective Period

Each Auto Req value starts at the first moment of its assigned interval.

It remains active until the next scheduled Req starts at the beginning of its own interval.

Example:

| Interval Start | LAT Req |
|---|---:|
| 3:00 PM | 10 |
| 3:30 PM | 12 |
| 4:00 PM | 11 |

Effective periods:

```text
3:00:00–3:29:59 = 10
3:30:00–3:59:59 = 12
4:00:00–4:29:59 = 11
```

## 9.3 Auto Req Is Scheduled Truth

Auto Req is not a manual correction.

Auto Req applies from its scheduled interval boundary.

It does not rewrite a prior interval unless the prior interval is the interval it is scheduled to own.

## 9.4 Manual Overrides Auto

Manual edits always override what came before for their corrected segment.

## 9.5 Auto Req After 7:15 AM

Auto Req data entered or loaded after 6:00 AM must survive the 7:15 AM lock/reset and remain available for the next 3:00 PM operating window.

---

# 10. Req Inputs

## 10.1 LAT Req

LAT Req is the base required headcount.

Rules:

```text
Default = 0
Allowed = decimal >= 0
Display input may show 00.00
```

Examples:

```text
0
5
5.5
12.25
00.00
```

## 10.2 BE Req

BE Req is the extra required headcount added on top of LAT Req.

Rules:

```text
Default = 0
Allowed = integer only
Allowed values = 0, 1, 2, 3, 4, 5, 6
```

## 10.3 BE Required

Formula:

```text
BE Required = LAT Req + BE Req
```

## 10.4 Requirement Zero / Invalid Requirement

If LAT Req is 0 or invalid:

- REQ92% displays `—`.
- LAT% displays `—`.
- Expected LAT% displays `—`.
- HEAD displays `—`.

If `LAT Req + BE Req` is 0 or invalid:

- BE% displays `—`.
- BE LAT% displays `—`.
- Expected BE% displays `—`.
- BE HEAD displays `—`.

Do not show false NEED.

Do not show false OVER.

---

# 11. Official IN / OUT

## 11.1 Official IN

Official IN is the parsed IN count from the latest accepted non-invalid snapshot.

It is used for all official calculations.

```text
Official IN = parsed IN count
```

## 11.2 Official OUT

Official OUT is the parsed OUT count from the latest accepted non-invalid snapshot.

```text
Official OUT = parsed OUT count
```

## 11.3 No Manual OUT

There is no Manual OUT adjustment.

The user cannot manually increase or decrease OUT.

## 11.4 Temporary Manual IN Does Not Replace Official IN

Temporary Manual IN +/- is separate and does not alter Official IN.

---

# 12. IN / OUT Classification

## 12.1 Known IN Statuses

An agent is counted as IN when status is one of:

- `AUXIN`
- `AUXOUT`
- `AVAILABLE`
- `FOLLOWUP/OUTBOUNDOUT`
- `HOLD`
- `BUSYIN`
- `ACW`
- `ACWOUT`

## 12.2 Known OUT Statuses

An agent is counted as OUT when status is one of:

- `PERSONAL`
- `SYSTEM PROBLEMS`
- `BREAK`
- `LUNCH`
- `COACHING`
- `NPS`
- `PROJECT`
- `FOLLOWUP/OUTBOUND`
- `AUXWK`
- `MEETING`

## 12.3 Unknown AUX Rule

If status is unknown:

- Unknown ending with `IN` counts as IN.
- Unknown ending with `OUT` counts as IN.
- Other unknown statuses count as OUT.

This is intentional.

## 12.4 Unknown Warnings

Unknown statuses must be visible in parse health.

Required categories:

- Unknown counted as IN.
- Unknown counted as OUT.
- Total unknown statuses.

---

# 13. Timer and Parse Health

## 13.1 Timer Format

Supported timer format:

```text
HH:MM:SS
```

Regex:

```text
^\d{1,2}:\d{2}:\d{2}$
```

Valid examples:

```text
00:00:10
00:15:42
01:03:09
10:22:31
```

## 13.2 Missing Timer

If an agent-like record has no timer, treat it as likely noise text.

Do not count it silently as an agent.

## 13.3 Exact Count Match

Parse health must compare:

```text
agent count = AUX/status count = timer count
```

If not exactly equal, show a warning.

## 13.4 Excluded Invalid Records

Any excluded agent-like record must be reported as:

```text
Excluded invalid records
```

Do not silently omit invalid records.

## 13.5 Fully Invalid Paste vs Partial Invalid Paste

A paste is fully invalid only if it produces:

```text
0 valid agent records
```

Fully invalid paste behavior:

- no Official IN/OUT update,
- no LAT/BE sample,
- no stale timer update,
- show invalid paste warning.

Partial invalid paste behavior:

- process valid agent records,
- update Official IN/OUT from valid records,
- create LAT/BE sample,
- update stale timer if not exact duplicate,
- show excluded invalid/noise warning.

---

# 14. Snapshot Rules

## 14.1 Valid Snapshot

A valid snapshot is a paste that produces at least one valid agent record for at least one supported LOB.

## 14.2 Valid Snapshot Updates

A non-duplicate valid snapshot updates:

- Official IN.
- Official OUT.
- current REQ92%.
- current BE%.
- LAT/BE samples.
- `lastValidSnapshotAt`.

## 14.3 Exact Duplicate Snapshot

An exact duplicate snapshot is one where the current parsed snapshot signature is identical to the previous parsed snapshot signature.

Exact duplicate means same:

- LOBs,
- agent names,
- statuses,
- timers,
- IN/OUT classification,
- parsed structure.

## 14.4 Duplicate Signature Comparison

Duplicate comparison must use parsed fields, not raw pasted text.

Comparison should be order-independent by agent identity where practical.

Recommended signature sorting:

```text
sort by LOB, then normalized agent name, then status, then timer
```

This prevents false non-duplicate alerts caused only by row order changes.

## 14.5 Duplicate Snapshot Behavior

Exact duplicate snapshots:

- are allowed,
- are calculated,
- show Duplicate Snapshot alert,
- do not reset the stale timer.

## 14.6 Non-Duplicate Valid Snapshot Behavior

A non-duplicate valid snapshot:

- updates official values,
- creates a new sample,
- resets stale timer,
- clears temporary Manual IN what-if if active.

---

# 15. REQ92%, BE%, and Headcount Metrics

## 15.1 REQ92%

Formula:

```text
REQ92% = Official IN / LAT Req × 100
```

Important naming note:

`REQ92%` is the inherited frozen-baseline label. The formula is a plain staffing ratio against LAT Req. The 92% target floor is applied separately in the HEAD NEED/OVER calculation.

## 15.2 BE%

Formula:

```text
BE% = Official IN / (LAT Req + BE Req) × 100
```

## 15.3 92% HEAD NEED / OVER

If LAT Req is 0 or invalid:

```text
HEAD = —
```

Otherwise:

```text
requiredIn92 = ceil(0.92 × LAT Req)
```

If:

```text
Official IN >= requiredIn92
```

show:

```text
OVER = Official IN - requiredIn92
```

If:

```text
Official IN < requiredIn92
```

show:

```text
NEED = requiredIn92 - Official IN
```

## 15.4 BE HEAD NEED / OVER

If `LAT Req + BE Req` is 0 or invalid:

```text
BE HEAD = —
```

Otherwise:

```text
requiredInBE = ceil(LAT Req + BE Req)
```

If:

```text
Official IN >= requiredInBE
```

show:

```text
BE OVER = Official IN - requiredInBE
```

If:

```text
Official IN < requiredInBE
```

show:

```text
BE NEED = requiredInBE - Official IN
```

No fractional people may be displayed.

---

# 16. LAT Engine

## 16.1 LAT% Definition

LAT% is the time-weighted average of REQ92% during the current 30-minute interval.

It is not a simple average of snapshots.

## 16.2 Time-Weighted LAT Formula

During elapsed portion of the interval:

```text
LAT% =
sum(segmentREQ92PctRaw × segmentDurationSeconds)
/
elapsedSeconds
```

## 16.3 First Snapshot Late Assumption

If the first valid snapshot arrives after interval start, assume its current REQ92% was active from interval start.

Example:

- Interval: 3:00–3:30.
- First paste: 3:20.
- REQ92% at 3:20: 95.00%.

Then:

```text
LAT% = 95.00%
Expected LAT% = 95.00%
Mode = ASSUMED
```

## 16.4 Expected LAT%

Expected LAT% projects the current REQ92% through the remaining interval time.

Formula:

```text
Expected LAT% =
(closedLatWeightedSeconds + currentREQ92PctRaw × remainingSeconds)
/
1800
```

Where:

```text
closedLatWeightedSeconds =
sum(closedSegmentREQ92PctRaw × closedSegmentDurationSeconds)
```

and:

```text
remainingSeconds = intervalEnd - now
```

At interval end:

```text
Expected LAT% = final LAT%
```

## 16.5 LAT Requirement Zero

If LAT Req is 0 or invalid, LAT% and Expected LAT% display `—`.

The engine may retain underlying sample timestamps, but no official percentage should be displayed until requirement becomes valid.

---

# 17. BE Engine

## 17.1 Background Tracking

BE tracking always runs in the background.

## 17.2 BE UI Trigger

While:

```text
BE Req = 0
```

BE UI/data display remains hidden or disabled.

When:

```text
BE Req >= 1
```

BE UI/data display appears.

## 17.3 BE Visible Metrics

When BE is active, show:

- BE Req.
- BE%.
- BE LAT%.
- Expected BE%.
- Manual BE% edit.
- BE HEAD NEED/OVER.

## 17.4 BE LAT%

BE LAT% is the time-weighted average of BE% during the current 30-minute interval.

Formula:

```text
BE LAT% =
sum(segmentBEPctRaw × segmentDurationSeconds)
/
elapsedSeconds
```

## 17.5 Expected BE%

Expected BE% projects the current BE% through the remaining interval time.

Formula:

```text
Expected BE% =
(closedBeWeightedSeconds + currentBEPctRaw × remainingSeconds)
/
1800
```

Where:

```text
closedBeWeightedSeconds =
sum(closedSegmentBEPctRaw × closedSegmentDurationSeconds)
```

and:

```text
remainingSeconds = intervalEnd - now
```

At interval end:

```text
Expected BE% = final BE LAT%
```

---

# 18. Manual LAT / Manual BE Corrections

## 18.1 Manual Values Are User-Entered

Manual LAT% and Manual BE% are values the user types and applies.

They are not system-computed values.

The UI may pre-fill the current calculated value as a suggestion.

The applied value is whatever the user submits on Apply.

## 18.2 Apply Timestamp

Manual correction applies at the moment the user presses Apply.

Typing into the field does not apply the value.

## 18.3 Manual LAT Segment

Manual LAT% applies from:

```text
intervalStart
```

to:

```text
manualApplyTimestamp
```

## 18.4 Manual BE Segment

Manual BE% applies from:

```text
intervalStart
```

to:

```text
manualApplyTimestamp
```

## 18.5 After Manual Apply

After the Apply timestamp:

- live tracking resumes for the rest of the interval,
- new valid snapshots may affect the live portion after the manual segment,
- Req changes may affect non-manual portions,
- the manual segment remains locked.

## 18.6 Manual Segment Blending Formula

For LAT:

```text
LAT% =
(manualLatPctRaw × manualSegmentDurationSeconds
 + sum(liveSegmentREQ92PctRaw × liveSegmentDurationSeconds))
/
elapsedSeconds
```

For BE:

```text
BE LAT% =
(manualBePctRaw × manualSegmentDurationSeconds
 + sum(liveSegmentBEPctRaw × liveSegmentDurationSeconds))
/
elapsedSeconds
```

## 18.7 Expected Formula With Manual Segment

For Expected LAT:

```text
Expected LAT% =
(manualLatPctRaw × manualSegmentDurationSeconds
 + sum(closedLiveREQ92PctRaw × closedLiveDurationSeconds)
 + currentREQ92PctRaw × remainingSeconds)
/
1800
```

For Expected BE:

```text
Expected BE% =
(manualBePctRaw × manualSegmentDurationSeconds
 + sum(closedLiveBEPctRaw × closedLiveDurationSeconds)
 + currentBEPctRaw × remainingSeconds)
/
1800
```

## 18.8 Manual Priority

Manual segments are highest source of truth.

They must not be overwritten by:

- snapshot paste,
- Auto Req change,
- Manual Req change,
- LAT Req change,
- BE Req change,
- live tick,
- temporary Manual IN what-if.

## 18.9 Second Manual Edit

If the user applies a second Manual LAT% in the same interval:

- replace the previous Manual LAT segment,
- use the new value,
- set the new segment from interval start to the second Apply timestamp,
- rebuild non-manual live portion after that timestamp.

If the user applies a second Manual BE% in the same interval:

- replace the previous Manual BE segment,
- use the new value,
- set the new segment from interval start to the second Apply timestamp,
- rebuild non-manual live portion after that timestamp.

Manual segments do not stack.

---

# 19. Req Change Recalculation

## 19.1 LAT Req Change

When LAT Req changes during an active interval:

- recalculate REQ92% using latest LAT Req,
- rebuild non-manual LAT portions from interval start,
- rebuild non-manual BE portions from interval start because BE Required includes LAT Req,
- keep Manual LAT segment locked,
- keep Manual BE segment locked.

## 19.2 BE Req Change

When BE Req changes during an active interval:

- recalculate BE% using latest BE Req,
- rebuild non-manual BE portions from interval start,
- keep Manual BE segment locked,
- do not change REQ92%,
- do not change LAT%,
- do not change Expected LAT%.

## 19.3 Latest Manual Wins

Manual edit always overrides what came before for its corrected segment.

## 19.4 Auto Req vs Manual Req

Auto Req is scheduled truth.

Manual Req edit is correction.

Manual correction overrides prior values for the corrected segment.

---

# 20. Temporary Manual IN What-If

## 20.1 Purpose

Temporary Manual IN +/- is a side display feature.

It lets the user briefly see "what if IN was higher/lower?"

It is not official.

## 20.1.1 Relationship to Existing Basic Simulation

If the frozen baseline already contains a Basic Simulation after paste feature, the Temporary Manual IN +/- feature standardizes and replaces that behavior for LAT/BE scope.

Do not keep two competing IN simulation controls.

The simulation behavior used after this implementation must follow Section 20:

- temporary only,
- IN +/- only,
- maximum range ±20,
- no official state impact,
- 10-second idle reset,
- 30-second hard reset,
- immediate reset to 0 on new valid paste.

## 20.2 Allowed Range

```text
-20 to +20
```

## 20.3 Adjusted IN

Formula:

```text
Adjusted IN = Official IN + temporaryManualInDelta
```

## 20.4 What It May Display

When active, show alternate values beside official values:

- adjusted REQ92%,
- adjusted HEAD NEED/OVER,
- adjusted LAT%,
- adjusted Expected LAT%,
- adjusted BE% if BE active,
- adjusted BE LAT% if BE active,
- adjusted Expected BE% if BE active,
- adjusted BE HEAD NEED/OVER if BE active.

## 20.5 What It Must Not Affect

Temporary Manual IN +/- must not:

- change Official IN,
- change Official OUT,
- create LAT sample,
- create BE sample,
- update stale timer,
- change interval history,
- change finalized intervals,
- change manual LAT segment,
- change manual BE segment,
- persist as saved state,
- affect official values after reset.

## 20.6 Idle Reset

If the value is unchanged for 10 seconds:

```text
temporaryManualInDelta = 0
```

Hide alternate values.

## 20.7 Hard Reset

If the user keeps changing the value, maximum display time is 30 seconds.

After 30 seconds:

```text
force temporaryManualInDelta = 0
```

Hide alternate values.

## 20.8 Hard Reset Timestamp Rule

Set:

```text
temporaryManualInStartedAt
```

only when:

```text
temporaryManualInDelta changes from 0 to non-zero
```

Do not update `temporaryManualInStartedAt` on later changes while the value remains non-zero.

This prevents indefinite extension.

## 20.9 Manual Reset to Zero

If user manually sets delta to 0:

- reset immediately,
- hide alternate values immediately.

## 20.10 New Valid Paste While Active

If a new valid snapshot arrives while Manual IN what-if is active:

- reset delta to 0 immediately,
- hide alternate values immediately,
- process the snapshot normally.

---

# 21. Stale / Frozen Rule

## 21.1 Stale Timeout

If 60 minutes pass without a new non-duplicate valid snapshot:

```text
now - lastValidSnapshotAt >= 60 minutes
```

the tool must freeze live LAT/BE tracking.

## 21.2 What Counts as Activity

Resets stale timer:

- non-duplicate valid snapshot.

Does not reset stale timer:

- exact duplicate snapshot,
- invalid paste,
- temporary Manual IN change,
- Manual LAT edit,
- Manual BE edit,
- Req edit,
- Auto Req schedule edit,
- live tick.

## 21.3 Frozen Behavior

When frozen:

- stop live LAT/BE updates,
- stop Expected LAT/Expected BE movement,
- stop interval rollover using old data,
- keep last displayed values visible,
- mark values clearly FROZEN/STALE,
- show Refresh / Start Over,
- pin interval indicator to the interval where freeze occurred.

## 21.4 Frozen Visual Requirement

Frozen/stale state must be visually distinct from live state.

Minimum:

- clear label,
- disabled/greyed live metrics,
- visible Refresh / Start Over button or instruction.

## 21.5 Refresh / Start Over After Frozen

Refresh / Start Over:

- clears live LAT/BE session state,
- clears snapshots/agents/Official IN/OUT if using clean waiting state,
- clears interval samples,
- clears manual LAT/BE segments,
- clears duplicate snapshot memory,
- clears frozen/stale state,
- preserves Auto Req schedules from `localStorage`,
- preserves configured Req values from `localStorage`.

## 21.6 Resume After Refresh / Start Over

After Refresh / Start Over:

- derive active interval from current local time,
- wait for next valid snapshot,
- next valid snapshot is treated as first snapshot for that active interval,
- if late in interval, apply ASSUMED rule.

---

# 22. Interval Rollover

## 22.1 Normal Rollover

At the start of a new interval:

1. Finalize the previous interval.
2. Archive finalized interval data, including manual segment flags/values.
3. Close previous interval.
4. Determine new current interval.
5. Apply Auto Req for the new interval if available.
6. Carry over Official IN and Official OUT from latest accepted snapshot.
7. Carry over current LAT Req if no Auto Req change applies.
8. Carry over current BE Req if no Auto Req change applies.
9. Initialize REQ92%.
10. Initialize BE%.
11. Initialize LAT% = current REQ92% if requirement valid.
12. Initialize Expected LAT% = current REQ92% if requirement valid.
13. Initialize BE LAT% = current BE% if BE requirement valid.
14. Initialize Expected BE% = current BE% if BE requirement valid.
15. Clear active interval-specific Manual LAT segment.
16. Clear active interval-specific Manual BE segment.
17. Clear temporary Manual IN what-if.

## 22.2 Manual Segments Must Be Archived Before Clearing

Before clearing manual segments for the new interval, preserve their effect in the finalized interval record.

Do not wipe manual corrections before finalizing.

## 22.3 Rollover While Frozen

If frozen/stale, do not continue rolling intervals using old data.

The interval indicator remains pinned to the freeze interval.

## 22.4 7:15 AM Overrides Rollover

At 7:15 AM, lock/reset overrides normal rollover.

---

# 23. Finalized Interval Record

Even if the UI does not display historical intervals immediately, the engine should maintain finalized interval records for audit and future display.

Recommended schema:

```json
{
  "intervalId": 1,
  "lobId": "T2",
  "intervalStart": "ISO timestamp",
  "intervalEnd": "ISO timestamp",
  "finalReq92PctRaw": 0,
  "finalLatPctRaw": 0,
  "finalExpectedLatPctRaw": 0,
  "finalBePctRaw": 0,
  "finalBeLatPctRaw": 0,
  "finalExpectedBePctRaw": 0,
  "latReq": 0,
  "beReq": 0,
  "officialIn": 0,
  "officialOut": 0,
  "sampleCount": 0,
  "hadManualLatCorrection": false,
  "manualLatPctRaw": null,
  "hadManualBeCorrection": false,
  "manualBePctRaw": null,
  "wasAssumed": false,
  "wasFrozen": false,
  "wasOutsideWindow": false
}
```

---

# 24. Sample Records

## 24.1 LAT Sample

Recommended schema:

```json
{
  "timestamp": "ISO timestamp",
  "lobId": "T2",
  "officialIn": 0,
  "officialOut": 0,
  "latReq": 0,
  "req92PctRaw": 0,
  "source": "paste|rollover|req-change|auto-req"
}
```

## 24.2 BE Sample

Recommended schema:

```json
{
  "timestamp": "ISO timestamp",
  "lobId": "T2",
  "officialIn": 0,
  "officialOut": 0,
  "latReq": 0,
  "beReq": 0,
  "bePctRaw": 0,
  "source": "paste|rollover|req-change|auto-req"
}
```

## 24.3 Manual Segment

Recommended schema:

```json
{
  "metric": "LAT|BE",
  "lobId": "T2",
  "valueRaw": 0,
  "intervalStart": "ISO timestamp",
  "appliedAt": "ISO timestamp",
  "source": "manual"
}
```

---

# 25. Live Tick

## 25.1 Tick Frequency

Recommended:

```text
1 second
```

Acceptable maximum:

```text
5 seconds
```

## 25.2 Tick Order

On each tick:

1. Check 7:15 AM hard lock.
2. Check whether current time is outside official window.
3. Check stale timeout.
4. If not locked/frozen, check interval rollover.
5. Recalculate current LAT%.
6. Recalculate Expected LAT%.
7. Recalculate current BE LAT%.
8. Recalculate Expected BE%.
9. Check temporary Manual IN idle reset.
10. Check temporary Manual IN hard reset.
11. Update UI.

## 25.3 No Live Movement While Frozen

When frozen or locked:

- no LAT movement,
- no BE movement,
- no Expected movement,
- no interval indicator advance.

---

# 26. UI Requirements

## 26.1 Always Visible Per LOB During Live Window

Show:

- LOB label.
- Official IN.
- Official OUT.
- LAT Req.
- REQ92%.
- LAT%.
- Expected LAT%.
- 92% HEAD NEED/OVER.
- Manual LAT% input/control.
- Current interval.
- Mode label.
- Longest Available agent name.
- Longest Available timer.
- Parse-health status.

## 26.2 BE UI When Active

When BE Req >= 1, show:

- BE Req.
- BE%.
- BE LAT%.
- Expected BE%.
- Manual BE% input/control.
- BE HEAD NEED/OVER.

When BE Req = 0:

- hide or disable BE UI.

## 26.3 Auto Req Schedule Entry UI

The tool must provide a way for the user to enter or edit Auto Req schedules inside the offline HTML file.

Minimum requirements:

- schedule entry must be per LOB: T2, SP, and T1,
- schedule entry must be per 30-minute interval,
- LAT Req entry must allow decimal values,
- BE Req entry must allow only integers 0–6,
- invalid entries must show validation feedback before saving,
- saved schedules must persist using the `localStorage` rules in Section 8,
- the UI must show whether the schedule was restored successfully, missing, corrupt, stale/unconfirmed, or confirmed for the upcoming operating window,
- the UI must allow the user to re-confirm a stale/unconfirmed schedule when `operatingWindowDate` does not match the next upcoming 3:00 PM window.

## 26.4 Temporary Manual IN What-If UI

When active:

- show official values and adjusted values side by side,
- visually label adjusted values as temporary/what-if,
- hide adjusted values immediately when reset.

## 26.5 Duplicate Alert UI

When exact duplicate snapshot is detected:

```text
Duplicate snapshot
```

The alert must not block calculation.

## 26.6 Parse-Health UI

Show warnings for:

- missing LOBs,
- unknown counted as IN,
- unknown counted as OUT,
- excluded invalid records,
- agent/status/timer mismatch,
- fully invalid paste.

## 26.7 Mode Labels

Allowed mode labels:

- AUTO
- ASSUMED
- MANUAL
- STALE
- FROZEN
- LOCKED
- RESET
- OUTSIDE_WINDOW

## 26.8 Frozen UI

When frozen:

```text
STALE / FROZEN — Refresh / Start Over required
```

## 26.9 7:15 Lock UI

At 7:15 AM:

```text
Session ended. Please refresh the page to start a new day.
```

---

# 27. Display and Precision

## 27.1 Internal Precision

All internal calculations use raw/full precision.

Do not round internal values before using them in later calculations.

## 27.2 Display Format

All displayed percentages must be formatted as:

```text
00.00%
```

Examples:

```text
0.00%
92.46%
100.00%
125.75%
```

## 27.3 Missing / Invalid Display

Use:

```text
—
```

for unavailable calculations.

---

# 28. State Model

Recommended per-LOB state:

```text
id
label
officialIn
officialOut
latReq
beReq
req92PctRaw
bePctRaw
latPctRaw
expectedLatPctRaw
beLatPctRaw
expectedBePctRaw
intervalId
intervalStart
intervalEnd
lastValidSnapshotAt
freezeStartedAt
lastParsedSnapshotSignature
isStale
isFrozen
isLocked
mode
latSamples
beSamples
manualLatSegment
manualBeSegment
temporaryManualInDelta
temporaryManualInStartedAt
temporaryManualInLastChangedAt
autoReqSchedule
finalizedIntervals
listSide
parseHealth
```

---

# 29. Event Impact Matrix

| Event | Official IN/OUT | LAT Samples | BE Samples | Stale Timer | Manual LAT/BE | Auto Req Storage | What-If |
|---|---|---|---|---|---|---|
| Non-duplicate valid paste | Update | Add | Add | Reset | Preserve | No change | Reset to 0 |
| Exact duplicate paste | Same values | Calculate allowed | Calculate allowed | No reset | Preserve | No change | Reset to 0 |
| Fully invalid paste | No change | No | No | No reset | Preserve | No change | No change |
| Partial invalid paste | Update valid records | Add | Add | Reset if non-duplicate | Preserve | No change | Reset to 0 |
| LAT Req change | No change | Rebuild non-manual | Rebuild non-manual | No reset | Preserve | Save if scheduled/configured | No change |
| BE Req change | No change | No LAT impact | Rebuild non-manual BE | No reset | Preserve | Save if scheduled/configured | No change |
| Manual LAT Apply | No change | Manual segment | No BE impact | No reset | Replace Manual LAT | No change | No change |
| Manual BE Apply | No change | No LAT impact | Manual segment | No reset | Replace Manual BE | No change | No change |
| Manual IN +/- | No official change | No official change | No official change | No reset | Preserve | No change | Show temporary |
| 60-min stale | Freeze | Stop | Stop | N/A | Preserve visible | Preserve | Reset |
| Refresh / Start Over | Clear live state | Clear | Clear | Clear | Clear | Preserve | Reset |
| 7:15 AM lock | Lock/clear live state | Clear | Clear | Clear | Clear | Preserve | Reset |

---

# 30. Test Scenarios

## 30.1 First Snapshot at Interval Start

Expected:

```text
REQ92 = LAT = Expected LAT
BE = BE LAT = Expected BE
```

if requirements are valid.

## 30.2 First Snapshot 20 Minutes Late

Expected:

- ASSUMED mode.
- First value assumed from interval start.
- LAT = current REQ92.
- Expected LAT = current REQ92.

## 30.3 Manual LAT After First Snapshot

Expected:

- user-entered Manual LAT applies from interval start to Apply timestamp,
- live tracking continues after Apply timestamp.

## 30.4 Manual LAT Twice

Expected:

- second Manual LAT replaces first Manual LAT,
- non-manual portion rebuilds after second Apply timestamp.

## 30.5 Manual BE Twice

Expected:

- second Manual BE replaces first Manual BE,
- non-manual BE portion rebuilds after second Apply timestamp.

## 30.6 BE Req = 0

Expected:

- BE engine may track internally,
- BE UI hidden/disabled.

## 30.7 BE Req Changes 0 to 3

Expected:

- BE UI appears,
- BE recalculates from interval start for non-manual BE portions,
- LAT does not change.

## 30.8 BE Req Changes 3 to 5

Expected:

- BE recalculates using BE Req 5 for non-manual BE portions,
- manual BE segment preserved.

## 30.9 BE Req = 6

Expected:

- accepted,
- BE Required = LAT Req + 6,
- BE HEAD uses ceil.

## 30.10 LAT Req Mid-Interval Change

Expected:

- LAT recalculates from interval start for non-manual LAT portions,
- BE recalculates from interval start for non-manual BE portions,
- manual segments preserved.

## 30.11 LAT Req = 0

Expected:

- official percentage metrics display `—`,
- no false NEED/OVER.

## 30.12 No Snapshot for 60 Minutes

Expected:

- tool freezes,
- Expected metrics stop moving,
- interval indicator pinned,
- Refresh / Start Over appears.

## 30.13 Exact Duplicate Snapshot Every 50 Minutes

Expected:

- duplicate alert appears,
- duplicate calculated,
- stale timer does not reset,
- tool eventually freezes if no non-duplicate valid snapshot arrives for 60 minutes.

## 30.14 Refresh / Start Over After Frozen

Expected:

- live session clears,
- Auto Req schedule persists,
- next valid paste becomes first snapshot for current active interval.

## 30.15 7:15 AM Lock

Expected:

- tool locks,
- user must refresh,
- live state cleared,
- Auto Req schedule preserved.

## 30.16 Auto Req Entered After 6:00 AM

Expected:

- schedule saved to localStorage,
- survives 7:15 AM lock,
- available at next 3:00 PM start.

## 30.17 7:00–7:15 AM

Expected:

- no Interval 33,
- hold end-of-window state,
- lock at 7:15 AM.

## 30.18 Invalid Paste

Expected:

- no official update if zero valid records,
- warning displayed.

## 30.19 Partial Invalid Paste

Expected:

- valid records processed,
- invalid/noise records reported.

## 30.20 Missing Timer

Expected:

- record excluded or warning shown,
- no silent counting.

## 30.21 Count Mismatch

Expected:

- warning if agent count, status count, and timer count are not exactly equal.

## 30.22 Unknown Ending OUT

Expected:

- counted as IN,
- warning shown.

## 30.23 Unknown Not Ending IN/OUT

Expected:

- counted as OUT,
- warning shown.

## 30.24 Manual IN +5 then No Change

Expected:

- adjusted values appear,
- after 10 seconds unchanged, reset to 0.

## 30.25 Manual IN Keeps Changing

Expected:

- max display time 30 seconds from first non-zero,
- force reset to 0.

## 30.26 Manual IN Reset to 0

Expected:

- alternate data disappears immediately.

## 30.27 New Paste While Manual IN What-If Active

Expected:

- Manual IN delta resets immediately,
- paste processes normally.

## 30.28 Outside Operating Window

Expected:

- operational IN/OUT can display,
- official LAT/BE tracking does not start,
- mode OUTSIDE_WINDOW.

## 30.29 Missing or Corrupt localStorage

Expected:

- tool does not crash,
- visible warning appears,
- Auto Req schedule is not silently assumed,
- user is asked to re-enter or re-confirm schedule.

## 30.30 operatingWindowDate Mismatch

Expected:

- stored schedule is marked stale/unconfirmed,
- schedule is not auto-applied as current-day truth,
- user can review and confirm reuse,
- confirming reuse updates `operatingWindowDate` and saves back to `localStorage`.

## 30.31 Auto Req Schedule Entry UI

Expected:

- user can enter schedule per LOB and per interval,
- LAT Req decimal validation works,
- BE Req integer 0–6 validation works,
- save writes valid schedule to `localStorage`,
- UI shows restored/missing/corrupt/stale/confirmed status.

## 30.32 Basic Simulation Replacement

Expected:

- no second competing simulation control exists,
- existing simulation behavior is standardized into Temporary Manual IN +/- rules,
- all temporary simulation values reset according to Section 20.

---

# 31. Freeze Criteria

The LAT/BE feature may be frozen only when all are true:

1. Tool remains one standalone offline HTML file.
2. No external dependency exists.
3. Frozen parser behavior is not changed.
4. LOB state is independent.
5. Timestamp source is local PC event time.
6. HH:MM:SS timer validation works.
7. Missing timer does not silently count as agent.
8. Agent/status/timer mismatch warning works.
9. Fully invalid paste is rejected.
10. Partial invalid paste processes valid records and warns on exclusions.
11. Unknown AUX warnings are visible.
12. Duplicate snapshot is allowed and calculated.
13. Duplicate snapshot alert appears.
14. Duplicate snapshot does not reset stale timer.
15. Non-duplicate valid snapshot resets stale timer.
16. LAT Req decimal validation works.
17. BE Req integer 0–6 validation works.
18. REQ92% formula works.
19. BE% formula works.
20. Requirement zero shows `—`.
21. 92% HEAD uses ceil(0.92 × LAT Req).
22. BE HEAD uses ceil(LAT Req + BE Req).
23. LAT% time-weighted average works.
24. Expected LAT% formula works.
25. BE LAT% time-weighted average works.
26. Expected BE% formula works.
27. Manual LAT is user-entered.
28. Manual BE is user-entered.
29. Manual Apply timestamp is button-click time.
30. Manual LAT segment blends correctly.
31. Manual BE segment blends correctly.
32. Second manual edit replaces first.
33. LAT Req changes rebuild non-manual LAT and BE portions.
34. BE Req changes rebuild non-manual BE only.
35. Manual segments remain locked during Req changes.
36. BE UI hidden while BE Req = 0.
37. BE UI shown when BE Req >= 1.
38. BE HEAD shown when BE active.
39. Manual LAT input visible during live operation.
40. Manual BE input visible when BE active.
41. Temporary Manual IN range is ±20.
42. Temporary Manual IN does not affect official state.
43. Temporary Manual IN 10-second idle reset works.
44. Temporary Manual IN 30-second hard reset works.
45. Temporary Manual IN startedAt cannot be extended by repeated changes.
46. New valid paste clears Manual IN what-if.
47. 60-minute stale freeze works.
48. Frozen interval indicator remains pinned.
49. Frozen state visually differs from live state.
50. Refresh / Start Over preserves Auto Req schedule.
51. Auto Req schedule persists through localStorage.
52. Auto Req schedule restores on page load.
53. Missing or corrupt Auto Req storage shows visible warning.
54. operatingWindowDate mismatch marks the schedule stale/unconfirmed and requires user confirmation before applying.
55. Auto Req Schedule Entry UI supports per-LOB and per-interval entry with validation.
56. Auto Req schedule survives 7:15 AM lock/refresh.
57. Auto Req applies from assigned interval boundary.
58. No Interval 33 starts from 7:00–7:15 AM.
59. 7:15 AM hard lock/reset works.
60. OUTSIDE_WINDOW mode works.
61. Basic Simulation after paste is replaced/standardized by Temporary Manual IN +/- with no competing control.
62. Internal raw precision retained.
63. Display formatting is `00.00%`.
64. No runtime console errors.
65. No horizontal overflow in target compact layouts.

---

# 32. Implementation Prompt

Use this prompt when handing the implementation to a developer or coding AI:

```text
You are modifying the INT 4.8 AUX Live Monitor single-file offline HTML tool.

Implement the LAT/BE engine using the specification:
INT4.8_LAT_BE_Business_Rules_and_Engine_Spec_v4.md

Hard constraints:
- Return the full updated HTML file.
- Keep it one standalone offline HTML file.
- Do not add external dependencies.
- Do not use external fonts.
- Do not add network calls.
- Do not rewrite the frozen parser unless the spec explicitly requires a parse-health warning enhancement.
- Preserve existing LOB extraction, IN/OUT classification, unknown AUX behavior, longest Available logic, and per-LOB UI state.

Implement:
1. 30-minute interval engine from 3:00 PM to 7:00 AM.
2. No Interval 33 between 7:00 AM and 7:15 AM.
3. 7:15 AM local PC lock/reset.
4. localStorage persistence for Auto Req schedules and configured Req values.
5. Auto Req schedule restore on page load.
6. Visible warning when localStorage is missing, corrupt, or cannot restore the schedule.
7. operatingWindowDate mismatch handling: mark stale/unconfirmed and require user confirmation before applying to current window.
8. Auto Req Schedule Entry UI: per LOB, per interval, LAT Req decimal validation, BE Req 0–6 validation, save to localStorage.
9. Auto Req schedule values start at assigned interval boundary and persist until the next schedule value.
10. Local PC event timestamps for paste, Apply, Req changes, and Auto Req boundaries.
11. LAT Req decimal input.
12. BE Req integer input 0–6.
13. BE tracking always in background.
14. BE UI hidden while BE Req = 0.
15. BE UI visible when BE Req >= 1.
16. REQ92% = Official IN / LAT Req × 100.
17. BE% = Official IN / (LAT Req + BE Req) × 100.
18. LAT% as time-weighted average.
19. Expected LAT% formula: (closedLatWeightedSeconds + currentREQ92PctRaw × remainingSeconds) / 1800.
20. BE LAT% as time-weighted average.
21. Expected BE% formula: (closedBeWeightedSeconds + currentBEPctRaw × remainingSeconds) / 1800.
22. Manual LAT% as user-entered correction applied on Apply click.
23. Manual BE% as user-entered correction applied on Apply click.
24. Manual segment blending from interval start to Apply timestamp, then live tracking continues.
25. Second manual edit replaces previous manual edit for the same metric in the same interval.
26. LAT Req changes rebuild non-manual LAT and BE portions while preserving manual segments.
27. BE Req changes rebuild non-manual BE portions only while preserving manual BE segment.
28. 92% HEAD NEED/OVER using ceil(0.92 × LAT Req).
29. BE HEAD NEED/OVER using ceil(LAT Req + BE Req).
30. Requirement zero displays — and never false OVER.
31. Exact duplicate snapshot detection using parsed fields, order-independent where practical.
32. Exact duplicate snapshots are allowed and calculated but do not reset stale timer.
33. Non-duplicate valid snapshots reset stale timer.
34. Fully invalid paste rejects with warning.
35. Partial invalid paste processes valid records and warns about excluded records.
36. Timer validation HH:MM:SS.
37. agent/status/timer exact count mismatch warning.
38. 60-minute stale freeze.
39. Frozen state pins interval indicator and stops expected metrics movement.
40. Refresh / Start Over clears live session but preserves Auto Req localStorage.
41. Temporary Manual IN +/- what-if only, range ±20.
42. Temporary Manual IN 10-second idle reset.
43. Temporary Manual IN 30-second hard reset from first non-zero timestamp.
44. New valid paste clears temporary Manual IN what-if.
45. No manual OUT.
46. Display all percentages as 00.00%, while retaining raw precision internally.
47. Add Manual LAT input/control and Manual BE input/control as required.
48. Add BE HEAD NEED/OVER display when BE active.
49. Add OUTSIDE_WINDOW mode.
50. Replace/standardize any existing Basic Simulation after paste behavior with Temporary Manual IN +/- rules; do not keep two competing simulation controls.


After implementation, run through the freeze criteria from Section 31 and report pass/fail for each item.
```

---

# 33. Final Readiness Statement

This v4 specification is intended to be a pre-implementation freeze candidate.

Implementation must not begin unless:

- the persistence model is accepted,
- the Expected LAT/BE formulas are accepted,
- the manual segment blending model is accepted,
- the duplicate/stale behavior is accepted,
- the 7:15 AM lock/reset behavior is accepted,
- the temporary Manual IN what-if isolation is accepted.

If accepted, this document is ready for a final red-team audit before coding.
