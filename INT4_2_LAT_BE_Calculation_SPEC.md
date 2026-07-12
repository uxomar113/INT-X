# INT 4.2 LAT / BE Expansion — Phase 2 and Phase 3 Calculation Specification

## 1. Authority and dependency

This file is the authoritative calculation specification for Phase 2 and Phase 3.

Source application:

- Repository: `uxomar113/INT-X`
- File: `INT4.2.html`
- Frozen baseline blob SHA before Phase 1: `161ac4d2054cc4279653f062477c0507704b2dcd`

Phase 2 starts only after Phase 1 passes audit. Phase 3 starts only after Phase 2 passes audit.

## 2. Permanent scope boundary

This update adds only:

- LAT%
- EXP%
- BE Req
- BE%
- EXP2%
- Minimal in-memory 30-minute interval state required for these metrics

Do not redesign or replace:

- Snapshot parser
- LOB detection
- IN / OUT status classification
- IN / OUT counting
- Manual IN / OUT adjustment
- Existing REQ input
- Existing AUTO REQ scheduler
- Existing `% Req` formula
- Existing `Heads` formula
- AVAIL timer
- Existing clear behavior except resetting the new interval state

Do not add localStorage, history UI, operating-window restrictions, stale/frozen modes, duplicate-snapshot logic, parser redesign, manual LAT/BE override, LAT velocity, new roster behavior, or new AUTO REQ syntax.

## 3. Shared definitions

All state and calculations are per LOB.

```text
effectiveIn = max(0, baseIn + adjIn)
effectiveOut = max(0, baseOut + adjOut)
effectiveReq = parseFloat(existing REQ input)
```

REQ is valid only when finite and greater than zero.

AUTO REQ already writes into the existing REQ input. LAT and BE must naturally use the updated value without changing AUTO REQ priority or syntax.

All four metrics display two decimals:

```text
00.00%
```

Invalid or unavailable values display:

```text
—
```

Never display `NaN%` or `Infinity%`.

## 4. Thirty-minute interval model

Use local device time and standard wall-clock intervals:

```text
HH:00–HH:30
HH:30–next HH:00
```

The engine must work at all times of day. Do not add an operating-hours filter.

Maintain minimal in-memory state per LOB, equivalent to:

```text
intervalStartMs
intervalEndMs
lastSampleMs
currentReqPctRaw
currentBePctRaw
latWeightedSum
beWeightedSum
latHasValidData
beHasValidData
```

Do not persist this state.

### First valid observation

When the first valid value is observed during an interval, treat that value as applying from the interval start to the observation time. This is required because no earlier sample exists.

LAT and BE are tracked independently. LAT may be valid while BE is unavailable.

### Segment update rule

When a value-changing event occurs:

1. Accumulate the prior instantaneous percentage from `lastSampleMs` to the event timestamp.
2. Replace it with the newly calculated instantaneous value.
3. Set `lastSampleMs` to the event timestamp.
4. Recalculate the displayed average and projection.

Repeated rendering without a value change must not duplicate time.

### Interval rollover

At each half-hour boundary:

1. Accumulate the final segment through the old interval end.
2. Discard old interval calculation state after finalization; no history UI is required.
3. Start a clean new interval.
4. Seed it at the boundary from the current effective IN, REQ, and BE Req values when valid.

At most one new recurring timer may be added for the LAT/BE engine. A one-second tick is acceptable.

## 5. Phase 2 — LAT% and EXP%

Phase 2 implements LAT% and EXP% only. BE Req remains disabled. BE% and EXP2% remain `—`.

### Instantaneous REQ percentage

Use the same raw ratio as the existing `% Req` metric:

```text
currentReqPctRaw = effectiveIn / effectiveReq * 100
```

Do not divide REQ by `0.92`. Do not alter the existing `% Req` calculation.

LAT pass target:

```text
92.00%
```

The target is for pass/fail styling only; it is not part of the denominator.

### LAT% definition

LAT% is the time-weighted average of `currentReqPctRaw` over the elapsed portion of the active 30-minute interval.

```text
elapsedSeconds = (now - intervalStartMs) / 1000
latNumerator = sum(segmentReqPctRaw * segmentDurationSeconds)
LAT% = latNumerator / elapsedSeconds
```

Include the open current segment through `now`.

### EXP% definition

EXP% projects the final 30-minute LAT average assuming the current instantaneous REQ percentage continues for the rest of the interval.

```text
remainingSeconds = (intervalEndMs - now) / 1000
EXP% = (latNumerator + currentReqPctRaw * remainingSeconds) / 1800
```

At interval end, `EXP% = LAT%`.

### Phase 2 update triggers

Close the previous LAT segment and update LAT/EXP when:

- A new snapshot changes effective IN
- Manual IN `+` or `-` changes effective IN
- Existing REQ changes
- AUTO REQ applies a new REQ
- Per-LOB clear resets that LOB
- Master clear resets all LOB LAT state
- A half-hour boundary occurs

Manual OUT changes do not mathematically change LAT, but existing refresh behavior must remain untouched.

### Invalid LAT state

If REQ is blank, non-finite, or less than or equal to zero:

```text
LAT% = —
EXP% = —
```

When REQ becomes valid again during the same interval, reset that LOB's LAT accumulator and apply the first-valid-observation rule from the current interval start.

### Phase 2 example

Current interval: 10:00–10:30.

```text
10:00–10:10 current REQ% = 80%
10:10–10:15 current REQ% = 100%
now = 10:15
```

```text
LAT numerator = 80 * 600 + 100 * 300 = 78,000
LAT% = 78,000 / 900 = 86.67%
EXP% = (78,000 + 100 * 900) / 1800 = 93.33%
```

## 6. Phase 3 — BE Req, BE%, and EXP2%

Phase 3 enables the Phase 1 BE Req field and implements BE% and EXP2%. It must not alter the completed LAT/EXP engine.

### BE Req rules

BE Req is:

- Per LOB
- Manual input only
- Integer
- Minimum `0`
- Step `1`
- Blank is invalid
- Zero is valid
- Independent from existing REQ and AUTO REQ

### Approved BE denominator and current value

```text
BE denominator = effectiveReq + beReq
currentBePctRaw = effectiveIn / (effectiveReq + beReq) * 100
```

BE pass target:

```text
100.00%
```

REQ is the existing visible REQ value. BE Req is added only to the BE denominator.

### BE% definition

BE% is the time-weighted average of `currentBePctRaw` over the elapsed portion of the active 30-minute interval.

```text
elapsedSeconds = (now - intervalStartMs) / 1000
beNumerator = sum(segmentBePctRaw * segmentDurationSeconds)
BE% = beNumerator / elapsedSeconds
```

Include the open current segment through `now`.

### EXP2% definition

EXP2% projects the final 30-minute BE average assuming the current instantaneous BE percentage continues for the rest of the interval.

```text
remainingSeconds = (intervalEndMs - now) / 1000
EXP2% = (beNumerator + currentBePctRaw * remainingSeconds) / 1800
```

At interval end, `EXP2% = BE%`.

### Phase 3 update triggers

Close the prior BE segment and update BE/EXP2 when:

- A new snapshot changes effective IN
- Manual IN `+` or `-` changes effective IN
- Existing REQ changes
- AUTO REQ applies a new REQ
- BE Req changes
- Per-LOB clear resets that LOB and clears its BE Req
- Master clear resets all BE interval state
- A half-hour boundary occurs

Changing BE Req must not modify LAT history. Changing REQ affects both LAT and BE.

### Invalid BE state

BE and EXP2 are valid only when:

```text
effectiveReq is finite and > 0
beReq is an integer and >= 0
(effectiveReq + beReq) > 0
```

Otherwise:

```text
BE% = —
EXP2% = —
```

When BE becomes valid again during the same interval, reset that LOB's BE accumulator and apply the first-valid-observation rule from the interval start.

### Phase 3 examples

Current BE value:

```text
effectiveReq = 10
beReq = 2
effectiveIn = 12
current BE = 12 / (10 + 2) * 100 = 100.00%
```

Mixed interval:

```text
first 10 minutes = 75%
next 5 minutes = 100%
now = minute 15
```

```text
BE% = (75 * 600 + 100 * 300) / 900 = 83.33%
EXP2% = (75 * 600 + 100 * 300 + 100 * 900) / 1800 = 91.67%
```

## 7. Rendering rules

LAT and EXP:

- `>= 92.00%`: good style
- `< 92.00%`: bad style
- `—`: neutral style

BE and EXP2:

- `>= 100.00%`: good style
- `< 100.00%`: bad style
- `—`: neutral style

Do not alter existing `% Req` or `Heads` styling beyond current behavior.

## 8. Integration requirements

Use one centralized update path, for example:

```text
updateLatBeMetrics(id, timestamp)
```

Equivalent naming is acceptable.

The engine must use event timestamps and interval boundaries, not render-count approximations.

Do not rebuild the full LOB DOM every second. Update only the four new metric text nodes and classes.

## 9. Audit tests

Regression tests for every phase:

- Same snapshot produces the same IN / OUT / TOT as the frozen baseline
- Existing `% Req` remains `IN / REQ * 100`
- Existing `Heads` remains unchanged
- AUTO REQ syntax and timing remain unchanged
- Ten-second textarea clear remains unchanged
- AVAIL remains unchanged
- No external dependency is added

Phase 2 tests:

- Constant 100% gives LAT 100.00% and EXP 100.00%
- Constant 80% gives LAT 80.00% and EXP 80.00%
- Mixed example gives LAT 86.67% and EXP 93.33%
- REQ change closes the previous segment at the change timestamp
- AUTO REQ change closes the previous segment at the application timestamp
- Half-hour rollover starts a clean accumulator
- Invalid REQ displays `—`

Phase 3 tests:

- IN 12, REQ 10, BE Req 2 gives current BE 100.00%
- Constant 100% gives BE 100.00% and EXP2 100.00%
- Mixed example gives BE 83.33% and EXP2 91.67%
- BE Req change closes only the BE segment and does not corrupt LAT
- REQ change closes both LAT and BE segments
- Blank BE Req leaves LAT/EXP operational while BE/EXP2 show `—`
- BE Req `0` is accepted

## 10. Phase gates

Phase 2 must modify only `INT4.2.html` and must not implement Phase 3 early.

Phase 3 must modify only `INT4.2.html` and must not expand scope.

Each phase requires:

- JavaScript syntax check
- HTML parse check
- Deterministic formula tests
- `git diff --check`
- Explicit confirmation that parser, AUTO REQ, `% Req`, Heads, AVAIL, and clear behavior remain preserved
